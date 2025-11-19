---
title: Kubernetes 
description: 
date: 2025-11-19
categories:
    - 
    - 
---
# Kubernetes (K8s) 完整指南

本文档基于 Kubernetes 2024 年版本整理，涵盖从基础概念到生产实践的完整知识体系。

---

## 1. Kubernetes 概述

### 1.1 什么是 Kubernetes

Kubernetes（K8s）是一个开源的容器编排平台，用于自动化部署、扩展和管理容器化应用。它提供了：
- **服务发现与负载均衡**：自动暴露容器，实现流量分发
- **存储编排**：自动挂载存储系统
- **自动部署与回滚**：声明式配置，自动更新应用
- **自动装箱**：根据资源需求智能调度容器
- **自我修复**：自动重启、替换失败的容器
- **密钥与配置管理**：安全地管理敏感信息

### 1.2 核心概念

| 概念            | 说明                                      | 使用场景           |
| --------------- | ----------------------------------------- | ------------------ |
| **Pod**         | 最小部署单元，一个或多个紧密耦合的容器    | 运行应用实例       |
| **Service**     | 定义一组 Pod 的访问策略，提供稳定访问端点 | 服务发现、负载均衡 |
| **Deployment**  | 管理 Pod 的声明式更新，提供副本控制       | 无状态应用部署     |
| **StatefulSet** | 管理有状态应用，提供稳定标识和存储        | 数据库、中间件     |
| **DaemonSet**   | 确保每个节点运行一个 Pod 副本             | 日志采集、监控代理 |
| **ConfigMap**   | 存储非敏感的配置信息                      | 应用配置管理       |
| **Secret**      | 存储敏感信息（Base64 编码）               | 密码、Token、证书  |

### 1.3 架构原理

Kubernetes 采用**控制平面-工作节点**架构：
- **控制平面（Master）**：管理集群状态，做出全局决策
- **工作节点（Node）**：运行应用容器，执行控制平面指令
- **ETCD**：分布式键值存储，保存集群所有状态数据

---

## 2. 部署方式详解

### 2.1 minikube（本地开发）

**适用场景**：个人开发、学习、功能测试

**特点**：
- 单节点伪集群，Master 和 Node 角色合一
- 一键启动：`minikube start`
- 支持多种虚拟化驱动（VirtualBox、Docker、Hyper-V）
- 内置插件系统（Dashboard、Ingress、Metrics-Server）

**快速开始**：
```bash
# 安装后启动
minikube start --driver=docker
# 启用插件
minikube addons enable dashboard
minikube addons enable ingress
# 访问 Dashboard
minikube dashboard
```

**局限性**：
- 无法模拟多节点网络策略
- 资源受限，不支持生产负载
- 部分高级功能（如多可用区）不可用

### 2.2 kubeadm（官方推荐）

**适用场景**：测试环境、生产环境、高可用集群部署

**特点**：
- 自动化部署官方认可的标准集群
- 支持多 Master 节点高可用
- 一键初始化：`kubeadm init`
- 一键加入：`kubeadm join`
- 平滑升级：`kubeadm upgrade`

**优势**：
- 组件版本一致性保障
- 暴露配置细节，便于学习理解
- 社区支持完善，文档丰富
- 适合构建生产级集群

**部署要求**：
- 至少 2 核 CPU，2GB 内存
- 操作系统：Ubuntu 16.04+、CentOS 7+
- 网络互通，无 NAT 阻断

### 2.3 二进制包部署

**适用场景**：高度定制化的企业生产环境、离线环境、深度研究

**特点**：
- 完全手动部署所有组件
- 可深度定制配置参数
- 完全掌控证书体系和组件版本
- 无自动化工具依赖

**优点**：
- 极致灵活性，满足特殊需求
- 深入理解 Kubernetes 内部机制
- 适用于网络隔离环境

**缺点**：
- 部署复杂，易出错
- 维护成本高，升级困难
- 需要高水平运维团队

**核心组件列表**：
```
kube-apiserver
kube-controller-manager
kube-scheduler
kubelet
kube-proxy
etcd
containerd/docker
```

### 2.4 其他部署方式

| 工具            | 特点                   | 适用场景           |
| --------------- | ---------------------- | ------------------ |
| **kops**        | 云原生集群生命周期管理 | AWS/GCP 生产集群   |
| **kubespray**   | Ansible 剧本部署       | 大规模多节点集群   |
| **Rancher**     | 可视化集群管理         | 多集群统一管理     |
| **EKS/GKE/AKS** | 托管 Kubernetes 服务   | 公有云环境，免运维 |

---

## 3. 集群组件深度解析

### 3.1 Master 节点组件

Master 节点是集群的"大脑"，负责管理全局状态和决策。

#### 3.1.1 kube-apiserver
- **作用**：集群统一入口，所有组件通过 REST API 与其通信
- **端口**：6443（HTTPS）
- **特性**：
  - 认证授权（RBAC、Webhook）
  - 数据验证和准入控制
  - 提供 watch 机制，支持组件监听资源变化
  - 水平扩展支持多实例 + 负载均衡

#### 3.1.2 kube-controller-manager
- **作用**：运行各种控制器，维持集群期望状态
- **核心控制器**：
  - **Node Controller**：节点宕机检测，5分钟无响应标记为 NotReady
  - **Replication Controller**：保证 Pod 副本数与 Deployment 定义一致
  - **Endpoint Controller**：维护 Service 的后端 Endpoint 列表
  - **Service Account Controller**：为 Namespace 创建默认 ServiceAccount

#### 3.1.3 kube-scheduler
- **作用**：负责 Pod 调度决策，为未分配的 Pod 选择最优节点
- **调度流程**：
  1. **过滤（Predicates）**：筛选满足条件的节点（资源足够、端口不冲突、容忍污点）
  2. **打分（Priorities）**：对候选节点打分（资源平衡、亲和性、镜像本地性）
- **自定义调度**：支持多调度器，可通过调度扩展器自定义逻辑

#### 3.1.4 cloud-controller-manager（可选）
- **作用**：对接云厂商 API，管理节点、负载均衡、存储卷
- **场景**：在云环境（AWS/GCP/阿里云）中部署时使用

### 3.2 Node 节点组件

Node 节点是工作负载承载者，负责运行容器。

#### 3.2.1 kubelet
- **作用**：节点代理，与 Master 通信并管理 Pod 生命周期
- **核心功能**：
  - 接收 PodSpec，确保容器健康运行
  - 定期向 API Server 注册节点和上报资源使用
  - 执行健康检查（Liveness/Readiness Probe）
  - 管理容器存储卷和网络附加

#### 3.2.2 kube-proxy
- **作用**：实现 Service 网络代理和负载均衡
- **工作模式**：
  - **iptables**（默认）：通过 iptables 规则实现流量转发，性能稳定
  - **ipvs**：内核级负载均衡，性能更高，支持更多调度算法
  - **userspace**：早期模式，已废弃

