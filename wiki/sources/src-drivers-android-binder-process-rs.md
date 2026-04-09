---
type: source
source_path: drivers/android/binder/process.rs
lines: 1745
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../data-structures/binder_proc.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-rust_binder_main-rs.md
  - ./src-drivers-android-binder-thread-rs.md
  - ./src-drivers-android-binder-freeze-rs.md
---

# Source: `drivers/android/binder/process.rs`

## Summary

Rust Binder 驅動中的 `Process` 類型，對應 C 版本的 `struct binder_proc`。管理行程擁有的所有 binder 資源：節點、引用、執行緒、緩衝區分配。每個 binder fd 對應一個 `Process` 物件。

## Key Functions

| Item | Line Range | Purpose |
|------|-----------|---------|
| `struct Mapping` | 58-70 | 映射狀態：address + RangeAllocator |
| `enum IsFrozen` | 77-80 | 凍結狀態：Yes/No/InProgress |
| `PROC_DEFER_FLUSH/RELEASE` | 73-74 | 延遲工作標誌位元 |
| `struct Process` | — | 行程主結構（`Arc<Process>` 共享所有權） |

## Notable Implementation Details

- **所有權模型**：使用 `kernel::sync::Arc` 管理 `Process` 的共享所有權，搭配 `ListArc` 支援列表操作。
- **鎖定策略**：`Mutex` + `SpinLock` + `CondVar`，使用 `LockedBy` 模式。
- **RBTree 管理**：執行緒和節點使用 `kernel::rbtree::RBTree`。
- **ID 池**：`kernel::id_pool::IdPool` 管理描述符 ID 分配。
- **凍結機制**：包含子模組 `freeze.rs`，`FreezeCookie` + `FreezeListener` 管理凍結通知。
- **Credential 追蹤**：`kernel::cred::Credential` 記錄開啟 binder fd 時的身份。
- **Work queue 整合**：`kernel::workqueue::Work` 用於延遲處理。
- **安全性**：所有可變狀態都受到適當的鎖保護，Rust 的型別系統在編譯時強制執行。

## Open Questions

- `ProcessInner` 結構的完整欄位佈局？
- 與 C 版本 `binder_proc` 的欄位對應關係？
