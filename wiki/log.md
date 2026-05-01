# Wiki Log

## [2026-04-20] cleanup | `task_struct.md` 全頁繁體中文化

使用者要求「全部使用繁體中文」。把 `data-structures/task_struct.md` 從混合的簡中內容整份重寫為繁體中文，同時做術語本地化（台灣慣用法）：

- 進程（process，保留）、執行緒（thread）、記憶體（memory）、訊號（signal）、佇列（queue）、函式（function）、實作（implement）、憑證（credential）、擷取/存取（access）、即時（realtime）、遮罩（mask）、參數（parameter）、檔案描述符（file descriptor）、位址空間（address space）、上下文切換（context switch）、架構（architecture）、排程類別（scheduling class）⋯等。
- 程式碼 block 內的註解一併轉繁。
- 修正原頁面的壞連結：`../subsystems/memory.md` → `../subsystems/memory-management.md`、`../subsystems/binder.md` → `../entities/binder.md`、`../concepts/signals.md`（不存在，移除）、`../concepts/vendor_hooks.md` → `../concepts/vendor-hooks.md`。
- 新增一條 Cross-Reference：ABI 穩定性機制（對應 ANDROID_VENDOR_DATA / OEM 巨集）。

**未變更**：frontmatter 內容、結構層級、章節順序與 Mermaid 圖。只動語言 + 壞連結修正。

---

## [2026-04-20] query + update | `struct task_struct` 定義位置 + forward declaration 地圖

回答使用者問題「struct task_struct 是在哪邊定義？」。雖然 `data-structures/task_struct.md` frontmatter 原本就已標註 `defined_in: common/include/linux/sched.h:821`，但頁面缺少「完整定義 vs forward declaration」的區分，以及 forward decl 為何散落各處 header 的解釋。

**核心發現**：
- 完整定義唯一出現在 `common/include/linux/sched.h:821`。
- `struct task_struct;` forward declaration 在 `common/include/` 下共 **50 個**檔案出現，遍布排程子 header、鎖／等待／同步、訊號／權能／IPC、檔案 I/O、除錯／追蹤、sanitizer、架構抽象層，以及 5 個 Android vendor hook 標頭（`trace/hooks/{sched,signal,sys,fpsimd,mpam}.h`）。
- 設計動機：`sched.h` 是數千行的巨型 header，若每個 API header 都直接 include 它，編譯時間會爆炸；呼叫端只需要 `struct task_struct *` 的場合（絕大多數）就改用 forward decl。

**更新**：
- `data-structures/task_struct.md` — 在「Purpose」之後、「Definition」之前新增「Where It Is Defined」段落：列出完整定義位置、按子系統分類的 forward declaration 分布、以及設計理由。同步更新 `last_updated: 2026-04-20`。
- `log.md` — 本條目。

**未變更**：index.md（無新頁面建立）、index 頁對 task_struct 的條目摘要仍有效。

---

## [2026-04-20] follow-up | Qualcomm 韌體堆疊加入 SCMI/ACPM 三方對照

接續稍早關於 ARM SCMI 與 Google ACPM 的分析，回答使用者問題「那 Qualcomm 是什麼做法？」。把 Qualcomm 整理擴充為三方對照。

**核心發現**：
- Qualcomm 的 AP↔SoC 韌體介面**不是單一協定**，而是多層平行協定堆疊，分布在 `drivers/firmware/qcom/`（8 檔、4,447 行）與 `drivers/soc/qcom/`（RPMh/RSC/Command DB/BCM Voter/AOSS QMP/SPM/SMD-RPM）。
- 三代演進：(1) SDM8xx 以前的 SMD-RPM；(2) SDM845 以後的 RPMh + RSC + Command DB + BCM Voter + AOSS QMP + SPM + SCM 主流棧；(3) Snapdragon X Elite（Hamoa, `qcom,x1e80100`）開始引入 **CPUCP + ARM SCMI**（僅 Perf protocol 0x13）作為 CPU 效能介面。
- 硬體加速：RPMh 透過 **RSC 內的 TCS 暫存器**允許 SLEEP/WAKE TCS 在 CPU 進入 deep sleep 時由硬體自動送出 vote，**完全不需 CPU 執行 SW**，這是 SCMI 難以等價實現的關鍵。
- 證據統計：ACK 樹中 `rpmh-rsc` compatible 出現於 **22 個 DTSI**；`arm,scmi` 僅出現於 **1 個**（X Elite hamoa.dtsi）。
- Qualcomm 主流路線堅持 RPMh 而非轉 SCMI 的 5 項理由：TCS 硬體旁路、BCM aggregation 協定空缺、OSM/EPSS 已比 SCMI 更快、客戶/BSP 生態慣性、X Elite 的策略性部分採用。

**新增/擴充頁面**：
- `entities/qualcomm-firmware-stack.md`（新建）— Qualcomm 完整韌體堆疊實體頁：三代演進、source layout、DT 證據鏈、consumer 驅動表、Kconfig。
- `analyses/scmi-vs-google-acpm.md`（擴充）— 標題改為「AP↔SCP 韌體介面三方對照」。新增 B 段 Qualcomm 證據鏈（B.1–B.4 含 compatible 統計、SM8550 DT 結構、X Elite hamoa.dtsi 片段、Qualcomm 功能對映表）、C 段三方對照總表（16 維度 × 4 欄），以及「為何 Qualcomm 主流路線堅持 RPMh 不轉 SCMI」的 5 項理由。
- `entities/arm-scmi.md`（cross-link 更新）— 加入 qualcomm-firmware-stack 相關連結與說明。
- `entities/google-acpm.md`（cross-link 更新）— 加入 qualcomm-firmware-stack 相關連結與說明。

**更新**：
- `index.md` — Entities 區塊新增 qualcomm-firmware-stack；Analyses 項目更名「SCMI vs Google ACPM 分析」為「AP↔SCP 韌體介面三方對照」並更新摘要。
- `log.md` — 本條目。

---

## [2026-04-20] query + ingest | ARM SCMI 實作細節 + Google Tensor 是否使用 SCMI

回答使用者問題「ARM SCMI 的實作說明，以及 Google SoC Accelerator Firmware 是否採用 SCMI 設計」。

**核心發現**：
- ARM SCMI 完整實作在 `drivers/firmware/arm_scmi/`（31 檔、19,481 行），分 10 個 protocol（Base/Power/System/Perf/Clock/Sensor/Reset/Voltage/Powercap/Pinctrl）、4 種 transport（mailbox/SMC/OP-TEE/virtio），有 SCMI bus、notification、raw mode、quirks framework 等子系統。
- `vendors/` 目錄目前僅 `imx` 一家 upstream vendor 擴展。
- **Google Tensor GS101（Pixel 6 系列）並未採用 ARM SCMI**。DT `firmware` 節點宣告 `google,gs101-acpm-ipc`（gs101.dtsi:486-493），由 `drivers/firmware/samsung/exynos-acpm.c:772-778` 的 of_match table 驅動承載——沿用 Samsung Exynos 的 ACPM（Alive Clock and Power Manager）私有協定。
- CPU DVFS clock 掛在 `acpm_ipc` 上（gs101.dtsi:76 起），不在 `scmi_clk`/`scmi_perf` 上；`drivers/firmware/arm_scmi/vendors/google/` 不存在；`drivers/firmware/google/` 全是 Chromebook coreboot 相關，與 accelerator 無關。
- Pixel 的 TPU/GXP accelerator 驅動不在 ACK，位於 Google vendor kernel (`private/google-modules/...`)，故 ACK 範圍內無可分析的 "Google SoC Accelerator Firmware" SCMI 介面。

**建立的 Wiki 頁面**：
- `entities/arm-scmi.md` — ARM SCMI 實作深度解析（protocol 分層、xfer 生命週期、transport 抽象、SCMI bus model、notification 子系統、raw mode、quirks）
- `entities/google-acpm.md` — Google Tensor GS101 ACPM 實作（DT binding、shmem layout、seqnum 機制、DVFS/PMIC 客戶端、clock provider 角色）
- `analyses/scmi-vs-google-acpm.md` — 兩者對照分析：證據鏈（DT/binding/clock/driver/vendors 目錄）、10 個 SCMI protocol 到 GS101 實作的功能對映表、設計成因、未來 SCMI 化可能路徑

**更新**：
- `index.md` — Entities 區塊新增 arm-scmi 與 google-acpm；Analyses 區塊新增 scmi-vs-google-acpm；Coverage Dashboard 的 Analyses 列從 6 → 7
- `log.md` — 本條目

