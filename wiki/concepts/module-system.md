---
type: concept
scope: kernel-wide
related:
  - ../concepts/gki.md
  - ../concepts/kconfig-and-build.md
  - ../concepts/vendor-hooks.md
last_updated: 2026-04-08
---

# 可載入核心模組系統

## 概述

Linux 核心模組系統允許在運行時動態載入和卸載核心功能。在 Android GKI 架構中，模組系統被擴展為符號保護機制——區分 Google 簽署的 GKI 模組與廠商未簽署模組，限制後者只能存取允許的符號子集。

核心實現位於 `kernel/module/` 目錄（11 個核心檔案），其中 `main.c`（3963行）包含載入、符號解析與 GKI 限制的核心邏輯。

## 機制

### 核心資料結構

**`struct module`**（`include/linux/module.h:403-594`）：

關鍵成員：
- `enum module_state state` — 模組狀態：`LIVE`、`COMING`、`GOING`、`UNFORMED`
- `struct list_head list` — 全域模組鏈表（受 `module_mutex` 保護）
- `const struct kernel_symbol *syms` / `*gpl_syms` — 匯出符號表
- `const u32 *crcs` / `*gpl_crcs` — 符號 CRC 校驗碼
- `bool sig_ok` — 簽署驗證結果（Android 特定）
- `struct module_memory mem[MOD_MEM_NUM_TYPES]` — 分段記憶體（TEXT、DATA、RODATA、RO_AFTER_INIT、INIT_*）
- `atomic_t refcnt` — 參考計數

**`struct kernel_symbol`**（`kernel/module/internal.h:35-45`）：

根據 `CONFIG_HAVE_ARCH_PREL32_RELOCATIONS`，使用相對位址或絕對位址表示：
- 帶位移：`value_offset`、`name_offset`、`namespace_offset`
- 絕對位址：`unsigned long value`、`const char *name`、`namespace`

### 符號匯出機制

**EXPORT_SYMBOL 變體**（`include/linux/export.h:89-94`）：

| 巨集 | 用途 |
|------|------|
| `EXPORT_SYMBOL(sym)` | 不限制許可證 |
| `EXPORT_SYMBOL_GPL(sym)` | 僅限 GPL 模組 |
| `EXPORT_SYMBOL_NS(sym, ns)` | 命名空間符號 |
| `EXPORT_SYMBOL_NS_GPL(sym, ns)` | GPL 命名空間符號 |
| `EXPORT_SYMBOL_FOR_MODULES(sym, mods)` | 特定模組（GKI 限制）|

實現上，符號被放置在 `.export_symbol` 段（`export.h:33-39`），每個匯出符號包含：許可證字串、命名空間、符號值參考。

**符號搜尋流程**（`main.c:388-424`）：

```
find_symbol(struct find_symbol_arg *fsa)
├─ 搜尋核心內建符號表（__ksymtab, __ksymtab_gpl）
└─ RCU 遍歷模組列表
   └─ 對每個模組搜尋其 syms 與 gpl_syms
```

### GKI 符號保護

**實現位於** `kernel/module/gki_module.c`（71行）：

- `gki_is_module_protected_export(name)` — `bsearch()` 判斷符號是否受保護
- `gki_is_module_unprotected_symbol(name)` — 判斷未簽署模組是否可存取

**載入時存取控制**（`main.c:1284-1292`）：

```c
is_vendor_module = !mod->sig_ok;
if (is_vendor_module &&
    !is_vendor_exported_symbol &&
    !gki_is_module_unprotected_symbol(name))
    fsa.sym = ERR_PTR(-EACCES);   // 拒絕存取
```

**匯出保護**（`main.c:1508-1512`）：未簽署模組不得匯出受保護符號，違反者整個模組載入失敗（`-EACCES`）。

### 模組簽署與驗證

**實現位於** `kernel/module/signing.c`（141行）：

`module_sig_check()`（第77-140行）：
1. 檢查 `MODULE_SIG_STRING` 標記
2. 呼叫 `mod_verify_sig()` 進行 PKCS#7 驗證
3. 在 `CONFIG_MODULE_SIG_PROTECT` 下由核心自身強制執行

**Android 特殊處理**（`signing.c:23-35`）：

與上游 Linux 不同，Android **不拒絕**未簽署模組的載入（因廠商模組通常未簽署），而是透過符號保護限制其存取範圍。

### 模組載入流程

**主要入口**（`main.c:3391-3549`）：

```
load_module()
├─ 解析 ELF 格式
├─ 驗證簽署（signing.c）
├─ simplify_symbols()（1543行）— 符號重定位與 GKI 檢查
│   └─ 若 PTR_ERR(ksym) == -EACCES → 記錄受保護符號違規
├─ 應用重定位
├─ 檢查匯出保護（1508-1512行）
├─ 設定模組記憶體權限
└─ do_init_module()（3050行）— 執行模組初始化函數
```

**系統呼叫**：
- `init_module()`（3609行）— 傳統 insmod
- `finit_module()`（3774行）— 檔案描述符方式

**模組狀態機**：

```
UNFORMED → COMING（載入中） → LIVE（執行中） → GOING（卸載中）
```

### 同步機制

全域 `module_mutex`（`main.c:75`）保護 `modules` 鏈表和模組載入/卸載操作。讀取側使用 RCU 無需持有互斥鎖。

## 使用模式

### GKI 模組分類

| 類型 | 簽署 | 符號存取 | 匯出能力 |
|------|------|---------|---------|
| GKI 系統模組 | 已簽署 | 所有匯出符號 | 可匯出受保護符號 |
| 廠商模組 | 未簽署 | 僅未受保護符號 + 其他廠商模組符號 | 不可匯出受保護符號 |

### MODVERSIONS 與 CRC

`CONFIG_MODVERSIONS=y` 為每個匯出符號計算 CRC 校驗碼。當結構體佈局或函數簽名改變時，CRC 變化導致模組載入失敗，防止 ABI 不相容。

## Android 相關性

模組系統是 GKI 的執行層——所有符號保護、簽署驗證、存取控制都在模組載入時由 `kernel/module/` 實現。關鍵配置（`gki_defconfig`）：

- `CONFIG_MODULES=y`（100行）
- `CONFIG_MODULE_UNLOAD=y`（101行）
- `CONFIG_MODVERSIONS=y`（102行）
- `CONFIG_MODULE_SIG=y`（105行）
- `CONFIG_MODULE_SIG_PROTECT=y`（106行）

## 交叉參考

- [GKI](gki.md) — 模組系統服務的架構
- [Kconfig 與 Build 系統](kconfig-and-build.md) — 模組的構建與清單管理
- [Vendor Hooks](vendor-hooks.md) — 廠商模組的主要擴展方式
