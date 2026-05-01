---
type: entity
kernel_path: drivers/firmware/qcom/ + drivers/soc/qcom/
config_option: CONFIG_QCOM_SCM, CONFIG_QCOM_RPMH, CONFIG_QCOM_SMD_RPM, CONFIG_QCOM_AOSS_QMP, CONFIG_QCOM_SPM, CONFIG_QCOM_COMMAND_DB
upstream: yes
related:
  - ../subsystems/firmware.md
  - ../entities/arm-scmi.md
  - ../entities/google-acpm.md
  - ../analyses/scmi-vs-google-acpm.md
  - ../sources/src-drivers-firmware-qcom-scm.md
last_updated: 2026-04-20
---

# Qualcomm Firmware Stack（SMD-RPM / RPMh / AOSS / SPM / SCM / CPUCP）

## Overview

Qualcomm 的 AP↔SoC 韌體介面**不是單一協定**，而是一套**跨 `drivers/firmware/qcom/` 與 `drivers/soc/qcom/` 的多層平行協定堆疊**，依功能（power voting、secure world、thermal、idle state、CPU DVFS）切成數條獨立通道。整體架構跨越三代演進，從 2015 年前的 SMD-RPM，過渡到 SDM845 以後的 RPMh+AOSS 主流棧，最近在 Snapdragon X Elite（Hamoa）上開始導入 **CPUCP + ARM SCMI** 作為 CPU perf 介面。

這種「多條 vendor-specific channel + 選擇性採用 SCMI」的設計，與 Google Tensor 純走 [ACPM](./google-acpm.md)、其他 SoC 廠純走 [SCMI](./arm-scmi.md) 的模式形成對比 — Qualcomm 用硬體加速（RPMh 的 TCS 暫存器）換取 sleep/wake 延遲，但代價是完全 vendor-specific 的韌體生態。

## Source Layout

**`drivers/firmware/qcom/`（8 檔、4,447 行）— TrustZone 層**

```
drivers/firmware/qcom/
├── qcom_scm.c              # 核心 SCM driver (2,476 行)
├── qcom_scm-smc.c          # SMC call wrapper (215 行)
├── qcom_scm-legacy.c       # 舊版 SMC 呼叫格式 (246 行)
├── qcom_scm.h
├── qcom_qseecom.c          # QSEECOM client interface (120 行)
├── qcom_qseecom_uefisecapp.c # UEFI Secure App shim (866 行)
├── qcom_tzmem.c            # TrustZone 記憶體池 (524 行)
└── qcom_tzmem.h
```

**`drivers/soc/qcom/` — Power/Resource/Messaging 各層（節錄）**

```
drivers/soc/qcom/
├── rpmh.c                  # RPMh client API (502 行)
├── rpmh-rsc.c              # RSC 硬體控制器 (1,156 行)
├── rpmh-internal.h
├── cmd-db.c                # RPMh resource address lookup (393 行)
├── smd-rpm.c               # 舊版 SMD-RPM client (272 行)
├── qcom_aoss.c             # AOSS QMP messaging (673 行)
├── spm.c                   # Subsystem Power Manager (576 行)
├── apr.c                   # Asynchronous Packet Router (音訊/modem)
├── smem.c / smem_state.c   # Shared memory & state
├── smp2p.c / smsm.c        # Subsystem-to-subsystem signaling
├── llcc-qcom.c             # Last-Level Cache Controller
├── rpm-proc.c, rpm_master_stats.c # RPM 統計
└── pdr_interface.c, pmic_glink.c  # Protection Domain / PMIC GLINK
```

**相關 Consumer Drivers（非 firmware/ 下，但構成 Qualcomm 韌體介面的上層）**

| Client | 位置 | 行數 | 透過 |
|--------|------|------|------|
| RPMh clock | `drivers/clk/qcom/clk-rpmh.c` | 1,043 | RPMh |
| SMD-RPM clock | `drivers/clk/qcom/clk-smd-rpm.c` | 1,446 | SMD-RPM |
| RPMh regulator | `drivers/regulator/qcom-rpmh-regulator.c` | 1,991 | RPMh |
| SMD regulator | `drivers/regulator/qcom_smd-regulator.c` | 1,492 | SMD-RPM |
| RPMh interconnect | `drivers/interconnect/qcom/icc-rpmh.c` + `bcm-voter.c` | 375 + 413 | RPMh BCM |
| RPM interconnect | `drivers/interconnect/qcom/icc-rpm.c` | 638 | SMD-RPM |
| CPU freq HW | `drivers/cpufreq/qcom-cpufreq-hw.c` | — | OSM/EPSS 硬體直寫 |

