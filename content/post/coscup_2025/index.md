---
title: "Coscup 2025: Let's Tracing Linux Kernel, MGLRU 實作與分析"
description: "Coscup 2025: Let's Tracing Linux Kernel, MGLRU 實作與分析"
date: 2025-07-20T13:23:54+08:00
image: 
math: 
license: 
image: coscup.png
categories:
    - Coscup 2025
tags:
    - MGLRU
    - Linux Kernel
    - Tracing
    - Perf
hidden: false
comments: true
draft: false
---
# Coscup 2025: Let's Tracing Linux Kernel, MGLRU 實作與分析
## 大綱
- 簡介，什麼是 page replacement?
- 回顧，關於 Active/Inactive LRU
- 追蹤，關於追蹤 kernel的 page replacement
- 介紹 MGLRU
- MGLRU call chain 解釋
	- Aging
	- Eviction
	- PID 控制
	- Rmap 優化
- 觀測 MGLRU 行為
- 結論


## 簡介，什麼是 page replacement?
首先回顧 Process 存取記憶體的流程

- CPU 存取記憶體: 當程式嘗試存取記憶體時，CPU 會先去查詢 TLB，如果 TLB Miss，則查詢 Page Table
- 查詢 Page Table: 如果發現 Page Table 沒有該虛擬記憶體對應到的實體記憶體映射，或是映射標記為無效，則 CPU 會觸發 Page Fault，產生 exception，並進入 Page Fault Handler
- 這時候有幾個選擇，如果 page 已經在記憶體，但沒有映射到目前 Process 的記憶體空間，則只需要更新映射就可以。如果 page 不在記憶體，需要從硬碟載入。如果 page 都不可使用，則系統需要選擇一個策略將目前實體記憶體的 page 進行替換，將某個 page 換出到次級記憶體，如 CXL, swap 等等，以釋放出空間給新的 page 使用，這時候可能使用 LRU 或是 MGLRU 等等策略

對於記憶體，主要問題有兩個，一個是什麼樣的物件應該保留在 cache 中，這個由 page replacement 策略決定，像是 MGLRU，而另一個議題是如何在 cache 中放入更多的物件，關於這部份近期的討論為 zram，不過我們會將重點放在 MGLRU。

### 詳細關於 CPU 記憶體存取機制
當程式嘗試存取記憶體 (虛擬記憶體) 時，會進行以下

#### CPU 查詢 TLB
確認虛擬記憶體地址是對應到哪一個實體記憶體地址
- TLB 查詢: CPU 先查詢 TLB，TLB 裡面儲存最近使用過得虛擬記憶體到實體記憶體之間的映射關係
- TLB Hit: 如果在 TLB 中找到對應的映射，發生 TLB hit，則 CPU 直接使用該實體記憶體來存取記憶體
- TLB Miss: 如果 TLB 中沒有找到對應的映射，則 CPU 需要先查詢主記憶體中的 Page Table 來獲取實體記憶體，這時候會需要進行 Page Table Walk

得到實體記憶體地址後，到 CPU cache 中查詢是否有實體記憶體地址對應的資料
- CPU Cache 檢查: CPU 首先檢查其內部 Cache，L1, L2, L3 Cache，如果資料已經被快取，則會發生 cache hit，直接使用該資料
- Cache Miss: 如果 Cache 中沒有需要的資料，會發生 Cache miss，CPU 需要從主記憶體中讀取資料

#### 關於 Page Table
Page Table 中儲存虛擬記憶體到實體記憶體之間的映射關係，最後一層為 PTE (Page Table Entry)，包含以下幾個欄位
- P (Present Flag): 表示該 page 是否位於實體記憶體中
- R/W (Read/Write Flag): 表示該 page 的讀寫權限
- A (Accessed Flag): 表示 Page 是否被存取過
- D (Dirty Flag): 表示 Page 是否被修改過


#### 關於 Page Fault
當 Page Table 中沒有找到有效的映射，或是映射被標記為無效時，又或者是因為缺少權限等，CPU 會產生 Page Fault Exception，控制權會轉交給 Page Fault Handler，Page Fault 可以分成以下幾種類型
- Minor Page Fault
  當 process 存取虛擬記憶體對應到的 page 已經存在於物理記憶體，但是尚未建立有效的映射到 Process Page Table，這時後會觸發 Minor Page Fault，這時候不需要硬碟 I/O，只需要更新 page table 的映射或是權限即可恢復執行
	- 常見的為 malloc 中的 lazy allocation，CPU 發現虛擬記憶體地址沒有對應的實體 page frame，這時 kernel 分配 page frame 並更新 PTE，接著恢復程式執行，不需要硬碟 I/O
	- Copy-On-Write (COW): 共享 page 首次寫入時觸發 copy
	- 動態函式庫首次載入時，如果其 page 已經存在於 page cache，則會觸發 Minor Page Fault，kernel 只需要更新 page table 的映射，映射到 Process address space
- Major Page Fault:
  當所需的 page 不在實體記憶體中，需要從次級記憶體，如硬碟, zswap, zram 等載入時，會發生 Major Page Fault，這種 Page Fault 處理需要消耗更多的時間，因為涉及 I/O 操作
	- 從 zram 或是 swap 載入資料
- Invalid Page Fault:
當 Process 存取的記憶體地址不在 Process 的虛擬記憶體空間中，或是違反存取權限，會發生 Invalid Page Fault。

#### 關於 page replacement
當系統中 free page list 沒有足夠的 free page frame，這時候需要啟動 page replacement 機制。作業系統會根據以下策略選擇 victim page，並進行 eviction 操作，常見策略包含以下
- LRU: 替換最久沒有使用的 Page
- Clock / Second-Chance: 硬體成本較低的近似 LRU
- MGLRU (Multi-Generational LRU): 自 Linux Kernel 6.1 之後引入的策略，將 page 根據存活週期以及 hot, cold 分成多個 generation

整理以上，我們知道 page replacement 會在兩個情境下觸發
- 記憶體壓力大時 (background reclaim): 當系統監測到目前記憶體壓力大時，如空閒記憶體空間低於某個特定水位，`kswapd` 會被喚醒，主動回收 page 維持系統可用的記憶體空間，主動選出哪一些 page 為 cold page 這部份會需要 page replacement 邏輯
- 由 page fault 觸發 (direct reclaim): 當發生 Major Page Fault，需要分配 page frame 時，如果 free page frame 不足，則會同步啟動 page replacement 機制，由 page fault handler 直接 evict 現有的 page，釋放空間給新的 page 使用

### 關於 Page Reclaim
在上面 Major Page Fault 中，提到當沒有足夠 free page frame 會需要啟動 page replacement 機制，Linux 會啟動 Page 回收機制，具體行為是由 (`kswapd` 或直接由 page fault handler 呼叫 `shrink_node()`) 嘗試釋放 page frame。回收時會根據 page 屬於的 LRU 層級，從 cold 資料優先選擇 victim page

- LRU page 分類 (會將 anon page 與 file-backed page 分開進行管理)
	- `inactive` list: 使用頻率低的 page，優先被回收
	- `active` list: 近期有被存取的 page，回收之前
	- MGLRU: 將 page 更細分成多個 generation

如果記憶體壓力過大，`kswapd` 無法即時釋放得到足夠多的 free page frame，則會呼叫 direct reclaim，或是進行 memory compaction，合併可用的 free page frame，如果還是無法分配，則可能導致 OOM (Out-of-Memory) 機制觸發

## 回顧，關於 Active/Inactive LRU
就是把 LRU 分成兩個 list，分別為 active list 和 inactive list，然後 anon page 和 file-backed page 互相獨立，再加上 unevictable page list，總共有 5  個 list

- Inactive LRU list: 新載入或是剛剛使用的 page 一開始放入 Inactive list 中 (這一些 page 處於未活躍狀態)
- Active LRU list: 如果某個 page 在 Inactive list 中再次被存取 (該 page 第二次被使用)，則該 page 會被 promote 到 Active list，並且被標記成 Active page。Active LRU list 中的 page 表示最近常用或是 hot page

5 個 list 可以在 Linux Kernel 原始碼中看到
```c
enum lru_list {
    LRU_INACTIVE_ANON = 0,
    LRU_ACTIVE_ANON = 1,
    LRU_INACTIVE_FILE = 2,
    LRU_ACTIVE_FILE = 3,
    LRU_UNEVICTABLE = 4,
    NR_LRU_LISTS
};
```
而這一些 list 會存放在 lruvec 中
```c
struct lruvec {
    struct list_head lists[NR_LRU_LISTS];
    ...
};
```

- 新的 page 預設會放入 Inactive LRU，當該 page 再次存取之後才會移動到 Active LRU
- 記憶體開始出現壓力之後會進行 Eviction，Eviction 的行為是從 Inactive LRU 的末端開始回收
- 如果 Inactive LRU 中 page 數量比較低，則從 Active LRU 尾端開始移動 page 到 Inactive LRU

