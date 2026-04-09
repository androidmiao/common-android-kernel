---
type: source
source_path: drivers/android/debug_kinfo.c
lines: 185
ingested: 2026-04-09
related:
  - ../entities/debug-kinfo.md
  - ./src-drivers-android-debug_kinfo-h.md
---

# Source: `drivers/android/debug_kinfo.c`

## Summary

核心除錯資訊匯出驅動，將 kallsyms 位址、段佈局、頁表位址等關鍵核心資訊寫入 Device Tree 保留記憶體區域，供 bootloader 在 crash dump 時使用（如生成核心堆疊回溯）。Google 專用的 Android 除錯基礎設施。

## Key Functions

| Function | Line Range | Purpose |
|----------|-----------|---------|
| `debug_kinfo_probe()` | 94-166 | Platform driver probe：解析 DT 保留記憶體、填充 `kernel_info` |
| `update_kernel_all_info()` | 45-58 | 計算 XOR checksum 供 bootloader 校驗 |
| `build_info_set()` | 60-85 | 模組參數回呼：設定 `build_info` 字串（`ro.build.fingerprint`） |

## Notable Implementation Details

- **Platform Driver**：使用 `module_platform_driver()` 註冊，DT compatible = `"google,debug-kinfo"`。
- **保留記憶體**：透過 `of_parse_phandle()` + `of_reserved_mem_lookup()` 取得 DT 保留記憶體，`rmem->priv` 為映射後的虛擬位址。
- **填充的核心資訊**（`struct kernel_info`）：
  - kallsyms 符號表的物理位址（`_offsets_pa`、`_names_pa`、`_token_table_pa`、`_token_index_pa`、`_markers_pa`、`_seqs_of_names_pa`、`_relative_pa`）
  - 段位址（`_text_pa`、`_stext_pa`、`_etext_pa`、`_sinittext_pa`、`_einittext_pa`、`_end_pa`）
  - 頁表（`swapper_pg_dir_pa`）
  - 配置標誌（`enabled_all`、`enabled_absolute_percpu`、`enabled_cfi_clang`、`enabled_modules_tree_lookup`）
  - 模組偏移量（`mod_mem_offset`、`mod_kallsyms_offset`）
  - 執行緒大小（`thread_size`）
  - 核心版本（`last_uts_release`）
- **XOR Checksum**：`combined_checksum` 對整個 `kernel_info` 結構進行逐 u32 XOR，供 bootloader 驗證完整性。
- **Build Info**：透過 `module_param_cb(build_info, ...)` 允許使用者空間寫入建置資訊。
- **`EPROBE_DEFER`**：支援延遲 probe，等待保留記憶體映射完成。
- **Magic Number**：`DEBUG_KINFO_MAGIC = 0xCCEEDDFF`。

## Open Questions

- 除了 Google Pixel 裝置，哪些其他 Android 裝置使用此機制？
- 與 `pstore`/`ramoops` 的關係？是否互補？