## Implementation Details

### 第一代：SMD-RPM（MSM8916 / MSM8974 / MSM8953 世代）

**定位**：透過 Shared Memory Driver 與 RPM（Resource Power Manager，SoC 內 Cortex-M3 小核）溝通，是 SCMI/RPMh 出現前的 Qualcomm 標準。

**Driver**：`drivers/soc/qcom/smd-rpm.c`（272 行）。

**Compatible**（`smd-rpm.c:219-231`）：
- `qcom,glink-smd-rpm`、`qcom,smd-rpm`（泛用，走 GLINK/SMD 子系統）
- per-SoC：`qcom,rpm-msm8916`、`qcom,rpm-msm8974`、`qcom,rpm-msm8936`、`qcom,rpm-msm8953`、`qcom,rpm-apq8084`、`qcom,rpm-ipq6018`、`qcom,rpm-ipq9574`、`qcom,rpm-msm8226`、`qcom,rpm-msm8909` 等

**訊息格式**：以 `qcom_smd-rpm` packet 結構透過 SMEM 傳遞，每則訊息含 resource type（clock / regulator / bus）+ resource ID + value。

**Client**：`clk-smd-rpm.c`（1,446 行）、`qcom_smd-regulator.c`（1,492 行）、`icc-rpm.c`（638 行）。

**現狀**：`smd-rpm.c:221-224` 的註解明言「Don't add any more compatibles to the list」— Qualcomm 不再新增 SMD-RPM SoC，新平台全部走 RPMh。此層持續維護主要是為支援舊平台。

### 第二代：RPMh + RSC + Command DB + BCM Voter（SDM845 以降的主流）

**核心概念**：RPMh（Resource Power Manager, **hardened**）是**硬體加速**的 resource voting 框架。與 SCMI 最大的區別是 **TCS（Trigger Control Set）暫存器**讓 sleep/wake 時資源投票由硬體直接執行，不需 CPU 跑 SW IPC。

#### RSC — Resource State Coordinator

檔案：`drivers/soc/qcom/rpmh-rsc.c`（1,156 行）

Compatible：`qcom,rpmh-rsc`（`rpmh-rsc.c:1135`）

RSC 是一塊 MMIO 硬體區塊，有多組 TCS（Trigger Control Set）暫存器。每個 TCS 存一組 `<addr, data>` command，對應一個 RPMh resource 狀態（例如把 `cx.lvl` 投票到某個 level）。TCS 類型：

- **AMC**（Active Mode Coordinator，主動投票）
- **SLEEP TCS**（進入 sleep 時自動觸發的 commands）
- **WAKE TCS**（離開 sleep 時自動觸發的 commands）
- **Control TCS**（RSC 自己的配置）

在 `rpmh-rsc.c:971-997` 解析 DT 的 `qcom,tcs-offset`、`qcom,tcs-config`、`qcom,drv-id` 屬性以建立 TCS 布局。硬體行為：CPU 執行 WFI 或進入 cluster off 時，RSC 自動 fire SLEEP TCS；被中斷喚醒時自動 fire WAKE TCS。這條路徑**不經過 CPU SW**，省掉 SCMI mailbox doorbell + polling 的延遲。

#### RPMh Client API

檔案：`drivers/soc/qcom/rpmh.c`（502 行）

對外 API（`rpmh.c:241-502`）：`rpmh_write_async()`、`rpmh_write()`（同步）、`rpmh_write_batch()`（一次多個）、`rpmh_invalidate()`。每個 request 以 `<resource addr, state, data>` 表達，state 有 `RPMH_ACTIVE_ONLY_STATE` / `RPMH_WAKE_ONLY_STATE` / `RPMH_SLEEP_STATE`。

內部 cache（`struct cache_req`, `rpmh.c:51-66`）避免重複送相同 vote。

#### Command DB — Resource Address Lookup

檔案：`drivers/soc/qcom/cmd-db.c`（393 行）

Compatible：`qcom,cmd-db`（`cmd-db.c:372`）

Bootloader 在 SMEM 中放一張 resource 名字 → address + arc 屬性的對照表（名稱如 `"cx.lvl"`、`"mx.lvl"`、`"clk0"`、`"ddr.lvl"`、`"ebi.lvl"` 等）。Client 透過 `cmd_db_read_addr("cx.lvl")` 取得 RPMh bus 上的實體位址，然後用此位址當 `rpmh_write()` 的 command target。

