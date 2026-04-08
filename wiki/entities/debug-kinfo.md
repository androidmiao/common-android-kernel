---
type: entity
kernel_path: drivers/android/debug_kinfo.c
config_option: (platform driver, "google,debug-kinfo")
upstream: no
related:
  - ../entities/binder.md
  - ../concepts/gki.md
last_updated: 2026-04-09
---

# Debug Kinfo — 核心除錯資訊匯出

## Overview

Debug Kinfo 是 Android 專屬的平台驅動程式，透過 Device Tree 保留記憶體區域將核心符號表位置、記憶體佈局等關鍵除錯資訊暴露給 bootloader 和 crash dump 工具。這使得在核心崩潰後，外部工具（如 bootloader crash dump handler）能夠在不依賴核心自身的情況下解析堆疊回溯和符號。不存在於上游 Linux `[android]`。

## Source Layout

| 檔案 | 行數 | 用途 |
|------|------|------|
| `drivers/android/debug_kinfo.c` | 185 | 平台驅動實現 |
| `drivers/android/debug_kinfo.h` | 69 | 資料結構定義 |

## Implementation Details

### 核心資料結構

**`struct kernel_info`**（debug_kinfo.h:20-61）— 打包結構，包含：

- **Kallsyms 配置旗標**：`enabled_all`、`enabled_base_relative`、`enabled_absolute_percpu`、`enabled_cfi_clang`
- **符號表參數**：`num_syms`、`name_len`、`module_name_len`、`symbol_len`
- **Kallsyms 表實體位址**：`_relative_pa`、`_offsets_pa`、`_names_pa`、`_token_table_pa`、`_token_index_pa`、`_markers_pa`、`_seqs_of_names_pa`
- **核心段位址**：`_text_pa`、`_stext_pa`、`_etext_pa`、`_sinittext_pa`、`_einittext_pa`、`_end_pa`
- **堆疊資訊**：`thread_size`（用於 frame pointer 回溯）
- **頁表**：`swapper_pg_dir_pa`（虛擬→實體位址映射）
- **版本**：`last_uts_release`（核心版本字串）
- **Build info**：256 bytes 的建置資訊緩衝區
- **模組支援**：`enabled_modules_tree_lookup`、`mod_mem_offset`、`mod_kallsyms_offset`

**`struct kernel_all_info`**（debug_kinfo.h:63-67）— 包裝結構，含 magic number 和 XOR checksum。

### 關鍵函式

| 函式 | 行號 | 用途 |
|------|------|------|
| `debug_kinfo_probe()` | :94-166 | 平台裝置 probe：取得保留記憶體、映射、填充 kernel_all_info |
| `update_kernel_all_info()` | :45-58 | 計算 XOR checksum |
| `build_info_set()` | :60-85 | 模組參數回調：寫入 build info 字串 |

### 工作流程

1. Device Tree 定義 `compatible = "google,debug-kinfo"` 節點與保留記憶體 phandle
2. 核心啟動時 `debug_kinfo_probe()` 被呼叫
3. 取得保留記憶體區域（若尚未映射返回 `-EPROBE_DEFER`）
4. 映射保留記憶體，填充 `struct kernel_all_info`（kallsyms 位址、段位址等）
5. 計算 XOR checksum 確保資料完整性
6. Bootloader / crash dump 工具讀取此區域以解析崩潰資訊

### 除錯用途

- **符號解析**：透過 kallsyms 表位址將核心位址轉換為函式名稱
- **堆疊回溯**：透過 `thread_size` 和 frame pointer 資訊走訪核心堆疊
- **記憶體佈局判斷**：透過段位址確定核心程式碼/資料範圍
- **模組符號**：透過 `mod_mem_offset`/`mod_kallsyms_offset` 解析可載入模組符號

## Userspace Interface

無直接使用者空間介面。資料透過保留記憶體區域供 bootloader 和核心外工具讀取。

## Android-Specific Notes

完全是 Android/Google 專屬 `[android]`：

- 依賴 Device Tree `"google,debug-kinfo"` compatible 字串
- 主要服務於 Pixel 裝置的 crash dump 基礎設施
- 與 GKI 的 kallsyms 配置密切相關（需要 `CONFIG_KALLSYMS_ALL` 等）

## Cross-References

- [GKI 架構](../concepts/gki.md) — Kallsyms 配置與 GKI 符號管理
- [Binder](binder.md) — 同位於 `drivers/android/` 目錄
