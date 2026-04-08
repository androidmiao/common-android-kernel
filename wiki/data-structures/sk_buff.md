---
type: data-structure
defined_in: common/include/linux/skbuff.h:885
size_approx: 232 bytes (core structure)
lifecycle: allocated via __alloc_skb()/napi_alloc_skb(), freed via __kfree_skb()/kfree_skb()
related: [net_device, sock, skb_shared_info, NAPI, GRO, Netfilter]
last_updated: 2026-04-09
---

# `struct sk_buff`

## Purpose

`struct sk_buff` 是 Linux 網路堆疊中最關鍵的資料結構，代表單個網路封包（packet）的記憶體容器及中繼資料。無論是接收、傳送、路由或過濾，所有網路操作都以 sk_buff 為中心展開。每個封包從 NIC 接收到應用層送出的整個生命週期，都被封裝在 sk_buff 中。

本結構是核心網路堆疊的基礎，隨著網路功能（如 GSO/GRO、Netfilter、TC、XDP、BPF）的發展而不斷演化，目前包含超過 80 個欄位，支援複雜的分片、複製、克隆等語意。

## Definition

```c
struct sk_buff {
	union {
		struct {
			/* 必須位於結構開頭，與 sk_buff_head 相容 */
			struct sk_buff		*next;
			struct sk_buff		*prev;
			
			union {
				struct net_device	*dev;
				unsigned long		dev_scratch;
			};
		};
		struct rb_node		rbnode;      /* netem/ip4 defrag/tcp 使用 */
		struct list_head	list;
		struct llist_node	ll_node;
	};

	struct sock		*sk;              /* 關聯的 socket */

	union {
		ktime_t		tstamp;           /* 時間戳記 */
		u64		skb_mstamp_ns;    /* GSO：最早出發時間 */
	};

	char			cb[48] __aligned(8);  /* 控制緩衝區，各層可用 */

	union {
		struct {
			unsigned long	_skb_refdst;
			void		(*destructor)(struct sk_buff *skb);
		};
		struct list_head	tcp_tsorted_anchor;
#ifdef CONFIG_NET_SOCK_MSG
		unsigned long		_sk_redir;
#endif
	};

#if defined(CONFIG_NF_CONNTRACK) || defined(CONFIG_NF_CONNTRACK_MODULE)
	unsigned long		 _nfct;            /* Netfilter conntrack */
#endif

	unsigned int		len,              /* 完整封包長度 */
				data_len;         /* 分片資料長度 */
	__u16			mac_len,          /* L2 標頭長度 */
				hdr_len;          /* 標頭長度（克隆用） */

	__u16			queue_mapping;    /* 裝置傳送佇列映射 */

	__u8			__cloned_offset[0];
	__u8			cloned:1,         /* 是否為克隆 */
				nohdr:1,          /* 無標頭克隆 */
				fclone:2,         /* fast clone 狀態 */
				peeked:1,         /* 已窺探 */
				head_frag:1,      /* head 使用頁面片段 */
				pfmemalloc:1,     /* 來自應急池 */
				pp_recycle:1;     /* page pool 回收指示 */
#ifdef CONFIG_SKB_EXTENSIONS
	__u8			active_extensions;  /* 活躍擴展 */
#endif

	struct_group(headers,               /* 單一 memcpy 複製的欄位組 */

	__u8			__pkt_type_offset[0];
	__u8			pkt_type:3,       /* 封包類型 */
				ignore_df:1,      /* 忽略 DF 旗標 */
				dst_pending_confirm:1,  /* 目的確認待定 */
				ip_summed:2,      /* 校驗和狀態 */
				ooo_okay:1;       /* 亂序 OK */

	__u8			__mono_tc_offset[0];
	__u8			tstamp_type:2,    /* 時間戳記類型 */
#ifdef CONFIG_NET_XGRESS
				tc_at_ingress:1,  /* 入口 TC 處理 */
				tc_skip_classify:1,   /* 跳過分類 */
#endif
				remcsum_offload:1,    /* 遠端校驗和卸載 */
				csum_complete_sw:1,   /* 軟體完整校驗和 */
				csum_level:2,     /* 校驗和層級 */
				inner_protocol_type:1;  /* 內部協定類型 */

	__u8			l4_hash:1,        /* L4 雜湊有效 */
				sw_hash:1,        /* 軟體雜湊 */
#ifdef CONFIG_WIRELESS
				wifi_acked_valid:1,   /* WiFi ACK 有效 */
				wifi_acked:1;     /* WiFi ACK 状態 */
#endif
				no_fcs:1,         /* 無 FCS */
				encapsulation:1,  /* 內部標頭有效 */
				encap_hdr_csum:1,     /* 封裝標頭校驗和 */
				csum_valid:1;     /* 校驗和有效 */

#if IS_ENABLED(CONFIG_IP_VS)
	__u8			ipvs_property:1;  /* IPVS 屬性 */
#endif
#if IS_ENABLED(CONFIG_NETFILTER_XT_TARGET_TRACE)
	__u8			nf_trace:1;       /* Netfilter 追蹤 */
#endif
#ifdef CONFIG_NET_SWITCHDEV
	__u8			offload_fwd_mark:1;   /* 卸載轉發標記 */
	__u8			offload_l3_fwd_mark:1;    /* L3 卸載標記 */
#endif
	__u8			redirected:1;     /* 已重定向 */
#ifdef CONFIG_NET_REDIRECT
	__u8			from_ingress:1;   /* 來自入口路徑 */
#endif
#ifdef CONFIG_NETFILTER_SKIP_EGRESS
	__u8			nf_skip_egress:1; /* 跳過 Netfilter 出口 */
#endif
#ifdef CONFIG_SKB_DECRYPTED
	__u8			decrypted:1;      /* 已解密 */
#endif
	__u8			slow_gro:1;       /* 慢速 GRO */
#if IS_ENABLED(CONFIG_IP_SCTP)
	__u8			csum_not_inet:1;  /* CRC32c（非 IP） */
#endif
	__u8			unreadable:1;     /* 不可讀 */

#if defined(CONFIG_NET_SCHED) || defined(CONFIG_NET_XGRESS)
	__u16			tc_index;         /* 流量控制索引 */
#endif

	u16			alloc_cpu;        /* 分配 CPU */

	union {
		__wsum		csum;             /* 校驗和值 */
		struct {
			__u16	csum_start;       /* 校驗和起始位移 */
			__u16	csum_offset;      /* 校驗和欄位位移 */
		};
	};

	__u32			priority;         /* QoS 優先級 */
	int			skb_iif;          /* 輸入介面索引 */
	__u32			hash;             /* 流雜湊 */

	union {
		u32		vlan_all;
		struct {
			__be16	vlan_proto;       /* VLAN 協定 */
			__u16	vlan_tci;         /* VLAN 標籤控制資訊 */
		};
	};

#if defined(CONFIG_NET_RX_BUSY_POLL) || defined(CONFIG_XPS)
	union {
		unsigned int	napi_id;      /* NAPI 上下文 ID */
		unsigned int	sender_cpu;   /* 傳送者 CPU */
	};
#endif

#ifdef CONFIG_NETWORK_SECMARK
	__u32		secmark;              /* SELinux 安全標記 */
#endif

	union {
		__u32		mark;             /* netfilter 標記 */
		__u32		reserved_tailroom;    /* 保留尾室 */
	};

	union {
		__be16		inner_protocol;
		__u8		inner_ipproto;
	};

	__u16			inner_transport_header;  /* 內部 L4 標頭偏移 */
	__u16			inner_network_header;    /* 內部 L3 標頭偏移 */
	__u16			inner_mac_header;        /* 內部 L2 標頭偏移 */

	__be16			protocol;         /* L3 協定類型 */
	__u16			transport_header; /* L4 標頭偏移 */
	__u16			network_header;   /* L3 標頭偏移 */
	__u16			mac_header;       /* L2 標頭偏移 */

#ifdef CONFIG_KCOV
	u64			kcov_handle;      /* 代碼覆蓋率句柄 */
#endif

	); /* 標頭組結尾 */

	/* 以下欄位必須位於結構結尾（alloc_skb 限制） */
	sk_buff_data_t		tail;         /* 有效資料結尾 */
	sk_buff_data_t		end;          /* 緩衝區結尾 */
	unsigned char		*head,        /* 緩衝區起始 */
				*data;        /* 有效資料起始 */
	unsigned int		truesize;     /* 實際佔用記憶體 */
	refcount_t		users;        /* 參考計數 */

#ifdef CONFIG_SKB_EXTENSIONS
	struct skb_ext		*extensions;  /* 擴展資料 */
#endif
};
```