#### 3.2.3 容器运行时
负责运行容器的底层软件：
- **containerd**：轻量级，CNCF 毕业项目，推荐
- **Docker**：早期主流，现已逐步淘汰
- **CRI-O**：Kubernetes 专用，轻量安全

### 3.3 ETCD 集群

ETCD 是 Kubernetes 的数据存储大脑，保存所有集群状态。

#### 3.3.1 核心特性
- **强一致性**：基于 Raft 算法，保证数据一致性
- **高可用**：奇数节点部署（3/5/7），容忍 (N-1)/2 个节点故障
- **性能**：写入性能影响集群整体响应速度
- **备份**：定期备份至关重要，建议每小时备份一次

#### 3.3.2 数据存储结构
```
/registry/pods/default/nginx-pod
/registry/services/default/nginx-service
/registry/deployments/default/nginx-deployment
/registry/secrets/default/db-password
```

#### 3.3.3 部署建议
- **独立集群**：生产环境建议 ETCD 与 Master 分离部署
- **SSD 存储**：使用 SSD 磁盘提升写入性能
- **监控**：监控 ETCD 延迟、存储空间、Leader 变化
- **备份脚本**：
```bash
#!/bin/bash
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d).db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 3.4 核心 Addons

#### 3.4.1 CoreDNS
- **作用**：集群内部 DNS 服务器，为 Service 提供域名解析
- **解析规则**：`service.namespace.svc.cluster.local`
- **配置**：可通过 ConfigMap 自定义 DNS 转发规则

#### 3.4.2 kube-dns（已废弃）
早期 DNS 方案，现被 CoreDNS 替代

#### 3.4.3 Metrics Server
- **作用**：提供资源使用指标，支持 `kubectl top` 和 HPA
- **部署**：必须安装才能使用自动扩缩容
- `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`

#### 3.4.4 Dashboard
- **作用**：官方 Web UI，可视化集群管理
- **访问**：`kubectl proxy` 后访问 `http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

---

## 4. 核心资源对象详解

### 4.1 Pod（最小程序单元）

**概念**：Pod 是 Kubernetes 的最小可部署和可调度单元，一个 Pod 可以包含一个或多个紧密耦合的容器。

**关键特性**：
- **共享网络**：Pod 内所有容器共享同一个网络命名空间（IP 和端口）
- **共享存储**：通过 Volume 共享存储卷
- **生命周期**：Pod 是临时实体，崩溃后不会被修复，由控制器重建
- **Sidecar 模式**：主容器 + 辅助容器（日志、代理、监控）

**示例配置**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    env:
    - name: NGINX_PORT
      value: "80"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    livenessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 15
      periodSeconds: 10
```

### 4.2 Service（服务发现）

**概念**：Service 定义了一组 Pod 的访问策略，提供稳定的网络端点。

**四种类型**：
1. **ClusterIP**（默认）
   - 集群内部虚拟 IP，仅集群内访问
   - 示例：`kubectl expose deployment/nginx --port=80`

2. **NodePort**
   - 在每个节点开放一个静态端口（30000-32767）
   - 外部可通过 `NodeIP:NodePort` 访问
   - 示例：`kubectl expose deployment/nginx --type=NodePort --port=80`

3. **LoadBalancer**
   - 云厂商负载均衡器自动配置
   - 需要云厂商支持（AWS/GCP/阿里云）
   - 示例：`kubectl expose deployment/nginx --type=LoadBalancer --port=80`

4. **ExternalName**
   - 将 Service 映射到外部 DNS 名称
   - 示例：访问外部数据库服务

**Headless Service**（无头服务）：
```yaml
spec:
  clusterIP: None
```
- 不分配虚拟 IP，直接返回后端 Pod IP 列表
- 适用场景：StatefulSet、需要直接访问 Pod 的场景

**Service 示例**：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: nginx
```

### 4.3 Deployment（无状态应用）

**概念**：Deployment 管理 Pod 的声明式更新，提供副本控制、滚动更新、回滚能力。

**关键功能**：
- **副本管理**：确保指定数量的 Pod 始终运行
- **滚动更新**：逐步替换旧版本 Pod，零停机更新
- **版本回滚**：快速回滚到历史版本
- **扩缩容**：手动或自动调整副本数

**Deployment 示例**：
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

**更新策略**：
- **RollingUpdate**：滚动更新，逐步替换
  - `maxUnavailable`：更新过程中不可用 Pod 的最大数量（默认 25%）
  - `maxSurge`：更新过程中可以超过期望副本数的数量（默认 25%）
- **Recreate**：先删除所有旧 Pod，再创建新 Pod（会导致短暂停机）

### 4.4 StatefulSet（有状态应用）

**概念**：管理有状态应用，为 Pod 提供稳定标识和持久化存储。

**适用场景**：
- 需要稳定网络标识（如 Kafka、Zookeeper）
- 需要持久化存储（如 MySQL、MongoDB）
- 有序部署和扩展（如主从架构）

**特点**：
- **稳定标识**：Pod 名称固定（如 web-0, web-1, web-2）
- **有序部署**：按索引顺序创建和删除 Pod
- **持久存储**：每个 Pod 绑定独立的 PV

**StatefulSet 示例**：
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: mysql-headless
  replicas: 3
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
```

### 4.5 DaemonSet（守护进程集）

**概念**：确保每个（或特定）节点运行一个 Pod 副本。

**适用场景**：
- 日志采集（Fluentd、Filebeat）
- 监控代理（Prometheus Node Exporter）
- 网络插件（Calico、Flannel）
- 安全代理

**DaemonSet 示例**：
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: fluent/fluentd-kubernetes-daemonset:v1-debian-elasticsearch
        env:
        - name: FLUENT_ELASTICSEARCH_HOST
          value: "elasticsearch.logging.svc.cluster.local"
```

### 4.6 Job/CronJob（批处理任务）

**Job**：运行一次性任务，确保任务完成。
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculation
spec:
  template:
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
      restartPolicy: Never
  backoffLimit: 4
```

**CronJob**：定时执行 Job。
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-cron
spec:
  schedule: "0 3 * * *"  # 每天凌晨3点执行
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: mysql:8.0
            command:
            - /bin/sh
            - -c
            - mysqldump -h mysql.default.svc.cluster.local -u root -p$MYSQL_ROOT_PASSWORD mydb > /backup/mydb-$(date +%Y%m%d).sql
            env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: password
          restartPolicy: OnFailure
```

### 4.7 ConfigMap 与 Secret

**ConfigMap**：存储非敏感配置信息。
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "mysql:3306"
  redis_url: "redis:6379"
  config.yaml: |
    server:
      port: 8080
      timeout: 30s
