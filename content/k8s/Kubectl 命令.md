---
title: kubectl命令
description: 
date: 2025-11-11
categories:
    - 
    - 
---
# Kubectl 命令

`kubectl` 是 Kubernetes 的命令行工具，用于管理集群和部署应用。本文档提供每个命令的详细解释、参数含义及使用场景。

---

## 基本语法

```bash
kubectl [command] [TYPE] [NAME] [flags]
```
- **command**：操作动作（如 `get`、`create`、`delete`）
- **TYPE**：资源类型（如 `pods`、`services`）
- **NAME**：资源名称
- **flags**：可选参数

---

## 一、集群信息查看（了解集群整体状况）

```bash
# 查看集群基本信息
kubectl cluster-info
```
- **作用**：显示 Kubernetes 集群的核心组件访问地址（API Server、DNS、Dashboard等）
- **场景**：初次连接集群或排查集群连接问题时使用
- **输出示例**：Kubernetes master is running at https://xxx:6443

```bash
# 查看节点状态
kubectl get nodes
kubectl get nodes -o wide
```
- **作用**：列出集群中所有节点（Node/Worker）的基本信息（状态、版本、运行时间）
- **场景**：检查节点是否正常运行
- **参数说明**：`-o wide` 会额外显示节点的 INTERNAL-IP、OS-IMAGE 等详细信息

```bash
# 查看集群组件状态
kubectl get componentstatuses
```
- **作用**：检查核心控制平面组件（etcd、scheduler、controller-manager）的健康状态
- **场景**：集群级别故障排查时使用
- **别名**：可简写为 `kubectl get cs`

```bash
# 查看 API 资源类型
kubectl api-resources
```
- **作用**：列出当前集群支持的所有资源类型（如 pods、services、deployments等）
- **场景**：记不清资源类型名称或想确认某个资源是否支持时使用

```bash
# 查看 Kubernetes 版本
kubectl version
```
- **作用**：显示客户端（kubectl）和服务器端（API Server）的版本信息
- **场景**：确认版本兼容性，排查版本相关的问题

---

## 二、资源查看命令（日常最常用）

```bash
# 查看所有 Pod
kubectl get pods
```
- **作用**：列出当前命名空间下的所有 Pod（容器组）
- **场景**：查看应用运行状态、定位问题 Pod

```bash
kubectl get pods --all-namespaces
# 或简写
kubectl get pods -A
```
- **作用**：查看集群中所有命名空间的 Pod（系统 Pod + 用户 Pod）
- **场景**：排查跨命名空间的问题，查看系统组件状态

```bash
# 查看特定命名空间的资源
kubectl get pods -n kube-system
```
- **作用**：查看指定命名空间（这里是 kube-system 系统空间）的 Pod
- **参数说明**：`-n` 是 `--namespace` 的简写

```bash
# 查看多种资源
kubectl get pods,svc,deploy
```
- **作用**：一次性查看 Pod、Service、Deployment 三种资源
- **语法**：资源类型用逗号分隔，可一次性查看多种

```bash
# 查看资源详情
kubectl describe pod <pod-name>
```
- **作用**：显示指定 Pod 的详细配置和事件信息（比 get 命令更详细）
- **内容包含**：Pod IP、节点、标签、事件日志（Events）、容器状态等
- **场景**：Pod 启动失败、一直 Pending 时排查原因

```bash
# 查看资源 YAML 配置
kubectl get pod <pod-name> -o yaml
```
- **作用**：以 YAML 格式导出资源的完整配置（包括系统添加的默认字段）
- **参数说明**：`-o yaml` 指定输出格式为 YAML（也可 `-o json`）
- **场景**：学习资源配置结构、备份配置、调试时使用

```bash
# 实时监视资源变化
kubectl get pods -w
```
- **作用**：持续监控 Pod 状态变化，有新变化会自动刷新显示
- **参数说明**：`-w` 是 `--watch` 的简写
- **场景**：观察滚动更新过程、监控 Pod 重启情况

```bash
# 按标签筛选
kubectl get pods -l app=nginx
```
- **作用**：只显示带有 `app=nginx` 标签的 Pod
- **参数说明**：`-l` 是 `--selector` 的简写，支持标签选择器
- **场景**：在复杂系统中快速筛选出相关资源

```bash
# 查看资源排序显示
kubectl get pods --sort-by=.status.startTime
```
- **作用**：按 Pod 启动时间排序显示
- **参数说明**：`--sort-by` 支持 JSONPath 表达式
- **场景**：找出最新启动或最老的 Pod

---

## 三、资源创建与删除（管理资源生命周期）

