---
type: source
source_path: include/linux/android_vendor.h
lines: 50
ingested: 2026-04-09
---

# Source: `android_vendor.h`

## Summary

Android 廠商/OEM 資料填充標頭，定義巨集在核心結構中嵌入 `u64` 欄位供廠商模組存取。這是 vendor hooks 存取私有行程/記憶體/裝置狀態的資料載體。透過 `CONFIG_ANDROID_VENDOR_OEM_DATA` 控制。

## Key Functions

| 巨集 | 行 | 用途 |
|------|-----|------|
| `ANDROID_VENDOR_DATA(n)` | 30 | 在結構中嵌入 `u64 android_vendor_data##n` |
| `ANDROID_VENDOR_DATA_ARRAY(n, s)` | 31 | 嵌入 `u64` 陣列，大小為 `s` |
| `ANDROID_OEM_DATA(n)` | 33 | 嵌入 `u64 android_oem_data##n`（OEM 專用） |
| `ANDROID_OEM_DATA_ARRAY(n, s)` | 34 | OEM 資料陣列 |
| `android_init_vendor_data(p, n)` | 36-37 | 初始化（memset 清零）vendor 資料欄位 |
| `android_init_oem_data(p, n)` | 38-39 | 初始化 OEM 資料欄位 |

## Notable Implementation Details

**雙層資料空間：** 區分 VENDOR（SoC 廠商如 Qualcomm、MediaTek）和 OEM（設備製造商如 Samsung、Xiaomi）兩種填充，允許兩個不同層級的模組在同一結構中儲存資料。

**與 vendor hooks 的搭配：** 廠商模組透過 vendor hook 取得結構指標（如 `task_struct *p`），然後透過 `p->android_vendor_data1` 存取/修改私有資料。這是 GKI 允許廠商擴展核心行為的主要資料機制。

**條件編譯：** `CONFIG_ANDROID_VENDOR_OEM_DATA=n` 時所有巨集為空，結構大小不受影響。

## Open Questions

- `task_struct` 中嵌入了多少 vendor/OEM 資料欄位？（已知至少 96 bytes）
