---
title: Kubernetes 生产级完整实战指南
description: 
date: 2025-11-19
categories:
    - 
    - 
---

# Kubernetes生产级完整实战指南

这是一份从零开始构建生产级Kubernetes应用的完整教程，涵盖Pod生命周期、数据持久化、服务暴露、配置管理、健康检查等核心内容。每一步操作都附有详细解释和原理说明。

---

## 第一部分：Pod核心原理与创建流程

### 1.1 Pod创建流程深度解析

```bash
# 示例：创建一个简单的Nginx Pod
kubectl run mynginx --image=nginx:1.14.2 --restart=Never
```

**完整创建流程详解**：

| 步骤 | 执行组件            | 具体操作                          | 详细解释                                                     |
| ---- | ------------------- | --------------------------------- | ------------------------------------------------------------ |
| 1    | kubectl客户端       | 解析命令参数，生成RESTful API请求 | 将`run`命令转换为JSON格式的Pod定义，包含镜像、名称等元数据   |
| 2    | kube-apiserver      | 接收POST请求，认证鉴权            | 验证客户端身份和权限，检查请求格式是否符合API规范            |
| 3    | etcd                | 存储Pod元数据                     | 将Pod状态设为"Pending"，记录spec信息但不分配节点             |
| 4    | kube-scheduler      | Watch机制检测到新Pod              | Scheduler持续监听apiserver的未调度Pod，根据资源需求筛选节点  |
| 5    | kube-scheduler      | 调度算法选择最优节点              | 经过预选（Predicate）和优选（Priority）两个阶段，考虑资源、亲和性、污点半径等因素 |
| 6    | kube-apiserver      | 更新Pod的nodeName字段             | 将调度结果写回etcd，Pod状态仍为"Pending"                     |
| 7    | kubelet（目标节点） | 监听分配到本节点的Pod             | 通过apiserver获取本节点需运行的Pod清单                       |
| 8    | kubelet             | 调用CRI创建容器                   | 通过Docker/containerd接口拉取镜像、创建容器、配置网络        |
| 9    | CNI插件             | 分配Pod IP地址                    | 从节点CIDR网段中分配IP，配置veth pair和iptables规则          |
| 10   | kubelet             | 执行健康检查                      | 启动后等待initialDelaySeconds，开始执行探针检测              |
| 11   | kubelet             | 更新Pod状态为"Running"            | 所有容器启动成功后，通过apiserver更新etcd中的状态            |
| 12   | etcd                | 持久化最终状态                    | 整个集群的状态同步完成                                       |

**关键概念解释**：
- **Watch机制**：Kubernetes组件使用长连接监听资源变化，无需轮询，实现毫秒级响应
- **CRI（容器运行时接口）**：kubelet与Docker/containerd通信的标准化接口
- **CNI（容器网络接口）**：负责Pod网络配置，常见实现有Flannel、Calico、Cilium

---

### 1.2 Pod YAML完整配置解析

```yaml
apiVersion: v1          # API版本，v1是稳定核心版本
kind: Pod               # 资源类型为Pod
metadata:
  name: mynginx         # Pod名称，同一命名空间下唯一
  namespace: default    # 命名空间，默认为default
  labels:               # 标签键值对，用于Service选择器
    app: frontend
    tier: web
    version: v1
  annotations:          # 注释信息，不影响功能，用于记录元数据
    description: "Production nginx pod"
    monitoring: "true"
spec:
  containers:
  - name: nginx         # 容器名称，Pod内唯一
    image: nginx:1.14.2 # 镜像地址，支持Docker Hub和私有仓库
    imagePullPolicy: IfNotPresent  # 镜像拉取策略
    # IfNotPresent: 本地不存在才拉取（生产推荐）
    # Always: 每次都拉取（开发环境）
    # Never: 仅使用本地镜像
    ports:
    - containerPort: 80 # 容器暴露的端口
      name: http        # 端口名称，便于引用
      protocol: TCP     # 协议类型
    env:                # 环境变量注入
    - name: NGINX_ENV
      value: "production"
    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP  # 引用Pod自身信息
    resources:          # 资源限制（生产环境必须设置）
      requests:
        memory: "64Mi"  # 请求内存，调度依据（软限制）
        cpu: "50m"      # 请求CPU，1核=1000m（50m=0.05核）
      limits:
        memory: "128Mi" # 内存硬限制，超过会触发OOMKill
        cpu: "100m"     # CPU硬限制，超过会被限流（不重启）
    volumeMounts:       # 存储卷挂载
    - name: cache
      mountPath: /cache  # 容器内挂载路径
      readOnly: false    # 是否只读
    livenessProbe:      # 存活探针（检测容器是否存活）
      httpGet:
        path: /healthz
        port: 80
      initialDelaySeconds: 10  # 启动后延迟10秒开始探测
      periodSeconds: 5         # 每5秒探测一次
      timeoutSeconds: 3        # 请求超时时间
      failureThreshold: 3      # 连续3次失败才重启
    readinessProbe:     # 就绪探针（检测容器是否准备好接收流量）
      httpGet:
        path: /ready
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 3
    startupProbe:       # 启动探针（保护慢启动应用）
      httpGet:
        path: /startup
        port: 80
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 30  # 最多容忍300秒启动时间
  volumes:              # 存储卷定义
  - name: cache
    emptyDir: {}       # 临时目录，Pod删除时清空
  restartPolicy: Always # 重启策略
  # Always: 无论何原因退出都重启（默认）
  # OnFailure: 仅错误退出时重启
  # Never: 从不重启
  nodeSelector:         # 节点选择器
    disktype: ssd       # 只调度到有ssd标签的节点
  tolerations:          # 污点容忍（允许调度到污点半径节点）
  - key: "node-role.kubernetes.io/master"
    operator: "Exists"
    effect: "NoSchedule"
```

---

## 第二部分：数据持久化完整方案

### 2.1 emptyDir临时存储

```bash
# 创建工作目录
mkdir 1119 && cd 1119

# 创建emptyDir配置文件
vim emptydir.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-empty
spec:
  containers:
  - name: container-empty
    image: nginx:1.19.5
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - mountPath: /cache      # 容器内挂载路径
      name: cache-volume    # 引用的卷名称
  volumes:
  - name: cache-volume
    emptyDir:
      sizeLimit: 500Mi      # 限制最大容量（可选）
      # medium: Memory      # 可选：存储在tmpfs（内存）中，速度极快但会消耗内存
```

**执行与验证**：
```bash
# 创建Pod
kubectl apply -f emptydir.yaml

# 查看Pod调度节点
kubectl get pods -o wide | grep pod-empty
# 输出示例：pod-empty   1/1   Running   0   2m   10.244.1.5   k8s-node1

# 查看Pod的UID（用于定位节点上的实际路径）
kubectl get pods pod-empty -o yaml | grep uid
# 输出：uid: 8f9c8d8a-1b2c-3d4e-5f6g-7h8i9j0k1l2m

# 在对应节点上查看emptyDir实际路径（需要SSH到节点）
# 路径：/var/lib/kubelet/pods/<uid>/volumes/kubernetes.io~empty-dir/<volume-name>
# 例如：/var/lib/kubelet/pods/8f9c8d8a-1b2c-3d4e-5f6g-7h8i9j0k1l2m/volumes/kubernetes.io~empty-dir/cache-volume
```

