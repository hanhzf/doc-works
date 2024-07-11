### VXLAN（虚拟扩展局域网）

#### 生动的例子

假设你有两个数据中心，分别在纽约和旧金山，每个数据中心都有多个服务器。你希望在这两个数据中心之间实现网络连接，使得虚拟机（VM）可以跨数据中心无缝迁移，同时保持相同的 IP 地址和二层网络环境。

传统的 VLAN 在跨数据中心时面临着限制，因为 VLAN 只能在同一个物理局域网内工作。而 VXLAN 通过在 IP 网络上封装以太网帧，使二层网络能够跨越三层网络，从而解决了这个问题。

#### 概念

VXLAN（Virtual Extensible LAN）使用 UDP 封装技术，将以太网帧封装在 IP 包中，从而实现二层网络的扩展。它创建了一个虚拟网络覆盖层（overlay network），通过三层网络基础设施（underlay network）进行传输。

#### 关键技术

- **VXLAN 网段 ID（VNI）**：每个 VXLAN 网段都有一个唯一的标识符，类似于 VLAN ID，但可以支持多达 16,777,216 个网段（24 位）。
- **VTEP（VXLAN Tunnel Endpoint）**：VTEP 负责在 VXLAN 网络和物理网络之间进行封装和解封装操作。VTEP 可以是物理交换机、虚拟交换机或服务器上的软件模块。

#### 工作原理

1. **封装**：当一个虚拟机（VM）在纽约的数据中心向同一 VXLAN 网段中的另一 VM 发送数据时，源 VTEP 将以太网帧封装在 UDP/IP 包中，并添加 VXLAN 标识符（VNI）。
2. **传输**：封装后的数据包通过三层网络传输到目的地（旧金山的数据中心），在这个过程中，它就像一个普通的 IP 数据包。
3. **解封装**：目的 VTEP 收到数据包后，解封装出原始的以太网帧，并将其转发到目标虚拟机。

### 配置 VXLAN

以下是如何在 Linux 系统上配置 VXLAN 的具体步骤：

#### 步骤 1：创建 VXLAN 接口

首先，在两个数据中心的服务器上创建 VXLAN 接口：

```sh
ip link add vxlan10 type vxlan id 10 dev eth0 dstport 4789
```

- `vxlan10`：VXLAN 接口名称。
- `id 10`：VXLAN 网段 ID（VNI）。
- `dev eth0`：使用的物理网络接口。
- `dstport 4789`：VXLAN 使用的 UDP 端口。

#### 步骤 2：创建桥接接口

创建一个桥接接口，将 VXLAN 接口和本地虚拟机的网络接口连接起来：

```sh
ip link add br0 type bridge
```

#### 步骤 3：将 VXLAN 接口添加到桥接接口

将 VXLAN 接口连接到桥接接口：

```sh
ip link set vxlan10 master br0
```

#### 步骤 4：启用接口

启用 VXLAN 接口和桥接接口：

```sh
ip link set dev vxlan10 up
ip link set dev br0 up
```

#### 步骤 5：配置 IP 地址

为桥接接口配置 IP 地址：

```sh
ip addr add 192.168.100.1/24 dev br0
```

#### 步骤 6：查看 VXLAN 配置

查看 VXLAN 接口的详细信息：

```sh
ip -d link show vxlan10
```

### 例子解释

- **纽约数据中心**：
  - VM1：IP 地址 `192.168.100.2`
  - VTEP1：在 `eth0` 接口上配置了 VXLAN 接口 `vxlan10`，桥接接口 `br0` 的 IP 地址 `192.168.100.1`

- **旧金山数据中心**：
  - VM2：IP 地址 `192.168.100.3`
  - VTEP2：在 `eth0` 接口上配置了 VXLAN 接口 `vxlan10`，桥接接口 `br0` 的 IP 地址 `192.168.100.1`

当 VM1 发送数据包给 VM2 时：

1. **封装**：VTEP1 将以太网帧封装在 UDP/IP 包中，添加 VXLAN ID 10，并发送到旧金山数据中心。
2. **传输**：数据包通过三层网络（互联网）传输到旧金山数据中心的 VTEP2。
3. **解封装**：VTEP2 收到数据包后，解封装出原始的以太网帧，并将其转发给 VM2。

### 总结