## Field Groups

### 鏈結與樹節點 (Linking and Tree Nodes)

- **next, prev**: 雙向鏈結串列指標，**必須位於結構開頭**，以相容 `sk_buff_head`
- **rbnode**: 紅黑樹節點，用於 netem 佇列管理、IP 分片重組、TCP 堆疊
- **list**: 通用雙向鏈結串列
- **ll_node**: 無鎖鏈結串列節點，用於高效能批量操作

### 網路裝置與協議 (Device and Protocol)

- **dev**: 所屬的 `struct net_device`（可能為 NULL）
- **dev_scratch**: 替代空間，某些協議（如 UDP）利用此空間存放資訊
- **sk**: 指向關聯的 `struct sock`，用於流狀態追蹤和套接字操作

### 時間戳記 (Timestamps)

- **tstamp**: 軟體時間戳記（ktime_t），由 `ktime_get_real()` 填充
- **skb_mstamp_ns**: GSO 相關，記錄最早出發時間（與 tstamp 互斥）

### 控制與私有資料 (Control and Private Data)

- **cb[48]**: 控制緩衝區，每個網路層可存放私有資料（如 TCP、UDP 使用）。容量 48 bytes，對齐到 8 bytes

### 目的路由 (Destination and Routing)

- **_skb_refdst**: dst_entry 指標（可選參考計數）
- **destructor**: 自訂解構器，可清理關聯資源（如 AF_UNIX sockets）
- **tcp_tsorted_anchor**: TCP 時序排序錨點（複用 _skb_refdst 聯合）
- **_sk_redir**: 套接字重定向（CONFIG_NET_SOCK_MSG，如 BPF sockmap）