```bash
# 从 YAML 文件创建/更新资源
kubectl apply -f deployment.yaml
```
- **作用**：根据 YAML 文件声明式创建或更新资源（推荐方式）
- **特点**：文件存在则更新，不存在则创建（幂等性）
- **场景**：应用部署、配置更新

```bash
kubectl create -f service.yaml
```
- **作用**：从 YAML 文件创建资源（命令式）
- **特点**：资源已存在会报错，不适合更新
- **场景**：首次创建资源

```bash
# 创建命名空间
kubectl create namespace <namespace-name>
```
- **作用**：创建逻辑隔离的命名空间
- **场景**：按项目/环境隔离资源（如 dev、test、prod）

```bash
# 从镜像直接创建 Deployment
kubectl create deployment nginx --image=nginx:latest
```
- **作用**：快速创建一个运行 nginx 镜像的 Deployment
- **场景**：快速测试、临时部署

```bash
# 删除资源
kubectl delete -f deployment.yaml
```
- **作用**：根据 YAML 文件删除其中定义的资源
- **场景**：清理通过文件创建的资源

```bash
kubectl delete pod <pod-name>
```
- **作用**：删除指定名称的 Pod
- **注意**：如果是 Deployment 管理的 Pod，会立即被重建

```bash
# 按标签删除
kubectl delete pods -l app=nginx
```
- **作用**：删除所有带有 `app=nginx` 标签的 Pod
- **场景**：批量清理测试资源

```bash
# 强制删除（立即）
kubectl delete pod <pod-name> --grace-period=0 --force
```
- **参数说明**：
  - `--grace-period=0`：立即删除，不等待优雅终止（默认30秒）
  - `--force`：强制删除，用于某些卡住无法删除的 Pod
- **警告**：可能导致数据丢失，慎用！

```bash
# 清空命名空间所有资源
kubectl delete all --all -n <namespace-name>
```
- **作用**：删除命名空间内所有类型资源（all 包括 pod、svc、deploy等）
- **参数说明**：第一个 `--all` 指所有资源类型，第二个 `--all` 指所有实例
- **危险操作**：会清空整个命名空间！

---

## 四、Pod 操作（深入容器内部）

```bash
# 查看 Pod 日志
kubectl logs <pod-name>
```
- **作用**：查看 Pod 中主容器的标准输出日志
- **场景**：应用报错排查

```bash
kubectl logs -f <pod-name>
```
- **参数说明**：`-f` 是 `--follow` 的简写，实时跟踪新日志
- **场景**：观察应用实时运行状态

```bash
kubectl logs --tail=20 <pod-name>
```
- **参数说明**：`--tail` 只显示最后 N 行日志
- **场景**：日志太多时只看最近的错误

```bash
kubectl logs <pod-name> -c <container-name>
```
- **参数说明**：`-c` 指定容器名称（多容器 Pod 必须指定）
- **场景**：Pod 内有多个容器时，查看特定容器日志

```bash
# 进入 Pod 容器（交互式）
kubectl exec -it <pod-name> -- /bin/bash
```
- **作用**：进入 Pod 容器的命令行终端
- **参数说明**：
  - `-i`：保持标准输入打开
  - `-t`：分配一个伪终端
  - `--`：分隔符，后面是容器内要执行的命令
- **场景**：调试应用、手动检查配置文件

```bash
kubectl exec -it <pod-name> -c <container-name> -- sh
```
- **作用**：多容器 Pod 中进入指定容器（使用 sh shell）
- **注意**：某些轻量镜像可能没有 bash，只有 sh

```bash
# 在 Pod 中执行一次性命令
kubectl exec <pod-name> -- ls /
```
- **作用**：在 Pod 中执行命令并返回结果，不进入交互模式
- **场景**：快速查看目录、执行简单命令

```bash
# 复制文件到/从 Pod
kubectl cp <pod-name>:/path/to/file /local/path
```
- **作用**：从 Pod 容器复制文件到本地
- **语法**：`pod-name:容器路径` → `本地路径`
- **场景**：导出日志、备份容器内文件

```bash
kubectl cp /local/file <pod-name>:/path/to/file
```
- **作用**：从本地上传文件到 Pod 容器
- **场景**：向容器内传配置文件、测试脚本

```bash
# 端口转发到本地
kubectl port-forward <pod-name> 8080:80
```
- **作用**：将本地 8080 端口转发到 Pod 的 80 端口
- **场景**：在本地调试集群内的服务，绕过 Service 直接访问 Pod

```bash
kubectl port-forward svc/<service-name> 3000:80
```
- **作用**：将本地 3000 端口转发到 Service 的 80 端口
- **场景**：本地访问集群内的 Service

