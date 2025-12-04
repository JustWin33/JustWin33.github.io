---
title: 运维全栈技术
description:
date: 2025-12-04
categories:
    - 
    - 
---

# 企业级 Linux 运维全栈技术（上）

| 文档版本     | v2.0 (终极修订版)                                    |
| :----------- | :--------------------------------------------------- |
| **发布日期** | 2025-12-04                                           |
| **适用环境** | CentOS 7.9 / Rocky Linux 9 / Ubuntu 22.04            |
| **主要修订** | 修正过时命令、适配 MySQL 8.0/K8s 1.25+、补充底层原理 |
| **作者**     | 企业级 Linux 运维架构师                              |

-----

## 目录

1.  **Linux 系统内核与存储管理**
      * 基础原理：文件系统、LVM、Load Average
      * 生产实践：分区标准、LVM 管理、内核调优
2.  **网络技术：内核栈与防火墙**
      * 基础原理：TCP状态机、Netfilter 框架
      * 生产实践：现代化网络命令、防火墙策略
3.  **Web 架构：Nginx 高级应用**
      * 基础原理：IO多路复用、负载均衡算法
      * 生产实践：高并发配置、健康检查方案
4.  **数据库运维：MySQL 8.0 体系**
      * 基础原理：InnoDB 架构、ACID、MVCC
      * 生产实践：配置文件优化、XtraBackup 8.0 备份
5.  **云原生架构：Docker 与 Kubernetes**
      * 基础原理：Namespace/Cgroup、CRI/CNI/CSI 标准
      * 生产实践：Containerd 配置、K8s 故障排查 SOP
6.  **自动化运维与脚本编程**
      * 基础原理：Shell 执行机制、信号处理
      * 生产实践：健壮性编程规范

-----

## 1\. Linux 系统内核与存储管理

### 1.1 核心基础知识点 (Knowledge Focus)

  * **Inode 与 Block**：
      * **Block**：文件系统中存储数据的最小单元（通常 4KB）。大文件占用多个 Block。
      * **Inode**：索引节点，存储文件的元数据（权限、所有者、时间戳、Block 位置等）。文件名实际上存储在目录的 Block 中，而非 Inode 中。
      * *运维启示*：磁盘空间未满但无法写入文件，通常是 Inode 耗尽（大量小文件导致）。
  * **LVM (Logical Volume Manager)**：
      * 通过将物理硬盘（PV）抽象为卷组（VG），再从卷组划分逻辑卷（LV）。
      * *核心优势*：支持在线动态扩容，无需停机。
  * **Load Average (平均负载)**：
      * 指单位时间内处于**可运行状态**（Running）和**不可中断状态**（Uninterruptible sleep, 通常是 IO 等待）的进程平均数。
      * *误区*：Load 高不一定是 CPU 忙，也可能是磁盘 I/O 阻塞。

### 1.2 生产环境初始化标准流程

#### 1.2.1 磁盘分区与 LVM 配置（修正版）

*适用场景：大于 2TB 的数据盘，使用 GPT 分区表*

```bash
# 1. 创建 PV 和 VG
# 使用 parted 处理 GPT 分区 (fdisk 不支持 >2TB)
parted /dev/sdb mklabel gpt
parted /dev/sdb mkpart primary 0% 100%
parted /dev/sdb set 1 lvm on

pvcreate /dev/sdb1
vgcreate vg_data /dev/sdb1

# 2. 创建逻辑卷 (LV)
# 修正：不建议初次分配所有空间，预留 20% 给快照或紧急扩容
lvcreate -L 800G -n lv_mysql vg_data
lvcreate -L 200G -n lv_redis vg_data
lvcreate -L 500G -n lv_logs vg_data

# 3. 格式化与挂载
# 修正：CentOS 7/8 推荐 XFS (格式化快、支持大文件)
mkfs.xfs /dev/vg_data/lv_mysql
mkfs.xfs /dev/vg_data/lv_redis

# 4. /etc/fstab 挂载参数优化
# noatime: 读取文件不更新 access time，减少 IO
# nodiratime: 不更新目录 access time
/dev/mapper/vg_data-lv_mysql  /data/mysql   xfs  defaults,noatime,nodiratime,inode64 0 0
```

