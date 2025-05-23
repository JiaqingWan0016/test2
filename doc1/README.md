# Linux 中数据报文进入 VXLAN 隧道的过程

在 Linux 中，数据报文进入 VXLAN 隧道的过程涉及多个网络层次和内核组件的协作。下面我将详细解析这个过程：

## 数据报文进入 VXLAN 隧道的完整流程

### 1. 数据包的发起

当一个应用程序或容器发送数据包时，这个数据包首先会通过本地网络协议栈，被打上源 MAC 地址和目标 MAC 地址，形成一个标准的二层以太网帧。

### 2. 路由决策

当数据包到达网络层时，内核会查询路由表，决定这个数据包应该通过哪个网络接口发送。如果目标地址属于 VXLAN 网络，路由表会将数据包引导到对应的 VXLAN 设备（如 vxlan0）。

### 3. VXLAN 设备处理

当数据包到达 VXLAN 设备时，会触发 `vxlan_xmit()` 函数，这是 VXLAN 处理发送数据包的核心函数：

```c
static netdev_tx_t vxlan_xmit(struct sk_buff *skb, struct net_device *dev)
{
    struct vxlan_dev *vxlan = netdev_priv(dev);
    struct vxlan_rdst *rdst, vdst;
    bool did_rsc = false;
    struct vxlan_fdb *f;
    struct ethhdr *eth;
    
    // 获取以太网头部
    eth = eth_hdr(skb);
    
    // 查找目标 MAC 地址对应的 FDB 条目
    f = vxlan_find_mac(vxlan, eth->h_dest);
    
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
    
    // 准备 VXLAN 封装
    vxlan_encap(skb, vxlan, rdst, did_rsc);
    
    return NETDEV_TX_OK;
}
```

### 4. FDB 查找

VXLAN 设备会根据目标 MAC 地址查询其 FDB（转发数据库）：

- 如果找到对应条目，就知道了目标 MAC 地址对应的远程 VTEP（VXLAN Tunnel Endpoint）的 IP 地址
- 如果没找到，则根据配置使用默认目标或多播地址

### 5. VXLAN 封装

一旦确定了目标 VTEP，VXLAN 设备会调用 `vxlan_encap()` 函数进行封装：

```c
static void vxlan_encap(struct sk_buff *skb, struct vxlan_dev *vxlan,
                        struct vxlan_rdst *rdst, bool did_rsc)
{
    struct vxlan_metadata _md;
    struct iphdr *iph;
    struct udphdr *uh;
    struct vxlanhdr *vxh;
    
    // 构建 VXLAN 头部
    vxh = (struct vxlanhdr *)(udp_hdr(skb) + 1);
    vxh->vx_flags = VXLAN_HF_VNI;
    vxh->vx_vni = vxlan->vni[0] << 16 | vxlan->vni[1] << 8 | vxlan->vni[2];
    
    // 构建 UDP 头部
    uh = udp_hdr(skb);
    uh->source = htons(vxlan_src_port(skb));
    uh->dest = rdst->remote_port;
    uh->len = htons(skb->len);
    uh->check = 0;
    
    // 构建 IP 头部
    iph = ip_hdr(skb);
    iph->version = 4;
    iph->ihl = 5;
    iph->tos = vxlan->tos;
    iph->ttl = vxlan->ttl;
    iph->protocol = IPPROTO_UDP;
    iph->saddr = vxlan->saddr.sin.sin_addr.s_addr;
    iph->daddr = rdst->remote_ip.sin.sin_addr.s_addr;
    
    // 发送封装后的数据包
    ip_local_out(skb);
}
```

封装过程包括：
- 添加 VXLAN 头部，包含 VNI（VXLAN Network Identifier）
- 添加 UDP 头部，使用 VXLAN 标准端口（通常是 4789）
- 添加 IP 头部，源 IP 是本地 VTEP 的 IP，目标 IP 是远程 VTEP 的 IP

### 6. 发送到物理网络

封装完成后，数据包通过 `ip_local_out()` 函数被送入 IP 层处理，然后：
- 经过路由查找，确定物理出口接口
- 通过 ARP 解析目标 IP 的 MAC 地址
- 添加外层以太网头部
- 最终通过物理网卡发送到网络中

### 7. 内核中的实际调用链

数据包进入 VXLAN 隧道的完整调用链如下：

```
应用程序/容器发送数据 
  → dev_queue_xmit() 
    → __dev_queue_xmit() 
      → vxlan_xmit() 
        → vxlan_find_mac() [FDB 查找]
        → vxlan_encap() [VXLAN 封装]
          → udp_tunnel_xmit_skb() [UDP 隧道发送]
            → ip_local_out() [IP 层发送]
              → dst_output() [路由输出]
                → ip_output() [IP 输出]
                  → ip_finish_output() 
                    → neigh_resolve_output() [ARP 解析]
                      → dev_queue_xmit() [物理设备发送]
```

## 总结

Linux 中数据报文进入 VXLAN 隧道的过程可以概括为：
1. 数据包通过路由决策到达 VXLAN 设备
2. VXLAN 设备查询 FDB 确定目标 VTEP
3. 对原始以太网帧进行 VXLAN 封装（添加 VXLAN 头部、UDP 头部和 IP 头部）
4. 封装后的数据包通过物理网络发送到目标 VTEP

这个过程完全在内核空间完成，对应用程序和容器来说是透明的，它们只需要知道目标 IP 地址，不需要关心 VXLAN 隧道的存在。