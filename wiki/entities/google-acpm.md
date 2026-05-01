---
type: entity
kernel_path: drivers/firmware/samsung/exynos-acpm.c
config_option: CONFIG_EXYNOS_ACPM_PROTOCOL
upstream: yes
related:
  - ../subsystems/firmware.md
  - ../entities/arm-scmi.md
  - ../entities/qualcomm-firmware-stack.md
  - ../analyses/scmi-vs-google-acpm.md
  - ../sources/src-drivers-firmware-samsung-acpm.md
last_updated: 2026-04-20
---

# Google ACPM (Alive Clock and Power Manager) on Tensor GS101

## Overview

ACPM（**A**live **C**lock and **P**ower **M**anager）是執行在 Samsung/Google SoC 內 **APM（Active Power Management）** Cortex-M 小核上的韌體，負責整體電源管理：DVFS 頻率/電壓調整、power domain 狀態、PMIC 暫存器存取、熱限制等。AP 端 Linux 透過 **mailbox + shared memory** 的私有訊息格式與 ACPM 韌體溝通，扮演 ARM SCMI 在其他 SoC 上的同一個角色，但是 **vendor-specific 的訊息格式**。

**重點**：Google **Tensor G1（代號 GS101，Pixel 6/6 Pro/6a 使用）重用了 Samsung Exynos 的 ACPM 協定**。DT compatible 是 `google,gs101-acpm-ipc`，由 `drivers/firmware/samsung/exynos-acpm.c` 這支 upstream driver 在 `of_match_table` 中透過 `acpm_gs101` match_data 直接支援（`drivers/firmware/samsung/exynos-acpm.c:772-778`）。換言之，**Tensor 沒有採用 ARM SCMI，而是走 ACPM**。

## Source Layout

Kernel 端驅動位於 `drivers/firmware/samsung/`（6 檔、1,111 行）：

```
drivers/firmware/samsung/
├── exynos-acpm.c          # 核心：mailbox 通訊、shmem 管理、channel 拆分、seqnum、timeout
├── exynos-acpm.h
├── exynos-acpm-dvfs.c     # DVFS client：頻率表、電壓/頻率綁定
├── exynos-acpm-dvfs.h
├── exynos-acpm-pmic.c     # PMIC client：I2C-like 暫存器讀寫
└── exynos-acpm-pmic.h
```

Uapi/consumer 用 header：`include/linux/firmware/samsung/exynos-acpm-protocol.h`。

Device tree binding 文件：`Documentation/devicetree/bindings/firmware/google,gs101-acpm-ipc.yaml`（compatible 目前只接受 `google,gs101-acpm-ipc`）。

DT clock-id 定義：`include/dt-bindings/clock/google,gs101-acpm.h`（DVFS domain 列表，如 `GS101_CLK_ACPM_DVFS_CPUCL0/1/2`、MIF、INT、GPU 等）。

## Implementation Details

### Device Tree 節點

在 `arch/arm64/boot/dts/exynos/google/gs101.dtsi:486-493`：

```
firmware {
    acpm_ipc: power-management {
        compatible = "google,gs101-acpm-ipc";
        #clock-cells = <1>;
        mboxes = <&ap2apm_mailbox>;
        shmem = <&apm_sram>;
    };
};
```

三個核心屬性對應 ACPM 的通訊機制：
- `mboxes`：AP→APM 的 mailbox 通道（Samsung Exynos message format，由 `drivers/mailbox/exynos-mailbox.c` 驅動）。
- `shmem`：APM SRAM 的保留記憶體區，放 channel 配置與 TX/RX ring buffer。
- `#clock-cells = <1>`：ACPM 同時是 clock provider，CPU DVFS clock 用 `<&acpm_ipc GS101_CLK_ACPM_DVFS_CPUCL0>` 形式引用（gs101.dtsi:76, 88, 100, ...）。

### 訊息格式與通道拆分

`exynos-acpm.c` 實作 ACPM 協定核心：

- **Shared memory layout**：`struct acpm_shmem` 描述 SRAM 起頭的 channel 配置；`struct acpm_chan_shmem` 描述每條邏輯通道（RX/TX ring buffer 位址、大小、通道 ID）。
- **Seqnum 流控**：`ACPM_PROTOCOL_SEQNUM = GENMASK(21, 16)`（6-bit），每通道 `ACPM_SEQNUM_MAX = 64` 個 in-flight 訊息，以 bitmap 管理。
- **訊息格式**：每則訊息以固定 32-bit word 為單位，最前面的 header word 包含 seqnum、command、channel 等欄位。
- **通道拆分**：不同功能（DVFS、PMIC、log、debug）使用不同通道；每個通道有獨立 ring buffer 與 seqnum 空間。
- **Timeout**：poll 模式 `ACPM_POLL_TIMEOUT_US = 100 ms`；中斷模式 `ACPM_TX_TIMEOUT_US = 500 ms`。
- **Match data**：`acpm_gs101.initdata_base = 0xa000` 指向 GS101 APM SRAM 內 acpm_shmem 結構起始位移。

