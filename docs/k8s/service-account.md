## service account
在 Kubernetes 中，Service Account 是一种特殊类型的账户，用于在集群内的 Pod 与 Kubernetes API 之间进行身份认证。Service Account 提供了一种安全机制，使得运行在 Pod 中的应用程序能够以特定的身份访问 Kubernetes API，并且可以定义不同权限的访问控制。

### Service Account 的主要作用

1. **身份验证**：
   - Service Account 用于在 Pod 与 Kubernetes API 之间进行身份验证，确保只有授权的 Pod 可以访问特定的 API 资源。

2. **权限管理**：
   - 通过关联 Role 或 ClusterRole，Service Account 可以被赋予特定的权限，这样可以控制 Pod 对 Kubernetes API 的访问范围和能力。

3. **隔离与安全**：
   - 使用不同的 Service Account，可以对不同的应用或组件进行隔离和权限控制，增强集群的安全性。

### 默认 Service Account

每个 Kubernetes 命名空间都会自动创建一个名为 `default` 的 Service Account。所有没有指定特定 Service Account 的 Pod 都会使用 `default` Service Account。

### 创建和使用 Service Account

#### 1. 创建 Service Account

使用 `kubectl` 命令创建一个新的 Service Account：

```bash
kubectl create serviceaccount my-service-account
```

#### 2. 在 Pod 中使用 Service Account

要在 Pod 中使用特定的 Service Account，需要在 Pod 的 YAML 配置文件中指定 `serviceAccountName` 字段：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  serviceAccountName: my-service-account
  containers:
  - name: my-container
    image: my-image
```

#### 3. 绑定角色权限

为了使 Service Account 能够访问特定的 Kubernetes API 资源，需要将它与 Role 或 ClusterRole 绑定。下面是一个示例，展示如何为 Service Account 绑定权限：

创建一个 Role：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```

创建一个 RoleBinding，将 Role 绑定到 Service Account：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: ServiceAccount
  name: my-service-account
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

### 使用 Service Account 进行身份验证

当一个 Pod 使用 Service Account 运行时，Kubernetes 会在 Pod 的文件系统中挂载一个包含该 Service Account 令牌的文件，默认路径为 `/var/run/secrets/kubernetes.io/serviceaccount/token`。应用程序可以读取这个令牌来与 Kubernetes API 进行身份验证。

### 总结

- **Service Account** 是 Kubernetes 中的一种机制，用于在 Pod 与 Kubernetes API 之间进行身份验证和权限控制。
- 每个命名空间都有一个默认的 `default` Service Account。
- 可以创建自定义的 Service Account，并在 Pod 配置中指定。
- 通过绑定 Role 或 ClusterRole，可以为 Service Account 配置访问权限。
- Pod 中的应用程序可以使用挂载的 Service Account 令牌来与 Kubernetes API 进行安全通信。

这种机制为 Kubernetes 集群内的应用程序提供了安全、灵活的身份验证和权限管理方式。

## service account token

### 什么是 Service Account Token？

在 Kubernetes 中，Service Account Token 是一种基于 JWT（JSON Web Token）的令牌，用于在 Kubernetes 集群内部进行身份验证。每个 Service Account 都会自动生成一个关联的 Token，并以 Secret 的形式存储在集群中。这个 Token 可以挂载到 Pod 中，使运行在 Pod 内的应用程序能够以该 Service Account 的身份与 Kubernetes API 进行交互。

### Service Account Token 的作用

1. **身份验证**：Service Account Token 用于在 Kubernetes API Server 中进行身份验证，确保请求来自于集群内部的授权实体。
2. **权限控制**：结合 Role-Based Access Control（RBAC），Service Account Token 确定该请求具备的权限，控制对 Kubernetes 资源的访问范围和操作权限。
3. **安全通信**：Service Account Token 允许 Pod 内的应用程序安全地与 Kubernetes API 进行通信，不需要暴露集群外部的认证机制。

### 如何生成 Service Account Token

当你创建一个 Service Account 时，Kubernetes 会自动生成一个关联的 Token 并以 Secret 的形式存储。以下是如何生成和使用 Service Account Token 的步骤：

1. **创建一个 Service Account**

   ```bash
   kubectl create serviceaccount my-service-account
   ```

2. **查看生成的 Secret**

   每个 Service Account 会自动创建一个包含 Token 的 Secret。可以使用以下命令查看：

   ```bash
   kubectl get secrets
   ```

   找到 `my-service-account-token-xxxxx` 的 Secret。

3. **获取 Token**

   使用以下命令获取 Service Account Token：

   ```bash
   kubectl get secret my-service-account-token-xxxxx -o jsonpath='{.data.token}' | base64 --decode
   ```

### 使用场景

1. **Pod 与 Kubernetes API 的交互**：在 Pod 中运行的应用程序需要与 Kubernetes API 进行交互，例如获取 Pod 信息、监控节点状态或动态调整资源配置。通过挂载 Service Account Token，可以安全地进行这些操作。

   示例：在 Pod 中使用 Service Account

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: example-pod
   spec:
     serviceAccountName: my-service-account
     containers:
     - name: example-container
       image: example-image
   ```

2. **CI/CD 管道**：在 CI/CD 工具（如 Jenkins、GitLab CI 等）中使用 Service Account Token，可以自动化 Kubernetes 资源的管理，例如部署、更新或监控应用。

3. **自定义控制器**：开发自定义 Kubernetes 控制器或 Operator 时，使用 Service Account Token 可以确保控制器具有正确的权限与集群 API 进行交互。

### 具体使用示例

假设我们需要编写一个脚本来访问 Kubernetes API 并获取节点列表。可以在 Pod 中挂载 Service Account Token，并使用该 Token 进行身份验证：

1. **创建 Pod 并挂载 Token**

   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
     name: api-access-pod
   spec:
     serviceAccountName: my-service-account
     containers:
     - name: api-access-container
       image: appropriate/curl
       command: ["sh", "-c", "while true; do sleep 3600; done"]
   ```

2. **进入 Pod 并使用 Token 访问 API**

   ```bash
   kubectl exec -it api-access-pod -- /bin/sh
   ```

   在 Pod 内，读取挂载的 Token：

   ```bash
   TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
   ```

   使用 Token 访问 Kubernetes API：

   ```bash
   curl -H "Authorization: Bearer $TOKEN" -k https://kubernetes.default.svc.cluster.local/api/v1/nodes
   ```

### 总结

- **Service Account Token** 是 Kubernetes 内部用于身份验证的 JWT 令牌。
- **作用**：在 Kubernetes 集群内部进行身份验证和权限控制。
- **生成**：自动生成并存储在 Secret 中，当创建 Service Account 时生成。
- **使用场景**：Pod 与 Kubernetes API 交互、CI/CD 管道、自定义控制器等。

通过使用 Service Account Token，可以确保 Kubernetes 集群内部的应用程序以安全和受控的方式与 Kubernetes API 进行交互。