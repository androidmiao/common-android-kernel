---
type: source
source_path: drivers/android/binder/transaction.rs
lines: 456
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-thread-rs.md
  - ./src-drivers-android-binder-process-rs.md
---

# Source: `drivers/android/binder/transaction.rs`

## Summary

Rust Binder 驅動的 `Transaction` 結構，對應 C 版本的 `struct binder_transaction`。封裝一筆完整的 IPC 交易，包含來源/目標資訊、緩衝區分配、安全上下文和時間戳記。

## Key Functions

| Item | Line Range | Purpose |
|------|-----------|---------|
| `struct Transaction` | 28-46 | 交易主結構（`#[pin_data(PinnedDrop)]`） |
| `Transaction::new()` | 53-60+ | 建立新交易 |

## Notable Implementation Details

- **Pin-Init 模式**：使用 `#[pin_data(PinnedDrop)]` 屬性，確保 `SpinLock<Option<Allocation>>` 等 pinned 欄位的正確初始化。
- **欄位結構**：
  - `debug_id: usize`：唯一偵錯 ID
  - `target_node: Option<DArc<Node>>`：目標節點
  - `from_parent: Option<DArc<Transaction>>`：父交易（巢狀交易支援）
  - `from: Arc<Thread>`：發起者執行緒
  - `to: Arc<Process>`：目標行程
  - `allocation: SpinLock<Option<Allocation>>`：受鎖保護的緩衝區分配
  - `is_outstanding: AtomicBool`：交易是否仍在處理中
  - `code/flags`：交易命令代碼和旗標
  - `data_size/offsets_size/data_address`：資料佈局
  - `sender_euid: Kuid`：發送者 UID
  - `txn_security_ctx_off: Option<usize>`：安全上下文偏移
  - `oneway_spam_detected: bool`：oneway spam 偵測
  - `start_time: Instant<Monotonic>`：交易開始時間
- **ListArc 整合**：`impl_list_arc_safe!` 巨集啟用 untracked ListArc 支援。
- **記憶體安全**：`ScopeGuard` 確保交易失敗時正確清理。

## Open Questions

- PinnedDrop 如何確保交易清理的正確順序？
