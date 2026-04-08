---
type: entity
kernel_path: drivers/staging/android/ashmem.c, mm/memfd-ashmem-shim.c
config_option: CONFIG_ASHMEM
upstream: no
related:
  - ../entities/dmabuf-heaps.md
  - ../subsystems/memory-management.md
  - ../subsystems/filesystems.md
last_updated: 2026-04-09
---

# Ashmem — 匿名共享記憶體（已廢棄）

## Overview

Ashmem（Anonymous Shared Memory）是 Android 早期的匿名共享記憶體機制，允許應用程式在不使用命名檔案的情況下共享記憶體區域。它提供 pin/unpin 語意，讓核心在記憶體壓力時可回收未固定的頁面。

Ashmem 目前位於 `drivers/staging/android/`（deprecated staging tree），正被 `memfd_create()` 系統呼叫取代。ACK 提供 memfd-ashmem 相容層確保二進制相容 `[android]`。

## Source Layout

| 檔案 | 行數 | 用途 |
|------|------|------|
| `drivers/staging/android/ashmem.c` | 989 | Ashmem 核心實現 |
| `drivers/staging/android/ashmem.h` | — | 核心標頭 |
| `drivers/staging/android/uapi/ashmem.h` | — | UAPI 標頭（ioctl 定義） |
| `drivers/staging/android/ashmem_rust.rs` | — | Rust 實現 |
| `drivers/staging/android/ashmem_range.rs` | — | Rust range 管理 |
| `drivers/staging/android/ashmem_shrinker.rs` | — | Rust shrinker |
| `drivers/staging/android/ashmem_toggle.rs` | — | Rust toggle |
| `drivers/staging/android/ashmem_rust_exports.c` | — | Rust FFI 匯出 |
| `mm/memfd-ashmem-shim.c` | 214 | Memfd 相容層 |
| `mm/memfd-ashmem-shim.h` | — | 相容層標頭 |
| `mm/memfd-ashmem-shim-internal.h` | — | 內部標頭 |

## Implementation Details

### Ashmem 核心（ashmem.c）

**核心資料結構：**

- **`struct ashmem_area`**（ashmem.c:42-48）— 一個已分配區域：名稱、shmem 後備檔案、大小、保護遮罩、未固定範圍列表
- **`struct ashmem_range`**（ashmem.c:62-69）— 未固定頁面範圍：`pgstart`/`pgend`、`purged` 狀態、LRU 鏈結
- **全域 LRU 列表**（ashmem.c:72）— `ashmem_lru_list`，由 `ashmem_mutex`（:89）保護

**關鍵函式：**

| 函式 | 行號 | 用途 |
|------|------|------|
| `ashmem_open()` | :247-266 | 開啟 /dev/ashmem，分配 ashmem_area |
| `ashmem_mmap()` | :366-448 | 建立 shmem 後備檔案（`shmem_file_setup()`） |
| `ashmem_ioctl()` | :818-885 | 處理 8 個 ioctl 命令 |
| `ashmem_pin()` | :633-694 | 固定頁面範圍（4 種重疊情況） |
| `ashmem_unpin()` | :701-733 | 解除固定，加入 LRU |
| `ashmem_shrink_scan()` | :465-504 | Shrinker 回調：`FALLOC_FL_PUNCH_HOLE` 清除頁面 |
| `ashmem_init_shrinker()` | :519-536 | 註冊核心 shrinker |

**Pin/Unpin 機制：**

1. 新分配的 Ashmem 區域預設為 pinned
2. `ASHMEM_UNPIN` 標記範圍為 unpinned，加入全域 LRU 列表
3. 記憶體壓力時，shrinker 對 LRU 最舊的 unpinned 範圍執行 `FALLOC_FL_PUNCH_HOLE`（fallocate hole punch），釋放實體頁面
4. `ASHMEM_PIN` 重新固定範圍（若已 purged 則回傳 `ASHMEM_WAS_PURGED`）
5. `ASHMEM_GET_PIN_STATUS` 查詢固定狀態

### Memfd-Ashmem 相容層（memfd-ashmem-shim.c）

為 memfd 檔案描述符提供 Ashmem ioctl 相容性，讓舊版程式碼透過 `memfd_create()` 建立的 fd 也能使用 Ashmem 語意。

**關鍵函式：**

| 函式 | 行號 | 用途 |
|------|------|------|
| `memfd_ashmem_shim_ioctl()` | :124-201 | 主要 ioctl 轉換器 |
| `get_prot_mask()` | :61-74 | memfd seals → PROT_* 旗標 |
| `set_prot_mask()` | :76-110 | PROT_* 旗標 → memfd F_SEAL_* |
| `get_name()` | :34-59 | 擷取 memfd 名稱（去除 "memfd:" 前綴） |

**Ioctl 轉換對映：**

| Ashmem Ioctl | Memfd 行為 |
|-------------|-----------|
| `ASHMEM_SET_NAME` / `SET_SIZE` | 回傳 `-EINVAL`（memfd 建立後不可變） |
| `ASHMEM_GET_NAME` | 去除 "memfd:" 前綴回傳檔名 |
| `ASHMEM_GET_SIZE` | `i_size_read(inode)` |
| `ASHMEM_SET_PROT_MASK` | 轉換為 `F_SEAL_FUTURE_WRITE` |
| `ASHMEM_GET_PROT_MASK` | 從 `F_SEAL_WRITE`/`F_SEAL_FUTURE_WRITE` 推導 |
| `ASHMEM_PIN` / `UNPIN` | No-op（永遠回傳 "pinned"） |
| `ASHMEM_GET_PIN_STATUS` | 永遠回傳 pinned |
| `ASHMEM_PURGE_ALL_CACHES` | 檢查 `CAP_SYS_ADMIN` |
| `ASHMEM_GET_FILE_ID` | 回傳 inode number |

**設計決策：** Pin/Unpin 在 memfd 路徑為 no-op — Ashmem 的 LRU 清除機制被 memfd + swap 取代（Android 10+）。

## Userspace Interface

### 傳統 Ashmem 路徑
```
open("/dev/ashmem") → ioctl(SET_NAME) → ioctl(SET_SIZE) → mmap() → [use] → ioctl(UNPIN)
```

### Memfd 路徑（新版）
```
memfd_create("name", MFD_ALLOW_SEALING) → [Ashmem ioctl via shim] → mmap() → [use]
```

## Android-Specific Notes

- **廢棄狀態**：位於 `drivers/staging/android/`，新程式碼應使用 `memfd_create()`
- **二進制相容**：memfd-ashmem-shim 確保舊版 APK 和 native 程式碼繼續運作
- **Rust 實現**：ACK 包含 Ashmem 的 Rust 重新實現（`ashmem_rust.rs` 等）
- **Android 10+ 遷移**：Pin/Unpin 語意已棄用，所有頁面視為永久 pinned

## Cross-References

- [DMA-BUF Heaps](dmabuf-heaps.md) — 現代 DMA 緩衝區分配（取代 Ion，非 Ashmem）
- [記憶體管理子系統](../subsystems/memory-management.md) — memfd-ashmem shim 記錄
- [Filesystems 子系統](../subsystems/filesystems.md) — shmem/tmpfs 後備檔案系統