**這取代了 SCMI Base protocol 的動態探測**：Qualcomm 以 **bootloader 靜態註入表** 換 runtime discovery。代價是 resource list 更新要改 bootloader image；好處是開機路徑短、Linux 不需要和 SCP 握手取得能力描述。

#### BCM Voter — Interconnect 專用投票器

檔案：`drivers/interconnect/qcom/bcm-voter.c`（413 行）+ `icc-rpmh.c`（375 行）

BCM（Bus Clock Manager）是 RPMh 世代的 interconnect aggregator — NoC 上各 master 的頻寬需求透過 BCM voter 先在本地聚合，再以 RPMh command 送入 RSC。DT 上 `apps_bcm_voter: bcm-voter { compatible = "qcom,bcm-voter"; }` 位於 `apps_rsc` 節點下作為子節點。

這對應 SCMI 中**沒有對等協定**的功能（SCMI 沒有 Interconnect protocol），Qualcomm 的 NoC 聚合需求催生了這個專屬層。

#### RPMhPD — Power Domain via RPMh

DT binding：`qcom,rpmhpd`。Linux side 在 `drivers/pmdomain/qualcomm/rpmhpd.c`（非 firmware/ 下，但是 RPMh consumer）。power domain 狀態（例如 `cx`、`mx`、`mmcx`）本質是 RPMh resource 的 level vote。

#### RPMhCC — Clock Controller via RPMh

檔案：`drivers/clk/qcom/clk-rpmh.c`（1,043 行）

每個 SoC 有自己的 compatible：`qcom,sm8550-rpmh-clk`、`qcom,x1e80100-rpmh-clk`、`qcom,sc7280-rpmh-clk` 等。此 driver 把 XO / ref / BB clocks 暴露為 common clock framework `struct clk_hw`，`clk_ops.prepare/unprepare` 轉成 RPMh vote。

### AOSS QMP — Always-On Subsystem Messaging

**定位**：另一條**獨立於 RPMh** 的 messaging channel，AOSS 是永遠在線的 Cortex-M 子系統，管理 thermal mitigation、subsystem wake/sleep（modem、ADSP、CDSP、WPSS）、debug state vote。

檔案：`drivers/soc/qcom/qcom_aoss.c`（673 行）

Compatible（`qcom_aoss.c:650-656`）：
- `qcom,sc7180-aoss-qmp`、`qcom,sc7280-aoss-qmp`
- `qcom,sdm845-aoss-qmp`
- `qcom,sm8150-aoss-qmp`、`qcom,sm8250-aoss-qmp`、`qcom,sm8350-aoss-qmp`
- `qcom,aoss-qmp`（泛用 fallback）

**訊息格式**：QMP（Qualcomm Messaging Protocol）是**字串形式的 key=value**，例如 `"{class: ddr, res: perf_mode, val: 1}"`。和 RPMh 的二進位 `<addr, data>` 完全不同格式。

對外 API（`qcom_aoss.c:278-489`）：`qmp_send()`、`qmp_get()`、`qmp_put()`。Consumer 包括 thermal cooling device（`qcom_aoss.c` 內建 thermal_cooling_device_ops）、modem/wpss 喚醒協調。

**SCMI 對等**：類似 SCMI notification + powercap 的混合，但格式私有。

### SPM / SAW — Per-CPU Power Manager

檔案：`drivers/soc/qcom/spm.c`（576 行）

Compatible（`spm.c:478-496`，一長串 per-SoC）：
- `qcom,msm8226-saw2-v2.1-cpu`、`qcom,msm8916-saw2-v3.0-cpu`、`qcom,msm8939-saw2-v3.0-cpu`、`qcom,msm8974-saw2-v2.1-cpu`、`qcom,msm8976-gold/silver-saw2-v2.3-l2`、`qcom,msm8998-gold-saw2-v4.1-l2`、`qcom,sdm660-gold/silver-saw2-v4.1-l2`、`qcom,msm8909-saw2-v3.0-cpu`

SAW = Stand-Alone Window，是每顆 CPU 或每個 cluster 的 low-power finite state machine 暫存器組。Linux 配置 SAW sequence 後，CPU 執行 WFI 觸發對應 idle state（retention/power collapse）。

