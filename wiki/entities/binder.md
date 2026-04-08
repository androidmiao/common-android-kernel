---
type: entity
kernel_path: drivers/android/binder.c, drivers/android/binder_alloc.c, drivers/android/binder/
config_option: CONFIG_ANDROID_BINDER_IPC, CONFIG_ANDROID_BINDER_IPC_RUST
upstream: no
related:
  - ../entities/binderfs.md
  - ../subsystems/ipc.md
  - ../concepts/vendor-hooks.md
  - ../concepts/rust-in-kernel.md
  - ../concepts/locking-primitives.md
  - ../subsystems/security.md
last_updated: 2026-04-09
---

# Binder — Android IPC 機制

## Overview

Binder 是 Android 的核心 IPC（行程間通訊）機制，為使用者空間提供高效能的物件導向遠端呼叫（RPC）框架。它直接在核心實現，以字元裝置 `/dev/binder`、`/dev/hwbinder`、`/dev/vndbinder` 呈現給使用者空間。Binder 不存在於上游 Linux 核心，完全是 Android 專屬元件 `[android]`。

在 ACK 中，Binder 同時擁有 C 語言實現（7,374 行）與 Rust 語言重新實現，兩者透過 `CONFIG_ANDROID_BINDER_IPC` / `CONFIG_ANDROID_BINDER_IPC_RUST` 互斥選擇。

## Source Layout

| 檔案 | 行數 | 用途 |
|------|------|------|
| `drivers/android/binder.c` | 7,374 | 核心驅動：交易處理、執行緒/節點/參考管理 |
| `drivers/android/binder_internal.h` | 637 | 核心資料結構定義 |
| `drivers/android/binder_alloc.c` | 1,410 | 交易緩衝區分配器（mmap + LRU shrinker） |
| `drivers/android/binder_alloc.h` | 189 | 分配器介面 |
| `drivers/android/binder_trace.h` | 472 | Tracepoint 定義（~20 個事件） |
| `drivers/android/binder_netlink.c` | 32 | Netlink 通知（凍結/解凍狀態） |
| `drivers/android/binder_netlink.h` | 21 | Netlink 介面（auto-generated） |
| `drivers/android/dbitmap.h` | 169 | 動態 bitmap 描述符分配 |
| `drivers/android/Kconfig` | — | 配置選項定義 |
| `drivers/android/Makefile` | — | 建置規則 |
| **Rust 重新實現** | | |
| `drivers/android/binder/rust_binder_main.rs` | ~500 | 模組入口、BinderModule |
| `drivers/android/binder/context.rs` | ~150 | Context 管理 |
| `drivers/android/binder/process.rs` | ~1,700 | Process 資源追蹤 |
| `drivers/android/binder/node.rs` | ~1,100 | Node 參考管理 |
| `drivers/android/binder/transaction.rs` | ~450 | 交易處理 |
| `drivers/android/binder/thread.rs` | ~1,700 | 執行緒狀態管理 |
| `drivers/android/binder/allocation.rs` | ~600 | 緩衝區分配 |
| `drivers/android/binder/page_range.rs` | ~800 | 頁面管理 |
| `drivers/android/binder/freeze.rs` | ~400 | 行程凍結/解凍 |

**總計：** C 實現約 10,300 行；Rust 實現約 7,400 行（含輔助 C 檔案）。

## Implementation Details

### 三層鎖定架構

Binder 使用嚴格的三層鎖定階層（binder.c 行 10-41）：

1. **`proc->outer_lock`**（spinlock）— 最外層，保護參考（`binder_ref`）、描述符查詢
2. **`node->lock`**（spinlock）— 中層，保護 `binder_node` 大多數欄位
3. **`proc->inner_lock`**（spinlock）— 最內層，保護執行緒列表、節點列表、todo 列表、transaction_stack

函式名稱後綴標示持鎖狀態：`_olocked`、`_nlocked`、`_ilocked`、`_oilocked`、`_nilocked`。

### 核心資料結構

