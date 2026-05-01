---
type: source
source_path: drivers/firmware/Kconfig
lines: 302
ingested: 2026-04-10
---

# Source: `drivers/firmware/Kconfig`

## Summary

Firmware Drivers 子系統的頂層 Kconfig 配置選單。定義 17 個直接配置選項，並透過 `source` 指令引入 14 個子目錄的 Kconfig（arm_scmi、arm_ffa、broadcom、cirrus、google、efi、imx、meson、microchip、psci、qcom、samsung、smccc、tegra、xilinx）。

## Key Configuration Options

| 選項 | 類型 | 架構依賴 | 功能 |
|------|------|---------|------|
| `ARM_SCPI_PROTOCOL` | tristate | ARM/ARM64 + MAILBOX | SCP 舊版通訊協定 |
| `ARM_SDE_INTERFACE` | bool | ARM64 | 軟體委派例外介面 (RAS) |
| `EDD` | tristate | X86 | BIOS 增強磁碟偵測 |
| `EDD_OFF` | bool | EDD | EDD 預設關閉 |
| `FIRMWARE_MEMMAP` | bool | (EXPERT) | 韌體記憶體映射 sysfs |
| `DMIID` | bool | DMI | DMI 識別碼匯出 |
| `DMI_SYSFS` | tristate | SYSFS + DMI | DMI 原始表匯出 |
| `ISCSI_IBFT_FIND` | bool | X86 + ISCSI_IBFT | iSCSI iBFT 記憶體搜尋 |
| `ISCSI_IBFT` | tristate | ACPI + SCSI | iSCSI iBFT 屬性模組 |
| `RASPBERRYPI_FIRMWARE` | tristate | BCM2835_MBOX | RPi 韌體驅動 |
| `FW_CFG_SYSFS` | tristate | 多架構 + HAS_IOPORT_MAP | QEMU fw_cfg sysfs |
| `FW_CFG_SYSFS_CMDLINE` | bool | FW_CFG_SYSFS | QEMU fw_cfg 命令列參數 |
| `INTEL_STRATIX10_SERVICE` | tristate | INTEL_SOCFPGA + ARM64 | Stratix10 服務層 |
| `INTEL_STRATIX10_RSU` | tristate | STRATIX10_SERVICE | 遠端系統更新 |
| `MTK_ADSP_IPC` | tristate | MTK_ADSP_MBOX | MediaTek ADSP IPC |
| `SYSFB` | bool | — | 系統 framebuffer (被 select) |
| `SYSFB_SIMPLEFB` | bool | X86/EFI | 通用 framebuffer 標記 |
| `TH1520_AON_PROTOCOL` | tristate | ARCH_THEAD + MAILBOX | T-Head AON 協定 |
| `TI_SCI_PROTOCOL` | tristate | TI_MESSAGE_MANAGER | TI 系統控制介面 |
| `TRUSTED_FOUNDATIONS` | bool | ARM + CPU_V7 | NVIDIA 安全監控 |
| `TURRIS_MOX_RWTM` | tristate | ARCH_MVEBU + HAS_DMA + OF + MAILBOX | Turris Mox 韌體 |
| `TURRIS_MOX_RWTM_KEYCTL` | bool | TURRIS_MOX_RWTM + KEYS | ECDSA 簽名支援 |

## Notable Implementation Details

Kconfig 結構反映了 firmware 子系統的歷史演進：早期驅動（SCPI、Trusted Foundations、EDD）直接定義在頂層，較新的子系統（SCMI、FFA、QCOM、Samsung）則以子目錄形式管理。架構依賴清楚地劃分了各驅動的適用平台 — ARM64 相關的佔絕大多數（與 Android 高度相關），X86 驅動（EDD、DMI、iSCSI iBFT）則主要服務伺服器/桌面用途。

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md)
- [Kconfig 與 Build 系統](../concepts/kconfig-and-build.md)
