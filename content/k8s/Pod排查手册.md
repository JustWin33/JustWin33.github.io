---
title: Kubernetes Pod
description: 
date: 2025-11-12
categories:
    - 
    - 
---
# Pod排查手册

## 一、快速诊断问题原因

### 1.1 检查 Pod 是否真实存在

#### 命令1：`kubectl get pods`

**命令说明**：
```bash
kubectl get pods [pod-name] [-n namespace] [flags]
```
- `kubectl get`：获取资源列表的基本命令
- `pods`：指定资源类型为 Pod
- 默认范围：**当前命名空间**（由 kubeconfig 上下文决定）
- **无参数时**：列出当前命名空间下所有 Pod 的名称、状态等基本信息

**作用**：
快速查看当前命名空间下是否存在目标 Pod，以及其当前状态（Running/Pending/CrashLoopBackOff 等）

**输出解读**：
```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          5m
```
- **READY**：1/1 表示 1 个容器中有 1 个就绪
- **STATUS**：Running 表示正常运行
- **RESTARTS**：重启次数，0 表示未重启
- **AGE**：运行时长

**如果输出为空**：
说明当前命名空间下**没有任何 Pod**，或 Pod 名称拼写错误

---

#### 命令2：`kubectl get pods -A`

**命令说明**：
```bash
kubectl get pods --all-namespaces
# 或简写为
kubectl get pods -A
```
- `-A`/`--all-namespaces`：**全局查询**，跨所有命名空间

**作用**：
当不确定 Pod 所在命名空间时，全局扫描所有命名空间下的 Pod 列表

**典型输出**：
```
NAMESPACE     NAME                               READY   STATUS
default       nginx-pod                          1/1     Running
kube-system   coredns-6d8c4cb59d-abcde           1/1     Running
kube-system   etcd-master                        1/1     Running
kube-system   kube-apiserver-master              1/1     Running
```

**权限要求**：
- 需要 **ClusterRole** 的 `list` 权限对所有命名空间生效
- 生产环境默认用户通常只有特定 namespace 权限，执行 `-A` 可能报 **Forbidden**

**安全注意事项**：
- 全局查询会**暴露集群所有 Pod 信息**，包括 kube-system 等系统组件
- 审计日志会记录此操作，遵守最小权限原则，非必要不使用

---

#### 命令3：`kubectl get pods -A | grep nginx`

**命令说明**：
```bash
kubectl get pods --all-namespaces | grep [pattern]
```
- `|`：Linux 管道符，将前一个命令的输出作为后一个命令的输入
- `grep nginx`：文本过滤，仅显示包含 "nginx" 的行

**作用**：
在全局 Pod 列表中**快速定位**包含特定关键词（如 nginx）的 Pod，应对不记得完整名称的场景

**输出示例**：
```
default       nginx-pod                          1/1     Running
project-a     nginx-deployment-67d4bdd6f5-cx2nz  1/1     Running
```

**预期结果分析**：
- **列表为空** → **Pod 从未创建**，需进入"场景1"处理
- **有相似名称** → 可能是**拼写错误**（如 nginx-pod-7788ff99c6-abcde），进入"场景4"
- **在其他 namespace** → 进入"场景2"，需指定 `-n` 参数

---

### 1.2 确认当前上下文与命名空间

#### 命令4：`kubectl config view --minify --output 'jsonpath={..namespace}'`

**命令说明**：
```bash
kubectl config view              # 查看完整 kubeconfig 配置
         --minify                # 仅显示当前上下文信息
         --output 'jsonpath=...' # 使用 JSONPath 提取 namespace 字段
```

**作用**：
精准获取**当前默认命名空间**，避免因上下文切换错误导致的"找不到 Pod"问题

**输出示例**：
```
# 如果设置了命名空间
project-a

# 如果未设置（默认返回空）
[空]
```
**空输出时**：默认命名空间为 `default`

**权限要求**：
- 仅读取本地 `~/.kube/config` 文件，**无需集群权限**
- 但配置文件需有**可读权限**（通常 600 或 644）