###  Inactive-Active LRU list 的運作:
當系統記憶體充足時，page 可以在兩個 LRU list 中移動。如果出現記憶體壓力，有 page 需要進行 Evict (Eviction: 將 page 從記憶體中移除)，Kernel 會從 Inactive LRU list 的末端開始回收不常使用的 page。這意味著只使用一次的那一些 page 會被優先淘汰，而 Active LRU list 中的 page 會被保留。

如果 Inactive LRU list 長度太短，不足以提供 page 回收，則這時候會從 Active LRU list 末端選出一些 page 進行 demote 到 Inactive LRU list，增加 kernel 可以回收的 page 數量。

### 雙 LRU list 設計目的:
這樣設計的核心概念是避免單次存取的 page 去干擾 working set，假想一下，如果只有一個 LRU list，由於 list 大小有限 (隱含 cache 大小有限)，那些只存取一次的 page 很有可能會把頻繁存取的 page 給擠出 LRU list，對整個程式的 working set 造成干擾。Active LRU list 相當於 working set 的保護區，Inactive LRU list 則是容納最近沒有重複使用的 page。

雙 list 可以增加他的狀態表示數量 (包含 timestamp)，只有單 list，可以表示 hot/cold/dirty 

對於掃描模式，雙 list 可以讓他只污染到 inactive list (不過這部份要看 promote 邏輯)

2Q lRU 與 active/inactive 是一樣的，MGLRU 動機: 為了提昇鑑別度，只有分兩層可能要掃描很久


:::info
working set 定義: 指一個應用程式在一段時間內正常執行所需要的時常存取的 page，也就是 hot page 所構成的集合
:::

### 理想情況下 Active-Inactive LRU 表現
理想況下我們希望 Active-Inactive LRU 能夠很好的去避免那一些一次存取的 page 對一個應用程式的 working set 造成干擾。也就是 working set 中的 page 不會被 evict，我們希望有一個足夠長的 Inactive LRU 來避免對 working set 的干擾。

舉一個例子，一個應用程式的 working set 大小為 6 個 page，而 Inactive LRU 長度為 8，接著開始存取 page，page 依序進入到 Inactive LRU，接著我們第二次存取 page，由於 Inactive LRU 長度大於 6，所以所有 page 都留在記憶體中，最終這一些 page 都可以被 promote 到 active list 中，也就是整個應用程式的 working set 順利保留在記憶體中。

### Active-Inactive LRU 可能的問題
Active-Inactive LRU 問題在理想情況下可以保護 working set，但在實際情況下存在以下不足
- Rmap 開銷問題: 每次在回收或是檢查 page 的時候都需要通過 Rmap 找到相關的 page table entries，要知道這個 page 是被哪一些 process 所使用，這對 CPU 有著莫大的開銷
- page 熱度探測: 必須等待該 page 再次被存取 (產生 page fault) 才能判斷該 page 為 hot page 並將其躍遷到 Active LRU list
- 分類簡單: 只有 Inactive/Active，也就是 page 不是 hot 就是 cold，分類上不夠細緻

上面的問題在非理想情況下 working set 可能會受到很大的影響，無法有效的保留在記憶體中，例如 working set 中 page 存取相對距離長，距離超過 Inactive LRU list 的大小，在 Active/Inactive LRU list 中可能會出現頻繁將 page 換出換入的情況 (Thrashing)。
  
為了解決上面的問題，希望有個演算法能夠減少 Rmap 開銷並且提供更細緻的分類，為此，MGLRU 在 Linux Kernel 6.1 版正式引入

#### 補充: 關於 Rmap
Rmap 為 Linux Kernel 中用於追蹤 Page Frame 被哪一些虛擬記憶體地址映射的機制，核心功能為以下
- 反向查詢，給定一個 Page Frame，找出所有映射他的虛擬記憶體地址，也就是該 Page Frame 的使用情況，過程中需要遍歷 Process 的 Page Table 
- 在 page 回收或是遷移時，需要更新所有相關 Process 的 Page Table，確保虛擬記憶體地址和實體記憶體地址之間的一致性

Rmap 在 Page Replacement 中通常會在以下兩個情況下觸發
- Page Reclaim: 當記憶體壓力大時，需要回收不常使用的 page，這時後會使用 Rmap 來找到哪一些 Process 正在使用該 Page Frame。接著會去更改這一些 Process 的 Page Table，移除這一些 Process 對於 Page Frame 的映射，讓 Page Frame 可以被釋放
- Page Migration: 當需要進行 Page Migration，例如將 Page 移動到其他 NUMA 節點上面優化資料存取，這時候 Rmap 會用來更新所有映射該 Page Frame 的 Process 的 Page Table 中虛擬記憶體地址，使其指向到 Page Frame 新的位置

當 Kernel 需要回收一個 page 時，需要執行以下操作
1. 藉由 page replacement 機制選出 Victim Page，通常這個 Page 為 Cold Page
2. 接著檢查 Page 狀態，如果是 Dirty 需要 write-back，如果是共享 Page，需要更新該 Page 關聯的所有 Process 的 Page Table
3. 接著需要清除該 Page 相關的所有 PTE 中 Present 標誌，表示該 Page 已經不在記憶體中

Rmap 在上面共享 page 處理以及存取訊息蒐集時都需要進行對應呼叫
1. 如果 Page Frame 被多的 Process 共享，需要通過 Rmap 找到所有相關 PTE 並更新
2. 檢查 PTE 的 Accessed, Dirty 等標記，輔助判斷 Page 的熱度



## 追蹤，關於追蹤 kernel 裡特定子系統

根據我們上面的推論，MGLRU 的整個流程會在兩個情況觸發

- 當 page fault 發生時
- 當記憶體壓力大時，觸發 swap 機制時

也就是我們可以嘗試製造記憶體壓力大的場景，喚醒 `kswapd` 觸發 MGLRU 或是 LRU 的回收機制，具體可以使用以下三種作法

- 使用 `stress-ng` 製造記憶體壓力
- 使用特定 workload，接著使用 `cgroup` 限制其能夠使用的記憶體資源
- 修改開機的 kernel-parameter，更改系統可以使用的記憶體大小，接著執行正常 workload

我們可以先使用 perf 去抓取 workload 期間的系統 call stack，得到 perf.data，接著對這個資料進行分析，得知 kernel 裡面某個 function 的執行流程，如 mglru

### 使用 `stress-ng` 製造記憶體壓力
```shell
$ stress-ng --vm 4 --vm-bytes 123G --vm-keep --timeout 30s
```

### 使用測試程式，搭配 cgroup 限制記憶體資源
```c
int main() {
    size_t size = 110 * GB;
    void *buffer = NULL;
    size_t pagesize = get_page_size();
    int ret = posix_memalign(&buffer, pagesize, size);
    FILE *fp = fopen("va_pa_map.txt", "w");

    if (ret != 0) {
        fprintf(stderr, "posix_memalign failed: %s\n", strerror(ret));
        return 1;
    }
    
    if (!buffer) {
        perror("malloc");
        return 1;
    }

    printf("Allocating and touching 110 GB...\n");
    volatile unsigned long long counter = 0;
    for (size_t i = 0; i < size; i += pagesize) {
        counter++;
        volatile char *ptr = (char *)buffer + i;
        *ptr = 1;  // touch page
        asm volatile("" ::: "memory");
        uintptr_t va = (uintptr_t)ptr;
        uint64_t pa = get_physical_address(va);
        if (pa == 0) {
            printf("[USER] VA: 0x%lx -> PA: not mapped\n", va);
            fprintf(fp, "[USER] VA: 0x%lx -> PA: not mapped\n", va);
        } else {
            printf("[USER] VA: 0x%lx -> PA: 0x%lx\n", va, pa);
            fprintf(fp, "[USER] VA: 0x%lx -> PA: 0x%lx\n", va, pa);
        }
        printf("Counter = %d\n", counter);
    }
    printf("Sleeping to allow kernel reclaim...\n");
    sleep(30);
    return 0;
}
```
接著使用 cgroup 執行
```shell
#!/bin/bash

SLICE_NAME=appdemo.slice
UNIT_NAME=alloc_$(date +%s)
MEM_LIMIT="300M"

echo "[*] build systemd slice：$SLICE_NAME"

# create slice, limit memory resource
sudo systemctl set-property --runtime "$SLICE_NAME" MemoryMax=$MEM_LIMIT

echo "[*] start running workload, running in slice"

sudo systemd-run --unit="$UNIT_NAME" --slice="$SLICE_NAME" --pty --same-dir ./alloc

echo "[*] workload finish, try cleanup"

# cleanup
sudo systemctl stop "$UNIT_NAME"
sudo systemctl reset-failed "$UNIT_NAME"
sudo systemctl stop "$SLICE_NAME"

CGROUP_PATH="/sys/fs/cgroup/memory/$SLICE_NAME"
if [ -d "$CGROUP_PATH" ]; then
    echo "[*] waiting for cleanup"
    while [ -s "$CGROUP_PATH/cgroup.procs" ]; do
        sleep 0.5
    done
    echo "[*] remove cgroup：$CGROUP_PATH"
    sudo rmdir "$CGROUP_PATH" 2>/dev/null || echo "Error"
fi

echo "[*] Done"
```

