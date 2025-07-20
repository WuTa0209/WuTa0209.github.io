---
title: "SIEVE_is_Simpler_than_LRU_an_Efficient_Turn Key_Eviction_Algorithm_for_Web_Cache"
description: 
date: 2025-07-20T17:23:27+08:00
image: 
math: 
license: 
categories:
    - Paper Note
hidden: false
comments: false
draft: true
---
- [repo](https://github.com/cacheMon/NSDI24-SIEVE?tab=readme-ov-file#traces)
- [home page](https://cachemon.github.io/SIEVE-website/)
方法論創新的論文
## 摘要

隨著 Web 技術不斷發展，cache 相關技術成為如資料庫，分散資料中心等等不可獲缺的技術。其中 cache 相關的演算法尤其重要，對於 Web 環境而言，cache 演算法主要聚焦於如何降低資料存取延遲，減少網路頻寬消耗以及提高系統的吞吐量。cache 演算法主要的目標為選出哪一些物件可以被逐出 cache 用來騰出 cache 空間，cache 演算法很大程度會影響 cache 命中率以及系統的吞吐量。但是對於大部分 cache 演算法，只能在吞吐量以及演算法複雜度之間進行取捨，通常高吞吐量或是高效能的近期 cache 演算法都有較高的演算法複雜性，而這一些高複雜性的演算法要投入實際環境中往往是十分困難的，且這一些算法因為其複雜性，會需要更高的維護成本，減少程式錯誤或是安全性問題。

在這一篇論文中提出了一個兼顧效能以及複雜度的演算法 SIEVE，SIEVE 有以下優勢
- 簡單性
	- 比 LRU 還要簡單，易於部屬以及維護
	- 在五個實際的 cache Library 實現 SIEVE，平均只需要大約 20 行的程式碼變更
- 高效能: SIEVE 測試 1559 條來自 7 個不同 dataset 的 cache 存取路徑，與 9 個 cache 演算法進行比較，發現實現了更低的 cache 未命中率
	- 部份 workload 比 ARC 低了 63.2% 的未命中率
	- 在超過 45% 的 cache 存取路徑中表現優於其他所有演算法
- 可擴展性:
	- 不需要在 cache 命中的時候進行 lock 的操作，提高平行度，在並行環境中提高吞吐量
	- 在 Cachelib 中，SIEVE 吞吐量是針對 16 thread 優化的 LRU 的兩倍
- 靈活性:
	- SIEVE 不僅僅是一個獨立的 cache Eviction 演算法，還以作為構建其他 cache Eviction 演算法的基本部份，如替換 ARC 或是 2Q 中 LRU 的部份，進一步提高效能

關於 SIEVE 實現:
SIEVE 設計是使用 FIFO queue 以及一個稱為 hand 的 pointer 進行實現
- 'r' bit 對於每一個 cache 的物件都有一個 bit 用於標記，表示該物件是否有被存取
- 當 cache 命中，'r' bit 設置為 1
- 當 cache 未命中，SIEVE 檢查 hand 指向的物件，如果其 'r' bit 為 1，保留該物件並重設該 bit，否則將 hand 指向的物件 Eviction 出 cache 並移動 hand
- 對於新加入的物件，始終插入到 FIFO queue.head
SIEVE 通過 Lazy promotion (惰性躍遷) 以及 Quick Demotion (快速降級) 有效的保留 hot 的物件並 Quick Demotion 不受歡迎的物件，實現高效率的 cache 管理

論文貢獻:
- 提出 SIEVE 演算法，針對 web caches 場景具有十分高的效能，且演算法結構簡單
- 證明 SIEVE 簡單性，對於常用的 cache library，平均只需要修改大約 20 行程式碼即可實現
- 從 7 個 datasets 中，1559 條 cache access route，發現超過 45% 的資料中，性能好過於所有目前常見的先進 cache/page replacement 相關演算法
- 在 cachelib (SOSP 23', 同一個團隊開發的) 專案中展示了 SIEVE 的可擴展性，分別在 single thread 以及 multi thread 的情境下，吞吐量比優化過的 LRU 高出 17% 和 125%
- 展示 SIEVE 具有隨插即用的特性，可以替換掉如 ARC, 2Q, LeCaR 中 LRU 演算法的部分
#### 什麼是 cache Library/cache system
cache Library 為用於實現 cache 機制的軟體工具或是程式庫，提供資料儲存以及資料管理的功能，目標是為了加速資料存取速度並提高系統效能。通常 cache Library 會使用類似於 key-value 的方式去管理記憶體的快取，並根據目前的需求或是 workload 來決定要使用什麼樣的 cache Eviction 演算法，如 LRU, FIFO 等等來管理有限的 cache 資源。

cache Library 有以下用途
- 加速資料存取速度: 將最近存取的資料放在 cache 或是其他快速存取的空間，如記憶體等等，從而減少從輔助記憶體，如硬碟或是網路等等讀取資料的時間
- 減少後端負擔: 好的 cache 可以減少後端資料庫查詢的頻率，從而降低後端伺服器負擔
- 減少頻寬消耗: 好的 cache 可以減少跨網路的傳輸量，減少傳輸成本
- 提高吞吐量: 對於高並行的情況，可以有更高的吞吐量

常見 cache Library 類型
- key-value 類型: 將資料儲存成 key-value，通常用於快速查詢資料，如 config 等等，常見使用此類型的為 Memcached, Redis, Cachelib
- CDN cache: 將圖片，影片，檔案放在離使用者最近的節點，減少延遲提高存取速度，如 Cloudflare, Akamai
- Page cache: 將網頁或是靜態內容進行緩衝，避免每次動態產生，可用於網站優化或是內容渲染加速
- 應用程式 cache: 緩衝一些中間資料或是計算邏輯，提高速度，如機器學習等等
以 groupcache 而言，groupcache 概念上是 golang 的 library，裡面有 client, server 以及一些快取機制，如果 client 向遠端要求資料，則可以使用 groupcache 建立中間層進行快取。請求成功直接回傳資料，請求失敗則向遠端伺服器要求數據並回傳。

Netflix 在 2021 年使用 18000 台 server，大約緩衝了 14PB 的資料
Twitter 在 2020 使用數百個 claster，消耗數百 TB 的 DRAM 以及數十萬個 CPU core 實現 cache 機制
## 簡介
目前雖然有許多演算法能提高效率，降低 cache miss，例如 ARC, SLRU, 2Q, MQ, LHD, CACHEUS, LRB, GL-Cache 等等，但這一些演算法都有顯著的複雜性
- ARC, SLRU, 2Q, MQ 使用多個 LRU queue 來提高效率，但是這增加了系統複雜性
- LHD, CACHEUS, LRB, GL-Cache: 使用機器學習的技術決定要驅逐的目標，但這一些演算法的查詢和設計系統更加的複雜，且通常需要給定參數進行調整

為什麼要追求算法簡單？
- 易於理解以及維護: 邏輯簡單的算法更加容易進行除錯以及部屬和測試
- 高效能: 簡單邏輯通常可以減少計算時間或是 lock 開銷等等，提供更好的吞吐量
- 可擴展性: 簡單的算法可以更好的適應多 thread 或是多核心環境，提高系統整理性能
目前如 ATS, Varnish, Nginx, Regis, groupcache 這一些 cache system，只使用 LRU 以及 FIO 而已，因為他們設計更加簡單且容易實做

關於 SIEVE 算法設計，關鍵部份是改進了 FIFO-Reinsertion，基於這個改進提出了 SIEVE。與傳統的 FIFO-Reinsertion 不同，SIEVE 不會將存取過的物件移動到 FIFO queue 的 head，而是將他們保留在原本的位置

SIEVE 整體設計理念非常簡單，新的物件依然會被插入到 queue head，通過一個 hand 指標指向到下一個可能會被驅逐的物件

### 關於 FIFO-Reinsertion
使用 FIFO-Reinsertion 的邏輯和 Clock 演算法非常的像，以下為 psudocode
```
obj = tail 
while obj.visited: 
	obj.visited = false 
	prev = obj.prev 
	move obj to head 
	obj = prev
```
從 queue 的尾端開始，檢查 obj.visited，如果物件被存取過，則將 visited 設置為 false，並將該物件重新移動到 queue 的頭部。接著繼續檢查前一個物件，概念上就是移動 pointer，直到找到一個未被存取的物件為止

當一個物件被存取，會重新被插入到 queue 的頭部，表示最近被使用。
保留存取過得物件，優先保留在 cache 中

接著是 SIEVE 的邏輯
```
obj = hand  
while obj.visited:  
    obj.visited = false  
    # skip obj, do nothing  
    obj = obj.prev  
hand = obj.prev  

```
從 hand 指向的物件開始，檢查 obj.visited，如果被存取過，將 visited 設為 false，但是不移動該物件在 queue 中的位置。接著跳過該物件，並繼續檢查該物件的前一個物件，也就是移動 hand，直到找到一個未被存取的物件為止

當一個物件被存取時，SIEVE 不會將該物件移動到 queue 的頭部，而是將其保留在原始位置
被存取的物件只會重置 visited，但該物件在 queue 裡面的位置不變

FIFO-reinsertion 優點是被存取的物件會重新移動到 queue 頭部，這可以增加物件被保留的機會，但缺點是每次檢查到已經存取過得物件，都需要將其移動到 queue 頭部，增加操作開銷。且對於並行情況，移動物件的操作可能需要修改 queue，queue  是一個全域資料結構，這會需要使用 lock 等等，降低系統並行度，影響吞吐量。

而 SIEVE 作法為不會移動存取的物件，因此不需要對 queue 進行 lock 保護。SIEVE 會跳過被存取過得物件，少了移動物件的操作，而因為少了這個操作，使的 SIEVE 可以快速的將冷門物件從 queue 中進行驅逐。但 SIEVE 的缺點為對於 hot的資料，無法像 clock 那樣保留在 cache 中較長的時間，可能會過早的就將 hot 資料驅逐。

所以這裡有一個選擇問題，在 SIEVE 理論上 cache miss 比 clock 還要常出現，那麼是 cache miss 的成本比較高，還是去使用 lock 維護 queue 移動資料來的成本高 (考慮在多核心情況，可能多個核心在競爭 queue 或是 lock 等等)。

接著是對於 workload 適應性的問題，本質上 Clock 演算法是一個 LRU 的近似解，檢查物件，如果物件被存取過把他移動到 queue 的頭部，最後被驅逐淘汰的物件為最近最少使用的物件，移動的操作保證進行被存取過得資料被保留。對於有存取時間局部性的資料存取表現的會比較好。

而當存取模式改變，如熱點改變，其他數據成為熱點，Clock 也可以快速的適應，把 hot 的資料移動到 queue 的頭部

而 SIEVE 對於 workload 適應性的問題，SIEVE 跳過被存取過得物件，而不是將存取過的物件重新移動到 queue 頭部，這樣某種程度熱的資料和冷的資料有同樣淘汰優先級，且正如上面所說，熱的資料可能很早就被驅逐 (隱含時間局部性的問題，熱的資料不會被優先保留)。且對於存取模式改變，不像是 clock 通過調整物件在 queue 的位置，而是按照固定邏輯進行淘汰。

## 關於 SIEVE 應用
在論文中提及應用到以下
- [mnemonist](https://github.com/cacheMon/mnemonist) (2.3k start) 使用 JS 寫的 cache system，裡面包含許多常用資料結構，heap, linked list 等等以及 LRUCache, LRUMap，SIEVE 修改了 LRUCache 中 LRUCache.prototype.evict 的邏輯，改成 none FIFO-reinsertion 的邏輯 (備註: 對 mnemonist 進行一些 code review 之後，發現其實 mnemonist 還沒將 none FIFO-reinsertion 合併到 master，還沒有合併的原因是因為將 SIEVE 應用在 LRUMap 中 cache 命中率低了很多，原因還在調查中 2/18)
- lru-rs 使用 rust 寫的 cache system，實作上為使用 rust 實作的 LRU cache，主要替換部分為 replace_or_create_node 中的邏輯，這個函式目標是當 cache 快滿的時候選擇一個節點進行替換，也就是 replacement 的概念，對於 replacement 的關鍵部分選擇使用 none FIFO-reinsertion 的邏輯 ( 備註: 同樣尚未被 merge 進入到 master 中)
- groupcache: 為使用 golang 實作的 cache system (備註: 不過 SIEVE 目前僅僅出現在 issue 相關討論，並沒有合併到 master)
-  Cachelib (SOSP 23'): 在 Cachelib 中，SIEVE 吞吐量是針對 16 thread 優化的 LRU 的兩倍
## 背景知識
### 關於 Web cache
Web cache 是現代網路中重要的基礎建設，主要用來降低資料存取延遲以及減少網路頻寬使用，實現 Web cache 可以使用 cache system/cache library 來實作。

### 評估 cache 存取效能的指標
通常我們可以使用效率 (Efficiency) 以及吞吐量 (Throughput) 進行評估
#### 效率
衡量 cache 在有限空間中儲存和提供資料的能力，常用的指標為物件未命中率，Byte 未命中率，未命中率越低，表示 cache 能夠直接服務 client 更多的請求，減少後端的負擔，存取延遲以及網路頻寬成本

#### 吞吐量
意義為 cache 每一秒可以處理多少請求，在 Web 領域中以 QPS (Queries Per Seconds) 為單位進行表示。

cache 的目標之一為快速的處理請求並提高應用程式的 scalability，scalability 指的是衡量 multi-thread 存取 cache 吞吐量增長的情況，當核心數越來越多時，cache 就需要高效率的利用 cache 資源，更好的吞吐量以及更好的擴展性能可以顯著的提升應用程式在高並行環境下的穩定性以及性能。

### 關於 Web cache 面臨的 workload
Web cache 面臨的 workload 通常具有 Power-law Distribution，也就是 Zipfian Distribution，概念上就是少數 hot 的物件佔據了大部分的請求，以下詳細解釋

- Power-law Distribution
	 在 Web cache 的存取模式中，第 $i$ 個受歡迎的物件的存取頻率和 $1/i^{\alpha}$ 成正比，其中 $i$ 表示物件受歡迎程度的排名，也就是排名 1 的物件受到存取次數最多。$\alpha$ 表示傾斜程度，決定物件存取分布的傾斜程度，$\alpha$ 越大，表示存取序列分布躍偏向於少數 hot 的物件，大量的請求集中在這一些 hot 的物件上
	 
	 根據不同的研究以及觀察，Web cache 的 $\alpha$ 差異十分的巨大，原因可以歸因是因為不同的 workload，如使用 Web Proxy Cache 或是 In-Memory Key-Value Cache，這一些 cache 的 workload 性質不同，導致 $\alpha$ 的分佈有所差異。另一個是 cache 層級問題，在多層級的 cache 架構中，如 Proxy Cache 或是 CDN cache，這些通常屬於二級 cache 或是三級 cache，對於較低等級的 cache，物件請求頻率以及傾斜程度可能和比較高級的 cache 有所不同。對於 web 應用程式中，最受歡迎的物件可能會接收更大量的請求，導致存取分布傾向這一些熱門的物件，如熱門貼文，影片或是圖片
 - Web Cache 存取特性
	Web Cache 較少出現循環存取模式，也就是同一批 data block 被多次重覆存取，另一個則是掃描模式，也就是一段時間內，存取 data block 按照記憶體地址範圍連續進行掃描，如果存在這一些模式，則我們需要專注在高效率處理固定 data block 的存取行為。這一些場景會出現在如 MySQL 的 cache pool，用來加速 page 資料的讀取和寫入。以及 file system 的 cache，如作業系統中的 page cache，用來幫助快速存取檔案資料。

### 關於 cache eviction policies
cache 淘汰演算法的主要目標是在有限大小的 cache 空間內選擇保留哪一些物件以最大化性能以及效率。儘管這方面有許多研究，但現代許多算法為了尋求更高的效能，複雜度也日益上升，而這一些複雜度的上升會使得這一些演算法要投入實際生產環境變得十分的困難，對於現代 cache 淘汰演算法，是否具備在實際生產環境中有可行性是一個重要的考量。

以下為關於 cache 淘汰演算法複雜性發展
- 1990 年代: 使用 multi-queue 或是維護多個資料結構去評估存取狀態，如 LRU-K, SLRU
- 2000 年代: 引入自適應資料結構或是更加複雜的資料結構去評估時間或是存取頻率，如 ARC, CAR
- 2010~2020 年代: 引入機器學習的技術，根據資料及決定要驅逐出 cache 的物件，如 CACHEUS, LRB, HALP
上面這一些演算法儘管在特定的 workload 底下表現很好，但整體效能提升十分有限，且因為其複雜性，要投入實際生產環境十分的困難

#### 演算法複雜性提升會有什麼問題?
- 除錯困難: 複雜邏輯增加除錯的困難度，例如在 LIRS 實作中，作者以及其團隊發現在許多開源模擬器中都有各自實作上的錯誤
- 效率異常 (Belady's Anomaly): 在某一些演算法，如 LIRS 或是 ARC 上面在某一些 workload 底下，隨著 cache 大小增加，為命中率反而上升，FIFO 也會出現類似這樣的問題
- 吞吐量下降以及可擴展性下降: 複雜演算法需要更長的計算時間或是某一些 critical section，減少系統的吞吐量以及 multi-thread 之下的可擴展性。另一個是複雜的演算法的 meta-data 通常比簡單演算法的 meta-data 來的大的許多，複雜演算法需要考慮更多的資訊，因此這一點無可避免，如 CACHEUS 演算法的 meta-data 是 LRU 的 3.3 倍
- 顯式或是隱式的參數調整問題: 基於機器學習的演算法需要調整許多參數，即便演算法沒有明顯的參數，如 LIRS，隱性的參數，如 ghost queue 的大小也會大大的對效能產生影響
#### 簡單淘汰機制的優劣
- 優勢
	- 除錯與測試容易
	- 較高吞吐量以及可擴展性，簡單算法有效減少計算開銷以及同步操作，適合並行環境
	- 較小的 meta data
- 劣勢
	- 可能在特定 workload 底下，如不斷變化的記憶體存取行為或是較小的 cache，效率可能低於複雜演算法
	- 可能忽視存取頻率或時間等歷史資訊，導致效能下降
- 實際例子
	- MemC3
	- MICA
	- Segcache
	- Frozenhot
關於 SIEVE 的權衡，SIEVE 是一個結合簡單性以及高效能的演算法
- 簡單設計: 基於 FIFO 或 CLOCK，不需要複雜的 queue 或是參數調整
- 高效率: 通過 Lazy Promotion 以及 Quick Demotion，快速剔除冷門的物件，保留熱門物件
- 吞吐量以及可擴展性: 不需要躍遷物件的操作，減少計算以及 lock 競爭問題，適用於 multi-thread 環境

### 關於 SIEVE 設計
SIEVE 主要是由一個 FIFO queue 以及 hand 所組成
- FIFO queue: 用來維護 cache 中的物件，新的物件會被插入到 FIFO queue 的頭部，而淘汰物件的操作可以針對 FIFO queue 裡面任何物件
- Hand pointer: 指向到下一個等待淘汰的候選物件，Hand pointer 從 FIFO queue 的尾部逐漸向頭部移動
- Visited Bit: 每個物件占用 1 bit，記錄是否在最近被存取過，1 表示最近被存取過，可能是一個 hot 的物件，0 表示最近沒有存取過，可能是 cold 物件
  
演算法運作流程如以下
- cache hit: 當發生 cache hit 的時候，將 Visited Bit 設為 1。如果已經為 1，則不需要進行其他操作 (這一點不同於 LRU)
- cache miss: 檢查 hand pointer 指向的物件，如果 visited bit 為 1，則將 access bit 重新設定為 0，並移動 hand pointer 到下一個物件 (物件保留在原本位置)。如果 visited bit 為 0，則將該物件直接淘汰並將新的物件加入到 queue 的頭部
- 淘汰完成後，hand pointer 會繼續移動已掃描下一個物件
- 整個 queue 會發現被淘汰的物件會集中於 queue 的中間部分。新物件與要保留的物件分離。新的物件集中在 queue 的頭部，要保留的物件集中在 queue 的尾部
  
![[Pasted image 20241219141045.png]]
  
設計關鍵部分為 Lazy Promotion 以及 Quick Demotion
- Lazy Promotion
	- 當一個物件的 visited bit 為 1 時，不進行任何位置移動，直到該物件需要被淘汰的時候才會有移動的操作，而非像 LRU 每次 cache 命中時立即將物件移動到 queue 的頭部。物件會保持在原始位置，避免針對整個 queue 的移動操作
	- 優勢為減少計算開銷以及提高效率，通過 Lazy Promotion 可以基於更多訊息，如存取記錄做出更好的決定
- Quick Demotion
	- 當新物件插入到 queue 頭部時，hand pointer 會開始進行掃描，如果 visited = 0 直接移除。如果 visited = 1 則保留，但 access bit 重設為 0
	- 優勢為可以快速清除 cold 的物件。而 hot 的物件被保留，並重設 visited = 0。如果 workload 符合 Power-law Distribution，則可以保留少數 hot 的物件
### 範例，LRU vs SIEVE
假設有存取序列 A → B → A → C → D → A → E → B → F → B → C, frame_size = 3

LRU 存取過程如下
	存取 A, queue 狀態: [A]
	存取 B, queue 狀態: [B, A]
	存取 A, queue 狀態: [A, B]
	存取 C, queue 狀態: [C, A, B]
	存取 D, queue 狀態: [D, C, A] B 被淘汰
	存取 A, queue 狀態: [A, D, C]
	存取 E, queue 狀態: [E, A, D] C 被淘汰
	存取 B, queue 狀態: [B, E, A] D 被淘汰
	存取 F, queue 狀態: [F, B, E] A 被淘汰
	存取 B, queue 狀態: [B, F, E]
	存取 C, queue 狀態: [C, B, F] E 被淘汰
	
SIEVE 存取過程如下
	存取 A, queue 狀態: [A(0)]
	存取 B, queue 狀態: [B(0), A(0)]
	存取 A, queue 狀態: [B(0), A(1)]
	存取 C, queue 狀態: [C(0), B(0), A(1)]
	存取 D, hand pointer 指向到 A, 清除 A 的 visited bit 之後移動到 B, B 的 visited bit 為 0，被淘汰, queue 狀態: [D(0), C(0), A(0)]
	存取 A, queue 狀態: [D(0), C(0), A(1)]
	存取 E, hand pointer 指向到 C, C 被移除, queue 狀態 [E(0), D(0), A(1)]
	存取 B, hand pointer 指向到 D, D 被移除, queue 狀態 [B(0), E(0), A(1)]
	存取 F, hand pointer 指向到 E, E 被移除, queue 狀態 [F(0), B(0), A(1)]
	存取 B, queue 狀態 [F(0), B(1), A(1)]
	存取 C, hand pointer 指向到 B, B visited bit 被設為 0，接著移動到 F, F 被移除, queue 狀態 [C(0), B(0), A(1)] 
上面給定的存取分布，有一定的 Power-law Distribution 的現象，可以看到 SIEVE 保留了少數 hot 的物件，比起 LRU 有更好的效果，且 cold 的物件也很快地被替換走

## 關於 SIEVE 的效能測試
![[Pasted image 20241219155638.png]]
cache type 取決於 cache library/cache system, 接下來實驗會使用 workload 來描述這一些追蹤 cache 記錄的 dataset。這一些資料大多數都屬於 web 類型的應用，實驗會分別在作者撰寫的 libCacheSim 中執行以及實機上面執行。實驗不會進行預熱 cache 等行為。

評估目標為以下
- SIEVE 有沒有比現在的演算法還要快?
- SIEVE 是否可以增進 cache 的可擴展性或是吞吐量
- SIEVE 有沒有比其他演算法簡單
(特別要注意的: 對於某些 workload，會不會發生 hot 物件過早被驅逐導致效率下滑)

### 關於測量指標
- Miss ration (MR): 用來評估 cache 的重要指標，但是不同資料之間的 Miss ration 會有非常大的差異，直接進行比較然後針對這一些資料進行視覺化表示並不可行，因此需要將資料進行標準化

為了解決上面 Miss ration 的問題，統計與比較引入了 Miss Ration Reduction (MRR) 作為比較的相對指標，公式如以下表示
$MRR = \frac {mr_{FIFO} - mr_{algo}} {mr_{FIFO}}$ 
如果演算法的 miss ration 高於 FIFO，則改使用
$MRR = \frac {mr_{FIFO}-mr_{algo}} {mr_{algo}}$
上面這個指標會將資料標準化，上面這個指標會落於 -1 到 1 之間，可以更加清楚的表現演算法改進的結果

接著是吞吐量，單位為每百萬操作數 (Milion operation per second, Mops) 衡量 cache 效能。為了評估可擴展性，實驗會調整 thread 數量，從 1 到 16 調整吞吐量

實驗部分，有使用三個環境，分別為雲環境，模擬器環境以及本地環境
雲環境，讓 libCacheSim 在 CloudLab 上執行，並和 CloudLab 本地節點進行比較
本地環境，使用 Clemson 大學提供的計算節點，配備為 Intel Gold 6142 CPU 以及 384GB DDR4 DRAM，並關閉 Turbo Boost，將 thread 都綁訂在同一個 NUMA 節點上的 CPU core

為了驗證本地以及模擬器結果的一致性，會隨機抽樣 60 個 entry 確保結論是相同的

#### 測試結果
SIEVE 主要是針對 eviction 的改善，因此這邊主要比較不同演算法在 eviction 的效率，由於現代許多 cache 管理機制都是使用基於 slab 的機制，eviction 針對相似大小的物件，因此這邊不考慮物件大小的因素。cache 大小設定為被追蹤物件數量的百分比，也就是 cache 大小不是一個固定大小，如 1000 等等，如果我們要追蹤的檔案包含 10000 個物件，則 cache 大小設定為 10%, 則 cache 容量為 1000 個物件。下面測試評估 8 種不同的物件大小，選擇 0.1% 和 10% 兩種快取大小進行測試，這些大小分別為對應於追蹤檔案中唯一物件數量，也就是 trace footprint 的比例，所謂 trace footprint 意思為假設給定 A, B, C, A, D, B, E, A, C 存取序列，出現過的物件為 A,B,C,D,E，我們定義上面存取序列的 trace footprint 為 5。

測試中使用三個大型資料集，CDN1, CDN2, Twitter，下圖顯示各追蹤檔案中不同演算法相當於 FIFO 的 miss ration 減少情況，圖中的 box plot 以 p10 和 p90 定義 whisker，用來排除極端的離群值

![[Pasted image 20241228131607.png]]
X 軸: 不同的快取演算法
Y 軸: 和基於 FIFO 的演算法相比，其他演算法在減少 miss ratio 的改進程度 (越高越好)
上下的 whisker (虛線): 表示第 10 百分位數以及第 90 百分位數的資料，用於排除離群以及極端值，專注於典型情況
box 本身，表示第 25 百分位數以及第 75 百分位數
三角形符號: 表示演算法在該 dataset 底下平均 miss ratio 的減少量

對於大的 cache 大小，也就是 10% 的 trace point，在 CDN1 的資料集中，SIEVE 幾乎在所有百分位數都有顯著的減少 miss ratio，在 10% 的 traces 上 miss ratio 減少了 42%，平均減少的 miss ratio 為 21%。作為比較，所有其他演算法在 CDN1 這個資料集中減少的幅度比較小，如和 SIEVE 概念相似的 CLOCK 演算法，平均減少 miss ratio 為 15%。SIEVE 和 ARC 相比，平均 miss ratio 減少 1.5%。

接著是對於小的 cache 大小，TwoQ 和 LHD 有時候會比 SIEVE 還要好，正如同我們前面所說，簡單的淘汰策略在小的 cache 大小表現可能會較差，在這個 benchmark 中可以看出這一件事情，TwoQ 以及 LHD 之所以在小的 cache 大小表現比較好，是因為這一些演算法能快速移除新插入的低價值物件，這一點與 SIEVE 相似，但 SIEVE 表現較差的原因是新的物件還不知道他是不是 hot 的就已經被移除了，造成較高的 miss ratio。

小的 cache 大小在 ARC 以及 LIRS 也會有各自的問題，對於 ARC 的自適應機制有時候會過度縮小 recency queue，導致較高的 miss ratio。而 LIRS 設計中也存在類似的問題，當 cache 大小再進一步縮小時，miss ratio 甚至比 FIFO 還要高。

而 TwoQ 在小的 cache 情況下表現穩定，因為它固定保留 25% 的 cache 空間給新物件，避免發生過度 demotion 的狀況。

雖然看起來 SIEVE 在小的 cache 大小情況下表現較差，但是根據作者在 XXX 的報告中顯示，在實際生產情況大多數 cache 大小都接近於大的 cache 大小。

![[Pasted image 20241228143454.png]]
接著是對於較少追蹤檔案資料的資料集，也就是較小的資料集 Meta KV, Meta CDN, Wiki 以及 TencentPhoto，這些資料集使用散點圖進行比較，結果顯示在較大 cache 情況下，SIEVE 在所有資料集中超越其他演算法，而對於較小的快取大小，TwoQ 以及 LHD 表現較好，雖然 SIEVE 並非最好，不過也有一定的競爭力。
![[Pasted image 20241228144456.png]]
上面這張圖比較各個演算法在不同資料集中成為最佳演算法的比例，也就是綜合圖 4 和圖 3 的結果。

對於大的 cache size，在 Tencent Photo, Wiki, Meta KV 資料集中，SIEVE 表現超越所有其他演算法。在其他資料集中，SIEVE 也在較大部分的資料軌跡中贏過其他演算法。而對於小的 cache size, TwoQ 為最好的演算法，接著是 SIEVE 以及 LHD，兩者有著差不多的效能。

### 關於吞吐量
這邊 dataset 的大小隨著硬體規格，也就是 core 數量以及 dram 大小而增加，大略的把 dataset 大小定義為 working set 的大小，cache size 和 working set size 以及 thread 數量一同擴展

![[Pasted image 20241228153752.png]]
使用 Meta 以及 Twitter 這兩個 trace 的資料進行回放，可以發現到 LRU 的擴展性十分的差，而對於 Cachelib (由 Facebook 開發) 裡面使用的優化的 LRU 演算法以及 TwoQ 演算法在可擴展性上有了顯著的提升，優化技巧包含過去 60 秒被躍遷到 head 的物件不會再次被躍遷，如此可以減少 lock 的競爭，並且不會影響到 miss ratio。Cachelib 還進一步做了一些 lock 合併的優化，避免共享結構的同步操作帶來的額外開銷，以提高吞吐量，不過我們還是可以看到這樣的優化還是遇到了瓶頸問題，大約在 8 核心之後吞吐量出現了瓶頸。

與這一些基於 LRU 的演算法，SIEVE 在每次 cache hit 不需要做躍遷操作，因此執行速度更快，且不需要對共享資料結構的同步操作，在可擴展性方面，可以看到 SIEVE 遠勝於優化過的 LRU 以及 TwoQ。

### 關於 SIEVE 作為其他演算法的基礎 (cache primitive)
SIEVE 除了作為一種 cache 演算法，也可以作為其他演算法的基礎，為了研究 SIEVE 作為其他演算法基礎的可行性以及適用範圍，可以簡略的將目前的 cache 演算法設計分成四個主要方向
- 簡單的 eviction 演算法: FIFO queue, LRU queue, LFU queue, random 等等都是這一種，SIEVE 也是
- 改進簡單 eviction 的演算法: 如 FIFO-reinsertion, LRU-K 
- 結合多個 cache primitive: 將多個 cache primitive 結合的演算法，讓物件這些物件之間移動，如 ARC, SLRU, MQ 使用多個 LRU queue
- 結合多個 cache primitive 並設計出決策者來選擇 eviction 物件: 對於決策者的部分，目前許多使用機器學習技術實現，如 LeCaR 使用強化學習選擇來自 LRU, LFU 的 eviction 候選物件, HALR 也是使用類似的技術

這邊進一步將 SIEVE 與 FIFO, LRU cache 進行比較，研究這幾個言算法執行他們所需要的指令數量，指令數量只是大致評估 CPU 資源使用量的一個指標，不值些與延遲或是吞吐量有相關，這裡使用 perf stat 測量具有 power-law distribution 的 dataset 進行測試。
![[Pasted image 20241228164523.png]]
可以看到 SIEVE 所需指令數量最少，因為 SIEVE 只需要檢查並更新 cache bit 的 bool, 比 reinsertion 操作簡單很多，這讓 SIEVE 在更大 cache size 以及高吞吐量的 workload 底下表現較為高效。大致上 SIEVE 比 LRU 減少最多 40% 指令數量，比 FIFO 減少 24%

下面嘗試將 LeCaR, TwoQ, ARC 中的 LRU 替換成 SIEVE，並對這一些演算法進行測試
![[Pasted image 20241228170459.png]]
可以發現到確實有效的減小了 miss ratio，而由於我們是在確定的序列進行存取，因此我們可以使用 Belady 進行比較，讓在決定是否要進行 eviction 可以根據未來存取情況決定，也就是一個完美的決策者，當我們讓 SIEVE, LRU, FIFO 都有完美決策者的情況下，SIEVE 在絕大部分情況都達到了最低 miss ratio，這表示 SIEVE 有足夠的潛力可以強化複雜演算法的效能。

### 與機器學習演算法比較討論
LRB 是一個基於機器學習的 cache eviction 演算法，在 cache_size 較小情況下表現比 SIEVE 好，這是因為在 cache size 較小情況，理想的 cache 物件應該是頻繁存取且具有更多特徵的物件，由於有更多的特徵，更容易被 LRB 偵測到，但是當 cache size 變大時，大部分物件存取次數變少，特徵減少，LRB 就無法精確地對物件進行辨認，這時候基於機器學習的演算法可能就無法提供性能上的提升

![[Pasted image 20241228171523.png]]

### SIEVE 與 Block cache workloads
資料以 block 形式進行管理和存取，block 大小可能是 4kb 等等，可能位於硬碟或是記憶體中，Block cache workloads 有高頻率存取以及 scan pattern 的模式，也就是對某一塊資料進行循序讀取或是搜尋，在 Block cache workloads 中 SIEVE 的 miss ratio 比 LRU 還要差，原因在於 scan pattern 會將頻繁存取的物件以及頻繁被掃描到，也就是掃描過程中會存取的資料這兩種資料混雜在一起，這一些物件被混合一起可能會導致 scan pattern 過程中 hot 物件被經常掃描到的物件覆蓋，並 eviction 出 cache，通常這種情況會需要使用 ghost cache 或是 ghost list 去維護一些物件資訊才能夠解決(如 ARC, LIRS, LRU-2 等等)，而因為 SIEVE 不具備 scan pattern 得能力，這很有可能是為什麼 SIEVE 沒有被發現的原因，因為許多關於 cache 的研究會考慮到 Block cache 的 workloads，而 Web 中不大存在這種 workloads。

https://news.ycombinator.com/item?id=38850202

比較 access bit 演算法與非 access bit 演算法要怎麼比較
有沒有辦法把 access bit 演算法改成依賴於非 access bit 的演算法? 改完才能夠進行比較
演算法比較可以跟 random 演算法進行比較