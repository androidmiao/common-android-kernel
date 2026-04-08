---
type: concept
scope: kernel-wide
related:
  - ../concepts/vendor-hooks.md
  - ../concepts/kconfig-and-build.md
  - ../concepts/module-system.md
  - ../subsystems/scheduler.md
last_updated: 2026-04-08
---

# Generic Kernel Image (GKI)

## 概述

GKI（Generic Kernel Image）是 Android Common Kernel 的核心架構設計，目標是將**通用核心映像**與**廠商特定模組**徹底分離。在 GKI 架構下，所有 Android 裝置共享同一個核心映像（`boot.img`），廠商差異透過可載入的核心模組（vendor modules）實現。這解決了 Android 生態系長期以來核心碎片化的問題——過去每家 OEM 各自維護自己的核心分支，導致安全更新延遲、維護成本高昂。

GKI 的設計理念：核心映像由 Google 統一構建與維護，廠商只需提供自己的模組，透過穩定的 KMI（Kernel Module Interface）與核心互動。

## 機制

### 架構分層

```
┌─────────────────────────────────────────────────┐
│           Vendor Kernel Modules                  │  ← OEM/SoC 特定模組
│  （透過穩定 KMI + Vendor Hooks 載入）             │
├─────────────────────────────────────────────────┤
│         GKI Kernel Image (boot.img)              │  ← 所有裝置共用
│  ┌─────────────────────────────────────────────┐ │
│  │  Upstream Linux + Android patches           │ │
│  │  ┌────────────────┐ ┌──────────────────┐   │ │
│  │  │ Binder IPC      │ │ Vendor Hooks     │   │ │
│  │  └────────────────┘ └──────────────────┘   │ │
│  └─────────────────────────────────────────────┘ │
├─────────────────────────────────────────────────┤
│              Hardware (SoC)                       │
└─────────────────────────────────────────────────┘
```

### KMI（Kernel Module Interface）穩定性

KMI 是 GKI 最關鍵的機制，確保廠商模組在核心更新後仍然相容。

**符號保護體系** 分為三層：

1. **簽署模組（Signed Modules）**：由 Google 簽署的 GKI 系統模組，可存取所有匯出符號。
2. **受保護符號（Protected Exports）**：只有簽署模組才能匯出的符號，定義在 `gki_module_protected_exports.h`。
3. **未受保護符號（Unprotected Symbols）**：未簽署的廠商模組可存取的符號子集，定義在 `gki_module_unprotected.h`。

**符號檢查的實現** 位於 `kernel/module/gki_module.c:37-70`：

- `gki_is_module_protected_export(name)` — 使用 `bsearch()` 在排序的符號清單中做二進位搜尋，判斷符號是否受保護
- `gki_is_module_unprotected_symbol(name)` — 判斷未簽署模組是否可存取該符號

**載入時的存取控制** 位於 `kernel/module/main.c:1284-1292`：

```c
is_vendor_module = !mod->sig_ok;               // 未簽署 = 廠商模組
is_vendor_exported_symbol = fsa.owner && !fsa.owner->sig_ok;

if (is_vendor_module &&
    !is_vendor_exported_symbol &&
    !gki_is_module_unprotected_symbol(name)) {
    fsa.sym = ERR_PTR(-EACCES);                // 拒絕存取
}
```

**匯出保護** 位於 `kernel/module/main.c:1508-1512`：未簽署模組不得匯出受保護符號，違反者整個模組載入失敗。

### ABI 穩定性工具鏈

ABI 監控由 `build/kernel/abi/` 目錄下的工具實現：

- `check_buildtime_symbol_protection.py` — 構建時驗證未簽署模組的符號依賴是否合法
- `symbol_extraction.py` — 從 ELF 二進位提取 `ksymtab` 匯出符號與未定義符號
- `kmi_defines.py` — 提取 C 語言 `#define` 編譯時常數，與 KMI 一起追蹤
- `verify_ksymtab.py` — 驗證匯出符號表的完整性

符號清單的生成由 `scripts/gen_gki_modules_headers.sh` 完成（第42-103行），將符號文件處理為排序的 C 結構體陣列，供 `gki_module.c` 的 `bsearch()` 使用。

### MODVERSIONS 與 GENDWARFKSYMS