#### 1.2.2 系统内核参数调优

*文件：/etc/sysctl.d/99-production.conf*

```ini
# --- 内存管理 ---
# 尽量少用 Swap，但设为 1 防止 OOM 立即杀进程 (0 在某些内核版本有 Bug)
vm.swappiness = 1
# 允许分配超过物理内存 (Redis RDB 或 Java 应用必备)
vm.overcommit_memory = 1

# --- 网络栈 (防 SYN Flood 与 高并发) ---
# 全连接队列长度 (Socket Listen 队列)
net.core.somaxconn = 65535
# 半连接队列长度 (SYN 队列)
net.ipv4.tcp_max_syn_backlog = 65535

# --- TCP 连接复用 ---
# 允许将 TIME_WAIT socket 用于新的 TCP 连接
net.ipv4.tcp_tw_reuse = 1
# 警告：必须关闭 tcp_tw_recycle，在 NAT 环境下会导致连接失败
net.ipv4.tcp_tw_recycle = 0
# 缩短 FIN 等待时间
net.ipv4.tcp_fin_timeout = 15

# --- 端口范围 ---
net.ipv4.ip_local_port_range = 1024 65535
```

-----

## 2\. 网络技术：内核栈与防火墙

### 2.1 核心基础知识点 (Knowledge Focus)

  * **TCP 三次握手与四次挥手**：
      * **握手**：SYN -\> SYN+ACK -\> ACK。建立连接。
      * **挥手**：FIN -\> ACK -\> FIN -\> ACK。断开连接。
      * *运维重点*：大量的 `SYN_RECV` 意味着遭受 SYN Flood 攻击；大量的 `TIME_WAIT` 意味着高并发短连接且未开启连接复用。
  * **Netfilter 框架**：
      * Linux 内核的网络包过滤框架。iptables、nftables、firewalld 都是基于 Netfilter 的用户态工具。
      * **四表五链**：Filter 表（过滤）、NAT 表（转发）。PREROUTING -\> INPUT -\> FORWARD -\> OUTPUT -\> POSTROUTING。
  * **OSI 七层模型**：
      * L4 (传输层)：TCP/UDP，负载均衡（LVS/Haproxy TCP模式）。
      * L7 (应用层)：HTTP/FTP，负载均衡（Nginx/Haproxy HTTP模式）。

### 2.2 现代化网络命令集 (iproute2)

*修正：废弃 `net-tools` (ifconfig/netstat)，使用 `iproute2`*

| 需求         | 旧命令 (Deprecated) | **新命令 (Recommended)** | 优势原理                                            |
| :----------- | :------------------ | :----------------------- | :-------------------------------------------------- |
| **查看 IP**  | `ifconfig`          | `ip addr show`           | 显示辅助 IP 和链路状态                              |
| **查看路由** | `route -n`          | `ip route`               | 支持策略路由表                                      |
| **查看端口** | `netstat -tulnp`    | `ss -tulnp`              | 直接读取内核 sock\_diag，不遍历 /proc，高并发下极快 |
| **网络统计** | `netstat -s`        | `nstat`                  | 统计计数器更准确                                    |
| **ARP 表**   | `arp -n`            | `ip neigh`               | 统一管理邻居表                                      |

### 2.3 生产级防火墙配置 (Firewalld 方案)

*CentOS 7+ 标准化方案，底层操作 nftables*

```bash
# 1. 默认区域设置
firewall-cmd --set-default-zone=public

# 2. 开放 SSH 运维端口 (非 22)
firewall-cmd --permanent --add-port=22888/tcp
firewall-cmd --permanent --remove-service=ssh

# 3. 开放 Web 服务
firewall-cmd --permanent --add-service=http
firewall-cmd --permanent --add-service=https

# 4. 信任内部网段 (如数据库访问)
# 允许 192.168.10.0/24 网段的所有访问
firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.10.0/24" accept'

# 5. 生效配置
firewall-cmd --reload
```

-----

## 3\. Web 架构：Nginx 高级应用