```bash
# 查看 Pod 资源使用（需安装 metrics-server）
kubectl top pods
```
- **作用**：显示 Pod 的 CPU 和内存使用量
- **前提**：集群必须安装 metrics-server 组件
- **场景**：监控资源消耗，定位资源瓶颈

```bash
kubectl top pods --containers
```
- **参数说明**：`--containers` 显示容器级别的资源使用
- **场景**：Pod 内多容器时，分析哪个容器占用资源最多

---

## 五、Deployment 操作（管理应用版本）

```bash
# 查看 Deployment
kubectl get deployments
```
- **作用**：列出当前命名空间的 Deployment（部署控制器）
- **场景**：查看应用部署状态、副本数

```bash
# 查看滚动更新状态
kubectl rollout status deployment/<deployment-name>
```
- **作用**：实时监控 Deployment 滚动更新的进度
- **场景**：发布新版本时观察是否成功

```bash
# 更新镜像版本
kubectl set image deployment/nginx nginx=nginx:1.20
```
- **作用**：不修改 YAML 文件，直接更新 Deployment 中容器的镜像版本
- **语法**：`deployment/名称 容器名=新镜像`
- **场景**：快速回滚或升级镜像

```bash
# 扩缩容（调整副本数）
kubectl scale deployment/nginx --replicas=5
```
- **作用**：将 nginx deployment 的副本数调整为 5 个
- **场景**：应对流量高峰、弹性伸缩

```bash
# 回滚到上一个版本
kubectl rollout undo deployment/<deployment-name>
```
- **作用**：一键回滚到上一个稳定版本
- **场景**：新版本发布失败时快速恢复

```bash
# 回滚到指定版本
kubectl rollout undo deployment/<deployment-name> --to-revision=2
```
- **参数说明**：`--to-revision` 指定回滚到的历史版本号
- **场景**：需要回滚到特定历史版本

```bash
# 查看历史版本
kubectl rollout history deployment/<deployment-name>
```
- **作用**：显示 Deployment 的所有历史版本记录
- **场景**：查看可回滚的版本列表，确认版本号

```bash
# 暂停/恢复滚动更新
kubectl rollout pause deployment/<deployment-name>
```
- **作用**：暂停滚动更新（只更新部分 Pod 后停止）
- **场景**：金丝雀发布，先更新一小部分观察效果

```bash
kubectl rollout resume deployment/<deployment-name>
```
- **作用**：恢复暂停的滚动更新
- **场景**：金丝雀验证通过后继续更新剩余 Pod

```bash
# 编辑 Deployment
kubectl edit deployment <deployment-name>
```
- **作用**：打开默认编辑器（通常是 vim）直接修改 Deployment 配置
- **特点**：修改后立即生效，无需重新 apply
- **场景**：紧急修改配置，快速调试

---

## 六、Service 操作（管理网络访问）

```bash
# 查看 Service
kubectl get services
# 或简写
kubectl get svc
```
- **作用**：列出当前命名空间的 Service（服务）
- **场景**：查看服务暴露的端口、类型、ClusterIP

```bash
# 查看 Service 详情
kubectl describe svc <service-name>
```
- **作用**：显示 Service 的详细信息，包括后端端点（Endpoints）
- **场景**：Service 无法访问时，检查是否关联了正确的 Pod

```bash
# 暴露 Deployment 为 Service
kubectl expose deployment/nginx --type=NodePort --port=80
```
- **作用**：为 nginx deployment 创建 Service，类型为 NodePort
- **参数说明**：
  - `--type`：Service 类型（ClusterIP/NodePort/LoadBalancer）
  - `--port`：Service 暴露的端口
- **场景**：快速让外部访问集群内的应用

```bash
# 创建 Service 并指定目标端口
kubectl expose deployment/nginx --type=LoadBalancer --port=80 --target-port=8080
```
- **参数说明**：`--target-port` 是 Pod 内容器监听的端口
- **场景**：容器内应用监听在 8080，但外部希望用 80 端口访问

```bash
# 查看 Service 后端端点
kubectl get endpoints <service-name>
```
- **作用**：显示 Service 关联的后端 Pod IP 和端口
- **场景**：验证 Service 是否正确路由到 Pod

---

## 七、配置与上下文管理（多集群切换）

```bash
# 查看 kubeconfig 完整配置
kubectl config view
```
- **作用**：显示 kubectl 的配置文件（~/.kube/config）内容
- **场景**：确认当前使用的集群、用户、上下文信息

