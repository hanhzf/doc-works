## admission controller

在 Kubernetes 中，API Server 的 Admission Control 是一个处理所有 API 请求的机制，它在请求被持久化到 etcd 之前进行额外的验证、修改或拒绝。Admission Controllers 是一组插件，它们在资源对象被创建、更新或删除时，提供了额外的安全性和策略执行。

### Admission Control 的作用

Admission Control 插件在以下几个方面起到关键作用：

1. **验证和限制**：确保请求符合集群策略和限制。例如，防止用户在特定命名空间中创建 Pod。
2. **资源修改**：自动向资源对象添加或修改特定字段。例如，向 Pod 添加默认的资源限制或调度策略。
3. **安全性**：执行安全策略，例如防止创建特权 Pod 或添加安全上下文。

### Admission Controller 的类型

Kubernetes 提供了多种 Admission Controller 插件，每个插件有特定的功能。以下是一些常见的 Admission Controller 插件：

1. **NamespaceLifecycle**：在命名空间被删除时阻止新对象的创建，并且在创建资源之前确保命名空间存在。
2. **LimitRanger**：强制执行 Pod 和 Container 的资源限制（如 CPU 和内存限制）。
3. **ServiceAccount**：自动为 Pod 分配 Service Account。
4. **DefaultStorageClass**：为没有指定 StorageClass 的 PersistentVolumeClaim 分配默认的 StorageClass。
5. **ResourceQuota**：确保命名空间内的资源使用不会超过分配的配额。
6. **PodSecurityPolicy**：确保 Pod 符合 Pod 安全策略。
7. **MutatingAdmissionWebhook**：使用 Webhook 修改传入的对象。
8. **ValidatingAdmissionWebhook**：使用 Webhook 验证传入的对象。

### Admission Controller 的工作流程

1. **请求处理**：当用户通过 kubectl 或 API 客户端发送请求时，API Server 接收该请求。
2. **认证和授权**：API Server 首先对请求进行认证和授权。
3. **执行 Admission Controller**：在请求被持久化到 etcd 之前，API Server 依次执行所有启用的 Admission Controller 插件。每个插件可以选择接受、拒绝或修改请求。
4. **持久化**：如果所有 Admission Controller 插件都接受请求，API Server 会将修改后的对象持久化到 etcd。

### 启用和配置 Admission Controller

Admission Controller 插件可以通过 API Server 的启动参数启用和配置。例如，可以在 API Server 的启动参数中指定启用哪些插件：

```bash
kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,PodSecurityPolicy,MutatingAdmissionWebhook,ValidatingAdmissionWebhook
```

### 使用 Webhook 的 Admission Controller

Webhook Admission Controller 提供了更大的灵活性，允许管理员编写自定义逻辑来验证或修改请求。Webhook 有两种类型：

1. **MutatingAdmissionWebhook**：用于修改传入的对象。
2. **ValidatingAdmissionWebhook**：用于验证传入的对象。

#### 示例：配置 MutatingAdmissionWebhook

