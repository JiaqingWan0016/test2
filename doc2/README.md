# VXLAN 中二层报文进入隧道的过程

在 VXLAN 网络中，二层报文（以太网帧）进入隧道的过程是通过内核中的 VXLAN 模块完成的。这个过程涉及到数据包的封装和转发机制。下面我将详细解析这个过程：

## 二层报文进入 VXLAN 隧道的完整流程

### 1. 报文的发起与路由

当一个应用程序或容器发送数据包时，这个二层报文首先会通过本地网络协议栈。系统会根据路由表决策，如果目标地址属于 VXLAN 网络，路由表会将数据包引导到对应的 VXLAN 设备（如 vxlan0）。

### 2. VXLAN 设备处理

当二层报文到达 VXLAN 设备时，会触发 `vxlan_xmit()` 函数：

```c
static netdev_tx_t vxlan_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct vxlan_dev *vxlan = netdev_priv(dev);
    struct ethhdr *eth = eth_hdr(skb);
    
    // 查找目标 MAC 地址对应的 FDB 条目
    struct vxlan_fdb *f = vxlan_find_mac(vxlan, eth->h_dest);
    
    if (f) {
        // 如果找到 FDB 条目，获取远程 VTEP 信息
        rdst = &f->remote;
    } else {
        // 如果没找到，使用默认目标或多播
        if (vxlan->default_dst.remote_ip.sa.sa_family)
            rdst = &vxlan->default_dst;
        else
            goto drop;
    }
    
    // 进行 VXLAN 封装
    vxlan_encap(skb, vxlan, rdst, did_rsc);
    
    return NETDEV_TX_OK;
}
```

### 3. FDB 查找

VXLAN 设备会根据目标 MAC 地址查询其转发数据库（FDB）：

- 如果找到对应条目，就知道了目标 MAC 地址对应的远程 VTEP 的 IP 地址
- 如果没找到，则根据配置使用默认目标或多播地址

### 4. VXLAN 封装过程

一旦确定了目标 VTEP，VXLAN 设备会调用 `vxlan_encap()` 函数进行封装：

```c
static void vxlan_encap(struct sk_buff *skb, struct vxlan_dev *vxlan,
                        struct vxlan_rdst *rdst, bool did_rsc)
{
    // 1. 为 VXLAN 头部、UDP 头部和 IP 头部预留空间
    skb_push(skb, sizeof(struct vxlanhdr));
    skb_push(skb, sizeof(struct udphdr));
    skb_push(skb, sizeof(struct iphdr));
    
    // 2. 构建 VXLAN 头部
    vxh = (struct vxlanhdr *)(udp_hdr(skb) + 1);
    vxh->vx_flags = VXLAN_HF_VNI;
    vxh->vx_vni = vxlan->vni[0] << 16 | vxlan->vni[1] << 8 | vxlan->vni[2];
    
    // 3. 构建 UDP 头部
    uh = udp_hdr(skb);
    uh->source = htons(vxlan_src_port(skb));
    uh->dest = rdst->remote_port;
    uh->len = htons(skb->len - sizeof(struct iphdr));
    uh->check = 0; // UDP 校验和可选
    
    // 4. 构建 IP 头部
    iph = ip_hdr(skb);
    iph->version = 4;
    iph->ihl = 5;
    iph->tos = vxlan->tos;
    iph->ttl = vxlan->ttl;
    iph->protocol = IPPROTO_UDP;
    iph->saddr = vxlan->saddr.sin.sin_addr.s_addr;
    iph->daddr = rdst->remote_ip.sin.sin_addr.s_addr;
    
    // 5. 发送封装后的数据包
    ip_local_out(skb);
}
```

封装过程包括：
- 添加 VXLAN 头部，包含 VNI（VXLAN Network Identifier）
- 添加 UDP 头部，使用 VXLAN 标准端口（通常是 4789）
- 添加 IP 头部，源 IP 是本地 VTEP 的 IP，目标 IP 是远程 VTEP 的 IP

### 5. 发送到物理网络

封装完成后，数据包通过 `ip_local_out()` 函数被送入 IP 层处理，然后：
- 经过路由查找，确定物理出口接口
- 通过 ARP 解析目标 IP 的 MAC 地址
- 添加外层以太网头部
- 最终通过物理网卡发送到网络中

## 二层报文封装的关键点

1. **原始二层帧保持不变**：原始的以太网帧（包括源MAC、目标MAC和以太网类型）在封装过程中保持完整，成为 VXLAN 数据包的负载部分。

2. **VXLAN 头部**：8字节的 VXLAN 头部包含 VNI 信息，用于在目标端识别数据包属于哪个虚拟网络。

3. **外层头部**：添加的 UDP/IP 头部使得原始二层帧可以通过 IP 网络传输，实现了二层网络的跨三层扩展。

4. **FDB 表的作用**：FDB 表维护了 MAC 地址与 VTEP IP 地址的映射关系，是 VXLAN 实现二层转发的关键。

通过这种封装机制，VXLAN 实现了将二层以太网帧通过 IP 网络传输的功能，使得分布在不同物理位置的虚拟机或容器可以像在同一个二层网络中一样通信。