```bash
# 查看当前上下文
kubectl config current-context
```
- **作用**：显示当前正在使用的上下文名称（cluster + user + namespace 的组合）
- **场景**：确认当前操作的集群，避免误操作

```bash
# 列出所有可用上下文
kubectl config get-contexts
```
- **作用**：显示配置文件中定义的所有上下文列表
- **场景**：在多集群环境中切换前查看可用选项

```bash
# 切换上下文（切换集群/用户）
kubectl config use-context <context-name>
```
- **作用**：切换到指定的上下文（快速切换集群）
- **场景**：管理多个 K8s 集群时切换操作目标

```bash
# 添加集群配置
kubectl config set-cluster <cluster-name> --server=<server-url>
```
- **作用**：向配置文件中添加新集群的访问地址
- **参数说明**：`--server` 指定 API Server 地址
- **场景**：手动配置新集群连接信息

```bash
# 添加用户凭证
kubectl config set-credentials <user-name> --token=<token>
```
- **作用**：为指定用户设置访问凭证（Token 方式）
- **场景**：配置不同用户的访问权限

```bash
# 设置上下文
kubectl config set-context <context-name> --cluster=<cluster-name> --user=<user-name>
```
- **作用**：创建上下文，关联集群、用户和命名空间
- **场景**：配置多集群、多用户、多环境的组合

```bash
# 删除上下文
kubectl config delete-context <context-name>
```
- **作用**：从配置文件中删除指定的上下文
- **场景**：清理无效的集群配置

```bash
# 设置默认命名空间
kubectl config set-context --current --namespace=<namespace-name>
```
- **参数说明**：`--current` 修改当前上下文，`--namespace` 设置默认命名空间
- **效果**：后续命令无需每次都指定 `-n` 参数
- **场景**：长期在特定命名空间工作时提高效率

---

## 八、调试与故障排查（问题定位）

```bash
# 查看事件（重要！）
kubectl get events
```
- **作用**：查看集群中最近发生的事件（如 Pod 调度、启动、错误等）
- **场景**：Pod 启动失败、调度失败时查看详细原因

```bash
kubectl get events --sort-by=.lastTimestamp
```
- **参数说明**：按最后发生时间排序，最新事件在最后
- **场景**：事件太多时，按时间顺序查看

```bash
# 查看 API Server 请求（调试）
kubectl get pods -v=6
```
- **参数说明**：`-v` 设置日志级别（0-9），数字越大越详细
- **场景**：调试 kubectl 本身的问题，查看底层 API 调用

```bash
# 查看资源使用情况
kubectl top nodes
kubectl top pods
```
- **作用**：显示节点和 Pod 的实时资源使用
- **场景**：集群资源不足、Pod 被 OOMKilled 时排查

```bash
# 权限检查（重要！）
kubectl auth can-i create pods
```
- **作用**：检查当前用户是否有创建 Pod 的权限
- **场景**：权限不足报错时，确认自己的权限范围

```bash
kubectl auth can-i "*" "*" --all-namespaces
```
- **作用**：检查当前用户在所有命名空间的所有权限
- **参数说明**：`"*"` 代表所有 verb 和所有 resource
- **场景**：全面了解自己的权限

```bash
# 启动临时调试 Pod
kubectl run debug-pod --rm -i --tty --image=busybox -- sh
```
- **参数说明**：
  - `--rm`：退出后自动删除 Pod
  - `-i --tty`：交互式终端
  - `--image`：使用轻量级镜像
  - `--` 后面是启动命令
- **场景**：集群内网络测试、临时工具使用

```bash
# 查看网络策略
kubectl get networkpolicies
```
- **作用**：查看当前命名空间的网络策略
- **场景**：网络不通时检查是否有网络策略限制

---

## 九、高级命令（高效技巧）

```bash
# 合并多个 kubeconfig 文件
KUBECONFIG=file1:file2:file3 kubectl config view --merge --flatten > ~/.kube/config
```
- **作用**：将多个 kubeconfig 文件合并为一个
- **环境变量**：`KUBECONFIG` 用冒号分隔多个文件路径
- **参数说明**：`--merge` 合并，`--flatten` 扁平化输出
- **场景**：管理多个集群的配置整合

```bash
# 生成 YAML 模板（不实际创建）
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml
```
- **作用**：生成资源配置的 YAML 模板，但不真正创建资源
- **参数说明**：
  - `--dry-run=client`：客户端试运行（不发送到 API Server）
  - `-o yaml`：输出 YAML 格式
- **场景**：快速生成模板，再手动修改

