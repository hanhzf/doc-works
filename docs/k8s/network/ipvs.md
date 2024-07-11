IPVS（IP Virtual Server）和 iptables 都是 Linux 内核中的数据包处理技术，kube-proxy 可以使用这两种模式来处理服务流量。尽管它们在总体流程上有些相似，但在具体实现和性能上存在显著差异。以下是它们的主要区别：

### iptables 模式

1. **工作原理**：
   - 使用 Netfilter 子系统的 iptables 来管理数据包的转发和 NAT（网络地址转换）规则。
   - `kube-proxy` 会在每个节点上设置 iptables 规则，这些规则将服务的 ClusterIP 和端口重定向到实际的 Pod IP 和端口。

2. **性能**：
   - iptables 的规则是基于链表结构的，规则数量较多时，数据包处理的性能会受到影响。
   - 数据包处理是在内核空间完成的，但随着规则数量增加，查找和匹配规则的开销也会增加。

3. **可扩展性**：
   - iptables 在大规模集群中处理大量规则时，性能可能会下降。
   - 缺乏高级的负载均衡算法。

4. **优点**：
   - 配置和使用较为简单。
   - 适用于小规模和中等规模的集群。

5. **缺点**：
   - 在规则数量较多时性能下降。
   - 负载均衡算法相对简单。

### IPVS 模式

1. **工作原理**：
   - 使用 Linux 内核中的 IPVS 模块（IP Virtual Server）来进行数据包的负载均衡和转发。
   - `kube-proxy` 会在每个节点上设置 IPVS 规则，这些规则将服务的 ClusterIP 和端口映射到实际的 Pod IP 和端口。

2. **性能**：
   - IPVS 采用哈希表和其他高效的数据结构来管理规则，处理速度比 iptables 更快。
   - 专门设计用于负载均衡，能够高效处理大量并发连接。

3. **可扩展性**：
   - 更适合大规模集群，能够高效处理大量服务和规则。
   - 提供多种高级的负载均衡算法（如 rr, lc, wlc, lblc, lblcr, dh, sh, sed, nq）。

4. **优点**：
   - 高性能，能够处理大量并发连接和规则。
   - 高扩展性，适用于大规模集群。
   - 支持多种负载均衡算法，灵活性更强。

5. **缺点**：
   - 相对配置和管理复杂，需要额外的内核模块支持。
   - 需要内核支持 IPVS 模块，某些环境可能需要手动配置内核模块。

### 具体示例和对比

假设有一个服务 `my-service`，ClusterIP 为 `10.104.14.67`，端口为 `80`，有三个后端 Pod（IP 分别为 `10.0.0.2`, `10.0.0.3`, `10.0.0.4`）。

#### iptables 模式

1. **设置 PREROUTING 规则**：
   ```sh
   iptables -t nat -A PREROUTING -d 10.104.14.67/32 -p tcp --dport 80 -j KUBE-SERVICES
   ```

2. **设置 KUBE-SERVICES 规则**：
   ```sh
   iptables -t nat -A KUBE-SERVICES -d 10.104.14.67/32 -p tcp --dport 80 -j KUBE-SVC-XXXX
   ```

3. **设置 KUBE-SVC-XXXX 规则**：
   ```sh
   iptables -t nat -A KUBE-SVC-XXXX -m statistic --mode random --probability 0.33333333349 -j KUBE-SEP-XXXX1
   iptables -t nat -A KUBE-SVC-XXXX -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-XXXX2
   iptables -t nat -A KUBE-SVC-XXXX -j KUBE-SEP-XXXX3
   ```

4. **设置 KUBE-SEP 规则**：
   ```sh
   iptables -t nat -A KUBE-SEP-XXXX1 -p tcp -m tcp -j DNAT --to-destination 10.0.0.2:80
   iptables -t nat -A KUBE-SEP-XXXX2 -p tcp -m tcp -j DNAT --to-destination 10.0.0.3:80
   iptables -t nat -A KUBE-SEP-XXXX3 -p tcp -m tcp -j DNAT --to-destination 10.0.0.4:80
   ```