```

**Secret**：存储敏感信息（Base64 编码）。
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  username: YWRtaW4=  # base64 编码
  password: MWYyZDFlMmU2N2Rm  # base64 编码
```

**使用方法**：
1. **环境变量**：
```yaml
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password
```

2. **Volume 挂载**：
```yaml
volumeMounts:
- name: secret-volume
  mountPath: "/etc/secret"
  readOnly: true
volumes:
- name: secret-volume
  secret:
    secretName: db-secret
```

---

## 5. 网络模型

### 5.1 网络基础

**Kubernetes 网络要求**：
- **所有 Pod 互通**：无论节点，所有 Pod 可相互通信
- **所有节点与 Pod 互通**：节点可访问所有 Pod
- **Pod 拥有独立 IP**：每个 Pod 拥有集群范围的唯一 IP

**网络模型实现**：
- **CNI（容器网络接口）**：插件化网络方案
- **Service 网络**：虚拟 IP 和端口代理
- **DNS 服务**：CoreDNS 提供域名解析

### 5.2 CNI 插件对比

| 插件        | 特点                           | 性能 | 适用场景                   |
| ----------- | ------------------------------ | ---- | -------------------------- |
| **Calico**  | BGP 路由，NetworkPolicy 支持   | 高   | 大规模生产环境             |
| **Flannel** | 简单易用，overlay 网络         | 中等 | 中小型集群                 |
| **Cilium**  | eBPF 技术，高性能安全          | 极高 | 对性能和安全性要求高的场景 |
| **Weave**   | 自动加密，易于部署             | 中等 | 开发测试、多云环境         |
| **Antrea**  | VMware 开源，基于 Open vSwitch | 高   | vSphere 环境               |

#### 5.2.1 Calico 配置示例
```yaml
# 下载配置
wget https://docs.projectcalico.org/manifests/calico.yaml

# 修改配置（如需要）
# 编辑 calico.yaml，设置 CALICO_IPV4POOL_CIDR 与集群 pod-network-cidr 一致

# 部署
kubectl apply -f calico.yaml

# 验证
kubectl get pods -n kube-system -l k8s-app=calico-node
```

#### 5.2.2 Flannel 配置示例
```bash
# 部署
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# 配置后重启节点
systemctl restart kubelet
```

### 5.3 Service 网络

**kube-proxy 工作原理**：
1. **iptables 模式**：
   - 为每个 Service 创建 iptables 规则
   - 流量通过 DNAT 转发到后端 Pod
   - 规则数量随服务增多线性增长

2. **IPVS 模式**（推荐）：
   - 内核级负载均衡
   - 支持多种调度算法（rr, lc, dh, sh, sed, nq）
   - 性能更好，规则处理更高效

**开启 IPVS 模式**：
```bash
# 在 kube-proxy ConfigMap 中修改
kubectl edit configmap kube-proxy -n kube-system
# 设置 mode: "ipvs"
# 重启 kube-proxy
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### 5.4 Ingress（HTTP/HTTPS 路由）

**概念**：管理外部访问集群服务的 HTTP/HTTPS 路由。

**常用 Ingress Controller**：
- **NGINX Ingress Controller**：最流行，功能丰富
- **Traefik**：云原生，自动服务发现
- **HAProxy**：高性能，成熟稳定

**Ingress 示例**：
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  tls:
  - hosts:
    - app.example.com
    secretName: tls-secret
```

---

## 6. 存储系统

### 6.1 存储概念

**存储抽象层次**：
```
StorageClass → PersistentVolume → PersistentVolumeClaim → Pod
```

- **StorageClass**：定义存储类型和提供者
- **PersistentVolume (PV)**：集群级存储资源，由管理员创建
- **PersistentVolumeClaim (PVC)**：命名空间级存储请求，由用户创建
- **Pod**：挂载 PVC 使用存储

### 6.2 PV/PVC 详解

**PV 示例**（静态供给）：
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv001
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  hostPath:
    path: /data/pv001
```

**PVC 示例**：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 50Gi
```

**在 Pod 中使用**：
```yaml
spec:
  containers:
  - name: mysql
    volumeMounts:
    - name: data
      mountPath: /var/lib/mysql
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: mysql-data
```

### 6.3 StorageClass（动态供给）

**概念**：自动创建 PV 的模板，支持按需分配存储。

**常用类型**：
- **hostPath**：单节点测试
- **NFS**：多节点共享存储
- **CephFS/RBD**：分布式存储
- **云存储**：AWS EBS、GCP PD、阿里云盘

**NFS StorageClass 示例**：
```yaml
# 部署 NFS-Subdir-External-Provisioner
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
parameters:
  server: nfs-server.example.com
  path: /exports
  archiveOnDelete: "true"
reclaimPolicy: Retain
allowVolumeExpansion: true
mountOptions:
- hard
- nfsvers=4.1
```

### 6.4 常用存储方案对比

| 方案          | 特点                  | 性能 | 适用场景           |
| ------------- | --------------------- | ---- | ------------------ |
| **hostPath**  | 单节点，简单易用      | 高   | 单节点测试         |
| **NFS**       | 多节点共享，成熟稳定  | 中等 | 开发测试、共享数据 |
| **Ceph**      | 分布式，高可用        | 高   | 大规模生产环境     |
| **GlusterFS** | 分布式，弹性扩展      | 中等 | 文件存储场景       |
| **Longhorn**  | Kubernetes 原生，轻量 | 中等 | 中小规模生产环境   |

---

## 7. 安全机制

### 7.1 RBAC（基于角色的访问控制）

**核心概念**：
- **Role/ClusterRole**：定义权限集合
- **RoleBinding/ClusterRoleBinding**：将 Role 绑定到用户/组/ServiceAccount

**Role 示例**（命名空间级）：
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

**ClusterRole 示例**（集群级）：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-admin
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "update", "patch"]
```

**RoleBinding 示例**：
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
- kind: ServiceAccount
  name: default
  namespace: kube-system
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

**内置 ClusterRole**：
- `cluster-admin`：超级管理员
- `admin`：命名空间管理员
- `edit`：可修改资源
- `view`：只读权限

### 7.2 NetworkPolicy（网络策略）

**概念**：定义 Pod 间网络访问规则，实现网络隔离。

**前提**：CNI 插件必须支持 NetworkPolicy（Calico、Cilium 支持）。

**示例：隔离 default 命名空间**：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  policyTypes:
  - Ingress
  - Egress
  podSelector: {}  # 选择所有 Pod
```

**示例：允许特定访问**：
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

### 7.3 PodSecurityPolicy（PSP）

**注意**：PSP 在 Kubernetes v1.25+ 已被移除，改用 PodSecurity Admission。

**替代方案 - PodSecurity 标准**：
```bash
# 为命名空间设置安全等级
kubectl label namespace default pod-security.kubernetes.io/enforce=baseline
kubectl label namespace default pod-security.kubernetes.io/warn=restricted
```

