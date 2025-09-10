---
title: "NV Jetson Debugging, Update BootLoader"
description: 
date: 2025-09-10T23:00:12+08:00
image: 
math: 
license:
categories:
    - NVIDIA 
tags:
    - NVIDIA
    - Debugging
hidden: false
comments: false
draft: false
---
# NV Jetson Debugging, Update BootLoader

## 問題描述
發現只要開機之前插入 webcam 就會無法開機，沒有畫面，推測是在 UEFI 或是 Second Bootloader 的時候發生錯誤，後面 Thrid Bootloader 到 Linux 都沒有被成功載入。

通過聽風扇轉速判斷在第一次失敗之後有重新開機過一次，在開機的時候風扇轉速較大，接著轉速維持在一定速率。

由於黑畫面，沒有任何訊息，這時候沒有 printf, gdb 那一些資源可以使用，直覺想到也許可以通過 GPIO, UART，但需要修改 UEFI 讓他的 TX 送出啟動階段的訊息，這時候先查詢 Jetson 使用 GPIO 的方法，發現除了側面 40 pin 的 GPIO 之外還有後面的 micro-USB 可以使用，且 UEFI 會通過這個界面送出訊息，只要接收端設置 baudrate 為 115200 就可以收到訊息。

### 通過 micro-usb 得到 Debug 訊息
將 micro-usb 與 host 相互連接之後會出現 ttyACM0 ~ ttyACM3 設備，選取 ttyAMA0 通過 `minicom` 成功讀取到啟動階段訊息，方便處理我們直接將訊息寫檔之後讀出
```shell
ls /dev/ttyACM*
sudo minicom -D /dev/ttyACM -b 115200 -8 -o -C <output_file_name>
```

訊息如下