**安全注意事项**：
- kubeconfig 文件包含**集群访问凭证**，权限必须严格限制（建议 600）
- 不要在共享目录存放含有 token 的 kubeconfig

---

#### 命令5：`kubectl config get-contexts`

**命令说明**：
```bash
kubectl config get-contexts          # 列出所有上下文
kubectl config current-context       # 显示当前活跃上下文
```

**作用**：
- **验证是否连接到正确的集群**
- 上下文包含：集群地址、用户、命名空间三元组
- 活跃上下文标记为 `*`

**输出示例**：
```
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         prod-cluster     prod.k8s.local   prod-admin       default
          test-cluster     test.k8s.local   test-user        test-env
```

**常见错误场景**：
- 上下文指向 **test 集群**，但用户在 **prod 集群**查找 Pod
- 切换上下文命令：`kubectl config use-context prod-cluster`

---

### 1.3 验证集群连接状态

#### 命令6：`kubectl cluster-info`

**命令说明**：
```bash
kubectl cluster-info          # 显示集群核心组件地址
```

**作用**：
**快速诊断** kubeconfig 配置是否正确、API Server 是否可达

**正常输出**：
```
Kubernetes control plane is running at https://192.168.1.100:6443
CoreDNS is running at https://192.168.1.100:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

**错误输出**：
```
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```
→ **kubeconfig 未配置**或**API Server 未启动**

**权限要求**：
- 仅需**匿名访问**或**最小只读权限**
- 如果提示 Unauthorized，说明证书或 token 错误

---

#### 命令7：`kubectl get nodes`

**命令说明**：
```bash
kubectl get nodes [-o wide] [node-name]
```

**作用**：
- 验证**集群基础设施状态**
- 确保至少有一个 Ready 节点可供调度

**典型输出**：
```
NAME         STATUS   ROLES           AGE   VERSION
k8s-master   Ready    control-plane   30d   v1.28.0
k8s-worker1  Ready    <none>          30d   v1.28.0
k8s-worker2  Ready    <none>          30d   v1.28.0
```

**关键排查点**：
- **STATUS = NotReady**：节点故障，所有 Pod 无法运行
- **ROLES = master**：是否为仅有 master 节点的单节点集群？
- **NoSchedule 污点**： master 节点默认有污点，普通 Pod 无法调度

---

## 二、针对性解决方案

### 场景 1：Pod 尚未创建（最常见）

#### 命令8：`kubectl run nginx-pod --image=nginx:1.25 --port=80`

**命令说明**：
```bash
kubectl run [pod-name] --image=[image] [--port=port] [--restart=Never]
```
- `kubectl run`：**快速创建 Pod/Deployment** 的命令
- `--image`：指定容器镜像（Docker Hub 或私有仓库）
- `--port`：声明容器暴露的端口（仅作为文档，不实际创建 Service）
- `--restart=Never`：创建独立 Pod（默认会创建 Deployment）

**作用**：
在测试环境中**快速拉起**一个 Nginx 容器，验证集群基本功能

**输出**：
```
pod/nginx-pod created
```

**权限要求**：
- 需要 **Role** 权限：`pods/create`
- 生产环境通常**禁止**普通用户直接创建 Pod（通过 RBAC 限制）

**安全注意事项**：
- **镜像拉取策略**：默认 `IfNotPresent`，可能拉取到被污染的镜像
- **镜像签名**：生产环境应启用镜像签名验证（如 Cosign）
- **资源限制**：未设置 limits，可能耗尽节点资源

---

#### 命令9：YAML 方式创建 Pod

**YAML 配置说明**：
```yaml
apiVersion: v1          # API 版本，v1 表示核心资源
kind: Pod               # 资源类型为 Pod
metadata:               # 元数据
  name: nginx-pod       # Pod 名称，集群内唯一
  labels:               # 标签，用于 Service 选择
    app: nginx