#### IPVS 模式

1. **设置服务规则**：
   ```sh
   ipvsadm -A -t 10.104.14.67:80 -s rr
   ```

2. **添加真实服务器**：
   ```sh
   ipvsadm -a -t 10.104.14.67:80 -r 10.0.0.2:80 -m
   ipvsadm -a -t 10.104.14.67:80 -r 10.0.0.3:80 -m
   ipvsadm -a -t 10.104.14.67:80 -r 10.0.0.4:80 -m
   ```

### 总结

- **iptables 模式**：
  - 适用于小规模和中等规模的集群。
  - 性能随着规则数量增加可能下降。
  - 配置和使用较为简单。

- **IPVS 模式**：
  - 适用于大规模集群，高性能和高扩展性。
  - 支持多种高级负载均衡算法。
  - 需要额外的内核模块支持，配置和管理相对复杂。

选择哪种模式取决于集群的规模和性能需求。对于大规模、高性能的集群，IPVS 模式是更好的选择。对于较小规模的集群，iptables 模式可能更简单直接。

## IPVS的历史

### 什么是 IPVS？

IPVS（IP Virtual Server）是 Linux 内核中的一个模块，用于实现负载均衡。它通过虚拟 IP 地址（Virtual IP）将网络流量分发到多个后端服务器（Real Servers），从而提高系统的可用性和扩展性。IPVS 是 LVS（Linux Virtual Server）的一部分，专门用于在内核空间进行高效的负载均衡处理。

### 历史

- **1998年**：IPVS 由章文嵩（Wensong Zhang）在 National Laboratory for Parallel and Distributed Processing, China, 作为 LVS 项目的一部分首次引入。
- **2004年**：IPVS 代码被合并到 Linux 内核 2.6 版本中，成为官方内核的一部分，从此开始广泛使用。

### 基础命令

IPVS 使用 `ipvsadm` 工具进行配置和管理。以下是一些常用的 `ipvsadm` 命令：

1. **查看当前配置**
   ```sh
   ipvsadm -L -n
   ```
   输出当前所有虚拟服务及其关联的真实服务器。

2. **添加虚拟服务**
   ```sh
   ipvsadm -A -t 192.168.1.1:80 -s rr
   ```
   在 TCP 端口 80 上添加一个虚拟服务，使用轮询（Round Robin）调度算法。

3. **添加真实服务器**
   ```sh
   ipvsadm -a -t 192.168.1.1:80 -r 10.0.0.2:80 -m
   ```
   将真实服务器 `10.0.0.2:80` 添加到虚拟服务 `192.168.1.1:80`，使用 NAT 模式。

4. **删除虚拟服务**
   ```sh
   ipvsadm -D -t 192.168.1.1:80
   ```
   删除虚拟服务 `192.168.1.1:80`。

5. **删除真实服务器**
   ```sh
   ipvsadm -d -t 192.168.1.1:80 -r 10.0.0.2:80
   ```
   从虚拟服务 `192.168.1.1:80` 中删除真实服务器 `10.0.0.2:80`。

### 使用场景

1. **Web服务器负载均衡**：
   - IPVS 可以将进入的 HTTP 请求分发到多个 Web 服务器，从而提高网站的性能和可靠性。

2. **数据库负载均衡**：
   - IPVS 可以用于将数据库查询请求分发到多个数据库实例，适用于读多写少的场景。

3. **应用服务负载均衡**：
   - 在微服务架构中，IPVS 可以用于将请求分发到多个微服务实例，确保高可用性和扩展性。

4. **Kubernetes**：
   - 在 Kubernetes 中，kube-proxy 使用 IPVS 作为一种高效的服务代理模式，将集群内的服务请求负载均衡到多个 Pod。

### 被哪些服务使用

1. **Kubernetes**：
   - Kubernetes 的 `kube-proxy` 组件可以使用 IPVS 模式来实现服务的负载均衡，特别适用于大规模集群。

2. **LVS集群**：
   - IPVS 是 LVS（Linux Virtual Server）的一部分，用于构建高可用、高性能的服务器集群。