### 關於 flamegraph 與 dot-graph
flamegraph 為視覺化工具，用來顯示程式在執行期間的 stack traces，可以用來得知某個系統的效能瓶頸以及分析函式呼叫關係。以 Linux 中的使用，我們可以將 perf.data 作為後端資料，接著通過前端腳本對 perf.data 進行過濾之後得到 flamegraph。

以下為通過 `perf report` 檢視的部份原始資料
```
kswapd0      73 [002] 4400339479.836174:   15575919 cycles:P: 
        ffffffff8122b313 shrink_folio_list+0x143 ([kernel.kallsyms])
        ffffffff8122f903 evict_folios+0x153 ([kernel.kallsyms])
        ffffffff81231285 lru_gen_shrink_node+0x125 ([kernel.kallsyms])
        ffffffff81231f1c balance_pgdat+0x29c ([kernel.kallsyms])
        ffffffff81232593 kswapd+0x1e3 ([kernel.kallsyms])
        ffffffff810c1bf7 kthread+0xd7 ([kernel.kallsyms])
        ffffffff810441fc ret_from_fork+0x3c ([kernel.kallsyms])
        ffffffff8100267a ret_from_fork_asm+0x1a ([kernel.kallsyms])

kswapd0      73 [002] 4400339479.836174:   15575919 cycles:P: 
        ffffffff81225938 folio_update_gen+0x38 ([kernel.kallsyms])
        ffffffff8127486a walk_pgd_range+0x24a ([kernel.kallsyms])
        ffffffff812750eb __walk_page_range+0x19b ([kernel.kallsyms])
        ffffffff8127525c walk_page_range+0x14c ([kernel.kallsyms])
        ffffffff8123049f try_to_inc_max_seq+0x46f ([kernel.kallsyms])
        ffffffff81231110 get_nr_to_scan+0x60 ([kernel.kallsyms])
        ffffffff8123126b lru_gen_shrink_node+0x10b ([kernel.kallsyms])
        ffffffff81231f1c balance_pgdat+0x29c ([kernel.kallsyms])
        ffffffff81232593 kswapd+0x1e3 ([kernel.kallsyms])
        ffffffff810c1bf7 kthread+0xd7 ([kernel.kallsyms])
        ffffffff810441fc ret_from_fork+0x3c ([kernel.kallsyms])
        ffffffff8100267a ret_from_fork_asm+0x1a ([kernel.kallsyms])

```

以下為 perf.data 過濾之後得到的 flamegraph