spec:                   # Pod 规约
  containers:           # 容器列表
  - name: nginx         # 容器名称
    image: nginx:1.25   # 精确版本号，避免 latest 歧义
    ports:
    - containerPort: 80 # 容器内部端口
    resources:          # 资源限制（生产必需）
      requests:         # 请求资源（调度依据）
        cpu: "100m"     # 100 毫核，1 核 = 1000m
        memory: "128Mi" # 128 MiB
      limits:           # 限制资源（防止耗尽）
        cpu: "200m"
        memory: "256Mi"
```

**作用**：
- **声明式部署**：YAML 是生产环境标准方式
- **版本控制**：可放入 Git，实现 Infrastructure as Code
- **可复用性**：便于修改和批量创建

**创建命令**：
```bash
kubectl apply -f nginx-pod.yaml    # 推荐：支持后续更新
# 或
kubectl create -f nginx-pod.yaml   # 仅创建，重复执行会报错
```

**权限要求**：
- `pods/create` + `pods/update`（对于 apply）
- 建议通过 CI/CD 流水线执行，**禁止**直接在生产集群手动 apply

---

#### 命令10：`kubectl get pods nginx-pod -w`

**命令说明**：
```bash
kubectl get pods [pod-name] -w
# -w = --watch，实时监视资源变化
```

**作用**：
**持续跟踪** Pod 状态，观察从 Pending → Running 的完整过程，快速定位卡在哪个阶段

**典型输出流**：
```
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   0/1     Pending   0          0s
nginx-pod   0/1     ContainerCreating   0          2s
nginx-pod   1/1     Running             0          8s
```

**状态分析**：
- **Pending**：调度中，可能资源不足、镜像拉取中
- **ContainerCreating**：正在创建容器，可能拉取镜像慢
- **Running**：成功运行

**排错提示**：
若长时间卡在 Pending，立即执行 `kubectl describe pod nginx-pod` 查看 Events

---

### 场景 2：Pod 存在于其他命名空间

#### 命令11：`kubectl get pods -A --field-selector metadata.name=nginx-pod`

**命令说明**：
```bash
kubectl get pods --all-namespaces --field-selector [field]=[value]
```
- `--field-selector`：**按字段精确过滤**，比 grep 更高效（服务端过滤）

**作用**：
**精准查询**指定名称的 Pod 存在于哪个命名空间，避免 grep 误匹配

**输出**：
```
NAMESPACE   NAME        READY   STATUS    RESTARTS   AGE
project-a   nginx-pod   1/1     Running   0          5m
```

**权限要求**：
- 需要 **ClusterRole** 的 `list` 权限
- 普通用户可能报 Forbidden

---

#### 命令12：`kubectl logs nginx-pod -n project-a`

**命令说明**：
```bash
kubectl logs [pod-name] -n [namespace]
# -n = --namespace，指定命名空间
```

**作用**：
**跨命名空间**查看 Pod 日志，解决因 namespace 错误导致的日志查询失败

**权限要求**：
- 需要 **Role** 权限：`pods/log`（在目标 namespace）
- 即使 `pods/get` 权限不足，只要显式指定 `-n` 且有 `pods/log` 权限即可

**注意事项**：
- 若忘记 `-n`，默认查询当前上下文 namespace，导致 NotFound
- **最佳实践**：总是显式指定 `-n`，避免依赖上下文

---

### 场景 3：Pod 已被删除

#### 命令13：`kubectl get events --field-selector involvedObject.name=nginx-pod`

**命令说明**：
```bash
kubectl get events                          # 获取集群事件
         --field-selector involvedObject.name=[pod-name]  # 过滤特定 Pod 的事件