**SCMI 對等**：SCMI 沒有 per-CPU idle 協定，idle 由 PSCI + ATF 處理；Qualcomm 在 ATF 實作 PSCI 時會驅動 SAW 暫存器。新 SoC（SM8550+）逐步把 SPM 功能整合進 PSCI/CPUCP，不再需要獨立 SPM driver。

### SCM — TrustZone 通訊

檔案：`drivers/firmware/qcom/qcom_scm.c`（2,476 行，為 firmware/ 下最大的 driver）

**SCM = Secure Channel Manager**。透過 SMC call 進 EL3，由 TrustZone 韌體處理：secure boot 驗證、pas_init_image（subsystem 載入前的簽章驗證，用於 modem/ADSP）、secure 記憶體分配保護、qcom_scm_io_read/writel（受保護暫存器存取）、set_warm/cold_boot_addr（CPU hotplug 喚醒位址）。

主要 export（`qcom_scm.c:433-889`，超過 15 個 `EXPORT_SYMBOL_GPL`）包含：`qcom_scm_set_warm_boot_addr`、`qcom_scm_cpu_power_down`、`qcom_scm_pas_init_image`、`qcom_scm_pas_auth_and_reset`、`qcom_scm_io_readl/writel`。

### QSEECOM — Qualcomm Secure Execution Environment Communication

檔案：`drivers/firmware/qcom/qcom_qseecom.c`（120 行）+ `qcom_qseecom_uefisecapp.c`（866 行）

把訊息送給 TrustZone 內跑的 QSEE Trusted Application；`qcom_qseecom_uefisecapp.c` 是其中一個特定 TA 的 shim，提供 UEFI variable service（`efivarfs` 底下的 set/get variable 會走這條路徑到 QSEE）。

### CPU DVFS：OSM / EPSS — 硬體直寫（不經任何 IPC）

在 SM8250 以後，**CPU DVFS 根本不經過任何軟體韌體介面**。由 `drivers/cpufreq/qcom-cpufreq-hw.c` 直接寫 OSM（Operating State Manager）或 EPSS（Enhanced Performance State Switch）暫存器。硬體 state machine 自己完成電壓/頻率/電流同步切換。

**SCMI 對等**：這比 SCMI perf 還激進 — SCMI 還是需要 AP ↔ SCP mailbox 往返。OSM/EPSS 連 mailbox 都省掉，單次 freq change 是 MMIO 寫入等級的延遲。

### 第三代：CPUCP + ARM SCMI（X Elite / Hamoa）

Snapdragon X Elite（compatible `qcom,x1e80100`，代號 Hamoa）是**目前 ACK 上唯一採用 `arm,scmi` 的 Qualcomm 平台**。

DT（`arch/arm64/boot/dts/qcom/hamoa.dtsi`）中關鍵節點：

```
cpucp_mbox: mailbox@17430000 {
    compatible = "qcom,x1e80100-cpucp-mbox";
    reg = <0 0x17430000 0 0x10000>, <0 0x18830000 0 0x10000>;
    interrupts = <GIC_SPI 28 IRQ_TYPE_LEVEL_HIGH>;
    #mbox-cells = <1>;
};

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
```

Oryon CPU 節點：
```
cpu0: cpu@0 {
    compatible = "qcom,oryon";
    power-domains = <&cpu_pd0>, <&scmi_dvfs 0>;
    power-domain-names = "psci", "perf";
};
```

**觀察**：
1. **CPUCP**（CPU Cluster Power Controller）是 X Elite 新增的專用 SCP 硬體，**speaks SCMI**。
2. 只啟用 **Perf protocol（0x13）一個**，`#power-domain-cells = <1>` 把它當 generic PD 用，cpufreq-dt 透過此 PD 驅動 DVFS。
3. **SCMI 與 RPMh 並存**：同一份 DT 下 `apps_rsc` (`qcom,rpmh-rsc`) 仍在，clock 仍走 `rpmhcc` (`qcom,x1e80100-rpmh-clk`)、regulator 仍走 RPMh、interconnect 仍走 BCM voter。SCMI 只接管 CPU DVFS 一件事。
4. `cpu_scp_lpri0/1` 是 `arm,scmi-shmem` 標準 shmem 區塊，對齊 ARM SCMI spec。

這種「**以 SCMI 包裝 CPU perf，其他維持 RPMh**」的混合，顯示 Qualcomm 正試驗性地把 ARM 標準引進 CPU 複合體，但整個 SoC-wide resource voting 仍維持 vendor-specific 架構。

