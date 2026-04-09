# Wiki Index

> Android Common Kernel (ACK) — `common-android-mainline` — Linux 6.19-rc8
> Total: 36,348 `.c` files, 26,402 `.h` files, 148 driver categories

## Overview
- [Architecture Overview](overview.md) — High-level map of the entire ACK, subsystem relationships, Android extensions

## Subsystems
- [Scheduler](subsystems/scheduler.md) — EEVDF/CFS、RT、Deadline、sched_ext、PELT、負載均衡、Android vendor hooks 完整分析
- [Memory Management](subsystems/memory-management.md) — Buddy/SLUB 分配器、Page Fault/CoW、Page Cache、kswapd/Multi-Gen LRU 回收、Compaction、OOM、THP、Swap/zswap、Memory Cgroup、DAMON、KSM、Android vendor hooks 與 memfd-ashmem 相容層
- [Networking](subsystems/networking.md) — Socket 層、TCP/IP 協定堆疊 (IPv4/IPv6)、Netfilter 封包過濾、Traffic Control (QoS)、BPF/XDP 整合、WiFi (mac80211)、藍牙、QRTR、NAPI/GRO/RPS 效能機制、2 個 Android vendor hooks
- [Filesystems](subsystems/filesystems.md) — VFS 核心層（super_block/inode/dentry/file 四大物件）、ext4/f2fs/erofs/fuse/overlayfs、incfs 增量安裝、fscrypt 加密/fsverity 驗證、路徑查詢/掛載/讀寫程式碼路徑、1 個 Android vendor hook
- [Block Layer](subsystems/block-layer.md) — Multi-Queue blk-mq 架構、三層佇列（sw/hw/request_queue）、三種 I/O 排程器（MQ Deadline/Kyber/BFQ）、Request QoS（WBT/iolatency/iocost）、Inline Encryption (blk-crypto)、Zoned Devices、Block Cgroup、無 Android vendor hooks
- [Security](subsystems/security.md) — LSM 框架（273 hooks、static call 派發）、SELinux（217 hooks、AVC 快取）、Capability、Landlock、SafeSetID、BPF LSM、Keys、IMA/EVM 完整性、核心強化（CFI/SCS/FORTIFY）、5 個 Android restricted vendor hooks
- [IPC](subsystems/ipc.md) — System V IPC（共享記憶體/信號量/訊息佇列）、POSIX mqueue、IDR+rhashtable ID 管理、RCU 無鎖讀取、雙層鎖定策略、Namespace 隔離、無 Android vendor hooks

## Concepts
- [GKI](concepts/gki.md) — Generic Kernel Image 架構：KMI 穩定性、符號保護、隱藏配置、System DLKM、ABI 監控工具鏈
- [Vendor Hooks](concepts/vendor-hooks.md) — 廠商 Hook 框架：130 個 hook（29 標頭檔）、DECLARE_HOOK vs DECLARE_RESTRICTED_HOOK、tracepoint 繼承、靜態呼叫優化
- [Kconfig 與 Build 系統](concepts/kconfig-and-build.md) — Kleaf/Bazel 構建系統、GKI defconfig、Kconfig.gki 隱藏配置、defconfig fragment 機制
- [Module 系統](concepts/module-system.md) — 可載入模組：EXPORT_SYMBOL、GKI 符號保護（gki_module.c）、模組簽署、載入流程
- [鎖定原語](concepts/locking-primitives.md) — Spinlock（qspinlock）、Mutex、RWSem、Seqlock、RT Mutex（優先權繼承）、Lockdep 死鎖檢測
- [中斷處理](concepts/interrupt-handling.md) — IRQ 描述符、流程處理器、Top-half/Bottom-half、Softirq（10 種）、Tasklet、執行緒化中斷、IRQ 域映射
- [記憶體分配](concepts/memory-allocation.md) — Buddy 系統、GFP 標誌、SLUB 分配器、vmalloc、CMA、DMA 池、Per-CPU 分配器
- [RCU](concepts/rcu.md) — Read-Copy-Update：Tree RCU、Preempt RCU、SRCU、Grace Period、QS 檢測、NOCB 離載、分段 callback 列表
- [追蹤與 ftrace](concepts/tracing-and-ftrace.md) — Tracepoint 基礎設施、ftrace 函數追蹤、Trace Events、Ring Buffer、Kprobes、與 Vendor Hooks 的關係
- [BPF](concepts/bpf.md) — 33 種程式類型、30 種映射類型、驗證器、Helpers/kfuncs、sched_ext、BTF、Android 網路/安全應用
- [Rust in Kernel](concepts/rust-in-kernel.md) — Rust 核心抽象（96 子模組）、FFI 整合、Rust Binder、安全性抽象、Pin-Init