3. **高流量网站**：
   - 大型网站和应用程序，如电子商务平台、社交媒体网站，经常使用 IPVS 来管理和分发流量。

### 总结

IPVS 是 Linux 内核中的一个模块，用于实现高效的负载均衡。它起源于 LVS 项目，并已被广泛应用于各种高可用性和高扩展性的场景。IPVS 提供了强大的负载均衡功能，支持多种调度算法，适用于 Web 服务器、数据库、应用服务以及 Kubernetes 等多个领域。通过使用 `ipvsadm` 工具，可以方便地配置和管理 IPVS 规则，确保系统能够高效地处理大量并发请求。


## ipvs 介绍

`ipvsadm` 是 Linux Virtual Server (LVS) 项目的用户空间工具，用于配置和管理 IP 虚拟服务器（IPVS）。IPVS 是 Linux 内核中的一项技术，用于实现四层（传输层）负载均衡。IPVS 使用虚拟 IP 地址将客户端请求分发到多个后端服务器（real server, RS），从而提高系统的可用性和扩展性。

在 IPVS 中，有两个核心实体：

1. **Service**：表示一个负载均衡实例。一个 Service 定义了客户端可以访问的虚拟服务，包括虚拟 IP 地址和端口号。
2. **Server**：表示一个后端服务器（real server, RS），即实际提供服务的服务器。一个 Service 可以有多个 Server，用于处理分发来的请求。

### 示例解释

假设我们要创建一个负载均衡实例，用于将客户端请求分发到两台后端 Web 服务器。

#### 1. 定义 Service

Service 表示客户端可以访问的虚拟服务。假设我们的虚拟 IP 地址为 `192.168.1.100`，端口为 `80`，使用轮询（round-robin）调度算法。

#### 2. 添加 Server

Server 是实际提供服务的后端服务器。我们有两台后端 Web 服务器，IP 地址分别为 `192.168.1.101` 和 `192.168.1.102`，它们都监听端口 `80`。

### 配置示例

#### 安装 `ipvsadm`

首先，确保系统已安装 `ipvsadm` 工具：

```sh
sudo apt-get install ipvsadm       # Ubuntu/Debian
sudo yum install ipvsadm           # CentOS/RHEL
```

#### 配置 IPVS

1. **添加 Service**

```sh
sudo ipvsadm -A -t 192.168.1.100:80 -s rr
```

- `-A`：添加一个新的虚拟服务。
- `-t 192.168.1.100:80`：指定虚拟服务的 IP 地址和端口号。
- `-s rr`：指定轮询（round-robin）调度算法。

2. **添加 Server**

```sh
sudo ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.101:80 -m
sudo ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.102:80 -m
```

- `-a`：添加一个新的后端服务器到虚拟服务。
- `-t 192.168.1.100:80`：指定虚拟服务的 IP 地址和端口号。
- `-r 192.168.1.101:80` 和 `-r 192.168.1.102:80`：指定后端服务器的 IP 地址和端口号。
- `-m`：指定 NAT 模式（Masquerading），即源地址转换模式。

### 完整配置过程

```sh
# 添加虚拟服务
sudo ipvsadm -A -t 192.168.1.100:80 -s rr

# 添加后端服务器
sudo ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.101:80 -m
sudo ipvsadm -a -t 192.168.1.100:80 -r 192.168.1.102:80 -m

# 查看配置
sudo ipvsadm -L -n
```

#### 查看配置

使用 `ipvsadm -L -n` 查看当前配置：

```sh
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.1.100:80 rr
  -> 192.168.1.101:80             Masq    1      0          0
  -> 192.168.1.102:80             Masq    1      0          0
```

### 总结

- **Service**：一个负载均衡实例，定义了虚拟 IP 地址和端口号，及其调度算法。
- **Server**：实际提供服务的后端服务器，负责处理由虚拟服务分发的请求。
- **配置示例**：通过 `ipvsadm` 命令添加虚拟服务和后端服务器，实现负载均衡。

通过 `ipvsadm` 配置和管理 IPVS，可以实现高性能的四层负载均衡，确保系统的高可用性和扩展性。