### 7.4 密钥管理最佳实践

1. **使用 Secret 而非 ConfigMap 存储敏感信息**
2. **启用静态加密**：
```bash
# 在 kube-apiserver 配置中启用
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml
```

3. **使用外部密钥管理服务**：
   - **Vault**：HashiCorp 的密钥管理
   - **AWS KMS**：云密钥管理服务
   - **External Secrets Operator**：同步外部密钥到 Kubernetes Secret

---

## 8. 调度机制

### 8.1 调度原理

**调度流程**：
1. **创建 Pod**：Pod 进入 Pending 状态
2. **调度队列**：调度器监听未分配节点的 Pod
3. **过滤阶段（Predicates）**：筛选可行节点
4. **打分阶段（Priorities）**：为可行节点打分
5. **绑定节点**：选择最高分节点，更新 Pod.spec.nodeName

### 8.2 亲和性与反亲和性

**节点亲和性（Node Affinity）**：
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/e2e-az-name
            operator: In
            values:
            - az1
            - az2
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: another-node-label-key
            operator: In
            values:
            - another-node-label-value
```

**Pod 亲和性（Pod Affinity）**：
```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - S1
        topologyKey: topology.kubernetes.io/zone
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - nginx
          topologyKey: kubernetes.io/hostname
```

### 8.3 污点与容忍

**污点（Taint）**：节点上的"排斥"标记，阻止 Pod 调度。

**容忍（Toleration）**：Pod 的"接受"标记，允许调度到带污点的节点。

**污点类型**：
- `NoSchedule`：不允许新 Pod 调度
- `PreferNoSchedule`：尽量不调度
- `NoExecute`：不允许调度，已运行的 Pod 会被驱逐

**为节点添加污点**：
```bash
kubectl taint nodes node1 key1=value1:NoSchedule
kubectl taint nodes node1 gpu=true:NoSchedule
```

**Pod 容忍示例**：
```yaml
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"
    value: "true"
    effect: "NoSchedule"
  - key: "node.kubernetes.io/unreachable"
    operator: "Exists"
    effect: "NoExecute"
    tolerationSeconds: 6000
```

**内置污点**：
- `node.kubernetes.io/not-ready`：节点未就绪
- `node.kubernetes.io/unreachable`：节点不可达
- `node.kubernetes.io/out-of-disk`：磁盘不足
- `node.kubernetes.io/memory-pressure`：内存压力

### 8.4 资源限制与服务质量

**资源请求与限制**：
- **requests**：调度依据，保证资源
- **limits**：运行上限，防止资源抢占

**QoS 类别**（Quality of Service）：
1. **Guaranteed**：所有容器都设置 requests=limits
   - 最高优先级，最后被驱逐
2. **Burstable**：至少一个容器设置 requests<limits
   - 中等优先级
3. **BestEffort**：未设置 requests 和 limits
   - 最低优先级，最先被驱逐

**示例**：
```yaml
spec:
  containers:
  - name: nginx
    resources:
      requests:
        memory: "256Mi"
        cpu: "500m"
      limits:
        memory: "512Mi"
        cpu: "1000m"
```

**LimitRange**：限制命名空间内资源默认值。
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 200m
    type: Container
```

---

## 9. 基于 kubeadm 的集群部署实践

### 9.1 环境准备

#### 9.1.1 主机规划
| 角色   | 主机名     | IP地址          | 配置要求       |
| ------ | ---------- | --------------- | -------------- |
| Master | k8s-master | 192.168.200.101 | 2核4G，50G磁盘 |
| Node1  | k8s-node1  | 192.168.200.102 | 2核4G，50G磁盘 |
| Node2  | k8s-node2  | 192.168.200.103 | 2核4G，50G磁盘 |

#### 9.1.2 系统要求
- **操作系统**：CentOS 7.9 或 Ubuntu 20.04 LTS
- **内核版本**：4.18+
- **容器运行时**：containerd 1.6+（推荐）

### 9.2 基础环境配置（所有节点）

#### 9.2.1 修改主机名
```bash
# Master 节点
hostnamectl set-hostname k8s-master
bash

# Node1 节点
hostnamectl set-hostname k8s-node1
bash

# Node2 节点
hostnamectl set-hostname k8s-node2
bash
```

#### 9.2.2 配置 hosts 文件
```bash
cat >> /etc/hosts <<EOF
192.168.200.101 k8s-master
192.168.200.102 k8s-node1
192.168.200.103 k8s-node2
EOF
```

#### 9.2.3 配置 SSH 免密登录（Master 节点）
```bash
# 生成密钥
ssh-keygen -t rsa -b 4096 -C "k8s-admin@cluster.local"

# 分发公钥
ssh-copy-id root@k8s-node1
ssh-copy-id root@k8s-node2

# 验证
ssh k8s-node1 "uptime"
ssh k8s-node2 "uptime"
```

#### 9.2.4 系统优化配置
```bash
# 1. 关闭 SELinux
setenforce 0
sed -i 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

# 2. 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 3. 禁用 Swap
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

# 4. 配置内核参数
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sysctl --system

# 5. 时间同步
timedatectl set-timezone Asia/Shanghai
systemctl enable chronyd
systemctl start chronyd
chronyc sources

# 6. 加载内核模块
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
modprobe overlay
modprobe br_netfilter
```

### 9.3 安装容器运行时

**安装 containerd（推荐）**：
```bash
# 所有节点执行
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y containerd.io

# 配置 containerd
mkdir -p /etc/containerd
containerd config default | tee /etc/containerd/config.toml

# 修改配置，启用 systemd cgroup
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

# 启动服务
systemctl enable containerd
systemctl start containerd
systemctl status containerd
```

**安装 Docker（备选）**：
```bash
yum install -y docker-ce docker-ce-cli containerd.io
systemctl enable docker
systemctl start docker

# 配置 cgroupdriver
cat <<EOF > /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF
systemctl restart docker
```

### 9.4 安装 kubeadm/kubelet/kubectl

**配置 Kubernetes 源**：
```bash
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 国内可使用阿里云镜像
cat <<EOF | tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF
```

**安装组件**：
```bash
yum install -y kubelet-1.28.0 kubeadm-1.28.0 kubectl-1.28.0 --disableexcludes=kubernetes

# 配置 kubelet cgroupdriver
sed -i 's/cgroupDriver: systemd/cgroupDriver: systemd/' /var/lib/kubelet/config.yaml

# 启动 kubelet
systemctl enable --now kubelet
```

### 9.5 初始化 Master 节点

```bash
# 在 Master 节点执行
kubeadm init \
  --apiserver-advertise-address=192.168.200.101 \
  --pod-network-cidr=10.244.0.0/16 \
  --service-cidr=10.96.0.0/12 \
  --image-repository=registry.aliyuncs.com/google_containers \
  --kubernetes-version=v1.28.0

# 配置 kubectl
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

# 查看初始化结果
kubectl get nodes
kubectl get pods -n kube-system
```