### Netfilter 連線追蹤 (Netfilter Conntrack)

- **_nfct**: conntrack 物件指標（CONFIG_NF_CONNTRACK），用於連線狀態追蹤和 NAT

### 封包長度 (Packet Length)

- **len**: 完整封包長度（線性 head 部分 + 分片 data_len）
- **data_len**: 分片中的資料長度（非線性部分，透過 skb_shared_info.frags[]）
- **mac_len**: L2 標頭長度（以位元組為單位）
- **hdr_len**: 標頭長度（無標頭克隆時使用，記錄可用 headroom）

### 裝置與隊列 (Device and Queue Management)

- **queue_mapping**: 傳送佇列索引映射（用於 SMP 多隊列傳送）
- **alloc_cpu**: 分配本 sk_buff 的 CPU 編號

### 克隆和狀態旗標 (Cloning and State Flags)

- **cloned**: 是否為克隆（資料緩衝區共享）
- **nohdr**: 無標頭模式（TCP 使用，允許加入新標頭）
- **fclone**: fast clone 狀態（0=不可用、1=原件、2=克隆件）
- **peeked**: 已被 `skb_peek()` 存取
- **head_frag**: head 指向頁面片段（而非 kmalloc 區域）
- **pfmemalloc**: 來自應急記憶體池（防 OOM 用）
- **pp_recycle**: page pool 回收指示，用於 XDP

### 封包類型與校驗和 (Packet Type and Checksumming)

