---
type: analysis
question: "ARM SCMI 的實作為何？Android 三大 SoC 陣營（ARM 標準 / Google Tensor / Qualcomm Snapdragon）在 AP↔SCP 韌體介面上各自採用什麼設計？"
pages_consulted:
  - ../subsystems/firmware.md
  - ../entities/arm-scmi.md
  - ../entities/google-acpm.md
  - ../entities/qualcomm-firmware-stack.md
  - ../sources/src-drivers-firmware-arm-scmi.md
  - ../sources/src-drivers-firmware-samsung-acpm.md
  - ../sources/src-drivers-firmware-google.md
  - ../sources/src-drivers-firmware-qcom-scm.md
created: 2026-04-20
last_updated: 2026-04-20
---

# AP↔SCP 韌體介面三方對照：ARM SCMI vs Google ACPM vs Qualcomm RPMh/CPUCP

## 核心結論

Android SoC 在同一個「Linux 向 SoC 內電源管理小核請求 DVFS、clock、PMIC、power domain 服務」角色上，分裂出**三條完全不同的路線**：

| 陣營 | 代表 SoC | 主要協定 | 特徵 |
|------|---------|----------|------|
| **ARM 標準派** | NXP i.MX95、STM32MP、Allwinner、Rockchip、AMD Versal | ARM SCMI | 10 個 protocol、4 種 transport、跨 SoC 標準化 |
| **Google / Samsung 派** | Tensor GS101（Pixel 6 系列）、Exynos 系列 | ACPM（私有）| Mailbox + shmem + 私有訊息格式、Linaro 近年才 upstream |
| **Qualcomm 派** | Snapdragon 8 Gen x、6/7 系列、X Elite | RPMh + AOSS + SPM + SCM + (SCMI for CPUCP) | 多層平行協定、硬體加速 TCS、CPU DVFS 硬體直寫 |

**Google Tensor（GS101，Pixel 6 系列）並未使用 ARM SCMI**，而是沿用 Samsung Exynos ACPM。**Qualcomm Snapdragon 主流平台（SM8xxx、SM7xxx）亦未使用 SCMI**，走自家 RPMh 堆疊；僅 **Snapdragon X Elite（`qcom,x1e80100`，代號 Hamoa）是 ACK 上唯一引入 `arm,scmi`** 的 Qualcomm SoC，而且只用 Perf protocol，其他 resource 仍走 RPMh。

另外，原問題中「Google SoC Accelerator Firmware」在 Android Common Kernel 樹中**沒有任何公開的韌體介面驅動**（Pixel 的 TPU/GXP accelerator 驅動在 Google vendor tree `private/google-modules/...`，不在 ACK），因此 ACK 範圍內只能討論主要系統管理韌體（ACPM），以及廣義 Google Coreboot firmware（`drivers/firmware/google/`，與 accelerator 無關）。

## A. Google Tensor（GS101）證據鏈

### A.1 Device Tree — `firmware { }` 節點

Tensor GS101 的 DT 中 `firmware` 節點宣告的是 ACPM，不是 SCMI：

```
// arch/arm64/boot/dts/exynos/google/gs101.dtsi:486-493
firmware {
    acpm_ipc: power-management {
        compatible = "google,gs101-acpm-ipc";
        #clock-cells = <1>;
        mboxes = <&ap2apm_mailbox>;
        shmem = <&apm_sram>;
    };
};
```

對整個 `arch/arm64/boot/dts/exynos/google/` 做 `grep -i scmi` 沒有任何 match。

相較之下，若走 SCMI，典型節點會是：

```
firmware {
    scmi {
        compatible = "arm,scmi";
        mboxes = <...>;
        shmem = <...>;
        scmi_clk: protocol@14 { reg = <0x14>; #clock-cells = <1>; };
        scmi_perf: protocol@13 { ... };
        ...
    };
};
```

### A.2 Device Tree Binding

`Documentation/devicetree/bindings/firmware/google,gs101-acpm-ipc.yaml` 明確把這個 firmware node 描述為「ACPM (Alive Clock and Power Manager) mailbox protocol」，與 `Documentation/devicetree/bindings/firmware/arm,scmi.yaml` 並列但格式完全獨立。

### A.3 CPU node 的 clock provider

```
// gs101.dtsi:76
cpu0: cpu@0 {
    ...
    clocks = <&acpm_ipc GS101_CLK_ACPM_DVFS_CPUCL0>;
};
```