### 3.1 核心基础知识点 (Knowledge Focus)

  * **IO 多路复用 (IO Multiplexing)**：
      * **Select/Poll**：轮询所有连接，效率随连接数增加而下降。
      * **Epoll** (Linux)：事件驱动。只处理活跃的连接，无轮询开销。Nginx 高性能的核心。
  * **正向代理 vs 反向代理**：
      * **正向代理**：代理**客户端**上网（如 VPN）。服务端不知道真实客户端是谁。
      * **反向代理**：代理**服务端**接收请求（如 Nginx）。客户端不知道真实服务端是谁。
  * **负载均衡算法**：
      * Round Robin (轮询)、Weight (加权)、IP Hash (会话保持)、Least Conn (最少连接)。

### 3.2 Nginx 生产配置与修正

#### 3.2.1 性能与并发优化

```nginx
user nginx;
# 自动匹配 CPU 核心数，减少进程切换
worker_processes auto;
# 绑定 CPU 亲和性
worker_cpu_affinity auto;

# 文件句柄限制 (需配合 ulimit -n)
worker_rlimit_nofile 65535;

events {
    # Linux 必选 epoll
    use epoll;
    # 单个 Worker 最大连接
    worker_connections 65535;
    # 收到新连接通知时，尽可能多地接受连接
    multi_accept on;
}
```

#### 3.2.2 Upstream 健康检查（重要修正）

*误区纠正：开源版 Nginx 不支持 `check` 参数，需使用 Tengine 或被动检查。*

```nginx
http {
    upstream backend_server {
        # 负载均衡算法：最少连接数 (适合长连接服务)
        least_conn;
        
        # 生产级被动检查配置：
        # 30秒内失败 3 次，则暂停分发请求 30 秒
        server 192.168.1.101:8080 weight=5 max_fails=3 fail_timeout=30s;
        server 192.168.1.102:8080 weight=5 max_fails=3 fail_timeout=30s;
        
        # Keepalive 连接池 (对后端保持长连接，减少握手消耗)
        keepalive 300;
    }
    
    server {
        location /api/ {
            proxy_pass http://backend_server;
            # 必须配置，否则 keepalive 不生效
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

-----

## 4\. 数据库运维：MySQL 8.0 体系

### 4.1 核心基础知识点 (Knowledge Focus)

  * **InnoDB 架构**：
      * **Buffer Pool**：内存缓冲区，缓存数据页和索引页。性能关键。
      * **Redo Log** (重做日志)：保证事务的**持久性** (Durability)。Crash 恢复使用。
      * **Undo Log** (回滚日志)：保证事务的**原子性** (Atomicity) 和 MVCC。
  * **ACID 特性**：
      * Atomicity (原子性)、Consistency (一致性)、Isolation (隔离性)、Durability (持久性)。
  * **隔离级别**：
      * Read Committed (RC)：防止脏读。
      * Repeatable Read (RR)：默认级别，防止不可重复读，配合 Next-Key Lock 防止幻读。
  * **GTID (Global Transaction ID)**：
      * 全局事务 ID，彻底解决了主从同步中断后寻找 Binlog 位点的难题。

### 4.2 MySQL 8.0 生产实践

#### 4.2.1 配置文件 (my.cnf) 关键变更

```ini
[mysqld]
# 8.0 默认认证插件改为 caching_sha2_password，旧客户端可能连不上
# 兼容配置：
default_authentication_plugin = mysql_native_password

# Binlog 设置
log_bin = mysql-bin
binlog_format = ROW
# 8.0 新参数，单位秒 (此处为7天)，取代 expire_logs_days
binlog_expire_logs_seconds = 604800
server_id = 101
gtid_mode = ON
enforce_gtid_consistency = ON

# InnoDB 优化
# 物理内存的 70-80%
innodb_buffer_pool_size = 12G
# 刷盘策略：1=最安全(每次提交刷盘)，2=高性能(每秒刷盘，宕机丢1秒)
innodb_flush_log_at_trx_commit = 1
# Redo Log 刷盘方式，Linux 下推荐 O_DIRECT 绕过文件系统缓存
innodb_flush_method = O_DIRECT
```

#### 4.2.2 物理备份：XtraBackup 8.0 (修正版)

*注意：MySQL 8.0 必须使用 Percona XtraBackup 8.0，且移除了 `innobackupex` 命令。*

**全量备份脚本片段**：

```bash
# 定义变量
BACKUP_ROOT="/data/backup"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
TARGET_DIR="${BACKUP_ROOT}/full_${TIMESTAMP}"

