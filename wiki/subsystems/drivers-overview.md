---
type: subsystem
kernel_path: drivers/
upstream: yes
android_patches: binder, vendor_hooks, debug_kinfo
related:
  - driver-framework.md
  - ../concepts/driver-model.md
  - ../concepts/gki.md
  - ../concepts/vendor-hooks.md
  - ../concepts/module-system.md
last_updated: 2026-04-10
---

# Drivers 總覽（Device Drivers Overview）

## Purpose

`drivers/` 是 Linux 核心中規模最大的頂層目錄，包含所有硬體裝置驅動程式與子系統框架。在 Android Common Kernel（ACK, common-android-mainline）中，此目錄包含 **145 個子目錄**、**33,299 個 .c/.h 原始檔案**，總計約 **1,773 萬行 C 程式碼**、**1,351 個 Kconfig 組態檔** 以及 **1,726 個 Makefile**。幾乎所有硬體抽象——從 GPU 繪圖、網路通訊到儲存、感測器、電源管理——都在此目錄中實作。

## Evidence Snapshot

| Claim | Source anchor |
|-------|---------------|
| `drivers/` 是頂層 Device Drivers 選單，按 bus/framework/driver families 分層 include Kconfig | `common/drivers/Kconfig:2-28`, `common/drivers/Kconfig:42-63` |
| build order 由 `drivers/Makefile` 控制，早期初始化框架（cache/irqchip/bus/phy/gpio/pci/clk/soc/virtio 等）先進入 | `common/drivers/Makefile:9-49` |
| GPU/IOMMU/USB/input/media 等主要 driver families 由條件式 `obj-$(CONFIG_*)` 納入 | `common/drivers/Makefile:64-120` |
| Android 專屬 driver family 集中在 `drivers/android/`，其 Kconfig 定義 Binder、BinderFS、Vendor Hooks、Debug Kinfo 與 KABI padding | `common/drivers/android/Kconfig:4-120` |

## 統計摘要

| 指標 | 數值 |
|------|------|
| 子目錄數 | 145 |
| .c / .h 檔案 | 33,299 |
| C 程式碼行數 | ~17,730,000 |
| Kconfig 組態檔 | 1,351 |
| Makefile | 1,726 |

---

## 子系統分類總覽

以下依功能類別列出所有 145 個驅動子目錄，並標註每個子系統的 .c/.h 檔案數量與簡要用途說明。

### 一、圖形與顯示（Graphics & Display）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `gpu/` | 7,250 | GPU 驅動框架，包含 DRM/KMS 核心、amdgpu、i915、nouveau、virtio-gpu、mali（Panthor/panfrost）、nova-core 等 |
| `video/` | 481 | 傳統 framebuffer（fbdev）驅動與背光控制 |
| `accel/` | 532 | 計算加速器驅動（habanalabs/Gaudi 等 AI 加速裝置） |

### 二、網路（Networking）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `net/` | 6,086 | 網路介面卡驅動：以太網、WiFi（ath/iwlwifi/brcm）、行動網路、虛擬網路（veth/tun/bonding） |
| `bluetooth/` | 49 | 藍牙 HCI 驅動（USB/UART/SDIO 傳輸層） |
| `nfc/` | 61 | 近場通訊（NFC）驅動框架與晶片驅動 |
| `infiniband/` | 579 | RDMA / InfiniBand 高速互連驅動 |
| `atm/` | 26 | 非同步傳輸模式（ATM）網路驅動 |
| `isdn/` | 55 | ISDN 數位電話網路驅動 |

### 三、多媒體（Multimedia）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `media/` | 2,618 | V4L2 視訊框架、DVB 數位電視、攝影機感測器、視訊編解碼器、CEC |
| `input/` | 463 | 輸入裝置驅動：觸控螢幕、鍵盤、滑鼠、遊戲手柄、感測器按鍵 |
| `hid/` | 281 | USB/BT HID（Human Interface Device）協定驅動 |
| `accessibility/` | 40 | 無障礙輔助驅動 |