CPU DVFS clock 指向 `acpm_ipc`（自行實作的 clock provider），而非 SCMI 的 `scmi_clk` 或 `scmi_perf`。`GS101_CLK_ACPM_DVFS_CPUCL0/1/2` 等 ID 定義在 `include/dt-bindings/clock/google,gs101-acpm.h`，屬於 ACPM 私有 clock ID 空間。

### A.4 Driver 端

`drivers/firmware/samsung/exynos-acpm.c:772-778` 的 of_match table：

```
static const struct of_device_id acpm_match[] = {
    {
        .compatible = "google,gs101-acpm-ipc",
        .data = &acpm_gs101,
    },
    {},
};
```

這支 driver 的 `MODULE_DESCRIPTION` 是 "Samsung Exynos ACPM mailbox protocol driver"，由 Linaro/Samsung/Google 三方 copyright，清楚標示**ACPM 在 upstream 是以 Samsung Exynos 協定家族出現**，Tensor GS101 只是其目前唯一的 upstream match target。

### A.5 SCMI vendors 目錄

`drivers/firmware/arm_scmi/vendors/` 底下**只有 `imx`**，沒有 `google` 或 `exynos`。若 Google 曾嘗試 SCMI 化，合理預期會在這個位置看到 `vendors/google/` — 事實上並不存在。

### A.6 `drivers/firmware/google/` 的內容

此目錄下全部 12 個檔案（cbmem/coreboot_table/gsmi/memconsole/vpd/framebuffer-coreboot）都是 **Chromebook coreboot** 相關，不涉及 Pixel 手機 SoC 的電源管理或 accelerator 韌體。

### A.7 功能對映：SCMI protocol 如何在 GS101 上實現？

| SCMI Protocol（其他 SoC） | GS101 對應實作 |
|---------------------------|----------------|
| Base (0x10) | ACPM 啟動時的 shmem/channel 握手（private） |
| Power Domain (0x11) | `drivers/soc/samsung/exynos-pm-domains`（ACPM 訊息為後端） |
| System Power (0x12) | PSCI（`arm,psci-1.0`，gs101.dtsi:517）+ ACPM 通道 |
| Performance (0x13) | `exynos-acpm-dvfs.c` + cpufreq-dt |
| Clock (0x14) | `acpm_ipc` 作為 clock provider（DVFS domain）；週邊 clock 由 `drivers/clk/samsung/clk-gs101.c` 直接操作 CMU |
| Sensor (0x15) | `drivers/thermal/samsung/exynos_tmu.c`（直接讀 TMU 暫存器，非走 ACPM 感測通道） |
| Reset (0x16) | `drivers/clk/samsung/` 內的 reset controller（直接操作 CMU） |
| Voltage (0x17) | `exynos-acpm-pmic.c` → S2MPG10 regulator |
| Powercap (0x18) | ACPM thermal mitigation（vendor 擴展，不在 ACK 樹中） |
| Pinctrl (0x19) | `drivers/pinctrl/samsung/pinctrl-exynos.c`（直接操作 GPIO 暫存器） |

差異之處：SCMI 把許多 resource 類服務集中在一個 protocol agent 底下；GS101 的做法是 **功能分散**——DVFS 與 PMIC 走 ACPM，clock/reset/pinctrl 走 AP 直接 MMIO，thermal 直接讀 TMU，system power 由 PSCI/ATF 處理。

## B. Qualcomm 證據鏈

### B.1 兩條路線並存：主流 SM8xxx 不走 SCMI，X Elite 走混合 SCMI+RPMh

對整個 `arch/arm64/boot/dts/qcom/` 做 compatible 統計：
- 使用 `qcom,rpmh-rsc`（主要 RPMh 節點）的 dtsi 檔案：**22 份**
- 使用 `arm,scmi` 的 dtsi 檔案：**僅 1 份**（`hamoa.dtsi` — Snapdragon X Elite / `qcom,x1e80100`）

即所有當代主流 Snapdragon 手機（SM6xxx/SM7xxx/SM8xxx、SDM845、SC7xxx）**完全不用 SCMI**，走 RPMh 堆疊。

### B.2 主流 Snapdragon 的 DT 結構（以 SM8550 為代表）

SM8550 `firmware { }` 相關節點**不含 `scmi`**，而是：