## Entities
- [Binder](entities/binder.md) — Android IPC 機制：7,374 行 C + Rust 重新實現、三層鎖定架構、交易處理、凍結/解凍、優先權繼承
- [Binderfs](entities/binderfs.md) — Binder 裝置動態管理檔案系統：動態裝置建立、IPC namespace 隔離、功能探測
- [Debug Kinfo](entities/debug-kinfo.md) — 核心除錯資訊匯出：kallsyms 位址、段佈局、保留記憶體供 bootloader crash dump
- [DMA-BUF Heaps](entities/dmabuf-heaps.md) — DMA 緩衝區分配框架：System Heap (buddy) + CMA Heap、Ion 替代品、多媒體記憶體共享
- [Ashmem](entities/ashmem.md) — 匿名共享記憶體（已廢棄）：pin/unpin 語意、memfd-ashmem 相容層、Rust 實現
- [Cgroup Controllers](entities/cgroup-controllers.md) — 行程資源群組控制器：CPUset/Freezer/PIDs/Memory/Block/DMEM、3 個 Android vendor hooks
- [Vendor Hooks Driver](entities/vendor-hooks-driver.md) — 廠商 Hook 匯出驅動：50 個 EXPORT_TRACEPOINT_SYMBOL_GPL、跨 15+ 子系統

## Data Structures
- [`task_struct`](data-structures/task_struct.md) — 行程/執行緒描述符：排程狀態、CPU 親和性、記憶體管理、信號、credentials、RCU、Android vendor hooks（96 bytes vendor data）
- [`mm_struct`](data-structures/mm_struct.md) — 行程虛擬位址空間：VMA Maple Tree、頁表、RSS 記帳、記憶體佈局、IOMMU 整合、futex 私有 hash
- [`sk_buff`](data-structures/sk_buff.md) — 網路封包緩衝區：記憶體布局、參考計數、GSO/GRO、層間傳遞、生命週期、Android vendor hooks
- [`binder_proc`](data-structures/binder_proc.md) — Binder 行程描述符：執行緒/節點/引用管理、交易佇列、凍結機制、動態 bitmap、Android 專屬
- [`inode`](data-structures/inode.md) — VFS inode：檔案屬性、操作指標、address_space、dcache 連結、安全標籤、Android F2FS/EROFS/incfs 整合
- [`page`/`folio`](data-structures/page.md) — 實體頁面描述符：buddy 系統、LRU 列表、page cache、reference counting、folio 轉型、Android Ion/Trusty/KGSL 整合

## APIs
- [ioctl 介面](apis/ioctl-interfaces.md) — ioctl 命令編碼、VFS 派發、1,834 個命令（132 標頭檔）、Binder/V4L2/DRM/Input/TTY 關鍵介面
- [Netlink](apis/netlink.md) — AF_NETLINK socket 通訊、17 個協定家族、20+ Generic Netlink 家族、RTNetlink/Netfilter/uevent、Binder Netlink 錯誤報告
- [sysfs/procfs](apis/sysfs-procfs.md) — 虛擬檔案系統介面：sysfs kobject 屬性巨集（DEVICE_ATTR 等）、procfs proc_ops/seq_file、/proc/[pid]/、/proc/sys/
- [BPF kfuncs](apis/kfuncs-bpf.md) — BPF 核心函式呼叫：45 個 kfunc 集合（31 檔案）、BTF 旗標系統、157 個傳統 helper proto、cpumask/crypto/network kfuncs
- [系統呼叫](apis/syscalls.md) — ARM64 系統呼叫介面：471 個 syscall、el0_svc 入口流程、SYSCALL_DEFINE 巨集、wrapper 機制、無 Android 專屬 syscall