### 四、儲存（Storage）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `block/` | 91 | 區塊裝置驅動（loop、nbd、null_blk、zram 等） |
| `scsi/` | 768 | SCSI 子系統與驅動（含 UFS HCI — Android 主流儲存） |
| `ufs/` | 45 | UFS（Universal Flash Storage）核心驅動，Android 手機主要儲存介面 |
| `nvme/` | 46 | NVMe SSD 驅動（host/target） |
| `mmc/` | 170 | MMC/SD/SDIO 卡片驅動，Android 外接儲存與 WiFi/BT SDIO |
| `ata/` | 138 | SATA/PATA 磁碟控制器驅動 |
| `md/` | 289 | 軟體 RAID（md）與 Device Mapper（dm，含 dm-verity — Android 驗證啟動核心） |
| `mtd/` | 291 | Memory Technology Devices（NOR/NAND Flash）驅動 |
| `nvdimm/` | 29 | 非揮發性記憶體（NVDIMM/PMEM）驅動 |
| `cdrom/` | 2 | CD-ROM 裝置驅動 |
| `memstick/` | 9 | Memory Stick 儲存卡驅動 |
| `target/` | 79 | SCSI Target（iSCSI/FC target）框架 |

### 五、匯流排與互連（Buses & Interconnects）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `pci/` | 235 | PCI/PCIe 匯流排核心與主控制器驅動 |
| `usb/` | 789 | USB 核心堆疊、xHCI/EHCI/OHCI 主控制器、Gadget 框架、USB Type-C |
| `i2c/` | 185 | I²C 匯流排框架與適配器驅動 |
| `spi/` | 177 | SPI 匯流排框架與控制器驅動 |
| `i3c/` | 27 | I3C 匯流排（I²C 後繼）框架 |
| `bus/` | 61 | 通用匯流排驅動（MHI — Qualcomm modem 介面等） |
| `interconnect/` | 68 | SoC 內部互連（NoC）頻寬管理框架 |
| `amba/` | 2 | ARM AMBA（APB/AHB）匯流排支援 |
| `pcmcia/` | 48 | PCMCIA/CardBus 舊式介面卡 |
| `rapidio/` | 14 | RapidIO 互連標準 |
| `ssb/` | 20 | Sonics Silicon Backplane（Broadcom 早期 SoC） |
| `bcma/` | 20 | Broadcom AMBA（新版 Broadcom SoC 匯流排） |
| `firewire/` | 23 | IEEE 1394 FireWire 驅動 |
| `thunderbolt/` | 36 | Thunderbolt/USB4 驅動 |
| `cxl/` | 29 | Compute Express Link（CXL）記憶體擴展 |
| `cdx/` | 11 | AMD CDX 匯流排驅動 |
| `dpll/` | 27 | 數位鎖相迴路（DPLL）子系統 |
| `eisa/` | 3 | EISA 匯流排（舊式 PC） |
| `ntb/` | 20 | Non-Transparent Bridge（多主機互連） |
| `vdpa/` | 34 | vDPA（Virtio 資料路徑加速）框架 |
| `most/` | 5 | MOST（Media Oriented Systems Transport）匯流排 |
| `slimbus/` | 6 | SLIMbus 音訊匯流排 |
| `soundwire/` | 30 | SoundWire 音訊匯流排 |
| `spmi/` | 6 | SPMI（System Power Management Interface — Qualcomm PMIC） |
| `greybus/` | 16 | Greybus（Project Ara 模組化手機）協定 |