**emptyDir核心特性详解**：
- **生命周期**：与Pod完全绑定，Pod删除→卷删除→数据丢失
- **适用场景**：
  - 应用程序缓存（重启可接受数据丢失）
  - 同一Pod内容器间文件共享（如sidecar日志收集）
  - 临时数据处理（排序、转换等）
- **性能**：
  - 默认使用节点磁盘，受磁盘IO限制
  - `medium: Memory`时使用tmpfs，内存速度但重启丢失且消耗内存
- **限制**：通过`sizeLimit`设置，超过后Pod会被驱逐
- **生产建议**：绝不用于持久化数据，仅用于可重建的临时数据

---

### 2.2 hostPath节点存储

```bash
# 将tomcat镜像分发到节点（因使用IfNotPresent策略）
ll tomcat.tar.gz
scp -r tomcat.tar.gz k8s-node1:/root/
scp -r tomcat.tar.gz k8s-node2:/root/
# 在节点上加载镜像
# ssh k8s-node1 "docker load -i /root/tomcat.tar.gz"
# ssh k8s-node2 "docker load -i /root/tomcat.tar.gz"

# 创建hostPath配置文件
vim hostpath.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - image: nginx:1.19.5
    imagePullPolicy: IfNotPresent
    name: test-nginx
    volumeMounts:
    - mountPath: /test-nginx   # 第一个容器挂载路径
      name: test-volume
  - image: tomcat:latest
    imagePullPolicy: IfNotPresent
    name: test-tomcat
    volumeMounts:
    - mountPath: /test-tomcat  # 第二个容器挂载路径
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data1             # 节点上的真实目录路径
      type: DirectoryOrCreate  # 目录不存在时自动创建
      # 其他type选项：
      # Directory: 目录必须存在
      # FileOrCreate: 文件不存在时自动创建
      # File: 文件必须存在
      # Socket: UNIX套接字必须存在
      # CharDevice: 字符设备必须存在
      # BlockDevice: 块设备必须存在
```

**执行与交互验证**：
```bash
kubectl apply -f hostpath.yaml
kubectl get pod -owide

# 进入Nginx容器写入数据
kubectl exec -it test-hostpath -c test-nginx -- bash
# 命令解析：
# exec: 在运行容器中执行命令
# -it: 交互式终端
# -c test-nginx: 指定容器名（Pod内有多个容器时必须指定）
# -- bash: 在容器内执行bash shell

# 在容器内执行
echo "I am test-nginx test data write" > /test-nginx/mslinux-data
exit

# 进入Tomcat容器读取数据
kubectl exec -it test-hostpath -c test-tomcat -- bash
cat /test-tomcat/mslinux-data
# 输出：I am test-nginx test data write
echo "I am test-tomcat test data write" >> /test-tomcat/mslinux-data
exit

# 在节点上验证文件实际位置
# SSH到Pod所在节点
# cat /data1/mslinux-data
# 输出包含两行数据，证明两个容器共享同一主机目录
```

**hostPath核心特性详解**：
- **生命周期**：独立于Pod，Pod删除后数据保留在节点上
- **适用场景**：
  - 单节点守护进程日志收集（如fluentd收集/var/log）
  - 需要访问节点系统文件（如/etc/hosts）
  - 开发测试环境快速共享数据
- **风险与限制**：
  - **Pod漂移问题**：Pod被调度到其他节点后无法访问原数据
  - **安全风险**：可访问节点任意路径，需谨慎使用
  - **权限问题**：容器内root可能映射到节点普通用户（取决于root squash设置）
- **生产建议**：
  - 几乎不用于生产环境，除非使用DaemonSet固定节点
  - 必须配合nodeSelector或nodeAffinity使用，确保Pod固定节点
  - 避免使用`type: FileOrCreate`，防止权限混乱

---

### 2.3 NFS网络存储（生产推荐）

#### 2.3.1 NFS Server配置（Master节点）

```bash
# 步骤1：安装NFS服务组件
yum install -y nfs-utils rpcbind
# nfs-utils: 提供NFS服务器和客户端工具
# rpcbind: 端口映射服务，NFS依赖的RPC服务

# 步骤2：创建共享目录并设置权限
mkdir -p /nfs/data
chmod -R 755 /nfs/data
chown nfsnobody:nfsnobody /nfs/data
# 权限详解：
# 755: 所有者可读写执行，其他用户只读执行
# nfsnobody: NFS服务的匿名用户，容器以root写入时映射为此用户

# 步骤3：配置exports文件（访问控制列表）
cat >> /etc/exports <<EOF
/nfs/data/ 192.168.200.0/24(insecure,rw,sync,no_root_squash)
# 参数详解：
# /nfs/data: 共享目录路径
# 192.168.200.0/24: 允许访问的网段
# insecure: 允许客户端使用大于1024的端口（容器常用）
# rw: 读写权限
# sync: 同步写入，数据安全但性能较低（async性能高但可能丢数据）
# no_root_squash: 容器root用户保持root权限（默认root_squash会映射为nobody）
EOF

# 步骤4：启动服务并设置开机自启
systemctl enable --now rpcbind nfs-server
# enable --now: 相当于 enable + start，设置开机启动并立即启动

# 步骤5：验证共享配置
exportfs -v
# 输出示例：
# /nfs/data 	192.168.200.0/24(sync,wdelay,hide,no_subtree_check,sec=sys,rw,secure,no_root_squash,no_all_squash)
# 每个参数含义：
# sync: 同步模式
# wdelay: 延迟写入，优化性能
# hide: 隐藏下级目录
# no_subtree_check: 禁用子树检查，提升性能
# sec=sys: 使用系统UID/GID认证
# secure: 要求客户端使用安全端口（与insecure互斥，以exports配置为准）

# 步骤6：检查NFS共享是否可用
showmount -e localhost
# 应输出：/nfs/data 192.168.200.0/24
```

**常见NFS故障排查**：
```bash
# 故障1：showmount -e 超时
# 解决：检查防火墙 firewall-cmd --add-service=nfs --permanent && firewall-cmd --reload
# 或 systemctl stop firewalld

# 故障2：容器内Permission denied
# 解决：检查exports权限是否为no_root_squash，检查目录chmod权限

# 故障3：挂载后文件所有者错乱
# 解决：确保chown nfsnobody:nfsnobody已执行，检查/etc/idmapd.conf配置
```

---

#### 2.3.2 NFS Client配置（所有Node节点）

```bash
# 在k8s-node1和k8s-node2上执行：

# 步骤1：验证NFS服务器可用性
showmount -e 192.168.200.100
# -e: 显示导出列表
# 输出应包含：/nfs/data 192.168.200.0/24
# 如果超时，检查Master节点防火墙和NFS服务状态

# 步骤2：创建本地挂载点
mkdir -p /nfs/data

# 步骤3：手动挂载测试（临时）
mount -t nfs 192.168.200.100:/nfs/data /nfs/data
# -t nfs: 指定文件系统类型
# 格式：server:remote-dir local-dir

# 验证挂载是否成功
df -h | grep nfs
# 输出示例：192.168.200.100:/nfs/data   50G  10G  40G  20% /nfs/data

# 步骤4：配置开机自动挂载（生产必备）
cat >> /etc/fstab <<EOF
192.168.200.100:/nfs/data /nfs/data nfs defaults,_netdev 0 0
# /etc/fstab字段说明：
# 第1列：远程NFS路径
# 第2列：本地挂载点
# 第3列：文件系统类型
# 第4列：挂载选项
#   defaults: 默认选项(rw, suid, dev, exec, auto, nouser, async)
#   _netdev: 网络设备，网络就绪后再挂载（避免开机卡死）
# 第5列：是否备份（0不备份）
# 第6列：fsck检查顺序（0不检查）
EOF

# 测试fstab配置
mount -a
# 无输出表示配置正确

# 步骤5：测试写入权限
touch /nfs/data/test.txt
# 在所有节点验证文件是否同步出现
ls -l /nfs/data/test.txt
# 检查文件大小、时间戳是否一致

# 步骤6：性能优化（可选）
cat >> /etc/fstab <<EOF
192.168.200.100:/nfs/data /nfs/data nfs rw,_netdev,rsize=1048576,wsize=1048576,hard,intr,nolock 0 0
# 优化参数：
# rsize/wsize: 读写块大小（字节），增大提升性能
# hard: 硬挂载，NFS服务器不可用时程序等待（soft会超时）
# intr: 允许中断等待操作
# nolock: 禁用文件锁，提升性能但可能不安全
EOF
```

