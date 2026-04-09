---
type: source
source_path: drivers/android/binder.c
lines: 7374
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../data-structures/binder_proc.md
  - ../subsystems/ipc.md
  - ../concepts/vendor-hooks.md
  - ./src-drivers-android-binder_internal-h.md
  - ./src-drivers-android-binder_alloc-c.md
  - ./src-drivers-android-binderfs-c.md
---

# Source: `drivers/android/binder.c`

## Summary

Android Binder IPC 驅動的 C 語言核心實現，是 Android 系統最關鍵的驅動程式之一。該檔案實現了完整的 Binder IPC 機制，包括行程間交易（transaction）、引用管理、執行緒池、凍結/解凍、優先權繼承、以及 debugfs/binderfs 整合。檔案歷史可追溯至 2007 年 Google 的原始實現。

## Key Functions

| Function | Line Range | Purpose |
|----------|-----------|---------|
| `binder_open()` | 6245 | 開啟 binder 裝置，建立 `binder_proc` 結構 |
| `binder_mmap()` | 6219 | 建立 binder 共享記憶體映射 |
| `binder_ioctl()` | 5965 | 主要 ioctl 入口，分派至各子命令處理 |
| `binder_ioctl_write_read()` | 5646 | BINDER_WRITE_READ ioctl 處理 |
| `binder_thread_write()` | 4308 | 處理使用者空間寫入命令（BC_* 協定） |
| `binder_thread_read()` | 4922 | 處理使用者空間讀取結果（BR_* 協定） |
| `binder_transaction()` | 3230 | 核心交易處理邏輯（1000+ 行巨型函式） |
| `binder_proc_transaction()` | 3016 | 將交易排入目標行程/執行緒 |
| `binder_get_ref_for_node_olocked()` | 1290 | 建立/查詢節點的引用 |
| `binder_inc_node_nilocked()` | 993 | 增加節點引用計數 |
| `binder_dec_node_nilocked()` | 1045 | 減少節點引用計數 |
| `binder_new_node()` | 967 | 建立新 binder 節點 |
| `binder_thread_release()` | 5531 | 釋放執行緒資源 |
| `binder_deferred_release()` | 6461 | 延遲釋放行程資源 |
| `binder_node_release()` | 6396 | 釋放節點並通知死亡 |
| `binder_ioctl_freeze()` | 5868 | 行程凍結/解凍 ioctl |
| `binder_ioctl_set_ctx_mgr()` | 5698 | 設定 context manager |
| `binder_apply_fd_fixups()` | 4881 | 處理交易中的 FD 修正 |
| `binder_wait_for_work()` | 4836 | 執行緒等待工作項目 |
| `binder_set_priority()` | 798 | 設定交易執行緒優先權 |
| `binder_transaction_priority()` | 810 | 交易優先權繼承邏輯 |
| `binder_netlink_report()` | 3174 | Netlink 交易報告 |
| `binder_init()` | 7294 | 模組初始化 |

## Notable Implementation Details

- **三層鎖定架構**：`proc->outer_lock` → `node->lock` → `proc->inner_lock`，嚴格的嵌套順序防止死鎖。函式命名後綴（`_olocked`、`_nlocked`、`_ilocked`）標示所需鎖。
- **交易日誌**：`binder_transaction_log` 環形緩衝區（32 個 entry）記錄最近交易，供 debugfs 使用。
- **延遲工作**：使用 `binder_deferred_list` + `binder_deferred_work` workqueue 處理 flush 和 release 操作。
- **描述符 bitmap**：整合 `dbitmap` 加速最小可用描述符 ID 的分配。
- **模組參數**：`debug_mask`（偵錯位元遮罩）、`devices`（裝置名稱）、`stop_on_user_error`。
- **debugfs 入口**：state、state_hashed、stats、transactions、transactions_hashed、transaction_log、failed_transaction_log、proc。
- **凍結機制**：支援 `BINDER_FREEZE` / `BINDER_GET_FROZEN_INFO` ioctl，配合 `binder_add_freeze_work()` 通知。
- **Netlink 報告**：透過 Generic Netlink 發送交易完成/錯誤通知至使用者空間。

## Open Questions

- `binder_transaction()` 超過 1000 行，是否有重構計畫？
- Rust 實現（`binder/rust_binder_main.rs`）何時完全取代此 C 實現？
- 是否有計畫增加更多 vendor hooks？（目前 binder.c 本身無 vendor hooks）
