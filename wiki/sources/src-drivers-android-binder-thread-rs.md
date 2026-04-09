---
type: source
source_path: drivers/android/binder/thread.rs
lines: 1596
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-process-rs.md
  - ./src-drivers-android-binder-transaction-rs.md
---

# Source: `drivers/android/binder/thread.rs`

## Summary

Rust Binder 驅動中的 `Thread` 類型，對應 C 版本的 `struct binder_thread`。處理 `binder_thread_write()` 和 `binder_thread_read()` 的等效邏輯，包含散佈-聚集（scatter-gather）傳輸機制和物件翻譯。

## Key Functions

| Item | Line Range | Purpose |
|------|-----------|---------|
| `struct ScatterGatherState` | 44-52 | 追蹤 scatter-gather 傳輸狀態 |
| `struct ScatterGatherEntry` | 56-60+ | 定義額外緩衝區複製條目 |
| `struct UnusedBufferSpace` | — | 追蹤未使用的緩衝區空間 |

## Notable Implementation Details

- **Scatter-Gather 機制**：`ScatterGatherState` 管理 SG 條目列表和祖先索引（排序陣列），實現 `BINDER_TYPE_PTR` 的深度複製。
- **物件翻譯**：`translate_objects()` 處理交易資料中的 binder 物件（flat objects、fd objects、buffer objects、fd arrays）。
- **列表整合**：使用 `ListArc`、`ListLinks`、`AtomicTracker` 管理 todo 列表。
- **安全性檢查**：`kernel::security` 模組進行 SELinux Binder 安全檢查。
- **輪詢支援**：`PollCondVar`、`PollTable` 支援 `poll()` 系統呼叫。
- **命令處理**：對應 C 版本的 `binder_thread_write()` (BC_* 命令) 和 `binder_thread_read()` (BR_* 回應)。
- **`PushWorkRes`** 公開型別：指示推送工作的結果。
- **原子操作**：`AtomicU32` 用於 looper 狀態等無鎖欄位。

## Open Questions

- Scatter-gather 實現是否有效能改善（vs. C 版本的多次 `copy_to_user`）？