**生产环境NFS最佳实践**：
- **多目录隔离**：为不同应用创建独立NFS目录，避免IO争抢
- **容量规划**：使用LVM管理NFS存储，便于后期扩容
- **性能监控**：使用`nfsstat`、`iostat`监控NFS性能
- **高可用**：使用DRBD+Heartbeat实现NFS主备，或使用Ceph/GlusterFS替代NFS

---

### 2.4 PV与PVC存储解耦体系

#### 2.4.1 PersistentVolume（集群级存储资源）

```bash
# 创建PV配置文件
vi pv-nfs.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
  labels:
    type: nfs         # 自定义标签，供PVC选择性绑定
    environment: prod # 可添加多个标签
spec:
  capacity:
    storage: 1Gi      # 存储容量，PVC申请量必须≤此值
  volumeMode: Filesystem  # 卷模式：Filesystem（文件系统）或Block（块设备）
  accessModes:
    - ReadWriteMany   # 访问模式：
    # ReadWriteOnce (RWO): 单节点读写（如AWS EBS）
    # ReadOnlyMany (ROX): 多节点只读
    # ReadWriteMany (RWX): 多节点读写（NFS支持）
  persistentVolumeReclaimPolicy: Retain  # 回收策略：
    # Retain: PVC删除后PV和数据保留（需手动清理，生产推荐）
    # Delete: PVC删除后自动删除PV和数据（云盘推荐）
    # Recycle: PVC删除后清空数据（已废弃，不推荐使用）
  storageClassName: nfs  # 存储类名称，必须与PVC匹配
  mountOptions:
    - hard              # 硬挂载（NFS服务不可用时程序会等待）
    - nfsvers=4.1       # 使用NFS v4.1协议（性能更好）
    - noatime           # 不更新访问时间，提升性能
  nfs:
    path: /nfs/data      # NFS服务器共享路径
    server: 192.168.200.100  # NFS服务器IP
    # readOnly: false   # 是否只读（默认false）
```

**创建与状态流转**：
```bash
kubectl apply -f pv-nfs.yaml
kubectl get pv -w
# 状态流转：
# Available → Bound → Released → Available（手动清理后）

# 查看PV详情
kubectl describe pv nfs-pv
# 关键信息：
# Source: NFS路径
# Claim: 绑定的PVC
# Status: 当前状态
# Access Modes: 支持的访问模式
# Capacity: 容量
```

**PV生命周期详解**：
1. **Available**：PV创建完成，等待PVC绑定
2. **Bound**：PVC成功绑定，PV被占用
3. **Released**：PVC被删除，但PV未回收
4. **Failed**：回收失败
5. **Available**：手动清理后重新可用

---

#### 2.4.2 PersistentVolumeClaim（命名空间级申请）

```bash
# 创建PVC配置文件
vi pvc-nfs.yaml
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
  namespace: default  # PVC是命名空间资源，不同NS可同名
spec:
  resources:
    requests:
      storage: 500Mi  # 申请500Mi存储（PV容量必须≥此值）
      # 注意：对于NFS，这只是逻辑限制，实际可超过
  accessModes:
    - ReadWriteMany   # 必须与PV的accessModes完全匹配
  storageClassName: nfs  # 必须与PV的storageClassName相同
  selector:           # 标签选择器（可选，精确匹配PV）
    matchLabels:
      type: nfs
    # matchExpressions:  # 复杂匹配规则
    #   - {key: environment, operator: In, values: [prod, staging]}
```

**创建与绑定验证**：
```bash
kubectl apply -f pvc-nfs.yaml
kubectl get pvc -w
# 状态：
# Pending → Bound（成功绑定PV）

# 查看PVC绑定详情
kubectl get pvc nfs-pvc -o yaml
# 关键字段：
# volumeName: 绑定的PV名称
# capacity.storage: 实际分配的容量

# 查看绑定关系
kubectl get pvc nfs-pvc -o yaml | grep volumeName
# 输出：  volumeName: nfs-pv

# 查看PV的绑定状态
kubectl get pv nfs-pv -o yaml | grep claimRef
# 输出包含绑定的PVC的namespace、name等信息
```

**PVC绑定PV的匹配规则**：
1. **storageClassName必须相同**：空字符串也视为相同
2. **accessModes必须匹配**：PVC的accessModes必须是PV的子集
3. **PV容量 ≥ PVC申请量**：500Mi的PVC可以绑定1Gi的PV
4. **selector标签匹配**（可选）：PV必须包含PVC指定的所有标签
5. **绑定是排他的**：一个PV同一时间只能绑定一个PVC

---

#### 2.4.3 在Pod中使用PVC

```bash
# 创建使用PVC的Pod
vi pod-with-pvc.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-storage
spec:
  containers:
  - name: app
    image: nginx:1.19.5
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: data-volume
      mountPath: /data  # 容器内挂载路径
      subPath: nginx    # 可选：使用卷中的子目录，避免数据污染
    - name: data-volume
      mountPath: /cache
      subPath: cache
  volumes:
  - name: data-volume
    persistentVolumeClaim:
      claimName: nfs-pvc  # 引用已创建的PVC名称
      readOnly: false     # 是否只读（默认false）
```

**创建与验证**：
```bash
kubectl apply -f pod-with-pvc.yaml

# 验证数据持久化
kubectl exec -it app-with-storage -- bash
echo "Persistent data test" > /data/test.txt
exit

# 删除Pod后重建
kubectl delete pod app-with-storage
kubectl apply -f pod-with-storage.yaml

# 验证数据是否保留
kubectl exec -it app-with-storage -- cat /data/test.txt
# 输出：Persistent data test （数据持久化成功）
```

**PV/PVC体系核心优势**：
- **存储与Pod解耦**：Pod删除重建不影响数据
- **存储提供者与使用者分离**：管理员管理PV，开发者申请PVC
- **跨命名空间共享**：不同NS的PVC可绑定不同PV
- **动态供给**：通过StorageClass自动创建PV

---

#### 2.4.4 StorageClass动态供给（进阶）

静态PV需要管理员手动创建，动态供给可自动创建PV。

```bash
# 创建StorageClass
vi storageclass-nfs.yaml
```

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-dynamic
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner
# provisioner: 指定动态供给驱动，需提前部署
parameters:
  archiveOnDelete: "true"  # PVC删除时归档而非真正删除
  pathPattern: "${.PVC.namespace}/${.PVC.name}"  # 目录命名模式
