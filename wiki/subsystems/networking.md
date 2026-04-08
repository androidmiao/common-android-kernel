---
type: subsystem
kernel_path: "net/"
upstream: yes
android_patches: 2 (vendor hook 插入點)
vendor_hooks: ["include/trace/hooks/net.h"]
related:
  - ../concepts/bpf.md
  - ../concepts/vendor-hooks.md
  - ../concepts/interrupt-handling.md
  - ../concepts/locking-primitives.md
  - ../concepts/rcu.md
  - ../concepts/tracing-and-ftrace.md
  - ../concepts/module-system.md
  - ../concepts/gki.md
last_updated: 2026-04-08
---

# Networking 子系統深度分析

> **核心路徑**: `net/`
> **核心版本**: Linux 6.19-rc8 (Android Common Kernel)
> **原始碼規模**: ~1,249,652 行 (.c)，1,534 個 .c 檔案，280 個 .h 檔案
> **協定堆疊**: Socket 層 → 傳輸層 (TCP/UDP/SCTP) → 網路層 (IPv4/IPv6) → 鏈路層 (Ethernet/WiFi/BT)

---

## 目錄

1. [架構總覽](#1-架構總覽)
2. [目錄地圖](#2-目錄地圖)
3. [核心資料結構](#3-核心資料結構)
4. [Socket 層](#4-socket-層)
5. [TCP/IP 協定堆疊](#5-tcpip-協定堆疊)
6. [封包接收路徑](#6-封包接收路徑)
7. [封包傳送路徑](#7-封包傳送路徑)
8. [Netfilter 框架](#8-netfilter-框架)
9. [Traffic Control (QoS)](#9-traffic-control-qos)
10. [BPF / XDP 整合](#10-bpf--xdp-整合)
11. [無線網路 (WiFi)](#11-無線網路-wifi)
12. [藍牙](#12-藍牙)
13. [Network Namespace](#13-network-namespace)
14. [Android 特定變更](#14-android-特定變更)
15. [Vendor Hooks](#15-vendor-hooks)
16. [QRTR — Qualcomm IPC Router](#16-qrtr--qualcomm-ipc-router)
17. [關鍵效能機制](#17-關鍵效能機制)
18. [配置選項 (Kconfig)](#18-配置選項-kconfig)
19. [Syscall 介面](#19-syscall-介面)
20. [檔案索引](#20-檔案索引)

---

## 1. 架構總覽

Linux 網路子系統採用**分層協定架構**，遵循 BSD socket 介面模型。每一層透過明確定義的介面與上下層互動，從使用者空間的 socket syscall 到硬體 NIC 驅動程式，形成完整的封包處理管線。

```
┌─────────────────────────────────────────────────────┐
│                  使用者空間應用程式                      │
│            socket() / bind() / connect()              │
├─────────────────────────────────────────────────────┤
│  Socket 層 (net/socket.c)                             │
│  struct socket → struct sock                          │
├─────────────────────────────────────────────────────┤
│  傳輸層 (Transport Layer)                              │
│  TCP (net/ipv4/tcp*.c) │ UDP (net/ipv4/udp.c)        │
│  SCTP (net/sctp/)      │ MPTCP (net/mptcp/)          │
├─────────────────────────────────────────────────────┤
│  網路層 (Network Layer)                                │
│  IPv4 (net/ipv4/)      │ IPv6 (net/ipv6/)            │
│  路由 (FIB)            │ Netfilter hooks              │
├─────────────────────────────────────────────────────┤
│  鏈路層 / 裝置層                                       │
│  net/core/dev.c (13,299 行)                           │
│  NAPI │ GRO │ RPS/RFS │ XDP │ TC (qdisc)             │
├─────────────────────────────────────────────────────┤
│  NIC 驅動程式 (drivers/net/)                           │
│  硬體 → DMA → NAPI poll → netif_receive_skb()        │
└─────────────────────────────────────────────────────┘
```

核心設計原則：

- **sk_buff 為中心**：所有封包以 `struct sk_buff` 表示，貫穿整個堆疊
- **協定獨立的裝置層**：`net/core/dev.c` 處理所有協定的封包收發
- **NAPI 輪詢模式**：高速網路使用中斷 + 輪詢混合模式，避免中斷風暴
- **Netfilter 掛勾點**：在關鍵路徑插入 5 個掛勾點 (PREROUTING, INPUT, FORWARD, OUTPUT, POSTROUTING)
- **Network Namespace 隔離**：每個命名空間擁有獨立的路由表、介面、防火牆規則

---

## 2. 目錄地圖

| 目錄 | .c 行數 | 說明 |
|------|---------|------|
| `net/core/` | 87,613 | 核心框架：裝置管理、sk_buff、socket、BPF filter、GRO、NAPI |
| `net/ipv4/` | 107,271 | IPv4 協定堆疊：TCP、UDP、IP 路由、ICMP、ARP、擁塞控制 |
| `net/ipv6/` | 70,066 | IPv6 協定堆疊：路由、地址配置 (SLAAC)、ICMPv6、MLD |
| `net/netfilter/` | 98,274 | 封包過濾框架：nftables、conntrack、NAT、iptables 相容層 |
| `net/mac80211/` | 77,402 | IEEE 802.11 MAC 層：訊框處理、聚合、速率控制 |
| `net/bluetooth/` | 61,703 | 藍牙協定堆疊：HCI、L2CAP、SCO、BLE |
| `net/sched/` | 62,654 | 流量控制：qdisc、TC classifier、traffic shaping |
| `net/wireless/` | 45,980 | cfg80211 無線管理：法規域、nl80211 netlink 介面 |
| `net/bridge/` | 37,269 | 橋接網路：MAC 學習、VLAN、STP、多播 |
| `net/mptcp/` | 18,142 | Multipath TCP：多路徑傳輸、WiFi/行動網路切換 |
| `net/xfrm/` | ~18,000 | IPsec 框架：SA/SP 管理、ESP/AH 協定 |
| `net/nfc/` | 19,389 | NFC 協定堆疊：LLCP、NCI |
| `net/qrtr/` | 2,612 | Qualcomm IPC Router：數據機/感測器通訊 [android] |
| `net/packet/` | 4,814 | AF_PACKET：原始封包存取 (tcpdump) |
| `net/unix/` | ~5,000 | Unix domain socket：AF_UNIX 本地 IPC |
| `net/tls/` | ~5,000 | Kernel TLS：核心層加解密卸載 |
| `net/xdp/` | ~2,500 | XDP socket：高效能封包處理使用者空間介面 |
| `net/socket.c` | ~3,800 | Socket syscall 入口：socket/bind/connect/listen/accept |
| `net/sysctl_net.c` | ~100 | 網路 sysctl 根節點 |

---

## 3. 核心資料結構

### 3.1 `struct sk_buff` — 封包緩衝區

> 定義於 `include/linux/skbuff.h:885`

Linux 網路堆疊中最關鍵的資料結構，每個封包（無論收發）都以 sk_buff 表示。

**關鍵欄位：**

| 欄位 | 行號 | 說明 |
|------|------|------|
| `next`, `prev` | :889-890 | 鏈結串列指標（必須位於結構開頭） |
| `dev` | :893 | 關聯的 `struct net_device` |
| `sk` | :906 | 關聯的 `struct sock`（反向指標） |
| `cb[48]` | :918 | 控制緩衝區，每層可存放私有資料（48 bytes） |
| `len`, `data_len` | :934-935 | 封包長度（線性 + 分片） |
| `protocol` | :1080 | L3 協定類型 (`__be16`) |
| `transport_header` | :1081 | L4 標頭偏移量 |
| `network_header` | :1082 | L3 標頭偏移量 |
| `mac_header` | :1083 | L2 標頭偏移量 |
| `head`, `data` | :1094-1095 | 緩衝區起始 / 有效資料起始 |
| `tail`, `end` | :1092-1093 | 有效資料結尾 / 緩衝區結尾 |
| `truesize` | :1096 | 實際佔用記憶體大小 |
| `users` | :1097 | 參考計數 |

**記憶體布局：**

```
head ──────► [headroom]
data ──────► [packet data (linear)]
tail ──────► [tailroom]
end  ──────► [skb_shared_info: frags[], frag_list]
```

**關鍵操作** (`net/core/skbuff.c`, 7,422 行)：

| 函式 | 行號 | 說明 |
|------|------|------|
| `__alloc_skb()` | :648 | 核心 sk_buff 分配，支援 NAPI 快取、fclone |
| `__netdev_alloc_skb()` | :738 | 裝置上下文分配（RX 導向） |
| `napi_alloc_skb()` | :815 | NAPI 上下文快速分配，整合 page pool |
| `skb_clone()` | :2069 | 淺複製（共享資料緩衝區） |
| `skb_copy()` | :2149 | 深複製（完整資料複製） |
| `__kfree_skb()` | :1194 | 核心釋放，清理 fraglist |

### 3.2 `struct net_device` — 網路裝置

> 定義於 `include/linux/netdevice.h:2105`

代表一個網路介面（eth0、wlan0 等），是裝置驅動程式與協定堆疊之間的橋樑。

**關鍵欄位：**

| 欄位 | 行號 | 說明 |
|------|------|------|
| `name[IFNAMSIZ]` | :2180 | 裝置名稱 |
| `netdev_ops` | :2118 | 網路裝置操作函式表 (`struct net_device_ops`) |
| `header_ops` | :2119 | 標頭操作 |
| `_tx` | :2120 | 傳送佇列陣列 (`struct netdev_queue *`) |
| `_rx` | :2166 | 接收佇列陣列 (`struct netdev_rx_queue *`) |
| `mtu` | :2132 | 最大傳輸單元 |
| `flags` | :2154 | 裝置旗標 (IFF_UP, IFF_RUNNING 等) |
| `features` | :2156 | 裝置特性 (checksum offload, GSO, GRO 等) |
| `ifindex` | :2164 | 介面索引 |
| `rx_handler` | :2169 | 自訂接收處理器（bridge、bonding 使用） |

### 3.3 `struct sock` — 協定層 Socket

> 定義於 `include/net/sock.h:359`

協定層的核心結構，每個開啟的 socket 對應一個 `struct sock`。包含連線狀態、記憶體管理、緩衝區佇列等。

**關鍵欄位：**

| 欄位 | 行號 | 說明 |
|------|------|------|
| `__sk_common` | :364 | 內嵌 `sock_common`：地址、埠、協定族、狀態 |
| `sk_receive_queue` | :405 | 接收封包佇列 |
| `sk_backlog` | :417-418 | 背壓佇列（head/tail） |
| `sk_rcvbuf` | :437 | 接收緩衝區大小 |
| `sk_filter` | :439 | BPF socket 過濾器 |
| `sk_socket` | :454 | 指向 `struct socket`（VFS 層） |
| `sk_write_queue` | :485 | 傳送封包佇列 |
| `sk_pacing_rate` | :503 | 封包發送速率 (bytes/sec) |
| `sk_priority` | :504 | QoS 優先級 |
| `sk_mark` | :505 | 封包標記（netfilter 使用） |
| `sk_uid` | :506 | 擁有者 UID |
| `sk_protocol` | :507 | 協定編號 |

**結構繼承關係：**

```
struct socket (VFS 層, inode 綁定)
  └─→ sk → struct sock (協定層)
           └─→ struct inet_sock (AF_INET 擴展)
                └─→ struct inet_connection_sock (連線導向)
                     └─→ struct tcp_sock (TCP 專屬, tcp.h:200)
```

---

## 4. Socket 層

> `net/socket.c` (~3,800 行)

Socket 層是使用者空間與核心網路堆疊之間的介面，實作 BSD socket API。

**Syscall 入口點：**

| Syscall | 函式 | 行號 | 說明 |
|---------|------|------|------|
| `socket(2)` | `SYSCALL_DEFINE3(socket)` | socket.c:1759 | 建立 socket |
| `bind(2)` | `SYSCALL_DEFINE3(bind)` | socket.c:1910 | 繫結地址 |
| `connect(2)` | `SYSCALL_DEFINE3(connect)` | socket.c:2114 | 發起連線 |
| `listen(2)` | `SYSCALL_DEFINE2(listen)` | socket.c:1948 | 監聽連線 |
| `accept4(2)` | `SYSCALL_DEFINE4(accept4)` | socket.c:2051 | 接受連線 |
| `sendto(2)` | `SYSCALL_DEFINE6(sendto)` | socket.c:2213 | 傳送資料 |
| `recvfrom(2)` | `SYSCALL_DEFINE6(recvfrom)` | socket.c:2271 | 接收資料 |

**Socket 建立流程：**

```
socket(AF_INET, SOCK_STREAM, 0)
  → __sys_socket() → __sys_socket_create()
    → sock_create() → __sock_create()
      → inet_create()  [af_inet.c:254]
        1. 在 inetsw[] 表查找 (SOCK_STREAM → TCP)
        2. sk_alloc() 分配 struct sock
        3. tcp_v4_init_sock() 初始化 TCP 狀態
    → sock_map_fd()  將 socket 映射到 file descriptor
```

---

## 5. TCP/IP 協定堆疊

### 5.1 IPv4 協定初始化

> `net/ipv4/af_inet.c:1891` — `inet_init()`

```
inet_init()
  1. proto_register(&tcp_prot)     — 註冊 TCP 協定  [:1901]
  2. proto_register(&udp_prot)     — 註冊 UDP 協定  [:1903]
  3. proto_register(&raw_prot)     — 註冊 RAW socket [:1906]
  4. proto_register(&ping_prot)    — 註冊 ICMP ECHO  [:1909]
  5. sock_register(&inet_family_ops) — 註冊 AF_INET
  6. inet_register_protosw()       — 填充 inetsw[] 表 [:1955-1960]
  7. arp_init()、ip_init()、tcp_init()、udp_init() — 子系統初始化
```

### 5.2 TCP 關鍵函式

| 函式 | 檔案:行號 | 說明 |
|------|-----------|------|
| `tcp_v4_connect()` | tcp_ipv4.c:224 | IPv4 TCP 連線建立 |
| `tcp_sendmsg()` | tcp.c:1407 | TCP 資料發送入口 |
| `tcp_sendmsg_locked()` | tcp.c:1077 | 鎖定後的發送路徑 |
| `tcp_recvmsg()` | tcp.c:2912 | TCP 資料接收入口 |
| `tcp_transmit_skb()` | tcp_output.c:1646 | 低層封包傳送 |
| `tcp_write_xmit()` | tcp_output.c:66 | 寫入封包到線路 |
| `tcp_rcv_established()` | tcp_input.c:6285 | 已建立連線的封包接收 |
| `tcp_rcv_state_process()` | tcp_input.c:6936 | TCP 狀態機封包處理 |

### 5.3 IP 層關鍵函式

| 函式 | 檔案:行號 | 說明 |
|------|-----------|------|
| `ip_rcv()` | ip_input.c:564 | IPv4 接收入口 |
| `ip_local_deliver()` | ip_input.c:250 | 本地遞送（路由後） |
| `ip_output()` | ip_output.c:428 | IPv4 輸出（路由後） |
| `ip_queue_xmit()` | ip_output.c:546 | 從 L4 佇列封包傳送 |
| `ipv6_rcv()` | ip6_input.c:304 | IPv6 接收入口 |
| `ip6_output()` | ip6_output.c:227 | IPv6 輸出 |

---

## 6. 封包接收路徑

完整的封包接收路徑，從硬體中斷到應用程式讀取：

```
NIC 硬體 → DMA 寫入 ring buffer
  → IRQ → 驅動 ISR → napi_schedule()
    → NAPI poll 回調
      → napi_alloc_skb()                        [skbuff.c:815]
      → netif_receive_skb()                      [dev.c:6411]
        → __netif_receive_skb()
          → RPS 判定 → 可能跨 CPU 投遞           [dev.c:5064]
          → __netif_receive_skb_core()           [dev.c:5929]
            → trace_android_vh_ptype_head()      [dev.c:593, Android hook]
            → packet_type 分發 (AF_PACKET 抓包)
            → rx_handler (bridge/bonding)
            → 協定分發 → ip_rcv() / ipv6_rcv()
              → NF_INET_PRE_ROUTING (Netfilter)
              → 路由查找
                → 本地: ip_local_deliver()       [ip_input.c:250]
                  → NF_INET_LOCAL_IN
                  → 協定 demux (TCP/UDP/ICMP)
                    → tcp_rcv_established()      [tcp_input.c:6285]
                    → sk_receive_queue 入隊
                      → sock_def_readable()      [sock.c:~3610]
                        → trace_android_vh_do_wake_up_sync() [Android hook]
                        → 喚醒等待中的應用程式
                → 轉發: ip_forward()
                  → NF_INET_FORWARD → ip_output()
```

**關鍵效能路徑：**

- **NAPI** (`__napi_poll()` @ dev.c:7674)：混合中斷/輪詢模式，單次 poll 處理最多 `weight`（預設 64）個封包
- **GRO** (Generic Receive Offload, `gro.c`)：在軟體層合併多個封包為一個大封包，減少上層處理次數
- **RPS** (Receive Packet Steering, `get_rps_cpu()` @ dev.c:5064)：軟體層多佇列，將封包分散到多個 CPU

---

## 7. 封包傳送路徑

```
應用程式 → sendto(2) / sendmsg(2)
  → tcp_sendmsg()                               [tcp.c:1407]
    → tcp_sendmsg_locked()                       [tcp.c:1077]
      → skb_page_frag_refill() → 複製資料到 sk_buff
      → tcp_push() → tcp_write_xmit()           [tcp_output.c:66]
        → tcp_transmit_skb()                     [tcp_output.c:1646]
          → 建構 TCP header、計算 checksum
          → ip_queue_xmit()                      [ip_output.c:546]
            → 路由查找
            → NF_INET_LOCAL_OUT (Netfilter)
            → ip_output()                        [ip_output.c:428]
              → NF_INET_POST_ROUTING
              → dev_queue_xmit()
                → __dev_queue_xmit()             [dev.c:4746]
                  → TC/qdisc 入隊
                  → XPS (Transmit Packet Steering) CPU 選擇
                  → 驅動 ndo_start_xmit()
                  → 硬體 DMA 傳送
```

---

## 8. Netfilter 框架

> `net/netfilter/` (98,274 行，255 個 .c 檔案)

Netfilter 是 Linux 封包過濾框架，提供防火牆、NAT、封包修改等功能。

### 5 個掛勾點 (Hook Points)

```
              ┌──────────────────────────┐
              │       PREROUTING         │ ← 封包進入
              └──────┬───────────────────┘
                     │ 路由判定
              ┌──────┴───────┐
              │              │
         本地遞送        轉發
              │              │
      ┌───────┴──┐   ┌──────┴──────┐
      │  INPUT   │   │  FORWARD    │
      └───────┬──┘   └──────┬──────┘
              │              │
         本地處理         ┌──┴──────────┐
              │          │  POSTROUTING  │ → 封包離開
      ┌───────┴──┐      └──────────────┘
      │  OUTPUT  │
      └───────┬──┘
              │
      ┌───────┴──────────┐
      │   POSTROUTING    │ → 封包離開
      └──────────────────┘
```

**核心函式：**

| 函式 | 檔案:行號 | 說明 |
|------|-----------|------|
| `nf_hook_slow()` | netfilter/core.c:616 | Netfilter 掛勾慢路徑遍歷 |
| `nf_register_net_hook()` | netfilter/core.c:554 | 註冊 per-netns 掛勾 |

**主要子系統：**

- **nf_tables** (`nf_tables_api.c`, 12,334 行)：現代 nftables 規則引擎
- **conntrack**：連線追蹤，NAT 和有狀態防火牆的基礎
- **iptables 相容層**：`net/ipv4/netfilter/`、`net/ipv6/netfilter/`

---

## 9. Traffic Control (QoS)

> `net/sched/` (62,654 行)

流量控制子系統實作佇列規則 (qdisc)、分類器 (classifier)、和動作 (action)。

**主要組件：**

- **qdisc** (Queue Discipline)：控制封包排程策略（FIFO、HTB、FQ、CAKE 等）
- **TC classifier**：封包分類（u32、flower、BPF）
- **TC action**：封包動作（mirror、redirect、NAT）
- **TCX**：TC extended，基於 BPF 的新式 TC 程式

---

## 10. BPF / XDP 整合

### 10.1 Socket Filter / BPF

> `net/core/filter.c` (12,580 行)

BPF 在網路堆疊中的整合極為深入，提供可程式化的封包處理能力。

**核心函式：**

| 函式 | 行號 | 說明 |
|------|------|------|
| `sk_filter_trim_cap()` | filter.c:134 | 執行 socket BPF 過濾器 |
| `sk_attach_filter()` | filter.c:1554 | 附加 BPF 程式到 socket |
| `xdp_do_generic_redirect()` | filter.c:4587 | Generic XDP 重定向 |

**BPF 網路程式類型：** Socket Filter、TC、XDP、cgroup/socket、流量分類、socket map 等。

### 10.2 XDP (eXpress Data Path)

> `net/xdp/` (~2,500 行) + `net/core/xdp.c` (1,055 行)

XDP 在驅動程式層執行 BPF 程式，提供最高效能的封包處理路徑，動作包括：

- `XDP_PASS`：正常進入協定堆疊
- `XDP_DROP`：丟棄封包（最快的防火牆）
- `XDP_TX`：原路返回
- `XDP_REDIRECT`：重定向到其他介面或 AF_XDP socket

**Generic XDP**：`do_xdp_generic()` @ dev.c:5606，在不支援原生 XDP 的驅動上以軟體模擬。

### 10.3 Android 的 BPF 網路應用

Android 透過 cgroup BPF 實現流量控制：

- **eBPF traffic controller**：使用 `BPF_PROG_TYPE_CGROUP_SKB` 對 per-app 流量進行標記和計費
- **UID-based 流量統計**：利用 `sk_uid` 欄位追蹤每個應用的網路使用量
- **Network policy enforcement**：BPF 程式在 cgroup 層級強制執行網路策略

---

## 11. 無線網路 (WiFi)

### cfg80211 — 無線管理

> `net/wireless/` (45,980 行，38 個 .c 檔案)

- **nl80211** (`nl80211.c`, 21,911 行)：最大的單一原始碼檔案，實作完整的 netlink 無線管理介面
- 法規域管理、掃描、連線、AP 模式、P2P 等

### mac80211 — MAC 層

> `net/mac80211/` (77,402 行，90 個 .c 檔案)

- IEEE 802.11 MAC 層軟體實作
- 訊框處理、A-MPDU/A-MSDU 聚合
- 速率控制演算法
- 電源管理、漫遊

---

## 12. 藍牙

> `net/bluetooth/` (61,703 行，52 個 .c 檔案)

**主要組件：**

| 組件 | 說明 |
|------|------|
| HCI (`hci_*.c`) | Host Controller Interface，與藍牙控制器通訊 |
| L2CAP (`l2cap_core.c`, 7,721 行) | 邏輯鏈路控制和適配協定 |
| SCO | 同步連線導向（語音） |
| BNEP | 藍牙網路封裝協定 |
| RFCOMM | 串列埠模擬 |
| BLE (低功耗藍牙) | GATT、SMP、廣播 |

藍牙堆疊使用標準 Linux 實作，透過 `AF_BLUETOOTH` socket family 與使用者空間通訊。

---

## 13. Network Namespace

> `net/core/net_namespace.c` (~1,549 行)

Network namespace 為每個命名空間提供獨立的網路環境：

- 獨立的網路介面列表
- 獨立的路由表 (FIB)
- 獨立的 Netfilter 規則
- 獨立的 socket 和埠空間

**Android 應用：** 容器化、per-app 網路隔離、VPN 實作。

---

## 14. Android 特定變更

Android Common Kernel 在網路子系統的修改**極為精簡**，僅有 2 個 vendor hook 插入點，體現了 GKI 最小侵入的設計原則。

### 修改策略

| 策略 | 說明 |
|------|------|
| **Vendor Hooks** | 2 個 hook（dev.c、sock.c），允許廠商模組擴展 |
| **BPF 擴展** | 透過 eBPF 實現流量控制，無需核心修改 |
| **QRTR** | 上游已合併的 Qualcomm IPC 協定 |
| **標準堆疊** | WiFi、藍牙、IPv4/IPv6、Netfilter 全部使用上游實作 |

**與其他子系統的比較：** 排程器有 78 個 vendor hooks，記憶體管理有 2 個。網路子系統僅有 2 個，顯示 Android 主要依賴 BPF 和使用者空間工具（如 netd、iptables/nftables）來實現網路管理。

---

## 15. Vendor Hooks

> 定義於 `include/trace/hooks/net.h`

### Hook 1: `android_vh_ptype_head`

- **位置**：`net/core/dev.c:593`（`ptype_head()` 函式內）
- **原型**：`TP_PROTO(const struct packet_type *pt, struct list_head *vendor_pt)`
- **用途**：允許廠商模組在封包類型查找時注入自訂的 packet_type 處理器
- **場景**：接收路徑早期，用於協定分發的自訂擴展

### Hook 2: `android_vh_do_wake_up_sync`

- **位置**：`net/core/sock.c:3615`（`sock_def_readable()` 函式內）
- **原型**：`TP_PROTO(struct wait_queue_head *wq_head, int *done, struct sock *sk)`
- **定義於**：`include/trace/hooks/sched.h:302`（跨子系統 hook）
- **用途**：允許廠商自訂 socket 喚醒行為，可透過 `done` 旗標跳過預設喚醒
- **場景**：可用於最佳化特定應用的 socket 喚醒延遲

---

## 16. QRTR — Qualcomm IPC Router

> `net/qrtr/` (2,612 行，6 個檔案) [upstream]

QRTR 是 Qualcomm 開發並已合併到上游的 IPC 協定，用於與硬體處理器（數據機、DSP、感測器 hub）通訊。

**檔案結構：**

| 檔案 | 行數 | 說明 |
|------|------|------|
| `af_qrtr.c` | 1,328 | 核心 socket 家族實作 (AF_QIPCRTR) |
| `ns.c` | 778 | 命名空間/服務查找 |
| `mhi.c` | 183 | MHI 傳輸後端（外部數據機） |
| `tun.c` | 176 | TUN 裝置（使用者空間端點） |
| `smd.c` | 111 | SMD/rpmsg 傳輸（晶片內通訊） |

**傳輸層 (Kconfig)：**

- `CONFIG_QRTR_SMD`：SMD/rpmsg 通道（最常見，depends on RPMSG）
- `CONFIG_QRTR_MHI`：MHI 通道（外部數據機，depends on MHI_BUS）
- `CONFIG_QRTR_TUN`：TUN 裝置（使用者空間端點、測試）

**Android 重要性：** QRTR 是 Android 裝置與 Qualcomm 數據機通訊的基礎，影響行動數據、通話、GPS 等功能。

---

## 17. 關鍵效能機制

### 17.1 NAPI (New API)

- **入口**：`__napi_poll()` @ dev.c:7674
- **機制**：中斷觸發首次 poll，之後切換為輪詢模式，單次處理最多 `weight` 個封包
- **優勢**：高流量時避免中斷風暴，低流量時回到中斷模式節省 CPU

### 17.2 GRO (Generic Receive Offload)

- **入口**：`gro_receive_skb()` @ gro.c:624
- **機制**：在軟體層合併多個小封包為一個大封包
- **合併**：`skb_gro_receive()` @ gro.c:92
- **完成**：`gro_complete()` @ gro.c:254

### 17.3 RPS / RFS (Receive Packet / Flow Steering)

- **RPS** (`get_rps_cpu()` @ dev.c:5064)：軟體層將接收封包分散到多個 CPU
- **RFS**：進一步將封包導向處理該 flow 的 CPU，改善快取局部性
- **配置**：`CONFIG_RPS`（預設 y）、`CONFIG_RFS_ACCEL`（預設 y）

### 17.4 XPS (Transmit Packet Steering)

- **配置**：`CONFIG_XPS`（預設 y）
- **機制**：根據 CPU 或 RX queue 選擇 TX queue，減少鎖競爭

### 17.5 BQL (Byte Queue Limits)

- **配置**：`CONFIG_BQL`（預設 y）
- **機制**：動態限制裝置佇列中的待發位元組數，減少緩衝膨脹 (bufferbloat)

### 17.6 Page Pool

- **核心**：`net/core/page_pool.c` (1,354 行)
- **機制**：預分配和回收頁面，避免頻繁的頁面分配器呼叫
- **整合**：NAPI 接收路徑透過 page pool 分配 sk_buff 資料緩衝區

---

## 18. 配置選項 (Kconfig)

> `net/Kconfig`

### 核心選項

| 選項 | 預設 | 說明 |
|------|------|------|
| `CONFIG_NET` | y | 網路支援總開關 |
| `CONFIG_INET` | y | TCP/IP 協定堆疊 |
| `CONFIG_NETFILTER` | y | 封包過濾框架 |
| `CONFIG_WIRELESS` | y | 無線網路支援 |

### 效能選項

| 選項 | 預設 | 說明 |
|------|------|------|
| `CONFIG_RPS` | y | 接收封包導向 |
| `CONFIG_RFS_ACCEL` | y | RFS 硬體加速 |
| `CONFIG_XPS` | y | 傳送封包導向 |
| `CONFIG_BQL` | y | Byte Queue Limits |
| `CONFIG_NET_RX_BUSY_POLL` | y | 接收 busy poll 低延遲 |
| `CONFIG_NET_FLOW_LIMIT` | y | 流量限制（防 DoS） |
| `CONFIG_MAX_SKB_FRAGS` | 17 | sk_buff 最大分片數 (17-45) |
| `CONFIG_PCPU_DEV_REFCNT` | y | 裝置參考計數使用 per-cpu 變數 |

### 進階功能

| 選項 | 預設 | 說明 |
|------|------|------|
| `CONFIG_BPF_STREAM_PARSER` | n | BPF 串流解析器 (SOCKMAP) |
| `CONFIG_LWTUNNEL` | n | 輕量級隧道 (MPLS 等) |
| `CONFIG_LWTUNNEL_BPF` | y (if LWTUNNEL) | BPF 路由動作 |
| `CONFIG_PAGE_POOL` | bool | 頁面池（NIC 驅動使用） |
| `CONFIG_ETHTOOL_NETLINK` | y | Netlink ethtool 介面 |
| `CONFIG_NET_SHAPER` | n | 網路整形器 |

### Android 相關

| 選項 | 說明 |
|------|------|
| `CONFIG_QRTR` | Qualcomm IPC Router |
| `CONFIG_QRTR_SMD` | SMD 傳輸（數據機通訊） |
| `CONFIG_QRTR_MHI` | MHI 傳輸（外部數據機） |
| `CONFIG_BT` | 藍牙堆疊 |
| `CONFIG_MAC80211` | WiFi MAC 層 |
| `CONFIG_NFC` | NFC 協定堆疊 |
| `CONFIG_MPTCP` | Multipath TCP（WiFi/行動網路切換） |

---

## 19. Syscall 介面

網路子系統透過以下 syscall 家族與使用者空間互動：

**Socket 操作：** `socket`, `socketpair`, `bind`, `listen`, `accept`, `accept4`, `connect`, `shutdown`, `close`

**資料傳輸：** `sendto`, `recvfrom`, `sendmsg`, `recvmsg`, `sendmmsg`, `recvmmsg`, `read`, `write`, `writev`, `readv`

**Socket 選項：** `getsockopt`, `setsockopt`

**位址/介面查詢：** `getpeername`, `getsockname`, `ioctl` (SIOCGIFCONF 等)

**多工：** `select`, `poll`, `epoll_ctl`, `epoll_wait` (透過 VFS 整合)

**Netlink 介面：** 透過 `AF_NETLINK` socket 提供路由管理 (NETLINK_ROUTE)、netfilter 管理 (NETLINK_NETFILTER)、通用 netlink (NETLINK_GENERIC) 等。

---

## 20. 檔案索引

### 核心框架 (`net/core/`, 87,613 行)

| 檔案 | 行數 | 說明 |
|------|------|------|
| `dev.c` | 13,299 | **協定獨立裝置層**：NAPI、RPS/RFS、封包收發核心路徑 |
| `filter.c` | 12,580 | **BPF/eBPF 整合**：socket filter、XDP、TC BPF |
| `skbuff.c` | 7,422 | **sk_buff 管理**：分配、複製、釋放 |
| `rtnetlink.c` | 7,103 | Routing netlink 介面 |
| `sock.c` | 4,561 | **Socket 核心**：分配、初始化、選項 |
| `pktgen.c` | 4,133 | 封包產生器（測試用） |
| `neighbour.c` | 3,932 | 鄰居發現 (ARP/NDP) |
| `net-sysfs.c` | 2,418 | 網路裝置 sysfs 介面 |
| `flow_dissector.c` | 2,101 | 封包 flow 解析 |
| `sock_map.c` | 1,959 | BPF socket map |
| `drop_monitor.c` | 1,789 | 封包丟棄監控 |
| `net_namespace.c` | 1,549 | 網路命名空間管理 |
| `fib_rules.c` | 1,478 | FIB 策略路由規則 |
| `page_pool.c` | 1,354 | 頁面池（高效能 RX 緩衝區） |
| `skmsg.c` | 1,289 | BPF socket 訊息 |
| `netdev-genl.c` | 1,203 | Netdev generic netlink |
| `xdp.c` | 1,055 | XDP 核心框架 |
| `gro.c` | ~700 | Generic Receive Offload |

### TCP/IP 堆疊 (`net/ipv4/`, 107,271 行)

| 檔案 | 行數 | 說明 |
|------|------|------|
| `tcp_input.c` | 7,580 | TCP 接收處理、狀態機 |
| `tcp.c` | 5,300 | TCP socket 操作 (send/recv) |
| `tcp_output.c` | 4,623 | TCP 傳送處理 |
| `ip_input.c` | ~600 | IP 接收入口 |
| `ip_output.c` | ~1,800 | IP 傳送處理 |
| `route.c` | ~3,800 | IPv4 路由 (FIB) |
| `udp.c` | ~3,500 | UDP 協定 |
| `af_inet.c` | ~2,000 | AF_INET socket 家族 |

---

## 交叉參考

- [BPF 子系統](../concepts/bpf.md) — 33 種程式類型中多種用於網路（socket filter、XDP、TC、cgroup/skb）
- [Vendor Hooks](../concepts/vendor-hooks.md) — 網路子系統 2 個 hook 的框架基礎
- [中斷處理](../concepts/interrupt-handling.md) — NAPI 中斷/輪詢混合模式的基礎
- [鎖定原語](../concepts/locking-primitives.md) — socket lock、per-CPU 變數、RCU 在網路中的應用
- [RCU](../concepts/rcu.md) — 路由表、socket 查找表等大量使用 RCU 保護
- [追蹤與 ftrace](../concepts/tracing-and-ftrace.md) — 網路 tracepoint、vendor hooks 的基礎設施
- [Module 系統](../concepts/module-system.md) — WiFi、藍牙、NFC 等作為可載入模組
- [GKI](../concepts/gki.md) — 網路子系統遵循 GKI 最小侵入原則
