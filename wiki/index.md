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
*No API pages yet.*

## Android-Specific
*No Android-specific pages yet.*

## Analyses
*No analysis pages yet.*

## Sources
*No source summary pages yet.*

---

### Coverage Dashboard

| Area | Pages | Key Files Ingested | Status |
|------|-------|--------------------|--------|
| Build System / GKI | 3 | 15+ | **Complete** |
| Android Components | 8 | 30+ | **Complete** (Entities + Vendor Hooks) |
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