**初始化参数说明**：
- `--apiserver-advertise-address`：API Server 广播地址
- `--pod-network-cidr`：Pod 网络 CIDR（需与 CNI 插件匹配）
- `--service-cidr`：Service 网络 CIDR
- `--image-repository`：镜像仓库（国内使用阿里云加速）
- `--kubernetes-version`：Kubernetes 版本

**获取加入命令**：
```bash
# 在 Master 节点获取 node 加入命令
kubeadm token create --print-join-command
# 输出示例：
# kubeadm join 192.168.200.101:6443 --token abcdef.0123456789abcdef \
#   --discovery-token-ca-cert-hash sha256:xxx
```

### 9.6 部署网络插件

```bash
# Calico
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# 或 Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# 验证部署
kubectl get pods -n kube-system -w
```

### 9.7 加入 Node 节点

```bash
# 在每个 Node 节点执行
kubeadm join 192.168.200.101:6443 --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:xxx

# 在 Master 节点验证
kubectl get nodes
# 状态应为 Ready
```

### 9.8 验证集群状态

```bash
# 查看节点状态
kubectl get nodes -o wide

# 查看核心组件
kubectl get pods -n kube-system

# 查看集群信息
kubectl cluster-info

# 测试 DNS
kubectl run dns-test --rm -it --image=busybox:1.28 -- nslookup kubernetes.default

# 测试部署
kubectl create deployment nginx --image=nginx:1.25
kubectl expose deployment/nginx --port=80 --type=NodePort
kubectl get svc nginx
# 访问测试
curl http://<NodeIP>:<NodePort>
```

### 9.9 高可用集群部署

**架构**：3 Master + 多 Node + 外部负载均衡

**步骤**：
1. **部署 ETCD 集群**（独立或堆叠模式）
2. **配置外部负载均衡**（HAProxy/NGINX）
3. **初始化第一个 Master**
4. **其他 Master 加入**
5. **配置负载均衡 VIP**

**堆叠式 ETCD 高可用配置**：
```bash
# 第一个 Master 初始化
kubeadm init --control-plane-endpoint "k8s-lb.example.com:6443" \
  --upload-certs --pod-network-cidr=10.244.0.0/16

# 其他 Master 加入
kubeadm join k8s-lb.example.com:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <certificate-key>
```

---

## 10. kubectl 命令大全

### 10.1 基础命令速查

#### 10.1.1 集群信息
```bash
# 集群基本信息
kubectl cluster-info

# 节点列表
kubectl get nodes
kubectl get nodes -o wide

# 组件状态
kubectl get componentstatuses
kubectl get cs

# API 资源类型
kubectl api-resources

# 集群版本
kubectl version --short

# 查看事件（按时间排序）
kubectl get events --sort-by=.lastTimestamp
```

#### 10.1.2 资源查看
```bash
# Pod 操作
kubectl get pods
kubectl get pods -A
kubectl get pods -n kube-system
kubectl get pods -l app=nginx
kubectl get pods --show-labels
kubectl get pods --sort-by=.status.startTime

# 查看详情
kubectl describe pod <pod-name>
kubectl get pod <pod-name> -o yaml
kubectl get pod <pod-name> -o json

# 实时监控
kubectl get pods -w
kubectl top pods
kubectl top pods --containers

# 多资源查看
kubectl get pods,svc,deploy
kubectl get all
```

#### 10.1.3 资源创建与删除
```bash
# 创建资源
kubectl apply -f deployment.yaml  # 推荐，幂等
kubectl create -f service.yaml    # 仅创建

# 删除资源
kubectl delete -f deployment.yaml
kubectl delete pod <pod-name>
kubectl delete pods -l app=nginx
kubectl delete pods --all

# 强制删除
kubectl delete pod <pod-name> --grace-period=0 --force

# 清空命名空间
kubectl delete all --all -n <namespace>
kubectl delete namespace <namespace-name>
```

#### 10.1.4 Pod 操作
```bash
# 查看日志
kubectl logs <pod-name>
kubectl logs -f <pod-name>
kubectl logs --tail=20 <pod-name>
kubectl logs --since=1h <pod-name>
kubectl logs <pod-name> -c <container-name>

# 进入容器
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -c <container-name> -- sh

# 执行命令
kubectl exec <pod-name> -- ls /

# 文件复制
kubectl cp <pod-name>:/path/to/file /local/path
kubectl cp /local/file <pod-name>:/path/to/file

# 端口转发
kubectl port-forward <pod-name> 8080:80
kubectl port-forward svc/<service-name> 3000:80

# 临时调试 Pod
kubectl run debug-pod --rm -i --tty --image=busybox -- sh
kubectl run curl-test --rm -i --image=curlimages/curl -- curl http://nginx-service
```

### 10.2 Deployment 操作
```bash
# 查看 Deployment
kubectl get deployments
kubectl get deploy

# 滚动更新
kubectl set image deployment/nginx nginx=nginx:1.20
kubectl rollout status deployment/nginx

# 扩缩容
kubectl scale deployment/nginx --replicas=5

# 回滚
kubectl rollout undo deployment/nginx
kubectl rollout undo deployment/nginx --to-revision=2

# 查看历史
kubectl rollout history deployment/nginx
kubectl rollout history deployment/nginx --revision=3

# 暂停/恢复
kubectl rollout pause deployment/nginx
kubectl rollout resume deployment/nginx

# 编辑配置
kubectl edit deployment/nginx
```

### 10.3 Service 操作
```bash
# 查看 Service
kubectl get services
kubectl get svc

# 暴露 Deployment
kubectl expose deployment/nginx --type=NodePort --port=80
kubectl expose deployment/nginx --type=LoadBalancer --port=80 --target-port=8080

# 查看 Endpoint
kubectl get endpoints <service-name>

# 查看详情
kubectl describe svc <service-name>
```

### 10.4 配置与上下文管理
```bash
# 查看配置
kubectl config view
kubectl config view --raw

# 查看当前上下文
kubectl config current-context

# 列出所有上下文
kubectl config get-contexts

# 切换上下文
kubectl config use-context <context-name>

# 添加集群配置
kubectl config set-cluster <cluster-name> --server=https://192.168.200.101:6443

# 添加用户凭证
kubectl config set-credentials <user-name> --token=<token>
kubectl config set-credentials <user-name> --client-certificate=/path/to/cert --client-key=/path/to/key

# 设置上下文
kubectl config set-context <context-name> --cluster=<cluster-name> --user=<user-name> --namespace=default

# 删除上下文
kubectl config delete-context <context-name>

# 设置默认命名空间
kubectl config set-context --current --namespace=<namespace-name>
```

