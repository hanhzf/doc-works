## flannel

在 Flannel 网络模式下，不同宿主机的网络不能使用同一个网段，因为这会导致 IP 地址冲突和路由混乱。Flannel 通过将集群的 Pod 网络划分为多个子网，每个节点分配一个唯一的子网来避免这种冲突。

### Flannel 的网络设计

1. **集群网络范围**：Flannel 为整个集群配置一个大的 IP 地址范围，例如 `10.244.0.0/16`。
2. **节点子网分配**：在这个大范围内，Flannel 为每个节点分配一个唯一的子网（例如 `/24` 子网），确保每个节点的 Pod IP 地址不重叠。

### 工作机制

#### 1. 集群网络范围配置
Flannel 在配置时指定一个大的网络范围，例如 `10.244.0.0/16`。这个范围被分成多个子网，每个节点一个子网。

#### 2. 子网分配
当一个新节点加入集群时，Flannel 会为其分配一个唯一的子网。例如：

- 节点1：`10.244.1.0/24`
- 节点2：`10.244.2.0/24`
- 节点3：`10.244.3.0/24`

这种方式确保了不同节点上的 Pod IP 地址不冲突。

#### 3. etcd 存储
Flannel 使用 etcd 来存储网络配置信息，包括每个节点的子网分配情况。这使得所有节点都能知道其他节点的子网信息。

### 示例配置

假设 Flannel 配置了 `10.244.0.0/16` 作为集群网络范围，etcd 中的存储可能如下：

```json
{
  "Network": "10.244.0.0/16",
  "Backend": {
    "Type": "vxlan"
  }
}
```

当新节点加入时，Flannel 分配子网，并在 etcd 中记录：

```json
{
  "SubnetLen": 24,
  "Subnets": [
    {
      "PublicIP": "192.168.1.101",
      "Subnet": "10.244.1.0/24"
    },
    {
      "PublicIP": "192.168.1.102",
      "Subnet": "10.244.2.0/24"
    }
  ]
}
```

### 网络通信

- **节点间通信**：当节点1上的 Pod 需要与节点2上的 Pod 通信时，节点1的 Flannel 代理会通过 VXLAN 隧道将数据包发送到节点2的 Flannel 代理。VXLAN 隧道使用公共 IP（如 `192.168.1.101` 和 `192.168.1.102`）在节点间传输数据包，封装和解封装数据包时会参考 FDB 表和 ARP 表。

- **Pod 间通信**：在 Pod 间通信过程中，每个 Pod 的 IP 地址在其所在节点的子网内唯一。例如，`10.244.1.2`（节点1的 Pod）可以与 `10.244.2.3`（节点2的 Pod）通信，因为它们在不同的子网中。

### 避免冲突

Flannel 确保每个节点分配一个唯一的子网，避免了 IP 地址冲突。以下是 Flannel 如何避免冲突的关键点：

1. **唯一子网分配**：每个节点分配一个唯一的子网，确保不同节点上的 Pod IP 不冲突。
2. **etcd 中的子网信息**：Flannel 使用 etcd 存储和共享网络配置信息，所有节点可以相互了解各自的子网范围。
3. **动态分配和管理**：Flannel 动态管理子网分配，当节点加入或离开时，及时更新子网信息，保持网络配置的一致性。

### 总结

在 Flannel 网络模式下，不同宿主机的网络不能是一个网段，这会导致 IP 地址冲突和网络混乱。Flannel 通过为每个节点分配唯一的子网来避免这种问题。每个节点的 Flannel 代理使用 etcd 中的网络配置，确保整个集群的网络分配合理且不冲突。这种设计确保了 Pod 网络的稳定性和可扩展性。

## 维护arp表

在 Flannel 的 VXLAN 模式中，负责维护 ARP 表并从 etcd 获取 VTEP IP 和 MAC 地址信息的服务是 Flannel 本身的守护进程（通常是 `flanneld`）。

### 具体流程

1. **Flannel 启动**：
   - 当 `flanneld` 启动时，它会在每个节点上创建 VTEP 设备（如 `flannel.1`），并将本节点的 VTEP 信息（包括 IP 和 MAC 地址）上报到 etcd。
   
2. **etcd 同步**：
   - Flannel 使用 etcd 作为分布式存储来共享网络配置和状态。每个 `flanneld` 实例从 etcd 获取其他节点的 VTEP 信息。