# 执行备份 (使用 xtrabackup 命令)
# --no-lock 参数在 8.0 中已变更行为，工具会自动处理锁，通常不需要手动加
xtrabackup --backup \
  --user=backup_ops \
  --password='ComplexPassword123!' \
  --target-dir=${TARGET_DIR} \
  --datadir=/data/mysql/data \
  --compress --parallel=4

# 备份预处理 (Prepare) - 恢复前必须执行，也可在恢复时执行
# xtrabackup --prepare --target-dir=${TARGET_DIR}
```

-----

## 5\. 云原生架构：Docker 与 Kubernetes

### 5.1 核心基础知识点 (Knowledge Focus)

  * **容器底层原理**：
      * **Namespace (命名空间)**：实现资源**隔离**（PID, Network, Mount, User, UTS, IPC）。
      * **Cgroups (控制组)**：实现资源**限制**（CPU 核数, 内存大小）。
      * **UnionFS (联合文件系统)**：实现镜像分层（Overlay2）。
  * **K8s 标准接口**：
      * **CRI** (Container Runtime Interface)：容器运行时接口（连接 Containerd/CRI-O）。
      * **CNI** (Container Network Interface)：网络接口（连接 Calico/Flannel）。
      * **CSI** (Container Storage Interface)：存储接口（连接 Ceph/NFS/EBS）。
  * **Pod 概念**：
      * K8s 最小调度单元。同一个 Pod 内的容器共享 Network Namespace (同一个 IP) 和存储卷。

### 5.2 Kubernetes (v1.25+) 生产实践

#### 5.2.1 运行时迁移：Containerd 配置

*修正：K8s 1.24+ 移除了 Dockershim，必须配置 Containerd。*

文件：`/etc/containerd/config.toml`

```toml
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  # 关键配置：必须使用 systemd 作为 cgroup 驱动，否则 kubelet 启动失败
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

[plugins."io.containerd.grpc.v1.cri"]
  # 替换为国内镜像源 (阿里云/腾讯云)，解决 pause 镜像拉取失败问题
  sandbox_image = "registry.aliyuncs.com/google_containers/pause:3.9"
```

#### 5.2.2 K8s 常见故障排查 SOP (整合版)

**场景 1：Pod 状态 `CrashLoopBackOff`**

  * **原理**：容器进程启动后退出，或健康检查失败。
  * **排查**：
    1.  `kubectl logs <pod> --previous`：查看上一次崩溃前的日志（关键）。
    2.  `kubectl describe pod <pod>`：查看 Liveness Probe 是否失败，或 OOMKilled（Exit Code 137）。

**场景 2：Pod 状态 `Pending`**

  * **原理**：调度失败。
  * **排查**：
    1.  `kubectl describe pod <pod>`：查看 Events。
    2.  **资源不足**：CPU/Mem 请求超过节点可分配值。
    3.  **污点 (Taint)**：节点有污点，Pod 无对应容忍度 (Toleration)。
    4.  **亲和性 (Affinity)**：NodeAffinity 规则不满足。

**场景 3：Service 无法访问 (Connection Refused)**

  * **排查**：
    1.  `kubectl get endpoints <svc-name>`：**最关键一步**。如果 endpoints 为空，说明 Selector 没匹配到 Pod，或者 Pod 未通过 Readiness Probe（未就绪）。
    2.  检查 Pod 应用是否监听了 `0.0.0.0` 而非 `127.0.0.1`。

-----

## 6\. 自动化运维与脚本编程

### 6.1 核心基础知识点 (Knowledge Focus)

  * **Shell 执行过程**：
      * 解析器读取脚本 -\> 变量替换/通配符扩展 -\> 执行命令。
      * **Subshell (子 shell)**：`()` 或 管道符 `|` 会启动子 Shell，子 Shell 中的变量修改不会影响父 Shell。
  * **标准输入输出**：
      * 0: stdin, 1: stdout, 2: stderr。
      * `2>&1`：将错误输出重定向到标准输出。
  * **信号 (Signal)**：
      * `kill -15 (SIGTERM)`：请求进程优雅退出（程序可以捕获并清理）。
      * `kill -9 (SIGKILL)`：强制杀死，不可捕获，可能导致数据损坏。

### 6.2 健壮性 Shell 脚本规范

*修正：增加严格模式和日志函数*

```bash
#!/bin/bash
# ==========================================
# Description: 生产级自动化脚本模板
# Author: DevOps Team
# ==========================================