- **pkt_type**: 3 位元，封包類型（PACKET_HOST/BROADCAST/MULTICAST/OTHERHOST/OUTGOING）
- **ip_summed**: 2 位元，校驗和狀態
  - `CHECKSUM_NONE = 0`: 無校驗和卸載
  - `CHECKSUM_UNNECESSARY = 1`: 硬體已驗證
  - `CHECKSUM_COMPLETE = 2`: 硬體提供完整校驗和
  - `CHECKSUM_PARTIAL = 3`: 待卸載（驅動完成）
- **csum**: 校驗和值（或起始/偏移拆分）
- **csum_start**: 校驗和計算起點位移
- **csum_offset**: 校驗和欄位位移
- **csum_level**: 連續校驗和層級（max 3）
- **csum_complete_sw**: 軟體計算的完整校驗和
- **remcsum_offload**: 遠端校驗和卸載（GUE tunnel）
- **csum_not_inet**: CRC32c（非 IP，用於 SCTP/FCOE）
- **csum_valid**: 校驗和有效

### 時間戳記類型與流量控制 (Tstamp Type and Traffic Control)

- **tstamp_type**: 2 位元，時間戳記來源（REALTIME/MONOTONIC/TAI）
- **tc_at_ingress**: 在 TC 入口路徑處理
- **tc_skip_classify**: 跳過 TC 分類器
- **tc_index**: 16 位元，TC qdisc 索引

### 協議與分段 (Protocol and Segmentation)

- **ignore_df**: 忽略 IP DF（勿分片）旗標
- **ooo_okay**: 允許亂序交付
- **encapsulation**: 內部標頭有效（隧道）
- **encap_hdr_csum**: 封裝標頭校驗和有效
- **inner_protocol_type**: 內部協定類型（IPv4/IPv6）
- **inner_transport_header**: 內部 L4 標頭偏移
- **inner_network_header**: 內部 L3 標頭偏移
- **inner_mac_header**: 內部 L2 標頭偏移
- **protocol**: L3 協定（`__be16`，如 ETH_P_IP）
- **transport_header**: L4 標頭偏移
- **network_header**: L3 標頭偏移
- **mac_header**: L2 標頭偏移

### 雜湊與特性 (Hashing and Features)

- **hash**: 流/連線雜湊值（用於 NIC RSS、TC flow director）
- **l4_hash**: L4 雜湊有效
- **sw_hash**: 軟體雜湊（非 RSS）
- **slow_gro**: GRO 聚合困難（無法最佳化）

### VLAN 標籤 (VLAN Tags)

- **vlan_proto**: VLAN 協定（`__be16`，如 ETH_P_8021Q）
- **vlan_tci**: VLAN 標籤控制資訊（優先級 + 標籤 ID）
- **vlan_all**: 32 位元聯合視圖

### QoS 與優先級 (QoS and Priority)

- **priority**: QoS 優先級（套接字 SO_PRIORITY 映射到 TC）
- **mark**: netfilter 標記（iptables -j MARK）
- **reserved_tailroom**: 保留尾室大小（與 mark 複用）
- **secmark**: SELinux 安全標記（CONFIG_NETWORK_SECMARK）

### 網路輸入輸出 (Network I/O)

- **skb_iif**: 輸入介面索引（接收時填充）
- **napi_id**: NAPI poll 上下文 ID（RX busy polling）
- **sender_cpu**: 傳送者 CPU 編號（CONFIG_XPS）
- **dst_pending_confirm**: 目的地址確認待定
- **redirected**: 已被 BPF 或 AF_XDP 重定向
- **from_ingress**: 來自入口路徑

### 無線網路特定 (Wireless Specific)

- **wifi_acked_valid**: WiFi ACK 狀態有效
- **wifi_acked**: WiFi MAC ACK 已收到
- **no_fcs**: 無 FCS（Frame Check Sequence）

### Netfilter 與安全 (Netfilter and Security)

- **nf_trace**: Netfilter 追蹤啟用（TRACE target）
- **ipvs_property**: IPVS（虛擬伺服器）屬性
- **nf_skip_egress**: 跳過出口 Netfilter 鉤點
- **offload_fwd_mark**: 交換機卸載轉發標記
- **offload_l3_fwd_mark**: L3 卸載轉發標記
- **decrypted**: 已被核心 TLS 解密