3. **更新 ARP 表和 FDB 表**：
   - `flanneld` 守护进程在获取到其他节点的 VTEP 信息后，会将这些信息添加到本地节点的 ARP 表和 FDB 表中。
   - 这使得本地节点能够通过 VTEP 设备正确封装和解封 VXLAN 数据包。

### 维护过程

具体维护 ARP 表的操作是由 `flanneld` 进程自动完成的。`flanneld` 通过以下步骤来确保 ARP 表的正确性：

- **监听 etcd**：`flanneld` 监听 etcd 中与网络配置相关的键值对的变化。
- **获取节点信息**：当有新的节点加入或节点信息变化时，`flanneld` 获取这些变化信息。
- **更新本地配置**：`flanneld` 更新本地的 ARP 表和 FDB 表，确保包含所有其他节点的 VTEP 信息。

### 代码层面的解释

在 `flanneld` 的源码中，通常会有类似以下的逻辑：

1. **监听 etcd**：
   - 通过监听 etcd 中的变化，获取新的节点信息。
2. **更新 ARP 表**：
   - 调用系统命令或 API 更新本地 ARP 表，例如：
     ```sh
     arp -s <VTEP IP> <VTEP MAC>
     ```
3. **更新 FDB 表**：
   - 调用系统命令或 API 更新本地 FDB 表，例如：
     ```sh
     bridge fdb add <VTEP MAC> dev <flannel.1> dst <VTEP IP>
     ```

通过这些机制，Flannel 确保所有节点的 ARP 表和 FDB 表都能够包含其他节点的 VTEP 信息，从而实现跨节点的二层通信。

### 总结

在 Flannel 的 VXLAN 模式中，负责维护 ARP 表并从 etcd 获取 VTEP IP 和 MAC 地址信息的服务是 `flanneld`。`flanneld` 通过监听 etcd 的变化，自动更新本地节点的 ARP 表和 FDB 表，确保跨节点的网络通信正常进行。

## mac地址学习

在 Flannel 的网络模式中，VTEP（VXLAN Tunnel Endpoint）MAC 地址的学习和记录是通过 Flannel 以及 Kubernetes 的配置和控制来实现的。Flannel 是一个简单而高效的网络插件，常用于 Kubernetes 集群中提供容器网络连接。在 VXLAN 模式下，Flannel 会自动管理和配置 VTEP 和其相关的 MAC 地址。

### Flannel 的工作原理

1. **网络初始化**：
   - Flannel 启动时，会向 etcd（或 Kubernetes API 服务器）注册并获取网络配置，确保每个节点有一个唯一的子网。
   - 每个节点会配置一个 VXLAN 设备，这个设备会作为 VTEP。

2. **分配子网**：
   - Flannel 为每个节点分配一个唯一的子网（例如，10.244.1.0/24 分配给节点 A，10.244.2.0/24 分配给节点 B）。
   - 每个节点上的 Flannel 守护进程负责在该子网内分配 IP 地址给该节点上的容器。

3. **MAC 地址学习和记录**：
   - 在 Flannel 的 VXLAN 模式下，VTEP MAC 地址的学习并不像传统的以太网交换机那样依赖于数据包流动和动态学习。相反，Flannel 依赖于其控制平面（即 etcd 或 Kubernetes API 服务器）来管理这些信息。
   - 当一个节点上的容器需要与另一个节点上的容器通信时，Flannel 会通过其控制平面查询目标容器所在节点的 VTEP IP 和相关信息。这些信息通常已经在网络初始化和子网分配阶段记录在 etcd 或 Kubernetes API 中。
   - 一旦查询到目标 VTEP 的信息，Flannel 会将数据包封装到 VXLAN 隧道中，并将其发送到目标 VTEP IP。

### 数据包封装和解封装

- **封装**：源节点的 Flannel 守护进程将原始数据包封装在 VXLAN 隧道中。封装的数据包包含源节点的 VTEP MAC 地址和目标节点的 VTEP MAC 地址。
- **解封装**：目标节点的 Flannel 守护进程接收封装的数据包，解封装后，将其发送到目标容器。

### 总结

在 Flannel 的 VXLAN 模式下，VTEP MAC 地址的学习和记录是通过 Flannel 自身的控制平面来实现的。Flannel 不依赖传统的以太网交换机的动态 MAC 地址学习机制，而是通过 etcd 或 Kubernetes API 服务器来管理和查询节点之间的网络信息。这种方式简化了网络管理，并适应了容器化环境中动态变化的需求。