### 10.5 调试与故障排查
```bash
# 权限检查
kubectl auth can-i create pods
kubectl auth can-i "*" "*" --all-namespaces
kubectl auth can-i list secrets --as=system:serviceaccount:default:test-sa

# 查看 API 请求日志
kubectl get pods -v=6  # 级别 0-9

# 启动临时调试容器
kubectl run debug-pod --rm -i --tty --image=nicolaka/netshoot -- sh

# 查看资源配额
kubectl get resourcequota -n <namespace>

# 查看网络策略
kubectl get networkpolicies

# 查看所有 API 资源
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n <namespace>
```

### 10.6 高级技巧

#### 10.6.1 合并多个 kubeconfig
```bash
KUBECONFIG=file1:file2:file3 kubectl config view --merge --flatten > ~/.kube/config
```

#### 10.6.2 生成 YAML 模板
```bash
# 生成 Pod 模板（不创建）
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# 生成 Deployment 模板
kubectl create deploy nginx --image=nginx --replicas=3 --dry-run=client -o yaml > deploy.yaml

# 生成 Service 模板
kubectl expose deploy/nginx --port=80 --dry-run=client -o yaml > service.yaml
```

#### 10.6.3 标签与注解操作
```bash
# 添加标签
kubectl label pods <pod-name> env=prod
kubectl label nodes node1 disk=ssd gpu=true

# 删除标签
kubectl label pods <pod-name> env-

# 添加注解
kubectl annotate pods <pod-name> description="Production nginx pod" commit-hash="a1b2c3d"

# 查看标签和注解
kubectl get pods --show-labels
kubectl describe pod <pod-name>
```

#### 10.6.4 JSONPath 提取
```bash
# 提取特定字段
kubectl get pods <pod-name> -o jsonpath='{.status.podIP}'
kubectl get pods <pod-name> -o jsonpath='{.spec.containers[*].name}'
kubectl get pods <pod-name> -o jsonpath='{range .spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'

# 提取 Secret 并解码
kubectl get secret <secret-name> -o jsonpath="{.data.password}" | base64 -d
```

#### 10.6.5 批量操作
```bash
# 批量删除 Evicted Pod
kubectl get pods | grep Evicted | awk '{print $1}' | xargs kubectl delete pod

# 批量重启 Deployment
kubectl get deploy -n default -o name | xargs -I {} kubectl rollout restart {}

# 导出所有 ConfigMap
kubectl get configmap -n default -o yaml > configmaps.yaml
```

---

## 11. 监控与日志

### 11.1 Metrics Server

**部署**：
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# 或修改配置以允许不安全 TLS
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
spec:
  template:
    spec:
      containers:
      - name: metrics-server
        args:
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP
EOF
```

**使用**：
```bash
kubectl top nodes
kubectl top pods
kubectl top pods --containers
```

### 11.2 Prometheus + Grafana 监控方案

**架构**：
```
Prometheus ← ServiceMonitor ← Prometheus Operator
    ↓
Prometheus Adapter ← HPA
    ↓
Grafana (可视化)
```

**部署步骤**：
```bash
# 添加 Helm 仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 安装 kube-prometheus-stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  --set grafana.adminPassword=admin123

# 查看访问地址
kubectl get svc -n monitoring

# 端口转发访问 Grafana
kubectl port-forward -n monitoring svc/monitoring-grafana 3000:80
# 访问 http://localhost:3000，用户名 admin，密码 admin123
```

**常用监控指标**：
- **集群层面**：节点 CPU/内存/磁盘使用率
- **Pod 层面**：CPU/内存使用量、重启次数
- **应用层面**：HTTP 请求量、错误率、延迟

### 11.3 EFK 日志栈

**组件**：
- **Elasticsearch**：日志存储与检索
- **Fluentd/Fluentbit**：日志采集与转发
- **Kibana**：日志可视化

**部署示例**：
```bash
# 添加 Helm 仓库
helm repo add elastic https://helm.elastic.co
helm repo update

# 安装 Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace logging --create-namespace \
  --set replicas=3 \
  --set volumeClaimTemplate.storageClassName=fast-ssd

# 安装 Fluentd
helm install fluentd stable/fluentd \
  --namespace logging \
  --set elasticsearch.host=elasticsearch-master.logging.svc.cluster.local

# 安装 Kibana
helm install kibana elastic/kibana \
  --namespace logging \
  --set elasticsearchHosts=http://elasticsearch-master:9200

# 端口转发访问 Kibana
kubectl port-forward -n logging svc/kibana-kibana 5601:5601
```

### 11.4 监控最佳实践

1. **监控分层**：
   - 基础设施监控（节点、网络、存储）
   - Kubernetes 组件监控（API Server、ETCD、调度器）
   - 应用性能监控（APM）
   - 业务指标监控

2. **告警策略**：
   - Pod 重启次数 > 5 次/小时
   - 节点磁盘使用率 > 85%
   - API Server 延迟 > 1s
   - ETCD 集群 Leader 变更

3. **日志规范**：
   - 使用 JSON 格式输出日志
   - 包含时间戳、日志级别、请求 ID
   - 避免日志过大，设置合理的日志轮转

---

## 12. 故障排查

### 12.1 Pod 问题排查流程

#### 12.1.1 Pod 一直处于 Pending
**排查步骤**：
```bash
# 1. 查看 Pod 详情
kubectl describe pod <pod-name>

# 2. 常见原因
- 资源不足：检查节点资源 kubectl describe nodes
- 调度约束：检查 nodeSelector/affinity/tolerations
- 镜像拉取失败：检查镜像地址和 Secret
- PV 绑定失败：检查 StorageClass 和 PVC

# 3. 查看调度事件
kubectl get events --sort-by=.lastTimestamp | grep <pod-name>
```

#### 12.1.2 Pod 一直处于 CrashLoopBackOff
**排查步骤**：
```bash
# 1. 查看日志
kubectl logs <pod-name> --previous  # 查看前一次日志

# 2. 进入容器调试
kubectl run debug --rm -it --image=busybox -- sh
# 在容器内测试网络、DNS、依赖服务

# 3. 检查健康检查配置
kubectl get pod <pod-name> -o yaml | grep -A 10 liveness

# 4. 常见问题
- 应用启动失败：检查镜像、配置、依赖
- 端口冲突：检查容器端口配置
- 权限不足：检查 SecurityContext 和 RBAC
```

#### 12.1.3 Pod 网络不通
**排查步骤**：
```bash
# 1. 检查 Pod IP
kubectl get pod <pod-name> -o wide

# 2. 检查 Service Endpoint
kubectl get endpoints <service-name>

# 3. 在节点上测试
# 进入节点网络命名空间
docker run --rm -it --net=host nicolaka/netshoot -- ping <pod-ip>

# 4. 检查网络插件
kubectl get pods -n kube-system -l k8s-app=calico-node