---

## [2026-04-10] ingest | drivers/firmware/ 韌體驅動子系統完整分析 — 8 個頁面建立

深度分析 `drivers/firmware/` 目錄（201 個 .c/.h 檔、15 個子目錄 + 21 個獨立檔案、~99,851 行程式碼），建立 8 個 Wiki 頁面：

**新增頁面：**
1. `subsystems/firmware.md` — 韌體驅動子系統總覽：四種通訊模式（SMC/Mailbox/MMIO/Hypervisor）、六大分類、Android 重要性評估、關鍵程式碼路徑
2. `sources/src-drivers-firmware-Kconfig.md` — 頂層 Kconfig（302 行）：17 個配置選項 + 14 個子目錄引入
3. `sources/src-drivers-firmware-arm-scmi.md` — ARM SCMI（31 檔, 19,481 行）：多傳輸後端、協定分層
4. `sources/src-drivers-firmware-qcom-scm.md` — Qualcomm SCM/QSEECOM（8 檔, 4,447 行）：TrustZone 通訊
5. `sources/src-drivers-firmware-samsung-acpm.md` — Samsung ACPM（6 檔, 1,111 行）：Exynos 電源管理
6. `sources/src-drivers-firmware-google.md` — Google Coreboot（12 檔, 2,359 行）：coreboot 表 bus
7. `sources/src-drivers-firmware-arm-psci.md` — ARM PSCI/SMCCC/FF-A（9 檔, 4,114 行）：CPU 熱插拔基礎
8. `sources/src-drivers-firmware-efi.md` — EFI/UEFI（75 檔, 17,024 行）：runtime services + libstub

**更新頁面：**
- `index.md` — 新增 Firmware Drivers 子系統條目、7 個 source summary 條目、Coverage Dashboard 更新

## [2026-04-09] ingest | include/ 核心標頭檔目錄完整分析 — 4 個頁面建立

深度分析 `include/` 目錄（6,593 個 `.h` 檔、33 個頂層子目錄），建立 4 個 Wiki 頁面。

**建立的頁面：**