- **`struct binder_proc`**（binder_internal.h:444）— 行程簿記：rb_tree 管理 threads/nodes/refs，`struct binder_alloc` 分配器，凍結狀態（`is_frozen`/`sync_recv`/`async_recv`）
- **`struct binder_thread`**（binder_internal.h:526）— 執行緒簿記：`transaction_stack`、`todo` 工作列表、優先權狀態機（`BINDER_PRIO_SET`/`PENDING`/`ABORT`）
- **`struct binder_node`**（binder_internal.h:233）— 服務節點：`ptr`/`cookie` 使用者空間指標、strong/weak 參考計數、`async_todo` 非同步工作佇列
- **`struct binder_ref`**（binder_internal.h:329）— 參考追蹤：雙 rb_tree（`rb_node_desc` 依 handle、`rb_node_node` 依節點）、death/freeze 通知
- **`struct binder_transaction`**（binder_internal.h:568）— 交易狀態：`from` 執行緒 → `to_proc`/`to_thread`、`buffer` 資料緩衝區、`fd_fixups` 檔案描述符修正
- **`struct binder_buffer`**（binder_alloc.h:41）— 緩衝區元資料：`data_size`/`offsets_size`/`extra_buffers_size`、`oneway_spam_suspect` 垃圾郵件偵測
- **`struct binder_alloc`**（binder_alloc.h:107）— 分配器狀態：`vm_start`、`free_buffers`/`allocated_buffers` rb_tree、`pages` 陣列、LRU shrinker
- **`struct dbitmap`**（dbitmap.h:26）— 動態 bitmap：`nbits`/`map`，支援成長/收縮，用於 handle 分配

### 交易流程

1. 使用者空間呼叫 `ioctl(fd, BINDER_WRITE_READ, &bwr)` → `binder_ioctl_write_read()` @ binder.c:5646
2. `binder_thread_write()` @ binder.c:4308 解析 `BC_TRANSACTION` 命令
3. `binder_transaction()` @ binder.c:3230（~1,000 行）執行核心邏輯：
   - 查找目標節點 → 取得目標 proc/thread
   - 分配交易緩衝區（`binder_alloc_new_buf()` @ binder_alloc.c:647）
   - 複製資料、翻譯 flat_binder_object（handle ↔ node 轉換）
   - 修正檔案描述符（`fd_fixups`）
   - 設定優先權繼承（`binder_transaction_priority()` @ binder.c:810）
   - 入隊至目標 proc/thread 的 todo 列表（`binder_proc_transaction()` @ binder.c:3016）
4. 目標執行緒被喚醒，`binder_thread_read()` @ binder.c:4922 產生 `BR_TRANSACTION` 回傳
5. 目標處理完成後發送 `BC_REPLY` → 原始執行緒收到 `BR_REPLY`

### 緩衝區分配器

- 每個行程透過 `mmap()` 映射最多 1MB 虛擬位址空間（`binder_alloc_mmap_handler()` @ binder_alloc.c:894）
- 實體頁面延遲分配，按需填充
- 已釋放緩衝區透過 LRU shrinker 回收頁面（`binder_alloc_free_page()` @ binder_alloc.c:1135）
- `binder_alloc_shrinker_init()` @ binder_alloc.c:1257 註冊核心 shrinker

### 描述符分配

`struct dbitmap`（dbitmap.h）提供動態 bitmap 管理 handle 編號分配，取代舊版線性搜尋。支援 `dbitmap_acquire_next_zero_bit()` 快速分配與自動擴展/收縮，最小大小 64 bit（64 位元系統）。

## Userspace Interface

### Ioctl 命令

| 命令 | 處理函式 | 用途 |
|------|----------|------|
| `BINDER_WRITE_READ` | `binder_ioctl_write_read()` :5646 | 主要讀寫通道 |
| `BINDER_SET_CONTEXT_MGR` | `binder_ioctl_set_ctx_mgr()` :5698 | 設定 context manager（servicemanager） |
| `BINDER_SET_CONTEXT_MGR_EXT` | 同上 | 帶 flat_binder_object 的 context manager 設定 |
| `BINDER_FREEZE` | `binder_ioctl_freeze()` :5868 | 凍結/解凍行程 |
| `BINDER_GET_NODE_INFO_FOR_REF` | `binder_ioctl_get_node_info_for_ref()` :5740 | 查詢節點資訊 |
| `BINDER_GET_EXTENDED_ERROR` | `binder_ioctl_get_extended_error()` :5949 | 取得擴展錯誤資訊 |
| `BINDER_ENABLE_ONEWAY_SPAM_DETECTION` | — | 啟用單向垃圾郵件偵測 |