- **VXLAN 的作用**：通过封装以太网帧在 IP 包中，实现二层网络的跨三层网络扩展，支持大规模网络虚拟化。
- **关键组件**：VXLAN ID（VNI）和 VTEP（VXLAN Tunnel Endpoint）。
- **配置命令**：`ip link add`、`ip link set`、`ip addr add` 等。

VXLAN 提供了一种灵活、高效的方式来扩展和管理大规模数据中心和云计算环境中的虚拟网络。

## vxlan 帧头

VXLAN（Virtual Extensible LAN）使用 UDP 封装技术，将以太网帧封装在 IP 包中。以下是 VXLAN 封装后的数据帧结构，从内层到外层的详细描述：

### VXLAN 帧结构

1. **原始以太网帧**：
   - **以太网帧头**（Ethernet Header）
     - 目的 MAC 地址（Destination MAC Address）
     - 源 MAC 地址（Source MAC Address）
     - 以太网类型（EtherType）
   - **以太网帧数据**（Ethernet Payload）
     - IP 数据包（IP Packet）
   - **以太网帧尾**（Frame Check Sequence，FCS）

2. **VXLAN 标头**：
   - 24 位的标识符（VNI），用于标识虚拟网络。
   - 8 位标志字段，其中第 3 位设置为 1 表示 VXLAN 标头有效。

3. **外层 UDP 标头**：
   - 源端口（Source Port）
   - 目的端口（Destination Port），通常为 4789。
   - 长度（Length）
   - 校验和（Checksum）

4. **外层 IP 标头**：
   - 版本（Version）
   - 头部长度（Header Length）
   - 服务类型（Type of Service，ToS）
   - 总长度（Total Length）
   - 标识（Identification）
   - 标志和片段偏移（Flags and Fragment Offset）
   - 生存时间（Time to Live，TTL）
   - 协议（Protocol），值为 17 表示 UDP。
   - 校验和（Header Checksum）
   - 源 IP 地址（Source IP Address）
   - 目的 IP 地址（Destination IP Address）

5. **外层以太网帧头**：
   - 目的 MAC 地址（Destination MAC Address）
   - 源 MAC 地址（Source MAC Address）
   - 以太网类型（EtherType），值为 0x0800 表示 IP。

### VXLAN 数据帧示例

以下是一个封装了原始以太网帧的 VXLAN 数据帧示例：

```plaintext
+---------------------------+
| 外层以太网帧头            |
+---------------------------+
| 外层 IP 标头              |
+---------------------------+
| 外层 UDP 标头             |
+---------------------------+
| VXLAN 标头                |
+---------------------------+
| 原始以太网帧头            |
+---------------------------+
| 原始以太网帧数据          |
+---------------------------+
| 原始以太网帧尾（FCS）     |
+---------------------------+
```

### 帧长和开销

1. **外层以太网帧头**：14 字节
2. **外层 IP 标头**：20 字节（不含可选字段）
3. **外层 UDP 标头**：8 字节
4. **VXLAN 标头**：8 字节
5. **原始以太网帧**：不固定，最大 1500 字节（典型 MTU 大小）

总的 VXLAN 开销为 14 + 20 + 8 + 8 = 50 字节。这意味着在标准以太网（MTU 为 1500 字节）中，VXLAN 封装后的有效载荷（原始以太网帧数据）不能超过 1450 字节。

### 示例数据帧

假设原始以太网帧为 1500 字节（标准 MTU），封装后的 VXLAN 帧为 1550 字节（1500 + 50）。这时，必须配置底层网络以支持更大的 MTU，例如 1600 字节或更高，以确保封装后的 VXLAN 帧不会被分片或丢弃。

### 配置示例

以下是如何在 Linux 系统上配置支持 VXLAN 的网络接口：

1. **配置 VXLAN 接口**：

```sh
ip link add vxlan10 type vxlan id 10 dev eth0 dstport 4789
```

2. **创建桥接接口**：

```sh
ip link add br0 type bridge
```

3. **将 VXLAN 接口和物理网络接口添加到桥接接口**：

```sh
ip link set vxlan10 master br0
ip link set eth0 master br0
```

4. **启用接口**：

```sh
ip link set dev vxlan10 up
ip link set dev eth0 up
ip link set dev br0 up
```

5. **配置 IP 地址**：

```sh
ip addr add 192.168.100.1/24 dev br0
```

