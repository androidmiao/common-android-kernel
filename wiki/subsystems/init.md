---
type: subsystem
kernel_path: "init/"
upstream: partial
android_patches: "Kconfig.gki hidden configuration and GKI boot defaults"
vendor_hooks: []
related:
  - ../concepts/gki.md
  - ../concepts/kconfig-and-build.md
  - ../subsystems/security.md
  - ../subsystems/memory-management.md
  - ../subsystems/filesystems.md
  - ../data-structures/task_struct.md
last_updated: 2026-04-25
---

# init/ 子系統分析

## 概述

`init/` 目錄是 Linux 核心啟動流程的核心，負責從架構特定的組合語言入口點接管控制權後，完成整個核心的初始化，最終啟動使用者空間的第一個行程（PID 1）。這個子系統涵蓋了從 `start_kernel()` 到 `/sbin/init` 執行的完整路徑。

## 目錄結構

```
init/
├── Kconfig              # 核心組態選項定義（編譯器、通用核心選項等）
├── Kconfig.gki          # Android GKI (Generic Kernel Image) 專用隱藏組態
├── Makefile             # 建構規則
├── .gitignore
├── .kunitconfig         # KUnit 測試組態
├── main.c               # 核心啟動主流程 (start_kernel, kernel_init)
├── init_task.c          # 初始任務 (PID 0 / swapper) 的靜態定義
├── calibrate.c          # BogoMIPS 延遲校正
├── do_mounts.c          # 根檔案系統掛載
├── do_mounts.h          # 掛載子系統內部標頭
├── do_mounts_initrd.c   # 傳統 initrd 支援（已標記為棄用）
├── do_mounts_rd.c       # RAM disk 映像載入與識別
├── initramfs.c          # initramfs (cpio) 解壓與填充
├── initramfs_internal.h # initramfs 內部標頭
├── initramfs_test.c     # initramfs 的 KUnit 測試
├── noinitramfs.c        # 無 initramfs 時的最小 rootfs 建立
├── version.c            # 核心版本資訊與 UTS namespace
└── version-timestamp.c  # 建構時間戳與 linux_banner
```

## 啟動流程

### 階段一：start_kernel()（main.c:1004）

這是核心的 C 語言入口點，由架構特定的組合語言程式碼呼叫。此函式標記為 `asmlinkage __visible __init __no_sanitize_address __noreturn __no_stack_protector`，表示它從組合語言呼叫、不可返回、且需要特殊的安全屬性。

主要初始化順序如下：

1. **早期基礎設施**：`set_task_stack_end_magic`、`smp_setup_processor_id`、`debug_objects_early_init`
2. **Cgroup 早期初始化**：`cgroup_init_early`
3. **中斷禁用階段**：`local_irq_disable` → `boot_cpu_init` → `page_address_init`
4. **架構設定**：`setup_arch` — 解析裝置樹、設定記憶體布局等
5. **安全框架**：`jump_label_init` → `static_call_init` → `early_security_init`
6. **Boot Config**：`setup_boot_config` — 解析 bootconfig 附加組態
7. **命令列處理**：`setup_command_line` → `parse_early_param` → `parse_args`
8. **記憶體管理**：`mm_core_init`、`vfs_caches_init_early`
9. **排程器**：`sched_init`
10. **RCU**：`rcu_init`、`kvfree_rcu_init`
11. **中斷與計時器**：`early_irq_init` → `init_IRQ` → `tick_init` → `timers_init` → `hrtimers_init`
12. **時間管理**：`timekeeping_init` → `time_init`
13. **安全增強**：`random_init`、`kfence_init`、`boot_init_stack_canary`
14. **中斷啟用**：`local_irq_enable`
15. **主控台**：`console_init`
16. **記憶體進階初始化**：`calibrate_delay`、`fork_init`、`proc_caches_init`
17. **VFS 與行程管理**：`vfs_caches_init`、`signals_init`、`proc_root_init`
18. **Cgroup**：`cgroup_init`
19. **完成**：`rest_init` — 建立 kernel_init 與 kthreadd 執行緒