### 記憶體與資料 (Memory and Data)

- **head**: 緩衝區起始指標（由 kmalloc/page allocator）
- **data**: 有效資料起始指標（位於 head 之後）
- **tail**: 有效資料結尾位置（可能為 offset 或指標，取決於 NET_SKBUFF_DATA_USES_OFFSET）
- **end**: 緩衝區結尾位置
- **truesize**: 實際佔用記憶體大小（包含分頁浪費）

### 引用與擴展 (References and Extensions)

- **users**: 原子參考計數（refcount_t），>0 時持有資源
- **active_extensions**: 活躍擴展位掩碼（CONFIG_SKB_EXTENSIONS）
- **extensions**: 動態擴展指標（存放 skb_ext_type 資料）

### 代碼覆蓋率 (Code Coverage)

- **kcov_handle**: 代碼覆蓋率句柄（CONFIG_KCOV，用於 syzkaller）

## Lifecycle

### 分配 (Allocation)

sk_buff 通常透過以下函式分配（定義於 `net/core/skbuff.c`）：

1. **`__alloc_skb(size, gfp_mask, flags, node)`** (line 648)
   - 核心分配函式，支援 NUMA 親和性
   - flags: `SKB_ALLOC_FCLONE`（快速克隆）、`SKB_ALLOC_RX`（RX 導向）、`SKB_ALLOC_NAPI`（NAPI 快取）
   - 流程：
     1. 由 NAPI 快取或 kmem_cache (skbuff_cache/skbuff_fclone_cache) 取得 sk_buff 結構
     2. 由 kmalloc_reserve() 分配資料緩衝區（head）
     3. 在緩衝區尾部構建 skb_shared_info（分片陣列容納位置）
     4. 初始化 tail/end 指標、users 計數器
     5. 若為 fclone 模式，初始化 fclone_ref

2. **`__netdev_alloc_skb(dev, len, gfp_mask)`** (line 738)
   - 裝置層分配，適合 RX（接收）
   - 在資料區開頭預留 NET_SKB_PAD headroom
   - 優先使用 page_frag_cache（高效能分片分配）

3. **`napi_alloc_skb(napi, len)`** (inline)
   - NAPI 專用快速分配
   - 整合 page pool 回收機制
   - 無鎖（NAPI context = 軟體中斷禁用）

4. **其他變體**
   - `alloc_skb_for_msg()`: 訊息層分配
   - `alloc_skb_with_frags()`: 預分配分片

### 使用與操作

核心操作函式（在 `net/core/skbuff.c` 及 inline 函式中）：

| 操作 | 行號 | 說明 |
|------|------|------|
| `skb_reserve(skb, len)` | inline | 移動 data/tail 指標，建立 headroom |
| `skb_put(skb, len)` | inline | 增加 tail，擴展封包資料 |
| `skb_push(skb, len)` | inline | 移動 data，加入新標頭 |
| `skb_pull(skb, len)` | inline | 移動 data，移除標頭 |
| `skb_clone(skb, gfp)` | 2069 | 淺複製（資料共享），增加 users/dataref |
| `skb_copy(skb, gfp)` | 2149 | 深複製（完整複製，包含所有資料） |
| `__pskb_copy_fclone()` | 1995 | 複製含 fclone 支援 |
| `skb_copy_expand()` | 1454 | 複製並擴展 headroom/tailroom |
| `skb_make_writable()` | inline | 確保標頭可寫（克隆時複製） |
| `skb_shared_info()` | inline | 取得 tail 的 skb_shared_info 指標 |

### 釋放 (Deallocation)

1. **`__kfree_skb(skb)`** (line 1194)
   - 內部私有函式
   - 流程：
     1. 呼叫 `skb_release_all(skb)` 解除資源（如 dst_entry、destructor、extensions）
     2. 呼叫 `kfree_skbmem(skb)` 解除結構和資料緩衝區
   - 不檢查參考計數（使用者必須先檢查）