# 5. 检查 NetworkPolicy
kubectl get networkpolicy -A
```

### 12.2 节点问题排查

#### 12.2.1 Node 状态为 NotReady
```bash
# 1. 查看节点详情
kubectl describe node <node-name>

# 2. 在节点上检查 kubelet 状态
systemctl status kubelet
journalctl -u kubelet -f

# 3. 检查容器运行时
crictl ps
crictl logs <container-id>

# 4. 常见原因
- kubelet 无法连接 API Server
- 容器运行时故障
- 磁盘空间不足（/var/lib/kubelet）
- 网络插件异常
```

#### 12.2.2 磁盘压力导致 Pod 驱逐
```bash
# 1. 查看节点资源
kubectl describe node <node-name> | grep -A 5 Capacity

# 2. 查看被驱逐 Pod
kubectl get pods --all-namespaces | grep Evicted

# 3. 清理 Docker 镜像
docker image prune -a

# 4. 清理无用卷
docker volume prune

# 5. 调整磁盘预留
vim /var/lib/kubelet/config.yaml
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
```

### 12.3 调度问题排查

#### 12.3.1 调度器无法调度
```bash
# 1. 查看调度器日志
kubectl logs -n kube-system kube-scheduler-<master-node>

# 2. 检查调度器扩展器
kubectl get configmap scheduler-config -n kube-system -o yaml

# 3. 模拟调度
kubectl create deployment test --image=nginx --dry-run=client -o yaml | kubectl apply -f-
kubectl describe pod <test-pod>
```

#### 12.3.2 亲和性配置错误
```bash
# 1. 检查 Pod 亲和性配置
kubectl get pod <pod-name> -o yaml | grep -A 20 affinity

# 2. 检查节点标签
kubectl get nodes --show-labels

# 3. 验证调度器配置
kubectl get configmap scheduler-config -n kube-system -o yaml
```

### 12.4 ETCD 问题排查

#### 12.4.1 ETCD 集群健康检查
```bash
# 检查 ETCD 集群健康
ETCDCTL_API=3 etcdctl endpoint health \
  --endpoints=https://127.0.0.1:2379,https://127.0.0.2:2379,https://127.0.0.3:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# 检查 Leader
ETCDCTL_API=3 etcdctl endpoint status \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
  --write-out=table
```

#### 12.4.2 ETCD 空间不足
```bash
# 1. 检查 ETCD 存储
ETCDCTL_API=3 etcdctl --write-out=table endpoint status

# 2. 清理缓存
ETCDCTL_API=3 etcdctl compact <revision>
ETCDCTL_API=3 etcdctl defrag

# 3. 调整 ETCD 配额
# 编辑 /etc/kubernetes/manifests/etcd.yaml
# 添加 --quota-backend-bytes=8589934592 (8GB)
```

### 12.5 性能问题排查

#### 12.5.1 API Server 响应慢
```bash
# 1. 检查 API Server 请求量
kubectl top pods -n kube-system -l component=kube-apiserver

# 2. 查看审计日志
# 启用审计日志
# /etc/kubernetes/manifests/kube-apiserver.yaml
--audit-policy-file=/etc/kubernetes/audit-policy.yaml
--audit-log-path=/var/log/kubernetes/audit.log

# 3. 检查 etcd 延迟
ETCDCTL_API=3 etcdctl check perf
```

#### 12.5.2 网络延迟高
```bash
# 1. 测试 Pod 间网络
kubectl run netshoot --rm -it --image=nicolaka/netshoot -- sh
# 在容器中
iperf3 -c <target-pod-ip>

# 2. 检查 CNI 插件日志
kubectl logs -n kube-system -l k8s-app=calico-node

# 3. 检查节点网络
ethtool -S eth0 | grep drop
netstat -i
```

---

## 13. 性能优化

### 13.1 集群优化

#### 13.1.1 API Server 优化
```yaml
# /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - name: kube-apiserver
    command:
    - kube-apiserver
    - --max-requests-inflight=400  # 限制并发请求数
    - --max-mutating-requests-inflight=200
    - --enable-aggregator-routing=true
    - --audit-log-maxsize=100
    - --audit-log-maxbackup=10
    - --audit-log-maxage=30
    resources:
      requests:
        cpu: 250m
        memory: 256Mi
      limits:
        cpu: 2000m
        memory: 2Gi
```

#### 13.1.2 调度器优化
```yaml
# 自定义调度配置
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      disabled:
      - name: ImageLocality
      enabled:
      - name: MyCustomPlugin
        weight: 10
```

#### 13.1.3 ETCD 优化
```yaml
# ETCD 配置优化
spec:
  containers:
  - name: etcd
    command:
    - etcd
    - --quota-backend-bytes=8589934592  # 8GB
    - --auto-compaction-mode=periodic
    - --auto-compaction-retention=8h
    - --snapshot-count=50000
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 2000m
        memory: 4Gi
```

### 13.2 应用优化

#### 13.2.1 资源请求设置最佳实践
```yaml
# 根据实际使用情况设置 request 和 limit
spec:
  containers:
  - name: app
    resources:
      requests:
        # 初始设置为 limit 的 70%
        memory: "700Mi"
        cpu: "700m"
      limits:
        # 通过监控确定峰值
        memory: "1Gi"
        cpu: "1000m"
```

#### 13.2.2 JVM 应用优化
```yaml
# Java 应用需调整 JVM 参数
spec:
  containers:
  - name: java-app
    image: my-java-app:1.0
    env:
    - name: JAVA_OPTS
      value: "-Xms512m -Xmx1024m -XX:+UseG1GC -XX:MaxRAMPercentage=75.0"
    resources:
      requests:
        memory: "800Mi"
        cpu: "500m"
      limits:
        memory: "1500Mi"
        cpu: "1000m"
```

#### 13.2.3 镜像优化
- **使用多阶段构建**：
```dockerfile
# 构建阶段
FROM golang:1.20 AS builder
WORKDIR /go/src/app
COPY . .
RUN go build -o main

# 运行阶段
FROM alpine:3.18
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /go/src/app/main .
CMD ["./main"]
```

- **使用 distroless 镜像**：减少攻击面
- **设置镜像拉取策略**：`imagePullPolicy: IfNotPresent`

### 13.3 资源管理优化

#### 13.3.1 资源配额（ResourceQuota）
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: production
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 100Gi
    limits.cpu: "40"
    limits.memory: 200Gi
    pods: "50"
    services: "20"
    persistentvolumeclaims: "30"
```

#### 13.3.2 LimitRange
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: default
spec:
  limits:
  - default:
      memory: 1Gi
      cpu: 1000m
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    min:
      memory: 128Mi
      cpu: 100m
    max:
      memory: 10Gi
      cpu: 4000m
    type: Container