### 階段二：rest_init()（main.c:711）

此函式建立兩個關鍵核心執行緒：

1. **kernel_init**（PID 1）：透過 `user_mode_thread` 建立，最終演變為使用者空間的 init 行程
2. **kthreadd**（PID 2）：透過 `kernel_thread` 建立，作為所有核心執行緒的父行程

完成後，啟動 CPU 進入 idle 迴圈 (`cpu_startup_entry`)。

### 階段三：kernel_init_freeable()（main.c:1659）

在 kthreadd 就緒後執行：

1. **SMP 啟動**：`smp_prepare_cpus` → `smp_init` → `sched_init_smp`
2. **基本設定**：`do_basic_setup` — 包含驅動程式核心初始化、建構函式執行、initcall 執行
3. **KUnit 測試**：`kunit_run_all_tests`
4. **等待 initramfs**：`wait_for_initramfs` → `console_on_rootfs`
5. **準備命名空間**：`prepare_namespace` — 掛載根檔案系統

### 階段四：kernel_init()（main.c:1569）

完成使用者空間轉換：

1. 釋放 init 記憶體段 (`free_initmem`)
2. 標記核心記憶體為唯讀 (`mark_readonly`)
3. 設定系統狀態為 `SYSTEM_RUNNING`
4. 依序嘗試執行 init 行程：`ramdisk_execute_command`（預設 `/init`）→ `execute_command`（由 `init=` 指定）→ `/sbin/init` → `/etc/init` → `/bin/init` → `/bin/sh`

## 核心檔案詳細分析

### main.c — 核心啟動主控制器

此檔案約 1700 行，是整個啟動流程的骨幹。

**命令列處理機制**：核心維護多個命令列副本：`boot_command_line`（原始）、`saved_command_line`（含 bootconfig 擴展）、`static_command_line`（用於參數解析）。Boot Config 機制允許透過 initrd 末尾附加的結構化組態來擴展命令列。

**Initcall 機制**：核心模組初始化透過分級的 initcall 系統組織，共 8 個等級：pure(0) → core(1) → postcore(2) → arch(3) → subsys(4) → fs(5) → device(6) → late(7)。每個等級的 initcall 函式指標存放在特定的 linker section 中，由 `do_initcalls` 依序執行。

**Initcall 除錯**：透過 `initcall_debug` 核心參數，可以追蹤每個 initcall 的執行時間。支援 tracepoint 機制和 `initcall_blacklist` 來禁用特定初始化函式。

**關鍵全域變數**：
- `system_state`：追蹤系統狀態（`SYSTEM_BOOTING` → `SYSTEM_SCHEDULING` → `SYSTEM_FREEING_INITMEM` → `SYSTEM_RUNNING`）
- `loops_per_jiffy`：CPU 延遲校正值
- `reset_devices`：控制裝置重設行為（用於 kdump 等場景）

### init_task.c — 初始任務

靜態定義了核心的第一個任務結構 `init_task`（即 swapper/idle 任務，PID 0）。這是所有行程的祖先，包含：

- **排程屬性**：`SCHED_NORMAL` 策略，優先權 `MAX_PRIO - 20`
- **初始憑證** (`init_cred`)：以 root 身份運行，擁有全部 capabilities
- **訊號處理** (`init_signals`, `init_sighand`)：預設訊號配置
- **Shadow Call Stack**：`init_shadow_call_stack`（`CONFIG_SHADOW_CALL_STACK`），為 ARM64 安全功能提供初始堆疊

### calibrate.c — 延遲校正

實現 BogoMIPS 校正，決定 `loops_per_jiffy` 值，用於精確的忙碌等待延遲。

提供兩種校正方法：
1. **直接校正** (`calibrate_delay_direct`)：使用硬體計時器（如 TSC），進行多次測量並剔除異常值
2. **收斂校正** (`calibrate_delay_converge`)：透過二分搜尋法逐步逼近正確值

支援 `lpj=` 核心參數來跳過校正，直接指定值。

### do_mounts.c — 根檔案系統掛載

