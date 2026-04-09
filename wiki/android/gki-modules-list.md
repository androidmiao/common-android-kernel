---
type: android
scope: kernel-wide
related:
  - "../concepts/gki.md"
  - "../concepts/module-system.md"
  - "abi-stability.md"
  - "drivers-android-overview.md"
last_updated: 2026-04-09
---

# GKI 模組清單與管理

## 概要

GKI（Generic Kernel Image）架構將核心分為兩部分：不可修改的 GKI 核心映像，以及可替換的核心模組。模組透過 KMI（Kernel Module Interface）與核心互動，確保二進位相容。本頁說明 GKI 模組的組織方式與管理機制。

## 模組分類

### 1. GKI 模組（內建或 System DLKM）
由 Google 維護的核心模組，包含在 GKI 發行版中。這些模組隨 GKI 核心映像一起分發，安裝在 `system_dlkm` 分區。

### 2. 廠商模組（Vendor DLKM）
由 SoC 廠商（Qualcomm、MediaTek、Samsung LSI 等）提供的模組。安裝在 `vendor_dlkm` 分區。只能使用已匯出的 KMI 符號和 vendor hooks。

### 3. OEM 模組
由設備製造商提供的模組。使用範圍更受限。

## drivers/android/ 中的模組

`drivers/android/` 目錄的元件全部編譯為**內建**（built-in，obj-y），不作為可載入模組：

| 元件 | Makefile 規則 | 類型 |
|------|--------------|------|
| Binder (C) | `obj-$(CONFIG_ANDROID_BINDER_IPC) += binder.o binder_alloc.o binder_netlink.o` | 內建 |
| Binder (Rust) | `obj-$(CONFIG_ANDROID_BINDER_IPC_RUST) += binder/` | 內建 |
| Binderfs | `obj-$(CONFIG_ANDROID_BINDERFS) += binderfs.o` | 內建 |
| Vendor Hooks | `obj-$(CONFIG_ANDROID_VENDOR_HOOKS) += vendor_hooks.o` | 內建 |
| Debug Kinfo | `obj-$(CONFIG_ANDROID_DEBUG_KINFO) += debug_kinfo.o` | 內建 |
| KUnit Tests | `obj-$(CONFIG_ANDROID_BINDER_ALLOC_KUNIT_TEST) += tests/` | tristate（可模組） |

注意：唯一可作為可載入模組的是 KUnit 測試（`CONFIG_ANDROID_BINDER_ALLOC_KUNIT_TEST` 為 tristate）。

## KMI 符號保護

### gki_module.c

`kernel/module/gki_module.c` 實作 GKI 模組保護：
- 維護受保護的 KMI 符號列表
- 模組載入時以 `bsearch` 驗證每個匯出符號的存取權限
- 未經授權的符號存取會導致模組載入失敗

### 符號匯出方式

`drivers/android/vendor_hooks.c` 匯出的符號使用：
```c
EXPORT_TRACEPOINT_SYMBOL_GPL(android_vh_*)
EXPORT_TRACEPOINT_SYMBOL_GPL(android_rvh_*)
```

這些符號是 GPL-only 的，廠商模組必須宣告 GPL 相容授權才能使用。

## 模組載入順序

Android 開機時的典型模組載入順序：
1. 核心內建驅動初始化（包含 Binder、vendor_hooks）
2. System DLKM 模組載入
3. Vendor DLKM 模組載入（此時可以註冊 vendor hooks）
4. OEM 模組載入

## 構建系統整合

GKI 使用 Kleaf/Bazel 構建系統管理模組：
- `kernel_build` 規則：構建核心映像和內建模組
- `gki_artifacts` 規則：產出 GKI 發行產物
- 模組清單檔案定義哪些模組包含在 GKI 中

→ 詳見 [Kconfig 與 Build 系統](../concepts/kconfig-and-build.md)

## 交叉參考

- [GKI](../concepts/gki.md) — GKI 整體架構
- [Module 系統](../concepts/module-system.md) — 模組載入機制
- [ABI 穩定性](abi-stability.md) — KABI 保留與穩定性
- [Kconfig 與 Build](../concepts/kconfig-and-build.md) — 構建系統
- [drivers/android/ 總覽](drivers-android-overview.md)
