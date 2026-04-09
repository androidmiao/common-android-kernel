---
type: api
direction: user-to-kernel
related:
  - ../subsystems/networking.md
  - ../entities/binder.md
  - ../subsystems/security.md
  - ../concepts/bpf.md
last_updated: 2026-04-09
---

# Netlink 介面

## Overview

Netlink 是 Linux 核心與使用者空間之間的 socket 式通訊機制（AF_NETLINK），支援雙向、非同步、多播的訊息傳遞。相較於 ioctl 的同步點對點模式，Netlink 提供了更結構化的訊息格式和群播能力。ACK 定義了 **17 個**協定家族（Protocol Family）和超過 **20 個** Generic Netlink 家族，核心實現約 **5,000 行**。

## Interface Definition

### 核心實現

| 檔案 | 行數 | 用途 |
|------|------|------|
| `net/netlink/af_netlink.c` | 2,953 | Netlink socket 家族實現 |
| `net/netlink/genetlink.c` | 1,997 | Generic Netlink 框架 |
| `net/netlink/policy.c` | 12,141 | 屬性驗證策略 |

### 訊息格式

```
┌─────────────────────────────────────────┐
│ nlmsghdr (16 bytes)                     │
│   nlmsg_len (u32)   — 完整訊息長度     │
│   nlmsg_type (u16)  — 訊息類型/命令    │
│   nlmsg_flags (u16) — 控制旗標         │
│   nlmsg_seq (u32)   — 序列號           │
│   nlmsg_pid (u32)   — 發送端 port ID   │
├─────────────────────────────────────────┤
│ Protocol-specific payload               │
│   ├── nlattr (4 bytes per attribute)    │
│   │     nla_len (u16)  — 屬性長度      │
│   │     nla_type (u16) — 屬性類型+旗標 │
│   │     [data, 4-byte aligned]          │
│   └── ...                               │
└─────────────────────────────────────────┘
```

**訊息旗標**（定義於 `include/uapi/linux/netlink.h:60-87`）：
- 基本：`NLM_F_REQUEST`(0x01)、`NLM_F_MULTI`(0x02)、`NLM_F_ACK`(0x04)、`NLM_F_ECHO`(0x08)
- GET 修飾：`NLM_F_ROOT`、`NLM_F_MATCH`、`NLM_F_DUMP`
- NEW 修飾：`NLM_F_REPLACE`、`NLM_F_EXCL`、`NLM_F_CREATE`、`NLM_F_APPEND`

**屬性類型**（`include/uapi/linux/netlink.h`）：
`NLA_U8`、`NLA_U16`、`NLA_U32`、`NLA_U64`、`NLA_STRING`、`NLA_NUL_STRING`、`NLA_BINARY`、`NLA_NESTED`、`NLA_NESTED_ARRAY`、`NLA_BITFIELD32`、`NLA_FLAG`、`NLA_REJECT` 等。

### 協定家族

定義於 `include/uapi/linux/netlink.h:9-31`：

| 編號 | 常數 | 核心 Socket 建立位置 | 用途 |
|------|------|---------------------|------|
| 0 | `NETLINK_ROUTE` | `net/core/rtnetlink.c:2024` | 路由/網路介面/位址管理 |
| 2 | `NETLINK_USERSOCK` | 使用者建立 | 使用者空間協定 |
| 4 | `NETLINK_SOCK_DIAG` | `net/core/sock_diag.c` | Socket 狀態診斷 |
| 5 | `NETLINK_NFLOG` | `net/netfilter/nfnetlink_log.c` | Netfilter 日誌 |
| 6 | `NETLINK_XFRM` | `net/xfrm/xfrm_user.c:4216` | IPSec SA/策略管理 |
| 7 | `NETLINK_SELINUX` | SELinux 模組 | SELinux 事件通知 |
| 9 | `NETLINK_AUDIT` | 審計子系統 | 系統審計事件 |
| 10 | `NETLINK_FIB_LOOKUP` | FIB 子系統 | FIB 查詢 |
| 11 | `NETLINK_CONNECTOR` | Connector 子系統 | 核心連接器 |
| 12 | `NETLINK_NETFILTER` | `net/netfilter/nfnetlink.c:778` | Netfilter 子系統 |
| 15 | `NETLINK_KOBJECT_UEVENT` | `lib/kobject_uevent.c` | 裝置熱插拔事件 |
| 16 | `NETLINK_GENERIC` | `net/netlink/genetlink.c` | Generic Netlink 框架 |