處理根檔案系統的探測與掛載，是啟動流程中從核心態過渡到使用者態的關鍵環節。

**掛載策略**：根據 `ROOT_DEV` 的值選擇不同的掛載路徑：
- `Root_NFS`：NFS 網路根檔案系統
- `Root_CIFS`：SMB/CIFS 網路根檔案系統
- `Root_Generic`：通用掛載（mtd、ubi 裝置）
- 預設：區塊裝置掛載

**支援的核心參數**：`root=`、`rootwait`、`rootwait=<秒>`、`rootflags=`、`rootfstype=`、`rootdelay=`、`ro`、`rw`

**rootfs 類型**：透過 `init_rootfs` 決定使用 `tmpfs` 或 `ramfs` 作為初始根檔案系統。

### do_mounts_initrd.c — 傳統 initrd

實現傳統的 initrd（initial ramdisk）載入機制。此機制已被標記為棄用，建議使用 initramfs。

流程：載入 initrd 映像到 `/dev/ram0` → 掛載為根 → 執行 `/linuxrc` → 切換到真正的根裝置

支援 `noinitrd`、`initrdmem=`、`initrd=` 核心參數，以及透過 `/proc/sys/kernel/real-root-dev` sysctl 動態指定真實根裝置。

### do_mounts_rd.c — RAM Disk 映像

負責識別和載入 RAM disk 映像。支援多種檔案系統格式的偵測：minix、ext2、romfs、cramfs、squashfs，以及壓縮格式：gzip、bzip2、lzma、xz、lzo、lz4。

### initramfs.c — initramfs 處理

實現 cpio 格式的 initramfs 解壓與檔案系統填充，這是現代核心啟動的主要機制。

**cpio 解析**：實作了一個有限狀態機（FSM），狀態包括 Start → Collect → GotHeader → SkipIt → GotName → CopyFile → GotSymlink → Reset。

**硬連結支援**：透過 hash table 追蹤 inode 以正確建立硬連結。

**記憶體管理**：
- `reserve_initrd_mem`：在 memblock 中保留 initrd 記憶體區域
- `free_initrd_mem`：啟動完成後釋放 initrd 佔用的記憶體
- `kexec_free_initrd`：處理 crashkernel 區域重疊的特殊情況

**異步載入**：`populate_rootfs` 使用 `async_schedule_domain` 實現異步初始化，可透過 `initramfs_async=` 參數控制。

**核心參數**：`retain_initrd`（保留 initrd 記憶體）、`keepinitrd`（同上，架構特定）、`initramfs_async=`

### noinitramfs.c — 最小根檔案系統

當未啟用 `CONFIG_BLK_DEV_INITRD` 時，建立一個最小的 rootfs：`/dev`（目錄）、`/dev/console`（字元裝置 5:1）、`/root`（目錄）。

### version.c / version-timestamp.c — 版本資訊

管理核心版本字串和 UTS (Unix Timesharing System) namespace。`version.c` 定義了 `linux_proc_banner`（用於 `/proc/version`），`version-timestamp.c` 定義了 `init_uts_ns`（UTS namespace 初始值）和 `linux_banner`（啟動時顯示的版本橫幅）。

`version.c` 使用 `__weak` 符號宣告，允許 `version-timestamp.c`（在最終建構步驟中編譯）覆蓋這些值，確保時間戳準確。

支援 `hostname=` 早期核心參數來設定初始主機名稱。

## 建構系統

### Makefile

核心目標檔案：
- **必建**：`main.o`、`version.o`、`mounts.o`（`do_mounts.o` 的別名）、`init_task.o`
- **條件編譯**：
  - `initramfs.o`（需要 `CONFIG_BLK_DEV_INITRD`）
  - `noinitramfs.o`（不需要 `CONFIG_BLK_DEV_INITRD` 時）
  - `calibrate.o`（需要 `CONFIG_GENERIC_CALIBRATE_DELAY`）
  - `do_mounts_rd.o`（需要 `CONFIG_BLK_DEV_RAM`）
  - `do_mounts_initrd.o`（需要 `CONFIG_BLK_DEV_INITRD`）

