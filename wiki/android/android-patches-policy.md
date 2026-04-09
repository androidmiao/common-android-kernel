---
type: android
scope: kernel-wide
related:
  - "../concepts/gki.md"
  - "abi-stability.md"
  - "drivers-android-overview.md"
last_updated: 2026-04-09
---

# Android 修補分類與政策

## 概要

Android Common Kernel（ACK）是 Linux 主線核心的下游分支（fork）。ACK 中的每一個 commit 都可以依其來源與目的分為不同類別。理解這些分類對於判斷特定程式碼是 Android 專屬還是來自上游 Linux 至關重要。

## 修補類別

### 1. 上游（Upstream）
直接來自 Linux 主線核心的程式碼，無任何修改。這些構成了 ACK 的基礎，佔整體程式碼的絕大部分。Linux 6.19-rc8 為當前的基礎版本。

### 2. ANDROID: 標籤
**格式：** `ANDROID: <描述>`

Android 專屬的修補，不存在於 Linux 主線中。這類修補包含：
- `drivers/android/` 中的所有驅動程式（Binder、Binderfs、vendor_hooks、debug_kinfo）
- Vendor Hook 框架（`include/trace/hooks/`）
- KABI 保留填充巨集（`include/linux/android_kabi.h`）
- Vendor/OEM 資料巨集（`include/linux/android_vendor.h`）
- GKI 模組保護機制（`kernel/module/gki_module.c`）
- 各種結構中的 `ANDROID_KABI_RESERVE` 和 `ANDROID_VENDOR_DATA` 欄位

### 3. BACKPORT: 標籤
**格式：** `BACKPORT: <原始 commit 訊息>`

從較新版本的 Linux 主線核心回移植到 ACK 的修補。回移植通常需要修改以適配當前核心版本的 API。

### 4. FROMGIT: 標籤
**格式：** `FROMGIT: <描述>`

從上游維護者的 git 樹中預先取得的修補，這些修補已被接受但尚未合併到 Linux 主線 release 中。

### 5. FROMLIST: 標籤
**格式：** `FROMLIST: <描述>`

從 Linux 核心郵件列表（LKML）中取得的修補，這些修補尚在審查中或已被提交但未被最終接受。這類修補的穩定性較低。

## drivers/android/ 中的分類

`drivers/android/` 目錄中的所有程式碼均屬於 **ANDROID:** 類別：

| 元件 | 分類 | 說明 |
|------|------|------|
| Binder (C) | ANDROID | 源自 OpenBinder (2005)，Android 獨有 |
| Binder (Rust) | ANDROID | 2025 年新增的 Rust 重新實作 |
| Binderfs | ANDROID | Binder 裝置動態管理 |
| vendor_hooks.c | ANDROID | Vendor Hook 框架核心 |
| debug_kinfo.c | ANDROID | Google 專用除錯輔助 |
| dbitmap.h | ANDROID | Binder 描述符最佳化（2024 年新增） |
| binder_netlink.c | ANDROID | Binder Netlink 介面 |

## Android 修補對核心結構的影響

Android 修補除了 `drivers/android/` 外，還對核心許多關鍵結構進行了侵入式修改：

### 結構填充（非侵入）
透過 `ANDROID_KABI_RESERVE` 和 `ANDROID_VENDOR_DATA`，Android 在數十個核心結構末端加入保留欄位。這些修改是加法式的，不影響現有欄位的語意。

### Vendor Hook 注入點（低侵入）
在核心程式碼路徑中插入 `trace_android_vh_*()` 或 `trace_android_rvh_*()` 呼叫。當沒有廠商模組註冊時，這些呼叫透過 static_key 機制幾乎零開銷。

### 功能性修改（高侵入）
少數情況下，Android 需要對核心行為進行實質修改，例如：
- 記憶體管理中的 `memfd`/`ashmem` 相容層
- 排程器中的 vendor hook 整合點
- Cgroup freezer 的 Binder 感知

## 上游化進程

Android 團隊持續努力將 Android 修補上游化到 Linux 主線：
- **已上游化：** 部分 Binder 改進、某些 cgroup 功能
- **進行中：** Rust Binder 驅動（已在 linux-next）
- **不太可能上游化：** Vendor Hook 框架（設計理念與上游不同）、KABI 保留機制

## 與 GKI 的關係

GKI 架構要求嚴格區分：
- **GKI 核心映像**：包含所有上游程式碼 + `ANDROID:` 修補
- **廠商模組**：僅透過穩定的 KMI 介面（匯出符號 + vendor hooks）與核心互動
- **ABI 穩定性**：KABI 保留 + vendor data 確保模組二進位相容

## 交叉參考

- [GKI](../concepts/gki.md) — Generic Kernel Image 架構
- [ABI 穩定性](abi-stability.md) — ABI 保留與穩定性機制
- [Vendor Hooks](../concepts/vendor-hooks.md) — Vendor Hook 框架
- [drivers/android/ 總覽](drivers-android-overview.md) — 目錄總覽
