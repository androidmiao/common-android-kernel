---
type: entity
kernel_path: drivers/android/binderfs.c
config_option: CONFIG_ANDROID_BINDERFS
upstream: no
related:
  - ../entities/binder.md
  - ../subsystems/filesystems.md
last_updated: 2026-04-09
---

# Binderfs — Binder 裝置動態管理檔案系統

## Overview

Binderfs 是一個虛擬檔案系統，用於動態管理 Binder IPC 裝置節點，取代傳統的靜態 `/dev/binder` 裝置建立方式。它允許每個 IPC namespace 獨立管理自己的 Binder 裝置，支援容器化場景。Binderfs 不存在於上游 Linux `[android]`。

## Source Layout

| 檔案 | 行數 | 用途 |
|------|------|------|
| `drivers/android/binderfs.c` | 786 | 完整 Binderfs 實現 |

## Implementation Details

### 檔案系統結構

掛載 Binderfs 後產生的目錄結構：

```
/dev/binderfs/
├── binder-control          # SECOND_INODE (ioctl 建立新裝置)
├── binder                  # 動態建立的裝置節點
├── hwbinder
├── vndbinder
├── features/               # 唯讀 Binder 功能旗標
│   ├── oneway_spam_detection
│   ├── extended_error
│   ├── freeze_notification
│   └── transaction_report
└── binder_logs/            # debugfs 風格日誌
    ├── state
    ├── stats
    ├── transactions
    ├── transaction_log
    └── failed_transaction_log
```

### 核心資料結構

- **`struct binder_features`**（binderfs.c:58-63）— 追蹤啟用的 Binder 功能旗標（4 個布林值）
- **`struct binderfs_info`**（透過 `BINDERFS_SB()` 巨集存取）— 每次掛載的資訊：裝置計數、掛載選項、IPC namespace、uid/gid、control dentry、proc log 目錄

### 關鍵函式

| 函式 | 行號 | 用途 |
|------|------|------|
| `binderfs_binder_device_create()` | :114-210 | 透過 ioctl 建立新 Binder 裝置 inode |
| `binder_ctl_ioctl()` | :225-248 | 處理 `BINDER_CTL_ADD` ioctl |
| `binderfs_evict_inode()` | :250-270 | Inode 清理 |
| `binderfs_binder_ctl_create()` | :385-448 | 建立 binder-control 裝置節點 |
| `binderfs_fill_super()` | :611-695 | 初始化 superblock |
| `init_binder_features()` | :539-572 | 建立 /features 目錄 |
| `init_binder_logs()` | :574-609 | 建立 /binder_logs 目錄 |
| `init_binderfs()` | :757-786 | 模組初始化、chrdev 區域分配 |

### 掛載機制

- 使用 Linux `fs_context` API（binderfs.c:272-314）進行掛載參數解析
- 掛載參數：`max`（u32，最大裝置數）、`stats`（enum: "global"）
- 掛載建立 superblock → root inode（`FIRST_INODE=1`）→ binder-control（`SECOND_INODE=2`）
- 裝置建立時從全域 `binderfs_minors` IDA 保留 minor number
- `BINDERFS_MAX_MINOR_CAPPED`（binderfs.c:42, 136）為 init namespace 保留 slot

### 裝置建立流程

1. 使用者空間對 `binder-control` 發出 `ioctl(fd, BINDER_CTL_ADD, &device_info)`
2. `binder_ctl_ioctl()` → `binderfs_binder_device_create()`
3. 分配 `binder_device` 結構，設定 `miscdevice` 與 `binder_context`
4. 建立 inode 與 dentry，透過 `binder_add_device()` 註冊至 Binder 核心

## Userspace Interface

- **掛載**：`mount -t binder binder /dev/binderfs [-o max=N,stats=global]`
- **裝置管理**：`ioctl(binder-control-fd, BINDER_CTL_ADD, &binderfs_device)`
- **功能查詢**：讀取 `/dev/binderfs/features/*` 檔案

## Android-Specific Notes

Binderfs 完全是 Android 專屬 `[android]`。它解決了以下問題：

- **容器隔離**：每個 IPC namespace 擁有獨立的 Binder 裝置集合
- **動態裝置管理**：無需提前在 init 腳本中建立靜態裝置節點
- **功能探測**：使用者空間可透過 `/features/` 目錄查詢核心 Binder 支援的功能

## Cross-References

- [Binder](binder.md) — Binder IPC 核心驅動
- [Filesystems 子系統](../subsystems/filesystems.md) — VFS 框架
