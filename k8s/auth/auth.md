Kubernetes 的认证和授权体系提供了强大的机制来管理和控制对集群资源的访问。它主要包括身份认证（Authentication）和权限授权（Authorization）两个部分。

### 1. 身份认证（Authentication）

身份认证是指验证访问者的身份。Kubernetes 支持多种身份认证方式：

#### a. 用户认证

- **X.509 客户端证书**：集群管理员为用户创建证书，并在 kube-apiserver 中配置 CA 证书。
- **静态 Token 文件**：在 kube-apiserver 启动时指定一个包含用户 Token 的文件。
- **Bootstrap Token**：用于引导节点加入集群，通常在 kubeadm 中使用。
- **静态密码文件**：不推荐使用，将用户名和密码存储在文件中供 kube-apiserver 使用。
- **OIDC（OpenID Connect）**：通过配置 OIDC 提供者（如 Google、GitHub）实现身份认证。
- **Webhook**：将身份认证请求转发到外部服务进行验证。

#### b. Service Account

- **Service Account Token**：用于在集群内部的 Pod 之间进行身份验证。每个 Pod 都可以通过挂载的 Service Account Token 文件与 Kubernetes API 进行交互。

### 2. 权限授权（Authorization）

权限授权是指决定经过身份认证的用户是否有权限执行某个操作。Kubernetes 提供了多种授权方式：

#### a. Node Authorization

- 专门用于管理节点的权限，确保节点只能访问自身的资源。

#### b. ABAC（Attribute-Based Access Control）

- 通过策略文件根据请求的属性进行访问控制。灵活但复杂，难以管理。

#### c. RBAC（Role-Based Access Control）

- 基于角色的访问控制，通过定义角色和角色绑定来管理权限。RBAC 是 Kubernetes 中最常用的授权方式。
  - **Role**：定义一组权限，适用于命名空间级别的资源。
  - **ClusterRole**：定义一组权限，适用于集群级别的资源。
  - **RoleBinding**：将 Role 绑定到用户、组或 Service Account，适用于命名空间级别。
  - **ClusterRoleBinding**：将 ClusterRole 绑定到用户、组或 Service Account，适用于集群级别。

#### d. Webhook Authorization

- 将授权请求转发到外部服务进行处理，适用于复杂的自定义授权逻辑。

### 3. 示例配置

#### a. 创建用户认证

假设使用 X.509 客户端证书认证，为用户创建证书：

```bash
openssl genrsa -out user.key 2048
openssl req -new -key user.key -out user.csr -subj "/CN=username/O=groupname"
openssl x509 -req -in user.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out user.crt -days 365
```

将生成的 `user.crt` 和 `user.key` 文件配置到 kubeconfig 中：

```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /path/to/ca.crt
    server: https://kubernetes.example.com
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: username
  name: username@kubernetes
current-context: username@kubernetes
users:
- name: username
  user:
    client-certificate: /path/to/user.crt
    client-key: /path/to/user.key
```

#### b. 创建 RBAC 授权

创建一个 ClusterRole，授予查看所有 Pod 的权限：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view-pods
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
```

创建一个 ClusterRoleBinding，将 ClusterRole 绑定到用户：

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: view-pods-binding
subjects:
- kind: User
  name: username
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view-pods
  apiGroup: rbac.authorization.k8s.io
```

### 总结

- **身份认证**：验证访问者身份，支持多种方式，如 X.509 客户端证书、OIDC、Webhook 等。
- **权限授权**：决定经过身份认证的用户是否有权限执行操作，支持多种方式，如 RBAC、ABAC、Webhook 等。
- **RBAC** 是 Kubernetes 中最常用的授权方式，通过定义角色和角色绑定来管理权限。

Kubernetes 的认证和授权体系提供了灵活而强大的访问控制机制，确保集群资源的安全性和隔离性。