```

#### 13.3.3 节点维护（Cordon/Drain）
```bash
# 标记节点不可调度
kubectl cordon node1

# 驱逐节点上 Pod
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data

# 恢复节点
kubectl uncordon node1
```

---

## 14. 最佳实践

### 14.1 生产环境建议

#### 14.1.1 高可用架构
- **Master 节点**：至少 3 节点，跨可用区部署
- **ETCD**：独立集群，SSD 存储，定期备份
- **负载均衡**：硬件或软件 LB（HAProxy/NGINX）
- **多可用区**：节点分布在多个可用区

#### 14.1.2 备份与恢复
```bash
# 备份 ETCD
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 备份集群配置
tar -czvf k8s-backup-$(date +%Y%m%d).tar.gz /etc/kubernetes/ /var/lib/etcd/

# 恢复 ETCD
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-from-backup
```

#### 14.1.3 升级策略
1. **小版本升级**：1.27 → 1.28（跳过不超过 2 个版本）
2. **先升级 Master**，再升级 Node
3. **逐个节点升级**，避免集群中断
4. **测试环境验证**，再升级生产环境

```bash
# Master 节点升级
kubeadm upgrade plan
kubeadm upgrade apply v1.28.0
apt-mark unhold kubelet kubeadm kubectl
apt-get upgrade kubelet=1.28.0-00 kubeadm=1.28.0-00 kubectl=1.28.0-00
systemctl restart kubelet

# Node 节点升级
kubeadm upgrade node
apt-get upgrade kubelet=1.28.0-00
systemctl restart kubelet
```

### 14.2 安全建议

#### 14.2.1 集群安全加固
1. **启用 RBAC**：禁用 ABAC 和 Webhook
2. **使用 NetworkPolicy**：默认拒绝所有流量
3. **启用 PSA**：PodSecurity Admission
4. **定期轮换证书**：kubeadm certs renew
5. **审计日志**：记录所有 API 请求
6. **镜像安全**：使用可信镜像仓库，扫描漏洞

#### 14.2.2 网络安全
```bash
# 启用 API Server 加密
--encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# 配置 TLS 1.2+
--tls-min-version=VersionTLS12

# 禁用不安全端口
--insecure-port=0
```

#### 14.2.3 密钥管理
- 使用 Sealed Secrets 加密 Git 中的 Secret
- 使用 External Secrets Operator 集成云 KMS
- 定期轮换数据库密码和证书

### 14.3 运维建议

#### 14.3.1 命令别名配置
```bash
# 添加到 ~/.bashrc 或 ~/.zshrc
alias k='kubectl'
alias kg='kubectl get'
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias ke='kubectl exec -it'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kdp='kubectl describe pod'
alias kds='kubectl describe svc'
alias kdd='kubectl describe deploy'
alias kaf='kubectl apply -f'
alias kdf='kubectl delete -f'
alias kgpw='kubectl get pods -w'
alias kgpn='kubectl get pods --sort-by=.status.startTime'
alias ksys='kubectl -n kube-system'
alias kdp='kubectl describe pod'
alias ksysg='kubectl get pods -n kube-system'

# 自动补全
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

#### 14.3.2 命名空间规划
```bash
# 按环境划分
kubectl create namespace dev
kubectl create namespace test
kubectl create namespace staging
kubectl create namespace prod

# 按团队/项目划分
kubectl create namespace team-a
kubectl create namespace team-b

# 设置资源配额
kubectl apply -f resourcequota.yaml -n prod
```

#### 14.3.3 GitOps 工作流
- **ArgoCD**：声明式持续部署
- **Flux**：GitOps 工具
- **最佳实践**：
  - 所有配置存储在 Git
  - 使用 PR 进行变更审核
  - 自动化测试和部署

---

## 15. 附录

### 15.1 常用资源清单模板

#### 15.1.1 完整应用栈模板
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  APP_ENV: production
  LOG_LEVEL: info
---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secret
  namespace: myapp
type: Opaque
data:
  DB_PASSWORD: cGFzc3dvcmQxMjM=  # base64 编码
---
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: myapp
        image: myapp:1.0
        ports:
        - containerPort: 8080
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secret
              key: DB_PASSWORD
        - name: APP_ENV
          value: production
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
---
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: myapp
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: myapp
---
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### 15.2 快速参考命令

#### 15.2.1 常用命令列表
```bash
# 集群状态
kubectl cluster-info
kubectl get nodes
kubectl top nodes

# 资源查看
kubectl get pods -A -o wide
kubectl get svc -A
kubectl get ingress -A
kubectl get pv,pvc -A

# 日志和调试
kubectl logs -f <pod> -c <container>
kubectl exec -it <pod> -- sh
kubectl debug node/<node-name> -it --image=busybox

# 资源清理
kubectl delete pod --field-selector=status.phase=Failed -A
kubectl delete pod --field-selector=status.phase=Succeeded -A

# 强制删除 Terminating 资源
kubectl patch pod <pod-name> -p '{"metadata":{"finalizers":null}}'
```

#### 15.2.2 检查清单
- [ ] 节点状态是否 Ready
- [ ] 核心 Pod 是否 Running
- [ ] 存储是否充足
- [ ] 网络是否通畅
- [ ] 证书是否过期
- [ ] 日志是否正常
- [ ] 监控是否告警

### 15.3 参考资料与工具

#### 15.3.1 官方资源
- **Kubernetes 官方文档**：https://kubernetes.io/docs/
- **kubectl 备忘单**：https://kubernetes.io/docs/reference/kubectl/cheatsheet/
- **API 参考**：https://kubernetes.io/docs/reference/kubernetes-api/

#### 15.3.2 推荐工具
- **k9s**：终端 UI 工具
  ```bash
  k9s
  ```
- **Lens**：桌面 UI 工具
  ```bash
  https://k8slens.dev/
  ```
- **kubectx/kubens**：快速切换上下文和命名空间
  ```bash
  kubectx context-name
  kubens namespace-name
  ```
- **kube-ps1**：命令行提示符增强
  ```bash
  source /usr/local/opt/kube-ps1/share/kube-ps1.sh
  PS1='$(kube_ps1)'$PS1
  ```

#### 15.3.3 学习资源
- **Kubernetes The Hard Way**：手动部署 Kubernetes
- **CKA/CKAD 认证**：官方认证考试
- **Katacoda 交互式教程**：在线实践环境
- **Kubernetes Patterns**：设计模式指南

---

## 文档说明

本指南整合了 Kubernetes 部署、架构、命令、监控、故障排查等完整知识体系。建议结合实际集群环境进行实践操作。对于生产环境部署，请务必：
1. 在测试环境充分验证
2. 实施高可用架构
3. 建立完善的监控告警
4. 定期备份 ETCD 数据
5. 遵循安全最佳实践

文档将持续更新，欢迎关注最新版本变化。