---
type: android
kernel_path: "drivers/android/debug_kinfo.c"
upstream: no
config_option: CONFIG_ANDROID_DEBUG_KINFO
related:
  - "../entities/debug-kinfo.md"
  - "drivers-android-overview.md"
last_updated: 2026-04-09
---

# Debug Kinfo 詳細分析

## 概要

`debug_kinfo` 是 Google 設計的平台驅動程式（platform driver），用於將核心關鍵除錯資訊寫入 Device Tree 定義的保留記憶體區域（reserved memory）。Bootloader 在系統 crash 後可以讀取這些資訊，進行 backtrace 的符號解析和 crash dump 分析。

## 原始碼

| 檔案 | 行數 | 用途 |
|------|------|------|
| `drivers/android/debug_kinfo.c` | 185 | 驅動實作 |
| `drivers/android/debug_kinfo.h` | 69 | 資料結構定義 |

## 資料結構

### struct kernel_info（packed）

匯出給 bootloader 的核心資訊，使用 `__packed` 屬性確保跨架構的二進位相容：

**Kallsyms 相關：**
- `enabled_all`：是否啟用 `CONFIG_KALLSYMS_ALL`
- `enabled_base_relative`：基址相對符號
- `enabled_absolute_percpu`：`CONFIG_KALLSYMS_ABSOLUTE_PERCPU`
- `enabled_cfi_clang`：`CONFIG_CFI`（Control Flow Integrity）
- `num_syms`：符號總數
- `name_len`：`KSYM_NAME_LEN`
- `symbol_len`：`KSYM_SYMBOL_LEN`
- `_relative_pa`：`kallsyms_relative_base` 實體位址
- `_offsets_pa`：`kallsyms_offsets` 實體位址
- `_names_pa`：`kallsyms_names` 實體位址
- `_token_table_pa`：token table 實體位址
- `_token_index_pa`：token index 實體位址
- `_markers_pa`：markers 實體位址
- `_seqs_of_names_pa`：排序名稱序列實體位址

**核心段佈局：**
- `_text_pa` / `_stext_pa` / `_etext_pa`：程式碼段範圍
- `_sinittext_pa` / `_einittext_pa`：初始化程式碼段
- `_end_pa`：核心映像結束位址

**其他：**
- `thread_size`：`THREAD_SIZE`
- `swapper_pg_dir_pa`：頁表目錄實體位址
- `last_uts_release[__NEW_UTS_LEN]`：核心版本字串
- `build_info[256]`：構建資訊（`ro.build.fingerprint`）
- `enabled_modules_tree_lookup`：`CONFIG_MODULES_TREE_LOOKUP`
- `mod_mem_offset`：`offsetof(struct module, mem)`
- `mod_kallsyms_offset`：`offsetof(struct module, kallsyms)`

### struct kernel_all_info（packed）

外層包裝結構：
- `magic_number`：`0xCCEEDDFF`（驗證資料有效性）
- `combined_checksum`：所有 `kernel_info` 欄位的 XOR 校驗和
- `info`：`struct kernel_info` 實體

## 運作流程

### 1. 平台探測（probe）

```
Device Tree: compatible = "google,debug-kinfo"
    ↓
debug_kinfo_probe()
    ↓
of_parse_phandle(memory-region) → 取得保留記憶體節點
    ↓
of_reserved_mem_lookup() → 取得 rmem（base + size）
    ↓
驗證：rmem->priv 已映射？base/size 有效？size ≥ sizeof(kernel_all_info)？
    ↓
填充 kernel_info 所有欄位
    ↓
update_kernel_all_info() → 計算 XOR checksum + 設定 magic number
```

### 2. Build Info 設定

透過 module_param_cb 機制，build_info 可在啟動時由 userspace 寫入：
```
module_param_cb(build_info, &build_info_op, NULL, 0200)
```
- 權限 `0200`：僅允許 root 寫入
- 寫入時更新 checksum

### 3. Bootloader 讀取

系統 crash 後，bootloader：
1. 讀取保留記憶體區域
2. 驗證 magic number（`0xCCEEDDFF`）
3. 驗證 XOR checksum
4. 使用 kallsyms 資訊進行符號解析
5. 使用段佈局資訊重建 backtrace

## Device Tree 綁定

```dts
reserved-memory {
    debug_kinfo_mem: debug_kinfo_mem@addr {
        compatible = "google,debug-kinfo";
        reg = <0x0 0xaddr 0x0 0xsize>;
        no-map;
    };
};

debug-kinfo {
    compatible = "google,debug-kinfo";
    memory-region = <&debug_kinfo_mem>;
};
```

## 延遲探測處理

若 reserved memory 尚未映射（`rmem->priv == NULL`），驅動返回 `-EPROBE_DEFER`，核心會稍後重試探測。

## 依賴

- `CONFIG_ANDROID_DEBUG_KINFO`
- `CONFIG_KALLSYMS`（提供符號資訊）
- Device Tree reserved memory 支援

## 交叉參考

- [Debug Kinfo Entity](../entities/debug-kinfo.md) — 實體頁面
- [drivers/android/ 總覽](drivers-android-overview.md) — 目錄總覽