## Android-Specific
- [drivers/android/ 目錄總覽](android/drivers-android-overview.md) — drivers/android/ 完整目錄結構、8 個 Kconfig 選項、元件分類（Binder/Vendor Hooks/Debug Kinfo）、編譯依賴圖
- [ABI 穩定性機制](android/abi-stability.md) — KABI 保留填充（android_kabi.h）、Vendor/OEM 資料填充（android_vendor.h）、GKI 模組保護、ABI 監控工具鏈
- [Vendor Hook 完整目錄](android/vendor-hook-catalogue.md) — 29 個標頭檔、~141 個 hooks、按子系統分類統計、vendor_hooks.c 匯出的 ~50 個符號
- [Android 修補分類與政策](android/android-patches-policy.md) — ANDROID:/BACKPORT:/FROMGIT:/FROMLIST: 標籤分類、上游化進程、與 GKI 的關係
- [Debug Kinfo 詳細分析](android/debug-kinfo.md) — debug_kinfo 平台驅動、kernel_info 結構、Device Tree 綁定、bootloader crash dump 流程
- [GKI 模組清單與管理](android/gki-modules-list.md) — GKI/Vendor/OEM 模組分類、KMI 符號保護、模組載入順序

## Analyses
- [Vendor Hooks 跨子系統分佈分析](analyses/vendor-hooks-distribution.md) — 130 個 hooks 的分佈模式、restricted vs unrestricted 設計選擇、排程器佔 60% 的策略含義
- [Android vs Upstream 差異全面分析](analyses/android-vs-upstream.md) — ACK 四層修改分類（專屬元件/補丁/配置/完全上游）、~26,000 行修改量化、設計哲學
- [Binder 交易流程深度分析](analyses/binder-transaction-flow.md) — 8 階段完整交易路徑、三層鎖定序列、效能瓶頸、凍結機制、Mermaid 序列圖
- [ACK 安全強化策略分析](analyses/security-hardening-strategy.md) — 六層防禦體系（MAC/LSM/編譯器/分配器/模組/BPF）、攻擊面對照表、GKI 配置
- [記憶體管理 Android 擴展分析](analyses/memory-android-extensions.md) — MGLRU 預設啟用、2 個 vendor hooks、memfd-ashmem 遷移、DMA-BUF heaps、lmkd 整合
- [跨子系統鎖定模式分析](analyses/locking-patterns.md) — 五種鎖定模式比較（階層 spinlock/per-CPU+RCU/rwsem/static call/原子操作）、各子系統策略表

## Sources

### C Implementation (`drivers/android/`)
- [`binder.c`](sources/src-drivers-android-binder-c.md) — Binder IPC 核心實現（7,374 行）：ioctl 入口、交易處理、引用管理、三層鎖定、凍結/解凍、debugfs
- [`binder_alloc.c`](sources/src-drivers-android-binder_alloc-c.md) — 緩衝區分配器（1,410 行）：紅黑樹管理、LRU shrinker、非同步空間限制
- [`binderfs.c`](sources/src-drivers-android-binderfs-c.md) — Binderfs 檔案系統（786 行）：動態裝置建立、IPC namespace 隔離、fs_context API
- [`vendor_hooks.c`](sources/src-drivers-android-vendor_hooks-c.md) — Vendor Hook 匯出驅動（93 行）：50 個 EXPORT_TRACEPOINT_SYMBOL_GPL、27 個 hook 標頭檔
- [`debug_kinfo.c`](sources/src-drivers-android-debug_kinfo-c.md) — Debug Kinfo 驅動（185 行）：DT 保留記憶體、kallsyms 位址匯出、XOR checksum
- [`binder_netlink.c`](sources/src-drivers-android-binder_netlink-c.md) — Binder Netlink 家族（33 行，自動生成）：Generic Netlink 交易報告

### Headers (`drivers/android/`)
- [`binder_internal.h`](sources/src-drivers-android-binder_internal-h.md) — 核心標頭檔（637 行）：所有主要資料結構（binder_proc/thread/node/ref/transaction）
- [`binder_alloc.h`](sources/src-drivers-android-binder_alloc-h.md) — 分配器標頭檔（189 行）：binder_buffer/binder_alloc/binder_shrinker_mdata
- [`dbitmap.h`](sources/src-drivers-android-dbitmap-h.md) — 動態 bitmap（169 行）：描述符 ID 快速分配、動態 grow/shrink
- [`binder_trace.h`](sources/src-drivers-android-binder_trace-h.md) — Trace Event 定義（472 行）：~20 個 tracepoint
- [`debug_kinfo.h`](sources/src-drivers-android-debug_kinfo-h.md) — Debug Kinfo 標頭（69 行）：kernel_info/kernel_all_info 結構
- [`binder_netlink.h`](sources/src-drivers-android-binder_netlink-h.md) — Netlink 標頭（22 行，自動生成）

