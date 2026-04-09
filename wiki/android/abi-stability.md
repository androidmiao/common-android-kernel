---
type: android
scope: kernel-wide
related:
  - "../concepts/gki.md"
  - "../concepts/module-system.md"
  - "../concepts/vendor-hooks.md"
  - "drivers-android-overview.md"
last_updated: 2026-04-09
---

# ABI 穩定性機制

## 概要

Android GKI（Generic Kernel Image）架構的核心承諾是 **KMI（Kernel Module Interface）穩定性**：在一個 GKI 核心版本的生命週期內，核心模組介面保持二進位相容。這意味著廠商模組（vendor modules）無需隨核心小版本更新而重新編譯。ABI 穩定性由三層機制共同保障：結構填充保留、符號版本化、以及運行時保護。

## 機制一：KABI 保留填充（android_kabi.h）

**檔案位置：** `include/linux/android_kabi.h`

### 凍結前使用

```c
// 在結構末端保留 u64 填充
ANDROID_KABI_RESERVE(1)  // → u64 __kabi_reserved1
ANDROID_KABI_RESERVE(2)  // → u64 __kabi_reserved2
ANDROID_BACKPORT_RESERVE(1)  // → u64 __kabi_reserved_backport1（計劃性 backport 用）
```

### 凍結後使用

```c
// 使用保留欄位替換為新欄位（union + _Static_assert 確保大小不超）
ANDROID_KABI_USE(1, bool new_flag)
ANDROID_KABI_USE2(2, u32 field_a, u32 field_b)  // 兩個欄位共用一個 u64

// 替換現有欄位類型
ANDROID_KABI_REPLACE(old_type, old_name, new_type new_name)

// 新增但不計入版本化的欄位
ANDROID_KABI_IGNORE(n, new_field_type new_field)
```

### 進階 ABI 規則巨集

```c
// 將結構視為宣告（不展開內容）
ANDROID_KABI_DECLONLY(fqn)

// 忽略特定枚舉值
ANDROID_KABI_ENUMERATOR_IGNORE(fqn, field)

// 覆蓋枚舉值
ANDROID_KABI_ENUMERATOR_VALUE(fqn, field, value)

// 設定結構大小
ANDROID_KABI_BYTE_SIZE(fqn, value)

// 覆蓋類型字串
ANDROID_KABI_TYPE_STRING(type, str)
```

### 實作原理

`_ANDROID_KABI_REPLACE` 使用 union 和 `_Static_assert` 確保：
1. 新欄位大小 ≤ 原欄位大小
2. 新欄位對齊 ≤ 原欄位對齊

規則巨集透過 `.discard.gendwarfksyms.kabi_rules` section 嵌入元數據，供 `gendwarfksyms` 工具在 ABI 版本計算時讀取。

### 依賴配置

- `CONFIG_ANDROID_KABI_RESERVE=y`（預設開啟，64 位元架構）
- 關閉時所有巨集展開為空或直接使用新欄位

### 使用範圍

KABI 保留廣泛使用於核心核心結構，包含但不限於：`task_struct`、`mm_struct`、`inode`、`sk_buff`、`page`/`folio`、`binder_proc` 等。在 `include/linux/` 目錄下有超過 10 個標頭檔使用此機制。

## 機制二：Vendor/OEM 資料填充（android_vendor.h）

**檔案位置：** `include/linux/android_vendor.h`

```c
ANDROID_VENDOR_DATA(n)           // → u64 android_vendor_data##n
ANDROID_VENDOR_DATA_ARRAY(n, s)  // → u64 android_vendor_data##n[s]
ANDROID_OEM_DATA(n)              // → u64 android_oem_data##n
ANDROID_OEM_DATA_ARRAY(n, s)     // → u64 android_oem_data##n[s]
```

**用途區分：**
- `VENDOR_DATA`：供 SoC 廠商（Qualcomm、MediaTek、Samsung 等）的核心模組使用
- `OEM_DATA`：供 OEM（設備製造商）使用

**典型範例：**
- `task_struct` 中包含 `ANDROID_VENDOR_DATA_ARRAY(1, 12)` = 96 bytes 的廠商資料空間
- 廠商模組透過 vendor hooks 在這些欄位中存儲私有資料

**依賴配置：** `CONFIG_ANDROID_VENDOR_OEM_DATA=y`（預設開啟）

## 機制三：GKI 模組保護（gki_module.c）

**檔案位置：** `kernel/module/gki_module.c`

GKI 模組保護機制確保只有經過核准的符號才能被模組使用：
- 維護一個受保護符號列表
- 使用 `bsearch` 在模組載入時驗證符號存取權限
- 拒絕未經核准的符號存取

→ 詳見 [GKI](../concepts/gki.md)、[Module 系統](../concepts/module-system.md)

## 機制四：ABI 監控工具鏈

Android 構建系統包含 ABI 監控工具：
- `gendwarfksyms`：從 DWARF 除錯資訊提取類型版本
- ABI 參考檔案：記錄基線 ABI
- CI 持續比對：每次核心變更都與基線比較

## KABI vs Vendor Data 的區別

| 面向 | KABI Reserve | Vendor Data |
|------|-------------|-------------|
| 目的 | LTS/backport 時新增核心欄位 | 廠商模組存儲私有資料 |
| 使用者 | Google 核心團隊 | SoC 廠商 / OEM |
| 凍結後行為 | 透過 ANDROID_KABI_USE 替換 | 廠商直接 cast 使用 |
| 版本化 | 計入 ABI 版本 | 不影響 ABI 版本 |

## 交叉參考

- [GKI](../concepts/gki.md) — Generic Kernel Image 整體架構
- [Module 系統](../concepts/module-system.md) — 模組載入與符號匯出
- [Vendor Hooks](../concepts/vendor-hooks.md) — Vendor Hook 框架
- [drivers/android/ 總覽](drivers-android-overview.md)
