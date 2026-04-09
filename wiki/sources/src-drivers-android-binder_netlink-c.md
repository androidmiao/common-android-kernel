---
type: source
source_path: drivers/android/binder_netlink.c
lines: 33
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ./src-drivers-android-binder-c.md
  - ./src-drivers-android-binder_netlink-h.md
---

# Source: `drivers/android/binder_netlink.c`

## Summary

Binder Generic Netlink 家族定義檔。自動生成自 `Documentation/netlink/specs/binder.yaml`，使用 YNL-GEN 工具。定義 `binder_nl_family` Generic Netlink 家族，用於將 binder 交易報告（transaction reports）傳送至使用者空間。

## Key Functions

| Function | Line Range | Purpose |
|----------|-----------|---------|
| （無函式定義） | — | 本檔案僅定義資料結構 |

## Notable Implementation Details

- **自動生成**：由 `tools/net/ynl/ynl-regen.sh` 從 YAML 規格生成，不應手動編輯。
- **Netlink 家族**：`binder_nl_family`，名稱從 `BINDER_FAMILY_NAME` 取得，支援 netns。
- **多播群組**：`BINDER_NLGRP_REPORT`（"report" 群組），用於交易報告事件。
- **空操作表**：`binder_nl_ops` 為空陣列，所有操作透過多播事件而非請求/回應進行。
- **使用處**：`binder.c` 中的 `binder_netlink_report()` 函式透過此家族發送交易完成/錯誤通知。

## Open Questions

- 未來是否會增加請求/回應類型的 Netlink 操作？
