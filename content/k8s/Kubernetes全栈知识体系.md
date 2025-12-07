---
title: Kubernetes全栈知识体系
description: 
date: 2025-12-07
categories:
    - 
    - 
---
Kubernetes全栈知识体系

## 第一部分：核心架构与原理

### 1.1 Kubernetes 定义与架构

Kubernetes 是一个开源的容器编排平台，用于自动化部署、扩展和管理容器化应用。

**集群架构（Control Plane - Node Model）**：

- **Master 节点（控制平面）**：
  - **kube-apiserver**：集群统一入口，处理 REST API 请求，负责认证鉴权。
  - **kube-controller-manager**：运行核心控制器（如 Node、Replication 控制器），维持集群期望状态。
  - **kube-scheduler**：负责 Pod 调度，根据资源、亲和性等策略选择最优节点。
  - **etcd**：分布式键值存储，保存集群所有状态数据，是集群的“大脑”。
- **Node 节点（工作节点）**：
  - **kubelet**：节点代理，管理 Pod 生命周期，执行健康检查。
  - **kube-proxy**：维护网络规则（iptables/IPVS），实现 Service 负载均衡。
  - **容器运行时**：如 containerd 或 Docker，负责运行容器。

### 1.2 Pod 核心原理与创建全流程

Pod 是 K8s 的最小调度单元。一个 Pod 创建的完整流程如下：

1. **客户端提交**：kubectl 向 apiserver 发送 REST API 请求。
2. **验证存储**：apiserver 认证鉴权后将 metadata 存入 etcd（状态 Pending）。
3. **调度决策**：scheduler 监听到新 Pod，执行**预选（Predicate）**和**优选（Priority）**算法选择节点。
4. **绑定节点**：apiserver 更新 Pod 的 `nodeName` 字段并写入 etcd。
5. **节点执行**：目标节点的 kubelet 监听到绑定事件。
6. **容器启动**：kubelet 调用 CRI（容器运行时接口）拉取镜像、启动容器；调用 CNI（网络接口）分配 IP。
7. **状态上报**：kubelet 执行健康检查，更新状态为 Running。

------

## 第二部分：生产级集群部署

### 2.1 部署方式选择

- **Minikube**：单节点，适合本地学习。
- **Kubeadm**：官方推荐，支持高可用，适合生产和测试。
- **二进制部署**：手动配置所有组件，极度灵活但维护成本高。

### 2.2 Kubeadm 部署关键步骤（CentOS/Ubuntu）

1. **环境准备**：关闭 Swap、SELinux、防火墙；配置内核参数（开启 IP 转发和 bridge-nf）。

2. **安装运行时**：推荐使用 **containerd**，需配置 `SystemdCgroup = true`。

3. **初始化 Master**：

   Bash

   ```
   kubeadm init --apiserver-advertise-address=<MasterIP> --pod-network-cidr=10.244.0.0/16 --image-repository=registry.aliyuncs.com/google_containers
   ```

4. **安装网络插件**：常用 Calico（适合大规模、支持策略）或 Flannel（简单 overlay）。

5. **节点加入**：在 Worker 节点执行 `kubeadm join` 命令。

------

## 第三部分：工作负载与调度管理

### 3.1 Deployment：无状态应用管理

Deployment 提供副本控制、滚动更新和回滚能力。

**生产级配置示例**：

YAML

```
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1  # 更新时最大不可用数
      maxSurge: 1        # 更新时最大激增数
  template:
    spec:
      affinity:          # 亲和性调度
        podAntiAffinity: # Pod反亲和（避免单点故障）
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: my-app
              topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        resources:       # 必须配置资源限制
          requests: {memory: "512Mi", cpu: "250m"}
          limits:   {memory: "1Gi", cpu: "500m"}
```

### 3.2 健康检查（Probes）

K8s 提供三种探针，生产环境建议全部配置：

1. **LivenessProbe（存活探针）**：检测容器是否死锁。失败则**重启容器**。
2. **ReadinessProbe（就绪探针）**：检测服务是否可用。失败则**从 Service Endpoints 移除**，不接收流量。
3. **StartupProbe（启动探针）**：保护启动慢的应用。成功前抑制其他探针，避免启动中被误杀。

### 3.3 其他工作负载

- **StatefulSet**：管理有状态应用（如 MySQL），提供稳定网络标识（web-0, web-1）和有序部署。
- **DaemonSet**：在每个节点运行一个副本（如日志采集 Fluentd、监控 NodeExporter）。

------

## 第四部分：存储体系

### 4.1 存储分层概念

- **Volume**：Pod 内部存储，随 Pod 生命周期消亡。
- **PV (PersistentVolume)**：集群级资源，由管理员创建。
- **PVC (PersistentVolumeClaim)**：用户对存储的申请请求。
- **StorageClass**：定义存储类型，实现动态自动创建 PV。

### 4.2 常用存储类型实战

1. **emptyDir**：

   - **特点**：临时存储，Pod 删除数据即丢失。
   - **场景**：缓存、日志收集 sidecar 共享数据。