# --- 严格模式 (Safety First) ---
set -o errexit   # 遇到任何错误立即退出 (等同 set -e)
set -o nounset   # 使用未定义变量时报错 (等同 set -u)
set -o pipefail  # 管道中只要有一个命令失败，整个管道视为失败

# --- 全局变量 ---
readonly LOG_FILE="/var/log/ops_script.log"
readonly DATE_TAG=$(date +%Y-%m-%d)

# --- 日志函数 ---
log_info() {
    echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') $1" | tee -a "$LOG_FILE"
}

log_error() {
    echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') $1" | tee -a "$LOG_FILE" >&2
}

# --- 错误捕获与清理 ---
cleanup() {
    # 脚本退出时（无论成功失败）都会执行
    rm -f /tmp/script.lock
    log_info "Script finished or interrupted. Cleanup done."
}
trap cleanup EXIT

# --- 主逻辑 ---
main() {
    log_info "Starting backup process..."
    
    # 检查 root 权限
    if [[ $EUID -ne 0 ]]; then
       log_error "This script must be run as root"
       exit 1
    fi
    
    # 业务逻辑...
}

main "$@"
```





# 企业级 Linux 运维全栈技术（下）

**Enterprise Linux Operations & Architecture Whitepaper**

| 文档版本       | v2.1 (基于实战增强版)                                     |
| :------------- | :-------------------------------------------------------- |
| **新增模块**   | DevOps流水线、Zabbix监控标准、等保2.0合规、Python运维开发 |
| **技术栈补充** | Jenkins/GitLab CI, OpenVAS, Python, StatefulSet, Harbor   |
| **参考项目**   | 核心业务容器化改造、等保三级整改、全栈监控平台            |

-----

## 1\. DevOps 与 CI/CD 流水线架构

[cite_start]*(基于简历项目一：核心业务系统 Kubernetes 容器化改造 [cite: 38, 42])*

### 1.1 核心基础知识点 (Knowledge Focus)

  * **CI/CD (持续集成/持续交付)**：
      * **CI (Continuous Integration)**：代码提交后自动触发构建和测试，尽早发现错误。
      * **CD (Continuous Delivery/Deployment)**：自动将代码部署到生产环境。
  * **流水线 (Pipeline)**：
      * 定义了从代码提交到上线的全过程：`Code -> Build -> Test -> Release -> Deploy`。
  * **镜像分层构建**：
      * 利用 Docker 缓存机制加速构建，利用多阶段构建（Multi-stage builds）减小镜像体积。

### 1.2 生产级流水线标准配置

[cite_start]**技术栈**：Jenkins + GitLab CI + Harbor [cite: 42, 45]

#### 1.2.1 镜像标准化与瘦身策略

[cite_start]简历中提到将镜像体积从 800MB 压缩至 250MB [cite: 41]，标准 Dockerfile 优化规范如下：

```dockerfile
# 阶段一：构建环境 (Build Stage)
FROM maven:3.8-jdk-11 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline  # 缓存依赖
COPY src ./src
RUN mvn package -DskipTests

# 阶段二：运行时环境 (Runtime Stage)
# 选用轻量级基础镜像 (如 Alpine 或 Distroless)
FROM openjdk:11-jre-slim
WORKDIR /app
# 仅复制构建产物，丢弃编译工具和源码
COPY --from=builder /app/target/app.jar /app/app.jar

# 设置时区与非 root 用户运行 (安全合规)
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    useradd -m -u 1000 appuser
USER 1000

ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### 1.2.2 Jenkinsfile 流水线脚本模板

