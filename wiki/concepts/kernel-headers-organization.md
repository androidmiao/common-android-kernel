---
type: concept
scope: kernel-wide
related:
  - ../subsystems/include-headers.md
  - ../concepts/gki.md
  - ../android/abi-stability.md
  - ../concepts/vendor-hooks.md
  - ../concepts/kconfig-and-build.md
last_updated: 2026-04-09
---

# Kernel Headers 組織架構

## Overview

Linux 核心的 `include/` 目錄是所有子系統的**公共介面定義中心**，包含 6,593 個標頭檔。這些標頭遵循嚴格的分層組織，將使用者空間介面 (UAPI) 與核心內部 API 分離，並透過 asm-generic 回退機制支援多架構。在 ACK 中，此分層額外包含 Android KABI 保留、vendor hook 定義、以及 Binder UAPI。

## Mechanism

### 三層標頭分離

```
include/
├── uapi/              ← 第一層：使用者空間可見（透過 make headers_install 匯出）
│   └── linux/foo.h        定義 ioctl 號碼、結構、常數
├── linux/             ← 第二層：核心內部（#include <uapi/linux/foo.h> + 核心專用欄位）
│   └── foo.h              完整定義，含內部欄位、inline 函式
└── asm-generic/       ← 第三層：架構無關預設實現
    └── foo.h              可被 arch/*/include/asm/foo.h 覆蓋
```

**UAPI 分離（Linux 3.5+）：** 之前所有標頭混在 `include/linux/` 中，由 `#ifdef __KERNEL__` 守衛分隔核心與使用者空間部分。3.5 後將使用者空間部分移至 `include/uapi/linux/`，使介面邊界明確。

**典型模式：**
- `include/uapi/linux/binder.h` — 定義 `struct binder_write_read`、`BINDER_WRITE_READ` ioctl 號碼
- `include/linux/binder.h`（若存在）— 添加核心內部結構
- `drivers/android/binder_internal.h` — 完全私有的實現細節

### asm-generic 回退機制

`include/asm-generic/Kbuild` 定義兩類標頭：

- **mandatory-y** — 所有架構必須提供（67 個），如 `atomic.h`、`barrier.h`、`irq.h`。若架構未自訂，build system 生成包含 asm-generic 版本的 wrapper。
- **optional headers** — 架構可選擇提供。

回退流程：
1. `#include <asm/foo.h>` → 尋找 `arch/$(SRCARCH)/include/asm/foo.h`
2. 若不存在，Kbuild 在 build 時生成 wrapper → 包含 `<asm-generic/foo.h>`

### Kbuild 標頭安裝

`make headers_install` 將 `include/uapi/` 下的標頭複製到 sysroot，供 userspace 編譯使用。Android 的 Bionic libc 包含從 ACK 匯出的核心標頭。`include/uapi/Kbuild` 控制哪些子目錄被匯出。

### Device Tree Bindings 共享常數

`include/dt-bindings/` 是一個特殊目錄——其標頭**同時被 Device Tree 原始檔 (`.dts`) 和 C 程式碼使用**。它只包含 `#define` 常數（無型別、無結構），因此 DT 編譯器 (dtc) 和 C 編譯器都能處理。37 個子目錄對應硬體子系統（clock、gpio、interrupt-controller、power 等），含 1,122 個標頭。

### Trace Events 雙重 Include 模式

`include/trace/events/*.h` 使用一種獨特的雙重 `#include` 模式：

1. 標頭定義 `TRACE_EVENT(name, proto, args, struct, assign, print)` 巨集呼叫
2. 第一次 include（由 `<linux/tracepoint.h>` 觸發）→ 展開為函式宣告
3. 在 `.c` 檔中 `#define CREATE_TRACE_POINTS` + include → 展開為完整實現

這允許單一標頭檔同時作為宣告和定義的來源，避免重複。

## Usage Patterns

### 核心模組如何使用標頭

```c
/* 驅動程式典型的 include 模式 */
#include <linux/module.h>     /* 模組基礎 */
#include <linux/device.h>     /* 裝置模型 */
#include <linux/platform_device.h> /* platform bus */
#include <linux/of.h>         /* Device Tree */
#include <linux/clk.h>        /* 時脈框架 */
#include <linux/pm_runtime.h> /* 電源管理 */

/* Android 廠商模組額外使用 */
#include <trace/hooks/sched.h>  /* vendor hook 註冊 */
```

### linux/ 子目錄分類

`linux/` 的 82 個子目錄遵循以下模式：

1. **子系統內部 API：** `sched/`、`net/`、`fs/`、`io_uring/`、`perf/` — 對應 top-level 核心子系統的內部標頭
2. **匯流排/裝置框架：** `usb/`、`spi/`、`mmc/`、`i3c/`、`can/` — 每個匯流排類型一個目錄
3. **硬體 vendor：** `mlx5/`、`qat/`、`habanalabs/` — 特定硬體的共用介面
4. **功能框架：** `clk/`、`regulator/`、`gpio/`、`pinctrl/`、`firmware/` — 跨 SoC 的抽象框架

### 子系統標頭目錄

`include/` 下的頂層子系統目錄（`net/`、`drm/`、`crypto/`、`sound/`、`media/` 等）定義**子系統內部但跨多個 `.c` 檔共用**的介面。它們與 `include/linux/` 的區別是：
- `include/linux/net.h` — 核心對網路子系統的通用介面
- `include/net/sock.h` — 網路子系統內部的 socket 實現細節

## Android Relevance

### KABI 保留填充

`include/linux/android_kabi.h` 定義的巨集是 GKI ABI 穩定性的核心機制。在 `task_struct`、`mm_struct`、`inode` 等約 60+ 個核心結構中嵌入 `ANDROID_KABI_RESERVE(n)` 保留欄位。ABI 凍結後，可透過 `ANDROID_KABI_USE(n, new_field)` 添加欄位而不破壞模組相容性。

### Vendor Hook 標頭

`include/trace/hooks/` 的 29 個標頭是 GKI vendor 擴展的**介面定義**。廠商模組 include 這些標頭並透過 `register_trace_android_vh_*()` 註冊 callback。這些標頭使用 `DECLARE_HOOK` / `DECLARE_RESTRICTED_HOOK` 巨集，基於核心的 tracepoint 基礎設施。

### UAPI Android 目錄

`include/uapi/linux/android/` 下的三個標頭（`binder.h`、`binderfs.h`、`binder_netlink.h`）定義 Android Framework 與 Binder 驅動之間的使用者空間介面。這是 Android userspace 直接依賴的核心 API。

## Cross-References

- [Include Headers 子系統頁面](../subsystems/include-headers.md) — 完整目錄結構與統計
- [GKI 架構](gki.md) — KABI 如何融入 GKI 設計
- [ABI 穩定性](../android/abi-stability.md) — KABI 保留填充的詳細使用
- [Vendor Hooks](vendor-hooks.md) — hook 註冊與呼叫機制
- [Kconfig 與 Build](kconfig-and-build.md) — Kbuild 如何處理標頭安裝
