---
type: concept
scope: kernel-wide
related:
  - ../concepts/module-system.md
  - ../concepts/locking-primitives.md
  - ../concepts/memory-allocation.md
last_updated: 2026-04-08
---

# Rust in Kernel

## 概述

Android Common Kernel 支持使用 Rust 編寫核心模組，提供記憶體安全的替代方案。Rust 支持由 `CONFIG_RUST=y` 啟用（`gki_defconfig:54`），目前的 Rust 編譯器版本為 1.91.1（`build.config.constants`）。最重要的 Rust 實現是 **Rust Binder**——Android IPC 機制的 Rust 重寫。

核心 Rust 基礎設施位於 `rust/` 目錄，提供 96 個核心抽象子模組，涵蓋設備驅動、檔案系統、網路、記憶體管理等核心子系統。

## 機制

### 配置與依賴

**Kconfig 定義**（`init/Kconfig:2148-2173`）：

```
config RUST
    bool "Rust support"
    depends on HAVE_RUST && RUST_IS_AVAILABLE
    select EXTENDED_MODVERSIONS if MODVERSIONS
    depends on !MODVERSIONS || GENDWARFKSYMS
    depends on !GCC_PLUGIN_RANDSTRUCT && !RANDSTRUCT
    depends on !DEBUG_INFO_BTF || (PAHOLE_HAS_LANG_EXCLUDE && !LTO)
    depends on !CFI || HAVE_CFI_ICALL_NORMALIZE_INTEGERS_RUSTC
```

要求 Rust 編譯器版本 >= 1.87.0（`RUSTC_VERSION >= 108700`）。

### 構建系統

**Makefile**（`rust/Makefile`）：

```makefile
obj-$(CONFIG_RUST) += core.o compiler_builtins.o ffi.o
always-$(CONFIG_RUST) += bindings/bindings_generated.rs
obj-$(CONFIG_RUST) += helpers/helpers.o
obj-$(CONFIG_RUST) += bindings.o pin_init.o kernel.o
obj-$(CONFIG_RUST) += uapi.o exports.o
```

構建產出：
- `bindings_generated.rs` — bindgen 自動生成的 C 綁定
- `exports_*.h` — 匯出給 C 的符號
- `kernel.rlib` — 核心 Rust 函式庫

### FFI 整合

**型別映射**（`rust/ffi.rs`）：

```rust
pub type c_char = u8;           // unsigned char（用 -funsigned-char）
pub type c_int = i32;
pub type c_long = isize;        // intptr_t（所有平台）
pub type c_ulong = usize;
pub type c_void = core::ffi::c_void;
pub use core::ffi::CStr;
```

**Bindgen 參數**（`rust/bindgen_parameters`）：

```
--blocklist-type __kernel_*size_t     // 手動映射
--opaque-type spinlock                // 隱藏內部細節
--with-derive-custom-struct .*=MaybeZeroable
--no-doc-comments
```

**Helpers 機制**（`rust/helpers/`，59 個檔案）：

C 巨集和 inline 函數無法直接從 Rust 呼叫，因此創建 C 助手函數包裝它們：

- `atomic.c` — 原子操作包裝
- `barrier.c` — 記憶體屏障
- `binder.c` — Binder 特定助手
- `spinlock.c` — 自旋鎖包裝
- 包裝的巨集：`container_of`、`ACCESS_ONCE`、`IS_ERR`、`PTR_ERR` 等

### 核心抽象（rust/kernel/）

**96 個子模組**，按功能分類：

**核心系統**：
- `alloc.rs` — 記憶體分配抽象
- `error.rs` — 錯誤處理（對應 C 的 errno）
- `types.rs` — 基本類型
- `sync/` — 同步原語（mutex、rwlock、atomic）
- `prelude/` — 常用匯入集合

**設備與驅動**：
- `device.rs` — 設備抽象
- `driver.rs` — 驅動框架
- `platform.rs` — 平台設備
- `pci/` — PCI 設備
- `block.rs` / `block/` — 區塊設備

**記憶體管理**：
- `mm/` — 記憶體管理抽象
- `page.rs` — 頁面操作
- `dma.rs` — DMA 抽象