## Userspace Interface

與 SCMI、ACPM 一樣，這些韌體介面本身不對 userspace 公開，而是由 Linux 標準框架包裝：

- **cpufreq**：OSM/EPSS（SM8xxx）或 SCMI（X Elite）
- **clk framework**：RPMhCC / SMD RPM clocks
- **regulator**：RPMh regulator / SMD regulator
- **interconnect**：`/sys/kernel/debug/interconnect/` via `icc-rpmh` + BCM voter
- **thermal**：AOSS QMP thermal cooling device
- **debugfs**：`/sys/kernel/debug/qcom_stats/`（`qcom_stats.c`）、`/sys/kernel/debug/qcom_rpmh/`、`qcom_aoss` tracing（`trace-aoss.h`）、`rpmh` tracing（`trace-rpmh.h`）
- **TrustZone / UEFI variable**：`efivarfs` via QSEECOM UEFI secure app

## Android-Specific Notes

`drivers/firmware/qcom/` 與 `drivers/soc/qcom/` **完全是 upstream Linux，沒有 `ANDROID:` 補丁或 vendor hook**。這是最主要 Qualcomm Android 手機的底層韌體介面，能完全 upstream 化是 Qualcomm 長期 upstream 投入的成果。

對 Android GKI：
- 所有 Qualcomm Android 手機依賴 `QCOM_SCM`（TrustZone、modem PAS authentication）和 `QCOM_RPMH` 或 `QCOM_SMD_RPM` 作為 GKI 模組。
- Snapdragon 的 `gki_defconfig` 通常啟用：`QCOM_SCM=y`、`QCOM_RPMH=m`、`QCOM_AOSS_QMP=m`、`QCOM_COMMAND_DB=y`、`QCOM_SPM=m`、`QCOM_QSEECOM=m`、`QCOM_QSEECOM_UEFISECAPP=m`。
- Samsung Galaxy S 系列（Snapdragon 版）、OnePlus、小米、Sony Xperia 等均用此棧。

ACK 中的 Qualcomm 驅動**不涵蓋**以下項目（它們在 Qualcomm vendor tree / Android private branch）：
- 專屬 GPU DVFS（Adreno msm-kgsl，走 RPMh + 自家 governor）
- 專屬 DSP 韌體介面（Hexagon Q6 的 FastRPC，`drivers/misc/fastrpc.c` 部分 upstream 但 vendor 仍有擴展）
- 專屬 modem 通訊增強
- 專屬 camera ISP power control

## Configuration

| Kconfig | 類型 | 說明 |
|---------|------|------|
| `QCOM_SCM` | tristate | SCM/TrustZone SMC 通訊 |
| `QCOM_QSEECOM` | bool | QSEECOM TA 介面（depends on `QCOM_SCM`） |
| `QCOM_QSEECOM_UEFISECAPP` | bool | UEFI 變數 shim |
| `QCOM_TZMEM_MODE_*` | choice | TZ 記憶體分配策略 |
| `QCOM_RPMH` | tristate | RPMh（RSC + client + cmd-db） |
| `QCOM_SMD_RPM` | tristate | 舊版 SMD RPM |
| `QCOM_COMMAND_DB` | bool | RPMh resource 靜態查表 |
| `QCOM_AOSS_QMP` | tristate | AOSS QMP messaging |
| `QCOM_SPM` | tristate | 舊版 SPM/SAW |
| `QCOM_SMEM` | tristate | Shared memory 基礎 |
| `QCOM_SMP2P`, `QCOM_SMSM` | tristate | Subsystem signaling |
| `QCOM_PMIC_GLINK` | tristate | PMIC GLINK channel |
| `QCOM_STATS` | tristate | sleep stats debugfs |

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md) — 包含 Qualcomm 驅動在整體韌體樹中的定位
- [ARM SCMI](./arm-scmi.md) — 標準對照；X Elite CPUCP 採用
- [Google ACPM](./google-acpm.md) — Tensor 的同位選擇
- [SCMI vs 廠商 Firmware 分析](../analyses/scmi-vs-google-acpm.md) — 三方對照與設計取捨
- [Qualcomm SCM Source Summary](../sources/src-drivers-firmware-qcom-scm.md) — 檔案層級摘要
- [Platform Bus](./platform-bus.md) — Qualcomm 驅動多以 platform device 掛載