reclaimPolicy: Retain      # 回收策略
volumeBindingMode: Immediate  # 绑定模式：Immediate或WaitForFirstConsumer
```

**使用动态PVC**：
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  storageClassName: nfs-dynamic  # 指定StorageClass
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
```

**自动创建PV**：
```bash
kubectl apply -f dynamic-pvc.yaml
kubectl get pvc dynamic-pvc
# 自动创建PV并绑定，无需手动创建PV
```

---

## 第三部分：生产级工作负载管理

### 3.1 Deployment完整解析

#### 3.1.1 命令行创建与核心概念

```bash
# 快速创建Deployment（生产不推荐，仅用于测试）
kubectl create deployment my-tomcat \
  --image=tomcat:9.0.55 \
  --replicas=3  # 创建3个副本保证高可用

# 与Pod的本质区别：
# - 自愈能力：节点宕机时自动重建
# - 滚动更新：逐个替换Pod，业务不中断
# - 版本管理：记录ReplicaSet历史，支持回滚
# - 扩缩容：手动或自动调整副本数
```

**Deployment控制层级**：
```
Deployment (my-tomcat)
  ↓ 管理
ReplicaSet (my-tomcat-1v1wf)  # 版本控制层，带随机哈希
  ↓ 管理
Pods (my-tomcat-1v1wf-xxx)    # 实际运行实例
```

**查看控制关系**：
```bash
# 查看Deployment管理的ReplicaSet
kubectl get rs -l app=my-tomcat
# 输出：my-tomcat-1v1wf   3   3   3   5m

# 查看Pod的ownerReferences
kubectl get pod my-tomcat-1v1wf-xxxx -o yaml | grep -A 5 ownerReferences
# 输出显示该Pod由哪个ReplicaSet管理

# 验证自愈能力
kubectl delete pod my-tomcat-1v1wf-74gp9
kubectl get pod -l app=my-tomcat -w
# 立即出现新Pod，名称后缀变化
```

---

#### 3.1.2 生产级Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-tomcat
  namespace: default
  labels:
    app: my-tomcat
    version: v1
spec:
  replicas: 3
  # 滚动更新策略
  strategy:
    type: RollingUpdate  # 或Recreate（全量删除再创建）
    rollingUpdate:
      maxUnavailable: 1  # 最大不可用Pod数（绝对值或百分比）
      maxSurge: 1        # 最大超出Pod数（可临时多创建1个）
  # 版本回滚策略
  revisionHistoryLimit: 10  # 保留的历史版本数
  # 选择器（必须匹配template.labels）
  selector:
    matchLabels:
      app: my-tomcat
  # Pod模板
  template:
    metadata:
      labels:
        app: my-tomcat
        version: v1
      annotations:
        prometheus.io/scrape: "true"  # 监控采集注解
        prometheus.io/port: "8080"
    spec:
      # 镜像拉取Secret（私有仓库）
      imagePullSecrets:
      - name: aliyun-registry
      # 存储卷定义
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: tomcat-pvc
      - name: config
        configMap:
          name: tomcat-config
      # 初始化容器（在主容器启动前执行）
      initContainers:
      - name: init-data
        image: busybox
        command: ['sh', '-c', 'cp /tmp/index.html /data']
        volumeMounts:
        - name: data
          mountPath: /data
      # 主容器
      containers:
      - name: tomcat
        image: tomcat:9.0.55
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: http
          protocol: TCP
        # 环境变量
        env:
        - name: JAVA_OPTS
          value: "-Xmx512m -Xms512m"
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        # 资源限制
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        # 存储挂载
        volumeMounts:
        - name: data
          mountPath: /usr/local/tomcat/webapps
          subPath: webapps  # 使用PV中的子目录
        - name: config
          mountPath: /usr/local/tomcat/conf/server.xml
          subPath: server.xml
        # 健康检查
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        # 安全上下文
        securityContext:
          runAsNonRoot: false  # 是否以非root运行
          runAsUser: 0         # 运行用户ID
          privileged: false    # 是否特权模式
      # 重启策略
      restartPolicy: Always
      # 节点亲和性（调度策略）
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values: ["ssd"]
        podAntiAffinity:  # Pod反亲和（避免同一应用调度到同节点）
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: my-tomcat
              topologyKey: kubernetes.io/hostname
      # 容忍设置
      tolerations:
      - key: "node.kubernetes.io/unreachable"
        operator: "Exists"
        effect: "NoExecute"
        tolerationSeconds: 300  # 容忍300秒后驱逐
```

**滚动更新演示**：
```bash
# 查看当前版本
kubectl get deployment my-tomcat -o wide

# 更新镜像版本
kubectl set image deployment/my-tomcat tomcat=tomcat:9.0.56 --record
# 或修改YAML后 apply

# 查看滚动更新状态
kubectl rollout status deployment/my-tomcat
# 输出：Waiting for deployment "my-tomcat" rollout to finish: 1 out of 3 new replicas have been updated...

# 查看ReplicaSet历史
kubectl get rs -l app=my-tomcat
# 输出：my-tomcat-1v1wf   0   0   0   10m  (旧版本)
#       my-tomcat-2b2cg   3   3   3   5m   (新版本)

# 回滚到上一个版本
kubectl rollout undo deployment/my-tomcat
# 回滚到指定版本
kubectl rollout undo deployment/my-tomcat --to-revision=2

# 查看历史版本
kubectl rollout history deployment/my-tomcat
```

---

### 3.2 Service服务暴露详解

#### 3.2.1 Service类型对比与选择

```bash
# 创建Service暴露Deployment
kubectl expose deployment my-tomcat \
  --name=tomcat-svc \
  --port=8080 \        # Service端口（集群内访问）
  --target-port=8080 \ # Pod端口（容器端口）
  --type=NodePort      # Service类型
```

**四种Service类型深度对比**：

| 类型             | 使用场景           | 外部访问         | 性能开销   | 适用环境                     |
| ---------------- | ------------------ | ---------------- | ---------- | ---------------------------- |
| **ClusterIP**    | 集群内部微服务调用 | ❌ 不可           | 低         | 所有环境（默认）             |
| **NodePort**     | 开发测试环境       | ✅ 节点IP+端口    | 中         | 开发、小规模生产             |
| **LoadBalancer** | 公有云生产环境     | ✅ 自动分配公网IP | 依赖云厂商 | 公有云（AWS ELB、阿里云SLB） |
| **ExternalName** | 指向集群外服务     | CNAME解析        | 无         | 访问外部数据库、API          |

**NodePort端口范围问题**：
```bash
# 默认范围：30000-32767（共2768个端口）

# 修改端口范围（避免与业务端口冲突）
# 编辑kube-apiserver配置
vi /etc/kubernetes/manifests/kube-apiserver.yaml
# 在command下添加：
- --service-node-port-range=30000-39999

# 修改后apiserver会自动重启
kubectl get pod -n kube-system kube-apiserver-master -w

