---
type: source
source_path: drivers/android/binder/node.rs
lines: 1131
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-process-rs.md
  - ./src-drivers-android-binder-transaction-rs.md
---

# Source: `drivers/android/binder/node.rs`

## Summary

Rust Binder 驅動中的 `Node` 類型和引用管理，對應 C 版本的 `struct binder_node` + `struct binder_ref`。實現節點的排程狀態機、引用計數、死亡通知，以及「關鍵增量」的包裝器機制。

## Key Functions

| Item | Line Range | Purpose |
|------|-----------|---------|
| `struct DeliveryState` | 58-60 | 節點排程狀態追蹤 |
| `struct CouldNotDeliverCriticalIncrement` | 29 | 關鍵增量傳遞失敗錯誤 |
| `struct CritIncrWrapper` (子模組 wrapper) | — | 關鍵增量包裝器 |
| `struct Node` | — | Binder 節點主結構 |
| `struct NodeRef` | — | 節點引用 |
| `struct NodeDeath` | — | 節點死亡通知 |

## Notable Implementation Details

- **排程狀態機** (`DeliveryState`)：追蹤節點是否已排入工作列表（`has_pushed_node`），以及是否有待處理的包裝器。
- **關鍵增量（Critical Increment）機制**：
  - 零到一的引用計數增量是「關鍵」的——必須傳遞給執行增量的正確執行緒。
  - `CritIncrWrapper` 分配一個包裝器物件並排入特定執行緒的 todo 列表。
  - 兩種情況：零到一的 strong increment 和零到一的 weak increment。
  - 若直接在 Node 上呼叫 `do_work`，存在 pending wrapper 時為 no-op。
- **ListArc + AtomicTracker**：使用 `ListArc` 管理節點的列表參與，`AtomicTracker` 進行原子性的列表成員追蹤。
- **LockedBy 模式**：節點內部狀態受行程鎖保護，使用 `LockedBy` 類型在編譯時關聯。
- **引用計數語意**：strong/weak/internal/local 四種引用類型，與 C 版本完全對等。

## Open Questions

- `CritIncrWrapper` 在高並行場景下的效能影響？
- 與 C 版本的 `binder_inc_node_nilocked` 的語意等效性如何驗證？
