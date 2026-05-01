---
type: source
source_path: drivers/firmware/google/
lines: 2359
ingested: 2026-04-10
---

# Source: `drivers/firmware/google/`

## Summary

Google Coreboot/Chromebook 韌體驅動集合，提供 coreboot 表發現、SMI 事件記錄、VPD (Vital Product Data) 讀取、韌體記憶體控制台、CBMEM 區域匯出等功能。12 個 .c/.h 檔案共 2,359 行。主要用於 Chromebook 裝置，在 Android 手機上較少使用。

## Key Functions

| 函式/檔案 | 行數 | 用途 |
|-----------|------|------|
| `coreboot_table.c` | ~350 | Coreboot 表 bus 驅動：發現韌體提供的資料區塊 |
| `gsmi.c` | ~700 | SMI 介面：事件記錄、NVRAM 存取（X86 + ACPI + DMI） |
| `vpd.c` | ~250 | VPD sysfs 匯出：RO/RW 分區的硬體配置資料 |
| `memconsole-coreboot.c` | ~100 | 韌體記憶體控制台（核心啟動日誌） |
| `memconsole-x86-legacy.c` | ~80 | X86 舊版記憶體控制台 |
| `framebuffer-coreboot.c` | ~100 | 韌體 framebuffer 裝置註冊 |
| `cbmem.c` | ~200 | CBMEM（coreboot 記憶體）區域匯出至 sysfs |

## Notable Implementation Details

**Coreboot 表 bus 模型**：`coreboot_table.c` 實現了一個自訂的匯流排類型 (`coreboot_bus_type`)，將 coreboot 表中的每個條目註冊為 `coreboot_device`。其他驅動（memconsole、vpd、cbmem）作為此 bus 上的驅動匹配對應的表條目。發現機制支援 ACPI（`GOOGCB00`）和 Device Tree。

**GSMI**：透過 BIOS SMI 呼叫存取事件記錄和 NVRAM，使用 DMA-safe 緩衝區（`gsmi_buf`）傳遞資料，需要 32-bit 物理位址。

**Kconfig 選項**：`GOOGLE_FIRMWARE`（頂層選單）、`GOOGLE_COREBOOT_TABLE`、`GOOGLE_SMI`、`GOOGLE_VPD`、`GOOGLE_MEMCONSOLE`、`GOOGLE_CBMEM`。

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md)