2. **`sk_skb_reason_drop(sk, skb, reason)`** (line 1231)
   - 公開函式，自動檢查參考計數
   - 解除計數後呼叫 `__kfree_skb()`
   - reason: drop 原因（用於追蹤）

3. **`kfree_skb(skb)` / `consume_skb(skb)`**
   - 高階封裝，區分 drop（異常）vs consume（正常處理）
   - 啟用 tracepoint 追蹤

4. **分片釋放**
   - 每個分片計數於 `skb_shared_info.dataref`（原子計數）
   - 最後一個引用釋放時，由 skb 中的 destructor 清理

### 參考計數語義

- **users**: sk_buff 本身的計數
  - 0: 可釋放
  - >1: 多個使用者（如克隆、TCP retrans queue）
  - 遞減至 0 時觸發 `__kfree_skb()`

- **dataref** (in skb_shared_info): 資料緩衝區計數
  - 低 16 位: 總參考數
  - 高 16 位: payload-only 參考數（無標頭克隆）
  - 用於 CoW（copy-on-write）語義

## Key Operations

### 標頭操作

```c
/* 在資料起點加入標頭 (push) */
unsigned char *skb_push(struct sk_buff *skb, unsigned int len)
{
    skb->data -= len;
    return skb->data;
}

/* 從資料起點移除標頭 (pull) */
unsigned char *skb_pull(struct sk_buff *skb, unsigned int len)
{
    return skb_pull_inline(skb, len);  /* inline 版本 */
}

/* 在資料結尾加入資料 (append) */
unsigned char *skb_put(struct sk_buff *skb, unsigned int len)
{
    skb->tail += len;
    return skb->tail - len;
}
```

### 複製與克隆

- **淺複製 (Clone)**: `skb_clone()` 共享資料，增加 users/dataref
  - 適合 TCP retransmit queue（發送相同資料）
  - 低開銷，但資料不可修改

- **深複製 (Copy)**: `skb_copy()` 完全複製
  - 適合層間資料隔離
  - 高開銷（涉及記憶體分配）

### 分片管理

```c
struct skb_shared_info {
    skb_frag_t  frags[MAX_SKB_FRAGS];  /* max 17 或更多 */
    atomic_t    dataref;               /* 分片參考計數 */
    ...
};
```

- **MAX_SKB_FRAGS**: 單個 skb 的最大分片數（預設 17）
- **skb_frag_t**: (netmem_ref, len, offset) 三元組
  - netmem_ref: page 或 netmem 對象
  - len: 分片長度
  - offset: 分片內偏移

### GSO/GRO 相關

- **gso_size**: 通用分段卸載（GSO）大小
- **gso_segs**: 分段數
- **gso_type**: GSO 類型 (SKB_GSO_TCPV4/TCPV6/UDP_TUNNEL 等)

## Relationships

### 相關結構

1. **struct net_device** (include/linux/netdevice.h:2105)
   - skb->dev: 接收/傳送所在裝置
   - napi_struct: NAPI poll context

2. **struct sock** (include/net/sock.h:359)
   - skb->sk: 關聯套接字
   - 用於優先級、流控制、應用層回呼

3. **struct dst_entry** (include/net/dst.h)
   - skb->_skb_refdst: 路由資訊、MTU、輸出 callback

4. **struct skb_shared_info** (include/linux/skbuff.h:593)
   - tail 指向的結構，包含分片陣列、GSO 資訊

5. **struct nf_conntrack** (include/net/netfilter/nf_conntrack.h)
   - skb->_nfct: 連線追蹤狀態

### 層間傳遞

```
應用層 (socket recv/send)
    ↓
傳輸層 (TCP/UDP 組裝/解析)
    ↓ skb_push() 加入 L4 標頭
網路層 (IPv4/IPv6 路由、Netfilter)
    ↓ skb_push() 加入 L3 標頭
鏈路層 (ARP 解析、VLAN)
    ↓ skb_push() 加入 L2 標頭
裝置層 (NIC 驅動、GSO/GRO)
```

