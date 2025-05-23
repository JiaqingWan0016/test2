# Linux 内核中 VLAN-over-VXLAN 的实现原理

Linux 内核中 VLAN-over-VXLAN 的实现是通过网络设备堆叠和数据包处理流程来完成的。这种配置允许在 VXLAN 覆盖网络内部使用 VLAN 进行更细粒度的网络分段。下面我将详细分析其实现原理。

## 内核实现机制

### 1. 网络设备堆叠

Linux 内核允许在一个网络设备上创建另一个网络设备，形成设备堆叠。对于 VLAN-over-VXLAN，实现步骤如下：

```c
// 首先创建 VXLAN 设备
struct net_device *vxlan_dev = vxlan_dev_create(net, name, vxlan_params);

// 然后在 VXLAN 设备上创建 VLAN 设备
struct net_device *vlan_dev = vlan_dev_create(net, vxlan_dev_name, vlan_id);
```

内核中，这种堆叠是通过设备的 `master/slave` 关系或 `parent/child` 关系实现的。

### 2. 数据包处理流程

当数据包通过 VLAN-over-VXLAN 设备时，处理流程如下：

#### 发送方向（出站流量）：

1. 数据包首先进入 VLAN 设备的发送函数 `vlan_dev_hard_start_xmit()`
2. VLAN 设备添加 802.1Q VLAN 标签
3. 修改后的数据包传递给底层 VXLAN 设备的 `vxlan_xmit()` 函数
4. VXLAN 设备进行 VXLAN 封装（添加 VXLAN 头、UDP 头和 IP 头）
5. 封装后的数据包通过物理网卡发送

```c
// VLAN 设备发送函数简化版
static netdev_tx_t vlan_dev_hard_start_xmit(struct sk_buff *skb, struct net_device *dev)
{
    // 添加 VLAN 标签
    skb = vlan_insert_tag(skb, vlan_dev_priv(dev)->vlan_proto,
                         vlan_dev_priv(dev)->vlan_id);
    
    // 将数据包传递给底层设备（这里是 VXLAN 设备）
    skb->dev = vlan_dev_priv(dev)->real_dev;
    return dev_queue_xmit(skb);
}

// 然后数据包会进入 VXLAN 的发送函数
static netdev_tx_t vxlan_xmit(struct sk_buff *skb, struct net_device *dev)
{
    // VXLAN 封装处理
    // ...
}
```

#### 接收方向（入站流量）：

1. 物理网卡接收到数据包
2. 数据包经过 IP 和 UDP 层处理
3. 根据 UDP 端口号，数据包被传递给 VXLAN 设备的 `vxlan_rcv()` 函数
4. VXLAN 设备解封装数据包，提取原始以太网帧
5. 解封装后的数据包被传递给网络栈
6. 如果数据包带有 VLAN 标签，VLAN 模块会处理它并将其传递给相应的 VLAN 设备

```c
// VXLAN 接收函数简化版
static int vxlan_rcv(struct sock *sk, struct sk_buff *skb)
{
    // VXLAN 解封装处理
    // ...
    
    // 移除 VXLAN 头部
    skb_pull(skb, sizeof(struct vxlanhdr));
    
    // 处理解封装后的以太网帧
    // 如果帧带有 VLAN 标签，会被 VLAN 模块处理
    netif_rx(skb);
}
```

### 3. VLAN 标签处理

对于 VLAN-over-VXLAN，VLAN 标签是在原始以太网帧中的，会被整体封装在 VXLAN 隧道中：

```
原始带 VLAN 标签的帧:
+---------------+----------+-------------+
| 以太网头部    | VLAN标签 | 原始数据    |
+---------------+----------+-------------+

VXLAN 封装后:
+---------------+----------+---------------+--------+---------------+----------+-------------+
| 外层以太网头部 | 外层IP头 | 外层UDP头部   | VXLAN头 | 原始以太网头部 | VLAN标签 | 原始数据    |
+---------------+----------+---------------+--------+---------------+----------+-------------+
```

## 内核代码关键部分

VLAN-over-VXLAN 的实现涉及到内核中的多个模块，主要包括：

1. **VLAN 模块**（`net/8021q/`）：负责 VLAN 标签的添加和处理
2. **VXLAN 模块**（`net/vxlan/`）：负责 VXLAN 封装和解封装
3. **网络设备子系统**（`net/core/`）：负责设备堆叠和数据包传递

### VLAN 设备创建

```c
// 在 VXLAN 设备上创建 VLAN 设备
static int vlan_dev_create(struct net *net, const char *name, u16 vlan_id,
                          struct net_device *real_dev)
{
    struct net_device *dev;
    
    // 分配 VLAN 设备
    dev = alloc_netdev(sizeof(struct vlan_dev_priv), name, NET_NAME_UNKNOWN,
                      vlan_setup);
    
    // 设置 VLAN 设备属性
    vlan_dev_priv(dev)->vlan_id = vlan_id;
    vlan_dev_priv(dev)->real_dev = real_dev;
    
    // 注册网络设备
    register_netdevice(dev);
    
    return 0;
}
```

### 数据包封装流程

```c
// VLAN 标签添加
static struct sk_buff *vlan_insert_tag(struct sk_buff *skb, __be16 vlan_proto,
                                      u16 vlan_tci)
{
    struct vlan_hdr *vhdr;
    
    // 为 VLAN 头部预留空间
    skb_push(skb, VLAN_HLEN);
    
    // 移动以太网头部
    memmove(skb->data, skb->data + VLAN_HLEN, 2 * ETH_ALEN);
    
    // 设置 VLAN 协议类型
    skb->protocol = vlan_proto;
    
    // 填充 VLAN 头部
    vhdr = (struct vlan_hdr *)(skb->data + 2 * ETH_ALEN);
    vhdr->h_vlan_TCI = htons(vlan_tci);
    vhdr->h_vlan_encapsulated_proto = skb->protocol;
    
    return skb;
}
```

## 实际应用场景

VLAN-over-VXLAN 在以下场景中特别有用：

1. **多租户环境中的网络细分**：使用 VXLAN 隔离不同租户，同时在每个租户内部使用 VLAN 进行更细粒度的网络分段

2. **网络服务质量（QoS）管理**：在 VXLAN 覆盖网络内部使用 VLAN 优先级标签实现 QoS

3. **兼容传统 VLAN 应用**：允许在现代 VXLAN 覆盖网络中继续使用依赖 VLAN 的传统应用程序

通过这种实现方式，Linux 内核能够灵活地支持复杂的网络虚拟化需求，同时保持高效的数据包处理性能。