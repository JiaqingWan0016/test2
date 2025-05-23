# Linux 中 VLAN 接口与 VXLAN 的关联方式

在 Linux 中，VLAN 接口和 VXLAN 可以通过多种方式关联起来，形成更复杂的网络虚拟化拓扑。这种关联通常用于在 VXLAN 覆盖网络中保留 VLAN 标签信息或者在 VXLAN 隧道上传输带有 VLAN 标签的流量。

## VLAN 与 VXLAN 的关联方式

### 1. VLAN 作为 VXLAN 的底层接口

最常见的关联方式是将 VLAN 接口作为 VXLAN 的底层物理接口：

```bash
# 创建 VLAN 接口
ip link add link eth0 name eth0.100 type vlan id 100

# 启用 VLAN 接口
ip link set eth0.100 up

# 在 VLAN 接口上创建 VXLAN
ip link add vxlan10 type vxlan id 10 dev eth0.100 local 192.168.1.1 dstport 4789
```

在这种配置中，VXLAN 隧道的外层 IP 包会通过 VLAN 接口传输，这意味着 VXLAN 流量会带有 VLAN 标签通过物理网络传输。

### 2. VXLAN 上创建 VLAN 接口

另一种方式是在 VXLAN 接口上创建 VLAN 接口，实现 VLAN-over-VXLAN：

```bash
# 先创建 VXLAN 接口
ip link add vxlan10 type vxlan id 10 dev eth0 local 192.168.1.1 dstport 4789

# 在 VXLAN 接口上创建 VLAN
ip link add link vxlan10 name vxlan10.200 type vlan id 200
```

这种配置允许在 VXLAN 覆盖网络内部使用 VLAN 进行更细粒度的网络分段。

### 3. 通过网桥关联

更常见的是通过 Linux 网桥将 VLAN 和 VXLAN 接口关联起来：

```bash
# 创建网桥
ip link add br0 type bridge

# 创建 VLAN 接口
ip link add link eth0 name eth0.100 type vlan id 100

# 创建 VXLAN 接口
ip link add vxlan10 type vxlan id 10 dev eth0 local 192.168.1.1 dstport 4789

# 将接口添加到网桥
ip link set eth0.100 master br0
ip link set vxlan10 master br0

# 启用所有接口
ip link set eth0.100 up
ip link set vxlan10 up
ip link set br0 up
```

这种配置使得 VLAN 接口和 VXLAN 接口可以在二层互通，网桥会负责在它们之间转发流量。

## VLAN-aware VXLAN 配置

在某些高级场景中，可以配置 VXLAN 保留原始 VLAN 标签信息，这称为 VLAN-aware VXLAN：

```bash
# 创建 VLAN-aware 网桥
ip link add br0 type bridge vlan_filtering 1

# 创建 VXLAN 接口并启用 VLAN-aware 模式
ip link add vxlan10 type vxlan id 10 dev eth0 local 192.168.1.1 dstport 4789
bridge vlan add vid 100-200 dev vxlan10

# 将 VXLAN 接口添加到网桥
ip link set vxlan10 master br0

# 配置网桥的 VLAN 过滤
bridge vlan add vid 100-200 dev br0
```

这种配置允许 VXLAN 隧道传输带有 VLAN 标签的流量，并在远端保留这些标签。

## 内核实现原理

从内核实现角度看，VLAN 和 VXLAN 的关联主要通过以下机制实现：

1. **网络设备堆叠**：Linux 内核允许在一个网络设备上创建另一个网络设备，形成设备堆叠。

2. **skb 处理**：当数据包通过这些堆叠的设备时，内核会按顺序处理各层封装：
   - 对于 VLAN-over-VXLAN，数据包首先被 VLAN 模块添加 VLAN 标签，然后被 VXLAN 模块封装
   - 对于 VXLAN-over-VLAN，数据包首先被 VXLAN 模块封装，然后被 VLAN 模块添加 VLAN 标签

3. **FDB 表与 VLAN 过滤**：当使用 VLAN-aware 网桥时，FDB 表会同时考虑 MAC 地址和 VLAN ID，确保流量正确转发。

## 实际应用场景

这种 VLAN 与 VXLAN 的关联在以下场景中特别有用：

1. **数据中心网络**：在物理网络使用 VLAN 分段，同时使用 VXLAN 实现跨数据中心的网络虚拟化

2. **多租户环境**：使用 VXLAN 隔离不同租户的网络，同时在每个租户内部使用 VLAN 进行更细粒度的网络分段

3. **网络迁移**：在迁移到 VXLAN 覆盖网络的过程中，保留现有的 VLAN 配置和策略

通过这些关联方式，Linux 网络虚拟化可以同时利用 VLAN 和 VXLAN 的优势，构建更灵活、更强大的网络架构。