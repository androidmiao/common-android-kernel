---
type: source
source_path: drivers/android/dbitmap.h
lines: 169
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ./src-drivers-android-binder-c.md
  - ./src-drivers-android-binder_internal-h.md
---

# Source: `drivers/android/dbitmap.h`

## Summary

動態大小的 bitmap 函式庫，由 Binder 驅動用於快速分配最小可用的描述符 ID。每個 bit 代表一個 ID 的使用狀態。支援動態 grow/shrink，且設計上允許在短暫釋放鎖的情況下分配新 bitmap 記憶體。

## Key Functions

| Function | Line | Purpose |
|----------|------|---------|
| `dbitmap_init()` | 157 | 初始化 bitmap（`NBITS_MIN` = `BITS_PER_LONG` bits） |
| `dbitmap_free()` | 36 | 釋放 bitmap |
| `dbitmap_enabled()` | 31 | 檢查 bitmap 是否啟用 |
| `dbitmap_acquire_next_zero_bit()` | 135-149 | 尋找並設定下一個空閒 bit |
| `dbitmap_clear_bit()` | 151-155 | 清除指定 bit |
| `dbitmap_grow()` | 103-128 | 擴展 bitmap（大小加倍） |
| `dbitmap_shrink()` | 78-95 | 縮減 bitmap（大小減半） |
| `dbitmap_grow_nbits()` | 98 | 計算 grow 目標大小 |
| `dbitmap_shrink_nbits()` | 44 | 計算 shrink 目標大小（最後 set bit 在前 1/4 時可縮） |
| `dbitmap_replace()` | 69-76 | 替換內部 bitmap 並釋放舊的 |

## Notable Implementation Details

- **最小大小**：`NBITS_MIN = BITS_PER_TYPE(unsigned long)` = 64 bits（64-bit 系統）。
- **Grow 策略**：大小加倍 (`nbits << 1`)。
- **Shrink 策略**：當最後 set bit 位於前 1/4 時可縮減至 1/2 大小。
- **無鎖分配設計**：`grow()` 和 `shrink()` 接受外部預先分配的 bitmap，在回呼時驗證是否仍有效。無效時釋放預分配記憶體。
- **Fallback**：若 grow 的記憶體分配失敗，呼叫 `dbitmap_free()` 停用 bitmap，Binder 回退到 `slow_desc_lookup_olocked()`（線性掃描）。
- **鎖定**：外部使用 `proc->outer_lock` 保護，bitmap 本身不提供並行保護。
- **全部 inline**：所有函式均為 `static inline`，定義於標頭檔。

## Open Questions

- 在高描述符使用量下，shrink 的閾值（1/4）是否過於積極？
