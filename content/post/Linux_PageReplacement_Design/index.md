---
title: "Linux_PageReplacement_Design"
description: 
date: 2025-09-02T11:51:13+08:00
image: 
math: 
license: 
categories:
    - Linux Kernel 
tags: 
    - Linux Kernel
    - Linux Kernel MM
hidden: false
comments: false
draft: false
---
在 Linux Kernel 2.6.28 合併了 Split LRU 的機制，並在後續版本中持續改進以及修復，直到 2.6.32 性能才變得穩定。而對於 tracking of recently reclaimed pages 還沒被整合，而是通過實作一種方法來動態調整 LRU list 的大小。

LRU，為最常見的 page replacement 機制，在傳統實現中會將 user space 的 page 和 kernel space 的 page 都放在同一個 LRU list 中。這個做法在部分情況下會導致非最佳化的 page replacement 發生，特別是當 user space 的程式和 kernel space 爭搶記憶體時會影響到性能。


## 2.6.27 之前 page replacement 問題
在 2.6.27 更早版本的 page replacement 存在兩大主要問題，這些問題在大型記憶體系統，如數 10 GB 甚至於上 100 GB 的系統中尤其嚴重。

- kernel 驅逐的 page 並非最佳的 page 甚至於不該驅逐的 page，導致性能下降
	- kernel 在回收記憶體時，可能會驅逐那些不該回收的 page，而保留應該被驅逐的 page
	- 這會導致 cache thrashing 的問題，因為應用程式可能會頻繁重新載入被錯誤回收的 page
- 真正要被回收的 page cache 可能會被隱藏在大量 [[anon memory]] 或是 anon page 後面
	- Linux 使用 LRU list 管理記憶體，但 list 沒有區分 anon page 以及 page cache ( 用於硬碟 I/O 的 page )
	- 在沒有 Split LRU 之前，這些 page 被統一放在 LRU list 中，導致 kernel 在尋找可釋放的 page，可能會優先回收最容易找到的 page，而非最應該被回收的 page
- kernel 會掃描到不應該被回收的 page，導致 CPU 負載過高
	- 在大型記憶系系統中，低效的 LRU 掃描只會增加 CPU 使用率
	- 但在超大型記憶體系統中，這種掃描不但會導致 CPU 負載巨幅增加，甚至出現嚴重的鎖競爭問題，影響系統效能
	- kernel 不斷重複掃描 LRU list，其中許多的 page 根本不應該回收
	- 不僅浪費 CPU 資源，還可能導致記憶體管理的 lock 競爭，拖垮系統
	  
例如在一個有 80GB anon page 和 10GB page cache，沒有 swap 的系統中，在舊的 LRU 設計中
- kernel 會試圖去釋放 page cache (因為這一些 page 比較容易重新載入)
- 但是由於 page cache 被 80GB 的 anon page 擋住了，因此 kernel 需要不斷掃描 80GB 的 anon page，試圖找到可釋放的 page，結果就是 kernel 做很多無效掃描

在後續使用 Split LRU 嘗試解決上面問題，將 LRU 分成兩類
- File-backed pages: 來自 disk 的 page，如 `.txt, .so, .bin` 等等
- Anon pages: 應用程式的私有記憶體，如 Heap, Stack

當記憶體壓力大時，kernel 可以優先回收 File-backed pages，不需要掃描大量的 Anon pages。

## High-Level Overview
Linux Kernel 的 VM 管理需要最小化掃描成本，確保 page replacement 在 best 和 worst 都可以高效率執行。Split LRU 的設計正式希望能夠解決這一些問題。

TODO: page evict vs page reclaim vs page free vs page replacement
### 核心設計概念
- 最小化掃描，避免無效的 page replacement
	- 掃描成本隨著記憶體增長而增長，在面臨記憶體壓力下，kernel 應該只掃描可能需要釋放的 page，而不是整個 LRU list
- 大多數的 [[Page Churn]] 來自於讀取大型檔案。在大多數 server 的 workload 下，最主要記憶體占用來自於讀取大量檔案，例如 Database Systems, Log Processing, High-Frequency Trading, ML/AI Training
	- 解決方案為優先驅逐這一些 file-backed pages，因為他們可以從硬碟重新載入，代價較低，避免掃描 anon pages，這一些屬於應用程式的私有記憶體，如 stack, heap 資料不能隨意丟棄
- 當記憶體不足時，才驅逐其他 page
	- 當記憶體充足時，僅驅逐 file-backed pages。如果驅逐 anon pages，可能會有 swap，swap 對效能影響大
- 優先考慮 FileSystem I/O，而非 Swap I/O
	- Swap 會有額外的硬碟寫入，影響性能
