---
type: source
source_path: drivers/firmware/efi/
lines: 17024
ingested: 2026-04-10
---

# Source: `drivers/firmware/efi/`

## Summary

EFI/UEFI (Unified Extensible Firmware Interface) 完整子系統，支援 EFI runtime services、啟動服務、記憶體映射、變數管理、韌體膠囊更新、持久儲存 (pstore)、CPER 錯誤報告等。75 個檔案共 17,024 行，包含主驅動和 `libstub/`（EFI 啟動 stub）。

## Key Functions

| 函式/檔案 | 行數 | 用途 |
|-----------|------|------|
| `efi.c` | ~800 | EFI 核心子系統初始化與 runtime 管理 |
| `efi-init.c` | ~300 | EFI 啟動時初始化（記憶體映射解析） |
| `runtime-wrappers.c` | ~600 | EFI runtime 呼叫的虛擬位址管理 |
| `vars.c` | ~400 | EFI 變數讀寫介面 |
| `capsule.c` | ~300 | 韌體膠囊更新機制 |
| `pstore.c` | ~200 | 透過 EFI 變數實現的持久崩潰日誌 |
| `reboot.c` | ~100 | EFI 重啟/關機 |
| `memmap.c` | ~200 | EFI 記憶體映射管理 |
| `memattr.c` | ~200 | 記憶體屬性表 |
| `cper.c` / `cper-arm.c` / `cper-x86.c` | ~1,500 | 平台錯誤記錄 (CPER) |
| `libstub/` | ~8,000 | EFI 啟動 stub（ARM/ARM64/RISC-V/LoongArch） |

## Notable Implementation Details

**EFI Stub**：`libstub/` 包含在 EFI 環境中啟動 Linux 核心的程式碼，處理解壓縮（gzip、zstd）、KASLR 隨機化、Secure Boot 驗證、DTB/ACPI 傳遞、TPM 測量。支援 ARM32、ARM64、RISC-V、LoongArch 架構。

**CPER 錯誤記錄**：Common Platform Error Record 解析器處理硬體錯誤報告（記憶體錯誤、PCIe 錯誤、處理器錯誤），近期加入 CXL 記憶體裝置支援。

**STM (Standalone MM) 支援**：`stmm/` 子目錄支援 EFI Standalone Management Mode，用於安全世界的 EFI 變數儲存。

**Android 相關性**：中等 — 主要用於 ARM64 伺服器和部分企業裝置。Qualcomm 裝置透過 QSEECOM UEFI Secure App 存取 EFI 變數（見 `qcom/qcom_qseecom_uefisecapp.c`），而非直接使用 EFI runtime。

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md)
- [Qualcomm SCM](src-drivers-firmware-qcom-scm.md) — QSEECOM UEFI 變數存取