```
apps_rsc: rsc@17a00000 {
    compatible = "qcom,rpmh-rsc";
    qcom,tcs-offset = <0xd00>;
    qcom,tcs-config = <...>;
    qcom,drv-id = <2>;

    apps_bcm_voter: bcm-voter { compatible = "qcom,bcm-voter"; };
    rpmhcc: clock-controller { compatible = "qcom,sm8550-rpmh-clk"; ... };
    rpmhpd: power-controller { compatible = "qcom,sm8550-rpmhpd"; ... };
};

aoss_qmp: power-controller@c300000 {
    compatible = "qcom,sm8550-aoss-qmp", "qcom,aoss-qmp";
    ...
};

cmd-db@c3f0000 { compatible = "qcom,cmd-db"; ... };
```

CPU node 則是：

```
cpu0: cpu@0 {
    compatible = "qcom,kryo";
    enable-method = "psci";
    operating-points-v2 = <&cpu0_opp_table>;
    /* 沒有 scmi_dvfs 或 scmi_perf power-domain */
};
```

CPU DVFS 由 `qcom-cpufreq-hw` driver 直寫 OSM/EPSS 暫存器，完全繞過任何韌體訊息通道。

### B.3 X Elite（Hamoa）— 混合案例

唯一例外：`arch/arm64/boot/dts/qcom/hamoa.dtsi` 中 SCMI 與 RPMh **並存**：

```
scmi {
    compatible = "arm,scmi";
    mboxes = <&cpucp_mbox 0>, <&cpucp_mbox 2>;
    mbox-names = "tx", "rx";
    shmem = <&cpu_scp_lpri0>, <&cpu_scp_lpri1>;

    scmi_dvfs: protocol@13 {
        reg = <0x13>;
        #power-domain-cells = <1>;
    };
};

cpucp_mbox: mailbox@17430000 {
    compatible = "qcom,x1e80100-cpucp-mbox";
    ...
};

apps_rsc: rsc@17500000 {
    compatible = "qcom,rpmh-rsc";
    ...
    rpmhcc: clock-controller {
        compatible = "qcom,x1e80100-rpmh-clk";
        ...
    };
};
```

CPU 節點：

```
cpu0: cpu@0 {
    compatible = "qcom,oryon";
    power-domains = <&cpu_pd0>, <&scmi_dvfs 0>;
    power-domain-names = "psci", "perf";
};
```

**觀察**：
1. **CPUCP**（CPU Cluster Power Controller）是 X Elite 新加的專用 SCP 硬體，speaks SCMI；`compatible = "qcom,x1e80100-cpucp-mbox"`。
2. **只啟用 SCMI Perf (0x13) 一個 protocol**，其他 SCMI protocol 一律不用。
3. Clock 仍走 `rpmhcc`（`qcom,x1e80100-rpmh-clk`）、regulator 仍走 RPMh、interconnect 仍走 BCM voter、AOSS QMP 仍存在。
4. SCMI shmem 用 `arm,scmi-shmem` 標準 compatible，對齊 ARM spec。

這是 **Qualcomm 首度在 upstream 平台引入 ARM 標準韌體協定**的實驗——以 SCMI 包裝 CPU perf，但整個 SoC-wide resource voting 仍維持 vendor-specific RPMh。

### B.4 Qualcomm 功能對映

| SCMI Protocol | Qualcomm 主流（SM8xxx） | Qualcomm X Elite（x1e80100） |
|---------------|------------------------|-----------------------------|
| Base (0x10) | 無（用 Command DB 靜態表）| 部分用於 CPUCP SCMI 握手 |
| Power Domain (0x11) | RPMhPD（`rpmhpd.c`）| RPMhPD 為主；`scmi_dvfs` 作 CPU perf PD |
| System Power (0x12) | PSCI（ATF）| PSCI（ATF） |
| Performance (0x13) | **OSM/EPSS 硬體直寫**（`qcom-cpufreq-hw`）— 連 IPC 都沒有 | **SCMI perf via CPUCP** ✓ |
| Clock (0x14) | RPMhCC（`clk-rpmh.c`）| RPMhCC（`x1e80100-rpmh-clk`） |
| Sensor (0x15) | TSENS 直讀 + AOSS QMP thermal | 同左 |
| Reset (0x16) | GCC 直寫 | GCC 直寫 |
| Voltage (0x17) | RPMh regulator（`qcom-rpmh-regulator.c`）| RPMh regulator |
| Powercap (0x18) | AOSS QMP thermal mitigation | 同左 |
| Pinctrl (0x19) | TLMM 直寫（`pinctrl-msm.c`）| TLMM 直寫 |
| 額外：Interconnect | RPMh BCM voter（SCMI 無此 protocol）| RPMh BCM voter |
| 額外：Secure world | SCM/QSEECOM（SCMI 無此層）| 同左 |
| 額外：Subsystem wake | AOSS QMP、SMP2P、SMSM | 同左 |

