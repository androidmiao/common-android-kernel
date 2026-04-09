---
type: source
source_path: drivers/android/binder/range_alloc/mod.rs
lines: 329 (mod.rs) + 488 (tree.rs) + 251 (array.rs) = 1068 total
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ../concepts/rust-in-kernel.md
  - ./src-drivers-android-binder-allocation-rs.md
  - ./src-drivers-android-binder_alloc-c.md
---

# Source: `drivers/android/binder/range_alloc/`

## Summary

Rust Binder 的範圍分配器，取代 C 版本 `binder_alloc.c` 的紅黑樹緩衝區管理。提供兩種實現：基於紅黑樹的 `TreeRangeAllocator`（大範圍使用）和基於陣列的 `ArrayRangeAllocator`（小範圍使用），根據分配數量自動切換。

## Key Functions

| Item | File | Purpose |
|------|------|---------|
| `struct RangeAllocator<T>` | mod.rs | 公開 API 入口 |
| `enum DescriptorState<T>` | mod.rs:13 | 分配狀態：Reserved 或 Allocated |
| `struct Reservation` | mod.rs:42 | 保留狀態（debug_id、is_oneway、pid） |
| `struct TreeRangeAllocator` | tree.rs | 紅黑樹實現 |
| `struct ArrayRangeAllocator` | array.rs | 陣列實現 |

## Notable Implementation Details

- **雙實現切換**：小規模時使用 `ArrayRangeAllocator`（簡單高效），超過閾值後遷移到 `TreeRangeAllocator`（O(log n) 操作）。
- **兩階段分配**：先 `Reserve`（佔位），再 `Allocate`（填充資料）。支援 oneway 的序列化語意。
- **PAGE_SIZE 感知**：分配粒度與頁面大小對齊。
- **Debug 支援**：每筆分配帶有 `debug_id` 和 `pid`，供 debugfs 顯示。
- **Oneway 追蹤**：`is_oneway` 標誌支援非同步空間記帳。
- **FromArrayAllocs**：樹實現可從陣列實現遷移。
- **ReserveNew/ReserveNewArgs**：預分配紅黑樹節點的類型，避免分配失敗。

## Open Questions

- 切換閾值是多少？效能交叉點如何決定？