![image](https://hackmd.io/_uploads/SyghIoyeee.png)
```shell
$ sudo perf record -F 99 --call-graph dwarf -ag -- sleep 60
# running workload
$ sudo perf script > mglru_trace.perf
$ ./FlameGraph/stackcollapse-perf.pl mglru_trace.perf > mglru_folded.txt
$ ./FlameGraph/flamegraph.pl --title="MGLRU Memory Reclaim" --width 2000 --colors mem mglru_folded.txt > mglru_flame.svg
```
![image](https://hackmd.io/_uploads/B1oNadU8lx.png)




:::info
使用 DWARF 可以提供更多 debug-info，詳細可以參考 Kernel Recipes 2017 - Perf in Netflix - Brendan Gregg
:::

可以看到數據非常的龐大，這時候我們需要對我們感興趣的部份進行二次過濾，我們感興趣的目標為 mglru，而我們上面推論得到 mglru 會在記憶體壓力大時執行，Linux 在記憶體壓力大時會觸發 swap 機制，而 swap 機制是由 kswapd 執行，因此我們可以過濾出關於 kswapd 的資料來得到 mglru 的 call stack

![image](https://hackmd.io/_uploads/rkS1vjyexx.png)
```shell
$ grep -E "kswapd0" mglru_folded.txt | ./FlameGraph/flamegraph.pl --colors mem --width 2000 --title="MGLRU" > mglru_specific.svg
```


從上圖我們就可以清楚的看到整個 mglru 的 stack traces。上面過濾出的，是針對記憶體壓力大時，觸發 swap 機制。

由 kswapd 的圖我們可以看出存在兩條主要路徑，分別為

路徑1. kswapd -> balance_pgdat -> shrink_node -> shrink_one  -> try_to_inc_max_seq -> walk_page_range

路徑2. kswapd -> balance_pgdat -> shrink_node -> shrink_one -> try_to_shrink_lruvec


下面我們也可以針對 page fault 進行過濾，得到以下 flamegraph

![image](https://hackmd.io/_uploads/rkh-Psyegx.png)
```shell
$ grep -E 'lru|folio|shrink|refault' mglru_folded.txt | head -n 20 > page_fault_info_mglru.txt
```

由 page fault 的圖我們可以看出一條主要路徑
路徑3. exec_page_fault -> do_fault -> filemap_fault -> lru_add -> lru_gen_add_folio

由 flamegraph 分析 mglru，我們得到 mglru 存在三條主要 stack straces 路徑，作用分別為以下
作用分別為以下作用分別為以下

- 路徑1. Promote/Aging (hot page 升級，產生新的 gen)
- 路徑2. Evict / Refault (用於 page 淘汰)
- 路徑3. Page Fault (將 page 加入到最新 gen，也就是 `max_seq`)

對於路徑分析，我們也可以使用 dot-graph 進行分析，可以更加具體的看到 function 之間的呼叫關係
![image](https://hackmd.io/_uploads/SJzzmiuxxx.png)
```shell
$ cat mglru_trace.perf | FlameGraph/stackcollapse-perf.pl > out.collapse
$ gprof2dot.py -f collapse out.collapse | dot -Tpng -o output.png
```

## MGLRU 描述
MGLRU 核心想法是將 page 按照熱度分層管理，但相比起 Inactive/Active LRU list，MGLRU 將 page 區分成多個 generation。每個 page 對應到一個 gen 值，表示該 page 屬於哪一個 geneartion。

以下為 MGLRU 中關鍵部份，大致上可以分成以下五個部份

- Geneartion 概念: page 兩種分類，gen 以及 tier
- Rmap 遍歷，得知一個 page/folio 是否為 young
- Page Table Walk: 查詢 Page Table 查看 PTE 是否為 young
- Bloom Filter: 縮小 Page Table Walk 需要走訪的範圍
- PID 控制器: 用來控制回收的 page 類型以及保護 page

### Geneartion 概念
MGLRU 分類 page 主要使用上面 Gen 以及 Tier，Gen 和 Tier 為不同的概念，以下詳細描述
- 世代 (Gen): 
	- Gen 意義: 每個 page 屬於一個 gen，表示 page 的熱度或是年齡。使用 sliding windows 實作
	- Gen 計算: gen 是由 `min_seq` 與 `max_seq` 定義，通過 `seq % MAX_NR_GENS` 計算出 gen
	- Gen 分類: `max_seq` 表示最年輕的 gen，`min_seq` 表示最老的 gen
		- youngtest gen: max_seq (最年輕，熱度最高)
		- young gen: max_seq - 1
		- old gen: min_seq + 1
		- oldest gen: min_seq (最老的，熱度最低)
	- Gen 維護: 對於 `max_gen`，anon page 和 file-backed 視為相同的 hot page，但是 anon page 和 file-backed 會分別維護 `min_seq`
- Promote 機制與 Demote 機制
	- Promote: 當 page 被再次存取時會 Promote 到新的 gen
	- Demote: 在 MGLRU 中不存在顯式 Demote 的操作，當 `max_seq` 增加時，沒有 Promote 的 page 相對年齡就增加了，達到隱式 Demote 的效果。
	- page 沈降: 隨著時間推移，沒有 Promote 的 page 自然變成 oldest gen，變得可能被回收的 page。當 oldest gen 中的 page 被全部回收，LRU list 被刪除並且 `min_seq++`，整個 windows 往前滑動
	- generation 不足: 當 generation 數量不足時，則會建立新的 generation 繼續區分 page 的熱度
  
- 層級 (Tier)
  定義: 每個 page (folio) 根據 (通過 fd 存取的次數，像是 `write(), read()`) 被分配到一個 Tier，假設 page 通過 fd 存取的次數為 $N$，則 $Tier = order\_base\_2(N)$。每個 Page 使用額外 2 bit 去紀錄他們屬於的 Tier，Tier 的最大值為 $MAX\_NR\_TIERS = 4$
	- Tier = 0 -> N = 0,1
	- Tier = 1 -> N = 2,3
	- Tier = 2 -> N = 4,5,6,7
	- Tier = 3 -> N = 8+
	(上面這個使用圖表進行表示)
	
- Refault 率統計
	- Refault 定義: 某個 page 被回收後再次被存取 (該 page 重新被載入到記憶體) 的比率
	- Refault 資料結構: 統計方式為通過 `lrugen->refaulted[hist][type][tier]` 記錄不同世代 (`hist`)，Page 類型 (`anon`/`file-backed`), Tier 的 refault 次數。使用這個陣列紀錄每一個 Tier 的 refault 率，也就是這個 Tier 被回收的 page 之後又被重新載入 (refault) 的比例。
	- Refault 率 = 被回收又重新載入的 page 數量 / 被回收的 page 數量
	- 決策機制:
		- 先比較 `min_seq` 的值，選擇較小值，也就是更老的 page 類型
		- 如果兩個 page 類型都相同，則比較 Tier 0 的 refault 率，選擇比較低的 page 類型進行回收
		- 使用 PID 控制器的回饋機制，動態平衡不同 page 類型的 refault 率
  
總結一下，每個 page 通過額外的 flags 紀錄自己屬於的 Tier，系統會維護額外的 array 去紀錄每一個 Tier 的 refault 率。eviction 策略會根據 refault 動態調整回收的行為，避免頻繁存取的 page 但是未達到 Promote 標準的 page 被過早的回收，也就是 refault 率可以用來判斷當兩種 page 類型年齡相同時，refault 率可以用來判斷要回收的 page，更加精細化 page 的分類。

由上面的描述，我們會發現到，當有一個 Page 位於最老的 gen，但是他的 refault 率高，這時候我們應該有一個機制對這類的 page 進行處理，避免他們被回收，影響到 working set，在 MGLRU 中存在 Protect 機制用來保護這一些 page。

- Protect 機制: 
	- 當某個 Tier 的 Refault 率高於 Tier 0 的 Refault 率，將會觸發 Protect 機制
	- 觸發 Protect 機制的 page，會被移動到 old gen `min_seq + 1`，相當於給這個 page second chance，避免整個 oldest gen 一次被回收乾淨，造成部份 Trashing 現象發生
	- 目標為了提高 MGLRU 對於密集 I/O 的 working set 保護效果
	  
以下為 mglru 關鍵資料結構程式碼
```c
struct lru_gen_folio {
	/* the aging increments the youngest generation number */
	unsigned long max_seq;
	/* the eviction increments the oldest generation numbers */
	unsigned long min_seq[ANON_AND_FILE];
	/* the birth time of each generation in jiffies */
	unsigned long timestamps[MAX_NR_GENS];
	/* the multi-gen LRU lists, lazily sorted on eviction */
	struct list_head folios[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];
	...
```
存放 page (folios) 為一個三維陣列結構，使用以下方式進行拆分
- 第一維度: Generation，最多 4 個 generation
- 第二維度: 屬於 anon page 還是 file-backed page
- 第三維度: NUMA Node，對於 UMA 為 1
  
記錄 Refault 率的結構也同樣位於 lru_gen_folio 中

```c
struct lru_gen_folio {
	/* the multi-gen LRU sizes, eventually consistent */
	long nr_pages[MAX_NR_GENS][ANON_AND_FILE][MAX_NR_ZONES];
	/* the exponential moving average of refaulted */
	unsigned long avg_refaulted[ANON_AND_FILE][MAX_NR_TIERS];
	/* the exponential moving average of evicted+protected */
	unsigned long avg_total[ANON_AND_FILE][MAX_NR_TIERS];
	/* can only be modified under the LRU lock */
	unsigned long protected[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS];
	/* can be modified without holding the LRU lock */
	atomic_long_t evicted[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS];
	atomic_long_t refaulted[NR_HIST_GENS][ANON_AND_FILE][MAX_NR_TIERS];
```
下圖簡單描述 gen, tier 之間的關係
![image](https://hackmd.io/_uploads/SkA9-sdgge.png)

下突圍 Protect 與 Promote 之間的關係
![image](https://hackmd.io/_uploads/BkZAbo_lge.png)

 
### 路徑1. Aging 機制
所謂 Aging，指的是通過建立新的 gen 提高 page 熱度判斷的機制。在 kernel 回收 page 的時候，當發現現有的 gen 數量不足以區分 hot/cold page 時，這時後會觸發 Aging 操作，Aging 操作會新增一個 gen。

```c
kswapd()
  └─ balance_pgdat()
      └─ shrink_node()
          └─ shrink_one()
              └─ try_to_shrink_lruvec()
                  └─ lru_gen_shrink_lruvec()
                      └─ should_run_aging()
                          └─ try_to_inc_max_seq()
                              └─ lru_gen_age_node()
                                  └─ walk_page_range (for each mm_struct)
                                      └─ folio_update_gen()
```
啟動 Aging 後，MGLRU 會通過 Page Table Walker 更新 page 的熱度，接著對於目前記憶體中每個 process/mm_struct，會呼叫 `walk_page_range()` 遍歷其 Page Table，以找到年輕的 page (PTE 的 Accessed bit 為 1 的)。為了降低開銷，遍歷之前會先查詢 Bloom Filter 找到值得掃描的 Page 區域 (例如包含多個 hot page 的 PMD)。在遍歷過程中，如果發現這個範圍內存在 Access Bit 為 1 的 PTE，表示該 page 近期有被存取過，屬於潛在的 hot page，這時後會通過 `folio_update_gen()` 標記該 page promote 到最新的 gen，也就是將該 page 的 generation 更新成 `max_seq`，而 promote 操作是只去修改 page 的 gen 標記，不是立即移動該 page 在 LRU list 中的位置，直到真正 evict 的時候才會進行移動

當一輪 Aging 的操作結束，新的 gen 產生，這時候 `max_seq + 1`，並紀錄目前的 timestamp。MGLRU 隨後會使用該 timestamp 實現 Thrashing 防護，如果之後嘗試回收 `min_seq` 裡面的 page，發現到 `min_seq` 的產生時間還沒超過預設的 TTL 閥值，則會暫緩回收該 gen。這個機制避免系統頻繁的回收/建立 gen，防止 working set 因為頻繁的在 gen 之間移動造成 Thrashing。

完成 Aging 之後，MGLRU 會繼續進行 page evict 的流程，不斷循環的執行 Promote / Evict / Aging 的流程，使得 hot page 保留在最新的 gen，cold page 沈積到最老的 gen。在任何時刻，MGLRU 最多保持 4 個 gen，這一件事情是通過 sliding windows (`4 >= max_seq - min_seq >= 2`) 保證。

以 call chain 進行分析，則可以得到以下

記憶體高壓，需要將 page swapout，這時候 kswapd 被喚醒，接著執行 `shrink_node()` -> `shrink_lruvec()` -> `lru_gen_shrink_lruvec` -> `should_run_aging` -> `try_to_inc_max_seq()` 執行主要的 Aging -> 內部對每個 memcg 的 `mm_list` 呼叫 `walk_page_range()` 遍歷 Page Table，在 callback 中對遍歷 Page Table 過程中發現的 young page (PTE 為 1) 的 page 呼叫 `folio_update_gen()` Promote 該 page 的 gen，同時將資訊回饋到 Bloom Filter 優化值得走訪的記憶體區域，全部完成之後準備進行 Eviction

整體流程為以下
<img src="https://hackmd.io/_uploads/r1GJMsdglg.png" width=350>

### 路徑2. Evict
:::info
關於 `walk_page_range()` 與 rmap 的差別

:::


Eviction 流程指的是從 `min_seq` 開始淘汰 page 的機制。在 MGLRU 中，始終只回收目前 `min_seq` LRU list 中的 page。當觸發 kswapd 時，kernel 會選擇 `min_seq` 進行掃描以及回收。由於 MGLRU 會將 hot page 通過 Aging 機制將 hot page Promote 到 `max_seq`，自然而然相對 cold page 就會沈降到 `min_seq`，`min_seq` 裡面的 page 為長時間沒有存取的 page，可作為回收的目標。對於 `min_seq`，MGLRU 會分別維護 anon page 和 file-backed page 兩個 list，因此可能存在 anon page 和 file-backed page 兩者 `min_seq` 不同步的情況。

以下為 Eviction 的 call chain
```c
kswapd()
  └─ balance_pgdat()
      └─ shrink_node()
          └─ shrink_one()
              └─ try_to_shrink_lruvec()
                  └─ evict_folios()
                      ├─ isolate_folios()
                      │    └─ scan_folios()
                      │         ├─ sort_folio()     // Promote 熱頁到次老世代
                      │         └─ isolate_folio()  // 選出 page
                      ├─ shrink_page_list()         // 對選出的 page 進行實際回收
                      │    ├─ folio_check_references()
                      │    │    └─ folio_referenced()
                      │    │         └─ lru_gen_look_around() // 局部性優化，尋找附近的 hot PTE
                      │    ├─ lru_gen_set_refs()    // 決定是否 Protect（移動到 min_seq+1）
                      │    └─ try_to_free_swap() / folio_free() // swap out 或釋放
                      └─ try_to_inc_min_seq()       // 若 oldest gen 已清空則淘汰並 min_seq++
```

進入 Eviction 流程後，關鍵函式 `lru_gen_shrink_lruvec()` 按照一定策略從 `min_gen` list 提取 page 進行處理。每次提取一批 page 後，會對這一些 page 逐一進行回收判斷。對每個 page (`struct folio`)，首先呼叫 `folio_check_references()` 檢查最近是否被引用或是屬於 working set 的 page，`folio_check_references()` 會進一步呼叫 `folio_referenced()` 執行 Rmap 走訪，查詢該 page 映射的所有 PTE，統計該 page 的引用情況。

關於 `folio_referenced` 的實現運用到空間局部性進行優化，當通過 Rmap 找到某個 process 的 PTE 值，MGLRU 會呼叫 `lru_gen_look_around` 查看該 PTE 相鄰的多個 PTE，如果附近的 PTE 也屬於年輕的 (也就是 Access Bit 為 1)，則這一些 page 會被標記為 Promote，之後便會真正 Promote 到 `max_seq`。利用 Look-around 機制，MGLRU 可以在一次 Rmap 中找到多個 hot page，減少重複遍歷 Page Table 的開銷，並且這些 PTE 對應到的上層 Page Table 節點，也就是 PMD 會被紀錄到 Bloom Filter 中，做為之後 Aging 階段的輔助判斷資訊 

對於每個從 `min_seq` 提取的 page，`folio_check_references` 會根據 rmap 的結果回傳 enum，該 enum 的意義是對 page eviction 的建議
- `FOLIOREF_RECLAIM`: page 最近沒有被 reference，可以回收
- `FOLIOREF_ACTIVATE`: page 最近被 reference，應該 Promote 並保留
- `FOLIOREF_KEEP`: 暫時不處理 (可能正在被其他程式使用或 lock)

如果 page 最近沒有任何引用，也就是 `referenced_ptes` 為 0，則屬於真正的 cold page，將按照該 page 屬於的類型進行 direct reclaim。屬於 file-backed page 會直接回收或是 write-back 到硬碟，而如果屬於 anon page 則會 swap 出去。

如果 page 最近有使用，則 MGLRU 會給該 page 第二刺激會，`folio_check_references()` 通過 `lru_gen_set_refs()` 函式判斷是否需要 promote 該 page，這裡具體為使用 Tier 去判斷該 page 是否最近被使用過以及是否屬於 working set (也就是被 evict 之後又重新載入的 page，refault 的概念)
- 第一次發現引用: 如果 page 此前沒有標記 Referenced 也不是屬於 working set 的 page，`lru_gen_set_refs()` 將該 page 標記為 referenced (設置 `PG_referenced` Flag)，接著回傳 false，意義為該 page 暫時不做 Promote。整個意義是紀錄該 page 引用了一次，給 page 一次機會留在目前的 gen，下一次再遇到該 page 的時候再做決定
- 再次被引用或是曾經被 refault: 如果 page 已經有 referenced 標記或者其 `PG_workingset` Flag 為 True (表示該 page 曾經 refault 過，屬於 working set 的一部分)，那麼 `lru_gen_set_refs` 會回傳 True，表示該 page 需要進行 Promote。這時候 `folio_check_references()` 會回傳 `FOLIOREF_ACTIVATE`，kernel 會將該 page 從 `min_seq` 移除並重新加入到 `max_seq`，實作為移動到 generation 最高的 list，避免這次被回收，這個提升 page 的操作也稱為對該 page 的 Protect

上面的策略，MGLRU 確保一個 page 要被 Protect，也就是 page 被 Promote 需要符合以下一個條件
- Page 被存取兩次
- 該 Page 屬於 working set
通過以上策略避免一些 page 因為一次偶然的存取就阻止該 page 被回收。而對於確實經常被存取的 page，MGLRU 會對該 page 進行 Protect 避免其被淘汰

隨著 eviction 機制進行，`min_seq` 裡面的 page 不是被回收釋放，不然就是因為他有引用而被移除 `min_seq`加入到 `max_seq`。Kernel 會持續進行上面的操作，直到 gen 裡面所有 page 處理完畢，接著該 gen 就會被丟棄，將 `min_seq + 1` (淘汰最老的 gen)，這部份由 `inc_min_seq()` 完成，負責更新 anon page 和 filed-back page 對應的 oldest gen seq 並清理相關資料結構。丟棄 gen 的同時，kernel 會去維護一些輔助資料結構，像是該 gen 中被回收的 page 數量，各 Tier 被回收和 refault 次數等等，特別會針對剛剛在回收或成因為 refault 率高而被 Protect 的 page 資訊也會被記錄下來，納入歷史統計中，用於調整後續的 eviction 策略。完成一個 gen 回收後，如果記憶體壓力沒有紓解並且需要釋放更多記憶體，MGLRU 會繼續重複上面 Aging + Eviction 過程

以 call chain 進行分析如下
`lru_gen_shrink_lruvec()` -> 從 `min_seq` list 中提取 page 進行處理 -> 對每個 page 呼叫 `folio_check_references()` 檢查是否被引用 -> `folio_eferenced()` 執行 rmap 查詢 Access Bit (裡面呼叫 `lru_gen_look_around`) 檢查鄰近 PTE 並標記 hot page，更新 Bloom Filter) -> 根據 `folio_check_references` 回傳的 enum 決定 folio 應該如何處理，沒有引用的進行回收 (呼叫 `try_to_free_swap` 或 `folio_free` 釋放)，如果有引用則呼叫 `lru_gen_set_refs` 判斷是否應該 Promote (進行 Protect) -> 如果需要 Promote 則將該 page 加入到 `max_seq` -> 繼續處理下一個 page，直到提取出的 page 都被掃描完畢 -> 處理完畢後呼叫 `inc_min_seq` 將該 gen 丟棄，並將 `min_seq + 1`

整體流程為以下
<img src="https://hackmd.io/_uploads/SyqGGj_exl.png" width=500>

整個 Eviction 和 Aging 和 Promote 行為一個循環圖，可以表示成以下
![image](https://hackmd.io/_uploads/ByJLfjuglg.png)


#### 關鍵函式 `try_to_shrink_lruvec` 分析
```c
static bool try_to_shrink_lruvec(struct lruvec *lruvec, struct scan_control *sc)
{
	long nr_to_scan;
	unsigned long scanned = 0;
	int swappiness = get_swappiness(lruvec, sc);

	while (true) {
		int delta;

		nr_to_scan = get_nr_to_scan(lruvec, sc, swappiness);
		if (nr_to_scan <= 0)
			break;

		delta = evict_folios(lruvec, sc, swappiness);
		if (!delta)
			break;

		scanned += delta;
		if (scanned >= nr_to_scan)
			break;

		if (should_abort_scan(lruvec, sc))
			break;

		cond_resched();
	}

	/*
	 * If too many file cache in the coldest generation can't be evicted
	 * due to being dirty, wake up the flusher.
	 */
	if (sc->nr.unqueued_dirty && sc->nr.unqueued_dirty == sc->nr.file_taken)
		wakeup_flusher_threads(WB_REASON_VMSCAN);

	/* whether this lruvec should be rotated */
	return nr_to_scan < 0;
}
```
上面為 MGLRU 中關鍵的程式碼，這是 Memory Reclaim 流程中真正要淘汰 page 的地方

整個函式可以分成以下部份
```c
int swappiness = get_swappiness(lruvec, sc);
```
- 首先取得 `swappiness`
	- `swappiness` 控制要回收的 page 傾向，是要傾向於 anon page 還是 file-backed page
	- 如果 `swappiness` 偏高，表示更傾向於回收 anon page，反之則 file-backed page。這個值會在 `get_nr_to_scan` 和 `evict_folios` 中使用，他們會根據 `swappiness` 決定要處理的 page 類型
    
```c
while (true) {
  nr_to_scan = get_nr_to_scan(lruvec, sc, swappiness);
  if (nr_to_scan <= 0)
    break;
```
- 接著進入主要回收迴圈，先計算要掃描的 page 數量
	- `get_nr_to_scan()` 根據目前的 memory pressure 和 generation 結構計算目前需要掃描
	- 如果發現結果 <= 0，表示此 lruvec 不需要掃描

```c
delta = evict_folios(lruvec, sc, swappiness);
if (!delta)
    break;
```
- 這裡是主要做回收的地方
	- `evict_folios()` 是實際掃描並嘗試回收某一層 generation 中的 page (folios) 的函式
	- `delta` 表示這次成功處理 (包含 reclaim 以及 skip) 的 page 數量
	- 如果 `delta == 0`，表示這一層 generation 沒有可以回收的 page，跳出迴圈

```c
scanned += delta;
if (scanned >= nr_to_scan)
    break;
```
- 計算已經掃描的 page 數量並確認是否達標
	- 通過 `scanned` 去累加這次總共掃描了多少 page
	- 如果已經達到預估得掃描數量 `nr_to_scan`，則退出
   
```c
if (should_abort_scan(lruvec, sc))
    break;
```
- 檢查是否該中止回收行為
	- 內部條件判斷是否要中止，例如 memory pressure 減緩，或 CPU 覆載過高

```c
cond_resched();
```
- 讓出 CPU 避免 starvation
	- `cond_resched()` 允許 scheduler 在這個點做 context switch，避免目前這個函式長時間佔用 CPU

```c
if (sc->nr.unqueued_dirty && sc->nr.unqueued_dirty == sc->nr.file_taken)
    wakeup_flusher_threads(WB_REASON_VMSCAN);
```
- 處理 dirty page 的狀況
	- 如果掃描到很多 page 都屬於 dirty file-backed page，且都沒有進入到 flusher queue，就會 wake up flusher thread
	- 避免 dirty page 卡在回收過程中

```c
return nr_to_scan < 0;
```
-  return 並由上層 caller 判斷是否需要進行 rotate
	- 如果 `nr_to_scan < 0`，表示某些 generation 掃描完畢，應該準備 rotate 到下一個 generation
	- 這個值會回傳到上層如 `shrink_one()` 決定是否進行 


`try_to_shrink_lruvec()`
- 真正執行 LRU page reclaim 的函式，函式目標為是對某個 memory cgroup 的某個 `lruvec`　嘗試執行 MGLRU 的回收邏輯
- caller 會根據目前的 memory pressure 決定要去哪一個 generation 做 reclaim
- MGLRU 會掃描某一層中 gen X (page vectors)，根據 page 的 hot cold 進行回收
- 之後通過 `evict_folios` 檢查後確認要回收


### PID 控制迴路與 anon/file-backed page 平衡
MGLRU 在 eviction 決策中引入類似於控制迴路中 PID 控制器的回饋機制，用於平衡 anon page 和 file-backed page 的回收比例。整套機制透過 Refault 率等指標作為回饋訊號來調整 eviction 策略，使得 working set 能夠得到保護。
- 回收 page 類型選擇 (P: 比例調節): MGLRU 會比較目前 anon page 與 file-backed page 的 refault 比率，傾向回收 refault 比率較低的 page 類型。如果每一類 page 被回收之後很少被再次存取，也就是很少被 refault，那表示這一類型的 page 對於目前的 working set 不重要，因此把這一些 page 回收掉對性能影響較小，反之，如果某一類型的 page 不斷被 refault，則表示這一類型的 page 包含目前的 working set，應該減少回收這一類型的 page。通過這種方式去平衡 anon page 和 file-backed page 的回收比例 (相比起 2Q LRU 使用 `swappiness` 固定比例，MGLRU 的實現為動態調整)。
  
  P 裡面的比率，指的是 refault 率
- working set 保護 (I: 積分調節): 為了進一步避免某部份 working set 被過度回收，MGLRU 引入 Tier 對 page 的存取頻率進行分類，並基於歷史統計數據對這一些 page 進行保護。每個 page 都有一個 2-bit 的欄位作為 Tier 值，總共分成四個層級。Tier 的計算主要是基於該 page 通過 syscall 如 `read(), write()` 的存取次數，如果存取次數為 $n$，則 Tier = $\lfloor \log_2(n) \rfloor$，例如如果一個 file-backed page 只被讀取一次，則 Tier=0。被 syscall 讀寫 2 次則 Tier = 1。anon page 通常不使用 Tier 劃分。MGLRU 會持續統計各個 Tier 中 page 被回收和 refault 的次數，從而計算每一 Tier 的 refault 率。在 eviction 過程中，會使用到積分調節的概念，也就是參考一段時間內的歷史資料，如果某些 Tier 較高的 page 歷史 refault 率偏高，即便在某一次掃描時該 page 沒有被引用，MGLRU 也會對該 page 進行 protect，避免一次回收過多此 Tier 的 page。使用過去累積的 refault 訊息調節 eviction 策略。
- D 在 MGLRU 中沒有使用

總而來說，MGLRU 的 PID 控制迴路將上面的 P 與 I 結合，P 即時的調整回收傾向的 page 類型以及層級，而 I 基於歷史統計對高 refault 率或是高 Tier 的 page 進行保護，P 和 I 共同作用達成保留 working set 的效果。例如某段時間內的 file-backed page 的 refault 率遠高於 anon page，P 會更傾向回收更多 anon page 以保護 working set，也就是保護 file-backed page。同時如果發現某 Tier 的 refault 率教高，則 I 會降低該 Tier page 的回收，必要時會針對個別 page 進行 Protect (移入 `min_seq + 1`)。MGLRU 對 `max_seq` 的 anon page 和 file-backed page 始終是同步的，但是對於 `min_seq` 是不同步的，原因是為了實現上面的 PID 動態平衡機制。傳統 2Q LRU 的第二次存取 page Promote 到 active list 的機制在 MGLRU 中使用 gen 以及上面的 Tier, PID 迴路機制取代，例如對於新映射的 page，一次存取立即放到 `max_seq`，而對於 syscall 存取的 file-backed page，則需要通過多次存取累積一定的 Tier 才會被 Promote。

### 關於 Bloom Filter 降低 Rmap 開銷
MGLRU 在 Aging 階段引入了 Bloom Filter 降低掃描 Page Table 和 Rmap 的開銷。2Q LRU 在判斷 page 是否進行有被使用時，需要對每個 page 進行 Rmap 操作來尋所有映射該 page 的 PTE 並檢查其 Accessed bit，Rmap 需要大量的開銷，為此，Rmap 通過以下方式進行優化
- 熱區紀錄: 在 eviction 過程中，當執行 Rmap 檢查 page 的引用時，MGLRU 會同時查看該 page 所在的 PMD 是否包含多個被引用的 page。這部份通過 `lru_gen_look_around()` 實現，如果 `lru_gen_look_around()` 在目標 page 的附近 PTE 中發現多個 page 的 Accessed Bit 都是 1，也就是這一些 page 都屬於年輕的，就表示這一整片 PMD 都是 hot page。MGLRU 會將對應的 PMD entry 紀錄到 Bloom Filter 中 (通過 `update_bloom_filter()` 將 PMD 紀錄到 Bloom Filter)
- double-buffering 與 flip filters: MGLRU 使用 double-buffering 的 Bloom Filter 實現 flip filter。Bloom Filter 實現的資料結構為 Bit Map 以及 hash function，Bit Map 大小為 $m=2^{15}$，hash function 使用 2 個，可在容納 10000 條 entry 時保持較低的誤判率。每次 Aging 迭代的時候會使用 flip filter 翻轉到另外一個 Bloom Filter，將新發現的熱區紀錄到目前的 Bloom Filter，另一個則清空或是逐步淘汰舊資訊。確保 Bloom Filter 中的內容隨時間更新，裡面的資訊為最近幾次 eviction 所發現的熱區，而不至於保留過時的訊息
- Page Table Walker: 在 Aging 流程中，kernel 對每個 process 進行 Page Table 走訪時，會先對每個 PMD 查詢 Bloom Filter (`test_bloom_filter()`)，如果一個 PMD entry 或是其他中間節點在 Bloom Filter 中存在，表示之前 eviction 該區域時裡面有不少 hot page，值得深入走訪。如果 PMD entry 不在 Bloom Filter 中，則該區域可能 hot page 的分佈非常稀疏，大部分都是冷的或是沒有映射，這時候我們就不需要再深入走訪了，可以直接跳過。通過這樣優化減少需要完整掃描的 Page Table 範圍。通過 Bloom Filter 塞選，Page Table Walker 可以跳過那些很可能是冷的區域，專注走訪那些存在較多 Accessed Bit 為 1 的 page。
- 降低 Rmap 次數: 許多 hot page 會在 Aging 時被標記並在 eviction 時 Promote，而 Bloom Filter 又確保在 Aging 階段不會對整個系統做沒必要的掃描。比起 2Q LRU 依賴 Rmap 一個一個 page 的檢查方式，MGLRU 相當於將部份工作分攤在其他階段處理
	- 在 Eviction 時藉由一次 Rmap 走訪收穫額外週邊多個 page 的 Accessed 訊息
	- 在 Aging 時利用 Eviction 得到的訊息進行掃描
	- MGLRU 在 Eviction 以及 Aging 之間就建立了一個迴路: Eviction 發現熱區 -> 更新 Bloom Filter -> Aging 根據 Bloom Filter 重點提升 hot page。
	藉由上面這種設計，減少不必要的 rmap 操作，減輕記憶體回收對 CPU 的負載。

## 測試環境
```
發行版: Fedora
Linux Kernel Version: Linux apt 6.14.0-63.fc42.x86_64
```
### 使用 MGLRU debugfs
首先建立 cgroup
```shell
$ sudo mkdir -p /sys/fs/cgroup/<name>
```
接著把指定 process 的 pid 加入到該 cgroup 中
```shell
$ echo pid | sudo tee /sys/fs/cgroup/wkgrp/cgroup.procs
```
可以使用以下指定觀察給定 process 是否加入到指定 cgroup 中
```shell
$ systemd-cgls /sys/fs/cgroup/wkgrp
```
接著執行 workload 觀察 `memory.stat`，我們可以特別選出 `anon page` 進行觀察 (`malloc` 出來的 page　屬於 `anon page`，我們頻繁的存取這一些 page　這一些 page　屬於 active_anon)
```shell
anon 4296462336
anon_thp 0
inactive_anon 36864
active_anon 4296425472
workingset_refault_anon 0
workingset_activate_anon 0
workingset_restore_anon 0
```
接著我們通過 `lru_gen` 去控制 page，我們可以通過建立新的 gen 把目前在 active_anon 擠到 inactive_anon 裡面，先查看目前 `lru_gen`　的資訊
```
memcg   133 /wkgrp  
node     0  
        14    5208406          9           1    
        15    5125504          0           0    
        16    5108911    1048932           0    
        17    5100847          0           0
```
可以看到目前 page 都在 young gen 裡面，我們通過以下指令建立新的 gen，我們原先在 young gen 的 page 應該會被擠到 old gen 裡面
```shell
echo "+ 133 0 0 0" > /sys/kernel/debug/lru_gen
```
可以看到我們的 page 進入到了 old gen 裡面
```
memcg   133 /wkgrp  
node     0  
        16    5159077        302           1    
        17    5151013    1048622           0    
        18      33125         17           0    
        19      21150          0           0
```
而 old gen 以及 oldest gen 都是屬於 inactive，我們可以在 `/sys/fs/cgroup/wkgrp/memory.stat` 中看到
```
anon 4296462336
anon_thp 0
inactive_anon 4296392704
active_anon 69632
workingset_refault_anon 0
workingset_activate_anon 0
workingset_restore_anon 0
```
我們可以做一個實驗，我們先嘗試存取資料，接著把這一些資料通過 lru_gen　進行 swapout，接著再次進行存取，概念上為以下
```
[root@apt]/home/fedora# echo "+ 133 0 0 0" > /sys/kernel/debug/lru_gen  
[root@apt]/home/fedora# echo "+ 133 0 0 0" > /sys/kernel/debug/lru_gen  
[root@apt]/home/fedora# echo "- 133 0 45 200" > /sys/kernel/debug/lru_gen
```
先把我們的 hot page 通過新建 gen 移動到 oldtest gen，接著直接回收最後一個世代，200 表示 swapiness，將其換出到 swapspace 中，我們可以通過 `cat /sys/fs/cgroup/wkgrp/memory.swap.current` 查看 swapspace 大小，以下為測試程式碼
```c
#define _GNU_SOURCE
#include <pthread.h>
#include <stdint.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include <unistd.h>

#define STEP 4096
#define NSEC(ns) ((double)(ns) / 1e9)

static uint8_t *cold;

static void touch(char *buf, size_t sz) {
  for (size_t i = 0; i < sz; i += STEP)
    buf[i]++; /* read-modify-write */
}

static uint64_t mono_ns(void) {
  struct timespec ts;
  clock_gettime(CLOCK_MONOTONIC_RAW, &ts);
  return (uint64_t)ts.tv_sec * 1e9 + ts.tv_nsec;
}

int main(int argc, char **argv) {
  size_t cold_mib = argc > 1 ? strtoull(argv[1], NULL, 10) : 4096;
  int wait_sec = argc > 2 ? atoi(argv[2]) : 30;
  size_t cold_sz = cold_mib * (1UL << 20); /* bytes */

  /* -------- malloc & fault-in -------- */
  if (posix_memalign((void **)&cold, 4096, cold_sz)) {
    perror("alloc");
    return 1;
  }
  memset(cold, 0, cold_sz);

  fprintf(stderr, "PID=%d cold=%zu MiB  wait=%d s\n", getpid(), cold_mib,
          wait_sec);

  /* -------- Phase-1: 首次觸碰 -------- */
  touch((char *)cold, cold_sz);
  fprintf(stderr, "Phase-1 done, cold pages fault-in & active\n");

  /* -------- Phase-2: 等待 mglru 操作 -------- */
  fprintf(stderr, "Sleep %d s, you may run mglru_ctl now…\n", wait_sec);
  sleep(wait_sec);

  /* -------- Phase-3: 重新觸碰 & 量時間 -------- */
  uint64_t t0 = mono_ns();
  touch((char *)cold, cold_sz);
  uint64_t t1 = mono_ns();

  double sec = NSEC(t1 - t0);
  printf("Re-access %.0f MiB took %.3f s  (%.1f MiB/s)\n", (double)cold_mib,
         sec, cold_mib / sec);
  
  return 0;
}
```
在不做任何調整的情況如下
```
PID=2696594 cold=8192 MiB  wait=10 s
Phase-1 done, cold pages fault-in & active
Sleep 10 s, you may run mglru_ctl now…
Re-access 8192 MiB took 0.046 s  (179475.2 MiB/s)
```
接著我們嘗試將 hot page swap out
```
PID=2697859 cold=8192 MiB  wait=60 s
Phase-1 done, cold pages fault-in & active
Sleep 60 s, you may run mglru_ctl now…
Re-access 8192 MiB took 2.986 s  (2743.4 MiB/s)
```
可以看到效能出現了巨大的下滑，因為這時候是從硬碟中讀取資料，從以上證明了 lru_gen 的可行性

### 使用 kprobe 觀測 funciton 行為
如果我們想要觀察 kernel function 內部行為，以 MGLRU 為例子，我們可能想要觀察具體進入 lruvec_folio 的 page 一些資訊，如他所在的 gen 等等去驗證我們的推論是否正確，對於這些情況，我們有許多方式可以實現對 kernel function 的觀察，以下分成靜態與動態的方式

### 靜態追蹤
Tracepoint: 為 Linux Kernel 中一種靜態插樁 (static instrumentation) 機制，可以在原始碼中函式部份預先定義 hook points，用來監控與追蹤 kernel 的行為。Tracepoint 在編譯時會嵌入到 kernel 中，可以使用 `sudo perf list tracepoint` 查看可用的 tracepoint。

不過缺點也很明顯，對於沒有預先定義 tracepoint 的 function，我們會需要修改並重新編譯 kernel。

### 動態追蹤
kprobe: 可以動態的在 kernel funciton 中任意位置插入探針，可以無須重新編譯 kernel，不需要修改 kernel code。

對於已經在 kallsym 中的 function，我們可以很輕鬆直接使用 symbol name 去 kprobe，但是對於不在 kallsym 的 function，如一些被 compiler 優化掉的 symbol 或是 inline，我們會需要得到該 function 的 function address，具體作法為我們需要 kernel-debuginfo 配合 DWARF 去得出該 function 的地址，這點是從 `perf probe` 中意外發現的，發現 `perf probe -L <function_name>` 可以印出 function 的 source code，於是我嘗試以下
```shell
$ sudo perf probe -L try_to_inc_max_seq                                       
Failed to find the path for the kernel: No such file or directory
  Error: Failed to show lines.
```
出現以上錯誤，嘗試使用 `strace` 對 `perf probe` 進行追蹤後發現是依賴於 debug-info，以下為 `strace perf probe -L <function_name>` 的輸出
```
- openat(AT_FDCWD, "/usr/lib/debug/usr/lib/debug/lib/modules/6.14.0-63.fc42.x86_64/vmlinux.debug", O_RDONLY)
- openat(AT_FDCWD, "/usr/lib/debug/usr/lib/debug/lib/modules/6.14.0-63.fc42.x86_64/vmlinux", O_RDONLY)
- openat(AT_FDCWD, "/usr/lib/debug/lib/modules/6.14.0-63.fc42.x86_64/.debug/vmlinux", O_RDONLY)
- openat(AT_FDCWD, "/usr/lib/debug/.build-id/a9/1e7e0ceeb1cc106e37bf89b80a69b39844a559.debug", O_RDONLY)
```

以 fedora 為例子，通過以下指令我們可以得到 kernel-debuginfo
```shell
$ sudo dnf --enablerepo=fedora-debuginfo,updates-debuginfo install kernel-debuginfo
```

接著再次使用 `strace perf probe -L <function_name>` 即可正常執行，並且還能夠列出不在 System.map 以及 kallsyms 裡面的 function 資訊，於是我想要通過 debuginfo 去得到那些被優化掉的 function 的記憶體地址

debuginfo 會位於 `/usr/lib/debug/lib/modules/$(uname -r)/vmlinux` 底下，接著我們可以通過以下腳本去得到任一 function 的記憶體地址
```py
import subprocess
import sys
import re
import os

def get_function_start_address(function_name):
    kernel_release = os.uname().release
    vmlinux_path = f"/usr/lib/debug/lib/modules/{kernel_release}/vmlinux"
    print(vmlinux_path)

    # Using gdb read vmlinux and get function address
    cmd = [
        "gdb",
        "-batch",
        "-ex",
        f"info line {function_name}",
        vmlinux_path
    ]

    try:
        output = subprocess.check_output(cmd, stderr=subprocess.STDOUT, text=True)
    except subprocess.CalledProcessError as e:
        print("GDB execution failed:")
        print(e.output)
        return None

    match = re.search(r"starts at address ([\w]+)", output)
    if match:
        return match.group(1)
    else:
        print("Failed to find start address.")
        return None

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(f"Usage: sudo python3 {sys.argv[0]} <function_name>")
        sys.exit(1)

    function = sys.argv[1]
    addr = get_function_start_address(function)
    if addr:
        print(f"{function} starts at address: {addr}")
```
有了 function address 之後，我們就可以用 kprobe 去探測並觀察其內部資料結構資訊。

使用 kprobe 有以下兩種方式
- 使用 `bpftrace`
- 使用 kernel module

#### 使用 kernel module
以 `try_to_shrink_lruvec` 為例子，如果我們想要得知 `lruvec` 裡面具體 page 的情況或是 gen 等等，我們可以通過以下 kprobe pre handler 存取 function 內的變數
```c
static int handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    struct lruvec *lruvec = NULL;
    struct lru_gen_folio *lrugen = NULL;
    int gen, type, zone, page_refcount;

    lruvec = (struct lruvec *)regs->di;

    if (!lruvec) {
        pr_warn("lruvec is NULL\n");
        return 0;
    }

    lrugen = &lruvec->lrugen;
    if (!lrugen) {
        pr_warn("lrugen is NULL\n");
        return 0;
    }

    pr_info("[KPROBE] try_to_shrink_lruvec called\n");
    pr_info("[KPROBE] max_seq: %lu\n", lrugen->max_seq);

    for (gen = 0; gen < MAX_NR_GENS; gen++) {
        for (type = 0; type < ANON_AND_FILE; type++) {
            for (zone = 0; zone < MAX_NR_ZONES; zone++) {
                struct list_head *head;
                struct folio *folio;
                struct page *page;
                long count = 0;

                head = &lrugen->folios[gen][type][zone];

                if (!head || list_empty(head))
                    continue;

                pr_info("Generation %d, type %s, zone %d:\n", gen,
                        (type == 0 ? "anon" : "file"), zone);

                list_for_each_entry(folio, head, lru) {
                    if (!folio) {
                        pr_warn("NULL folio in list\n");
                        break;
                    }
                    
                    page = &folio->page;
                    page_refcount = page_ref_count(page);
                    
                    unsigned long pfn = page_to_pfn(page);
                    unsigned long long page_phys_addr = PFN_PHYS(pfn);
                    bool dirty = PageDirty(page);
                    bool writeback = PageWriteback(page);
                    
                    pr_info("(folio: %px) (page addr: %px) (page phys addr: %px) (page ref count: %d)\n", folio, page, page_phys_addr, page_refcount);
                }
            }
        }
    }

    return 0;
}
```
根據 x86 calling convention 我們知道第一個參數位於 `rdi` 暫存器中，接著按照 MGLRU 關鍵資料結構，由 `lruvec` 存取 `lrugen`，接著依照 `gen`, `type`, `zone` 走訪 folio list 便可以得到 page 資訊。

接著指定我們要 hook 的 function 以及對應的 pre handler 便完成註冊
```c
static int __init mglru_monitor_init(void)
{
    int ret;

    kp.symbol_name = "try_to_shrink_lruvec";
    kp.pre_handler = handler_pre;

    ret = register_kprobe(&kp);
    if (ret < 0) {
        pr_err("register_kprobe failed, returned %d\n", ret);
        return ret;
    }

    pr_info("MGLRU monitor module loaded.\n");
    return 0;
}
```
如果我們要 hook 的 function 的 symbol 在 System.map 中查詢不到，而是使用 Debuginfo 得到的 raw address，則需要使用以下
```c
static int __init kprobe_stack_init(void)
{
    kp.addr = (kprobe_opcode_t *)FUNCTION_ADDRESS;
    kp.pre_handler = handler_pre;

    ret = register_kprobe(&kp);
    ...
}
```

成功執行後我們便可以得到具體進入到 `try_to_shrink_lruvec()` 的 page 相關資訊，得到以下輸出
```shell
$ sudo dmesg
[29971.000568] (folio: ffffea001480e480) (page addr: ffffea001480e480) (page phys addr: 0000000520392000) (page ref count: 1)
[29971.000569] (folio: ffffea001480e440) (page addr: ffffea001480e440) (page phys addr: 0000000520391000) (page ref count: 1)
[29971.000570] (folio: ffffea001480e400) (page addr: ffffea001480e400) (page phys addr: 0000000520390000) (page ref count: 1)
[29971.000571] (folio: ffffea001480e3c0) (page addr: ffffea001480e3c0) (page phys addr: 000000052038f000) (page ref count: 1)
[29971.000572] (folio: ffffea001480e380) (page addr: ffffea001480e380) (page phys addr: 000000052038e000) (page ref count: 1)
[29971.000574] (folio: ffffea001480e340) (page addr: ffffea001480e340) (page phys addr: 000000052038d000) (page ref count: 1)
[29971.000575] (folio: ffffea001480e300) (page addr: ffffea001480e300) (page phys addr: 000000052038c000) (page ref count: 1)
[29971.000576] (folio: ffffea001480e2c0) (page addr: ffffea001480e2c0) (page phys addr: 000000052038b000) (page ref count: 1)
[29971.000577] (folio: ffffea001480e280) (page addr: ffffea001480e280) (page phys addr: 000000052038a000) (page ref count: 1)
[29971.000578] (folio: ffffea001480e240) (page addr: ffffea001480e240) (page phys addr: 0000000520389000) (page ref count: 1)
...
```

#### 使用 bpftrace
可以使用以下 oneline 指令查看給定 function 傳遞的參數
```shell
sudo bpftrace -e 'kprobe:try_to_shrink_lruvec { $lruvec = (struct lruvec *)arg0; printf("[KPROBE] try_to_shrink_lruvec called, lruvec: %px, max_seq: %lu\n", $lruvec, $lruvec->lrugen.max_seq); }'
```
或是直接使用 raw function address (如果 function 不在 kernel symbol table，可以使用 `sudo bpftrace -l` 查詢)
```shell
sudo bpftrace -e 'kprobe:0xffffffff81684c0c { printf("arg0=%lx\n", arg0);}'
```

或是將以下檔案儲存成 .bt，使用 `sudo bpftrace` 執行
```c
#!/usr/bin/env bpftrace
BEGIN 
{
    printf("Trace try_to_shrink_lruvec function\n");
    printf("Press Ctrl+C to stop\n");
}

kprobe:try_to_shrink_lruvec
{
    $lruvec = (struct lruvec *)arg0;
    
    if ($lruvec == 0) {
        printf("[Warning]: lruvec is NULL\n");
        return;
    }
    

    printf("[KPROBE] try_to_shrink_lruvec been called\n");
    printf("[KPROBE] lruvec address: %px\n", $lruvec);
    printf("[KPROBE] timestramp: %lu\n", nsecs);
    printf("[KPROBE] process PID: %d, process_name: %s\n", pid, comm);
    printf("[KPROBE] max_seq: %lu\n", $lruvec->lrugen.max_seq);
    printf("[KPROBE] call_count: %d\n", ++@call_count);
    printf("---\n");
}

END
{
    printf("total call_count: %d\n", @call_count);
    printf("End\n");
}
```
可以看到上面是一個簡單的測試範例，如果我們要對 lruvec 裡面的成員進行遍歷等等，使用 bpftrace 會非常難去實現，因為無法使用變數作為陣列的索引且一些 kernel 中的巨集也無法使用。

如果只是查看傳遞參數等等，可以快速的使用 bpftrace 進行實現，需要進一步對參數進行解析則使用 Kernel Module 進行實現。
## 結論
- 解釋了 MGLRU 改進以及大致運作流程
- 運用 Flamegraph/dot-graph 對 kernel 子系統進行分析
- 運用 kprobe 對 kernel 某個 function 進行精細分析

## Reference
- [MGLRU - Yu Zhao](https://www.youtube.com/watch?v=9HvJfN21H9Y&ab_channel=TheLinuxFoundation)
- [Multi-Gen LRU](https://www.kernel.org/doc/html/v6.1/mm/multigen_lru.html)
- [Multi-Gen LRU Kernel.org](https://www.kernel.org/doc/html/v6.1/mm/multigen_lru.html)
- [brendangregg/FlameGraph](https://github.com/brendangregg/FlameGraph)
- [jrfonseca/gprof2dot](https://github.com/jrfonseca/gprof2dot)
- [Linux Kprobe](https://docs.kernel.org/trace/kprobes.html)
- [Kernel Recipes 2017 - Perf in Netflix - Brendan Gregg](https://www.youtube.com/watch?v=UVM3WX8Lq2k&ab_channel=KernelRecipes)