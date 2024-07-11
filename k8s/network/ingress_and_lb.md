## ingress controller
`Ingress` 是一种 Kubernetes 资源，用于管理外部 HTTP 和 HTTPS 流量进入 Kubernetes 集群内的服务。它通过 `Ingress Controller` 将外部流量路由到集群内部的服务。为了理解这个过程，我们需要了解 `Ingress Controller` 的工作原理及其与其他 Kubernetes 组件的交互方式。

### Ingress 的工作原理

1. **Ingress Controller 部署**：
   - `Ingress Controller` 是一个运行在 Kubernetes 集群中的 Pod，负责监控 `Ingress` 资源并根据这些资源的配置动态配置反向代理服务器（如 NGINX、Traefik）。
   - `Ingress Controller` 通常会部署为一个或多个副本的 Deployment。

2. **外部流量进入集群**：
   - 要将外部流量引导到 `Ingress Controller`，需要将 `Ingress Controller` 暴露在外部网络上。通常有以下几种方法：
     - **NodePort**：将 `Ingress Controller` 服务配置为 `NodePort`，从而在每个节点上打开一个静态端口。
     - **LoadBalancer**：在云环境中，将 `Ingress Controller` 服务配置为 `LoadBalancer`，通过云提供商的负载均衡器暴露。
     - **HostPort**：直接在节点的某个端口上监听（不推荐，因端口冲突和管理复杂）。

3. **路由规则配置**：
   - 使用 `Ingress` 资源定义域名和路径的路由规则。`Ingress Controller` 读取这些规则并配置反向代理服务器，以便将流量正确路由到目标服务。
   
4. **将流量路由到服务**：
   - `Ingress Controller` 根据配置的路由规则将外部请求转发到 Kubernetes 集群内的服务。具体的转发方式由 `Ingress` 资源中的规则和 `Ingress Controller` 的实现决定。

### 通过 `LoadBalancer` 方式实现

在云环境中，通过 `LoadBalancer` 服务将 `Ingress Controller` 暴露到外部网络：

1. **创建 `LoadBalancer` 服务**：
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: ingress-nginx
     namespace: ingress-nginx
   spec:
     type: LoadBalancer
     ports:
       - port: 80
         targetPort: 80
       - port: 443
         targetPort: 443
     selector:
       app: ingress-nginx
   ```

2. **部署 `Ingress Controller`**：
   ```yaml
   apiVersion: apps/v1
   kind: Deployment
   metadata:
     name: ingress-nginx
     namespace: ingress-nginx
   spec:
     replicas: 1
     selector:
       matchLabels:
         app: ingress-nginx
     template:
       metadata:
         labels:
           app: ingress-nginx
       spec:
         containers:
         - name: nginx-ingress-controller
           image: quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
           args:
             - /nginx-ingress-controller
             - --configmap=$(POD_NAMESPACE)/nginx-configuration
           env:
             - name: POD_NAME
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.name
             - name: POD_NAMESPACE
               valueFrom:
                 fieldRef:
                   fieldPath: metadata.namespace
           ports:
             - name: http
               containerPort: 80
             - name: https
               containerPort: 443
   ```

3. **创建 `Ingress` 资源**：
   ```yaml
   apiVersion: networking.k8s.io/v1
   kind: Ingress
   metadata:
     name: example-ingress
     namespace: default
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

### 通过 `NodePort` 方式实现

在私有化部署中，可以通过 `NodePort` 暴露 `Ingress Controller`：

1. **创建 `NodePort` 服务**：
   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: ingress-nginx
     namespace: ingress-nginx
   spec:
     type: NodePort
     ports:
       - port: 80
         targetPort: 80
         nodePort: 30080
       - port: 443
         targetPort: 443
         nodePort: 30443
     selector:
       app: ingress-nginx
   ```

2. **部署 `Ingress Controller` 和 `Ingress` 资源**（同上）。

### Ingress 如何将外部流量路由到集群内部

1. **外部请求到达节点**：
   - 当外部请求到达集群节点时，负载均衡器或 `NodePort` 将流量转发到运行 `Ingress Controller` 的节点。

2. **Ingress Controller 处理请求**：
   - `Ingress Controller` 接收请求，并根据配置的 `Ingress` 资源规则决定将流量转发到哪个服务。

3. **流量转发到服务**：
   - `Ingress Controller` 将请求转发到目标服务的 `ClusterIP`，然后服务再将请求路由到对应的 Pod。

### 总结

- **`Ingress`**：提供 HTTP/HTTPS 路由，通过 `Ingress Controller` 实现复杂的路由和负载均衡。
- **`LoadBalancer` 和 `NodePort`**：用于将 `Ingress Controller` 暴露到外部网络。
- **工作流程**：外部请求通过 `LoadBalancer` 或 `NodePort` 进入集群，然后由 `Ingress Controller` 根据路由规则转发到相应的服务。

通过以上机制，`Ingress` 实现了从外部网络到 Kubernetes 集群内服务的流量管理和路由。