Qualcomm 的路線是**把 resource voting 丟給硬體 TCS 自動化，CPU DVFS 丟給硬體 state machine**——這兩條路徑的延遲都低於 SCMI mailbox 往返。SCMI 能切進來的只剩 CPUCP 場景（新 CPU 複合體有專用 SCP 而且 Qualcomm 願意讓它說 SCMI）。

## C. 三方對照總表

| 功能 | ARM SCMI 標準 | Google Tensor GS101 | Qualcomm SM8xxx（主流）| Qualcomm X Elite |
|------|--------------|--------------------|----------------------|------------------|
| **AP↔SCP 主協定** | SCMI（10 protocols） | ACPM（私有 mailbox）| **無單一協定**（多層平行）| RPMh + SCMI（CPUCP） |
| **CPU DVFS** | perf (0x13) | ACPM DVFS + cpufreq-dt | **OSM/EPSS 硬體直寫**（無 IPC）| **SCMI perf (0x13) via CPUCP** |
| **Clock** | clock (0x14) | ACPM + Samsung CMU | RPMhCC（`clk-rpmh.c`）| RPMhCC（`x1e80100-rpmh-clk`） |
| **Regulator** | voltage (0x17) | ACPM-PMIC（S2MPG10）| RPMh regulator | RPMh regulator |
| **Power Domain** | power (0x11) | Samsung Exynos PD | RPMhPD | RPMhPD + `scmi_dvfs` |
| **Interconnect** | 無（SCMI 不含此）| 無獨立協定 | **RPMh BCM voter**（硬體聚合）| 同左 |
| **Sensor/Thermal** | sensor (0x15) | TMU 直讀 | TSENS + AOSS QMP | 同左 |
| **Reset** | reset (0x16) | CMU 直寫 | GCC 直寫 | GCC 直寫 |
| **Pinctrl** | pinctrl (0x19) | GPIO 直寫 | TLMM 直寫 | TLMM 直寫 |
| **System Power** | system (0x12) | PSCI | PSCI | PSCI |
| **CPU idle** | 無（屬 PSCI）| PSCI | SPM/SAW + PSCI | CPUCP + PSCI |
| **TrustZone** | 無 | 走 Samsung EL3 | **SCM + QSEECOM**（`qcom_scm.c` 2,476 行）| 同左 |
| **Subsystem messaging** | notify framework | ACPM IRQ 回覆 | **AOSS QMP + SMP2P + SMSM** | 同左 |
| **Resource discovery** | Base protocol（動態）| 無 | **Command DB**（bootloader 靜態表）| Command DB + SCMI Base |
| **硬體加速路徑** | 無 | 無 | **RSC 的 TCS 暫存器**（sleep/wake 不經 CPU SW）| 同左 + SCMI shmem |
| **主要 Kernel 驅動** | `drivers/firmware/arm_scmi/` 31 檔 19,481 行 | `drivers/firmware/samsung/exynos-acpm.c` 1,111 行 | `drivers/firmware/qcom/` 4,447 行 + `drivers/soc/qcom/` 多檔 | 同左 + SCMI |
| **Upstream 狀態** | 完全 upstream | 2024 年 Linaro 才 upstream | 完全 upstream（Qualcomm 長期投入）| 完全 upstream |

## 為何 Google 不轉 SCMI

這部分屬於設計選擇與歷史脈絡，ACK 樹內無一手文件，但根據上述證據可推論：

**1. 重用 Exynos Tensor 共享的 SCP 韌體棧。** Tensor GS101 是 Samsung Foundry 代工、基於 Exynos 9855 衍生的 SoC。Samsung 自 Exynos 7 起便使用 ACPM 家族韌體，若改用 SCMI 需重寫 SCP 端整套韌體、PMIC 交易協定、ECT 載入流程。新舊韌體一起維護，在 Pixel 這種需要同步追 Samsung 母平台的產品上代價過高。

**2. ECT（Energy & Clock Table）的複雜性。** Samsung/Google 的 DVFS 表結構（包含 margin、ASV lot、temperature compensation）與 SCMI spec 的 perf domain level 表達能力不完全對齊，轉換需要 SCMI vendor-specific message，等同重新發明一次協定。