### 模組參數

- `debug_mask`（uint, 0644）— 除錯輸出控制，含 9 個 flag（`BINDER_DEBUG_USER_ERROR` 等）
- `devices`（charp, 0444）— 裝置名稱列表（預設 "binder,hwbinder,vndbinder"）
- `stop_on_user_error`（int, 0644）— 使用者錯誤時暫停

### Tracepoints

binder_trace.h 定義約 20 個 TRACE_EVENT：`binder_ioctl`、`binder_transaction`、`binder_transaction_received`、`binder_set_priority`、`binder_wait_for_work`、`binder_txn_latency_free`、`binder_command`、`binder_return`、`binder_update_page_range` 等。另有多個 DEFINE_EVENT 用於緩衝區和 LRU 頁面操作追蹤。

### Netlink 介面

`binder_netlink.c` 定義 Generic Netlink 家族，multicast group `BINDER_NLGRP_REPORT`，用於非同步凍結/解凍狀態通知。

## Android-Specific Notes

Binder 完全是 Android 專屬 `[android]`，不存在於上游 Linux：

- **凍結/解凍**：支援 `BINDER_FREEZE` ioctl，配合 Android 低記憶體管理
- **單向垃圾郵件偵測**：`oneway_spam_suspect` 標記防止非同步交易洪水
- **優先權繼承**：RT 排程策略從呼叫者傳播至被呼叫者
- **安全上下文傳播**：`txn_security_ctx` 在交易中傳遞 SELinux 安全上下文（配合 `subsystems/security.md` 中 4 個 SELinux Binder hooks）
- **Rust 重新實現**：`CONFIG_ANDROID_BINDER_IPC_RUST` 啟用完整 Rust 版本，與 C 版互斥
- **三裝置架構**：`binder`（framework）、`hwbinder`（HAL）、`vndbinder`（vendor）分離不同 IPC 域
- **Vendor Hooks**：Binder 本身不使用 vendor hooks，而是使用標準 Linux tracepoints

### 匯出符號

僅 `EXPORT_SYMBOL_IF_KUNIT(binder_vm_fault)` @ binder.c:6211 — 僅供 KUnit 測試使用。

## Configuration

| 選項 | 說明 |
|------|------|
| `CONFIG_ANDROID_BINDER_IPC` | 啟用 C 版 Binder（依賴 MMU, NET） |
| `CONFIG_ANDROID_BINDER_IPC_RUST` | 啟用 Rust 版 Binder（與 C 版互斥） |
| `CONFIG_ANDROID_BINDERFS` | 啟用 Binderfs 虛擬檔案系統 |
| `CONFIG_ANDROID_BINDER_DEVICES` | 預設裝置名稱（"binder,hwbinder,vndbinder"） |
| `CONFIG_ANDROID_BINDER_ALLOC_KUNIT_TEST` | 分配器 KUnit 測試 |

## Cross-References

- [Binderfs](binderfs.md) — Binder 裝置動態管理檔案系統
- [IPC 子系統](../subsystems/ipc.md) — System V IPC / POSIX mqueue（Binder 不屬於此子系統，但為 Android 主要 IPC）
- [Security 子系統](../subsystems/security.md) — SELinux 提供 4 個 Binder 安全 hooks
- [Vendor Hooks](../concepts/vendor-hooks.md) — Binder 不使用 vendor hooks，但 vendor_hooks.c 位於同一目錄
- [Rust in Kernel](../concepts/rust-in-kernel.md) — Rust Binder 實現細節
- [鎖定原語](../concepts/locking-primitives.md) — Binder 三層 spinlock 架構