2. **hostPath**：

   - **特点**：挂载节点本地文件系统。
   - **风险**：Pod 漂移后数据丢失，存在安全风险。
   - **场景**：监控、日志采集类 DaemonSet。

3. **NFS（网络存储）**：

   - **实战**：需在 Master 部署 NFS Server，Node 安装 `nfs-utils`。
   - **PV 配置**：

   YAML

   ```
   apiVersion: v1
   kind: PersistentVolume
   spec:
     capacity: {storage: 1Gi}
     accessModes: [ReadWriteMany] # 支持多节点读写
     nfs:
       path: /nfs/data
       server: 192.168.200.100
   ```

------

## 第五部分：网络与服务暴露

### 5.1 Service 类型对比

| **类型**         | **描述**                       | **适用场景**           |
| ---------------- | ------------------------------ | ---------------------- |
| **ClusterIP**    | 默认，仅集群内访问             | 内部微服务调用         |
| **NodePort**     | 开放节点静态端口 (30000-32767) | 开发测试，非 HTTP 业务 |
| **LoadBalancer** | 需云厂商支持，分配公网 IP      | 公有云生产环境         |
| **ExternalName** | 映射到外部 DNS                 | 访问集群外服务         |

### 5.2 Ingress（统一入口）

Ingress 是 HTTP/HTTPS 的路由规则，需配合 Ingress Controller（如 Nginx）使用。

**生产级 Nginx Ingress 部署**：

- **模式**：建议使用 `hostNetwork: true`，直接监听节点 80/443 端口，减少 NAT 性能损耗。

- **功能**：支持 URL 重写、SSL 卸载、限流、黑白名单。

- **YAML 示例**：

  YAML

  ```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    annotations:
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
  spec:
    rules:
    - host: app.example.com
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service: {name: my-svc, port: {number: 80}}
  ```

------

## 第六部分：配置管理与安全

### 6.1 ConfigMap 与 Secret

- **ConfigMap**：明文存储非敏感配置（如配置文件、环境变量）。
- **Secret**：Base64 编码存储敏感信息（密码、证书）。
  - **最佳实践**：推荐使用 **Volume 挂载**方式使用 Secret，支持**热更新**（kubelet 周期性同步），而环境变量方式必须重启 Pod。

### 6.2 RBAC 权限控制

- **Role/ClusterRole**：定义权限集合（Rules）。
- **RoleBinding/ClusterBinding**：将角色绑定到用户或 ServiceAccount。

------

## 第七部分：系统化排错与诊断

### 7.1 核心排错命令三剑客

1. **`kubectl get pods`**：查看状态（Pending/CrashLoopBackOff）。
2. **`kubectl describe pod `**：**最关键一步**，查看 Events 事件（调度失败、镜像拉取失败、挂载失败）。
3. **`kubectl logs `**：查看应用内部日志。
   - 若容器频繁重启，使用 `kubectl logs -p ` 查看**上一次**崩溃的日志。

### 7.2 常见状态故障排查

| **状态**             | **常见原因**                     | **排查与解决**                                               |
| -------------------- | -------------------------------- | ------------------------------------------------------------ |
| **Pending**          | 资源不足、污点限制、PVC未绑定    | 检查 `describe` 中的 Events；检查节点资源；检查 StorageClass。 |
| **ImagePullBackOff** | 镜像名错误、认证失败、网络问题   | 检查镜像名称；检查 Secret 是否配置正确；手动在节点 docker pull 测试。 |
| **CrashLoopBackOff** | 应用启动报错、配置错误、探针失败 | 查看 `logs` 和 `logs --previous`；检查 ConfigMap 挂载；调整探针阈值。 |
| **NotReady**         | 就绪探针失败                     | 检查应用端口是否监听；检查 ReadinessProbe 路径是否正确。     |
| **Terminating**      | finalizer 卡住、挂载无法卸载     | 检查存储连接；强制删除：`kubectl delete pod  --force --grace-period=0`。 |

### 7.3 网络与 Service 排错

1. **检查 Endpoints**：`kubectl get endpoints `，若为空，说明 Pod 标签未匹配或 Pod 未就绪。
2. **测试连通性**：使用 `busybox` 容器在集群内 `curl :`。
3. **检查 DNS**：`nslookup `，检查 CoreDNS 状态。

------

## 第八部分：生产环境最佳实践清单

1. **资源管理**：所有 Pod 必须设置 request（调度保障）和 limits（防止 OOM）。
2. **高可用**：
   - Master 节点至少 3 个。
   - 关键应用多副本 + `podAntiAffinity` 强制反亲和。
3. **安全性**：
   - 镜像不使用 `:latest` 标签。
   - 启用 RBAC，遵循最小权限原则。
   - 敏感信息必须使用 Secret，甚至结合 Vault。
4. **监控运维**：
   - 部署 Prometheus + Grafana 监控集群和应用指标。
   - 部署 EFK（Elasticsearch, Fluentd, Kibana）统一收集日志。
   - 定期备份 etcd 数据。

