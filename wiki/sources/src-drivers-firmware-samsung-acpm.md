---
type: source
source_path: drivers/firmware/samsung/
lines: 1111
ingested: 2026-04-10
---

# Source: `drivers/firmware/samsung/`

## Summary

Samsung Exynos ACPM (Alive Clock and Power Manager) 驅動，負責與 Exynos SoC 上的 Cortex-M 微控制器韌體通訊，管理電源域、DVFS 頻率調整和 PMIC 操作。6 個 .c/.h 檔案共 1,111 行。

## Key Functions

| 函式/檔案 | 行數 | 用途 |
|-----------|------|------|
| `exynos-acpm.c` | ~400 | 核心 ACPM 協定驅動：mailbox 通訊、訊息排隊 |
| `exynos-acpm-dvfs.c` | ~250 | DVFS 控制：頻率表查詢、電壓/頻率設定 |
| `exynos-acpm-pmic.c` | ~300 | PMIC 控制：I2C-like 暫存器讀寫透過 ACPM |

## Notable Implementation Details

**三層架構**：`exynos-acpm.c` 提供通用的訊息傳輸層，`exynos-acpm-dvfs.c` 和 `exynos-acpm-pmic.c` 作為功能特定的客戶端構建在其上。每個功能使用不同的 mailbox 通道。

**Mailbox 通訊**：每條訊息由固定格式的 32-bit 字組成，包含命令 ID、通道號、資料負載。韌體端（Cortex-M）處理完畢後通過中斷回覆。

**Kconfig**：`EXYNOS_ACPM_PROTOCOL`（depends on `ARCH_EXYNOS` + `MAILBOX`）。

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md)