特殊建構旗標：`-fno-function-sections -fno-data-sections`，確保程式碼不被分割到不同的 section。

版本時間戳透過 `filechk_uts_version` 機制生成，支援 `KBUILD_BUILD_VERSION` 和 `KBUILD_BUILD_TIMESTAMP` 環境變數覆蓋。

### Kconfig

`init/Kconfig` 是整個核心組態系統的入口點，定義了：
- 編譯器偵測（GCC/Clang 版本）
- 通用核心選項（SMP、Preemption、Cgroups、Namespaces 等）
- Init 相關選項（`CONFIG_DEFAULT_INIT`、`CONFIG_BOOT_CONFIG`）

### Kconfig.gki

Android GKI 專用組態，定義了一系列 `GKI_HIDDEN_*` 選項，用於在樹外模組編譯場景中正確啟用被隱式依賴的核心組態（如 DRM、Regmap、Crypto、Sound 等子系統的輔助選項）。

## Android / GKI 相關注意事項

1. **Shadow Call Stack** (`init_task.c`)：`CONFIG_SHADOW_CALL_STACK` 為 ARM64 安全功能，在 init_task 中分配初始的影子呼叫堆疊
2. **sched_ext** (`init_task.c`)：`CONFIG_SCHED_CLASS_EXT` 支援可擴展排程類別，是 Android 效能調優的重要機制
3. **Kconfig.gki**：大量的隱藏組態選項反映了 GKI 模組化架構的需求，讓樹外模組能正確連結核心提供的基礎設施
4. **initramfs 異步載入**：Android 裝置的快速啟動需求使得異步 initramfs 載入特別重要
5. **Boot Config**：Android 的 bootloader 經常透過 bootconfig 機制傳遞額外的核心參數

## 核心參數參考

| 參數 | 檔案 | 說明 |
|------|------|------|
| `init=` | main.c | 指定自訂 init 程式路徑 |
| `rdinit=` | main.c | 指定 ramdisk init 程式路徑 |
| `debug` | main.c | 設定控制台日誌等級為 DEBUG |
| `quiet` | main.c | 設定控制台日誌等級為 QUIET |
| `loglevel=` | main.c | 設定控制台日誌等級 |
| `bootconfig` | main.c | 啟用 boot config 機制 |
| `reset_devices` | main.c | 強制裝置重設（用於 kdump） |
| `rodata=` | main.c | 控制核心記憶體唯讀保護 |
| `initcall_debug` | main.c | 追蹤 initcall 執行時間 |
| `initcall_blacklist=` | main.c | 禁用特定 initcall |
| `randomize_kstack_offset=` | main.c | 控制核心堆疊隨機化 |
| `root=` | do_mounts.c | 指定根裝置 |
| `rootwait` | do_mounts.c | 無限等待根裝置就緒 |
| `rootwait=` | do_mounts.c | 限時等待根裝置（秒） |
| `rootflags=` | do_mounts.c | 根檔案系統掛載旗標 |
| `rootfstype=` | do_mounts.c | 指定根檔案系統類型 |
| `rootdelay=` | do_mounts.c | 掛載前延遲（秒） |
| `ro` / `rw` | do_mounts.c | 以唯讀/讀寫方式掛載根 |
| `noinitrd` | do_mounts_initrd.c | 禁用 initrd |
| `initrdmem=` / `initrd=` | do_mounts_initrd.c | 指定 initrd 記憶體位置 |
| `retain_initrd` | initramfs.c | 保留 initrd 記憶體不釋放 |
| `keepinitrd` | initramfs.c | 同上（架構特定） |
| `initramfs_async=` | initramfs.c | 控制異步 initramfs 載入 |
| `lpj=` | calibrate.c | 預設 loops_per_jiffy 值 |
| `load_ramdisk=` | do_mounts.c | 已棄用，被忽略 |
| `prompt_ramdisk=` | do_mounts_rd.c | 已棄用，被忽略 |
| `ramdisk_start=` | do_mounts_rd.c | RAM disk 映像起始區塊 |
| `hostname=` | version.c | 設定初始主機名稱 |
