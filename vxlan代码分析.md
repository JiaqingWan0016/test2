# Linux 内核中 VXLAN 实现分析

从提供的 vxlan.c 和 vxlan.h 文件中，我们可以分析 Linux 内核中 VXLAN (Virtual eXtensible Local Area Network) 的实现。这是一个网络虚拟化技术，允许在现有的三层网络上创建二层覆盖网络。

## 核心数据结构

### 1. VXLAN 头部结构

在 vxlan.h 中定义了 VXLAN 协议头部：

```c
struct vxlanhdr {
    __be32 vx_flags;
    __be32 vx_vni;
};
```

这是标准 VXLAN 头部，包含标志位和 VNI (VXLAN Network Identifier)。

### 2. 扩展头部

VXLAN 支持多种扩展：

- **GBP (Group Based Policy)**: 通过 `struct vxlanhdr_gbp` 实现
- **GPE (Generic Protocol Extension)**: 通过 `struct vxlanhdr_gpe` 实现
- **远程校验和卸载**: 使用 `VXLAN_HF_RCO` 标志

### 3. 主要结构体

```c
/* 每个网络命名空间的 VXLAN 私有数据 */
struct vxlan_net {
    struct list_head  vxlan_list;
    struct hlist_head sock_list[PORT_HASH_SIZE];
    spinlock_t        sock_lock;
};

/* 转发表项 */
struct vxlan_fdb {
    struct hlist_node hlist;    /* 条目链表 */
    struct rcu_head   rcu;
    unsigned long     updated;  /* jiffies */
    unsigned long     used;
    struct list_head  remotes;
    u8                eth_addr[ETH_ALEN];
    u16               state;    /* 见 ndm_state */
    __be32            vni;
    u8                flags;    /* 见 ndm_flags */
};
```

## 核心功能实现

### 1. 转发数据库 (FDB) 管理

VXLAN 使用 FDB 表来维护 MAC 地址与远程 VTEP 的映射关系：

- **查找**: `vxlan_find_mac()` 函数根据 MAC 地址和 VNI 查找 FDB 条目
- **添加/更新**: `vxlan_fdb_update()` 函数处理 FDB 条目的添加和更新
- **删除**: `vxlan_fdb_destroy()` 函数删除 FDB 条目

### 2. 地址学习机制

```c
static bool vxlan_snoop(struct net_device *dev,
                        union vxlan_addr *src_ip, const u8 *src_mac,
                        u32 src_ifindex, __be32 vni)
```

这个函数实现了 VXLAN 的地址学习机制，当收到未知源 MAC 地址的数据包时，会自动学习 MAC 地址与 VTEP IP 的映射关系。

### 3. 数据包处理

#### 发送路径

当数据包通过 VXLAN 设备发送时：

1. 查找目标 MAC 地址对应的 FDB 条目
2. 获取远程 VTEP 的 IP 地址
3. 封装 VXLAN 头部、UDP 头部和 IP 头部
4. 通过底层网络发送封装后的数据包

#### 接收路径

当 VXLAN 封装的数据包到达时：

1. UDP 协议栈将数据包传递给 VXLAN 处理函数
2. 解析 VXLAN 头部，提取 VNI
3. 根据 VNI 找到对应的 VXLAN 设备
4. 解封装数据包，提取原始以太网帧
5. 可能进行地址学习
6. 将解封装后的数据包传递给上层

### 4. 多播组管理

VXLAN 支持通过多播实现未知目标地址的泛洪：

- `vxlan_igmp_join()`: 加入多播组
- `vxlan_igmp_leave()`: 离开多播组

### 5. 套接字管理

VXLAN 使用 UDP 套接字进行数据传输：

- `vxlan_sock_add()`: 创建和添加 VXLAN 套接字
- `vxlan_sock_release()`: 释放 VXLAN 套接字

## 特殊功能支持

### 1. ECN (Explicit Congestion Notification) 支持

```c
static bool vxlan_ecn_decapsulate(struct vxlan_sock *vs, void *oiph,
                                 struct sk_buff *skb)
```

实现了 ECN 信息在 VXLAN 隧道中的传递。

### 2. 校验和卸载

```c
static struct vxlanhdr *vxlan_gro_remcsum(struct sk_buff *skb,
                                         unsigned int off,
                                         struct vxlanhdr *vh, size_t hdrlen,
                                         __be32 vni_field,
                                         struct gro_remcsum *grc,
                                         bool nopartial)
```

支持远程校验和卸载功能，提高性能。

### 3. GRO (Generic Receive Offload) 支持

```c
static struct sk_buff **vxlan_gro_receive(struct sock *sk,
                                         struct sk_buff **head,
                                         struct sk_buff *skb)
```

实现了 VXLAN 的 GRO 功能，合并小数据包提高性能。

## VXLAN 与 VLAN 的关联

虽然代码中没有直接显示 VXLAN 与 VLAN 的关联实现，但从整体架构可以看出：

1. VXLAN 设备作为虚拟网络设备，可以像物理设备一样创建 VLAN 接口
2. 当 VLAN 接口建立在 VXLAN 设备上时，数据包会先经过 VLAN 处理，然后再由 VXLAN 设备处理
3. 这种堆叠实现了 VLAN-over-VXLAN 的功能

## 总结

Linux 内核中的 VXLAN 实现是一个完整的网络虚拟化解决方案，支持：

1. 标准 VXLAN 协议及多种扩展
2. 动态 MAC 地址学习
3. 单播和多播传输模式
4. 高性能特性（GRO、校验和卸载等）
5. 与其他网络虚拟化技术（如 VLAN）的集成

