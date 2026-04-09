---
type: source
source_path: drivers/android/binderfs.c
lines: 786
ingested: 2026-04-09
related:
  - ../entities/binderfs.md
  - ../entities/binder.md
  - ./src-drivers-android-binder_internal-h.md
  - ./src-drivers-android-binder-c.md
---

# Source: `drivers/android/binderfs.c`

## Summary

Binderfs 虛擬檔案系統實現，允許透過 `binder-control` 裝置動態建立/刪除 binder 裝置節點。支援 IPC namespace 隔離，可在容器化環境中運行多個 Android 實例。使用 `fs_context` 新式掛載 API。

## Key Functions

| Function | Line Range | Purpose |
|----------|-----------|---------|
| `binderfs_binder_device_create()` | — | 透過 binder-control ioctl 建立新裝置 |
| `binderfs_binder_ctl_create()` | — | 建立 binder-control 控制裝置 |
| `binderfs_fill_super()` | — | 填充 superblock |
| `binderfs_fs_context_get_tree()` | — | fs_context 掛載入口 |
| `binderfs_fs_context_parse_param()` | — | 解析掛載參數（max、stats） |
| `init_binderfs()` | — | 模組初始化，註冊檔案系統 |

## Notable Implementation Details

- **掛載參數**：`max`（最大裝置數）和 `stats`（統計模式：global）。
- **功能探測**：`binder_features` 結構匯出 4 個能力標誌：`oneway_spam_detection`、`extended_error`、`freeze_notification`、`transaction_report`，全部預設啟用。
- **IDA 管理**：`binderfs_minors` IDA 分配器管理裝置 minor number，上限 `BINDERFS_MAX_MINOR_CAPPED`。
- **IPC Namespace 隔離**：每個 binderfs 掛載與一個 `ipc_namespace` 關聯，確保容器間隔離。
- **固定 inode 編號**：`FIRST_INODE` (1) = binder-control，`SECOND_INODE` (2) = root，其餘從 `INODE_OFFSET` (3) 開始。
- **Proc 日誌目錄**：支援 `binder_logs/` 子目錄（與 C Binder 驅動的 debugfs 對等功能）和 `proc_log_dir`。

## Open Questions

- Rust 版本 `rust_binderfs.c` 如何與此 C 版本共存？
