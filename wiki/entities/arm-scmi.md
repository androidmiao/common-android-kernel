---
type: entity
kernel_path: drivers/firmware/arm_scmi/
config_option: CONFIG_ARM_SCMI_PROTOCOL
upstream: yes
related:
  - ../subsystems/firmware.md
  - ../entities/google-acpm.md
  - ../entities/qualcomm-firmware-stack.md
  - ../analyses/scmi-vs-google-acpm.md
  - ../sources/src-drivers-firmware-arm-scmi.md
  - ../concepts/driver-model.md
  - ../concepts/interrupt-handling.md
last_updated: 2026-04-20
---

# ARM SCMI (System Control and Management Interface)

## Overview

ARM System Control and Management Interface（SCMI）是 ARM 定義的一套**作業系統無關**的訊息協定，位於 `drivers/firmware/arm_scmi/`，共 **31 個 .c/.h 檔、~19,481 行**。SCMI 規範 AP（Application Processor，跑 Linux）與 SCP（System Control Processor，常為 SoC 內的 Cortex-M3/M4 小核）之間的通訊格式，讓 Linux 能以標準化方式請求 SCP 提供的電源域、時脈、DVFS 效能、感測器、重設、電壓、pinctrl 等系統管理服務。

SCMI 是 ARM 生態系統中**取代 SCPI**（舊版協定）的新世代標準，使用者包含 NXP i.MX95、STMicroelectronics STM32MP、Allwinner、Rockchip、AMD Versal、Amazon Graviton、部分 Qualcomm 平台等。**Google Tensor（GS101）並未採用 SCMI**，詳見 [google-acpm.md](./google-acpm.md) 與 [scmi-vs-google-acpm.md 分析](../analyses/scmi-vs-google-acpm.md)。

## Source Layout

```
drivers/firmware/arm_scmi/
├── Kconfig / Makefile
├── driver.c              # 核心：xfer 生命週期、protocol 註冊、SCMI bus (~3,500 行)
├── bus.c                 # SCMI device/driver model
├── common.h              # 內部介面 (xfer, protocol_info, transport_ops)
├── protocols.h           # 每個 protocol 共用定義
├── msg.c                 # inline message payload 封裝
├── shmem.c               # A2P/P2A 共享記憶體 payload
├── notify.c / notify.h   # 非同步通知框架
├── raw_mode.c / .h       # debugfs 注入 (compliance 測試)
├── quirks.c / quirks.h   # per-SoC workaround 框架
│
├── base.c                # Protocol 0x10 — 能力探測、agent discovery
├── power.c               # Protocol 0x11 — Power Domain
├── system.c              # Protocol 0x12 — System Power
├── perf.c                # Protocol 0x13 — Performance (DVFS)
├── clock.c               # Protocol 0x14 — Clock
├── sensors.c             # Protocol 0x15 — Sensor
├── reset.c               # Protocol 0x16 — Reset
├── voltage.c             # Protocol 0x17 — Voltage
├── powercap.c            # Protocol 0x18 — Powercap
├── pinctrl.c             # Protocol 0x19 — Pinctrl
│
├── scmi_power_control.c  # Consumer：System Power notifications → reboot/shutdown
│
├── transports/
│   ├── mailbox.c         # MHU doorbell + shared memory
│   ├── smc.c             # SMC call 進入 EL3 的 SCMI server
│   ├── optee.c           # OP-TEE 中轉
│   └── virtio.c          # virtio-scmi（虛擬化場景）
│
└── vendors/
    └── imx/              # NXP i.MX BBM/CPU/LMM/MISC vendor protocols
```

目前 `vendors/` 底下**只有 `imx`** 一家 vendor 擴充 upstream（`vendors/imx/Kconfig`, NXP i.MX95 等）；其他 SoC 廠商若要加入私有 protocol，依循相同結構即可。

## Implementation Details

### Protocol 分層

每個 SCMI protocol 編號對應一個 .c 檔，各自透過 `scmi_protocol_register()` 註冊到核心的 `scmi_protocols` XArray：