1. **Webhook 配置**：创建 MutatingWebhookConfiguration 资源，定义 Webhook 的端点和规则。

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: example-mutating-webhook
webhooks:
  - name: webhook.example.com
    clientConfig:
      service:
        name: example-webhook-service
        namespace: default
        path: "/mutate"
      caBundle: <base64-encoded-CA-cert>
    rules:
      - operations: ["CREATE", "UPDATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
```

2. **Webhook 服务**：部署一个 Webhook 服务来处理请求。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: example-webhook-service
  namespace: default
spec:
  ports:
    - port: 443
      targetPort: 8443
  selector:
    app: example-webhook
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-webhook
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: example-webhook
  template:
    metadata:
      labels:
        app: example-webhook
    spec:
      containers:
        - name: webhook
          image: example/webhook:latest
          ports:
            - containerPort: 8443
          volumeMounts:
            - name: webhook-certs
              mountPath: "/etc/webhook/certs"
      volumes:
        - name: webhook-certs
          secret:
            secretName: example-webhook-certs
```

### 总结

- **Admission Control** 是 Kubernetes API Server 处理请求时的一个关键机制，用于在资源对象被持久化到 etcd 之前进行验证、修改或拒绝。
- **常见的 Admission Controller 插件** 包括 NamespaceLifecycle、LimitRanger、ServiceAccount、DefaultStorageClass、ResourceQuota、PodSecurityPolicy、MutatingAdmissionWebhook 和 ValidatingAdmissionWebhook。
- **启用和配置** Admission Controller 可以通过 API Server 的启动参数进行。
- **Webhook Admission Controller** 提供了更大的灵活性，允许管理员编写自定义逻辑来处理请求。

通过 Admission Control，Kubernetes 提供了一种强大的机制来增强集群的安全性和策略执行。

### 典型场景

在生产环境中，`ValidatingAdmissionWebhook` 和 `MutatingAdmissionWebhook` 被广泛用于实施和增强 Kubernetes 集群的安全性、策略执行和自动化配置。以下是这两种 Webhook 在生产中的一些典型使用场景：

### MutatingAdmissionWebhook 典型使用场景

1. **自动注入 Sidecar 容器**：
   - 例如，使用 Istio 等服务网格时，会自动向每个 Pod 注入 Sidecar 容器，以实现流量管理和监控。

2. **设置默认值**：
   - 自动为没有指定资源请求和限制的 Pod 设置默认的 CPU 和内存请求和限制。
   - 为没有指定存储类的 PersistentVolumeClaim (PVC) 设置默认的存储类。

3. **添加标准标签和注解**：
   - 向所有新创建的 Pod 添加标准标签和注解，以便于监控和管理。例如，添加 `team` 标签、`environment` 标签等。

4. **安全增强**：
   - 自动为 Pod 添加安全上下文（Security Context），如非特权模式、只读文件系统等。

#### 示例：自动注入 Sidecar 容器

以下是一个 MutatingAdmissionWebhook 的配置示例，用于自动注入一个 Sidecar 容器：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector-webhook
webhooks:
  - name: sidecar-injector.example.com
    clientConfig:
      service:
        name: sidecar-injector
        namespace: default
        path: "/mutate"
      caBundle: <base64-encoded-CA-cert>
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1", "v1beta1"]
    sideEffects: None
```

Webhook 服务实现示例：

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"

    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
)

func mutatePods(w http.ResponseWriter, r *http.Request) {
    admissionReview := admissionv1.AdmissionReview{}
    if err := json.NewDecoder(r.Body).Decode(&admissionReview); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    pod := corev1.Pod{}
    if err := json.Unmarshal(admissionReview.Request.Object.Raw, &pod); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Define the sidecar container
    sidecar := corev1.Container{
        Name:  "sidecar",
        Image: "example/sidecar:latest",
    }

    // Inject the sidecar container
    pod.Spec.Containers = append(pod.Spec.Containers, sidecar)

    patchBytes, err := json.Marshal([]map[string]interface{}{
        {"op": "add", "path": "/spec/containers/-", "value": sidecar},
    })
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    admissionResponse := admissionv1.AdmissionResponse{
        Allowed: true,
        Patch:   patchBytes,
        PatchType: func() *admissionv1.PatchType {
            pt := admissionv1.PatchTypeJSONPatch
            return &pt
        }(),
    }

    admissionReview.Response = &admissionResponse
    respBytes, err := json.Marshal(admissionReview)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(respBytes)
}

func main() {
    http.HandleFunc("/mutate", mutatePods)
    fmt.Println("Starting Sidecar Injector Webhook Server...")
    http.ListenAndServeTLS(":8443", "/etc/webhook/certs/tls.crt", "/etc/webhook/certs/tls.key", nil)
}
```

### ValidatingAdmissionWebhook 典型使用场景

1. **实施安全策略**：
   - 禁止特权容器的创建，确保 Pod 不使用特权模式。
   - 验证 Pod 是否具有安全的运行时配置，如禁止使用特定的卷类型（如 hostPath）。

2. **强制资源配额和限制**：
   - 确保所有 Pod 都设置了资源请求和限制，以防止资源滥用。
   - 验证 PVC 的请求是否符合存储策略。

3. **命名约定和标签约定**：
   - 确保资源名称符合公司命名约定。
   - 验证所有资源是否具有必需的标签，如 `owner`、`team` 等。

4. **多租户隔离**：
   - 确保不同租户的资源不会相互干扰，例如，禁止租户 A 的 Pod 访问租户 B 的命名空间。

#### 示例：禁止特权容器

以下是一个 ValidatingAdmissionWebhook 的配置示例，用于禁止创建特权容器：

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: privileged-pod-validator
webhooks:
  - name: privileged-pod-validator.example.com
    clientConfig:
      service:
        name: privileged-pod-validator
        namespace: default
        path: "/validate"
      caBundle: <base64-encoded-CA-cert>
    rules:
      - operations: ["CREATE"]
        apiGroups: [""]
        apiVersions: ["v1"]
        resources: ["pods"]
    admissionReviewVersions: ["v1", "v1beta1"]
    sideEffects: None
```

Webhook 服务实现示例：

```go
package main

import (
    "encoding/json"
    "fmt"
    "net/http"

    admissionv1 "k8s.io/api/admission/v1"
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

func validatePods(w http.ResponseWriter, r *http.Request) {
    admissionReview := admissionv1.AdmissionReview{}
    if err := json.NewDecoder(r.Body).Decode(&admissionReview); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    pod := corev1.Pod{}
    if err := json.Unmarshal(admissionReview.Request.Object.Raw, &pod); err != nil {
        http.Error(w, err.Error(), http.StatusBadRequest)
        return
    }

    // Check if the Pod is privileged
    isPrivileged := false
    for _, container := range pod.Spec.Containers {
        if container.SecurityContext != nil && container.SecurityContext.Privileged != nil && *container.SecurityContext.Privileged {
            isPrivileged = true
            break
        }
    }

    admissionResponse := admissionv1.AdmissionResponse{
        Allowed: !isPrivileged,
    }

    if isPrivileged {
        admissionResponse.Result = &metav1.Status{
            Message: "Privileged Pods are not allowed",
        }
    }

    admissionReview.Response = &admissionResponse
    respBytes, err := json.Marshal(admissionReview)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
        return
    }

    w.Header().Set("Content-Type", "application/json")
    w.Write(respBytes)
}

func main() {
    http.HandleFunc("/validate", validatePods)
    fmt.Println("Starting Privileged Pod Validator Webhook Server...")
    http.ListenAndServeTLS(":8443", "/etc/webhook/certs/tls.crt", "/etc/webhook/certs/tls.key", nil)
}
```

### 总结

- **MutatingAdmissionWebhook** 典型使用场景：
  - 自动注入 Sidecar 容器。
  - 设置默认值。
  - 添加标准标签和注解。
  - 增强安全性。

- **ValidatingAdmissionWebhook** 典型使用场景：
  - 实施安全策略。
  - 强制资源配额和限制。
  - 确保命名和标签约定。
  - 多租户隔离。

通过这些 Webhook，可以在 Kubernetes 集群中实施细粒度的策略和自动化配置，增强集群的安全性和可管理性。
