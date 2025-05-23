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