| ID | Protocol | 檔案 | 主要功能 |
|----|----------|------|----------|
| 0x10 | Base | `base.c` | 能力探測、版本、agent discovery |
| 0x11 | Power Domain | `power.c` | 電源域狀態切換 |
| 0x12 | System Power | `system.c` | 系統級 shutdown/reboot 通知 |
| 0x13 | Performance | `perf.c` | DVFS domain、opp level、limit |
| 0x14 | Clock | `clock.c` | 時脈速率查詢/設定 |
| 0x15 | Sensor | `sensors.c` | 感測器讀值、閾值通知 |
| 0x16 | Reset | `reset.c` | Reset domain 控制 |
| 0x17 | Voltage | `voltage.c` | 電壓域控制 |
| 0x18 | Powercap | `powercap.c` | 功耗上限（RAPL 類比） |
| 0x19 | Pinctrl | `pinctrl.c` | 引腳配置 |

協定版本協商由 Base protocol 負責；SCP 回報 protocol list 與各 protocol 的 minor version，AP 端據此啟用相容的 ops。

### Xfer 生命週期（`driver.c`）

SCMI 傳輸以 `struct scmi_xfer` 為單位：

- `struct scmi_xfers_info` 管理每個 transport 的 in-flight 訊息池，包含 `xfer_alloc_table` bitmap、`free_xfers` list、`pending_xfers` hashtable、`max_msg`。
- 每個 xfer 分配到 8-bit `msg_hdr.seq` 作為 SCP 回覆時的比對鍵。
- `do_xfer()` / `do_xfer_with_response()` 負責送出訊息並等待（polling 或 IRQ），`scmi_rx_callback()` 處理 SCP 回覆。
- Delayed response（SCP 事後主動回覆）透過 `pending_xfers` hashtable 查找並 wake up 等待者。

### Transport 抽象（`common.h` + `transports/`）

核心 driver 對傳輸層透明，每種傳輸實作 `struct scmi_transport_ops`：

**mailbox**（`transports/mailbox.c`）：最常見的 ARM reference design 方式。AP 把訊息寫入 shmem（`shmem.c` 包裝），然後 ring 一個 MHU doorbell，SCP 用另一個 MHU 通道回覆。

**smc**（`transports/smc.c`）：把 SCMI 訊息包進一個 SMC call，由 EL3 firmware（ATF）承載 SCMI server；適合沒有專用 SCP 的裝置。

**optee**（`transports/optee.c`）：透過 OP-TEE TA 中繼，SCMI server 跑在 TEE（Secure EL1）。

**virtio**（`transports/virtio.c`）：virtio-scmi device，虛擬化 guest 透過 VirtIO queue 與 host 端 SCMI server 通訊。pKVM、ACRN、KVM 場景會用到。

### SCMI Bus Model（`bus.c`）

`bus.c` 實作一條獨立的 `scmi_bus_type`。每個 protocol handle 註冊為 scmi_device，用 `scmi_device_id` 匹配 scmi_driver（例如 `scmi-cpufreq`、`scmi-hwmon`、`scmi-clk`、`scmi-regulator` 等 consumer driver）。Consumer 不直接觸碰 transport，而是透過 `scmi_handle->devm_protocol_get()` 拿到每個 protocol 的 ops 表。

### Notification 子系統（`notify.c`）

SCP 可主動發送非同步事件（效能變更、感測器跨越閾值、系統電源事件），`notify.c` 實作訂閱框架：

- Consumer 透過 `scmi_register_notifier()` 訂閱 event（protocol id + event id + src id）。
- SCP 事件經 transport 的 rx 回呼路徑進入 `scmi_notify()`，查找 `REGISTERED_EVENTS_NOTIFIERS` 並以 notifier chain 分派。
- 典型用戶：thermal framework 訂閱 sensor event、cpufreq 訂閱 perf limit change、`scmi_power_control.c` 訂閱 system shutdown event。

### Raw Mode（`raw_mode.c`）

`ARM_SCMI_RAW_MODE_SUPPORT` 提供 debugfs 介面（`/sys/kernel/debug/scmi/0/raw/`）直接注入/讀取 bare SCMI message。用於 ARM 提供的 **SCMI Compliance Test Suite**。`ARM_SCMI_RAW_MODE_SUPPORT_COEX` 允許與正常 SCMI stack 共存（雖可能影響測試可靠性）。

### Quirks Framework（`quirks.c`）