每層可存放私有資訊於 `cb[48]` 控制緩衝區。

## Android-Specific Changes

Android Common Kernel (ACK) 在網路堆疊中主要新增以下特性（與上游相比）：

1. **Vendor Hooks** (include/trace/hooks/net.h)
   - `DECLARE_HOOK(android_vh_skb_classify)`: TC 分類器前插入點
   - 允許廠商 (OEM/高通) 加入自訂邏輯，無需修改核心代碼

2. **Page Pool 整合** (net/core/page_pool.c)
   - napi_alloc_skb() 優先使用 page pool 而非 kmalloc
   - pp_recycle flag: 標記可由 page pool 回收
   - 提升 RX 效能，減少記憶體碎片

3. **QRTR 支援** (net/qrtr/)
   - Qualcomm IPC Router: 用於數據機/感測器通訊
   - 透過 AF_QIPCRTR socket 族

4. **IPv6 私密地址** (net/ipv6/)
   - RFC 4941 隱私地址生成（用於 IPv6 連接）

5. **SELinux 網路增強**
   - 擴展 mediation 至更多網路操作
   - CONFIG_NETWORK_SECMARK 支援

### 未來演進

隨著 Linux mainline 演進，ACK 新增的欄位可能包括：

- **xdp_frags_size / xdp_frags_truesize**: XDP multi-buffer 支援
- **active_extensions**: skb_ext 動態擴展（避免結構膨脹）
- **kcov_handle**: syzkaller 程式化模糊測試支援

## Cross-References

### 核心操作實現

- **alloc_skb()**: net/core/skbuff.c:648-722
- **__kfree_skb()**: net/core/skbuff.c:1194-1199
- **skb_clone()**: net/core/skbuff.c:2069+
- **skb_push/pull/put**: include/linux/skbuff.h (inline)

### 層級整合

- **Socket 層**: include/net/sock.h、net/socket.c
- **TCP**: net/ipv4/tcp.c、net/ipv4/tcp_input.c
- **UDP**: net/ipv4/udp.c
- **IPv4**: net/ipv4/ip_input.c、ip_output.c
- **IPv6**: net/ipv6/ip6_input.c、ip6_output.c
- **Netfilter**: net/netfilter/nf_conntrack.c、nf_nat.c
- **TC (Qdisc)**: net/sched/sch_*.c
- **NAPI/GRO**: net/core/dev.c:4000-5000 行
- **XDP**: net/core/dev.c + include/net/xdp.h

### 追蹤點

- **trace_kfree_skb**: 記錄 skb 釋放事件及 drop reason
- **trace_consume_skb**: 記錄正常消費事件
- **trace_napi_poll**: NAPI 輪詢追蹤
- **trace_net_dev_queue**: 裝置傳送佇列追蹤

### 配置選項 (Kconfig)

- **CONFIG_HAVE_EFFICIENT_UNALIGNED_ACCESS**: 使用 offset 而非指標
- **CONFIG_SKB_EXTENSIONS**: 啟用動態擴展
- **CONFIG_NF_CONNTRACK**: Netfilter 連線追蹤
- **CONFIG_NET_SCHED**: 流量控制
- **CONFIG_NET_RX_BUSY_POLL**: RX 忙輪詢優化
- **CONFIG_WIRELESS**: WiFi ACK 追蹤

---

## Summary

`struct sk_buff` 是 Linux 網路堆疊的基石，以統一的記憶體容器表示所有網路封包，支援複雜的複製、克隆、分片、分段卸載等語義。其設計平衡了效能（NAPI 快取、fast clone、無鎖操作）與功能性（Netfilter 掛勾、TC 分類、BPF/XDP 集成）。

在 Android Common Kernel 中，透過 vendor hook 和 page pool 整合進一步優化，使其適應高效能行動裝置的需求。理解 sk_buff 的生命週期、參考計數語義及層間操作，是深入網路堆疊優化與除錯的必備基礎。
