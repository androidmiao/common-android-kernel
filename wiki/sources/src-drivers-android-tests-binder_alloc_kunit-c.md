---
type: source
source_path: drivers/android/tests/binder_alloc_kunit.c
lines: ~300+
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ./src-drivers-android-binder_alloc-c.md
  - ./src-drivers-android-binder_alloc-h.md
---

# Source: `drivers/android/tests/binder_alloc_kunit.c`

## Summary

Binder 緩衝區分配器的 KUnit 單元測試。測試不同對齊方式和釋放順序下的分配/釋放行為，使用獨立的 `binder_alloc` 結構和測試專用的 freelist，不影響運行中的系統。

## Key Functions

| Item | Line | Purpose |
|------|------|---------|
| `enum buf_end_align_type` | 51 | 緩衝區對齊類型：SAME_PAGE_UNALIGNED、SAME_PAGE_ALIGNED、NEXT_PAGE_UNALIGNED、NEXT_PAGE_ALIGNED、NEXT_NEXT_UNALIGNED |
| 常數定義 | 26-41 | `BINDER_MMAP_SIZE` (128K)、`BUFFER_NUM` (5)、`BUFFER_MIN_SIZE` (PAGE_SIZE/8) |

## Notable Implementation Details

- **窮舉測試**：`TOTAL_EXHAUSTIVE_CASES = 3125 * 2 * 120` = 750,000 個測試案例（5^5 對齊組合 × 2 頁面共享位置 × 5! 釋放順序）。
- **隔離測試環境**：使用獨立的 `binder_alloc` 結構和測試專用 freelist，透過 `__binder_alloc_init()` 初始化。
- **MODULE_IMPORT_NS**：`EXPORTED_FOR_KUNIT_TESTING` 命名空間，存取 `binder_alloc_buffer_size()` 等內部函式。
- **對齊類型**：5 種緩衝區結尾相對於前一緩衝區的頁面對齊方式，覆蓋所有邊界情況。
- **匿名 inode**：使用 `anon_inodes` 建立測試用的假檔案描述符。

## Open Questions

- 測試覆蓋率如何量化？是否涵蓋 shrinker 回收路徑？
