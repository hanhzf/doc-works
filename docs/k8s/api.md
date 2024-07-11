Kubernetes 的 API 是其核心功能之一，通过 API，用户和系统可以与 Kubernetes 集群进行交互。Kubernetes API 的版本通常遵循 `apiVersion` 字段格式，如 `v1`、`apps/v1`、`batch/v1` 等，不同的 API 组和版本提供不同的功能和资源。以下是一些常用 API 组及其功能、作用和示例：

### Core API (`v1`)

- **功能**：核心 API 组，包含 Kubernetes 最基础的资源对象。
- **常见资源**：
  - Pod：表示集群中运行的容器。
  - Service：定义如何在集群内部和外部访问 Pod。
  - ConfigMap：存储非机密配置数据。
  - Secret：存储敏感数据（如密码、OAuth 令牌等）。
  - PersistentVolume (PV) 和 PersistentVolumeClaim (PVC)：管理存储卷。
  
- **示例**：
  ```yaml
  apiVersion: v1
  kind: Pod
  metadata:
    name: my-pod
  spec:
    containers:
    - name: my-container
      image: nginx
  ```

### Apps API (`apps/v1`)

- **功能**：用于管理更复杂的应用部署和运行时管理。
- **常见资源**：
  - Deployment：管理 Pod 的副本集，支持滚动更新。
  - StatefulSet：管理有状态应用。
  - DaemonSet：在每个节点上运行一个 Pod。
  - ReplicaSet：确保指定数量的 Pod 副本运行。

- **示例**：
  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: my-deployment
  spec:
    replicas: 3
    selector:
      matchLabels:
        app: my-app
    template:
      metadata:
        labels:
          app: my-app
      spec:
        containers:
        - name: my-container
          image: nginx
  ```

### Batch API (`batch/v1`)

- **功能**：用于批处理和定时任务。
- **常见资源**：
  - Job：运行一次性任务，直到成功完成。
  - CronJob：定期运行任务，基于 Cron 表达式。

- **示例**：
  ```yaml
  apiVersion: batch/v1
  kind: Job
  metadata:
    name: my-job
  spec:
    template:
      spec:
        containers:
        - name: my-container
          image: busybox
          command: ["echo", "Hello, Kubernetes!"]
        restartPolicy: Never
  ```

### Networking API (`networking.k8s.io/v1`)

- **功能**：管理集群内的网络配置和通信。
- **常见资源**：
  - Ingress：管理外部 HTTP/HTTPS 访问。
  - NetworkPolicy：定义 Pod 之间的网络策略和规则。

- **示例**：
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: my-ingress
  spec:
    rules:
    - host: example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: my-service
              port:
                number: 80
  ```

### Policy API (`policy/v1`)

- **功能**：用于管理集群的安全策略。
- **常见资源**：
  - PodDisruptionBudget：限制自愿的 Pod 中断，以确保一定数量的 Pod 始终可用。

- **示例**：
  ```yaml
  apiVersion: policy/v1
  kind: PodDisruptionBudget
  metadata:
    name: my-pdb
  spec:
    minAvailable: 2
    selector:
      matchLabels:
        app: my-app
  ```

### Rbac API (`rbac.authorization.k8s.io/v1`)

- **功能**：用于管理集群的访问控制和权限。
- **常见资源**：
  - Role：定义在某个命名空间内的权限。
  - ClusterRole：定义在集群范围内的权限。
  - RoleBinding：将 Role 绑定到用户或组。
  - ClusterRoleBinding：将 ClusterRole 绑定到用户或组。

- **示例**：
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

### Custom Resource Definitions (CRD) API (`apiextensions.k8s.io/v1`)

- **功能**：用于定义自定义资源，以扩展 Kubernetes 的功能。
- **常见资源**：
  - CustomResourceDefinition：定义新的自定义资源。

- **示例**：
  ```yaml
  apiVersion: apiextensions.k8s.io/v1
  kind: CustomResourceDefinition
  metadata:
    name: crontabs.stable.example.com
  spec:
    group: stable.example.com
    versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                cronSpec:
                  type: string
                image:
                  type: string
                replicas:
                  type: integer
    scope: Namespaced
    names:
      plural: crontabs
      singular: crontab
      kind: CronTab
      shortNames:
      - ct
  ```

### 总结

- **Core API (`v1`)**：管理基础资源，如 Pod、Service、ConfigMap、Secret、PV、PVC。
- **Apps API (`apps/v1`)**：管理复杂的应用部署，如 Deployment、StatefulSet、DaemonSet、ReplicaSet。
- **Batch API (`batch/v1`)**：管理批处理任务和定时任务，如 Job、CronJob。
- **Networking API (`networking.k8s.io/v1`)**：管理网络配置，如 Ingress、NetworkPolicy。
- **Policy API (`policy/v1`)**：管理安全策略，如 PodDisruptionBudget。
- **Rbac API (`rbac.authorization.k8s.io/v1`)**：管理访问控制和权限，如 Role、ClusterRole、RoleBinding、ClusterRoleBinding。
- **CRD API (`apiextensions.k8s.io/v1`)**：定义自定义资源，扩展 Kubernetes 功能。

这些 API 版本和资源使 Kubernetes 成为一个强大且灵活的容器编排平台，能够满足各种应用和部署需求。