**3. Upstream 時間壓力。** 直到 2024 年 Linaro 才把 GS101 的 ACPM 驅動 upstream 到 mainline（`exynos-acpm.c`）。在此之前 Pixel 6 多年走 Google vendor tree，延續 Samsung 的 API 是最低阻力路徑。

**4. 缺乏 consumer 拉力。** SCMI 的最大價值是讓 Linux 對許多 SoC 用同一組 consumer driver（scmi-cpufreq、scmi-hwmon）。Pixel 有自家完整軟體棧，SCMI 帶來的解耦效益相對有限。

## 為何 Qualcomm 主流路線堅持 RPMh 不轉 SCMI

Qualcomm 不轉的動機和 Google 不同 — 主要是**技術性能優勢**，而不只是歷史包袱：

**1. TCS 硬體旁路 SW。** RPMh RSC 的 TCS 暫存器讓 sleep/wake 路徑上 resource 投票**不需要 CPU 執行 SW IPC**：AP 預先寫好 SLEEP/WAKE TCS commands，CPU 一進 WFI，硬體自動 fire。SCMI 仍需走 mailbox doorbell + SCP 回覆，延遲高一個量級。這在手機 idle 頻繁進出 low power 的場景很關鍵。

**2. BCM 是 SCMI 沒有的協定。** Qualcomm 的 NoC 聚合有明確的「多個 master 先在 BCM 本地加總、再送 RPMh」的兩層結構，SCMI spec 沒有對應 Interconnect protocol。轉 SCMI 等於要在 vendor protocol 上重新造一次 BCM voting，沒有收益。

**3. CPU DVFS 已經比 SCMI 更快。** OSM/EPSS 是**硬體 FSM 自己跑 DVFS**，連 mailbox 都不需要。從 SM8250 開始這條路已確立；SCMI perf 在 Qualcomm 主流路線上**反而會變慢**。

**4. 完整軟體棧 + 客戶要求。** Qualcomm 的 BSP 客戶（Samsung、小米、OnePlus 等）長期基於 RPMh 軟體棧，整條 cpufreq/thermal/regulator 驅動都已是自家。SCMI 化要同步改 Linux、ATF、SCP 韌體、BCM 邏輯四處，投入非常大。

**5. X Elite 的部分採用是策略性的。** 為何只在 CPUCP 上啟用 SCMI perf？可能原因：
   - X Elite 要進筆電市場，Linux distro（Fedora、Ubuntu、Arch）偏好標準 SCMI 的驅動 supportability。
   - CPUCP 是全新硬體，沒有過去 OSM/SAW 的遺留程式碼包袱。
   - 只先動 CPU perf 這一塊，風險可控。
   - 未來若證實可行，可能逐步擴大 SCMI 覆蓋範圍。

## 若未來要 SCMI 化

從 upstream 結構面看，若 Google 日後想對齊 ARM 生態，合理做法是：

**短期**：在 `drivers/firmware/arm_scmi/vendors/google/` 建立 Google vendor SCMI protocols（類似 `vendors/imx/`），把 Google 專屬的 thermal mitigation、accelerator power hint 等訊息定義為 vendor protocol，但核心 DVFS/clock 仍走 ACPM 段時間共存。

**長期**：由 APM 韌體同時 speak SCMI 與 ACPM，AP 端逐步把 consumer 從 `acpm_ipc` 遷移到 SCMI clock/perf protocol。此路徑需要 Samsung 與 Google 兩方共同維護 APM 韌體。

目前 ACK 樹中沒有看到這類轉換的 scaffolding，也沒有相關 Kconfig hint。

## 總結一句話

在 Android Common Kernel 可見範圍內，**ARM SCMI 是標準、跨 SoC、公開協定格式的 AP↔SCP 介面**；**Google Tensor GS101 則透過 Samsung Exynos 系列的 ACPM 協定**達成同樣功能，格式私有、目前由 `drivers/firmware/samsung/exynos-acpm.c` 單一 driver 承載。兩條路線並無相容性，Tensor SoC 將來是否走向 SCMI 是 Google 與 Samsung 韌體路線圖的策略問題，現階段無此訊號。

## Cross-References

- [ARM SCMI](../entities/arm-scmi.md)
- [Google ACPM](../entities/google-acpm.md)
- [Firmware 子系統](../subsystems/firmware.md)
- [ARM SCMI Source Summary](../sources/src-drivers-firmware-arm-scmi.md)
- [Samsung ACPM Source Summary](../sources/src-drivers-firmware-samsung-acpm.md)
- [Google Coreboot Source Summary](../sources/src-drivers-firmware-google.md)
