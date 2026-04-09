---
type: source
source_path: drivers/android/binder_internal.h
lines: 637
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../data-structures/binder_proc.md
  - ./src-drivers-android-binder-c.md
  - ./src-drivers-android-binder_alloc-h.md
  - ./src-drivers-android-dbitmap-h.md
---

# Source: `drivers/android/binder_internal.h`

## Summary

Binder 驅動的核心內部標頭檔，定義了所有主要資料結構、工作類型、統計枚舉，以及 binderfs 相關的輔助結構。是理解 Binder 驅動架構的關鍵入口。

## Key Functions

本檔案主要定義資料結構，函式宣告較少：

| Declaration | Line | Purpose |
|-------------|------|---------|
| `binder_fops` | 75 | Binder file_operations 匯出 |
| `is_binderfs_device()` | 80/85 | 判斷 inode 是否為 binderfs 裝置 |
| `binderfs_create_file()` | 81-83 | 在 binderfs 中建立檔案 |
| `init_binderfs()` | 99/101 | Binderfs 初始化 |
| `binder_add_device()` | 625 | 新增 binder 裝置到全域列表 |
| `binder_remove_device()` | 631 | 從全域列表移除 binder 裝置 |

## Notable Implementation Details

- **核心資料結構**（全部定義於此檔案）：
  - `struct binder_context` (:18)：上下文管理器資訊（manager node、UID、名稱）
  - `struct binder_device` (:33)：裝置節點資訊（miscdev、context、binderfs_inode、refcount）
  - `struct binderfs_mount_opts` (:46)：掛載選項（max、stats_mode）
  - `struct binderfs_info` (:65)：binderfs 掛載資訊（ipc_ns、control_dentry、device_count）
  - `struct binder_work` (:147)：工作佇列項目，11 種工作類型
  - `struct binder_node` (:233)：Binder 節點，包含引用計數、排程策略、非同步 todo
  - `struct binder_ref` (:329)：節點引用，雙紅黑樹查詢（按 desc 和 node）
  - `struct binder_ref_death` (:273)：死亡通知
  - `struct binder_ref_freeze` (:283)：凍結通知（`is_frozen`/`sent`/`resend` 位元欄位）
  - `struct binder_priority` (:355)：排程優先權（policy + prio）
  - `struct binder_proc` (:444)：行程描述符（詳見 [binder_proc](../data-structures/binder_proc.md)）
  - `struct binder_thread` (:526)：執行緒描述符
  - `struct binder_transaction` (:568)：交易描述符
  - `struct binder_txn_fd_fixup` (:561)：FD 修正項目
  - `struct binder_object` (:611)：扁平化物件聯合體
- **工作類型枚舉** (`binder_work_type`)：TRANSACTION、TRANSACTION_COMPLETE、TRANSACTION_PENDING、ONEWAY_SPAM_SUSPECT、RETURN_ERROR、NODE、DEAD_BINDER、DEAD_BINDER_AND_CLEAR、CLEAR_DEATH_NOTIFICATION、FROZEN_BINDER、CLEAR_FREEZE_NOTIFICATION
- **統計類型枚舉** (`binder_stat_types`)：PROC、THREAD、NODE、REF、DEATH、TRANSACTION、TRANSACTION_COMPLETE、FREEZE
- **優先權狀態機** (`binder_prio_state`)：SET → PENDING → ABORT
- **debugfs 巨集**：`binder_for_each_debugfs_entry()` 迭代 debugfs 項目

## Open Questions

- `binder_ref_freeze` 結構的 `resend` 欄位在什麼情況下使用？
