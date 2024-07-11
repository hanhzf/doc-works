## pv & pvc

### PVC 和 PV 的区别和关系

#### 区别

1. **定义和角色**：
   - **Persistent Volume (PV)**：
     - PV 是集群管理员创建的存储资源，表示实际的存储卷。PV 是集群级别的资源，与具体的 Pod 无关。
     - PV 的生命周期独立于 Pod，PV 可以在多个 Pod 生命周期中持续存在。
     - PV 是由集群管理员管理的资源，定义了存储的实际配置，如存储类型（NFS、iSCSI、Cloud Storage 等）、存储容量和访问模式等。

   - **Persistent Volume Claim (PVC)**：
     - PVC 是用户对存储资源的请求。PVC 是命名空间级别的资源。
     - PVC 描述了应用程序对存储的需求，如存储大小和访问模式。
     - PVC 的生命周期与请求存储的应用程序相关，用户通过 PVC 请求特定的存储资源。

2. **创建和管理**：
   - **PV** 由集群管理员预先创建或通过 StorageClass 动态创建。
   - **PVC** 由用户或应用程序在需要持久化存储时创建。

3. **功能和用途**：
   - **PV** 作为实际的存储卷，提供存储容量和访问模式等配置。
   - **PVC** 作为对存储资源的声明，描述应用程序的存储需求，并通过绑定获取实际的存储卷。

#### 关系

- **绑定关系**：
  - 当用户创建 PVC 时，Kubernetes 会在集群中查找与 PVC 请求匹配的 PV。如果找到合适的 PV，PVC 将与该 PV 绑定。
  - 绑定过程是单向的，即一个 PVC 只能绑定一个 PV，而一个 PV 在绑定后只能供一个 PVC 使用（除非使用共享访问模式，如 ReadWriteMany）。

- **匹配条件**：
  - **存储容量**：PV 的容量必须大于或等于 PVC 请求的容量。
  - **访问模式**：PV 的访问模式必须包含 PVC 请求的访问模式（如 ReadWriteOnce、ReadWriteMany）。
  - **存储类**：如果 PVC 指定了存储类（StorageClass），则只能绑定到相同存储类的 PV。

- **生命周期管理**：
  - PVC 的生命周期与应用程序相关，应用程序删除时，PVC 可能会被删除。
  - PV 的生命周期由集群管理员管理，PV 可以在 PVC 被删除后继续存在，除非配置了回收策略（如删除、保留、回收）。

### 示例

#### 创建 PV 和 PVC 并绑定

**Persistent Volume (PV) 配置示例**：

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    path: /mnt/data
    server: nfs-server.example.com
```

创建 PV：

```bash
kubectl apply -f example-pv.yaml
```

**Persistent Volume Claim (PVC) 配置示例**：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: example-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

创建 PVC：

```bash
kubectl apply -f example-pvc.yaml
```

查看 PV 和 PVC：

```bash
kubectl get pv
kubectl get pvc
```

### 总结

- **PV 和 PVC 的区别**：
  - PV 是集群管理员预先创建的实际存储资源，独立于 Pod。
  - PVC 是用户对存储资源的请求，描述存储需求，并通过绑定获取实际的存储卷。

- **PV 和 PVC 的关系**：
  - PVC 通过请求匹配到合适的 PV，并建立绑定关系。
  - 绑定后，PVC 获取 PV 提供的存储资源，并将其用于 Pod 中的持久化存储。

这种机制使得 Kubernetes 中的存储管理灵活且高效，支持多种存储后端，并通过声明式的方式简化了存储资源的分配和管理。

## pv 和 pvc如何绑定

PV（Persistent Volume）和 PVC（Persistent Volume Claim）的绑定过程是 Kubernetes 中持久化存储管理的核心部分。这个过程由 Kubernetes 的控制器自动处理，确保用户请求的存储资源（PVC）能够匹配到集群中可用的实际存储资源（PV）。以下是详细的绑定过程和机制。

### 2. 绑定机制

Kubernetes 的控制器自动处理 PV 和 PVC 的绑定过程。以下是详细步骤：

1. **PVC 提交请求**：
   - 当用户提交 PVC 请求时，Kubernetes 会自动创建一个 `PersistentVolumeClaim` 对象。

2. **控制器查找匹配的 PV**：
   - Kubernetes 控制器（PersistentVolumeController）会监视 PVC 请求，并在集群中查找与 PVC 匹配的 PV。
   - 匹配条件包括：
     - **存储容量**：PV 的容量必须大于或等于 PVC 请求的容量。
     - **访问模式**：PV 的访问模式必须包含 PVC 请求的访问模式（如 ReadWriteOnce、ReadWriteMany）。
     - **存储类**：如果 PVC 指定了存储类（StorageClass），则 PV 必须属于相同的存储类。

3. **绑定 PV 和 PVC**：
   - 一旦找到匹配的 PV，控制器会将 PV 和 PVC 绑定。绑定过程会更新 PVC 的 `status.phase` 为 `Bound`，并将 PV 的 `claimRef` 字段设置为 PVC。
   - 更新 PV 的 `claimRef` 字段示例：
     ```yaml
     claimRef:
       name: example-pvc
       namespace: default
     ```

4. **状态更新**：
   - PVC 的状态从 `Pending` 更新为 `Bound`，表示 PVC 已成功绑定到 PV。
   - PV 的状态也会更新，以反映其已经被绑定的状态。

### 动态存储分配

除了静态创建 PV 并绑定 PVC 外，Kubernetes 还支持动态存储分配。通过 StorageClass 和动态卷提供者（如 CSI 驱动），PVC 可以动态创建和绑定 PV，而无需管理员预先创建 PV。

示例 PVC 动态分配配置文件：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  storageClassName: my-storage-class
```

### 总结

- **绑定过程**：
  - PVC 提交请求后，Kubernetes 控制器根据 PVC 的需求在集群中查找匹配的 PV。
  - 匹配成功后，控制器会将 PV 和 PVC 绑定，更新两者的状态。

- **查看状态**：
  - 使用 `kubectl get pv` 和 `kubectl get pvc` 查看 PV 和 PVC 的状态，确认绑定是否成功。

- **动态存储分配**：
  - 通过 StorageClass，可以实现 PVC 的动态存储分配，无需管理员预先创建 PV。

这种机制使得 Kubernetes 中的持久化存储管理灵活高效，支持多种存储后端，并通过声明式的方式简化了存储资源的分配和管理。

### storage class 动态存储