- 使用 Reference Count 和 Refault Data 平衡檔案快取以及 anon page
	- Reference Count: 記錄一個 page 最近被 access 的頻率
	- Refault Data: 如果一個 page 在被驅逐之後又很快被載入，代表他應該被保留
	- 根據這些資料動態調整 LRU，確保系統平衡
		- 如果檔案快取 Refault Data 次數多，代表要保留更多檔案快取
		- 如果 anon page Refault Data 次數多，代表要減少檔案快取，保留更多應用程式記憶體
- 關於 Best Case 和 Worst Case 的最佳化
	- 正常情況: 優先驅逐 file-backed pages，減少沒必要的 page replacement
	- 即便發生最壞情況，也不會發生因為過度掃描而發生 Lock Contention
## File Cache
File Cache 主要負責儲存來自於硬碟，NFS, CIFS 的 data 或是 metadata，Linux Kernel 在處理這一些 File Cache 需要考慮以下幾點

- 檔案系統大小遠大於記憶體
	- 對於 Server 動則 TB 以上，記憶體比起硬碟小的許多。Kernel 需要動態管理 File Cache，確保最常用的 File Cache 儲存在記憶體中，而不是浪費記憶體儲存了一堆 Cold data
- 檔案存取特性
	- 大多數檔案被存取的機率很低: 這一些資料通常是一次性讀取，如影片串流，備份與恢復，大規模資料處理，這些資料讀取完後不會重複存取
	- 少量資料會被頻繁存取: 例如 database index, 應用程式 config, log 等等，因此這一些應該優先保留在記憶體中，避免 Page Fault  發生
- 使用 Used-Once 的 page replacement
	- 對於 Used-once page replacement algorithm 的實現，Linux Kernel 使用兩條 LRU list 來對這一些 File Cache 進行管理
		- Inactive File List: 新載入的 File Cache 會放入 Inactive File List，如果只被存取一次，當它到達 List tail 的時候回收
		- Active File List: 當一個 page 被多次存取時，會從 Inactive List 遷移到 Active List，確保常用的 page 不會過早被驅逐
- 關於管理 Active List 與 Inactive List
	- 如果 Inactive List 變太小，則將 Active List 中移動一些到 Inactive List: 當新的 File Cache 載入時，確保他們還有機會進入到 Active List
	- 如果 Active List 大於 Inactive List / 2 的大小，則會將 Active List 部份元素開始降級，避免 hot 資料無限制的佔據記憶體
- 如何確保 hot 資料不會被驅逐
	- 只要 Active List 佔據的記憶體大小不要超過 File Cache 總容量的一半，那麼這一些頻繁存取的 hot 資料將不會被 kernel 的 page replacement 逐出記憶體
- 關於 Active List 大小優化
	- 可能可以考慮 Recently Evicted Pages 因素來調整 Active List 的大小

## Anonymous Memory
為 Process 的私有記憶體空間，Shared Memory Segments 以及 tmpfs，這一些記憶體區段與 File Cache 不同，他們不直接對應到硬碟上的檔案，而是僅存在於 RAM 或是 swap 空間

- 相較於 File Cache，數量較少: anon memory 通常和系統總記憶體大小相近，不像 File Cache 存在可能大於 RAM 容量的情況
- 由於 anon memory 主要存放程式的 stack, heap 等等，會比 File Cache 更加頻繁的被存取
- 通常不會被 swap out，除非記憶體嚴重不足。如果 kernel 頻繁掃描 anon memory，會導致效能損失以及記憶體管理效率下降

關於 Anon Memory 的管理其中一個方法為 SEQ replacement (Sequential Replacement)
- 所有 anon page 一開始都是被引用的
- 無論 page 是否被引用，都會定從 Active List 移動到 Inactive 
- 在 Inactive 的 page 被存取會躍遷到 Active
- 如果在 Inactive tail 的 page 還是沒有被存取，則會被 swap out

## Non reclaimable
- 被 `mlock()` 鎖定的 page
- 使用 `shmget()` 分配的共享記憶體，如果加上 `mlock()`，則不會被回收
- `ramfs, ramdisk` 不會被換到 swap，也不會受到 LRU 回收機制影響，標記成 Non reclaimable

### Recently Evicted/ Nonresident
輔助 page fault 發生時的置換策略，以及調整 LRU List 大小。希望提供額外訊息，幫助 VM 更加好的進行 page replacement。不會儲存真正回收的 page，而是紀錄哪一些 page 最近被回收過。當發生 page fault 時，知道該 page 是否最近被回收過，進而決定是否優先分配回來，這個資訊可以避免 Thrashing 情況的發生。

調整 LRU List 大小，例如當最近回收 page 比較多時，可以將 Active List 增大，減少頻繁的 swap out。

因為我們儲存的是 meta data，考慮使用 radix tree 實做