[cite_start]实现代码提交到 K8s 滚动发布的全流程自动化 [cite: 42]：

```groovy
pipeline {
    agent any
    environment {
        HARBOR_REGISTRY = "registry.example.com"
        IMAGE_NAME = "project/backend"
        TAG = "${env.BUILD_NUMBER}"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build & Push') {
            steps {
                sh "docker build -t ${HARBOR_REGISTRY}/${IMAGE_NAME}:${TAG} ."
                // 需在 Jenkins 配置好 Harbor 的凭证
                withCredentials([usernamePassword(credentialsId: 'harbor-auth', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh "docker login ${HARBOR_REGISTRY} -u $USER -p $PASS"
                    sh "docker push ${HARBOR_REGISTRY}/${IMAGE_NAME}:${TAG}"
                }
            }
        }
        stage('Deploy to K8s') {
            steps {
                // 使用 sed 替换 K8s yaml 中的镜像版本并应用
                sh "sed -i 's|IMAGE_TAG|${TAG}|g' k8s/deployment.yaml"
                sh "kubectl apply -f k8s/deployment.yaml"
            }
        }
    }
}
```

-----

## 2\. 全栈监控体系：Zabbix + Prometheus

[cite_start]*(基于简历项目二：全栈监控与自动化运维平台建设项目 [cite: 47, 51])*

### 2.1 核心基础知识点 (Knowledge Focus)

  * **监控分层**：
      * [cite_start]**基础设施层**：CPU、内存、磁盘 I/O、网络流量（top, vmstat, iostat）[cite: 19]。
      * [cite_start]**中间件层**：Nginx 连接数、MySQL 慢查询/QPS、Redis 命中率 [cite: 51]。
      * **业务层**：接口响应时间、错误率、订单量。
  * **推拉模型**：
      * **Zabbix (Push/Pull)**：适合物理机、虚拟机等静态资产监控，报警功能强大。
      * **Prometheus (Pull)**：适合 K8s 等动态云原生环境，基于时序数据库 TSDB。

### [cite_start]2.2 Zabbix 5.0 企业级部署标准 [cite: 51]

**适用场景**：300+ Linux 服务器及传统中间件监控。

1.  **分级告警策略配置**：
      * **P0 (严重)**：服务宕机、端口不通。 -\> 电话/短信通知。
      * **P1 (警告)**：CPU \> 90%，磁盘使用率 \> 85%。 -\> 邮件/钉钉通知。
      * **P2 (信息)**：系统重启、配置变更。 -\> 仅记录日志。
2.  **自定义监控脚本 (UserParameter)**：
      * *案例：监控 TCP 连接状态*
    <!-- end list -->
    ```bash
    # /etc/zabbix/zabbix_agentd.d/tcp_status.conf
    UserParameter=tcp.status[*],/usr/bin/netstat -an | grep -c $1
    # 使用: zabbix_get -s IP -k tcp.status[ESTABLISHED]
    ```

### [cite_start]2.3 Prometheus + Grafana 云原生监控 [cite: 43]

**适用场景**：K8s 集群节点、Pod、容器资源监控。

  * **架构**：Node Exporter (节点监控) + Kube-state-metrics (K8s 对象状态) + cAdvisor (容器资源)。
  * **关键告警规则 (PromQL)**：
    ```yaml
    # 节点内存不足告警
    - alert: NodeMemoryUsageHigh
      expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.9
      for: 5m
      labels:
        severity: warning
    ```

-----

## 3\. Python 自动化运维开发

[cite_start]*(基于简历项目二及专业能力：Python 工具开发 [cite: 53, 54])*

### 3.1 核心基础知识点 (Knowledge Focus)

  * **Python 在运维中的优势**：
      * 相比 Shell，Python 更适合处理复杂的逻辑、数据分析（如日志分析）、API 调用和数据库操作。
  * **常用库**：
      * `os`, `sys`, `subprocess` (系统交互)
      * `pymysql` (数据库操作)
      * `requests` (API 调用)

### 3.2 实战：MySQL 慢查询自动优化工具

[cite_start]简历中提到“自研 MySQL 慢查询自动优化工具，将慢查询率降低 70%” [cite: 54]。

**工具实现逻辑范例**：