```

**作用**：
**回溯历史**，查看 Pod 的完整生命周期事件，包括创建、调度、删除等

**典型输出**：
```
LAST SEEN   TYPE     REASON      OBJECT         MESSAGE
5m          Normal   Scheduled   pod/nginx-pod   Successfully assigned default/nginx-pod to worker1
4m          Normal   Pulling     pod/nginx-pod   Pulling image "nginx:1.25"
4m          Normal   Pulled      pod/nginx-pod   Successfully pulled image
4m          Normal   Created     pod/nginx-pod   Created container nginx
4m          Normal   Started     pod/nginx-pod   Started container nginx
2m          Normal   Killing     pod/nginx-pod   Stopping container nginx
2m          Normal   Deleted     pod/nginx-pod   Pod deleted by user "admin"
```

**关键事件**：
- **Deleted**：明确显示 Pod 被删除
- **Killing**：容器被终止

**权限要求**：
- 需要 **ClusterRole** 的 `events/list` 权限
- 事件默认保留 1 小时（可由 `--event-ttl` 配置）

**注意事项**：
- 事件存储在 etcd，**不支持长期审计**，如需持久化需接入外部系统（如 ELK）
- 若未找到 Deleted 事件，可能 Pod **从未创建成功**

---

### 场景 4：名称拼写错误

#### 命令14：`kubectl get pods -A | grep -i nginx`

**命令说明**：
```bash
grep -i [pattern]      # -i = --ignore-case，不区分大小写
```

**作用**：
**模糊匹配** Pod 名称，快速找到相似名称，解决大小写、后缀差异问题

**典型输出**：
```
default       nginx-Pod                          1/1     Running  # 大小写差异
default       nginx-pod-7788ff99c6-abcde         1/1     Running  # Deployment 生成的 Pod
default       my-nginx-app                       1/1     Running  # 包含关键词
```

**后续操作**：
```bash
# 找到正确名称后，查询日志
kubectl logs nginx-pod-7788ff99c6-abcde
```

**注意事项**：
- grep 是**客户端过滤**，会先获取所有 Pod 再过滤，大集群性能差
- 生产环境建议使用 `--field-selector` 替代 grep

---

## 三、查看日志的正确命令

### 命令15：`kubectl logs nginx-pod`

**命令说明**：
```bash
kubectl logs [pod-name] [-c container-name] [-n namespace]
```

**作用**：
获取 Pod 中容器的**标准输出（stdout）和标准错误（stderr）**日志

**输出示例**：
```
2025/01/15 10:30:15 [notice] 1#1: using the "epoll" event method
2025/01/15 10:30:15 [notice] 1#1: nginx/1.25.1
2025/01/15 10:30:15 [notice] 1#1: start worker processes
```

**权限要求**：
- 需要 **Role** 权限：`pods/log`
- 默认 Role `edit` 包含此权限，`view` 也包含

**注意事项**：
- 仅显示**当前运行**容器的日志，崩溃重启后旧日志丢失（需用 `--previous`）
- 日志存储在节点 `/var/log/pods/` 目录，受节点磁盘空间限制

---

### 命令16：`kubectl logs -f nginx-pod`

**命令说明**：
```bash
kubectl logs --follow [pod-name]
# -f = --follow，实时流式传输
```

**作用**：
**持续监视**日志输出，类似 `tail -f`，用于观察实时请求或调试线上问题

**使用场景**：
- 观察用户请求处理日志
- 监控错误发生时机
- 性能压测时实时查看

**退出方式**：
按 `Ctrl+C` 终止

**权限要求**：
同 `pods/log`，但会**保持长连接**，防火墙需允许 WebSocket 连接

**注意事项**：
- 长时间运行可能**消耗较多网络带宽**
- 若 Pod 重启，连接会中断，需重新执行命令

---

### 命令17：`kubectl logs --tail=100 nginx-pod`

**命令说明**：
```bash
kubectl logs --tail=[行数] [pod-name]
```

**作用**：
仅显示**最后 N 行**日志，避免全量日志刷屏，快速查看最新事件

**输出**：
直接输出最近的 100 条日志记录

**对比**：`tail -f` vs `tail -n`
- `--tail`：**静态**输出最后 N 行
- `-f`：**动态**流式输出

**权限要求**：
同 `pods/log`

---

### 命令18：`kubectl logs nginx-pod -c sidecar`

**命令说明**：
```bash
kubectl logs [pod-name] -c [container-name]
# -c = --container，指定容器名称
```

**作用**：
**多容器 Pod** 中，精确查看特定容器的日志（默认仅查看第一个容器）

**使用场景**：
- Sidecar 日志收集器（Fluentd）异常
- Istio 代理（envoy）日志
- 初始化容器（initContainer）日志

**典型 Pod 结构**：
```yaml
spec:
  containers:
  - name: nginx          # 主容器
  - name: sidecar        # 辅助容器
