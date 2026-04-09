---
type: source
source_path: drivers/android/Makefile
lines: 10
ingested: 2026-04-09
related:
  - ../concepts/kconfig-and-build.md
  - ./src-drivers-android-Kconfig.md
---

# Source: `drivers/android/Makefile`

## Summary

Android 驅動程式的建置規則，定義 5 個編譯目標與其 CONFIG 依賴。

## Key Functions

| Target | Config | Objects |
|--------|--------|---------|
| Binderfs | `CONFIG_ANDROID_BINDERFS` | `binderfs.o` |
| Binder (C) | `CONFIG_ANDROID_BINDER_IPC` | `binder.o` + `binder_alloc.o` + `binder_netlink.o` |
| Binder KUnit | `CONFIG_ANDROID_BINDER_ALLOC_KUNIT_TEST` | `tests/` 子目錄 |
| Binder (Rust) | `CONFIG_ANDROID_BINDER_IPC_RUST` | `binder/` 子目錄 |
| Vendor Hooks | `CONFIG_ANDROID_VENDOR_HOOKS` | `vendor_hooks.o` |
| Debug Kinfo | `CONFIG_ANDROID_DEBUG_KINFO` | `debug_kinfo.o` |

## Notable Implementation Details

- **ccflags**：`-I$(src)` 確保 trace event 標頭檔可被找到。
- **C Binder 為多物件**：`binder.o` + `binder_alloc.o` + `binder_netlink.o` 三個 .o 檔。
- **Rust Binder 為子目錄**：`binder/` 子目錄有自己的 Makefile，編譯所有 .rs 檔。
- **互斥性**：由 Kconfig 保證 C 和 Rust 版本不會同時編譯。