`ARM_SCMI_QUIRKS` 啟用時，以 jump label 為基礎可依 SCMI firmware 版本或 machine compatible 套用 workaround（例如某版 ATF SCMI server 的 bug）。避免把 workaround 散布在 protocol code。

### Debug Counters

`ARM_SCMI_DEBUG_COUNTERS` 在 debugfs 下建立計數器目錄，記錄送出/接收訊息數、失敗次數與失敗類別，對 SCMI 韌體除錯與監控很有幫助。

## Userspace Interface

SCMI 本身是 kernel-internal 的 firmware interface，userspace 不直接與 SCMI 對話，而是透過 consumer driver 暴露的標準介面：

- cpufreq sysfs（`scmi-cpufreq`）
- thermal sysfs / hwmon sysfs（`scmi-hwmon`）
- clk 與 regulator framework（經由 DT phandle）
- `/sys/kernel/debug/scmi/`（當 `ARM_SCMI_RAW_MODE_SUPPORT` 或 `ARM_SCMI_DEBUG_COUNTERS` 啟用）

## Android-Specific Notes

SCMI driver 本身**完全是 upstream Linux、無 `ANDROID:` 補丁、無 vendor hook**。對 Android GKI 的意義：

- 可作為 GKI module 編譯（`CONFIG_ARM_SCMI_PROTOCOL=m`），在 vendor partition 出貨。
- Android Virtualization Framework（AVF）/ pKVM 上的 protected VM 可用 **virtio-scmi** 取得受限的電源/時脈管理能力。
- 並非所有 Android SoC 都採用 SCMI：Qualcomm 走自家 RPMh/AOSS/SPM/SCM 多層堆疊（詳見 [qualcomm-firmware-stack.md](./qualcomm-firmware-stack.md)），僅 Snapdragon X Elite 的 CPUCP 選擇性採用 SCMI Perf protocol；Samsung/Google 走 ACPM、MediaTek 早期用 SCPI 現漸進轉向、NVIDIA Tegra 走 BPMP。

## Configuration

| Kconfig | 類型 | 說明 |
|---------|------|------|
| `ARM_SCMI_PROTOCOL` | tristate | 核心 |
| `ARM_SCMI_RAW_MODE_SUPPORT` | bool | compliance 測試用 raw 介面 |
| `ARM_SCMI_RAW_MODE_SUPPORT_COEX` | bool | 允許 raw 與正常 stack 並存 |
| `ARM_SCMI_DEBUG_COUNTERS` | bool | 通訊度量 |
| `ARM_SCMI_QUIRKS` | bool | per-SoC workaround framework |
| `ARM_SCMI_NEED_DEBUGFS` | bool | 隱式由以上啟用 |
| `ARM_SCMI_POWER_CONTROL` | tristate | System Power notification → reboot/shutdown |
| `ARM_SCMI_TRANSPORT_MAILBOX` | bool | Mailbox transport |
| `ARM_SCMI_TRANSPORT_SMC` | bool | SMC transport |
| `ARM_SCMI_TRANSPORT_OPTEE` | bool | OP-TEE transport |
| `ARM_SCMI_TRANSPORT_VIRTIO` | bool | virtio-scmi transport |
| `IMX_SCMI_*` | bool | i.MX vendor protocols |

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md) — drivers/firmware/ 整體概覽
- [Google ACPM](./google-acpm.md) — Google Tensor 的等價替代方案
- [Qualcomm Firmware Stack](./qualcomm-firmware-stack.md) — Snapdragon 的 RPMh/AOSS/SPM/SCM 多層平行協定堆疊，以及 X Elite 的 CPUCP+SCMI 混合路線
- [AP↔SCP 韌體介面三方對照](../analyses/scmi-vs-google-acpm.md) — ARM SCMI vs Google ACPM vs Qualcomm RPMh/CPUCP 三方比較
- [ARM SCMI Source Summary](../sources/src-drivers-firmware-arm-scmi.md) — 檔案層級摘要
- [Driver Model](../concepts/driver-model.md) — SCMI bus 的 bus-device-driver 模型
- [中斷處理](../concepts/interrupt-handling.md) — 通知的中斷路徑
- [Platform Bus](./platform-bus.md) — SCMI 以 platform device 掛載
