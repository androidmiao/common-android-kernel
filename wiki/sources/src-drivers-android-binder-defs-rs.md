---
type: source
source_path: drivers/android/binder/defs.rs
lines: 182
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-thread-rs.md
---

# Source: `drivers/android/binder/defs.rs`

## Summary

Rust Binder 的常數定義模組，使用 `pub_no_prefix!` 巨集將 C UAPI 常數（`binder_driver_return_protocol_*` 和 `binder_driver_command_protocol_*`）重新匯出為無前綴的 Rust 常數。同時定義交易資料的 Rust 安全包裝型別。

## Key Functions

| Item | Line Range | Purpose |
|------|-----------|---------|
| `pub_no_prefix!` 巨集 | 12-16 | 批量移除 C 常數前綴並匯出 |
| BR_* 常數 | 18-41 | 21 個 Binder Return 協定常數 |
| BC_* 常數 | 43-65+ | Binder Command 協定常數 |

## Notable Implementation Details

- **巨集技巧**：`pub_no_prefix!` 使用 `kernel::macros::concat_idents!` 將 `binder_driver_return_protocol_BR_TRANSACTION` 映射為 `BR_TRANSACTION`。
- **FromBytes/AsBytes**：交易資料結構實現 `kernel::transmute::{AsBytes, FromBytes}`，確保安全的記憶體表示轉換。
- **Deref/DerefMut**：包裝型別透過 `Deref` trait 提供透明存取。
- **MaybeUninit**：用於未初始化記憶體的安全處理。
- **完整協定覆蓋**：BR_TRANSACTION、BR_REPLY、BR_DEAD_REPLY、BR_FROZEN_REPLY、BR_NOOP、BR_SPAWN_LOOPER、BR_TRANSACTION_COMPLETE、BR_TRANSACTION_PENDING_FROZEN、BR_ONEWAY_SPAM_SUSPECT、BR_OK、BR_ERROR、BR_INCREFS、BR_ACQUIRE、BR_RELEASE、BR_DECREFS、BR_DEAD_BINDER、BR_CLEAR_DEATH_NOTIFICATION_DONE、BR_FROZEN_BINDER、BR_CLEAR_FREEZE_NOTIFICATION_DONE。

## Open Questions

- 是否所有 C UAPI 常數都已映射？