# 验证新范围
kubectl expose deployment test --port=80 --type=NodePort --dry-run=client -o yaml
# 查看nodePort是否在30000-39999范围内
```

---

#### 3.2.2 Service YAML生产配置

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-tomcat
  namespace: default
  labels:
    app: my-tomcat
  annotations:
    prometheus.io/scrape: "true"  # 监控注解
spec:
  type: NodePort
  # ClusterIP: None  # 设置为None可创建Headless Service，直接访问Pod IP
  # 端口定义（可定义多个）
  ports:
  - port: 8080        # Service的集群内端口
    targetPort: 8080  # 转发到Pod的端口
    nodePort: 30001   # 手动指定节点端口（可选，不指定则随机分配）
    protocol: TCP     # 协议：TCP或UDP
    name: http        # 端口名称，用于识别（多端口时必须命名）
  - port: 8443
    targetPort: 8443
    nodePort: 30002
    protocol: TCP
    name: https
  # 会话保持
  sessionAffinity: ClientIP  # 或None（默认）
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 会话保持超时时间
  # 外部IP（LoadBalancer类型使用）
  externalIPs:
  - 192.168.200.201
  # 后端Pod选择器（核心）
  selector:
    app: my-tomcat  # 匹配Pod的label，Service通过此字段找到后端Pod
  # 健康检查（1.21+）
  publishNotReadyAddresses: false  # 是否将未就绪Pod加入Endpoints
```

**Service工作原理详解**：
```bash
# Service通过标签选择器动态维护后端Pod列表
kubectl get endpoints my-tomcat -o yaml
# 输出包含所有就绪Pod的IP和端口

# 集群内DNS访问
# 格式：<service-name>.<namespace>.svc.cluster.local
curl http://my-tomcat.default.svc.cluster.local:8080

# 在不同命名空间访问
curl http://my-tomcat.prod.svc.cluster.local:8080

# 查看Service的IPTables规则（NodePort模式）
iptables-save | grep tomcat
# 显示KUBE-SVC-XXX链和DNAT规则
```

---

### 3.3 Ingress统一入口（HTTP/HTTPS）

#### 3.3.1 Ingress Controller部署

Ingress Controller是实现Ingress规则的反向代理（通常为Nginx/Traefik）。

```bash
# 步骤1：下载官方部署文件
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.47.0/deploy/static/provider/baremetal/deploy.yaml
# 文件内容详解：
# - Namespace: ingress-nginx
# - Deployment: ingress-nginx-controller
# - Service: ingress-nginx-controller (LoadBalancer类型)
# - ConfigMap: 存储Nginx配置
# - RBAC: 权限配置

# 步骤2：替换镜像为国内源（避免墙）
sed -i 's#k8s.gcr.io/ingress-nginx/controller#registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/ingress-nginx-controller#g' deploy.yaml
sed -i 's#k8s.gcr.io/ingress-nginx/kube-webhook-certgen#registry.cn-hangzhou.aliyuncs.com/lfy_k8s_images/kube-webhook-certgen#g' deploy.yaml

# 步骤3：修改部署模式为hostNetwork（生产推荐）
vi deploy.yaml
# 在spec.template.spec下添加：
hostNetwork: true          # 使用主机网络，性能更好，端口不冲突
dnsPolicy: ClusterFirstWithHostNet  # DNS策略
# 优势：
# - 避免NodePort的端口限制
# - 减少一层NAT，性能提升
# - 直接使用节点80/443端口
# 劣势：
# - 每个节点只能运行一个Controller实例
# - 与节点其他服务可能端口冲突

# 步骤4：应用并验证
kubectl apply -f deploy.yaml

# 步骤5：查看部署状态
kubectl get pod -n ingress-nginx -w
# 等待Controller Pod进入Running状态

# 步骤6：查看Service
kubectl get svc -n ingress-nginx ingress-nginx-controller
# 注意：使用hostNetwork模式时，此Service可能仍为ClusterIP，实际通过节点IP访问
```

**Ingress Controller工作原理**：
- **监听Ingress资源**：通过Watch机制实时获取集群Ingress规则
- **生成Nginx配置**：将规则转换为nginx.conf配置
- **热加载**：检测到配置变化后执行`nginx -s reload`无需重启
- **负载均衡**：将流量转发到Service的Endpoints（后端Pod）

---

#### 3.3.2 Ingress规则配置

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-tomcat-ingress
  namespace: default
  annotations:  # 注解用于配置Nginx行为
    nginx.ingress.kubernetes.io/rewrite-target: /  # URL重写
    nginx.ingress.kubernetes.io/ssl-redirect: "true"  # 强制HTTP跳转HTTPS
    nginx.ingress.kubernetes.io/rate-limit: "100"  # 限速100rps
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8"  # IP白名单
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"  # 上传大小限制
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"  # CORS配置
spec:
  ingressClassName: nginx  # 指定使用的Controller（支持多Controller）
  # TLS配置
  tls:
  - secretName: tls-secret  # 包含证书密钥的Secret
    hosts:
    - tomcat.example.com
  # 路由规则
  rules:
  - host: tomcat.example.com  # 域名路由（基于Host头）
    http:
      paths:
      - path: /                 # 路径匹配
        pathType: Prefix        # 路径类型：Prefix（前缀）、Exact（精确）、ImplementationSpecific（依赖实现）
        backend:
          service:
            name: my-tomcat     # 后端Service名称
            port:
              number: 8080      # Service端口
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-service-v1
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-service-v2
            port:
              number: 80
  # 默认后端（无匹配规则时）
  defaultBackend:
    service:
      name: default-http-backend
      port:
        number: 80
```

**创建TLS Secret**：
```bash
# 方式1：使用现有证书
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key

# 方式2：自签名证书（测试）
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=tomcat.example.com"

kubectl create secret tls tls-secret --cert=tls.crt --key=tls.key
```

**域名解析配置**：
```bash
# 本地hosts文件（开发测试）
# 编辑 /etc/hosts
192.168.200.101 tomcat.example.com  # 任意Node IP

# 内网DNS（生产环境）
# 添加A记录：tomcat.example.com → NodeIP

# 访问
curl http://tomcat.example.com  # 自动跳转到HTTPS
curl -k https://tomcat.example.com  # -k忽略自签名证书警告
```

---

## 第四部分：配置与安全管理

### 4.1 Secret敏感信息管理

#### 4.1.1 通用Secret类型

```bash
# 方式1：命令行创建（少量键值）
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password='Admin@123!'
# 命令详解：
# generic: 通用类型的Secret
# --from-literal: 从字面量创建，格式key=value
# Secret名称：db-secret

# 方式2：文件创建（证书、大文件）
kubectl create secret generic ssl-cert \
  --from-file=tls.crt=/path/to/cert.pem \
  --from-file=tls.key=/path/to/key.pem
# --from-file: 从文件创建，文件名作为key，内容作为value

# 方式3：目录批量创建
kubectl create secret generic configs \
  --from-file=./config/dir/
# 目录下所有文件都会被加入Secret

# 方式4：YAML创建（不推荐，需手动base64编码）
echo -n 'admin' | base64  # YWRtaW4=
echo -n 'Admin@123!' | base64  # QWRtaW5AMTIzIQ==
# 然后创建YAML
```

**查看与解码Secret**：
```bash
# 查看Secret（值会base64编码）
kubectl get secret db-secret -o yaml
# 输出：
# data:
#   password: QWRtaW5AMTIzIQ==
#   username: YWRtaW4=

# 解码查看
echo 'QWRtaW5AMTIzIQ==' | base64 -d
# 输出：Admin@123!

# 直接获取某个key
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

**Secret类型详解**：
- **Opaque**：通用类型（默认），用于密码、密钥等
- **kubernetes.io/service-account-token**：服务账户令牌
- **kubernetes.io/dockerconfigjson**：镜像仓库凭证
- **kubernetes.io/basic-auth**：Basic认证
- **kubernetes.io/ssh-auth**：SSH密钥
- **kubernetes.io/tls**：TLS证书

