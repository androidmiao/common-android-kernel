---
type: source
source_path: drivers/android/binder_netlink.h
lines: 22
ingested: 2026-04-09
related:
  - ../entities/binder.md
  - ./src-drivers-android-binder_netlink-c.md
---

# Source: `drivers/android/binder_netlink.h`

## Summary

Binder Generic Netlink 標頭檔（自動生成）。定義多播群組枚舉和 `binder_nl_family` 外部宣告。

## Notable Implementation Details

- **自動生成**：由 YNL-GEN 從 `Documentation/netlink/specs/binder.yaml` 生成。
- **多播群組**：`BINDER_NLGRP_REPORT = 0`。
- **外部家族**：`extern struct genl_family binder_nl_family`。