```
CPU switching to normal world boot

Jetson UEFI firmware (version 36.4.4-gcid-41062509 built on 2025-06-16T15:25:51+00:00)


I/TC: Reserved shared memory is disabled
I/TC: Dynamic shared memory is enabled
I/TC: Normal World virtualization support is disabled
I/TC: Asynchronous notifications are disabled
I/TC: WARNING: Test UEFI variable auth key is being used !
I/TC: WARNING: UEFI variable protection is not fully enabled !

[     6.318531] Camera-FW on t234-rce-safe started
TCU early console enabled.
[     6.380337] Camera-FW on t234-rce-safe ready SHA1=e2238c99 (crt 1.360 ms, total boot 63.229 ms)
3h
ASSERT [XhciDxe] /out/nvidia/bootloader/uefi/Jetson_RELEASE/edk2/MdeModulePkg/Bus/Pci/XhciDxe/XhciSched.c(3154): Interval >= 1 && Interval <= 16

Resetting the system in 5 seconds.
Shutdown state requested 1
Rebooting system ...

[0000.062] I> MB1 (version: 1.4.0.4-t234-54845784-e89ea9bc)
[0000.067] I> t234-A01-1-Silicon (0x12347) Prod
[0000.072] I> Boot-mode : Coldboot
[0000.075] I> Entry timestamp: 0x00000000
[0000.078] I> last_boot_error: 0x0
[0000.082] I> BR-BCT: preprod_dev_sign: 0
[0000.085] I> rst_source: 0xb, rst_level: 0x1
[0000.089] I> Task: SE error check
[0000.093] I> Task: Bootchain select WAR set
[0000.097] I> Task: Enable SLCG
[0000.100] I> Task: CRC check
[0000.102] I> Task: Initialize MB2 params
[0000.107] I> MB2-params @ 0x40060000
[0000.110] I> Task: Crypto init
[0000.113] I> Task: Perform MB1 KAT tests
[0000.117] I> Tas
[0000.124] I> Task: MSS Bandwidth limiter settings for iGPU clients
[0000.130] I> Task: Enabling and initialization of Bandwidth limiter
[0000.136] I> No request to configure MBWT settings for any PC!
[0000.142] I> Task: Secure debug controls
[0000.146] I> Task: strap war set
[0000.149] I> Task: Initialize SOC Therm
[0000.153] I> Task: Program NV master stream id
[0000.157] I> Task: Verify boot mode
[0000.163] I> Task: Alias fuses
[not supported.
[0000.173] I> Task: Print SKU type
[0000.177] I> FUSE_OPT_CCPLEX_CLUSTER_DISABLE = 0x00000000
[0000.182] I> FUSE_OPT_GPC_DISABLE = 0x00000000
[0000.186] I> FUSE_OPT_TPC_DISABLE = 0x00000000
[0000.190] I> FUSE_OPT_DLA_DISABLE = 0x00000000
[0000.195] I> FUSE_OPT_PVA_DISABLE = 0x00000000
[0000.199] I> FUSE_OPT_NVENC_DISABLE = 0x00000000
[0000.203] I> FUSE_OPT_NVDEC_DISABLE = 0x00000000
[0000.208] I> FUSE_OPT_FSI_DISABLE = 0x00000000
[0000.212] I> FUSE_OPT_EMC_DISABLE = 0x00000000
[0000.216] I> FUSE_BOOTROM_PATCH_VERSION = 0x7
[0000.221] I> FUSE_PSCROM_PATCH_VERSION = 0x7
[0000.225] I> FUSE_OPT_ADC_CAL_FUSE_REV = 0x2
[0000.229] I> FUSE_SKU_INFO_0 = 0xd0
[0000.232] I> FUSE_OPT_SAMPLE_TYPE_0 = 0x3 PS 
[0000.236] I> FUSE_PACKAGE_INFO_0 = 0x2
[0000.240] I> SKU: Prod
[0000.242] I> Task: Boost clocks
[0000.245] I> Initializing NAFLL for BPMP_CPU_NIC.
[0000.250] I> BPMP NAFLL: fll_lock = 1, dvco_min_reached = 0
[0000.256] 42, divisor = 0
[0000.264] I> Initializing PLLC2 for AXI_CBB.
[0000.268] I> AXI_CBB : src = 35, divisor = 0
[0000.272] I> Task: Voltage monitor
[0000.275] I> VMON: Vmon re-calibration andHSIO UPHY init done
[0000.289] W> Skipping GBE UPHY config
[0000.292] I> Task: Boot device init
[0000.295] I> Boot_device: QSPI_FLASH instance: 0
[0000.300] I> Qspi clock source : pllc_out0
[0000.304] I> QSPI Flash: Macronix 64MB
[0000.308] I> QSPI-0315] I> Task: Load membct
[0000.318] I> RAM_CODE 0x4000431
[0000.321] I> Loading MEMBCT 
[0000.324] I> Slot: 0
[0000.326] I> Binary[0] block-3840 (partition size: 0x40000)
[0000.331] I> B92
[0000.338] I> Size of crypto header is 8192
[0000.342] I> strt_pg_num(3840) num_of_pgs(16) read_buf(0x40050000)
[0000.348] I> BCH of MEM-BCT-0 read from storage
[0000.353] I> BCH address is : 0x40050000
[0000.357] I> MEM-BCT-0 header integrity checEM0
[0000.367] I> component binary type is 0
[0000.370] I> strI> MEM-BCT-0 binary is read from storage
[0000.382] I> MEM-BCT-0 binary integrity check is success
[0000.387] I> Binary MEM-BCT-0 loaded successfully at 0x40040000 (0xe580)
[0000.394] I> RAM_CODE 0x4000431
[0000.399] I> RAM_CODE 0x4000431
[0000.403] Iams override
[0000.411] I> Task: Save mem-bct info
[0000.414] I> Task: Carveout allocate
[0000.418] I> RCM blob carveout will not be allocated
[0000.423] I> Update CCPLEX IST carveout from MB1-BCT
[0000.427] I> ECC region[0]: Start:0x0, End:0x0
[0000.432] I> ECC region[1]: Start:0x0, End:0x0
[0000.436] I> ECC region[2]: Start:0x0, End:0x0
[0000.440] I> ECC region[3]: Start:0x0, End:0x0
[0000.445] I> ECC region[4]: Start:0x0, End:0x0
[0000.449] I> Non-ECC region[0]: Start:0x80000000, End:0x10800000ECC region[3]: Start:0x0, End:0x0
[0000.469] I> Non-ECC region[d(CO:31) base:0x1040000000 size:0x8000000 align: 0x8000000
[0000.494] I> allocated(CO:43) base:0x103c000000 size:0x4000000 align: 0x200000
[0000.501] I> allocated(CO:39) base:0x1039e00000 size:0x2200000 align: 0x10000
[0000.508] I> allocated(CO:20) base:0x1036000000 size:0x2000000 align: 0x2000000
[0000.515] I> allocated(CO:24) base:0x1034000000 size:0x2000000 align: 0x2000000
[0000.523] I> allocated(CO:28) base:0x1032000000 size:0x2000000 align: 0x2000000
[0000.530] I> allocated(CO:29) base:0x1030000000 size:0x2000000 align: 0x2000000
[0000.537] I> allocated(CO:22) base:0x1048000000 size:0x1000000 align: 0x1000000
[0000.544] I> allocated(CO:35) base:0x1038e00000 size:0x1000000 align: 0x100000
[0000.551] I> allocated(CO:41) base:0x102f000000 size:0x1000000 align: 0x100000
[0000.558] I> allocated(CO:02) base:0x1049000000 size:0x800000 align: 0x800000
[0000.565] I> allocated(CO:03) base:0x1038000000 size:0x800000 align: 0x800000
[0000.572] I> allocated(CO:06) base:0x102e800000 size:0x800000 align: 0x800000
[0000.579] I> allocated(CO:56) base:0x102e000000 size:0x800000 align: 0x200000
[0000.586] I> allocated(CO:07) base:0x1038800000 size:0x400000 align: 0x400000
[0000.594] I> allocated(CO:33) base:0x102dc00000 size:0x400000 align: 0x200000
[0000.601] I> allocated(CO:19) base:0x102d980000 size:0x280000 align: 0x10000
[0000.607] I> allocated(CO:23) base:0x1038c00000 size:0x200000 align: 0x200000
[0000.615] I> allocated(CO:01) base:0x102d800000 size:0x100000 align: 0x100000
[0000.622] I> allocated(CO:05) base:0x102d700000 size:0x100000 align: 0x100000
[0000.629] I> allocated(CO:08) base:0x102d600000 size:0x100000 align: 0x100000
[0000.636] I> allocated(CO:09) base:0x102d500000 size:0x100000 align: 0x100000
[0000.643] I> allocated(CO:12) base:0x102d400000 size:0x100000 align: 0x100000
[0000.650] I> allocated(CO:15) base:0x102d300000 size:0x100000 align: 0x100000
[0000.657] I> allocated(CO:17) base:0x102d200000 size:0x100000 align: 0x100000
[0000.664] I> allocated(CO:27) base:0x102d100000 size:0x100000 align: 0x100000
[0000.671] I> allocated(CO:42) base:0x102d000000 size:0x100000 align: 0x100000
[0000.678] I> allocated(CO:54) base:0x102d900000 size:0x80000 align: 0x80000
[0000.685] I> allocated(CO:34) base:0x102cff0000 size:0x10000 align: 0x10000
[0000.692] I> allocated(CO:72) base:0x102cdf0000 size:0x200000 align: 0x10000
[0000.698] I> allocated(CO:47) base:0x102c800000 size:0x400000 align: 0x200000
[0000.706] I> allocated(CO:50) base:0x102c600000 size:0x200000 align: 0x100000
[0000.713] I> allocated(CO:52) base:0x102cdc0000 size:0x30000 align: 0x10000
[0000.719] I> allocated(CO:48) base:0x102cda0000 size:0x20000 align: 0x10000
[0000.726] I> allocated(CO:69) base:0x102cd80000 size:0x20000 align: 0x10000
[0000.733] I> allocated(CO:49) base:0x102cd70000 size:0x10000 align: 0x10000
[0000.740] I> NSDRAM base: 0x80000000, end: 0x102cdf0000, size: 0xfacdf0000
[0000.747] I> Task: Thermal check
[0000.750] I> Using min_chip_limit as min_tmon_limit
[0000.755] I> Using max_chip_limit as max_tmon_limit
[0000.759] I> BCT max_tmon_limit = 105
[0000.763] I> BCT min_tmon_limit = -28
[0000.766] I> BCT max_tmon_limit = 105
[0000.770] I> BCT min_tmon_limit = -28
[0000.773] I> SKU specific max_chip_limit = 105
[0000.777] I> SKU specific min_chip_limit = -28
[0000.782] I> BCT max_chip_limit = 105
[0000.785] I> BCT min_chip_limit = -28
[0000.789] I> enable_soctherm_polling = 0
[0000.793] I> max temp read = 37
[0000.795] I> min temp read = 36
[0000.798] I> Enabling thermtrip
[0000.801] I> Task: Update FSI SCR with thermal fuse data
[0000.807] I> Task: Enable WDT 5th expiry
[0000.810] I> Task: I2C register
[0000.813] I> Task: Set I2C bus freq
[0000.817] I> Task: Reset FSI
[0000.819] I> Task: Pinmux init
[0000.823] I> Task: Prod config init
[0000.826] I> Task: Pad voltage init
[0000.829] I> Task: Prod init
[0000.832] I> Task: Program rst req config reg
[0000.836] I> Task: Common rail init
[0000.841] W> DEVICE_PROD: module = 13, instance = 4 not found in device prod.
[0000.851] I> DONE: Thermal config
[0000.856] I> DONE: SOC rail config
[0000.859] W> PMIC_CONFIG: Rail: MEMIO rail config not found in MB1 BCT.
[0000.866] I> DONE: MEMIO rail config
[0000.869] I> DONE: GPU rail info
[0000.873] I> DONE: CV rail info
[0000.876] I> Task: Mem clock src
[0000.879] I> Task: Misc. board config
[0000.883] I> PMIC_CONFIG: Platform config not found in MB1 BCT.
[0000.889] I> Task: SDRAM init
[0000.892] I> MemoryType: 4 MemBctRevision: 8
[0000.899] I> MSS CAR: PLLM/HUB programming for MemoryType: 4 and MemBctRevision: 8
[0000.906] I> MSS CAR: Init PLLM
[0000.909] I> MSS CAR: Init PLLHUB
[0000.914] I> Encryption:   MTS: en, TX: en, VPR: en, GSC: en
[0000.926] I> SDRAM initialized!
[0000.929] I> SDRAM Size in Total 0x1000000000
[0000.933] I> Task: Dram Ecc scrub
[0000.936] I> Task: DRAM alias check
[0000.952] I> Task: Program NSDRAM carveout
[0000.956] I> NSDRAM carveout encryption is enabled
[0000.961] I> Program NSDRAM ck: Enable clock-mon
[0000.982] I> FMON: Fmon re-programming done
[0000.986] I> Task: Mapper init
[0000.989] I> Task: SC7 Context Init
[0000.993] I> Task: CCPLEX IST init
[0000.996] I> Task: CPU WP0
[0001.000] I> Loading MCE
[0001.002] I> Slot: 0
[0001.004] I> Binary[8] block-22784 (partition size: 0x80000)
[00 is 8192
[0001.016] I> Size of crypto header is 8192
[0001.020] I> strt_pg_num(22784) num_of_pgs(16) read_buf(0x4003e000)
[0001.027] I> BCH of MCE read from storage
[0001.031] I> BCH addresuccess
[0001.039] I> Binary magic in BCH component 0 is MTSM
[0001.044] I> component binary type is 8
[0001.048] I> Size of crypto header is 8192
[0001.051] I> strt_pg_num(22800) num_of_pgs(350) read_buf(0x40000000)
[0001.060] I> MCE binary is read from storage
[0001.064] I> MCE binary integrity check is success
[0001.069] I> Binary MCE loaded successfully at 0x40000000 (0x2baf0)
[0001.075] I> Size of crypto header is 8192
[0001.086] I> Size of crypto header is 8192
[0001.090] I> Sending WP0 mailbox command to PSC
[0001.099] I> Task: XUSB Powergate
[0001.102] I> Skipping powergate XUSB.
[0001.106] I> Task: MB1 fixed firewalls
[0001.112] W> Firewall readback mismatch
[0001.117] I> Task: Load bpmp-fw
[0001.120] I> Slot: 0
[0001.122] I> Binary[15] block-9984 (partition size: 0x180000)
[0001.128] I> Binary name: BPMP_FW
[0001.131] I> Size of crypto header is 8192
[0001.135] I> Size of crypto header is 8192
[0001.139] I> strt_pg_num(9984) num_of_pgs(16) read_buf(0x807fe000)
[0001.145] I> BCH of BPMP_FW read from storage
[0001.149] I> BCH address is : 0x807fe000
[0001.153] I> BPMP_FW header integrity check is success
[0001.158] I> Binary magic in BCH component 0 is BPMF
[0001.163] I> component binary type is 15
[0001.166] I> Size of crypto header is 8192
[0001.170] I> strt_pg_num(10000) num_of_pgs(1990) read_buf(0x80000000)
[0001.188] I> BPMP_FW binary is read from storage
[0001.194] I> BPMP_FW binary integrity check is success
[0001.199] I> Binary BPMP_FW loaded successfully at 0x80000000 (0xf8bc0)
[0001.206] I> Slot: 0
[0001.208] I> Binary[16] block-13056 (partition size: 0x400000)
[0001.213] I> Binary name: BPMP_FW_DTB
[0001.217] I> Size of crypto header is 8192
[0001.221] I> Size of crypto header is 8192
[0001.225] I> strt_pg_num(13056) num_of_pgs(16) read_buf(0x807fc000)
[0001.231] I> BCH of BPMP_FW_DTB read from storage
[0001.236] I> BCH address is : 0x807fc000
[0001.239] I> BPMP_FW_DTB header integrity check is success
[0001.245] I> Binary magic in BCH component 0 is BPMD
[0001.250] I> component binary type is 16
[0001.253] I> Size of crypto header is 8192
[0001.257] I> strt_pg_num(13072) num_of_pgs(502) read_buf(0x807bd3f0)
[0001.266] I> BPMP_FW_DTB binary is read from storage
[0001.272] I> BPMP_FW_DTB binary integrity check is success
[0001.277] I> Binary BPMP_FW_DTB loaded successfully at 0x807bd3f0 (0x3eb40)
[0001.284] I> Task: BPMP fw ast config
[0001.287] I> Task: Load psc-fw
[0001.290] I> Slot: 0
[0001.292] I> Binary[17] block-21248 (partition size: 0xc0000)
[0001.298] I> Binary name: PSC_FW
[0001.301] I> Size of crypto header is 8192
[0001.305] I> Size of crypto header is 8192
[0001.309] I> strt_pg_num(21248) num_of_pgs(16) read_buf(0x80ffe000)
[0001.315] I> BCH of PSC_FW read from storage
[0001.319] I> BCH address is : 0x80ffe000
[0001.323] I> PSC_FW header integrity check is success
[0001.328] I> Binary magic in BCH component 0 is PFWP
[0001.333] I> component binary type is 17
[0001.337] I> Size of crypto header is 8192
[0001.340] I> strt_pg_num(21264) num_of_pgs(591) read_buf(0x80fb4200)
[0001.350] I> PSC_FW binary is read from storage
[0001.355] I> PSC_FW binary integrity check is success
[0001.360] I> Binary PSC_FW loaded successfully at 0x80fb4200 (0x49df0)
[0001.366] I> Task: Load nvdec-fw
[0001.369] I> Slot: 0
[0001.371] I> Binary[7] block-6400 (partition size: 0x100000)
[0001.377] I> Binary name: NVDEC
[0001.380] I> Size of crypto header is 8192
[0001.384] I> Size of crypto header is 8192
[0001.388] I> strt_pg_num(6400) num_of_pgs(16) read_buf(0x800fe000)
[0001.394] I> BCH of NVDEC read from storage
[0001.398] I> BCH address is : 0x800fe000
[0001.402] I> NVDEC header integrity check is success
[0001.407] I> Binary magic in BCH component 0 is NDEC
[0001.411] I> component binary type is 7
[0001.415] I> Size of crypto header is 8192
[0001.419] I> strt_pg_num(6416) num_of_pgs(560) read_buf(0x80000000)
[0001.428] I> NVDEC binary is read from storage
[0001.433] I> NVDEC binary integrity check is success
[0001.438] I> Binary NVDEC loaded successfully at 0x80000000 (0x46000)
[0001.444] I> Size of crypto header is 8192
[0001.456] I> Task: Load tsec-fw
[0001.459] I> TSEC-FW load support not enabled
[0001.463] I> Task: GPIO interrupt map
[0001.467] I> Task: SC7 context save
[0001.470] I> Slot: 0
[0001.472] I> Binary[27] block-0 (partition size: 0x100000)
[0001.477] I> Binary name: BR_BCT
[0001.480] I> Size of crypto header is 8192
[0001.484] I> Size of crypto header is 8192
[0001.488] I> Size of crypto header is 8192
[000001.498] I> BR_BCT binary is read from storage
[0001.503] I> BR_BCT binary integrity check is success
[0001.507] I> Binary BRlot: 0
[0001.516] I> Binary[13] block-23808 (partition size: 0x30000)
[0001.521] I> Binary name: SC7-FW
[0001.524] I> Size of crypto header is 8192
[0001.528] I> Size of crypto header is 8192
[0001.532] I> Size of crypto header is 8192
[0001.536] I> Size of crypto header is 8192
[0001.540] I> strt_pg_num(23808) num_of_pgs(16) read_buf(0xa0002000)
[0001.546] I> BCH of SC7-FW read from storage
[0001.551] I> BCH address is : 0xa0002000
[0001.554] I> SC7-FW header integrity check is success
[0001.559] I> Binary magic in BCH component 0 is WB0B
[0001.564] I> component binary type is 13
[0001.568] I> Size of crypto header is 8192
[0001.572] I> strt_pg_num(23824) num_of_pgs(349) read_buf(0xa0004000)
[0001.580] I> SC7-FW binary is read from storage
[0001.585] I> SC7-FW binary integrity check is success
[0001.590] I> Binary SC7-FW loaded successfully at 0xa0004000 (0x2ba00)
[0001.596] I> Slot: 0
[0001.598] I> Binary[22] block-24192 (partition size: 0x30000)
[0001.604] I> Binary name: PSC_RF
[0001.607] I> Size of crypto header is 8192
[0001.611] I> Size of crypto header is 8192
[0001.615] I> Size of crypto header is 8192
[0001.618] I> Size of crypto header is 8192
[0001.622] I> strt_pg_num(24192) num_of_pgs(16) read_buf(0xa002fa00)
[0001.629] I> BCH of PSC_RF read from storage
[0001.633] I> BCH address is : 0xa002fa00
[0001.637] I> PSC_RF header integrity check is success
[0001.641] I> Binary magic in BCH component 0 is PSCR
[00(224) read_buf(0xa0031a00)
[0001.662] I> PSC_RF binary is read a00 (0x1be60)
[0001.680] I> Task: Save WP0 payload to SC7 ctx
[0001.685] I> Task: Load MB2rf binary to SC7 ctx
[0001.689] I> Slot: 0
[0001.691] I> Binary[14] block-24576 (partition size: 0x20000)
[0001.697] I> Binary name: MB2_RF
[0001.700] I> Size of crypto header is 8192
[0001.704] I> Size of crypto header is 8192
[0001.708] I> Size of crypto header is 8192
[0001.712] I> Size of crypto header is 8192
[0001.715] I> strt_pg_num(24576) num_of_pgs(16) read_buf(0xa00d5d10)
[0001.722] I> BCH of MB2_RF read from storage
[0001.726] I> BCH address is : 0xa00d5d10
[0001.730] I> MB2_RF header integrity check is success
[0001.735] I> Binary magic in BCH component 0 is MB2R
[0001.739] I> component binary type is 14
[0001.743] I> Size of crypto header is 8192
[0001.747] I> strt_pg_num(24592) num_of_pgs(224) read_buf(0xa00d7d10)
[0001.755] I> MB2_RF binary is read from storage
[0001.759] I> MB2_RF binary integrity check is success
[0001.764] I> Binary MB2_RF loaded successfully at 0xa00d7d10 (0x1bf60)
[0001.771] I> Task: Save fuse alias data to SC7 ctx
[0001.775] I> Task: Save PMIC data to SC7 ctx
[0001.779] I> Task: Save Pinmux data to SC7 ctx
[0001.784] I> Task: Save Pad Voltage data to SC7 ctx
[0001.788] I> Task: Save controller prod data to SC7 ctx
[0001.793] I> Task: Save prod cfg data to SC7 ctx
[0001.798] I> Task: Save I2C bus freq data to SC7 ctx
[0001.803] I> Task: Save SOCTherm data to SC7 ctx
[0001.807] I> Task: Save FMON data to SC7 ctx
[0001.811] I> Task: Save VMON data to SC7 ctx
[0001.815] I> Task: Save TZDRAM data to SC7 ctx
[0001.820] I> Task: Save GPIO int data to SC7 ctx
[0001.824] I> Task: Save clock data to SC7 ctx
[0001.828] I> Task: Save debug data to SC7 ctx
[0001.832] I> Task: Save MBWT data to SC7 ctx
[0001.840] I> SC7 context save done
[0001.844] I> Task: Load MB2/Applet/FSKP
[0001.847] I> Loading MB2
[0001.850] I> Slot: 0
[0001.852] I> Binary[6] block-8448 (partition size: 0x80000)
[0001.857] I> Binary name: MB2
[0001.860] I> Size of crypto header is 8192
[0001.864] I> Size of crypto header is 8192
[0001.868] I> strt_pg_num(8448) num_of_pgs(16) read_buf(0x8007e000)
[0001.874] I> BCH of MB2 read from storage
[0001.878] I> BCH address is : 0x8007e000
[0001.882] I> MB2 header integrity check is success
[0001.886] I> Binary magic in BCH component 0 is MB2B
[0001.891] I> component binary type is 6
[0001.895] I> Size of crypto header is 8192
[0001.899] I> strt_pg_num(8464) num_of_pgs(846) read_buf(0x80000000)
[0001.910] I> MB2 binary is read from storage
[0001.915] I> MB2 binary integrity check is success
[0001.919] I> Binary MB2 loaded successfully at 0x80000000 (0x69a70)
[0001.926] I> Task: Map CCPLEX SHARED carveout
[0001.930] I> Task: Prepare MB2 params
[0001.934] I> Task: Dram ecc test
[0001.937] I> Task: Misc NV security settings
[0001.941] I> NVDEC sticky bits programming done
[0001.945] I> Successfully powergated NVDEC
[0001.949] I> Task: Disable/Reload WDT
[0001.953] I> Task: Program misc carveouts
[0001.957] I> Program IPC carveouts
[0001.960] I> Task: Disable SCPM/POD reset
[0001.964] I> SLCG Global override status := 0x0
[0001.968] I> MB1: MSS reconfig completed
I> MB2 (version: 0.0.0.0-t234-54845784-22833a33)
I> t234-A01-1-Silicon (0x12347)
I> Boot-mode : Coldboot
I> Emulation: 
I> Entry timestamp: 0x001e7647
I> Regular heap: [base:0x40040000, size:0x10000]
I> DMA heap: [base:0x102e000000, size:0x800000]
I> Task: SE error check
I> Task: Crypto init
I> Task: MB2 Params integrity check
I> Task: Enable CCPLEX WDT 5th expiry
I> Task: ARI update carveout TZDRAM
I> Task: Configure OEM set LA/PTSA values
I> Task: Check MC errors
I> Task: SMMU external bypass disable
I> Task: Enable hot-plug capability
I> Task: TZDRAM heap init
I> Task: PSC mailbox init
I> Task: Enable clock for external modules
I> Task: Measured Boot init
I> Task: fTPM silicon identity init
I> fTPM is not enabled.
I> Task: OEM SC7 context save init
I> Task: I2C register
I> Task: Map CCPLEX_INTERWORLD_SHMEM carveout
I> Task: Program CBB PCIE AMAP regions
I> Task: Boot device init
I> Boot_device: QSPI_FLASH instance: 0
I> QSPI-0l initialized successfully
I> Secondary storage device: QSPI_FLASH instance: 0
I> Secondary storage device: SDMMC_USER instance: 3
I> sdmmc HS400 mode enabled
I> Task: Partition Manager Init
I> strt_pg_num(1) num_of_pgs(1) read_buf(0x102e001000)
I> strt_pgm(131039) num_of_pgs(32) read_buf(0x102e001200)
I> Found 60 partitions in QSPI_FLASH (instance 0)
W> Cannot find any partition table for 00000003
W> PARTITION_MANAGER: Failed to publish partition.
I> Found 15 partitions in SDMMC_USER (instance 3)
I> Task: Pass DRAM ECC PRL Flag to FSI
I> Task: Load and authenticate registered FWs
I> Task: Load AUXP FWs
I> Successfully regisCE FW load task with MB2 loader
I> Successfully register DCE FW load task with MB2 loader
I> Unpowergating APE
I> Unpowergate done
I> Successfully register APE FW load task with MB2 loader
I> Skipping FSI FW load
I> Successfully register XUSB FW load task with MB2 loader
I> Successfully register PVA FW load task with MB2 loader
I> Partition name: A_spe-fw
I> Size of partition: 589824
I> Binary@ device:3/0 block-55040 (partition size: 0x90000), name: A_spe-fw
I> strt_pg_num(55040) num_of_pgs(16) read_buf(0x40067a30)
I> strt_pg_num(55056) num_of_pgs(512) read_buf(0x102d600000)
I> Partition name: A_rce-fw
I> Size of partition: 1048576
I> Binary@ device:3/0 block-56192 (partition size) read_buf(0x40067a30)
I> strt_pg_num(56208) num_of_pgs(880) reinary spe loaded successfully at 0x102d600000
I> Partition name: A_dce-fw
I> Size of partition: 5242880
I> Binary@ device:3/0_pg_num(44800) num_of_pgs(16) read_buf(0x40067a30)
I> rce: Authentication Finalize Done
I> Binary rce loaded successfully at 0x102d200000
I> Successfully register RCE FW context save task wtrt_pg_num(44816) num_of_pgs(1) read_buf(0x102e1403d8)
I> strt_a-blob integrity check is success.
I> strt_pg_num(44824) num_of_pgs(512) read_buf(0x102e0003c0)
I> strt_pg_num(45336) num_of_p 0x1036000000
I> version 1 Bin 1 BCheckSum 0 content_size 0 Conerved11 0
I> strt_pg_num(45848) num_of_pgs(512) read_buf(0x102e0803c0)
I> dce : decompressed to 12067600 bytes
I> dce: plain binary integrity check is success
I> Partition name: A_adsp-fw
I> Size of partition: 2097152
I> Binary@ device:strt_pg_num(58240) num_of_pgs(16) read_buf(0x40067a30)
I> strt_ 0x1036000000
I> Partition name: A_xusb-fw
I> Size of partition: 262144
I> Binary@ device:3/0 block-9472 (partition size: 0x40000), name: A_xusb-fw
I> strt_pg_num(9472) num_of_pgs(16) read_buf(0x40067a30)
I> strt_pg_num(9488) num_of_pgs(312) read_buf(0x102d700000)
I> ape: Authentication Finalize Done
I> Binary ape loaded successfully at 0x1038800000
I> Successfully register APE FW context save task with MB2 loader
I> Partition name: A_pva-fw
I> Size of partition: 262144
I> Binary@ device:3/0 block-62336 (partition size: 0x40000), name: A_pva-fw
I> strt_pg_num(62336) num_of_pgs(16) read_buf(0x40067a30)
I> xusb: Authentication Finalize Done
I> Binary xusb loaded successfully at 0x102d700000
I> Successfully register XUSB FW context save task with MB2 loader
I> pva-fw : oem authentication of header done
I> strt_pg_num(62352) num_of_pgs(1) read_buf(0x102e1403d8)
I> strt_pg_num(62352) num_of_pgs(8) read_buf(0x102e1403d8)
I> pva-fw : meta-blob integrity check is success.
I> strt_pg_num(62360) num_of_pgs(512) read_buf(0x102e0003c0)
I> pva-fw : will be decompressed at 0x102d980000
I> version 1 Bin 1 BCheckSum 0 content_size 0 Content ChkSum 1 reserved_00  0
I> Reserved10 0 BlockMaxSize 5 Reserved11 0
I> pva-fw : decompressed to 2156512 bytes
I> pva-fw: plain binary integrity check is success
I> pva-fw: Authentication Finalize Done
I> Binary pva-fw loaded successfully at 0x102d980000
I> Successfully register PVA FW context save task with MB2 loader
I> Task: Check MC errors
I> Task: Carveout setup
I> Program remaining OEM carveouts
I> Task: Enable FSITHERM
I> Task: Enable FSI VMON
I> FSI VMON: FSI Vmon re-calibration and fine tuning done
I> Task: Validate FSI Therm readings
I> Task: Restore XUSB sec
I> Task: Enable FSI SE clock
I> Enable FSI-SE clock...
I> Task: Initialize SBSA UART CAR
I> Task: Initialize CPUBL Params
I> CPUBL-params @ 0x1032000000
I> Task: Ratchet update
W> Skip ratchet update - OPTIN fuse not set
I> Task: Prepare eeprom data
I> Task: Revoke PKC fuse
I> PKC revoke fuse burn not requested
I> Task: FSI padctl context save
I> Task: Unpowergate APE
W> mb2_unpowergate_ape: skip! APE is in unpowergated state
I> Task: Memctrl reconfig pending clients
I> Task: OEM firewalls
I> OEM firewalls configured
I> Task: Powergate APE
I> Powergating APE
I> Powergate done
I> Task: OEM firewall restore saved settings
I> Task: Unhalt AUXPs
I> Unhalting SPE..
I> Enabling combined UART 
spe: early_init
vic initialized
tsc initialized
aon lic initialized
spe: tag is 524398t
scheduler initialized
aon hsp initialized
tag initialized
tcu initialized
bpmp ipc initialized
spe: late init
cpu_nic clock initialized
apb clock initialized
pm initialized
bpmp hsp initialized
top1 hsp initialized
ccplex ipc initialized
spe: start scheduler

I> Task: Trigger mailbox for PSC-BL1 exit
I> Sending opcode 0x4d420802 to psc
I> Received ACK from psc
I> Task: Start secure NOR provision
I> Skip Secure NOR provisioning
I> Task: Trigger load FSI keyblob
I> Skipping FSI key blob copy
I> Task: Complete load FSI keyblob
I> Skipping FSI key blob copy
I> Task: MB2-PSC_FW Key Manager Init
I> Sending opcode OP_PSC_KEY_MANAGER to psc-fw
I> Sending opcode 0x4b45594d t
hwwdt_init: WDT boot cfg 0x710010 sts 0x10
bpmp: socket 0
bpmp: base binary md5 is da583751bbfe2b7f6e204562d97ff39e
bpmp: combined binary md5 is 39f77b2baaf3f0522607569dd3ae9a48
bpmp: firmware tag is 39f77b2baaf3f0522607-da583751bbf
initialized vwdt
initialized mail_early
initialized fuse
initialized vfrel
initialized adc
fmon_populate_monitors: found 199 monitors
initialized fmon
initialized mc
initialized reset
initialized uphy_early
initialized emc_early
initialized pm
465 clocks registered
initialized clk_mach
initialized clk_cal_early
initialized clk_mach_early_config
initialized io_dpd
initialized soctherm
initialized regime
initialized i2c
vrmon_dt_init: vrmon node not found
vrmon_chk_boot_state: found 0 rail monitors
initialized vrmon
initialized regulator
o psc
I> Received ACK from psc
I> Task: Unhalt FSI
I> FSI unhalt skipped
I> Task: Unhalt AUXPs
I> Unhalting RCE
I> RCE unhalt successful
I> Unhalting DCE
I> DCE unhalt successful
I> APE unhalt skipped
I> Task: Load HV/CPUBL
I> Task: Load TOS
I> Task: Trigger load TS[     2.609271] Camera-FW on t234-rce-safe started
initialized avfs_clk_platform
initialized powergate
TCU early console enabled.
EC leyinitialized dvs
initialized clk_mach_config
suspend progress: 0x80000000
initialized suspend
initialized strap
initialized mce_dbell
blob

I> Sending opcode 0x53535452 to psc
I> Sent opcode to psc
I> Task: Load and authenticate registered FWs
I> Partition name: A_cpu-bootloader
I> Size of partition: 3670016
I> Binary@ device:3/0 block-24832 (partition size: 0x380000), name: A_cpu-bootloader
DCE Started
I> strt_pg_num(24832) num_of_pgs(16) read_buf(0x40067a30)
I> cpubl : oem authentication of header done
DCE_R5_Init
I> strt_pg_num(24848) num_of_pgs(1) read_buf(0x102e143f98)
I> strt_pg_num(24848) num_of_pgs(8) read_buf(0x102e143f98)
I> cpubl : meta-blob integrity check isinitialized emc
initialized emc_mrq
MPU enabled
 success.
I> strt_pg_num(24856)initialized clk_cal
initialized uphy_dt
initialized uphy_mrq
HSIO UPHY reset has been de-asserted 0x0
 num_oinitialized uphy
f_pgs(512) r
swdtimer_init: reg polling start w period 47 ms
initialized swdtimer
initialized hwwdt_late
initialized bwmgr
initialized thermal_host_trip
initialized thermal_mrq
initialized oc_mrq
initialized reset_mrq
initialized mail_mrq
initialized fmon_mrq
initialized clk_mrq
initialized avfs_mrq
initialized i2c_mrq
initialized tag_mrq
initialized bwmgr_mrq
initialized console_mrq
missing prod DT calibration data for 199 fmons
initialized clk_sync_fmon_post
DCE_SW_Init
80)
I> strt_pg_num(25368) num_of_pgs(512) read_buf(0x10initialized clk_cal_late
initialized noc_late
initialized cvc
2e043f80)
I> cpubl : will be decompreinitialized avfs_clk_mach_post
initialized avfs_clk_platform_post
initialized cvc_late
initialized rm
initialized console_late
handling unreferenced clks
enable can1_core
enable can1_host
enable can2_core
enable can2_host
enable pwm3
enable mss_encrypt
enable maud
enable pllg_ref
enable dsi_core
enable aza_2xbit
enable pllc4_muxed
enable sdmmc4_axicif
enable xusb_ss
enable xusb_fs
enable xusb_falcon
enable xusb_core_mux
enable dsi_lp
enable sdmmc_legacy_tm
initialized clk_mach_post
initialized pg_post
[     2.811175] Camera-FW on t234-rce-safe ready SHA1=e2238c99 (crt 12.427 ms, total boot 215.403 ms)
initialized regulator_post
initialized profile
initialized mrq
initialized patrol_scrubber
initialized cactmon
initialized extras_post
bpmp: init complete
ssed at 0x102c800000
I> version 1 Bin 1 BCheckSum 0 content_size 0 Content ChkSum 1 reserved_00  0
I> Reserved10 0 BlockMaxSize 5 Reserved11 0
I> strt_pg_num(25880) num_of_pgs(512) read_buf(0x102e083f80)
I> strt_pg_num(26392) num_of_pgs(512) read_buf(0x102e0c3f80)
I> strt_pg_num(26904) num_of_pgs(512) read_buf(0x102e103f80)
I> strt_pg_num(27416) num_of_pgs(512) read_buf(0x102e003f80)
I> strt_pg_num(27928) num_of_pgs(512) read_buf(0x102e043f80)
I> strt_pg_num(28440) num_of_pgs(512) read_buf(0x102e083f80)
I> strt_pg_num(28952) num_of_pgs(512) read_buf(0x102e0c3f80)
I> strt_pg_num(29464) num_of_pgs(512) read_buf(0x102e103f80)
I> strt_pg_num(29976) num_of_pgs(512) read_buf(0x102e003f80)
Admin Task Init
Admin Task Init complete
Print Task Init
RM Task Init
SHA Task Init
Admin Task Started
I> strt_pg_num(30488) num_of_pgs(512) read_buf(0x102e043f80)
DCE SC7 SHA Enabled
RM Task Started
RM Task Running
Print Task Started
Print Task Running
SHA Task Started
DCE: FW Boot Complete
Admin Task Running
Sf(0x102e083f80)
I> cpubl : decompressed to 3661952 bytes
I> cpubl: plain binary integrity check is success
I> Partition name: A_secure-os
I> Size of partition: 4194304
I> Binary@ device:3/0 block-32000 (partition size: 0x400000), name: A_secure-os
I> strt_pg_num(32000) num_of_pgs(16) read_buf(0x40067a30)
I> strt_pg_num(32016) num_of_pgs(3672) read_buf(0x103fd35000)
I> MB2-params @ 0x40060000
I> NSDRAM carveout base: 0x80000000, size: 0xfacdf0000
I> cpubl_params: nsdram: carveout: 1, encryption: 1
I> cpubl: Authentication Finalize Done
I> Binary cpubl loaded ne
I> Binary tos loaded successfully at 0x103fd35000
I> Relocating OP-TEE dtb from: 0x103feff180 to 0x103c040020, size: 0x2754
I> [0] START: 0x80000000, SIZE: 0xfacdf0000
I> [1] START: 0x1E dtb finished.
I> Partition name: A_eks
I> Size of partition: 262144
I> Binary@ device:3/0 block-44288 (partition size: 0x40000), name: A_eks
I> strt_pg_num(44288) num_of_pgs(16) read_bufc020000)
I> eks: Authentication Finalize Done
I> Binary eks lo0) @ VA:0x103c020000
I> Task: Add cpubl params integrity check
I> Added cpubl params digest.
I> Task: Prepare TOS params
I> Setting EKB blob info to OPTEE dtb finished.
I> Setting OPTEE arg3: 0x103c040020
I> NVRNG: Health check success
I> NVRNG: Health check success
I> Task: OEM SC7 context save
I> OEM sc7 context saved
I> Task: Disable MSS perf stats
I> Task: Program display sticky bits
I> Task: Storage device deinit
I> Task: SMMU init
I> Task: Program GICv3 registers
I> Task: Audit firewall settings
I> Task: Bootchain failure check
I> Current Boot-Chain Slot: 0
I> BR-BCT Boot-Chain is 1, and status is 1. Set UPDATE_BRBCT bit to 1
I> Task: Burn RESERVED_ODM0 fuse
I> Task: Lock fusing
I> Task: Clear dec source key
I> MB2 finished

NOTICE:  BL31: v2.8(release):e12e3fa93
NOTICE:  BL31: Built : 17:14:28, Jan  7 2025
I/TC: 
I/TC: Non-secure external DT found
I/TC: OP-TEE version: 4.2 (gcc version 11.3.0 (BuildrootTC: WARNING: This OP-TEE configuration might be insecure!
I/TC: WARNING: Please check https://optee.readthedocs.io/en/latest/architecture/porting_guidelines.html
I/TC: Primary CPU initializing
I/TC: Test OEM keys are being used. This is insecure for shipping products!
I/TC: fTPM ID is not enabled.
I/TC: ftpm-helper PTA: fTPM DT or EKB is not available. fTPM provisioning is not supported.
I/TC: Primary CPU switching to normal world boot

Jetson UEFI firmware (version 36.4.3-gcid-38968081 built on 2025-01-08T01:18:20+00:00)

I/TC: Reserved shared memory is disabled
I/TC: Dynamic shared memory is enabled
I/TC: Normal World virtualization support is disabled
I/TC: Asynchronous notifications are disabled
I/TC: WARNING: Test UEFI variable auth key is being used !
I/TC: WARNING: UEFI variable protection is not fully enabled !

[     6.143531] Camera-FW on t234-rce-safe started
TCU early console enabled.
[     6.205341] Camera-FW on t234-rce-safe ready SHA1=e2238c99 (crt 1.367 ms, total boot 63.240 ms)

3h

ASSERT [XhciDxe] /out/nvidia/bootloader/uefi/Jetson_RELEASE/edk2/MdeModulePkg/Bus/Pci/XhciDxe/XhciSched.c(3154): Interval >= 1 && Interval <= 16

Resetting the system in 5 seconds.
Shutdown state requested 1
Rebooting system ...
MMof crypto header is 8192
[0000.344] I> strt_pg_num(66816) num_of_pgs(16) read_buf(0x40050000)
[0000.350] I> BCH of MEM-BCT-0 read from storage
...

I/TC: Reserved shared memory is disabled
I/TC: Dynamic shared memory is enabled
I/TC: Normal World virtualization support is disabled
I/TC: Asynchronous notifications are disabled
I/TC: WARNING: Test UEFI variable auth key is being used !
I/TC: WARNING: UEFI variable protection is not fully enabled !
[     6.145784] Camera-FW on t234-rce-safe started
TCU early console enabled.
[     6.207503] Camera-FW on t234-rce-safe ready SHA1=e2238c99 (crt 1.357 ms, total boot 63.140 ms)
3h
ASSERT [XhciDxe] /out/nvidia/bootloader/uefi/Jetson_RELEASE/edk2/MdeModulePkg/Bus/Pci/XhciDxe/XhciSched.c(3154): Interval >= 1 && Interval <= 16

Resetting the system in 5 seconds.
Shutdown state requested 1
Rebooting system ...
```