1. **subsystems/include-headers.md** — Include Headers 子系統完整分析
   - 33 個子目錄分類統計：核心 API 層（linux/2,789 + uapi/964 + asm-generic/152）、子系統標頭（net/383、drm/148、sound/203、media/111、crypto/82）、硬體平台（dt-bindings/1,122、soc/74）、追蹤（trace/219、含 29 個 vendor hook 標頭）
   - UAPI/內部 API 分離機制詳述
   - Android 專屬標頭完整分析：android_kabi.h、android_vendor.h、trace/hooks/*（29 檔 1,121 行）、uapi/linux/android/*（3 檔 713 行）
   - linux/ 82 個子目錄分類說明

2. **concepts/kernel-headers-organization.md** — Headers 組織概念
   - 三層標頭分離（UAPI → linux/ → asm-generic）
   - asm-generic 回退機制（67 個 mandatory headers）
   - Device Tree Bindings 共享常數模式
   - Trace Events 雙重 include 模式

3. **sources/src-include-linux-android_kabi-h.md** — android_kabi.h 原始碼摘要
   - 11 個關鍵巨集分析、_Static_assert 保護、gendwarfksyms 整合

4. **sources/src-include-linux-android_vendor-h.md** — android_vendor.h 原始碼摘要
   - VENDOR/OEM 雙層資料空間、與 vendor hooks 的配合模式

**更新的頁面：** index.md（新增 Subsystems/Concepts/Sources 條目、Coverage Dashboard）

---

## [2026-04-09] ingest | Samples 範例程式碼目錄分析 — 2 個頁面建立

深度分析 `samples/` 目錄（46 個子目錄、289 個檔案、40,417 行程式碼），建立 2 個 Wiki 頁面。

**建立的頁面：**

1. **subsystems/samples.md** — Samples 子系統完整分析
   - 46 個子目錄分類：追蹤與偵錯（12）、驅動框架（8）、網路與 BPF（4）、安全機制（4）、核心基礎設施（7）、虛擬化（4）、Rust（1）、Android（1）
   - BPF 範例集深度分析：~100 檔案、17,797 行、XDP/TC/Socket/追蹤/LWT/HBM 六大類
   - Rust 核心範例分析：15 個模組、展示 module!/Platform/PCI/I2C/USB/Faux/DMA/DebugFS/ConfigFS
   - Ftrace 範例：direct function/multi/ops/trace array 四種模式
   - Livepatch 範例：基本修補、回呼機制、shadow 變數三階段
   - DAMON 範例：WSSE/PRCL/MTIER 三種記憶體監控場景
   - Android 相關：binderfs 範例（namespace 隔離、裝置動態建立）、BPF cookie-UID helper

2. **sources/src-samples-Kconfig.md** — Kconfig 配置檔摘要
   - ~53 個配置選項完整列表
   - 兩類編譯模型（核心模組 vs 使用者空間程式）
   - 架構限制與依賴關係

**更新的頁面：** index.md（Subsystems 新增 Samples 條目、Sources 新增 samples/Kconfig、Coverage Dashboard 新增 Samples 列）

---

## [2026-04-09] ingest | ARM/ARM64/RISC-V 架構分析 — 3 個頁面建立

深度分析三個處理器架構目錄（`arch/arm`、`arch/arm64`、`arch/riscv`），建立 3 個 Wiki 頁面。

**建立的頁面：**

1. **subsystems/arch-arm.md** — ARM 32-bit 架構完整分析
   - 4,559 個檔案、145 個子目錄、55 個 mach-* 平台目錄
   - ARMv3m 至 ARMv7-M 全系列 CPU 支援
   - 76 個 defconfig、2,771 個 Device Tree 檔案
   - 安全特性（Stack Protector/KASAN/CFI）、加密子系統（AES-CE/NEON）
   - 無 Android 特定修改，GKI 不再支援 ARM32

2. **subsystems/arch-arm64.md** — ARM64 架構完整分析（GKI 主要目標）
   - 3,489 個檔案（38 MB）、40+ SoC 平台、2,907 個 Device Tree
   - gki_defconfig 詳細分析（691 行）：Binder/Vendor Hooks/Debug Kinfo
   - 安全機制深度分析：PAC/MTE/SCS/CFI/BTI 五層防護
   - KVM/pKVM 虛擬化（Google 工程師主導）
   - 加密子系統（40 個檔案，Google LLC 貢獻）
   - GKI Fragment Configs：db845c/amlogic/rockpi4/microdroid

3. **subsystems/arch-riscv.md** — RISC-V 架構完整分析
   - 564 個檔案、17 個 SoC 平台、70+ 個 Device Tree
   - 模組化 ISA 設計：20+ 個擴展配置選項
   - KVM 虛擬化（含 AIA 進階中斷架構）
   - ZVK 向量加密擴展
   - CPU 勘誤修正（Andes/SiFive/T-HEAD/StarFive）
   - 與 ARM64 的詳細功能對比表
   - 無 Android 特定修改，尚未納入 GKI 支援

**更新的頁面：** index.md（Subsystems 新增 3 個架構條目、Coverage Dashboard 新增 Architecture 列）

---

## [2026-04-09] ingest | Init 子系統完整分析 — 1 個頁面建立

深度分析核心啟動子系統（`init/`，~1,700 行核心程式碼），建立 1 個 Wiki 頁面。

**建立的頁面：**

1. **subsystems/init.md** — Init 子系統完整分析（245 行）
   - 涵蓋全部 16 個檔案：main.c、init_task.c、calibrate.c、do_mounts.c、do_mounts.h、do_mounts_initrd.c、do_mounts_rd.c、initramfs.c、initramfs_internal.h、initramfs_test.c、noinitramfs.c、version.c、version-timestamp.c、Makefile、Kconfig、Kconfig.gki
   - 四階段啟動流程：start_kernel → rest_init → kernel_init_freeable → kernel_init
   - Initcall 分級機制（pure/core/postcore/arch/subsys/fs/device/late 8 個等級）
   - 根檔案系統掛載策略（NFS/CIFS/Block/nodev 四種路徑）
   - initramfs cpio 有限狀態機解析器（8 個狀態）
   - init_task（PID 0）靜態定義：排程屬性、初始憑證、Shadow Call Stack
   - BogoMIPS 延遲校正：直接校正與收斂校正兩種方法
   - Boot Config 機制：initrd 末尾附加結構化組態
   - Android/GKI 注意事項：Shadow Call Stack、sched_ext、Kconfig.gki 隱藏組態
   - 26 個核心啟動參數對照表

**更新的頁面：** index.md（Subsystems 新增 Init 條目、Coverage Dashboard 新增 Init 列）

---

## [2026-04-09] ingest | Driver Framework 完整分析 — 6 個頁面建立

深度分析核心裝置驅動框架（`drivers/base/`，~27,800 行），建立 6 個 Wiki 頁面。

**建立的頁面：**

1. **subsystems/driver-framework.md** — 子系統完整分析
   - 目錄結構（30+ 檔案）、架構圖（Mermaid）、內部資料結構（subsys_private/driver_private/device_private）
   - 四條關鍵程式碼路徑：裝置註冊、驅動探測、延遲探測重試、fw_devlink 連結建立
   - Kconfig 選項分析、EXPORT_SYMBOL 統計（~149 個）
   - Android 修改評估：核心檔案零修補，僅 arch_topology.c 含 2 個 vendor hooks

2. **concepts/driver-model.md** — 驅動模型概念
   - Bus-Device-Driver 三層架構設計原理
   - 六大機制：匹配綁定、Deferred Probing、fw_devlink、Device Links、Devres、Component 框架
   - Faux Bus（2025 新增）、sysfs 拓撲、Platform Driver 使用模式
   - Android 最小化修改策略分析

3. **data-structures/device.md** — `struct device` 資料結構
   - 欄位群組（身份/匯流排/PM/DMA/devres/fwnode）、生命週期八階段、關鍵操作函式表

4. **data-structures/device_driver.md** — `struct device_driver` 資料結構
   - 匹配表優先順序、探測策略、PM ops、生命週期五階段

5. **data-structures/bus_type.md** — `struct bus_type` 資料結構
   - match/probe 策略、subsys_private 內部狀態、主要匯流排實例表

6. **entities/platform-bus.md** — Platform Bus 實體分析
   - platform_device/platform_driver 結構、五級匹配流程、資源存取 API、platform_bus_type 定義

**更新的頁面：** index.md（新增 6 個條目、更新 Coverage Dashboard：Driver Framework → Complete）

**關鍵發現：**
- Driver Framework 核心（core.c/bus.c/dd.c/driver.c/class.c/platform.c）在 ACK 中與上游 Linux 完全一致
- 僅 arch_topology.c 含 2 個 Android vendor hooks（thermal_stats、topology_flags）
- 此為 GKI 刻意設計：驅動框架作為穩定基礎 API，不允許廠商修改

---

## [2026-04-09] apis | 核心 API 介面全面分析 — 5 個 API 頁面建立

深度分析核心五大 API 介面類型，建立 `wiki/apis/` 下全部 5 個頁面，總計約 1,500 行中文 Wiki。

**建立的頁面：**

1. **apis/ioctl-interfaces.md** — ioctl 介面
   - 命令編碼格式（direction/size/type/number 四欄位）
   - VFS 派發機制：`SYSCALL_DEFINE3(ioctl)` → `do_vfs_ioctl()` → `vfs_ioctl()`
   - 1,834 個命令（132 標頭檔、40+ magic number 群組）
   - Android Binder ioctl（13 命令 + BC 21 + BR 23）、Binderfs 控制
   - 主要群組：V4L2(116)、Input(18)、DRM、TTY(30)、USB(55)
   - 32/64-bit 相容處理

2. **apis/netlink.md** — Netlink 介面
   - AF_NETLINK socket 架構：af_netlink.c(2,953行) + genetlink.c(1,997行)
   - 17 個協定家族（ROUTE/NETFILTER/XFRM/KOBJECT_UEVENT/GENERIC 等）
   - Generic Netlink 框架：IDR 動態 ID 分配、genl_family 結構、20+ 家族
   - RTNetlink 詳述：RTM_* 訊息類型、handler 註冊、屬性策略
   - Android Binder Netlink 家族：錯誤報告與交易除錯
   - 訊息格式：nlmsghdr + nlattr 結構

3. **apis/sysfs-procfs.md** — sysfs/procfs 介面
   - sysfs：kobject 基礎設施、DEVICE_ATTR/BUS_ATTR/DRIVER_ATTR/CLASS_ATTR 巨集
   - 屬性群組與 is_visible() 動態控制
   - procfs：proc_ops 結構、6 種 proc_create API 變體
   - seq_file 介面：start/stop/next/show 回呼模式
   - /proc/[pid]/ 行程資訊（base.c 98,726 行）
   - Android 整合：Trusty sysfs、cpufreq times、LSM hook

4. **apis/kfuncs-bpf.md** — BPF kfuncs 介面
   - 45 個 kfunc 集合（BTF_KFUNCS_START）、31 個檔案
   - 16 種 KF_* 旗標（ACQUIRE/RELEASE/SLEEPABLE/DESTRUCTIVE/ITER 等）
   - 主要集合：generic(47)、common(88+)、cpumask(28)、arena(3)、crypto(5)
   - 網路 kfuncs：skb/XDP/sock_addr/TCP 六個集合
   - 傳統 BPF helpers：104 個 BPF_CALL_N、157 個 bpf_func_proto
   - kfunc vs helper 比較、驗證器檢查

5. **apis/syscalls.md** — 系統呼叫介面
   - 471 個系統呼叫（__NR_syscalls = 471，編號 0-470）
   - ARM64 呼叫慣例（X0-X5 引數、X8 syscall number）
   - 入口流程：el0_svc → do_el0_svc() → el0_svc_common() → sys_call_table
   - SYSCALL_DEFINE/wrapper 巨集機制
   - 系統呼叫分類（行程/檔案/記憶體/網路/IPC/信號/排程/安全）
   - 無 Android 專屬 syscall，擴充透過 ioctl/vendor hooks/BPF

**驗證：** 交叉比對原始碼行號、函式簽名、資料結構定義。

Pages created: `apis/ioctl-interfaces.md`, `apis/netlink.md`, `apis/sysfs-procfs.md`, `apis/kfuncs-bpf.md`, `apis/syscalls.md`
Pages updated: `index.md`（APIs 區段、Coverage Dashboard 新增 APIs 列）

## [2026-04-09] android | drivers/android/ 目錄分析 — Android-Specific 頁面建立

深度分析 `common/drivers/android/` 目錄及相關標頭檔（`android_kabi.h`、`android_vendor.h`、`include/trace/hooks/`、`include/uapi/linux/android/`），建立 `wiki/android/` 下 6 個主題頁面。

**涵蓋範圍：**
- `drivers/android/` 全部 44 個檔案（~21,700 行），含 C 版 Binder (7,374 行)、Rust 版 Binder (9,080 行)、Binderfs、binder_alloc、vendor_hooks、debug_kinfo
- 8 個 Kconfig 選項分析（ANDROID_BINDER_IPC / RUST、BINDERFS、VENDOR_HOOKS、DEBUG_KINFO、KABI_RESERVE、VENDOR_OEM_DATA）
- 29 個 vendor hook 標頭檔、~141 個 hooks 完整分類統計
- KABI 保留機制（android_kabi.h）與 Vendor/OEM 資料填充（android_vendor.h）
- Android 修補政策（ANDROID:/BACKPORT:/FROMGIT:/FROMLIST:）
- GKI 模組管理與 KMI 符號保護

**建立的頁面：**
1. `android/drivers-android-overview.md` — 目錄總覽、元件分類、編譯依賴圖
2. `android/abi-stability.md` — ABI 穩定性四層機制
3. `android/vendor-hook-catalogue.md` — 全部 hook 標頭檔清單與統計
4. `android/android-patches-policy.md` — 修補分類與上游化進程
5. `android/debug-kinfo.md` — debug_kinfo 驅動詳細分析
6. `android/gki-modules-list.md` — GKI 模組分類與管理

Pages created: 6 pages in `android/`
Pages updated: `index.md`（新增 Android-Specific 區段 6 個連結、Coverage Dashboard 更新）

## [2026-04-08] init | Wiki bootstrap
Created wiki structure and schema. Directories: `subsystems/`, `concepts/`, `entities/`, `data-structures/`, `apis/`, `android/`, `analyses/`, `sources/`. Schema documented in `CLAUDE.md`. Index and overview pages created. No source files ingested yet.

Pages created: `CLAUDE.md`, `index.md`, `log.md`, `overview.md`

## [2026-04-08] scheduler | Scheduler 子系統完整分析

深度分析 `kernel/sched/` 目錄下全部 35 個原始碼檔案 (~62,743 行)，產出 765 行中文 Wiki。

**涵蓋範圍：**
- 核心排程類別：stop、deadline (EDF/CBS)、RT (FIFO/RR)、fair (EEVDF)、sched_ext (BPF)、idle
- 核心資料結構：`struct rq`、`cfs_rq`、`rt_rq`、`dl_rq`、`sched_entity`、`sched_class`，附原始碼行號
- EEVDF 演算法：vruntime 更新、deadline 計算、資格判定、紅黑樹搜尋、lag preservation
- 負載追蹤 (PELT)：指數衰減、三項指標 (load/runnable/util)、階層聚合、clock scaling
- 負載均衡：多層排程域、group_type 分類、misfit 任務、NUMA 感知
- CPU 頻率調節 (schedutil)：utilization-based scaling、IO boost、rate limiting
- 核心排程 (Core Scheduling)：SMT 安全、cookie 機制
- PSI：SOME/FULL 壓力追蹤、三資源 (CPU/memory/IO)
- Android Vendor Hooks：78 個 tracepoint 分類與用途
- Feature flags：EEVDF 相關、負載均衡、延遲出隊等，含預設值
- 建置系統：build_policy / build_utility 編譯單元、CONFIG 選項
- Syscall 介面、統計/除錯、輔助機制 (clock、wait queue、completion、membarrier、isolation、autogroup、cpuacct)

**驗證：** 交叉比對原始碼行號與資料結構位置，修正 PLACE_REL_DEADLINE 預設值 (OFF→ON) 及 vendor hooks 數量 (90+→78)。

Pages created: `subsystems/scheduler.md`
Pages updated: `index.md` (新增 Scheduler 連結、Coverage Dashboard 狀態更新為 Complete)

## [2026-04-08] concepts | 全部 11 個 Concept 頁面建立

深度分析核心原始碼，建立 `concepts/` 目錄下全部 11 個跨系統概念頁面，總計約 2500 行中文 Wiki。

**建立的頁面：**

1. **concepts/gki.md** — Generic Kernel Image 架構
   - KMI 穩定性機制（gki_module.c 符號保護、bsearch）
   - ABI 監控工具鏈（build/kernel/abi/）
   - 隱藏配置（Kconfig.gki, GKI_HACKS_TO_FIX）
   - System DLKM 模組清單、構建產出

2. **concepts/vendor-hooks.md** — Vendor Hook 框架
   - 130 個 hook、29 個標頭檔完整分類統計
   - DECLARE_HOOK vs DECLARE_RESTRICTED_HOOK 機制差異
   - static_call 優化、tracepoint 繼承關係
   - 排程子系統 78 個 hook 範例

3. **concepts/kconfig-and-build.md** — Kconfig 與 Build 系統
   - Kleaf/Bazel 架構（kernel_build、gki_artifacts 規則）
   - GKI defconfig 785 行關鍵配置分析
   - build.config 廢棄遷移狀態
   - Defconfig fragment 疊加機制

4. **concepts/module-system.md** — 可載入核心模組系統
   - struct module、kernel_symbol 資料結構
   - EXPORT_SYMBOL 變體與符號搜尋流程
   - GKI 簽署 vs 未簽署模組存取控制
   - 模組載入完整流程（load_module → do_init_module）

5. **concepts/locking-primitives.md** — 鎖定原語
   - Spinlock（qspinlock MCS 壓縮）、Mutex（OSQ 適應性自旋）
   - RWSem（55 位讀者計數）、Seqlock（讀者重試）
   - RT Mutex 優先權繼承、Lockdep 死鎖檢測

6. **concepts/interrupt-handling.md** — 中斷處理
   - irq_desc/irq_data/irqaction 資料結構
   - Top-half 流程（GIC → handle_irq → handler）
   - Softirq 10 種、Tasklet、Workqueue
   - 執行緒化中斷（IRQF_THREAD、IRQF_ONESHOT）

7. **concepts/memory-allocation.md** — 記憶體分配
   - Buddy 系統、Per-CPU 頁面快取
   - GFP 標誌分類、SLUB 分配器三級鎖定
   - vmalloc、CMA、DMA 池、Per-CPU 分配器

8. **concepts/rcu.md** — RCU 機制
   - 四種 RCU 實現（Tree/Preempt/Tiny/SRCU）
   - Grace Period、QS 檢測、樹階層結構
   - Callback 分段列表、NOCB 離載
   - RCU 受保護列表 API

9. **concepts/tracing-and-ftrace.md** — 追蹤與 ftrace
   - Tracepoint 基礎設施（struct tracepoint）
   - ftrace 動態函數追蹤、Ring Buffer
   - Trace Events（TRACE_EVENT 巨集）、Kprobes
   - 與 Vendor Hooks 的繼承關係

10. **concepts/bpf.md** — BPF 子系統
    - 33 種程式類型、30 種映射類型
    - 驗證器（25403 行）靜態分析機制
    - Helpers/kfuncs、sched_ext、BTF
    - Android 網路/安全/排程應用

11. **concepts/rust-in-kernel.md** — Rust in Kernel
    - 96 個核心抽象子模組
    - FFI 整合（bindgen、helpers）
    - Rust Binder 實現
    - 安全性抽象、Pin-Init 框架

**驗證：** 所有頁面包含 YAML frontmatter、原始碼行號引用、交叉參考連結。

Pages created: `concepts/gki.md`, `concepts/vendor-hooks.md`, `concepts/kconfig-and-build.md`, `concepts/module-system.md`, `concepts/locking-primitives.md`, `concepts/interrupt-handling.md`, `concepts/memory-allocation.md`, `concepts/rcu.md`, `concepts/tracing-and-ftrace.md`, `concepts/bpf.md`, `concepts/rust-in-kernel.md`
Pages updated: `index.md` (新增 11 個 Concept 連結與摘要、Coverage Dashboard 更新)

## [2026-04-08] memory-management | 記憶體管理子系統完整分析

深度分析 `mm/` 目錄下約 150 個原始碼檔案（核心 .c 檔案超過 140,000 行），產出約 450 行中文 Wiki。

**涵蓋範圍：**
- 目錄地圖：Top 30 個最大檔案的行數、用途說明
- 架構圖：Mermaid 流程圖展示子系統各層次關係
- 核心設計概念：雙層分配架構、需求分頁、CoW、頁面回收、Maple Tree VMA、Folio 抽象
- 關鍵資料結構：struct page (mm_types.h:79)、struct folio (:401)、struct vm_area_struct (:904)、struct mm_struct (:1075)、struct ptdesc (:572)
- 5 條關鍵程式碼路徑（附原始碼行號）：頁面分配、Page Fault、kswapd 回收、OOM Killer、mmap
- 子系統組件詳述：Buddy Allocator、SLUB、Page Cache、Multi-Gen LRU、Compaction、Memory Cgroup、THP、KSM、Swap/zswap
- Android 特定變更：2 個 vendor hooks（vmscan balance + mmap check）、memfd-ashmem shim（214 行）、MGLRU 預設啟用、memfd LUO、Ashmem Rust 實作
- 配置選項：20+ 個 Kconfig 選項、DAMON 子模組 7 個選項
- 交叉參考：連結到 7 個相關 wiki 頁面

**驗證：** 所有原始碼行號引用已交叉比對，vendor hooks 數量與匯出位置已確認。

Pages created: `subsystems/memory-management.md`
Pages updated: `index.md` (新增 Memory Management 連結、Coverage Dashboard 更新為 Complete)

## [2026-04-08] networking | 網路子系統完整分析

深度分析 `net/` 目錄下約 1,534 個 .c 原始碼檔案（~1,249,652 行），產出約 650 行中文 Wiki。

**涵蓋範圍：**
- 架構總覽：分層協定架構圖（Socket → 傳輸 → 網路 → 鏈路 → 驅動）
- 核心資料結構：`struct sk_buff` (skbuff.h:885)、`struct net_device` (netdevice.h:2105)、`struct sock` (sock.h:359)，附原始碼行號
- Socket 層：7 個主要 syscall 入口點（socket.c），socket 建立流程
- TCP/IP 堆疊：IPv4 初始化 (af_inet.c:1891)、TCP 關鍵函式 8 個、IP 層函式 6 個
- 封包接收路徑：NIC → NAPI → netif_receive_skb → ip_rcv → tcp_rcv_established 完整流程
- 封包傳送路徑：tcp_sendmsg → ip_queue_xmit → dev_queue_xmit 完整流程
- Netfilter 框架：5 個 hook point、nf_tables_api.c (12,334 行)
- Traffic Control：qdisc/classifier/action 架構
- BPF/XDP 整合：socket filter (filter.c:12,580 行)、XDP 動作、Android cgroup BPF 流量控制
- 無線網路：cfg80211 (45,980 行)、mac80211 (77,402 行)
- 藍牙：61,703 行、HCI/L2CAP/SCO/BLE
- QRTR：Qualcomm IPC Router (2,612 行)、3 種傳輸層
- 效能機制：NAPI、GRO、RPS/RFS、XPS、BQL、Page Pool
- Android 特定：僅 2 個 vendor hooks（android_vh_ptype_head、android_vh_do_wake_up_sync）
- 配置選項：20+ 個關鍵 Kconfig 選項
- 檔案索引：核心框架 18 個主要檔案、TCP/IP 堆疊 8 個主要檔案

**驗證：** 所有原始碼行號引用已交叉比對，vendor hooks 位置與參數已確認。

Pages created: `subsystems/networking.md`
Pages updated: `index.md` (新增 Networking 連結、Coverage Dashboard 狀態更新為 Complete)

## [2026-04-08] filesystems | 檔案系統子系統完整分析

深度分析 `fs/` 目錄下約 157 個項目（71 個核心 .c 檔案、75+ 個檔案系統子目錄），產出約 430 行中文 Wiki。

**涵蓋範圍：**
- 目錄地圖：核心 VFS 檔案與主要檔案系統子目錄的行數、用途說明
- 架構圖：Mermaid 流程圖展示 VFS 分層架構（使用者空間介面 → VFS 核心 → Page Cache/I/O → 安全/通知 → 具體檔案系統）
- 核心設計概念：VFS 四大物件模型、dcache、fs_context 新式掛載 API、Page Cache 整合、fscrypt/fsverity 框架、fsnotify
- 關鍵資料結構：struct super_block、struct inode (fs.h:765)、struct dentry (dcache.h:92)、struct file、struct file_system_type、struct address_space、struct mnt_namespace、struct mount
- 5 條關鍵程式碼路徑（附函式呼叫鏈）：open、path lookup、mount、read、write
- Android 關鍵檔案系統詳述：ext4 (54 檔/~2.0MB)、f2fs (34 檔/~1.4MB)、erofs (25 檔/~308KB)、fuse (26 檔/~560KB)、overlayfs (18 檔/~388KB)、incfs (21 檔/Android 專屬)
- 安全與完整性：fscrypt (15 檔/~228KB) 檔案級加密、fsverity (13 檔/~92KB) Merkle tree 驗證
- 通知子系統：inotify/fanotify/dnotify (14 檔/~96KB)
- 虛擬檔案系統：procfs (38 檔)、sysfs (10 檔)、kernfs (10 檔)
- Android 特定變更：1 個 vendor hook (android_vh_check_file_open @ open.c:939)、incfs 增量安裝、FUSE passthrough 最佳化、fscrypt Android 相容模式、f2fs SQLite 調校、EROFS Android 使用情境
- VFS 操作結構體參考：super_operations、inode_operations、file_operations 完整函式列表
- 配置選項：18+ 個核心 Kconfig 選項、各檔案系統子選項
- 交叉參考：連結到 6 個相關 wiki 頁面

**驗證：** vendor hook 位置 (fs/open.c:939) 已由原始碼確認，vendor hook 標頭檔路徑修正為 include/trace/hooks/syscall_check.h。

Pages created: `subsystems/filesystems.md`
Pages updated: `index.md` (新增 Filesystems 連結、Coverage Dashboard 更新為 Complete)、`log.md`

## [2026-04-08] block-layer | Block Layer 子系統完整分析

深度分析 `block/` 目錄下 80+ 個原始碼檔案（約 64,528 行），產出約 350 行中文 Wiki。

**涵蓋範圍：**
- 目錄地圖：Top 30 個檔案的行數、用途說明
- 架構圖：Mermaid 流程圖展示 submit_bio → blk-mq → driver → completion 完整路徑
- Multi-Queue (blk-mq) 三層佇列架構：Software Queue (`blk_mq_ctx`)、Hardware Queue (`blk_mq_hw_ctx`)、Request Queue (`request_queue`)
- 核心資料結構：`struct bio` (blk_types.h:210)、`struct request` (blk-mq.h:103)、`struct request_queue` (blkdev.h:479)、`struct blk_mq_hw_ctx` (blk-mq.h:320)
- 5 條關鍵程式碼路徑（附原始碼行號）：I/O 提交、blk-mq 派送、完成、Queue Freeze/Quiesce、Flush/FUA 狀態機
- Plugging 機制：BLK_MAX_REQUEST_COUNT (32)、BLK_PLUG_FLUSH_SIZE (128KB)、自動 sleep flush
- 三種 I/O 排程器：MQ Deadline（deadline+FIFO 雙列）、Kyber（延遲導向佇列深度控制）、BFQ（B-WF2Q+ 頻寬公平）
- Request QoS 框架：WBT (CoDel writeback throttling)、iolatency (延遲保護)、iocost (成本模型比例控制)
- Block Cgroup：blk-throttle (BPS/IOPS)、ioprio、iocost/iolatency 整合
- Inline Encryption (blk-crypto)：4 種加密模式、硬體 profile、軟體 fallback
- Zoned Block Devices：zone write plugging、zone append、zone management
- Android 特定變更：僅 DM Default Key 整合 (bio.c:274 bi_skip_dm_default_key)、TEST_MAPPING
- **無 Android vendor hooks**——UFSHCD (9 hooks) 和 IOMMU (3 hooks) 在更低層提供擴展
- 配置選項：18+ 個 Kconfig 選項、重要內部常數
- 309 個 EXPORT_SYMBOL（跨 39 個檔案）

**驗證：** 所有原始碼行號引用已交叉比對，DM_DEFAULT_KEY 位置已確認，vendor hooks 缺失已經由 include/trace/hooks/ 目錄搜尋證實。

Pages created: `subsystems/block-layer.md`
Pages updated: `index.md` (新增 Block Layer 連結、Coverage Dashboard 更新為 Complete)、`log.md`

## [2026-04-09] security | Security 子系統完整分析

深度分析 `security/` 目錄下 163 個 .c 原始碼檔案（約 110,586 行），產出約 350 行中文 Wiki。

**涵蓋範圍：**
- LSM 框架核心：273 個 hook 定義（lsm_hook_defs.h）、static call 派發機制、17 種 blob 類型管理、三階段初始化流程
- LSM 排序機制：ORDER_FIRST/MUTABLE/LAST、FLAG_EXCLUSIVE 互斥、`lsm=` 命令列覆蓋
- SELinux（25,424 行）：217 個 hooks、AVC 快取（hash table + LRU）、Security Server（policy 載入/存取向量計算）、selinuxfs
- 次要 LSM：AppArmor (80 hooks)、SMACK (122 hooks)、TOMOYO (30 hooks)、Landlock (34 hooks)、SafeSetID (4 hooks)、Yama (4 hooks)、LoadPin (3 hooks)、Lockdown (1 hook)、IPE (10 hooks)、BPF LSM
- Capability 模組（commoncap.c）：17 hooks、POSIX Capabilities、ORDER_FIRST
- 支援子系統：Key Retention Service (13,508 行)、IMA/EVM 完整性 (12,462 行)、Device Cgroup
- 核心強化：INIT_STACK_ALL_ZERO、INIT_ON_ALLOC、FORTIFY_SOURCE、HARDENED_USERCOPY、CFI、Shadow Call Stack、SLAB_FREELIST_HARDENED、RANDSTRUCT
- GKI defconfig：SELinux + Landlock + SafeSetID + BPF LSM + CFI + SCS + FORTIFY
- 5 個關鍵程式碼路徑：LSM 初始化、Hook 派發、SELinux 存取決策、Capability 檢查、Key Retention
- Android vendor hooks：5 個 restricted hooks（4 個 AVC + 1 個 SELinux state），均為 DECLARE_RESTRICTED_HOOK
- Binder 安全 hooks：4 個 SELinux Binder 專用 hooks

**驗證：** LSM hook 數量 273 由 lsm_hook_defs.h grep 確認；SELinux 217 hooks 由 hooks.c LSM_HOOK_INIT grep 確認；vendor hooks 位置由 include/trace/hooks/avc.h 和 selinux.h 確認；GKI defconfig 配置由 gki_defconfig grep 確認。

Pages created: `subsystems/security.md`
Pages updated: `index.md` (新增 Security 連結、Coverage Dashboard 更新為 Complete)、`log.md`

## [2026-04-09] ipc | IPC 子系統分析與 Wiki 更新

分析 `ipc/` 目錄下全部 14 個原始碼檔案（約 9,889 行），驗證既有 Wiki 內容並修正。

**涵蓋範圍：**
- 三層架構：系統呼叫介面（System V syscalls + POSIX mqueue）→ 核心基礎設施（IDR + rhashtable ID 管理、權限檢查、RCU）→ 物件實現（SHM/SEM/MSG/MQUEUE）
- 核心資料結構：`kern_ipc_perm`（基礎權限）、`ipc_ids`（ID 空間管理）、`ipc_namespace`（容器隔離）、`shmid_kernel`、`sem_array`/`sem`、`msg_queue`/`msg_msg`、`mqueue_inode_info`
- ID 分配機制：15-bit 索引 + 16-bit 序號（預設）/ 24-bit + 7-bit（擴展模式）、循環分配
- 三層鎖定策略：RCU 讀取鎖 → `ipc_ids.rwsem` → `kern_ipc_perm.lock`
- 信號量雙層鎖定：細粒度模式 vs 全域鎖定、`USE_GLOBAL_LOCK_HYSTERESIS = 10`
- 訊息佇列 `pipelined_send()` 零複製直接遞交最佳化
- POSIX mqueue 紅黑樹優先權排序、狀態機無鎖接收
- Namespace 隔離完整生命週期（create/copy/destroy/install）
- 5 條關鍵程式碼路徑（附原始碼行號）
- Android 特定：無 vendor hooks、無 Android 專屬修改（純上游 Linux 程式碼）

**驗證：** 逐一比對 14 個原始碼檔案的函式行號（ipc_addid@278、ipcperms@553、newseg@702、do_shmat@1519、do_semtimedop@2222、find_alloc_undo@1906、do_msgsnd@848、do_msgrcv@1098 等），全部正確。修正 `struct sem_array` 行號引用（130→114）。新增 Makefile 條目至目錄地圖。

Pages updated: `subsystems/ipc.md`（修正 sem_array 行號、新增 Makefile、更新日期）、`index.md`（新增 IPC 連結、Coverage Dashboard 更新為 Complete）、`log.md`

## [2026-04-09] entities | 全部 7 個 Entity 頁面建立

深度分析 `drivers/android/`、`drivers/dma-buf/`、`drivers/staging/android/`、`kernel/cgroup/`、`mm/` 等目錄，建立 `entities/` 目錄下全部 7 個元件實體頁面，總計約 2,000 行中文 Wiki。

**建立的頁面：**

1. **entities/binder.md** — Android Binder IPC 驅動
   - C 實現 10,300 行 + Rust 實現 7,400 行完整分析
   - 三層鎖定架構（outer_lock → node->lock → inner_lock）
   - 8 個核心資料結構附行號（binder_proc:444、binder_thread:526、binder_node:233、binder_ref:329、binder_transaction:568、binder_buffer:41、binder_alloc:107、dbitmap:26）
   - 交易流程完整 walk-through（ioctl → thread_write → transaction → thread_read）
   - 緩衝區分配器（mmap + LRU shrinker）、描述符 bitmap、Netlink 通知
   - ~20 個 tracepoints、模組參數、Kconfig 選項
   - Android 專屬功能：凍結/解凍、垃圾郵件偵測、優先權繼承、SELinux 安全上下文

2. **entities/binderfs.md** — Binderfs 虛擬檔案系統
   - 786 行完整分析：fs_context 掛載機制、binder-control ioctl 裝置建立
   - features/ 目錄（4 個功能旗標）、binder_logs/ 目錄
   - IPC namespace 隔離、動態 minor number 管理

3. **entities/debug-kinfo.md** — 核心除錯資訊匯出
   - 185+69 行分析：struct kernel_info（kallsyms 位址、段位址、頁表）
   - Platform driver probe 流程（Device Tree 保留記憶體）
   - XOR checksum 完整性、build info 模組參數

4. **entities/dmabuf-heaps.md** — DMA-BUF Heaps 框架
   - 三檔案分析：框架層 (414行)、System Heap (556行)、CMA Heap (448行)
   - System Heap 分配策略（order {8,4,0}、SWIOTLB 偵測）
   - 與 Ion 分配器的功能對照表
   - Android 多媒體記憶體共享用途

5. **entities/ashmem.md** — Ashmem 匿名共享記憶體（已廢棄）
   - 989 行核心 + 214 行 memfd 相容層分析
   - Pin/Unpin 機制與 LRU shrinker（FALLOC_FL_PUNCH_HOLE）
   - Memfd-Ashmem Shim ioctl 轉換對映表
   - Rust 重新實現狀態

6. **entities/cgroup-controllers.md** — Cgroup 控制器
   - 核心框架 7,452 行 + 10 個控制器共 ~18,447 行分析
   - CPUset (4,552行)、Freezer (326行)、PIDs (460行)、DMEM (830行) 等
   - 3 個 Android vendor hooks（cgroup_attach、cpu_cgroup_attach、cpu_cgroup_online）
   - Android 使用模式（AMS、lmkd、EAS、thermal）

7. **entities/vendor-hooks-driver.md** — Vendor Hooks 匯出驅動
   - 93 行分析：50 個 EXPORT_TRACEPOINT_SYMBOL_GPL
   - 按子系統分類（信號/CPU/記憶體/UFS/SELinux/IOMMU 等 15+ 子系統）

**驗證：** 所有頁面包含 YAML frontmatter、原始碼行號引用、雙向交叉參考連結。Binder 資料結構行號由 binder_internal.h 確認，vendor hooks 數量由 vendor_hooks.c grep 確認。

Pages created: `entities/binder.md`, `entities/binderfs.md`, `entities/debug-kinfo.md`, `entities/dmabuf-heaps.md`, `entities/ashmem.md`, `entities/cgroup-controllers.md`, `entities/vendor-hooks-driver.md`
Pages updated: `index.md`（新增 7 個 Entity 連結與摘要、Coverage Dashboard Android Components 更新為 Complete）、`log.md`

## [2026-04-09] data-structures | 全部 6 個資料結構頁面建立

深度分析核心原始碼標頭檔與實現檔案，建立 `data-structures/` 目錄下全部 6 個關鍵核心資料結構頁面，總計約 2,800+ 行中文 Wiki。

**建立的頁面：**

1. **data-structures/task_struct.md** — 行程/執行緒描述符
   - 定義於 include/linux/sched.h:821，約 4-5 KB（架構相關）
   - 11 個欄位群組：狀態旗標、排程/CPU 親和性、行程身分/親屬、記憶體管理、檔案系統/I/O、信號/credentials、效能記帳、追蹤除錯、同步原語、RCU、Android 專屬
   - 生命週期：copy_process() (kernel/fork.c:1979) → dup_task_struct() → free_task()
   - Android 專屬：vendor hooks（android_vh_dup_task_struct、android_vh_free_task、android_rvh_sched_fork）、96 bytes vendor/OEM data
   - Mermaid 關係圖

2. **data-structures/mm_struct.md** — 行程虛擬位址空間
   - 定義於 include/linux/mm_types.h，約 2-4 KB（CPU 數量相關）
   - 11 個欄位群組：身分/參考計數、VMA 管理、頁表保護/鎖定、記憶體記帳、位址佈局、排程/CPU 親和性、NUMA、MMU notifier、HugeTLB
   - 生命週期：mm_alloc() (kernel/fork.c:1164) → dup_mm() → mmput() → __mmput()
   - Android 相關：IOMMU mm、MM ID 系統、futex 私有 hash

3. **data-structures/sk_buff.md** — 網路封包緩衝區
   - 定義於 include/linux/skbuff.h:885，約 256 bytes
   - 23 個欄位群組：鏈結/樹節點、裝置/協定、時間戳記、控制緩衝區、路由、Netfilter、長度、克隆、校驗和、流量控制、GSO/GRO 等
   - 生命週期：__alloc_skb() → 層間傳遞 → __kfree_skb()
   - Android 專屬：vendor hooks (include/trace/hooks/net.h)、QRTR 支援

4. **data-structures/binder_proc.md** — Binder 行程描述符
   - 定義於 drivers/android/binder_internal.h:444，約 400-600 bytes
   - 10 個欄位群組：列表/樹結構、行程身分、交易/工作管理、生命週期狀態、記憶體/資源、排程優先權、除錯、並行控制、Android 功能
   - 生命週期：binder_open() (binder.c:6245) → binder_release() → binder_deferred_release()
   - 完全 Android 專屬：凍結機制、垃圾郵件偵測、binderfs 整合

5. **data-structures/inode.md** — VFS inode
   - 定義於 include/linux/fs.h:765，約 600-700 bytes（randomized layout）
   - 13 個欄位群組：基本屬性、時間戳記、區塊裝置、操作指標、列表/hash、狀態管理、版本/序列、dirty 頁面、cgroup writeback、同步原語、類型專屬、安全/通知
   - 生命週期：alloc_inode() → iget5_locked() → iput() → destroy_inode()
   - Android 相關：F2FS/SELinux/ext4/Binder/incfs/APEX 整合

6. **data-structures/page.md** — 實體頁面描述符與 folio
   - struct page 定義於 include/linux/mm_types.h:79，64 bytes
   - struct folio 為現代多頁邏輯單元抽象
   - 欄位群組：flags、LRU/buddy/pcp 列表、mapping/cache、reference counting、memory cgroup、large page
   - 生命週期：啟動初始化 → page allocator → folio API → put_page()/folio_put()
   - page-to-folio 轉型說明
   - Android 相關：Ion/Trusty secure page pool/KGSL GPU/LMK 整合

**驗證：** 所有頁面包含 YAML frontmatter、原始碼行號引用、雙向交叉參考連結、Mermaid 關係圖。欄位群組涵蓋各結構體的架構關鍵欄位。

Pages created: `data-structures/task_struct.md`, `data-structures/mm_struct.md`, `data-structures/sk_buff.md`, `data-structures/binder_proc.md`, `data-structures/inode.md`, `data-structures/page.md`
Pages updated: `index.md`（新增 6 個 Data Structure 連結與摘要、Coverage Dashboard 新增 Data Structures 列）、`log.md`

## [2026-04-09] analyses | 全部 6 個 Analysis 頁面建立

基於已建檔的 7 個子系統、11 個概念、7 個實體、6 個資料結構頁面，產出 6 篇跨子系統深度分析，總計約 3,500 行中文 Wiki。

**建立的頁面：**

1. **analyses/vendor-hooks-distribution.md** — Vendor Hooks 跨子系統分佈分析
   - 130 個 hooks 按子系統統計表（排程器 78/60%、UFS 9、SELinux 5、Cgroup 3、IOMMU 3 等）
   - Restricted vs Unrestricted 設計選擇分析
   - 排程器佔六成的策略含義、安全子系統全部 restricted 的原因
   - Block Layer 與 IPC 零 hooks 的替代擴展機制（sched_ext、BPF LSM、Netfilter）
   - 效能影響分析（static_branch_unlikely NOP、static_call 直接呼叫）

2. **analyses/android-vs-upstream.md** — Android vs Upstream 差異全面分析
   - 四層修改分類框架：Android 專屬元件、Android 補丁、配置差異、完全上游
   - Android 專屬元件清單（Binder、Vendor Hooks、IncFS、Debug Kinfo、memfd-ashmem shim、GKI 基礎設施）
   - 各子系統 Android 補丁詳細表格（修改位置、行號、說明）
   - GKI defconfig vs 上游配置差異（安全強化、LSM 堆疊、Rust 支援）
   - 修改量化：~26,000 行 / < 0.15% 核心程式碼
   - 五項設計哲學（最小侵入、可觀察而非可覆寫、擴展點優於分支、漸進廢棄、Rust 安全替代）

3. **analyses/binder-transaction-flow.md** — Binder 交易流程深度分析
   - 8 階段完整路徑：使用者空間進入 → 目標解析 → SELinux 檢查 → 緩衝區分配 → flat object 轉換 → 優先權繼承 → 交易入列 → 目標讀取 → 回覆
   - 每階段的鎖定序列（outer_lock → node->lock → inner_lock）
   - 效能瓶頸分析（copy_from_user、頁面表操作、inner_lock 競爭、SELinux AVC）
   - 凍結機制與異步垃圾郵件偵測
   - Mermaid 序列圖

4. **analyses/security-hardening-strategy.md** — ACK 安全強化策略分析
   - 六層防禦體系：MAC (SELinux)、補充 LSM、編譯器保護 (CFI/SCS)、記憶體分配器強化、模組/完整性保護、BPF 安全
   - 攻擊類型 vs 防禦機制對照表
   - Vendor hook 對安全的影響（僅觀察、不可覆寫）
   - 與上游 Linux 的安全差異分析

5. **analyses/memory-android-extensions.md** — 記憶體管理 Android 擴展分析
   - MGLRU 預設啟用的設計決策分析
   - 2 個 vendor hooks 的用途與 restricted/unrestricted 選擇原因
   - memfd-ashmem 相容層 ioctl 對映表
   - 外部機制（DMA-BUF heaps、Binder alloc、Cgroup Memory、lmkd）
   - 「最小核心修改 + 豐富外部機制」設計哲學

6. **analyses/locking-patterns.md** — 跨子系統鎖定模式分析
   - 五種鎖定模式：階層 spinlock（Binder/IPC）、per-CPU+RCU（排程器/Block/網路）、rwsem+細粒度（mm/fs）、static call（LSM/Vendor Hooks）、原子操作（sk_buff/page）
   - 8 個子系統鎖定策略比較表
   - IPC 信號量自適應鎖定特殊案例
   - RT Preemption 影響分析
   - Android 效能影響（per-VMA lock、static call、Binder inner_lock 競爭）

**驗證：** 所有分析頁面包含 YAML frontmatter、交叉參考連結。數據來源為已建檔的 wiki 頁面內容，統計數字與各子系統頁面一致。

Pages created: `analyses/vendor-hooks-distribution.md`, `analyses/android-vs-upstream.md`, `analyses/binder-transaction-flow.md`, `analyses/security-hardening-strategy.md`, `analyses/memory-android-extensions.md`, `analyses/locking-patterns.md`
Pages updated: `index.md`（新增 6 個 Analysis 連結與摘要、Coverage Dashboard 新增 Analyses 列）、`log.md`

## [2026-04-09] sources | drivers/android/ 完整原始碼摘要頁面建立

深度分析 `drivers/android/` 目錄下全部 40+ 個原始碼檔案（C 實現約 10,300 行、Rust 實現約 10,000 行、標頭檔約 1,500 行），建立 `sources/` 目錄下 25 個 Source Summary 頁面。

**建立的頁面：**

### C 實現（6 頁）
1. **sources/src-drivers-android-binder-c.md** — Binder IPC 核心（7,374 行）：22 個關鍵函式、三層鎖定、交易日誌、延遲工作、Netlink 報告
2. **sources/src-drivers-android-binder_alloc-c.md** — 緩衝區分配器（1,410 行）：雙紅黑樹、LRU shrinker、KUnit 整合
3. **sources/src-drivers-android-binderfs-c.md** — Binderfs（786 行）：fs_context API、IDA 管理、功能探測
4. **sources/src-drivers-android-vendor_hooks-c.md** — Vendor Hooks 匯出（93 行）：50 個符號、27 個標頭檔、14 restricted + 36 regular
5. **sources/src-drivers-android-debug_kinfo-c.md** — Debug Kinfo（185 行）：platform driver、kallsyms 匯出、XOR checksum
6. **sources/src-drivers-android-binder_netlink-c.md** — Binder Netlink（33 行）：Generic Netlink 家族、多播群組

### 標頭檔（6 頁）
7. **sources/src-drivers-android-binder_internal-h.md** — 核心標頭（637 行）：14 個主要結構、11 種工作類型、統計枚舉
8. **sources/src-drivers-android-binder_alloc-h.md** — 分配器標頭（189 行）：binder_buffer/binder_alloc/shrinker_mdata
9. **sources/src-drivers-android-dbitmap-h.md** — 動態 bitmap（169 行）：grow/shrink 策略、全部 inline
10. **sources/src-drivers-android-binder_trace-h.md** — Trace Event（472 行）：~20 個 tracepoint
11. **sources/src-drivers-android-debug_kinfo-h.md** — Debug Kinfo 標頭（69 行）：__packed 結構
12. **sources/src-drivers-android-binder_netlink-h.md** — Netlink 標頭（22 行）

### Rust 實現（13 頁）
13. **sources/src-drivers-android-binder-rust_binder_main-rs.md** — 頂層模組（611 行）：13 子模組、FFI 橋接
14. **sources/src-drivers-android-binder-process-rs.md** — Process（1,745 行）：Arc 所有權、RBTree、IdPool
15. **sources/src-drivers-android-binder-thread-rs.md** — Thread（1,596 行）：scatter-gather、物件翻譯、SELinux
16. **sources/src-drivers-android-binder-node-rs.md** — Node（1,131 行）：排程狀態機、CritIncrWrapper
17. **sources/src-drivers-android-binder-page_range-rs.md** — 頁面管理（731 行）：Shrinker 包裝、LRU 整合
18. **sources/src-drivers-android-binder-allocation-rs.md** — 分配邏輯（602 行）：RAII Allocation、FileList
19. **sources/src-drivers-android-binder-transaction-rs.md** — Transaction（456 行）：Pin-Init、SpinLock<Option<Allocation>>
20. **sources/src-drivers-android-binder-freeze-rs.md** — 凍結機制（398 行）：FreezeCookie/FreezeListener
21. **sources/src-drivers-android-binder-range_alloc-rs.md** — 範圍分配器（1,068 行）：Tree/Array 雙實現
22. **sources/src-drivers-android-binder-context-rs.md** — Context（180 行）：全域鎖、Manager 節點
23. **sources/src-drivers-android-binder-defs-rs.md** — 常數定義（182 行）：pub_no_prefix! 巨集
24. **sources/src-drivers-android-binder-error-rs.md** — 錯誤型別（100 行）：BinderError、三種 From
25. **sources/src-drivers-android-binder-stats-rs.md** — 統計追蹤（89 行）：原子計數器

### 建置與測試（3 頁）
- **sources/src-drivers-android-Kconfig.md** — 8 個配置選項、C/Rust 互斥
- **sources/src-drivers-android-Makefile.md** — 5 個編譯目標
- **sources/src-drivers-android-tests-binder_alloc_kunit-c.md** — 750,000 個窮舉測試案例

**驗證：** 所有頁面包含 YAML frontmatter、原始碼行號引用、雙向交叉參考連結。函式行號引用已與原始碼交叉比對。

Pages created: 25 個 source summary pages in `sources/`
Pages updated: `index.md`（新增 Sources 區段含 25 個連結、Coverage Dashboard 新增 Source Summaries 列）、`log.md`

## [2026-04-25] analysis | 專案初始導覽

將初次研究 `common-android-mainline` checkout 後的專案導覽整理進 wiki，建立 `analyses/project-orientation.md`。

**重點內容：**

1. **Checkout 身分與版本**
   - 此工作區是 Android `repo` checkout，不是單一 Git repository
   - kernel 主體位於 `common/`，版本為 Linux `6.19.0-rc8`
   - manifest 指向 `common-android-mainline` superproject 與 `kernel/common` 的 `android-mainline`

2. **目錄與子專案分工**
   - `common/`：kernel 原始碼
   - `build/kernel/`：Kleaf/Bazel 建置系統
   - `common-modules/`：Trusty、virtio-media、virtual-device 等共用模組
   - `devices/google/raviole/`：Pixel 6 / GS101 device tree 與 fragments
   - `prebuilts/`、`kernel/tests/`、`test/`：工具鏈與測試支援

3. **建置入口**
   - `tools/bazel build //common:kernel`
   - `tools/bazel build //common:kernel_aarch64`
   - `tools/bazel run //common:kernel_aarch64_dist`
   - `tools/bazel run //common:kernel_aarch64_config`

4. **Android 特有層**
   - `drivers/android/` 是 Binder、Rust Binder、BinderFS、Vendor Hooks、debug_kinfo、KABI/OEM data padding 的主要入口
   - 將 Android driver Kconfig 選項與用途整理成表格

5. **本地狀態注意事項**
   - `common/` 目前有既存未提交修改，集中在 netfilter xtables/UAPI 與 memory model litmus test
   - 後續實作前應先確認這些 diff 是否屬於使用者正在進行的工作

Pages created: `analyses/project-orientation.md`
Pages updated: `index.md`（新增 Analysis 連結、Coverage Dashboard Analyses 由 7 更新為 8）、`log.md`

## [2026-04-25] analysis+lint | Subsystem wiki 健康檢查與第一輪整理

回顧既有 Linux subsystem wiki 分析品質，建立 `analyses/wiki-subsystem-review.md`，並開始依四點優先順序整理。

**已完成：**

1. **記錄 review 建議**
   - 建立 `analyses/wiki-subsystem-review.md`
   - 彙整強項：覆蓋範圍完整、Android/GKI 視角清楚、橫向分析有價值
   - 彙整缺口：格式一致性、壞連結、survey 頁 evidence 不足、統計可能 stale

2. **Wiki hygiene**
   - `subsystems/scheduler.md` 補 YAML frontmatter
   - `subsystems/init.md` 補 YAML frontmatter
   - `subsystems/memory-management.md` 將 `folio` 連結改到既有 `data-structures/page.md`
   - 新增 `data-structures/vm_area_struct.md` 補齊 VMA 資料結構頁

3. **Scheduler cross-links**
   - `subsystems/scheduler.md` 新增「交叉參考」章節
   - 連到 `task_struct`、locking、vendor hooks、BPF、GKI、memory management 與相關 analyses

4. **Survey 頁 evidence snapshot**
   - `subsystems/drivers-overview.md` 補 `drivers/Kconfig`、`drivers/Makefile`、`drivers/android/Kconfig` 證據表
   - `subsystems/firmware.md` 補 `drivers/firmware/Kconfig` 證據表
   - `subsystems/include-headers.md` 補 Android KABI、vendor hooks、Binder UAPI 證據表
   - `subsystems/samples.md` 補 `samples/Kconfig`、`samples/Makefile` 證據表
   - `subsystems/arch-arm.md`、`subsystems/arch-arm64.md`、`subsystems/arch-riscv.md` 補架構 Kconfig/Makefile/defconfig 證據表

5. **Consistency checks**
   - 檢查 `wiki/subsystems/`、`wiki/analyses/`、`wiki/data-structures/` frontmatter：無缺漏
   - 檢查 markdown file links：修正 `data-structures/inode.md` 中指向未建立頁面的 cross-links
   - 修正 `wiki/CLAUDE.md` 中 relative link 範例
   - 檢查 survey 頁 `Evidence Snapshot`：已新增 7 個

Pages created: `analyses/wiki-subsystem-review.md`, `data-structures/vm_area_struct.md`
Pages updated: `subsystems/scheduler.md`, `subsystems/init.md`, `subsystems/memory-management.md`, `subsystems/drivers-overview.md`, `subsystems/firmware.md`, `subsystems/include-headers.md`, `subsystems/samples.md`, `subsystems/arch-arm.md`, `subsystems/arch-arm64.md`, `subsystems/arch-riscv.md`, `data-structures/inode.md`, `CLAUDE.md`, `index.md`, `log.md`

## [2026-05-01] schema-update | 新增 wiki 同步到 GitHub repo 工作流程

將 wiki 鏡像到獨立 GitHub repo 的同步流程文件化，避免每次 session 重新摸索。

**已完成：**

1. **同步目前 wiki 到 GitHub repo**
   - 來源：`/Users/alex.miao/android-kernel/common-android-mainline/wiki/`(source of truth)
   - 目標：`/Users/alex.miao/Documents/.../GitHub/common-android-kernel/wiki/`(mirror)
   - 指令：`rsync -av --update --exclude='.DS_Store'`
   - 結果：41 個檔案被覆寫或新建；同步後兩端各 112 個檔案(已排除 `.DS_Store`)，完全一致

2. **CLAUDE.md 新增第 10 節「Mirroring to GitHub Repository」**
   - 記錄來源/鏡像路徑、`rsync` 指令、政策說明
   - 政策：使用 `--update`(只覆寫較新檔案)、排除 `.DS_Store`、不啟用 `--delete`(保留鏡像端額外檔案，需明確要求才剪除)
   - 同步後驗證步驟：比對來源/目標檔案數
   - 註明 Cowork session 需先 `request_cowork_directory` 取得 GitHub 資料夾存取權

3. **再同步 CLAUDE.md 變更到 GitHub mirror**
   - 兩端 `CLAUDE.md` 皆 17,733 bytes、皆有第 10 節

Pages updated: `CLAUDE.md`(新增 §10)、`log.md`
