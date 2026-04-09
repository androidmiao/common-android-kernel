---
type: source
source_path: drivers/android/binder/error.rs
lines: 100
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-defs-rs.md
---

# Source: `drivers/android/binder/error.rs`

## Summary

Rust Binder 的錯誤型別定義。`BinderError` 封裝透過 `BINDER_WRITE_READ` ioctl 回傳給使用者空間的錯誤（而非 errno），支援 `BR_DEAD_REPLY`、`BR_FROZEN_REPLY`、`BR_FAILED_REPLY`、`BR_TRANSACTION_PENDING_FROZEN` 四種回覆。

## Key Functions

| Function | Line | Purpose |
|----------|------|---------|
| `BinderError::new_dead()` | 20 | 建立 BR_DEAD_REPLY 錯誤 |
| `BinderError::new_frozen()` | 27 | 建立 BR_FROZEN_REPLY 錯誤 |
| `BinderError::new_frozen_oneway()` | 33 | 建立 BR_TRANSACTION_PENDING_FROZEN 錯誤 |
| `BinderError::is_dead()` | 41 | 檢查是否為死亡回覆 |
| `BinderError::as_errno()` | 45 | 轉換為 errno |
| `BinderError::should_pr_warn()` | 49 | 是否應列印警告 |
| `impl From<Error>` | 56-63 | 從核心 Error 轉換（→ BR_FAILED_REPLY） |
| `impl From<BadFdError>` | 65-68 | 從 FD 錯誤轉換 |
| `impl From<AllocError>` | 71-78 | 從記憶體分配錯誤轉換（→ ENOMEM） |
| `impl Debug` | 80-100 | 偵錯格式化 |

## Notable Implementation Details

- **雙層錯誤**：`reply: u32`（Binder 協定層錯誤）+ `source: Option<Error>`（核心 errno 層錯誤）。
- **型別別名**：`BinderResult<T> = core::result::Result<T, BinderError>`。
- **三種 From 實現**：`Error`、`BadFdError`、`AllocError` 均可自動轉換為 `BinderError`，支援 `?` 運算子。