### 六、電源與時脈管理（Power & Clock Management）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `clk/` | 1,278 | 時脈框架（CCF）與 SoC 時脈驅動（qcom/mediatek/samsung/exynos 等） |
| `regulator/` | 230 | 電壓/電流穩壓器框架與 PMIC 驅動 |
| `power/` | 179 | 電源供應類別（battery/charger）與系統電源管理 |
| `cpufreq/` | 92 | CPU 動態頻率調節（含 schedutil — Android 預設 governor） |
| `cpuidle/` | 36 | CPU 閒置狀態管理 |
| `devfreq/` | 20 | 裝置頻率/電壓動態調整（GPU/DDR DVFS） |
| `opp/` | 6 | Operating Performance Points（OPP）頻率/電壓對應表 |
| `pmdomain/` | 84 | 電源域（Power Domain）管理框架 |
| `powercap/` | 10 | 功率上限（Power Capping）框架 |

### 七、核心平台框架（Core Platform Frameworks）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `base/` | 95 | 裝置驅動核心框架：Bus-Device-Driver 模型、devres、fw_devlink（[詳見](driver-framework.md)） |
| `of/` | 23 | Device Tree（Open Firmware）解析框架 |
| `firmware/` | 201 | 韌體載入框架、ARM SCMI/SCPI、PSCI、EFI、DMI |
| `irqchip/` | 155 | 中斷控制器驅動（GIC/GICv3/GICv4 — ARM 主流） |
| `clocksource/` | 98 | 計時器與時脈源驅動（ARM arch_timer 等） |
| `iommu/` | 124 | IOMMU 框架與驅動（ARM SMMU — Android DMA 隔離核心） |
| `dma/` | 206 | DMA 引擎框架與控制器驅動 |
| `dma-buf/` | 25 | DMA-BUF 共享緩衝區框架（Android 圖形管線核心） |
| `reset/` | 61 | 硬體重置控制器框架 |
| `cache/` | 4 | 快取控制器驅動 |
| `platform/` | 391 | 平台特定驅動（x86/Chrome/Mellanox/Goldfish 等） |
| `soc/` | 216 | SoC 廠商專屬驅動（Qualcomm/MediaTek/Samsung/Google） |

### 八、安全與可信執行（Security & TEE）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `crypto/` | 527 | 硬體加密加速器驅動（CryptoCell/QCE/CAAM 等） |
| `tee/` | 37 | 可信執行環境框架（OP-TEE / TrustZone） |
| `virt/` | 29 | 虛擬化相關驅動（SEV/TDX 等） |
| `vfio/` | 60 | VFIO 裝置直通（用於虛擬機器裝置分配） |
| `vhost/` | 10 | Vhost 核心態 virtio 後端 |
| `virtio/` | 22 | Virtio 虛擬裝置框架 |

### 九、感測器與周邊（Sensors & Peripherals）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `iio/` | 750 | 工業 I/O 框架：加速度計、陀螺儀、光感、溫度、ADC/DAC 等感測器 |
| `hwmon/` | 317 | 硬體監控（溫度/風扇/電壓感測器） |
| `thermal/` | 134 | 熱管理框架（thermal zones/trip points/cooling devices） |
| `leds/` | 147 | LED 子系統與觸發器驅動 |
| `gpio/` | 220 | GPIO 框架與晶片驅動 |
| `pinctrl/` | 554 | 腳位複用（Pin Mux）框架與 SoC 驅動 |
| `phy/` | 274 | 通用 PHY 框架（USB PHY / PCIe PHY / MIPI 等） |
| `pwm/` | 79 | PWM（脈寬調變）框架 |
| `rtc/` | 193 | 即時時鐘（RTC）驅動 |
| `watchdog/` | 193 | 看門狗計時器驅動 |
| `extcon/` | 29 | 外部連接器偵測（USB/耳機插拔） |
| `gnss/` | 7 | GNSS（GPS/GLONASS/Galileo）接收器驅動 |
| `pps/` | 14 | Pulse-Per-Second 精密時間驅動 |
| `ptp/` | 25 | IEEE 1588 PTP 精密時間協定 |
| `counter/` | 16 | 計數器子系統 |
| `w1/` | 35 | 1-Wire 匯流排驅動 |
| `hte/` | 3 | Hardware Timestamp Engine |