**資料結構**：
- `list/` — 鏈表容器
- `rbtree.rs` — 紅黑樹
- `maple_tree.rs` — Maple 樹

**追蹤與除錯**：
- `tracepoint.rs` — 追蹤點
- `jump_label.rs` — 靜態分支
- `debugfs.rs` / `debugfs/` — debugfs

**其他**：
- `fs.rs` / `fs/` — 檔案系統
- `net/` — 網路
- `irq/` — 中斷
- `task.rs` — 任務/進程
- `cred.rs` — 認證資訊
- `security.rs` — 安全模組

### 安全性抽象

Rust 的核心價值在於**編譯時記憶體安全**。核心抽象使用以下模式：

**SAFETY 註釋**（`rust/kernel/error.rs`）：

```rust
const unsafe fn from_errno_unchecked(errno: c_int) -> Error {
    // SAFETY: errno 已在上方驗證為有效範圍
}
```

每個 `unsafe` 區塊都必須附帶 `// SAFETY:` 註釋，解釋為何此操作是安全的。

**Pin-Init 框架**（`rust/pin-init/`）：

確保在堆棧上正確初始化引腳結構，避免 TOCTOU（Time-of-Check-to-Time-of-Use）問題。

### Rust Binder

**入口**（`drivers/android/binder/rust_binder_main.rs`）：

```rust
module! {
    type: BinderModule,
    name: "rust_binder",
    authors: ["Wedson Almeida Filho", "Alice Ryhl"],
    description: "Android Binder",
    license: "GPL",
}
```

**子模組**：
- `allocation` — 記憶體分配
- `context` — Binder 上下文
- `node` — Binder 節點
- `process` — 進程管理
- `thread` — 執行緒管理
- `transaction` — 事務處理
- `trace` — 追蹤
- `deferred_close` — 延遲關閉

**C 互操作**：提供 `extern "C"` 函數（`init_rust_binderfs()`、`rust_binderfs_create_proc_file()` 等）與 binderfs 整合。

### 範例模組

**位於** `samples/rust/`：

- `rust_minimal.rs` — 最小模組範例
- `rust_print_main.rs` — 列印演示
- `rust_misc_device.rs` — 混雜設備
- `rust_driver_platform.rs` — 平台驅動
- `rust_driver_pci.rs` — PCI 驅動
- `rust_driver_usb.rs` — USB 驅動
- `rust_dma.rs` — DMA 操作
- `rust_configfs.rs` — ConfigFS
- `rust_debugfs*.rs` — DebugFS

## 使用模式

### Rust 模組 vs C 模組

| 面向 | Rust 模組 | C 模組 |
|------|----------|--------|
| 記憶體安全 | 編譯時保證 | 依賴開發者 |
| 效能 | 等同（零成本抽象）| 基準 |
| 生態成熟度 | 早期（96 個抽象）| 完整 |
| FFI 開銷 | 最小（bindgen 生成）| 無 |
| 學習曲線 | 較高 | 較低（核心傳統）|

### 支援的驅動類型

目前 Rust 抽象覆蓋：平台設備、PCI、I2C、USB、輔助匯流排、區塊設備、網路、DRM/GPU、DMA。

## Android 相關性

Rust 在 Android 核心中的主要價值：

1. **Binder 安全性**：Binder 是 Android IPC 的核心，記憶體錯誤可能導致嚴重安全漏洞。Rust 重寫提供了編譯時記憶體安全保證。

2. **驅動安全**：隨著更多 Rust 抽象成熟，廠商可選擇使用 Rust 編寫安全的驅動模組。

3. **GKI 整合**：`CONFIG_RUST=y` 已在 GKI defconfig 中啟用，Rust 是 GKI 的一級公民。

## 交叉參考

- [Module 系統](module-system.md) — Rust 模組的載入與符號管理
- [鎖定原語](locking-primitives.md) — Rust `sync/` 抽象包裝的 C 鎖
- [記憶體分配](memory-allocation.md) — Rust `alloc.rs` 包裝的分配器