```

**权限要求**：
- `pods/log` 权限即可，无需额外配置

---

### 命令19：`kubectl logs --previous nginx-pod`

**命令说明**：
```bash
kubectl logs --previous [pod-name]
# 或 -p（缩写）
```

**作用**：
查看**上一个容器实例**的日志（容器崩溃重启前的日志），诊断 CrashLoopBackOff 关键命令

**输出示例**：
```
panic: runtime error: index out of range [10] with length 10
goroutine 1 [running]:
main.main()
    /app/main.go:15 +0x2f
```

**使用前提**：
- Pod 必须**至少重启过一次**（RESTARTS > 0）
- kubelet 保留旧日志（默认保留，直到被新日志覆盖）

**权限要求**：
- `pods/log` 权限（部分集群需额外配置 `pods/log/previous`）

**注意事项**：
- 若容器频繁重启（如每 10 秒），旧日志可能很快被覆盖
- 配合 `kubectl describe pod` 查看 Restart Count 和 Last State

---

## 四、生产环境建议

### 4.1 避免手动创建 Pod

#### 命令20：`kubectl create deployment nginx-deploy --image=nginx:1.25 --replicas=2`

**命令说明**：
```bash
kubectl create deployment [name] --image=[image] --replicas=[number]
```

**作用**：
创建 **Deployment** 控制器，自动管理 Pod 副本，实现**自愈、滚动更新、回滚**

**对比：Pod vs Deployment**

| 特性         | 独立 Pod         | Deployment      |
| ------------ | ---------------- | --------------- |
| **自愈**     | ❌ 删除后永久消失 | ✅ 自动重建      |
| **扩缩容**   | 手动创建多个     | ✅ 一键扩缩      |
| **更新**     | 手动删除重建     | ✅ 滚动更新      |
| **版本管理** | ❌                | ✅ 记录 revision |
| **生产推荐** | ❌ 仅测试         | ✅ 标准方式      |

**底层机制**：
- Deployment → ReplicaSet → Pod
- 删除 Pod 后，ReplicaSet 立即创建新 Pod 保证副本数

**权限要求**：
- `deployments/create`、`replicasets/create`、`pods/create`
- 生产环境应通过 CI/CD 执行，**禁止**手动创建

---

#### 命令21：`kubectl get pods -l app=nginx-deploy`

**命令说明**：
```bash
kubectl get pods -l [label-key]=[label-value]
# -l = --selector，标签选择器
```

**作用**：
通过**标签**查询 Deployment 管理的所有 Pod，无需关心具体名称

**输出示例**：
```
NAME                            READY   STATUS    RESTARTS   AGE
nginx-deploy-5c6f7d8f99-abcde   1/1     Running   0          5m
nginx-deploy-5c6f7d8f99-fghij   1/1     Running   0          5m
```

**优势**：
- 无需记忆自动生成的 Pod 后缀
- Deployment 更新后 Pod 名称变化，但标签不变

---

#### 命令22：`kubectl logs -l app=nginx-deploy`

**命令说明**：
```bash
kubectl logs -l [label]
```

**作用**：
**聚合查询**所有匹配标签的 Pod 日志，批量查看

**权限要求**：
- `pods/log` 权限 + 标签选择器范围

**注意事项**：
- 日志会**交错输出**，需配合 `--prefix` 区分来源
- 单个 Pod 日志量大的场景慎用

---

### 4.2 常用别名与快捷方式

#### 命令23：配置 Shell 别名

**配置方法**：
```bash
# ~/.bashrc 或 ~/.zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgn='kubectl get nodes'
alias klogs='kubectl logs'
alias kdesc='kubectl describe pod'
alias kevents='kubectl get events'
```

**作用**：
**提升效率**，减少重复输入（运维高频操作）

**生效方式**：
```bash
source ~/.bashrc
```

**安全注意事项**：
- 别名在**共享环境**（如 jump server）可能覆盖他人配置
- 脚本中**禁用别名**，应使用完整命令保证可移植性

---

### 4.3 调试技巧

#### 命令24：`kubectl describe pod nginx-pod`

**命令说明**：
```bash
kubectl describe [resource-type] [resource-name]
```

**作用**：
**获取 Pod 的详细状态信息**，包括事件、资源、调度、网络等，是排查问题的**首选命令**

**输出详解**：

| 段落               | 关键信息     | 排查价值                         |
| ------------------ | ------------ | -------------------------------- |
| **Name/Namespace** | Pod 全称     | 确认查询对象正确                 |
| **Node**           | 所在节点     | 节点故障时定位                   |
| **Labels**         | 标签         | Service 是否匹配                 |
| **Annotations**    | 注解         | 审计、监控配置                   |
| **Status**         | Pod 阶段     | Pending/CrashLoopBackOff         |
| **IP**             | Pod IP       | 网络连通性测试                   |
| **Containers**     | 容器状态     | 退出码、重启次数                 |
| **Events**         | **核心事件** | **调度失败、镜像拉取、启动错误** |

**典型 Events 分析**：
```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  2m    default-scheduler  0/3 nodes are available: ...
  Warning  FailedMount       1m    kubelet            MountVolume.SetUp failed for volume "pvc-xxx"
  Warning  BackOff           30s   kubelet            Back-off restarting failed container
