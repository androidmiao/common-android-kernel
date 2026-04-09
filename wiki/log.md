# Wiki Log

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