### 主要協定詳述

#### NETLINK_ROUTE（RTNetlink）[upstream]

最重要的 Netlink 協定，實現於 `net/core/rtnetlink.c`（7,103 行）。

**RTM_* 訊息類型**（`include/uapi/linux/rtnetlink.h`）：

| 命令群組 | NEW/DEL/GET | 用途 |
|----------|-------------|------|
| LINK | 16/17/18 | 網路介面管理 |
| ADDR | 20/21/22 | IP 位址管理 |
| ROUTE | 24/25/26 | 路由表管理 |
| NEIGH | 28/29/30 | 鄰居（ARP/NDP）管理 |
| RULE | 32/33/34 | 路由規則管理 |
| QDISC | 36/37/38 | 流量控制佇列 |
| TCLASS | 40/41/42 | 流量分類 |
| TFILTER | 44/45/46 | 流量過濾器 |
| MDB | 84/85/86 | 多播橋接資料庫 |
| NSID | 88/89/90 | 網路命名空間 ID |

**屬性策略**（`rtnetlink.c`）：`ifla_policy[]`（line 2201）、`ifla_info_policy[]`（line 2263）、`ifla_xdp_policy[]`（line 2305）。

**註冊機制**：`rtnl_register_internal()` @ line 387，使用 protocol × message_type 索引的 handler 表。

#### NETLINK_NETFILTER [upstream]

實現於 `net/netfilter/nfnetlink.c`（821 行），採用子系統式派發模型：

| 子系統編號 | 名稱 | 用途 |
|-----------|------|------|
| 1 | `NFNL_SUBSYS_CTNETLINK` | 連線追蹤 |
| 2 | `NFNL_SUBSYS_CTNETLINK_EXP` | 連線追蹤預期 |
| 3 | `NFNL_SUBSYS_QUEUE` | 封包佇列 |
| 6 | `NFNL_SUBSYS_IPSET` | IP 集合 |
| 10 | `NFNL_SUBSYS_NFTABLES` | nftables |

**多播群組**：`NFNLGRP_CONNTRACK_NEW`/`UPDATE`/`DESTROY`、`NFNLGRP_NFTABLES`、`NFNLGRP_NFTRACE`。

#### NETLINK_KOBJECT_UEVENT [upstream]

實現於 `lib/kobject_uevent.c`，負責裝置模型事件通知。

**事件類型**（lines 50-59）：`KOBJ_ADD`、`KOBJ_REMOVE`、`KOBJ_CHANGE`、`KOBJ_MOVE`、`KOBJ_ONLINE`、`KOBJ_OFFLINE`、`KOBJ_BIND`、`KOBJ_UNBIND`。

### Generic Netlink 框架

Generic Netlink（NETLINK_GENERIC, 協定 16）提供動態家族註冊機制，避免靜態協定編號耗盡。

**家族註冊 API**（`net/netlink/genetlink.c`）：
- `genl_register_family()` @ line 780 — 註冊家族，分配動態 ID
- ID 範圍：`GENL_MIN_ID`(0x10) 到 `GENL_MAX_ID`(1023)
- 使用 IDR 進行 ID 管理（line 69：`DEFINE_IDR(genl_fam_idr)`）

**`struct genl_family`**（`include/net/genetlink.h:78-116`）：

```c
struct genl_family {
    char name[GENL_NAMSIZ];      // 家族名稱（16 字元）
    u32 version;                  // 協定版本
    u32 maxattr;                  // 最大屬性數
    bool netnsok;                 // 網路命名空間支援
    bool parallel_ops;            // 平行操作
    const struct genl_ops *ops;   // 操作定義
    const struct genl_multicast_group *mcgrps; // 多播群組
    int (*pre_doit)(...);         // 操作前回呼
    int (*post_doit)(...);        // 操作後回呼
};
```