```bash
# 标签操作
kubectl label pods <pod-name> env=prod
```
- **作用**：给 Pod 添加标签 `env=prod`
- **场景**：资源分类、标记环境、配合选择器使用

```bash
kubectl label pods <pod-name> env-
```
- **语法**：标签名后加 `-` 表示删除该标签
- **场景**：清理不再需要的标签

```bash
# 注解操作
kubectl annotate pods <pod-name> description="My pod"
```
- **作用**：添加注解（Annotation），用于存储非标识性元数据
- **场景**：记录资源说明、配置信息（对人可读）

```bash
# 查看资源配额
kubectl get resourcequota -n <namespace>
```
- **作用**：查看命名空间的资源配额限制（CPU、内存、Pod 数量等）
- **场景**：资源不足时，检查是否触发了配额限制

```bash
# 查看持久卷
kubectl get pv
```
- **作用**：查看集群级别的持久卷（PersistentVolume）
- **场景**：存储管理、排查存储问题

```bash
kubectl get pvc -n <namespace>
```
- **作用**：查看命名空间的持久卷声明（PersistentVolumeClaim）
- **场景**：应用申请存储后查看绑定状态

```bash
# 查看 ConfigMap 和 Secret
kubectl get configmaps
kubectl get secrets
```
- **作用**：查看配置信息和敏感信息
- **场景**：检查应用配置、验证 Secret 是否存在

```bash
# 查看 Secret 内容（解码）
kubectl get secret <secret-name> -o jsonpath="{.data.password}" | base64 -d
```
- **作用**：Secret 的值是 base64 编码的，需要解码查看明文
- **参数说明**：`jsonpath` 提取特定字段
- **场景**：查看数据库密码、Token 等敏感信息

---

## 十、常用选项速记（命令行参数）

| 选项               | 全称               | 含义与用途                        |
| ------------------ | ------------------ | --------------------------------- |
| `-n`               | `--namespace`      | 指定命名空间，隔离资源范围        |
| `-A`               | `--all-namespaces` | 查看所有命名空间，跨空间操作      |
| `-o wide`          | `--output wide`    | 显示额外信息（如节点 IP、Pod IP） |
| `-o yaml/json`     | `--output`         | 输出完整配置，用于导出或调试      |
| `-l`               | `--selector`       | 按标签筛选资源，精准定位          |
| `--show-labels`    | --                 | 显示资源的标签列                  |
| `--sort-by`        | --                 | 按字段排序，整理输出结果          |
| `-w`               | `--watch`          | 实时监控，观察变化                |
| `--dry-run=client` | --                 | 试运行，验证配置但不执行          |
| `--grace-period=0` | --                 | 立即删除，不等待优雅终止          |

---

## 实用技巧详解

### 1. **命令补全（节省打字时间）**

```bash
# Bash 环境
echo 'source <(kubectl completion bash)' >> ~/.bashrc
source ~/.bashrc
```
- **作用**：输入 `kubectl get p` 后按 Tab 自动补全为 `pods`
- **生效**：重新加载 bash 配置或重启终端

```bash
# Zsh 环境
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
```
- **作用**：Zsh 用户同样享受自动补全

### 2. **常用别名（极致偷懒）**

```bash
alias k='kubectl'
alias kg='kubectl get'
alias kgp='kubectl get pods'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias ke='kubectl exec -it'
alias kl='kubectl logs'
```
- **作用**：将长命令缩短为 2-3 个字母
- **配置**：添加到 `~/.bashrc` 或 `~/.zshrc`

### 3. **快速删除 Evicted Pods（清理僵尸）**

```bash
kubectl get pods | grep Evicted | awk '{print $1}' | xargs kubectl delete pod
```
- **执行过程**：
  1. `kubectl get pods`：获取所有 Pod
  2. `grep Evicted`：筛选状态为 Evicted 的
  3. `awk '{print $1}'`：提取 Pod 名称（第一列）
  4. `xargs kubectl delete pod`：批量删除这些 Pod
- **场景**：节点资源不足时，大量 Pod 被驱逐，清理残留记录

### 4. **清空命名空间（极度危险！）**

```bash
kubectl delete all --all -n <namespace>
```
- **执行逻辑**：删除指定命名空间内 `all` 资源类型的 `all` 实例
- **会删除**：Pod、Service、Deployment、StatefulSet 等
- **不会删除**：命名空间本身、ConfigMap、Secret（需单独删）
- **替代方案**：直接删除命名空间更彻底
  ```bash
  kubectl delete namespace <namespace-name>
  ```

---

**文档说明**：本指南基于 Kubernetes 2024 年版本整理，建议根据实际集群版本使用 `kubectl --help` 查看最新参数。