### Client 子驅動

**`exynos-acpm-dvfs.c`**：在 ACPM 之上實作 clock provider（`struct clk_hw` ops）。每個 DVFS domain（CPUCL0/1/2、MIF、INT、GPU、DSU、AUR 等）對應一個 `clk_ops.set_rate` 操作，將目標頻率 ID 轉成 ACPM 訊息送至 APM。APM 韌體查內建 DVFS table、同步調整 PMIC 電壓與 CMU 時脈。

**`exynos-acpm-pmic.c`**：提供 PMIC 暫存器的 regmap 類介面，把 I2C-like 交易（addr/value）轉成 ACPM 訊息交由 APM 執行（因為 PMIC I2C 可能連在 APM 直接控制的 bus 上，AP 無法直接寫）。`regulator`、`rtc`、`power-supply` 等子系統透過此介面存取 S2MPG10 PMIC。

### DVFS domain 範例

gs101.dtsi 中的 CPU node 宣告：

```
cpu0: cpu@0 {
    ...
    clocks = <&acpm_ipc GS101_CLK_ACPM_DVFS_CPUCL0>;
};
```

cpufreq-dt 取得這個 clock 後用 `clk_set_rate()` 觸發 ACPM 訊息，效果等同 SCMI perf protocol 的 `scmi_perf_level_set()`。差異是訊息格式、seqnum 機制、shmem layout 全是 Samsung/Google 自家定義，不是 ARM SCMI spec。

## Userspace Interface

和 SCMI 一樣，ACPM 本身不直接暴露 userspace 介面。使用者經由標準框架接觸：

- **cpufreq**：`/sys/devices/system/cpu/cpufreq/policy*/` 最終呼叫 acpm DVFS clock 的 `set_rate`。
- **devfreq**：MIF/INT 的頻率調整透過 devfreq + ACPM DVFS domain。
- **regulator**：S2MPG10 PMIC 透過 exynos-acpm-pmic 的 regmap。
- **thermal**：thermal governor 限制最大頻率時，最終也落到 ACPM DVFS。

## Android-Specific Notes

此驅動**完全是 upstream Linux**，沒有 `ANDROID:` 補丁或 vendor hook。這是 Linaro（Tudor Ambarus）協助把 Google 外部維護的 ACPM 程式碼 upstream 後的成果：`MODULE_AUTHOR("Tudor Ambarus <tudor.ambarus@linaro.org>")`。

在 Android vendor kernel（`private/google-modules/...`）上，Google 另有擴展的 ACPM 訊息（例如 thermal、mitigation、accelerator power states），但這些目前都**不在 Android Common Kernel（ACK）樹中**。ACK 樹內可見的 ACPM 就是 upstream 的 mainline 版本。

Pixel 裝置開機與 DVFS 高度依賴 ACPM 韌體；`apm_sram` 節點還會保留 bootloader 放入 DVFS 表（ECT, Energy & Clock Table）的記憶體，`ect { };` 空節點在 gs101.dtsi:471 也是為此存在（bootloader 在解析 DT 時注入 ECT blob）。

## Configuration

| Kconfig | 類型 | 說明 |
|---------|------|------|
| `EXYNOS_ACPM_PROTOCOL` | tristate | 核心 ACPM 協定驅動（depends on `ARCH_EXYNOS || COMPILE_TEST`, `MAILBOX`） |
| 無獨立選項 | — | DVFS/PMIC 兩個 client 與主驅動一起編譯 |

模組名 `exynos-acpm-protocol`（`drivers/firmware/samsung/exynos-acpm.c:784`）。

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md) — 包含此驅動在整體韌體介面中的定位
- [ARM SCMI](./arm-scmi.md) — 等價的 ARM 標準協定
- [Qualcomm Firmware Stack](./qualcomm-firmware-stack.md) — 另一個 vendor-specific 路線：RPMh/AOSS/SPM/SCM 多層平行協定
- [AP↔SCP 韌體介面三方對照](../analyses/scmi-vs-google-acpm.md) — ARM SCMI vs Google ACPM vs Qualcomm 三方比較
- [Samsung ACPM Source Summary](../sources/src-drivers-firmware-samsung-acpm.md)
- [Platform Bus](./platform-bus.md)