```

**权限要求**：
- 需要 `pods/get` 权限
- 节点信息需 `nodes/get`（系统组件事件）

---

## 五、快速检查清单

### 完整排查流程（含命令详解）

```bash
# Step 1: 全局确认 Pod 是否存在
kubectl get pods -A | grep nginx
# 含义：扫描所有命名空间，过滤含 "nginx" 的 Pod
# 若为空 → 未创建或被删除
# 若有 → 记录 NAMESPACE 和完整 NAME

# Step 2: 确认当前操作上下文
kubectl config view --minify | grep namespace
# 含义：查看当前默认命名空间
# 若返回空 → 使用 default
# 若返回其他 → 与 Step 1 结果对比

# Step 3: 按需创建测试 Pod
kubectl run nginx-pod --image=nginx:1.25
# 含义：快速创建单容器 Pod测试
# 注意：生产环境禁用，应使用 Deployment

# Step 4: 持续监视状态变化
kubectl get pods nginx-pod -w
# 含义：watch 模式实时查看
# 预期：Pending → ContainerCreating → Running（30s内）
# 若卡 Pending → 立即执行 Step 5

# Step 5: 详细诊断
kubectl describe pod nginx-pod
# 含义：获取完整信息和事件
# 重点查看 Events 段的 Reason/Message
# 常见：FailedScheduling/ImagePullBackOff/CrashLoopBackOff

# Step 6: 成功运行后查看日志
kubectl logs nginx-pod --tail=20
# 含义：查看最后 20 行日志验证服务
# 预期：看到 Nginx 启动 success 信息
```

### 生产环境标准排查命令集

```bash
# 1. 检查 Deployment 状态（推荐）
kubectl get deploy -n your-ns
kubectl describe deploy/nginx-deploy -n your-ns

# 2. 查看 Pod 列表（通过标签）
kubectl get pods -l app=nginx -n your-ns

# 3. 查看 Pod 详细信息
kubectl describe pod [pod-name] -n your-ns

# 4. 查看实时日志
kubectl logs -f [pod-name] -n your-ns

# 5. 查看历史事件（30分钟内）
kubectl get events -n your-ns --sort-by='.lastTimestamp'
```

---

**总结**：`kubectl get/describe/logs` 是排查 Pod NotFound 的核心三剑客，理解每个命令的**权限范围、命名空间上下文、输出含义**，是 Kubernetes 运维的基本功。生产环境务必遵循**最小权限、声明式部署、标签化管理**三大原则。