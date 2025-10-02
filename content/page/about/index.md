---
title: "About Me"
date: 2025-05-19
layout: "about"
slug: "about"
menu:
    main:
        name: AboutMe
        weight: 0
        params: 
            icon: user
---

## 簡介
我是吳承翰，目前在國立中正大學資訊工程所，作業系統實驗室功讀碩士學位。

我對於 Linux Kernel 的優化以及除錯有著濃厚的興趣，在這方面我有使用 perf, eBPF/Bpftrace 的相關經驗，以及使用視覺化工具了解效能瓶頸以及子系統的運作，如 flamegraph, gprof2dot 等工具。

在研究方面，我特別專注於 Linux 的記憶體管理部份。目前我的研究方向是專注於使用 CPU 的 PMU 資訊改善 Linux 記憶體管理機制，針對於記憶體冷熱分層進行改善，期望能夠提高系統的吞吐量。

在個人閒暇之餘，我喜歡撰寫技術教學相關文章，希望藉由這類文章對他人有所啟發並且和他人交流，一同在技術上有所成長，目前我的創作主要在 Hackmd 以及 it 邦幫忙上，累積觀看次數有 40 萬次以上觀看。

## 學歷
- 2024 ~ 現在 國立中正大學 資訊工程研究所 作業系統實驗室
- 2020 ~ 2024 逢甲大學 資訊工程學系

## 經歷
- Coscup 2025 講者: Let's Tracing Linux Kernel, MGLRU 實作與分析
- 2025 中正大學資訊工程學系系統程式教學助教
- 2024 中正大學資訊工程學系作業系統教學助教
- 2023 逢甲大學 Google Developer Student Clubs (GDSC) 教學組
- 2022 逢甲大學程式設計課程教學助教
- 2022 逢甲大學黑客社教學組

## 專案
- 以 Rust 程式語言改寫 Linux Kernel Module 以增強其記憶體管理安全性 - 以 ksmbd 模組為例 (獲得國科會大專生計畫補助)
    - 使用 Rust 重構程式碼邏輯，復現記憶體相關漏洞，比較底層實作差異
- Google Chrome Memory Manager
    - 使用 Rust 結合 Kernel Module 解決 Google Chrome 占用記憶體問題，偵測閒置分頁並清除該分頁記憶體資源
- Linux Memory Exp Box
    - 這是提供給實驗室分析記憶體行為的工具組，可以快速驗證頁置換演算法的效能差異，針對子系統中特定函式進行觀測，以及檢視特定負載觸發的 IBS 事件

## 獎項
- 2022 ICPC Taiwan Private University Programming Contest 銀獎
- [2022 iTHome 鐵人賽軟體開發組鐵人練成獎，以著作「與作業系統的第一類接觸：探索 xv6 參賽」](https://ithelp.ithome.com.tw/users/20138181/ironman/5395)
- [2021 iTHome 鐵人賽自我挑戰組鐵人練成獎，以著作「從 0 開始啃 Introduction of algorithms 心得與記錄參賽」](https://ithelp.ithome.com.tw/users/20138181/ironman/4156)

## 開源貢獻
- Linux Kernel: [ksmbd: Remove unused field in ksmbd_user struct](https://patchew.org/linux/20231002053203.17711-1-hank20010209@gmail.com/)
- Linux Kernel: [mm: remove unused macro INIT_PASID](https://lore.kernel.org/all/20250427145004.13049-1-hank20010209@gmail.com/T/#u)
- Linux Kernel: [mm/DAMON: BUG fix, add target_pid to DAMOS example command](https://lore.kernel.org/all/20250915015807.101505-6-sj@kernel.org/)
- Linux Kernel: [mm/DAMON: _damon_args: check --target_pid is set or not](https://git.kernel.org/pub/scm/linux/kernel/git/sj/damo.git/commit/?id=f865d981a3e1eda94b191fbba74aff9348691d3c)
- nushell: [Fix broken links in configuration.md, environment.md and explore.md](https://github.com/nushell/nushell.github.io/pull/1852)