---
type: source
source_path: drivers/android/binder/allocation.rs
lines: 602
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-process-rs.md
  - ./src-drivers-android-binder-page_range-rs.md
---

# Source: `drivers/android/binder/allocation.rs`

## Summary

Rust Binder 的緩衝區分配邏輯，對應 C 版本的 `binder_alloc.c` 的上層抽象。定義 `AllocationInfo`（分配中繼資料）和 `Allocation`（活躍分配的所有權代表）。管理交易緩衝區的分配、資料複製、FD 處理和釋放。

## Key Functions

| Item | Line Range | Purpose |
|------|-----------|---------|
| `struct AllocationInfo` | 27-42 | 分配中繼資料：偏移範圍、目標節點、oneway 節點、clear_on_free、file_list |
| `struct Allocation` | 44-50+ | 活躍分配的所有權代表（RAII 風格） |
| `struct NewAllocation` | — | 新建分配的構建器 |
| `struct TranslatedFds` | — | 已翻譯的 FD 集合 |

## Notable Implementation Details

- **所有權語意**：`Allocation` 代表 range allocator 中的一筆活躍分配。當 `Allocation` drop 時，自動歸還空間。
- **Oneway 序列化**：`oneway_node: Option<DArc<Node>>` 確保對同一節點的 oneway 交易按順序處理。
- **Clear on Free**：`clear_on_free: bool` 在釋放時歸零緩衝區資料，防止資訊洩漏。
- **檔案描述符管理**：`FileList` 追蹤嵌入在交易中的檔案描述符。
- **DeferredFdCloser 整合**：安全地延遲關閉 FD，避免 `fdget` 使用中關閉的 use-after-free。
- **AllocationView**：提供對分配資料的視圖，支援偏移量讀寫。
- **BinderObject/BinderObjectRef**：交易物件的 Rust 型別安全表示。

## Open Questions

- 與 C 版 `binder_alloc.c` 的效能比較？