这种实现使得 Linux 可以作为高性能的 VXLAN 终端，为云计算和数据中心网络提供强大的网络虚拟化支持。


# VXLAN 接收函数 vxlan_rcv 实现分析

`vxlan_rcv` 函数是 VXLAN 模块中处理接收数据包的核心函数，它负责解析和处理从网络接收到的 VXLAN 封装的数据包。下面我将分析其实现并提供流程图。

## 函数实现分析

`vxlan_rcv` 函数的主要功能是接收 UDP 封装的 VXLAN 数据包，解封装后将内部的以太网帧传递给上层网络栈。函数的实现逻辑如下：

1. **基本检查**：
   - 检查数据包是否包含完整的 VXLAN 头部
   - 验证 VXLAN 标志位，确保 VNI 标志已设置

2. **获取 VXLAN 套接字和设备**：
   - 从 socket 获取关联的 VXLAN 套接字
   - 根据 VNI 查找对应的 VXLAN 设备

3. **解析 VXLAN 扩展头部**：
   - 处理 GPE (Generic Protocol Extension) 头部（如果启用）
   - 处理 GBP (Group Based Policy) 头部（如果启用）
   - 处理远程校验和卸载（如果启用）

4. **提取内部数据包**：
   - 从 VXLAN 封装中提取原始以太网帧
   - 设置正确的协议类型

5. **MAC 地址处理**：
   - 设置 MAC 头部
   - 可能进行 MAC 地址学习（如果启用）

6. **ECN 处理**：
   - 处理 ECN (Explicit Congestion Notification) 信息

7. **统计和传递**：
   - 更新接收统计信息
   - 将解封装后的数据包传递给 GRO (Generic Receive Offload) 处理

## 流程图

```
vxlan_rcv 函数流程图
┌─────────────────────────────────┐
│ 开始 vxlan_rcv                  │
└───────────────┬─────────────────┘
                ▼
┌─────────────────────────────────┐
│ 检查数据包是否包含完整VXLAN头部  │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查VXLAN标志位是否包含VNI标志   ├─否──► 丢弃数据包      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 是
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 获取VXLAN套接字和VNI             ├─失败► 丢弃数据包      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 成功
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 根据VNI查找对应的VXLAN设备       ├─失败► 丢弃数据包      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 成功
                ▼
┌─────────────────────────────────┐
│ 如果启用GPE，解析GPE头部         │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│ 从VXLAN封装中提取原始以太网帧    │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│ 如果启用远程校验和卸载，处理校验和│
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│ 如果启用GBP，解析GBP头部         │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 检查是否有未处理的标志位         ├─是──► 丢弃数据包      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 否
                ▼
┌─────────────────────────────────┐
│ 设置MAC头部并获取协议类型        │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 如果启用MAC学习，进行地址学习    ├─失败► 丢弃数据包      │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 成功
                ▼
┌─────────────────────────────────┐     ┌─────────────────┐
│ 处理ECN信息                      ├─失败► 增加错误计数    │
└───────────────┬─────────────────┘     └─────────────────┘
                │ 成功
                ▼
┌─────────────────────────────────┐
│ 更新接收统计信息                 │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│ 将数据包传递给GRO处理            │
└───────────────┬─────────────────┘
                │
                ▼
┌─────────────────────────────────┐
│ 结束 vxlan_rcv                  │
└─────────────────────────────────┘
```

## 关键代码分析

1. **VXLAN 头部检查**：
```c
if (!pskb_may_pull(skb, VXLAN_HLEN))
    goto drop;

unparsed = *vxlan_hdr(skb);
if (!(unparsed.vx_flags & VXLAN_HF_VNI))
    goto drop;
```

2. **查找 VXLAN 设备**：
```c
vs = rcu_dereference_sk_user_data(sk);
if (!vs)
    goto drop;

vni = vxlan_vni(vxlan_hdr(skb)->vx_vni);
vxlan = vxlan_vs_find_vni(vs, skb->dev->ifindex, vni);
if (!vxlan)
    goto drop;
```

3. **处理扩展头部**：
```c
if (vs->flags & VXLAN_F_GPE) {
    if (!vxlan_parse_gpe_hdr(&unparsed, &protocol, skb, vs->flags))
        goto drop;
    raw_proto = true;
}

if (vs->flags & VXLAN_F_REMCSUM_RX)
    if (!vxlan_remcsum(&unparsed, skb, vs->flags))
        goto drop;

if (vs->flags & VXLAN_F_GBP)
    vxlan_parse_gbp_hdr(&unparsed, skb, vs->flags, md);
```

4. **MAC 地址处理和学习**：
```c
if (!raw_proto) {
    if (!vxlan_set_mac(vxlan, vs, skb, vni))
        goto drop;
}
```

5. **ECN 处理**：
```c
if (!vxlan_ecn_decapsulate(vs, oiph, skb)) {
    ++vxlan->dev->stats.rx_frame_errors;
    ++vxlan->dev->stats.rx_errors;
    goto drop;
}
```

6. **统计和传递**：
```c
stats = this_cpu_ptr(vxlan->dev->tstats);
u64_stats_update_begin(&stats->syncp);
stats->rx_packets++;
stats->rx_bytes += skb->len;
u64_stats_update_end(&stats->syncp);

gro_cells_receive(&vxlan->gro_cells, skb);
```

通过这个函数的实现，Linux 内核能够高效地处理 VXLAN 封装的数据包，支持各种扩展功能，并与 Linux 网络栈紧密集成。