### Rust Implementation (`drivers/android/binder/`)
- [`rust_binder_main.rs`](sources/src-drivers-android-binder-rust_binder_main-rs.md) — Rust Binder 頂層模組（611 行）：模組初始化、13 個子模組、FFI 橋接
- [`process.rs`](sources/src-drivers-android-binder-process-rs.md) — Process 類型（1,745 行）：行程資源管理、RBTree 節點/執行緒、凍結機制
- [`thread.rs`](sources/src-drivers-android-binder-thread-rs.md) — Thread 類型（1,596 行）：命令處理、scatter-gather、物件翻譯
- [`node.rs`](sources/src-drivers-android-binder-node-rs.md) — Node 類型（1,131 行）：節點排程狀態機、關鍵增量包裝器、引用計數
- [`page_range.rs`](sources/src-drivers-android-binder-page_range-rs.md) — 頁面範圍管理（731 行）：Shrinker 整合、LRU 頁面回收
- [`allocation.rs`](sources/src-drivers-android-binder-allocation-rs.md) — 分配邏輯（602 行）：RAII 分配所有權、FD 管理、oneway 序列化
- [`transaction.rs`](sources/src-drivers-android-binder-transaction-rs.md) — Transaction 結構（456 行）：Pin-Init 交易、安全上下文、時間戳記
- [`freeze.rs`](sources/src-drivers-android-binder-freeze-rs.md) — 凍結機制（398 行）：FreezeCookie/FreezeListener、通知狀態機
- [`range_alloc/`](sources/src-drivers-android-binder-range_alloc-rs.md) — 範圍分配器（1,068 行）：Tree/Array 雙實現、自動切換
- [`context.rs`](sources/src-drivers-android-binder-context-rs.md) — Context 類型（180 行）：上下文管理器、SELinux 整合、全域鎖
- [`defs.rs`](sources/src-drivers-android-binder-defs-rs.md) — 常數定義（182 行）：BC_*/BR_* 協定常數、型別安全包裝
- [`error.rs`](sources/src-drivers-android-binder-error-rs.md) — 錯誤型別（100 行）：BinderError、三種 From 轉換
- [`stats.rs`](sources/src-drivers-android-binder-stats-rs.md) — 統計追蹤（89 行）：原子計數器、外部 C 字串表

### Build & Test
- [`Kconfig`](sources/src-drivers-android-Kconfig.md) — Kconfig 選單（122 行）：8 個配置選項（Binder C/Rust、Binderfs、Vendor Hooks、Debug Kinfo、KABI、OEM Data）
- [`Makefile`](sources/src-drivers-android-Makefile.md) — 建置規則（10 行）：5 個編譯目標
- [`tests/binder_alloc_kunit.c`](sources/src-drivers-android-tests-binder_alloc_kunit-c.md) — KUnit 測試：750,000 個窮舉測試案例、5 種對齊方式 × 120 種釋放順序

---

### Coverage Dashboard

| Area | Pages | Key Files Ingested | Status |
|------|-------|--------------------|--------|
| Build System / GKI | 3 | 15+ | **Complete** |
| Android Components | 14 | 30+ | **Complete** (Entities + Vendor Hooks + Android Wiki) |
| Scheduler | 1 | 35 | **Complete** |
| Memory Management | 2 | 150+ | **Complete** |
| IPC | 1 | 14 | **Complete** |
| Filesystems | 1 | 157+ | **Complete** |
| Networking | 1 | 1500+ | **Complete** |
| Security | 1 | 163+ | **Complete** |
| Driver Framework | 0 | 0 | Not started |
| Block / I/O | 1 | 80+ | **Complete** |
| **Data Structures** | **6** | **20+** | **Complete** |
| **Cross-cutting Concepts** | **11** | **80+** | **Complete** |
| **Analyses** | **6** | — | **Complete** |
| **Source Summaries** | **25** | **40+ files** | **Complete** (drivers/android/) |
| **APIs** | **5** | **50+** | **Complete** (ioctl/netlink/sysfs-procfs/kfuncs/syscalls) |