```python
#!/usr/bin/env python3
import pymysql

# 配置数据库连接
db_config = {
    'host': '192.168.1.100',
    'user': 'monitor',
    'password': 'password',
    'db': 'information_schema'
}

def analyze_slow_queries():
    conn = pymysql.connect(**db_config)
    cursor = conn.cursor()
    
    # 1. 查询执行时间超过 2秒 的 TOP 10 SQL
    sql = """
    SELECT sql_text, query_time, rows_examined 
    FROM slow_log 
    WHERE query_time > '00:00:02' 
    ORDER BY query_time DESC LIMIT 10;
    """
    cursor.execute(sql)
    results = cursor.fetchall()
    
    for row in results:
        sql_text = row[0]
        # 2. 简单的 EXPLAIN 分析逻辑 (模拟)
        if "select *" in sql_text.lower():
            print(f"[优化建议] 避免全字段查询: {sql_text[:50]}...")
        if row[2] > 10000: # rows_examined
            print(f"[优化建议] 扫描行数过多({row[2]})，建议添加索引: {sql_text[:50]}...")
            
    conn.close()

if __name__ == "__main__":
    analyze_slow_queries()
```

-----

## 4\. 安全合规：等保 2.0 三级整改

[cite_start]*(基于简历项目三：等保 2.0 三级安全合规整改项目 [cite: 57, 59])*

### 4.1 核心基础知识点 (Knowledge Focus)

  * **等保 2.0 (MLPS 2.0)**：中国网络安全等级保护制度。三级系统要求较高，涉及物理、网络、主机、应用、数据五个层面。
  * [cite_start]**OpenVAS**：开源漏洞扫描器，用于发现系统漏洞（如未打补丁的内核、弱密码服务） [cite: 24, 63]。
  * **基线检查 (Baseline)**：系统最小化安全配置标准（如密码复杂度、SSH 配置、文件权限）。

### [cite_start]4.2 生产级安全加固标准 (Ansible 实现) [cite: 60]

通过 Ansible 批量执行加固脚本，满足合规要求。

```yaml
# secure_baseline.yml
- name: 等保三级主机加固
  hosts: all
  tasks:
    # 1. 密码复杂度策略 (pam_pwquality)
    - name: 设置密码复杂度
      lineinfile:
        path: /etc/security/pwquality.conf
        regexp: '^minlen'
        line: 'minlen = 12' # 最小长度12位
    
    # 2. SSH 安全配置
    - name: 禁止 root 远程登录
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^PermitRootLogin'
        line: 'PermitRootLogin no'
      notify: restart sshd

    # 3. 设置会话超时 (防挂机)
    - name: 设置 TMOUT
      lineinfile:
        path: /etc/profile
        line: 'export TMOUT=300' # 5分钟无操作自动登出
        
  handlers:
    - name: restart sshd
      service:
        name: sshd
        state: restarted
```

-----

## 5\. Kubernetes 存储与有状态应用

[cite_start]*(基于简历项目一：StatefulSet 与 PV/PVC 应用 [cite: 41])*

### 5.1 核心基础知识点 (Knowledge Focus)

  * **StatefulSet**：
      * 用于管理**有状态**应用（如 MySQL, Redis）。
      * 特点：稳定的网络标识（Pod名固定 `mysql-0`, `mysql-1`）、有序部署、稳定的持久化存储。
  * **PV (PersistentVolume) 与 PVC (Claim)**：
      * **PV**：集群中的存储资源（由管理员创建，连接 NFS/Ceph）。
      * **PVC**：用户对存储的申请（Pod 绑定 PVC，PVC 绑定 PV）。
      * **StorageClass**：实现 PV 的动态供给。

### 5.2 生产级 MySQL on K8s 配置范例

[cite_start]简历中提到使用 PV/PVC 动态供应持久化存储 [cite: 41]。

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  serviceName: "mysql"
  replicas: 1
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
        volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
  # 关键：volumeClaimTemplates (自动创建 PVC)
  volumeClaimTemplates:
  - metadata:
      name: mysql-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "managed-nfs-storage" # 需提前配置 StorageClass
      resources:
        requests:
          storage: 10Gi
```