**主要 Generic Netlink 家族**：

| 家族名稱 | 檔案 | 用途 |
|----------|------|------|
| `nl80211` | `net/wireless/nl80211.c`（19,027 行） | WiFi/802.11 管理（最大家族） |
| `devlink_nl` | `net/devlink/core.c` | 裝置連結管理 |
| `mptcp_genl` | `net/mptcp/pm_netlink.c` | MPTCP 路徑管理 |
| `tcp_metrics_nl` | `net/ipv4/tcp_metrics.c` | TCP 度量 |
| `ip_vs_genl` | `net/netfilter/ipvs/ip_vs_ctl.c` | IPVS 負載均衡 |
| `net_drop_monitor` | `net/core/drop_monitor.c` | 封包丟棄監控 |
| `netdev_nl` | `net/core/netdev-genl.c` | 網路裝置通用 |
| `tipc_genl` | `net/tipc/netlink.c` | TIPC 協定管理 |
| `nfc_genl` | `net/nfc/netlink.c` | NFC 管理 |
| `l2tp_nl` | `net/l2tp/l2tp_netlink.c` | L2TP 隧道 |
| `fou_nl` | `net/ipv4/fou_core.c` | Foo-over-UDP |

**控制器**（GENL_ID_CTRL）：`genetlink.c:1227`，提供 `CTRL_CMD_GETFAMILY` 等操作查詢已註冊家族。

## Semantics

### 鎖定策略

- **RTNL mutex**：RTNetlink 操作的全域鎖
- **genl_lock**（mutex）：Generic Netlink 序列化
- **callback lock**（rwsem）：平行操作支援
- **per-subsystem mutex**：Netfilter 各子系統獨立鎖

### 命名空間支援

- 網路命名空間隔離各協定家族的 socket 表
- `genl_info._net` 提供命名空間上下文
- 家族層級 `netnsok` 旗標控制命名空間感知

### 多播群組管理

`genetlink.c:71-93`：使用 bitmap 管理家族本地群組 ID 到全域群組 ID 的映射。

## Android-Specific Notes

### Binder Netlink 家族 [android]

**標頭：** `include/uapi/linux/android/binder_netlink.h`
**家族名稱：** `"binder"`（BINDER_FAMILY_NAME, line 10）
**版本：** 1

**命令：**
- `BINDER_CMD_REPORT`(1) — 報告 binder 錯誤

**屬性：**
- `BINDER_A_REPORT_ERROR`(1)、`BINDER_A_REPORT_CONTEXT`(2)
- `BINDER_A_REPORT_FROM_PID`(3)、`BINDER_A_REPORT_FROM_TID`(4)
- `BINDER_A_REPORT_TO_PID`(5)、`BINDER_A_REPORT_TO_TID`(6)
- `BINDER_A_REPORT_IS_REPLY`(7)、`BINDER_A_REPORT_FLAGS`(8)
- `BINDER_A_REPORT_CODE`(9)、`BINDER_A_REPORT_DATA_SIZE`(10)

**多播群組：** `"report"`（BINDER_MCGRP_REPORT, line 36）

用於 Binder IPC 的錯誤報告與交易除錯，使用者空間可訂閱多播群組接收錯誤事件。

## Cross-References

- [Networking](../subsystems/networking.md) — RTNetlink 與網路子系統的關係
- [Security](../subsystems/security.md) — NETLINK_SELINUX、NETLINK_AUDIT
- [BPF](../concepts/bpf.md) — BPF 程式的 Netlink 整合
- [Binder](../entities/binder.md) — Binder Netlink 家族的實現
- [ioctl 介面](ioctl-interfaces.md) — 另一種核心-使用者空間通訊機制
- [sysfs/procfs](sysfs-procfs.md) — 基於檔案系統的核心介面