---

#### 4.1.2 Docker Registry Secret

```bash
# 创建阿里云私有仓库凭证
kubectl create secret docker-registry aliyun-registry \
  --docker-server=registry.cn-hangzhou.aliyuncs.com \
  --docker-username=fox666 \
  --docker-password='AliyunPwd123' \
  --docker-email=fox@example.com
# 参数详解：
# --docker-server: 仓库地址（域名+端口）
# --docker-username: 登录用户名
# --docker-password: 登录密码或访问令牌
# --docker-email: 邮箱（部分仓库要求）

# 查看凭证结构
kubectl get secret aliyun-registry --output="jsonpath={.data.\.dockerconfigjson}" | base64 -d
# 输出JSON格式：
# {
#   "auths": {
#     "registry.cn-hangzhou.aliyuncs.com": {
#       "username": "fox666",
#       "password": "AliyunPwd123",
#       "email": "fox@example.com"
#     }
#   }
# }

# 在Pod中使用（两种方式）
```

**在Pod中引用镜像拉取Secret**：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-registry-pod
spec:
  imagePullSecrets:  # 在spec级别定义
  - name: aliyun-registry
  containers:
  - name: app
    image: registry.cn-hangzhou.aliyuncs.com/fox666/app:v1
```

---

#### 4.1.3 在Pod中使用Secret

**方式1：环境变量注入（简单但不安全）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: app
    image: nginx:1.19.5
    imagePullPolicy: IfNotPresent
    env:
    - name: DB_USER          # 容器内变量名
      valueFrom:
        secretKeyRef:
          name: db-secret    # Secret名称
          key: username      # Secret中的key
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
    command:
    - sh
    - -c
    - echo "Database: $DB_USER/$DB_PASS && nginx -g 'daemon off;'"
```

**方式2：文件挂载（推荐，支持热更新）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-pod
spec:
  containers:
  - name: app
    image: nginx:1.19.5
    imagePullPolicy: IfNotPresent
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets  # 挂载路径
      readOnly: true          # 必须为只读
  volumes:
  - name: secret-volume
    secret:
      secretName: db-secret
      defaultMode: 0400       # 文件权限（八进制）
      items:  # 自定义文件名（可选）
      - key: username
        path: db-user.txt     # /etc/secrets/db-user.txt
        mode: 0444
      - key: password
        path: db-pass.txt     # /etc/secrets/db-pass.txt
        mode: 0400
```

**Secret热更新演示**：
```bash
# 1. 启动Pod
kubectl apply -f secret-volume-pod.yaml

# 2. 进入Pod查看文件
kubectl exec -it secret-volume-pod -- ls -l /etc/secrets/
# 输出：-r--r--r-- 1 root root 5 Dec 1 10:00 db-user.txt

# 3. 更新Secret
kubectl create secret generic db-secret \
  --from-literal=username=newadmin \
  --from-literal=password='NewPass123!' \
  --dry-run=client -o yaml | kubectl apply -f -

# 4. 等待约60秒（kubelet同步周期）
# 5. 验证文件内容已更新
kubectl exec -it secret-volume-pod -- cat /etc/secrets/db-user.txt
# 输出：newadmin

# 注意：环境变量方式不支持热更新，必须重启Pod
```

**Secret安全最佳实践**：
1. **启用RBAC**：限制Secret访问权限
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  resourceNames: ["db-secret"]
  verbs: ["get"]
```
2. **使用Vault或SealedSecret**：加密存储etcd中的数据
3. **定期轮换**：手动或结合Vault Operator自动轮换
4. **避免环境变量**：环境变量可通过`docker inspect`和`/proc`查看，安全性低
5. **网络加密**：启用TLS加密kubelet与apiserver通信

---

### 4.2 ConfigMap配置管理（补充）

ConfigMap用于存储非敏感配置信息。

```bash
# 创建ConfigMap
kubectl create configmap tomcat-config \
  --from-file=server.xml=./server.xml \
  --from-literal=env=prod

# 使用ConfigMap（与Secret类似）
```

---

## 第五部分：健康检查与自愈机制

### 5.1 三种探针详解

#### 5.1.1 LivenessProbe（存活探针）

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: app
    image: nginx:1.19.5
    livenessProbe:
      # 探测方式：httpGet、exec、tcpSocket
      httpGet:
        path: /healthz
        port: 8080
        scheme: HTTP      # 或HTTPS
        httpHeaders:      # 自定义请求头
        - name: X-Custom-Header
          value: Awesome
      # 探测时机参数
      initialDelaySeconds: 30  # 容器启动后等待30秒开始探测
      periodSeconds: 10        # 每10秒探测一次
      timeoutSeconds: 5        # 请求超时5秒
      successThreshold: 1      # 连续1次成功认为健康
      failureThreshold: 3      # 连续3次失败重启容器
      # 总失败容忍时间 = initialDelaySeconds + (periodSeconds * failureThreshold)
      # 本例：30 + (10*3) = 60秒
```

**作用**：检测容器是否存活，失败则重启容器。**适用于恢复死锁、内存泄漏等问题**。

---

#### 5.1.2 ReadinessProbe（就绪探针）

```yaml
readinessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2  # 连续2次失败从Service移除
```

**作用**：检测容器是否准备好接收流量，失败则从Service的Endpoints移除，不重启容器。**适用于应用启动慢、依赖外部服务场景**。

---

#### 5.1.3 StartupProbe（启动探针，1.16+）

```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 30  # 最多容忍300秒启动时间
  timeoutSeconds: 5
# 成功一次后，不再执行startupProbe，转由liveness/readiness接管
```

**作用**：保护慢启动应用，避免被过早重启。**适用于启动时间超过liveness容忍期的应用**。

---

### 5.2 完整探针使用案例

```bash
# 创建测试目录
mkdir -pv 1117 && cd 1117

# 1. Exec探针（检测文件存在）
vi liveness-exec.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-exec
spec:
  containers:
  - name: liveness
    image: busybox:1.35
    imagePullPolicy: IfNotPresent
    args:
    - /bin/sh
    - -c
    - touch /tmp/healthy; sleep 30; rm -rf /tmp/healthy; sleep 600
    # 容器启动流程：
    # 1. 创建健康标志文件
    # 2. 等待30秒（模拟正常运行）
    # 3. 删除文件（模拟故障）
    # 4. 等待600秒（保持容器运行）
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 10
      periodSeconds: 5
      failureThreshold: 1  # 立即重启
```

**执行与观察**：
```bash
# 分发镜像到节点（因使用IfNotPresent策略）
# scp busybox.tar k8s-node1:/root/
# scp busybox.tar k8s-node2:/root/
# ssh k8s-node1 "docker load -i /root/busybox.tar"
# ssh k8s-node2 "docker load -i /root/busybox.tar"

kubectl apply -f liveness-exec.yaml

# 实时监控Pod状态
kubectl get pod liveness-exec -w
# 30秒后开始重启：
# NAME            READY   STATUS    RESTARTS   AGE
# liveness-exec   1/1     Running   0          25s
# liveness-exec   1/1     Running   1          40s  # 第一次重启
# liveness-exec   1/1     Running   2          55s  # 第二次重启

# 查看重启原因
kubectl describe pod liveness-exec
# Events中显示：Liveness probe failed: cat: can't open '/tmp/healthy': No such file or directory