通過以上訊息，可以看到出現了 ASSERT 之後卡住，根據名稱判斷這是 UEFI 裡面的 xHCI USB 驅動 (Xhci) 崩潰。

觸發 ASSERT 之後 5 秒強制重新開機，接著無限重新啟動，反映出的結果就是只看到黑畫面，永遠無法進入作業系統。從上面 Interval 判斷這是某個 USB 端點回傳了不合法輪詢的 Interval，或是被 UEFI 解析成非法的範圍，導致 ASSERT 觸發。

關於合法的 Interval，查閱 USB 2.0 標準了解對 Isochronous 的 bInterval 在 Full-/High-Speed 常見的歸範圍 1 到 16，表示 $2^{(n-1)}$ 的 microframe 週期，中斷在 Full/Low-Speed 可以達到 1 到 255。

詳細規格可以閱讀 USB 2.0 與 xHCI 1.1 的相關說明

所以上面這個程式碼很可能是在判斷插入的 USB 2.0 裝置是否符合標準。如果不符合標準則觸發 assertion

### 詳細解讀 boot log
根據 NVIDIA [官方文件](https://docs.nvidia.com/jetson/archives/r35.1/DeveloperGuide/text/AR/BootArchitecture/JetsonAgxOrinBootFlow.html#bootrom)，得知啟動流程如以下
- BootROM
- PSCROM
- MB1
- MB2
- UEFI

接著解讀以上啟動訊息

首先可以看到最上面有大量 MB1/MB2 的訊息，包含載入 BPMP_FW, PSC_FW, RCE/DCE/XUSB 等等，屬於 Jetson Boot 啟動流程。SDRAM 初始化，時脈溫度監控等等，沒有看到錯誤訊息，所以在 MB1/MB2 階段沒有發生問題。

接著看到一些警告，可能是來自 OP-TEE 的訊息，像是 Test UEFI variable，看起來像是開發用的測試 key 等等。

在 MB1/MB2 之後，接著進入到 Jetson UEFI，UEFI 階段會載入內建的 xHCI 驅動 (XhciDxe) 掃描鍵盤，隨身碟等等週邊裝置，可能是為了提供 pre-boot I/O 或是從 USB 開機，通過 ASSERT 資訊判斷這是使用 edk2 框架進行開發的 Bootloader，可以看到進入到 UEFI firmware 之後就會崩潰 (判斷這個崩潰應該是在 UEFI 很早期的情況發生的，所以導致我們開機連 BIOS 的畫面都沒有看到)
```
Jetson UEFI firmware (version 36.4.x ...)
ASSERT [XhciDxe] ... XhciSched.c(3154): Interval >= 1 && Interval <= 16
Resetting the system in 5 seconds.
```

我把這個 webcam 插到我的 fedora laptop 上，發現可以正常的開機，這邊我猜想 Linux Kernel 的 xHCI/USB driver 對 out-of-spac 的描述或許比較寬容，於是我查看 Linux Kernel xHCI/USB driver 的相關程式碼，希望把他移植到 edk2 UEFI 上，打上 patch，接著把 Jetson 上面的 Bootloader 抽換掉，解決這個問題。


### 關於 Linux Kernel xHCI/USB
位於 `drivers/usb/core/config.c`
```c
/*
	 * Fix up bInterval values outside the legal range.
	 * Use 10 or 8 ms if no proper value can be guessed.
	 */
	i = 0;		/* i = min, j = max, n = default */
	j = 255;
	if (usb_endpoint_xfer_int(d)) {
		i = 1;
		switch (udev->speed) {
		case USB_SPEED_SUPER_PLUS:
		case USB_SPEED_SUPER:
		case USB_SPEED_HIGH:
			/*
			 * Many device manufacturers are using full-speed
			 * bInterval values in high-speed interrupt endpoint
			 * descriptors. Try to fix those and fall back to an
			 * 8-ms default value otherwise.
			 */
			n = fls(d->bInterval*8);
			if (n == 0)
				n = 7;	/* 8 ms = 2^(7-1) uframes */
			j = 16;

			/*
			 * Adjust bInterval for quirked devices.
			 */
			/*
			 * This quirk fixes bIntervals reported in ms.
			 */
			if (udev->quirks & USB_QUIRK_LINEAR_FRAME_INTR_BINTERVAL) {
				n = clamp(fls(d->bInterval) + 3, i, j);
				i = j = n;
			}
			/*
			 * This quirk fixes bIntervals reported in
			 * linear microframes.
			 */
			if (udev->quirks & USB_QUIRK_LINEAR_UFRAME_INTR_BINTERVAL) {
				n = clamp(fls(d->bInterval), i, j);
				i = j = n;
			}
			break;
		default:		/* USB_SPEED_FULL or _LOW */
			/*
			 * For low-speed, 10 ms is the official minimum.
			 * But some "overclocked" devices might want faster
			 * polling so we'll allow it.
			 */
			n = 10;
			break;
		}
	} else if (usb_endpoint_xfer_isoc(d)) {
		i = 1;
		j = 16;
		switch (udev->speed) {
		case USB_SPEED_HIGH:
			n = 7;		/* 8 ms = 2^(7-1) uframes */
			break;
		default:		/* USB_SPEED_FULL */
			n = 4;		/* 8 ms = 2^(4-1) frames */
			break;
		}
	}
	if (d->bInterval < i || d->bInterval > j) {
		dev_notice(ddev, "config %d interface %d altsetting %d "
		    "endpoint 0x%X has an invalid bInterval %d, "
		    "changing to %d\n",
		    cfgno, inum, asnum,
		    d->bEndpointAddress, d->bInterval, n);
		endpoint->desc.bInterval = n;
	}

```
USB 的 `bInterval` 因為不同速度或傳輸型別而有不同的意義。而部份設備廠商可能會錯誤的填入資訊，例如把 FS 的語意用在 HS 或是 SS，或是錯誤的寫成 microsecond, microframe 而不是規格要求的 2 的冪次數。

上面的程式碼是為了容錯考慮，猜出一個正確值，或是給出一個合理的預設值，而不是直接報錯中止，如同 edk2 中做的那樣。

`i` 表示合法的下限值 (min)，`j` 表示合法的上限值 (max)，`n` 表示預設值或是推測之後應該要設置的值。

最下面的程式碼
```c
if (d->bInterval < i || d->bInterval > j) {
		dev_notice(ddev, "config %d interface %d altsetting %d "
		    "endpoint 0x%X has an invalid bInterval %d, "
		    "changing to %d\n",
		    cfgno, inum, asnum,
		    d->bEndpointAddress, d->bInterval, n);
		endpoint->desc.bInterval = n;
	}
```
如果超出 min 或是 max，則會將 `bInterval` 設置為 n。

所以目前的想法是希望可以放寬 Bootloader 的 `bInterval`，為此我們需要取得 Bootloader 的 source code，修改完成之後替換掉原先的 Bootloader。

### 環境建置
```shell
# sudo systemctl enable --now docker
# sudo groupadd -f docker
# sudo usermod -aG docker $USER
# newgrp docker 
```
接著按照 nvidia-edk2 裡面的 wiki 進行建置。

```shell
# Point to the Ubuntu-22 dev image
export EDK2_DEV_IMAGE="ghcr.io/tianocore/containers/ubuntu-22-dev:latest"

# Required
export EDK2_USER_ARGS="-v \"${HOME}\":\"${HOME}\" -e EDK2_DOCKER_USER_HOME=\"${HOME}\""

# Required, unless you want to build in your home directory.
# Change "/build" to be a suitable build root on your system.
export EDK2_BUILD_ROOT="/build"
export EDK2_BUILDROOT_ARGS="-v \"${EDK2_BUILD_ROOT}\":\"${EDK2_BUILD_ROOT}\""

# Create the alias
alias edk2_docker="docker run -it --rm -w \"\$(pwd)\" ${EDK2_BUILDROOT_ARGS} ${EDK2_USER_ARGS} \"${EDK2_DEV_IMAGE}\""
```

### 程式碼修改與編譯
接著修改程式碼，這邊先採用最小可行性方案，以 Linux Kernel 的作法整體概念是猜一個數字放入到 `bInterval`，這裡我們仿造類似的方式，直接使用以下

```diff
else if ((DeviceSpeed == EFI_USB_SPEED_HIGH) || (DeviceSpeed == EFI_USB_SPEED_SUPER)) {
          Interval = EpDesc->Interval;
-         ASSERT (Interval >= 1 && Interval <= 16);
+         if(Interval < 1 || Interval > 16) {
+         Interval = (Interval < 1) ? 1 : 16;
          }
```
強制讓 Interval 落於一個範圍內，先驗證這樣是否會造成問題，之後再將 Linux Kernel 的作法移入

==TODO: 移植使用 Linux Kernel 的方案==


修改完成之後執行以下開始編譯並產生出二進位檔案

```
edk2_docker edk2-nvidia/Platform/NVIDIA/Jetson/build.sh --init-defconfig edk2-nvidia/Platform/NVIDIA/Jetson/Jetson.defconfig
```
二進位檔案會存在於 `images` 資料夾中，在資料夾中可以看到許多 `.dtbo` 以及 `BOOTAA64_Jetson_RELEASE.efi` 和 `uefi_Jetson_RELEASE.bin`，很明顯 `.efi` 以及 `.bin` 是我們主要需要的兩個檔案，根據 [NVIDIA Jetson Linux Developer Guide](https://docs.nvidia.com/jetson/archives/r38.2/DeveloperGuide/RM/PackageManifest.html) 可以知道這一些檔案的用途
- `BOOTAA64.efi` 為 Jetson Linux OS Launcher UEFI Application，跟我們在 x86 會看到的 `BOOTX64.efi` 是類似的東西，放在 ESP (EFI System Partition) 上的 UEFI 應用程式，由 UEFI 韌體 (`uefi*.bin`) 呼叫，負責找到 Kernel, DTB, initrd 並把控制權交給 Linux。Jetson 預設 Loader 為 L4TLauncher
- `uefi*.bin` 為 Jetson 的 CPU BootLoader，為 UEFI firmware 的本體 (使用 EDK2 框架)，燒入在 Jetson 的 Bootloader Partition (在 Jetson 有 Slot A 以及 B，是為了在某一個 Bootloader 啟動失敗之後還能夠 rollback)。負責硬體初始化，載入各種 DXE 驅動，管理 UEFI NVRAM 變數，處理 Capsule 更新跟 A/B rollback，根據 Boot Order 去啟動 OS Loader，也就是上面說的 `BOOTAA64.efi` 會由 `uefi*.bin` 啟動

所以我們修改完 Xhci 相關程式碼之後，我們主要產生的是 `uefi*.bin`。

接著我們需要 L4T，裡面包含打包/燒錄/更新工具等等，能處理 Jetson 機種, Partition Table, A/B slot 等等細節，幫我們把做好的 UEFI 與週邊裝置，如 DTB, DTBO, loader 組織成正確的 Image 或是 Capsule，接著通過 Jetson 內部工具更新 Bootloader。

首先我們將 `uefi*.bin` 以及 `BOOTAA64.efi` 複製到 L4T 專案底下的 bootloader 資料夾中，接著開始打包

```shell
# sudo ./l4t_generate_soc_bup.sh -e t23x_agx_bl_spec t23x
```
使用以上指令產生 BUP (Bootloader Update Payload)

接著使用 BUP 產生 UEFI Capsule
```shell
# ./generate_capsule/l4t_generate_soc_capsule.sh \
    -i bootloader/payloads_t23x/bl_only_payload \
    -o TEGRA_BL.Cap t234
```

得到 `TEGRA_BL.Cap` 之後將其放到 Jetson AGX Orin 中。

在 Jetson AGX Orin 通過以下指令更新 Bootloader，新的 Bootloader 會被套用到非目前的 Slot，重開機之後我們會切換到該 Slot
```shell
# sudo nv_bootloader_capsule_updater.sh -q TEGRA_BL.Cap
```
在開機的時候可以看到畫面最上方的 firmware 日期更新成我們新的 Bootloader 日期。

而進入系統之後可以通過 `sudo nvbootctrl --dump-slots-info` 中 `Capsule update status` 的訊息，如果為 1 則表示更新成功。

經過以上更新之後，順利進入到系統，解決之前系統黑畫面的問題。但是隨之而來又有新的問題產生
```shell
[   29.317410] usb 1-4.4: Failed to query (GET_DEF) UVC control 2 on unit 1: -32 (exp. 1).
[   29.318661] usb 1-4.4: Failed to query (GET_DEF) UVC control 2 on unit 1: -32 (exp. 1).
[   29.319911] usb 1-4.4: Failed to query (GET_DEF) UVC control 2 on unit 1: -32 (exp. 1).
[   29.321161] usb 1-4.4: Failed to query (GET_DEF) UVC control 2 on unit 1: -32 (exp. 1).
[   29.322426] usb 1-4.4: Failed to query (GET_DEF) UVC control 2 on unit 1: -32 (exp. 1).
[   29.323664] usb 1-4.4: Failed to query (GET_DEF) UVC control 2 on unit 1: -32 (exp. 1).
...
```
可以看到在 dmesg 中有大量的錯誤訊息，關於這部份將在後續檢查 UVC Driver 排除問題。總之目前能夠順利啟動並進入系統了。


### 結論與整理
1. 發現按下開機鍵沒有反應，通過風扇聲音判斷可能是在進入到 OS 之前就 Crash 了，通過 Micro-USB Debuf UART 得到 boot log
2. 發現在 UEFI 階段 (XhciDxe) 在配置 USB 節點時，遇到裝置回傳不在範圍內的 `bInterval`，觸發 ASSERT，導致黑畫面以及卡住。這個範圍參考 USB 2.0 與 xHCI 1.1 相關描述
3. 修改 edk2-nvidia 的 `XhciSched.c` (不是 ASSERT 裡面的路徑，那個是建制該專案電腦上的路徑)，將 `bInterval` 強制落於該範圍
4. 將編譯完成的 UEFI 放入 L4T BSP，接著使用 `l4t_generate_soc_bup.sh` 產生 BUP，再用 `lr4_generate_soc_capsule.sh` 產生 `TEGRA_BL.Cap`
5. 在裝置上使用 `TEGRA_BL.Cap` 完成 Bootloader 更新



#### 參考資料，NVIDIA L4T 中的 README

```
# SPDX-FileCopyrightText: Copyright (c) 2023 NVIDIA CORPORATION & AFFILIATES. All rights reserved.
# SPDX-License-Identifier: LicenseRef-NvidiaProprietary
#
# NVIDIA CORPORATION, its affiliates and licensors retain all intellectual
# property and proprietary rights in and to this material, related
# documentation and any modifications thereto. Any use, reproduction,
# disclosure or distribution of this material and related documentation
# without an express license agreement from NVIDIA CORPORATION or
# its affiliates is strictly prohibited.

The NVIDIA Public release Package provides a tool to update the QSPI flash partitions
of Jetson devkits. This document provides information about updating the bootloader in
the QSPI flash.


1. Prerequisites:

   - This tool is in the Public Release Package. It must be extracted to the Jetson
     Linux Package work directory (${Your_path}/Linux_for_Tegra/).


2. Preparation.

   - Download the Jetson Linux Package and make it ready for flashing. The work
     directory is "${Your_path}/Linux_for_Tegra/".


3. Generate the QSPI flash bootloader payload.

   Here take the IGX as an example.

   a. Generate the BUP payload.
         $ cd ${Your_path}/Linux_for_Tegra/
         $ sudo ./l4t_generate_soc_bup.sh -e t23x_igx_bl_spec t23x

   b. Pack the generated BUP to the Capsule payload.
          $ ./generate_capsule/l4t_generate_soc_capsule.sh \
              -i ./bootloader/payloads_t23x/bl_only_payload \
              -o ./Tegra_IGX_BL.Cap t234

      Refer to the Jetson-Linux Developer Guide for more information about the Capsule payload.

4. Update the QSPI flash.

   a. Copy the generated QSPI flash payload to the target filesystem.
      $ scp ${Your_host_user_name}@${Your_host_IP}:${Your_path}/Linux_for_Tegra/Tegra_IGX_BL.Cap /opt

   b. Execute the bootloader_updater utility to update the IGX bootloader on QSPI flash.
      $ sudo nv_bootloader_capsule_updater.sh -q /opt/Tegra_IGX_BL.Cap

   c. Reboot the tareget to update the QSPI flash image on non-current slot bootloader.

   d. To check the Capsule update status, run the nvbootctrl command after boot to system:
      $ sudo nvbootctrl dump-slots-info

      Note: Capsule update status value "1" means update successfully. About the Capsule update status,
            please refer to developer guide for more information.

   e. To sync bootloader A/B slots, do the step b to d again.

   f. Then the bootloader partitions on QSPI flash(both A/B slots) are updated.

```

### 關於 DXE
DXE 為 Driver Execution Environment，為 UEPI/PI 架構中的一個開機階段，參考下圖
![image](https://hackmd.io/_uploads/r1z6qoacxg.png)
在 PEI 之後，BDS (Boot Device Selection) 之前。這個階段由 Dispatcher 將各種 DXE 驅動載入到記憶體，初始化平台上大部分硬體，建立 UEFI 的 Boot Services 與整個 protocol, handler 等等，最後把系統交給 BDS 挑選並啟動 boot loader。

以 `xhciDxe` 為例子，他是 DXE 驅動，負責在韌體時期把 xHCI USB 控制器 bring up，讓 UEFI 能夠使用 USB 鍵盤，從 USB 開機等等。

### Capsule
Capsule 為一種帶有標頭簽章的韌體更新包，作業系統把他交給 UEFI 的 `UpdateCapsule()`，UEFI 依照 FMP (Fireware Management Protocol) 驗證與寫入指定韌體區域，通常會在下一次開機執行。

Capsule 包含 header (Capsule GUID, size, flag)，後面加上 payload，多數平台使用 FMP Data Capsule，描述要更新的元件，版本以及 Image。UEFI 先使用 `QueryCapsuleCapabilities()` 檢查是否能處理，接著使用 `UpdateCapsule()` 申請更新。如果不使用 Jetson 裡面的腳本，或許也可以通過 `fwnpdtool` 進行 Bootloader 更新。

Capsule 就是軟體更新包，將要更新哪個元件，版本以及 Image 使用標準格式描述並簽章，交給 UEFI 的 FMP 在下次開機安全寫入目標韌體區域，在 Jetson 開機時可以看到寫入階段。而上面的 `TEGRA_BL.Cap` 就是用來更新 bootloader/UEFI 的 Capsule。


<style>
/* for code block（HackMD using .markdown-body container） */
.chroma pre {
  /* if line grater than 20, using rowing */
  max-height: 20lh;         
  overflow: auto;             /* 水平/垂直捲動 */
}

/* rollback：using em */
@supports not (max-height: 1lh) {
  .markdown-body pre {
    line-height: 1.4;       
    max-height: calc(20 * 1.4em);
  }
}

.markdown-body pre > code {
  display: block;
}

@media print {
  .markdown-body pre { max-height: none; overflow: visible; }
}
</style>