GKI defconfig 啟用了 `CONFIG_MODVERSIONS=y` 和 `CONFIG_GENDWARFKSYMS=y`（`arch/arm64/configs/gki_defconfig:102-103`），透過符號 CRC 校驗碼追蹤 ABI 變更。當結構體佈局或函數簽名改變時，CRC 值會變化，模組載入失敗，從而防止不相容的模組被載入。

### GKI 隱藏配置（Hidden Configs）

GKI 核心需要預先啟用某些只在模組中使用的配置。這些定義在 `init/Kconfig.gki`（336行）：

- `GKI_HIDDEN_DRM_CONFIGS` — DRM/顯示相關依賴
- `GKI_HIDDEN_SND_CONFIGS` / `GKI_HIDDEN_SND_SOC_CONFIGS` — 音頻系統
- `GKI_HIDDEN_QCOM_CONFIGS` — 高通 SoC 特定
- `GKI_HIDDEN_MEDIA_CONFIGS` — 媒體控制器
- `GKI_HIDDEN_USB_CONFIGS` — USB PHY

集合點 `CONFIG_GKI_HACKS_TO_FIX`（`Kconfig.gki:299-335`）一次啟用所有隱藏配置。這是臨時措施，長期目標是讓驅動程式在 GKI 核心中可完全模組化。

### System DLKM 模組

GKI 將部分核心模組打包為 System DLKM（Dynamic Loadable Kernel Module），放在 `system_dlkm` 分區。ARM64 的模組清單定義在 `bazel/modules_private.bzl:11-91`，共 91 個通用模組，包括 virtio_blk、zram、bluetooth、CAN、NFC、PPP、USB 等。

### 構建產出

GKI 的 Bazel 構建規則（`BUILD.bazel:212-228`）生成以下產出：

- `Image` / `Image.lz4` — 核心映像（多種壓縮格式）
- `boot.img` — 使用 `mkbootimg` 生成的啟動映像
- GKI 系統 DLKM 模組
- `gki-info.txt` — 元數據
- AVB（Android Verified Boot）簽名

## 使用模式

### 廠商整合流程

1. Google 發布 GKI 核心映像與 KMI 符號清單
2. 廠商基於 KMI 開發自己的核心模組
3. 廠商模組透過 [Vendor Hooks](vendor-hooks.md) 在排程、記憶體、安全等子系統中插入自定義邏輯
4. 構建時工具鏈自動驗證廠商模組不違反 KMI 約束
5. 裝置啟動時載入 GKI 映像 + 廠商模組

### 支援的架構與變體

- **ARM64**（主要）：`arch/arm64/configs/gki_defconfig`，支援 4K / 16K / 64K 頁面大小
- **x86_64**：`arch/x86/configs/gki_defconfig`
- 頁面大小變體透過 defconfig fragment 疊加實現（`build/kernel/kleaf/impl/defconfig/`）

## Android 相關性

GKI 是 Android 核心策略的基石。它使得：

- **安全更新**可以不依賴 OEM 獨立推送核心修補
- **碎片化**大幅降低——所有裝置使用相同的核心基礎
- **廠商差異**被限制在明確的擴展點（Vendor Hooks + 模組）
- **ABI 穩定性**保障了跨版本的模組相容性

GKI defconfig 中的關鍵 Android 配置（`gki_defconfig`）：

| 配置 | 行號 | 用途 |
|------|------|------|
| `CONFIG_MODULES=y` | 100 | 啟用模組支援 |
| `CONFIG_MODULE_SIG=y` | 105 | 模組簽署 |
| `CONFIG_MODULE_SIG_PROTECT=y` | 106 | GKI 符號保護 |
| `CONFIG_MODVERSIONS=y` | 102 | 符號版本追蹤 |
| `CONFIG_GENDWARFKSYMS=y` | 103 | DWARF 符號資訊 |
| `CONFIG_GKI_HACKS_TO_FIX=y` | 113 | 啟用所有隱藏配置 |
| `CONFIG_ANDROID_BINDER_IPC=y` | 630 | Binder IPC |
| `CONFIG_ANDROID_VENDOR_HOOKS=y` | 633 | Vendor Hook 框架 |

## 交叉參考

- [Vendor Hooks](vendor-hooks.md) — GKI 的廠商擴展機制
- [Kconfig 與 Build 系統](kconfig-and-build.md) — GKI 的構建基礎設施
- [Module 系統](module-system.md) — 模組載入、簽署與符號管理
- [Scheduler](../subsystems/scheduler.md) — 包含 78 個 Vendor Hooks 的子系統範例