# 清理
kubectl delete -f liveness-exec.yaml
```

---

**2. HTTPGet探针（检测Web接口）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: liveness
    image: mydlqclub/springboot-helloworld:0.0.1
    imagePullPolicy: IfNotPresent
    livenessProbe:
      httpGet:
        path: /actuator/health  # Spring Boot健康检查端点
        port: 8080              # 实际端口，文档中8081可能是笔误
        scheme: HTTP
      initialDelaySeconds: 20   # 应用启动时间通常较长
      periodSeconds: 5
      timeoutSeconds: 10        # 请求超时10秒
      failureThreshold: 3       # 允许3次失败（共15秒）
```

**验证健康端点**：
```bash
# 启动Pod
kubectl apply -f liveness-http.yaml

# 等待Pod Running
kubectl get pod liveness-http -owide
# 获取Pod IP

# 手动测试健康接口
curl http://10.244.1.5:8080/actuator/health
# 应返回：{"status":"UP"}

# 实时查看日志
kubectl logs -f liveness-http --tail 100
# 观察日志确认接口正常响应

# 查看探针事件
kubectl describe pod liveness-http | grep -A 10 Liveness
```

---

**3. TCPSocket探针（端口检测）**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-tcp
spec:
  containers:
  - name: liveness
    image: nginx:1.19.5
    imagePullPolicy: IfNotPresent
    livenessProbe:
      tcpSocket:
        port: 80        # 只需端口可连接即认为健康
      initialDelaySeconds: 1
      periodSeconds: 20   # 较长间隔，减少资源消耗
      timeoutSeconds: 5
      failureThreshold: 2
```

**适用场景**：
- Redis、MySQL等TCP服务
- gRPC服务（无HTTP接口）
- 自定义TCP协议服务

---

### 5.3 探针参数总结与生产建议

| 参数                            | 含义         | 推荐值   | 说明                               |
| ------------------------------- | ------------ | -------- | ---------------------------------- |
| `initialDelaySeconds`           | 首次探测延迟 | 30-120秒 | 需大于应用平均启动时间             |
| `periodSeconds`                 | 探测间隔     | 10-30秒  | 太频繁增加负载，太稀疏延迟发现问题 |
| `timeoutSeconds`                | 超时时间     | 3-10秒   | 应小于探测间隔                     |
| `successThreshold`              | 成功阈值     | 1        | 通常保持1                          |
| `failureThreshold`              | 失败阈值     | 3-5      | 避免偶发网络波动导致重启           |
| `terminationGracePeriodSeconds` | 终止宽限期   | 30-60秒  | 在探针中可覆盖Pod级别的设置        |

**生产环境探针配置最佳实践**：
```yaml
# 完整生产配置示例
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 60  # 根据启动时间调整
  periodSeconds: 15
  timeoutSeconds: 5
  failureThreshold: 5      # 容忍5次失败（共75秒）
  successThreshold: 1
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 5
  timeoutSeconds: 3
  failureThreshold: 3      # 3次失败后从Service移除
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 30     # 最长容忍300秒启动时间
  timeoutSeconds: 5
```

**常见探针错误排查**：
```bash
# 错误1：Pod不断重启
kubectl describe pod <pod-name>
# 查看Liveness probe failed事件，检查路径、端口是否正确

# 错误2：Service无法访问
kubectl get endpoints <svc-name>
# 检查Pod是否通过Readiness probe，未通过不会加入Endpoints

# 错误3：启动慢导致被误杀
# 增加initialDelaySeconds或添加startupProbe
```

---

## 第六部分：完整生产环境部署案例

### 6.1 部署带持久化存储的高可用Tomcat应用

```bash
#!/bin/bash
set -e  # 遇到错误立即退出

echo "=== 步骤1：创建命名空间 ==="
kubectl create ns mall
# 命名空间作用：资源隔离、权限控制、资源配额

echo "=== 步骤2：创建PVC ==="
kubectl apply -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: tomcat-data
  namespace: mall
  labels:
    app: mall-tomcat
spec:
  accessModes: ["ReadWriteMany"]
  storageClassName: nfs
  resources:
    requests:
      storage: 2Gi  # 申请2Gi存储
EOF
# 检查PVC状态
kubectl get pvc tomcat-data -n mall -w
# 等待状态变为Bound

echo "=== 步骤3：创建Secret（数据库密码）==="
kubectl create secret generic db-pass \
  --from-literal=password='ComplexPass123!ComplexPass123!' \
  --from-literal=username='mall_admin' \
  -n mall
# 使用强密码，16位以上，包含大小写、数字、特殊字符
# 检查Secret是否创建
kubectl get secret db-pass -n mall -o yaml

echo "=== 步骤4：创建ConfigMap（应用配置）==="
kubectl create configmap tomcat-config \
  --from-literal=JAVA_OPTS='-Xmx1024m -Xms1024m -XX:+UseG1GC' \
  --from-literal=SPRING_PROFILES_ACTIVE='prod' \
  -n mall

echo "=== 步骤5：创建Deployment（2副本高可用）==="
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mall-tomcat
  namespace: mall
  labels:
    app: mall-tomcat
spec:
  replicas: 2  # 2副本保证高可用
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: mall-tomcat
  template:
    metadata:
      labels:
        app: mall-tomcat
    spec:
      imagePullSecrets:
      - name: aliyun-registry
      containers:
      - name: tomcat
        image: registry.cn-hangzhou.aliyuncs.com/fox666/tulingmall-product:0.0.5
        imagePullPolicy: IfNotPresent
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-pass
              key: password
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-pass
              key: username
        - name: JAVA_OPTS
          valueFrom:
            configMapKeyRef:
              name: tomcat-config
              key: JAVA_OPTS
        ports:
        - containerPort: 8080
          name: http
        volumeMounts:
        - name: data
          mountPath: /usr/local/tomcat/webapps
          subPath: webapps
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 15
          failureThreshold: 5
        readinessProbe:
          httpGet:
            path: /actuator/ready
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          failureThreshold: 30
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: tomcat-data
EOF
# 等待Pod就绪
kubectl get pod -n mall -w

echo "=== 步骤6：创建Service（NodePort暴露）==="
kubectl expose deployment mall-tomcat \
  --name=mall-tomcat-svc \
  --port=8080 \
  --target-port=8080 \
  --type=NodePort \
  -n mall
# 获取NodePort
NODE_PORT=$(kubectl get svc mall-tomcat-svc -n mall -o jsonpath='{.spec.ports[0].nodePort}')

echo "=== 步骤7：创建Ingress（域名访问）==="
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mall-ingress
  namespace: mall
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"  # 自动证书管理
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - mall.example.com
    secretName: mall-tls
  rules:
  - host: mall.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: mall-tomcat-svc
            port:
              number: 8080
EOF

echo "=== 步骤8：获取访问地址 ==="
NODE_IP=$(kubectl get node -o jsonpath='{.items[0].status.addresses[0].address}')
echo "HTTP访问: http://${NODE_IP}:${NODE_PORT}"
echo "HTTPS访问: https://mall.example.com (需配置DNS和证书)"

echo "=== 步骤9：配置资源限制 ==="
kubectl apply -f - <<EOF
apiVersion: v1
kind: LimitRange
metadata:
  name: mall-limits
  namespace: mall