通过这些步骤，VXLAN 可以实现跨三层网络的二层网络扩展，提供灵活的网络虚拟化解决方案。

## vxlan 转发学习

目标 MAC 地址为 BBBB 的数据帧是如何通过 VTEP 路由的？以下是详细的解释：

### 基本概念

1. **VTEP（VXLAN Tunnel Endpoint）**：VTEP 是 VXLAN 的关键组件，它负责在物理网络和虚拟网络之间进行封装和解封装操作。
2. **MAC 地址学习**：类似于传统的以太网交换机，VTEP 也需要学习并记录 MAC 地址和对应的 VTEP IP 的映射关系。这可以通过数据包的流动和控制协议（如 EVPN，Ethernet VPN）来实现。

### 数据流过程

假设 Host1 和 Host2 之间已经建立了 VXLAN 隧道，且 Host1 的 VTEP IP 地址为 192.168.57.50，Host2 的 VTEP IP 地址为 192.168.57.54。以下是详细的路由过程：

#### 1. 初始数据包发送

- **源**：应用程序在 Host1 内的容器或虚拟机（ns0）。
- **目标**：应用程序在 Host2 内的容器或虚拟机（ns0），MAC 地址为 BBBB，IP 地址为 172.18.1.3。

应用程序生成数据包，并通过以下层次封装：

1. **应用层数据**：实际传输的数据。
2. **传输层（TCP/UDP）头部**：包含源和目标端口。
3. **网络层（IP）头部**：包含源 IP（172.18.1.2）和目标 IP（172.18.1.3）。
4. **数据链路层（以太网）头部**：包含源 MAC（AAAA）和目标 MAC（BBBB）。

#### 2. 二层转发到 VTEP

- **交换机操作**：Host1 内部的虚拟交换机（br0）根据目标 MAC 地址（BBBB）查找转发表。
  - 如果目标 MAC 地址（BBBB）已经被学习到对应的 VTEP IP 地址（192.168.57.54），则数据帧被转发到 vxlan0 接口。
  - 如果未被学习到，则可能通过泛洪或ARP请求等方式获取目标MAC地址的对应VTEP。

#### 3. VXLAN 封装

在 Host1 的 VTEP（vxlan0）上，数据帧被封装成 VXLAN 数据包：

1. **VXLAN 头部**：添加 VNI（42）。
2. **外层 UDP 头部**：源端口（随机），目标端口（4789，VXLAN 默认端口）。
3. **外层 IP 头部**：源 IP（192.168.57.50），目标 IP（192.168.57.54）。
4. **外层以太网头部**：封装外层源 MAC 和目标 MAC 地址。

封装后的数据包通过物理网络发送到 Host2 的 VTEP。

#### 4. VXLAN 解封装

在 Host2 的 VTEP（vxlan0）上，收到封装的 VXLAN 数据包并进行解封装：

1. **移除外层以太网头部**。
2. **移除外层 IP 和 UDP 头部**。
3. **移除 VXLAN 头部**，恢复原始的以太网帧（目标 MAC：BBBB）。

#### 5. 二层转发到目标设备

- **交换机操作**：Host2 内部的虚拟交换机（br0）根据目标 MAC 地址（BBBB）将数据帧转发到对应的 veth0 接口。
- **数据帧到达目标容器或虚拟机**：最终数据帧到达目标命名空间（ns0），并传递给目标应用程序。

### MAC 地址学习

在实际网络中，MAC 地址和对应 VTEP IP 地址的映射关系可以通过多种方式学习和维护：

1. **静态配置**：手动配置 MAC 地址到 VTEP IP 的映射。
2. **动态学习**：VTEP 可以通过数据帧的流动动态学习 MAC 地址。例如，当 Host2 的应用程序发送数据包到 Host1 时，Host1 的 VTEP 可以学习到 Host2 的 MAC 地址（BBBB）对应的 VTEP IP 地址（192.168.57.54）。
3. **控制协议**：使用控制协议（如 EVPN）在网络设备之间共享和同步 MAC 地址到 VTEP IP 的映射信息。

### 总结

通过上述过程，目标 MAC 地址 BBBB 的数据帧能够通过 VTEP 正确路由到目标设备。这依赖于 VTEP 的封装和解封装操作，以及 MAC 地址到 VTEP IP 地址映射关系的学习和维护。