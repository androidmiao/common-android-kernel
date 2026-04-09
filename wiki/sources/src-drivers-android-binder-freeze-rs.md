---
type: source
source_path: drivers/android/binder/freeze.rs
lines: 398
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-process-rs.md
---

# Source: `drivers/android/binder/freeze.rs`

## Summary

Rust Binder 的凍結機制實現。管理行程凍結/解凍狀態通知，實現 `FreezeCookie` 和 `FreezeListener` 類型。對應 C 版本中 `binder_ioctl_freeze()` 和 `binder_add_freeze_work()` 的功能。

## Key Functions

| Item | Line | Purpose |
|------|------|---------|
| `struct FreezeCookie` | 22 | 凍結通知的 cookie（u64 包裝） |
| `struct FreezeListener` | 25-43 | 凍結狀態監聽器 |
| `FreezeListener::is_ok_duplicate()` | 47+ | 判斷重複監聽器是否可接受 |

## Notable Implementation Details

- **FreezeListener 狀態**：
  - `node: DArc<Node>`：監聽的目標節點
  - `cookie: FreezeCookie`：此監聽器的 cookie
  - `last_is_frozen: Option<bool>`：最近告知使用者空間的凍結狀態
  - `is_pending: bool`：等待 `BC_FREEZE_NOTIFICATION_DONE` 確認
  - `is_clearing: bool`：收到 `BC_CLEAR_FREEZE_NOTIFICATION`，待回覆
  - `num_pending_duplicates/num_cleared_duplicates`：處理重複監聽器的計數器
- **重複處理**：使用者空間可能刪除後立即重建相同 cookie 的監聽器，導致重複。透過計數器追蹤避免歧義。
- **協定對映**：`BR_FROZEN_BINDER`（通知凍結）→ `BC_FREEZE_NOTIFICATION_DONE`（確認）→ 下一筆通知。
- **Ord 實現**：`FreezeCookie` 實現 `Ord` trait 以支援排序和比較。

## Open Questions

- 凍結通知在行程快速凍結/解凍時的競態條件處理？