spec:
  limits:
  - default:  # 默认limit
      memory: 1Gi
      cpu: 500m
    defaultRequest:  # 默认request
      memory: 512Mi
      cpu: 250m
    type: Container
EOF

echo "=== 部署完成 ==="
```

---

## 第七部分：系统化排错与调试

### 7.1 Pod无法启动排错流程

```bash
# 排错三步曲：描述→日志→执行

# 步骤1：查看Pod事件（定位问题阶段）
kubectl describe pod <pod-name>
# 关键事件：
# - FailedScheduling: 调度失败（资源不足、节点亲和性）
# - FailedMount: 挂载失败（PV/PVC问题、NFS不可达）
# - ImagePullBackOff: 镜像拉取失败（认证、网络）
# - CrashLoopBackOff: 容器启动后崩溃（应用错误）

# 步骤2：查看容器日志（定位应用错误）
kubectl logs <pod-name>
# 查看之前崩溃的日志
kubectl logs <pod-name> --previous
# 实时跟踪日志
kubectl logs -f <pod-name> --tail 100

# 步骤3：进入容器内部排查
kubectl exec -it <pod-name> -- /bin/sh
# 在容器内执行：
# - 检查文件权限：ls -l /data
# - 测试网络：curl http://my-tomcat:8080
# - 检查进程：ps aux
# - 查看资源：df -h, free -m
```

**常见Pod状态分析**：
- **Pending**：调度失败，检查describe events和StorageClass
- **ContainerCreating**：镜像拉取或存储挂载中，等待或检查网络
- **CrashLoopBackOff**：容器崩溃重启，查看logs --previous
- **ImagePullBackOff**：镜像问题，检查镜像名、Secret、网络
- **Terminating**：删除中，可能卡在finalizer，需强制删除

---

### 7.2 Service无法访问排错

```bash
# 步骤1：检查Service是否存在且类型正确
kubectl get svc <svc-name> -o wide
# 检查CLUSTER-IP、PORT(S)、TYPE

# 步骤2：检查Endpoints是否包含Pod IP
kubectl get endpoints <svc-name>
# 输出应包含后端Pod IP和端口
# 如果为空，检查：
# - Pod的label是否与Service的selector匹配
# - Pod是否通过ReadinessProbe（未通过不会加入Endpoints）

# 步骤3：从集群内测试Service连通性
kubectl run debug-tool --rm -it --image=busybox --restart=Never -- wget -O- <svc-name>:<port>
# 或使用curl镜像
kubectl run curl-test --rm -it --image=curlimages/curl --restart=Never -- curl <svc-name>:<port>

# 步骤4：检查kube-proxy和iptables
# 在节点上执行
iptables-save | grep <svc-ip>  # Service IP
# 查看KUBE-SERVICES链和DNAT规则

# 步骤5：检查CNI网络
kubectl get pod -o wide  # 确认Pod IP分配正常
# 在节点上ping Pod IP
ping <pod-ip>
# 使用tcpdump抓包
tcpdump -i any port <port> -nn
```

---

### 7.3 存储问题排错

```bash
# 步骤1：查看PV/PVC绑定状态
kubectl get pv,pvc -A

# 步骤2：检查PVC绑定详情
kubectl describe pvc <pvc-name> -n <namespace>
# 关键信息：
# - Capacity: 容量
# - Access Modes: 访问模式
# - Volume: 绑定的PV名称
# - Events: 绑定失败原因

# 步骤3：检查PV状态
kubectl describe pv <pv-name>
# 检查：
# - Status: 是否为Available或Bound
# - Claim: 绑定的PVC
# - Capacity: 容量

# 步骤4：在节点检查NFS挂载
df -h | grep nfs
# 检查挂载是否成功

# 步骤5：测试NFS写入权限
touch /nfs/data/test.txt
# 检查权限和所有者

# 步骤6：查看Pod挂载详情
kubectl get pod <pod-name> -o yaml | grep volumeMounts -A 5
```

**常见存储问题**：
- **PVC Pending**：无可用PV，检查storageClassName、accessModes、容量
- **FailedMount**：挂载失败，检查NFS服务、路径、权限
- **Permission denied**：NFS权限问题，检查exports和目录chmod

---

### 7.4 网络问题排错

```bash
# 步骤1：检查Pod IP分配
kubectl get pod -o wide
# 确认IP在集群CIDR范围内

# 步骤2：检查Service IP
kubectl get svc -o wide
# 确认CLUSTER-IP在service-cluster-ip-range范围内

# 步骤3：测试Pod间通信
kubectl exec -it pod-a -- ping <pod-b-ip>

# 步骤4：检查CoreDNS
kubectl get pod -n kube-system -l k8s-app=kube-dns
# 测试DNS解析
kubectl exec -it pod-a -- nslookup my-tomcat.default.svc.cluster.local

# 步骤5：抓包分析
# 在源Pod所在节点
tcpdump -i cni0 host <src-pod-ip> and host <dst-pod-ip> -nn
# cni0: CNI网桥接口

# 步骤6：检查路由
ip route show
# 检查Pod网段路由是否正确
```

---

### 7.5 强制删除资源

```bash
# Pod卡在Terminating状态
kubectl delete pod <pod-name> --grace-period=0 --force
# --grace-period=0: 立即删除，不等待优雅退出
# --force: 强制删除，绕过apiserver直接操作etcd

# 删除namespace卡在Terminating
kubectl get ns <ns-name> -o json | jq '.spec.finalizers = []' | kubectl replace --raw /api/v1/namespaces/<ns-name>/finalize -f -

# 删除PV卡在Released状态
kubectl patch pv <pv-name> -p '{"spec":{"claimRef": null}}'
```

**⚠️ 强制删除风险**：可能导致数据不一致、资源泄漏，生产环境慎用，优先解决根本原因。

---

## 总结与最佳实践

### Kubernetes核心设计思想
- **分层解耦**：Pod（应用）、Deployment（运维）、Service（访问）、PV/PVC（存储）、Secret（配置）、Ingress（入口）各层独立
- **声明式API**：描述期望状态，系统自动调和实际状态
- **自愈能力**：通过探针、控制器实现自动恢复
- **弹性伸缩**：手动或自动扩缩容应对负载变化

### 生产环境 checklist
- ✅ 所有Pod设置resources limits
- ✅ 使用PV/PVC持久化数据，避免emptyDir/empty
- ✅ 配置liveness/readiness/startup探针
- ✅ 使用Secret管理敏感信息，避免明文
- ✅ 镜像使用固定tag，避免latest
- ✅ 启用RBAC限制权限
- ✅ 配置PodDisruptionBudget保障可用性
- ✅ 使用Namespace隔离环境
- ✅ 配置资源配额（ResourceQuota）
- ✅ 使用NetworkPolicy限制网络访问
- ✅ 启用审计日志（Audit Log）
- ✅ 配置监控告警（Prometheus+Grafana）
- ✅ 定期备份etcd数据

### 推荐学习路径
1. 掌握Pod基础与生命周期
2. 理解PV/PVC存储体系
3. 熟练使用Deployment/StatefulSet
4. 配置Service和Ingress
5. 实现健康检查与自愈
6. 配置RBAC和安全策略
7. 部署监控日志系统
8. 实践CI/CD流水线

---

**文档结束语**：Kubernetes是一个复杂的分布式系统，掌握其核心概念和最佳实践需要持续学习和实践。建议从简单的Pod开始，逐步深入到控制器、存储、网络、安全等各个层面，结合实际项目不断积累经验。