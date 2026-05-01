---
type: source
source_path: include/linux/android_kabi.h
lines: 209
ingested: 2026-04-09
---

# Source: `android_kabi.h`

## Summary

Android KABI（核心 ABI）保留填充標頭，定義一套巨集用於在核心資料結構中預留空間並在 ABI 凍結後安全替換欄位。靈感來自 Red Hat 的 `rh_kabi.h`。透過 `CONFIG_ANDROID_KABI_RESERVE` 條件編譯控制。

## Key Functions

| 巨集 | 行範圍 | 用途 |
|------|--------|------|
| `ANDROID_KABI_RESERVE(n)` | 101 | ABI 凍結前：保留 `u64 __kabi_reserved##n` 填充 |
| `ANDROID_BACKPORT_RESERVE(n)` | 102 | 同上，但用於計畫中的功能 backport |
| `ANDROID_KABI_USE(n, _new)` | 171-172 | ABI 凍結後：用 `_new` 欄位替換保留填充 |
| `ANDROID_KABI_USE2(n, _new1, _new2)` | 181-182 | 用兩個較小欄位替換一個 64-bit 保留 |
| `ANDROID_KABI_REPLACE(_old, _oldname, _new)` | 162-163 | 替換同大小欄位（union 方式） |
| `ANDROID_KABI_IGNORE(n, _new)` | 152-156 | 添加版本計算中被忽略的新欄位 |
| `ANDROID_KABI_DECLONLY(fqn)` | 113 | 將 struct/union/enum 視為宣告（不展開定義） |
| `ANDROID_KABI_ENUMERATOR_IGNORE(fqn, field)` | 120-121 | enum 版本計算中跳過指定欄位 |
| `ANDROID_KABI_ENUMERATOR_VALUE(fqn, field, value)` | 129-130 | 覆蓋 enum 欄位的版本計算值 |
| `ANDROID_KABI_BYTE_SIZE(fqn, value)` | 137-138 | 設定結構的 byte_size 屬性 |
| `ANDROID_KABI_TYPE_STRING(type, str)` | 145-146 | 覆蓋 symtypes 中的型別字串 |
| `_ANDROID_KABI_NORMAL_SIZE_ALIGN` | 60-73 | 內部：_Static_assert 檢查大小和對齊 |
| `__ANDROID_KABI_RULE` | 50-54 | 內部：生成 `.discard.gendwarfksyms.kabi_rules` 段 |

## Notable Implementation Details

**靜態斷言保護：** `_ANDROID_KABI_REPLACE` 使用 `_Static_assert` 確保新欄位不超過原欄位的大小和對齊要求，編譯期即可捕獲錯誤。

**gendwarfksyms 整合：** `__ANDROID_KABI_RULE` 將規則寫入 `.discard.gendwarfksyms.kabi_rules` ELF 段，供 ABI 工具鏈（gendwarfksyms）讀取。格式為 `"1\0" hint "\0" target "\0" value`。

**條件編譯：** 當 `CONFIG_ANDROID_KABI_RESERVE` 未啟用時，所有巨集退化為空或直接展開新欄位，不引入任何開銷。

**VDSO 排除：** 在 `BUILD_VDSO` 或 `__DISABLE_EXPORTS` 環境下，`__ANDROID_KABI_RULE` 為空，避免在 vDSO 中生成無用段。

## Open Questions

- 目前有多少核心結構使用了 `ANDROID_KABI_RESERVE`？（粗估 60+ 結構，需全面掃描確認）
- `ANDROID_BACKPORT_RESERVE` 與 `ANDROID_KABI_RESERVE` 的使用策略差異何在？
