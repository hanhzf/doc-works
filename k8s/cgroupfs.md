### 什么是 cgroupfs？

`cgroupfs` 是一种用于管理 Linux 控制组（cgroups）文件系统的虚拟文件系统。控制组（cgroups）是 Linux 内核提供的一种机制，用于对进程进行资源限制和管理，包括 CPU、内存、磁盘 I/O 和网络带宽等。通过 cgroupfs，用户和系统管理员可以通过文件系统接口来配置和管理这些控制组。

### cgroupfs 的工作原理

cgroupfs 通过挂载控制组文件系统来工作。每个控制组（cgroup）在文件系统中表示为一个目录，每个资源控制子系统（如 `cpu`、`memory` 等）在控制组目录中表示为文件。这些文件可以用来配置资源限制、监控资源使用以及管理进程。

#### 挂载 cgroupfs

通常，cgroupfs 可以通过以下命令来挂载：

```bash
sudo mount -t cgroup -o cpu,memory cgroup /sys/fs/cgroup
```

这个命令将 CPU 和内存的控制组挂载到 `/sys/fs/cgroup` 目录下。

### cgroupfs 的常见使用场景

1. **容器化环境**：在 Docker 和 Kubernetes 等容器技术中，cgroups 被广泛用于限制和隔离容器的资源使用。每个容器通常对应一个或多个控制组，通过 cgroupfs 接口进行管理。
2. **资源管理和监控**：系统管理员可以使用 cgroupfs 配置进程的资源限制，监控资源使用情况，从而优化系统性能和资源分配。
3. **多租户环境**：在云计算和虚拟化平台上，cgroups 帮助确保不同租户的资源隔离，防止资源争用和相互影响。

### cgroupfs 示例

以下是一个使用 cgroupfs 管理进程 CPU 和内存使用的简单示例：

1. **创建控制组**：
   ```bash
   sudo mkdir /sys/fs/cgroup/cpu/mygroup
   sudo mkdir /sys/fs/cgroup/memory/mygroup
   ```

2. **设置 CPU 使用限制**：
   限制控制组 `mygroup` 中的进程只能使用 50% 的 CPU：
   ```bash
   echo 50000 | sudo tee /sys/fs/cgroup/cpu/mygroup/cpu.cfs_quota_us
   echo 100000 | sudo tee /sys/fs/cgroup/cpu/mygroup/cpu.cfs_period_us
   ```

3. **设置内存使用限制**：
   限制控制组 `mygroup` 中的进程最多使用 256MB 内存：
   ```bash
   echo 256M | sudo tee /sys/fs/cgroup/memory/mygroup/memory.limit_in_bytes
   ```

4. **将进程添加到控制组**：
   将进程（例如 PID 为 12345）添加到控制组 `mygroup`：
   ```bash
   echo 12345 | sudo tee /sys/fs/cgroup/cpu/mygroup/tasks
   echo 12345 | sudo tee /sys/fs/cgroup/memory/mygroup/tasks
   ```

### 使用systemd管理cgroups

现在你的Docker已经切换到使用`systemd`作为cgroup驱动，可以利用systemd的工具和机制来管理和监控cgroups。以下是一些常用的方法和命令来管理cgroups：

### 1. 查看和管理cgroup

#### 使用`systemd-cgls`

`systemd-cgls`命令可以列出cgroup层次结构，显示系统中所有cgroup的树状视图。

```bash
systemd-cgls
```

#### 使用`systemd-cgtop`

`systemd-cgtop`命令提供了一个类似于`top`的界面，用于实时查看cgroup资源使用情况，如CPU、内存和I/O。

```bash
systemd-cgtop
```

#### 使用`systemctl status`查看服务状态

你可以使用`systemctl status`命令来查看某个服务的详细状态信息，包括cgroup资源的使用情况。例如，查看Docker服务的状态：

```bash
systemctl status docker
```

### 2. 管理Docker容器的cgroups

由于Docker容器是作为systemd服务的一部分运行的，可以通过Docker服务的cgroup路径来管理和监控容器的资源使用情况。

#### 使用`systemctl set-property`

可以使用`systemctl set-property`命令动态调整服务的cgroup资源限制。例如，限制Docker服务的CPU使用：

```bash
sudo systemctl set-property docker.service CPUQuota=50%
```

限制Docker服务的内存使用：

```bash
sudo systemctl set-property docker.service MemoryLimit=1G
```

这些命令会即时生效，无需重启服务。

### 3. 查看和调整特定容器的cgroup

#### 找到容器的cgroup路径

Docker容器的cgroup路径通常位于`/sys/fs/cgroup/systemd/docker`目录下。你可以通过以下命令找到特定容器的cgroup路径：

```bash
docker inspect --format '{{.Id}}' <container_name_or_id>
```

使用容器ID，找到对应的cgroup路径：

```bash
ls /sys/fs/cgroup/systemd/docker/<container_id>
```

#### 调整容器的cgroup资源限制

例如，调整特定容器的CPU配额：

```bash
echo 50000 > /sys/fs/cgroup/cpu/docker/<container_id>/cpu.cfs_quota_us
```

调整容器的内存限制：

```bash
echo 1073741824 > /sys/fs/cgroup/memory/docker/<container_id>/memory.limit_in_bytes
```

### 4. 使用Docker命令设置cgroup限制

在启动容器时，可以直接通过Docker命令设置cgroup限制：

```bash
docker run -it --cpus="0.5" --memory="512m" <image_name>
```

这个命令将限制容器使用不超过50%的CPU和512MB的内存。

### 参考资料

- [systemd-cgls Documentation](https://man7.org/linux/man-pages/man1/systemd-cgls.1.html)
- [systemd-cgtop Documentation](https://man7.org/linux/man-pages/man1/systemd-cgtop.1.html)
- [Docker Documentation on Resource Constraints](https://docs.docker.com/config/containers/resource_constraints/)
- [Red Hat Documentation on cgroups and systemd](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/resource_management_guide/sec-cgroups)

通过这些工具和方法，你可以更好地管理和监控Docker容器的资源使用，确保系统的稳定性和性能。