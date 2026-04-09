---
type: source
source_path: drivers/android/debug_kinfo.h
lines: 69
ingested: 2026-04-09
related:
  - ../entities/debug-kinfo.md
  - ./src-drivers-android-debug_kinfo-c.md
---

# Source: `drivers/android/debug_kinfo.h`

## Summary

Debug Kinfo 的標頭檔，定義 `struct kernel_info` 和 `struct kernel_all_info` 資料結構，以及 magic number 常數。這些結構被寫入 DT 保留記憶體供 bootloader 讀取。

## Key Functions

| Structure | Line | Purpose |
|-----------|------|---------|
| `struct kernel_info` | 20-61 | 核心除錯資訊（`__packed`） |
| `struct kernel_all_info` | 63-67 | 包含 magic + checksum + kernel_info（`__packed`） |

## Notable Implementation Details

- **`__packed` 屬性**：兩個結構都是 byte-packed，因為要提供給 bootloader 使用，不能有填充。
- **`struct kernel_info` 欄位**：
  - Kallsyms：`enabled_all/base_relative/absolute_percpu/cfi_clang`（配置標誌）、`num_syms`、`name_len`、`bit_per_long`、`symbol_len`、7 個 `_*_pa` 物理位址
  - 段位址：`_text_pa`、`_stext_pa`、`_etext_pa`、`_sinittext_pa`、`_einittext_pa`、`_end_pa`
  - 頁表：`swapper_pg_dir_pa`
  - 版本：`last_uts_release[__NEW_UTS_LEN]`
  - Build info：`build_info[256]`
  - 模組：`enabled_modules_tree_lookup`、`mod_mem_offset`、`mod_kallsyms_offset`
  - Frame pointer：`thread_size`
- **Magic Number**：`DEBUG_KINFO_MAGIC = 0xCCEEDDFF`
- **BUILD_INFO_LEN**：256 bytes，儲存 `ro.build.fingerprint` 等資訊。
