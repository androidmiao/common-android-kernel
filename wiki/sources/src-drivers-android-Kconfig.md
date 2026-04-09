---
type: source
source_path: drivers/android/Kconfig
lines: 122
ingested: 2026-04-09
related:
  - ../concepts/kconfig-and-build.md
  - ../concepts/gki.md
  - ../entities/binder.md
  - ../entities/vendor-hooks-driver.md
  - ../entities/debug-kinfo.md
---

# Source: `drivers/android/Kconfig`

## Summary

Android 驅動程式的 Kconfig 選單定義，包含 7 個配置選項，控制 Binder IPC（C/Rust）、Binderfs、Vendor Hooks、Debug Kinfo、KABI Reserve 和 Vendor/OEM Data 的編譯。

## Key Functions

（Kconfig 檔案無函式，以配置選項列表替代）

| Config Option | Default | Dependencies | Purpose |
|--------------|---------|--------------|---------|
| `ANDROID_BINDER_IPC` | n | MMU, NET | C 版 Binder IPC 驅動 |
| `ANDROID_BINDER_IPC_RUST` | — | RUST, MMU, !ANDROID_BINDER_IPC | Rust 版 Binder IPC 驅動（與 C 版互斥） |
| `ANDROID_BINDERFS` | n | ANDROID_BINDER_IPC | Binderfs 虛擬檔案系統 |
| `ANDROID_BINDER_DEVICES` | "binder,hwbinder,vndbinder" | BINDER_IPC \|\| BINDER_IPC_RUST | 預設裝置名稱 |
| `ANDROID_BINDER_ALLOC_KUNIT_TEST` | KUNIT_ALL_TESTS | BINDER_IPC, KUNIT | Binder 分配器 KUnit 測試 |
| `ANDROID_VENDOR_HOOKS` | — | TRACEPOINTS | 廠商 Hook 框架 |
| `ANDROID_DEBUG_KINFO` | — | KALLSYMS | 核心除錯資訊匯出 |
| `ANDROID_KABI_RESERVE` | y | 64BIT | KABI 保留填充（GKI ABI 穩定性） |
| `ANDROID_VENDOR_OEM_DATA` | y | 64BIT | 廠商/OEM 資料填充 |

## Notable Implementation Details

- **C/Rust 互斥**：`ANDROID_BINDER_IPC_RUST` 依賴 `!ANDROID_BINDER_IPC`，確保同時只啟用一種實現。
- **三個預設裝置**：`binder`（framework）、`hwbinder`（HAL）、`vndbinder`（vendor），可透過模組參數覆蓋。
- **KABI/OEM 建議**：兩者預設啟用，僅建議非 GKI 核心映像關閉，並明確警告關閉後不受 Google 核心團隊支持。
- **KUnit 測試**：僅在 `KUNIT_ALL_TESTS` 或手動啟用時編譯。