### 十、字元裝置與雜項（Character Devices & Misc）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `char/` | 209 | 字元裝置驅動（/dev/mem, /dev/random, TPM 等） |
| `misc/` | 246 | 雜項驅動（含各類無法歸類的裝置驅動） |
| `mfd/` | 272 | 多功能裝置（Multi-Function Device）框架與 PMIC 核心驅動 |
| `tty/` | 217 | 終端/序列埠驅動（UART/serial） |
| `uio/` | 13 | 使用者空間 I/O 框架 |
| `connector/` | 3 | Netlink connector 驅動 |
| `auxdisplay/` | 17 | 輔助顯示器（LCD 字元面板等） |
| `parport/` | 17 | 並列埠驅動 |

### 十一、Android 專屬驅動（Android-Specific）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `android/` | 20 | **Binder IPC** 驅動（Android 核心 IPC 機制）、binderfs、debug_kinfo、vendor_hooks |

**Android Binder** 是 Android 作業系統最關鍵的核心驅動，`binder.c` 單一檔案超過 21 萬行（含詳盡的交易處理、記憶體管理、死亡通知機制）。`vendor_hooks.c` 提供廠商擴展注入點。`debug_kinfo.c` 提供核心除錯資訊匯出。

### 十二、FPGA 與加速器（FPGA & Accelerators）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `fpga/` | 51 | FPGA Manager 框架與驅動 |
| `dax/` | 10 | Direct Access（DAX）持久記憶體直接存取 |
| `perf/` | 54 | 效能監控單元（PMU）驅動 |
| `hwtracing/` | 83 | 硬體追蹤驅動（ARM CoreSight、Intel PT、STM） |

### 十三、韌體與記憶體技術（Firmware & Memory Tech）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `nvmem/` | 51 | Non-Volatile Memory 框架（eFuse/OTP/EEPROM 讀取） |
| `remoteproc/` | 49 | Remote Processor 框架（DSP/MCU/modem 韌體載入與管理） |
| `rpmsg/` | 15 | Remote Processor Messaging（核心間通訊） |
| `mailbox/` | 43 | 硬體信箱（處理器間通訊機制） |
| `memory/` | 52 | 記憶體控制器驅動（DDR 頻寬監控等） |
| `edac/` | 79 | Error Detection And Correction（記憶體 ECC 錯誤報告） |

### 十四、雜項與平台特定（Misc & Platform-Specific）

| 子目錄 | 檔案數 | 說明 |
|--------|--------|------|
| `staging/` | 1,065 | 暫存區：尚未成熟的驅動（可能在未來被整合或移除） |
| `comedi/` | 193 | 資料擷取卡驅動（測量儀器用） |
| `gpib/` | 46 | IEEE 488 GPIB 儀器匯流排 |
| `hv/` | 32 | Microsoft Hyper-V 客戶端驅動 |
| `xen/` | 68 | Xen 虛擬化半虛擬驅動 |
| `s390/` | 228 | IBM System z（s390）專屬驅動 |
| `parisc/` | 21 | HP PA-RISC 專屬驅動 |
| `sh/` | 14 | SuperH 處理器專屬驅動 |
| `macintosh/` | 47 | Apple Macintosh 專屬驅動 |
| `ps3/` | 8 | PlayStation 3 專屬驅動 |
| `sbus/` | 10 | SPARC SBus 匯流排驅動 |
| `dio/` | 3 | HP 9000/300 DIO 匯流排 |
| `zorro/` | 7 | Amiga Zorro 匯流排 |
| `nubus/` | 3 | Apple NuBus 匯流排 |
| `tc/` | 2 | DEC TURBOchannel |
| `pnp/` | 21 | 隨插即用（Plug and Play）驅動 |
| `peci/` | 8 | Intel PECI 溫度讀取介面 |
| `hwspinlock/` | 8 | 硬體自旋鎖（多處理器同步） |
| `fsi/` | 14 | IBM FSI（FRU Service Interface） |
| `mcb/` | 5 | MEN Chameleon Bus |
| `siox/` | 3 | Eckelmann SIO 匯流排 |
| `fwctl/` | 3 | 韌體控制介面 |
| `hsi/` | 11 | 高速同步序列介面 |
| `ipack/` | 6 | IndustryPack 載板驅動 |
| `ras/` | 15 | Reliability/Availability/Serviceability 框架 |
| `resctrl/` | 3 | 資源控制（Intel RDT/ARM MPAM） |
| `mux/` | 5 | 多工器（Mux）框架 |
| `dibs/` | 3 | DIBS（Display Independent Bitmap Stream） |
| `idle/` | 1 | Intel idle 驅動 |
| `dca/` | 2 | Direct Cache Access |
| `message/` | 26 | Fusion MPT 訊息傳遞驅動 |

