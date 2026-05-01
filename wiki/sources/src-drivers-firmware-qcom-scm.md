---
type: source
source_path: drivers/firmware/qcom/
lines: 4447
ingested: 2026-04-10
---

# Source: `drivers/firmware/qcom/`

## Summary

Qualcomm Secure Configuration Manager (SCM) 驅動，是所有 Qualcomm Snapdragon Android 裝置與 TrustZone 安全世界通訊的關鍵介面。透過 SMC 呼叫存取 Qualcomm Secure Execution Environment (QSEE)，支援 CPU 啟動配置、下載模式控制、安全記憶體分配、QSEECOM 可信任應用程式互動。8 個 .c/.h 檔案共 4,447 行。

## Key Functions

| 函式/檔案 | 行數 | 用途 |
|-----------|------|------|
| `qcom_scm.c` | ~2,500 | 核心 SCM 驅動：API 介面、ICQ 頻寬投票、reset controller |
| `qcom_scm-smc.c` | ~350 | SMC 呼叫傳輸層（現代介面） |
| `qcom_scm-legacy.c` | ~250 | 舊版 SCM 介面（早期 SoC） |
| `qcom_tzmem.c` | ~400 | TrustZone 記憶體分配器（Generic/SHM Bridge 雙模式） |
| `qcom_qseecom.c` | ~300 | QSEECOM 介面驅動（可信任應用程式通訊） |
| `qcom_qseecom_uefisecapp.c` | ~600 | UEFI Secure App 驅動（EFI 變數存取） |

## Notable Implementation Details

**雙傳輸模式**：早期 Qualcomm SoC 使用 legacy 介面（`qcom_scm-legacy.c`），較新的 SoC 使用基於 SMCCC 的現代 SMC 介面（`qcom_scm-smc.c`）。驅動在啟動時偵測並選擇合適的傳輸。

**TrustZone 記憶體**：`qcom_tzmem.c` 實現兩種模式 — Generic 模式分配非快取 DMA 記憶體供 TZ 使用；SHM Bridge 模式提供共享記憶體橋接，安全性更高。

**QSEECOM 子系統**：允許核心驅動與 TrustZone 中的 Trusted Application 通訊，`qcom_qseecom_uefisecapp.c` 展示了如何透過 QSEECOM 存取 UEFI Secure App 管理 EFI 變數。

**Kconfig 選項**：`QCOM_SCM`（核心，自動 select `QCOM_TZMEM`）、`QCOM_TZMEM`（記憶體分配器，可選 Generic 或 SHM Bridge）、`QCOM_QSEECOM`（QSEE 介面）、`QCOM_QSEECOM_UEFISECAPP`（UEFI 變數）。

## Open Questions

- QSEECOM 在 Android 上的具體使用場景（指紋、DRM、Keymaster）的驅動位於何處？

## Cross-References

- [Firmware 子系統](../subsystems/firmware.md)
- [Security 子系統](../subsystems/security.md) — TrustZone 與 LSM 的關係