---

## Android 關鍵驅動子系統

以下是對 Android 裝置運作最為關鍵的驅動子系統：

1. **`android/`（Binder IPC）** — Android 程序間通訊的基石，所有 App ↔ System Service 通訊都經過 Binder
2. **`gpu/`（DRM/KMS）** — 圖形渲染管線，SurfaceFlinger 依賴 DRM 框架進行顯示合成
3. **`dma-buf/`** — 跨裝置緩衝區共享，圖形/相機/影片管線的記憶體共享核心
4. **`iommu/`（ARM SMMU）** — 裝置記憶體隔離，Android 安全模型的硬體基礎
5. **`tee/`（OP-TEE）** — 可信執行環境，金鑰管理（Keymaster/KeyMint）與 DRM 解密
6. **`clk/` + `regulator/` + `cpufreq/` + `devfreq/`** — 電源管理堆疊，影響續航與效能
7. **`mmc/` + `ufs/`** — 儲存驅動，Android 裝置的主要儲存介面
8. **`usb/`** — USB 連接：充電、OTG、MTP、ADB 等
9. **`input/`** — 觸控螢幕、按鍵與感測器輸入
10. **`iio/`** — 加速度計、陀螺儀、光感等感測器

---

## 檔案規模排名（前 20 大子系統）

| 排名 | 子目錄 | .c/.h 檔案數 |
|------|--------|-------------|
| 1 | `gpu/` | 7,250 |
| 2 | `net/` | 6,086 |
| 3 | `media/` | 2,618 |
| 4 | `clk/` | 1,278 |
| 5 | `staging/` | 1,065 |
| 6 | `usb/` | 789 |
| 7 | `scsi/` | 768 |
| 8 | `iio/` | 750 |
| 9 | `infiniband/` | 579 |
| 10 | `pinctrl/` | 554 |
| 11 | `accel/` | 532 |
| 12 | `crypto/` | 527 |
| 13 | `video/` | 481 |
| 14 | `input/` | 463 |
| 15 | `platform/` | 391 |
| 16 | `acpi/` | 338 |
| 17 | `hwmon/` | 317 |
| 18 | `mtd/` | 291 |
| 19 | `md/` | 289 |
| 20 | `hid/` | 281 |

---

## 與其他 Wiki 文件的關聯

- **[Driver Framework](driver-framework.md)** — `drivers/base/` 的詳細分析，Bus-Device-Driver 三層模型
- **[Block Layer](block-layer.md)** — 區塊 I/O 子系統（與 `drivers/block/`、`drivers/scsi/`、`drivers/ufs/` 相關）
- **[Networking](networking.md)** — 網路堆疊核心（與 `drivers/net/` 相關）
- **[GKI](../concepts/gki.md)** — Generic Kernel Image 架構，決定哪些驅動編譯進 GKI、哪些為可載入模組
- **[Vendor Hooks](../concepts/vendor-hooks.md)** — 廠商擴展機制，允許 OEM 在不修改核心程式碼下注入功能
- **[Driver Model](../concepts/driver-model.md)** — 裝置驅動模型概念說明
