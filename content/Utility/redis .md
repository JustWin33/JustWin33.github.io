---
title: redis
description: 
date: 2025-12-14
categories:
    - 
    - 
---
# redis 生产环境部署手册 

## 第一部分：部署前置检查与规划

### 1\. 操作系统选择

  * **推荐**：**AlmaLinux 8 / Rocky Linux 8** (RHEL 8 下游，稳定，内核较新) 或 **Ubuntu 22.04 LTS**。
  * **避免**：CentOS 7 (GCC 版本过低，编译 Redis 6/7 需繁琐升级，且官方已停止维护)。
  * **本手册环境**：以 **AlmaLinux 8** 为例。

### 2\. 架构规划决策树

在部署前，请根据数据量和业务场景选择架构：

| 场景特征                             | 推荐架构                                                     | 核心优势                                              |
| :----------------------------------- | :----------------------------------------------------------- | :---------------------------------------------------- |
| **数据量 \< 10GB** 且 **读多写少**   | **主从复制 + Sentinel (哨兵)**                               | 结构简单，维护成本低，具备自动故障转移能力。          |
| **数据量 \> 50GB** 或 **写请求极高** | **Redis Cluster (集群)**                                     | 数据分片存储 (16384 槽)，水平扩展能力强，无中心架构。 |
| **数据量巨大且只需简单缓存**         | **Codis / Twemproxy** (即使现在 Cluster 已成熟，部分旧系统仍在使用) | (本手册不涉及，建议优先使用原生 Cluster)              |

-----

## 第二部分：AlmaLinux 8 一键安装脚本

此脚本会自动安装依赖、下载 Redis 7.2.4 (目前稳定版)、编译安装、配置 Systemd 服务和基础配置文件。

**使用方法**：将以下内容保存为 `install_redis_el8.sh`，赋予执行权限 `chmod +x install_redis_el8.sh` 并以 root 运行。

```bash
#!/bin/bash
# 
# Redis 7.x One-Click Install Script for AlmaLinux 8 / Rocky Linux 8
#

REDIS_VERSION="7.2.4"
INSTALL_DIR="/usr/local/redis"
DATA_DIR="/data/redis"
CONFIG_DIR="/etc/redis"
LOG_DIR="/var/log/redis"

# 1. 系统检查与依赖安装
echo "[1/6] Installing dependencies..."
dnf install -y epel-release
dnf groupinstall -y "Development Tools"
dnf install -y openssl-devel systemd-devel wget tar

# 2. 下载与解压
echo "[2/6] Downloading Redis ${REDIS_VERSION}..."
cd /tmp
if [ ! -f redis-${REDIS_VERSION}.tar.gz ]; then
    wget https://download.redis.io/releases/redis-${REDIS_VERSION}.tar.gz
fi
tar xzf redis-${REDIS_VERSION}.tar.gz
cd redis-${REDIS_VERSION}

# 3. 编译安装
echo "[3/6] Compiling Redis..."
# 使用 systemd 支持编译
make -j $(nproc) USE_SYSTEMD=yes
make PREFIX=${INSTALL_DIR} install

# 4. 环境配置
echo "[4/6] Configuring environment..."
# 创建用户
id redis &> /dev/null
if [ $? -ne 0 ]; then
    useradd -r -s /sbin/nologin -M redis
fi

# 创建目录
mkdir -p ${DATA_DIR} ${CONFIG_DIR} ${LOG_DIR}
chown -R redis:redis ${DATA_DIR} ${CONFIG_DIR} ${LOG_DIR}
chmod 750 ${DATA_DIR} ${CONFIG_DIR} ${LOG_DIR}

# 环境变量
echo "export PATH=\$PATH:${INSTALL_DIR}/bin" > /etc/profile.d/redis.sh
source /etc/profile.d/redis.sh

# 5. 生成配置文件
echo "[5/6] Generating redis.conf..."
cat > ${CONFIG_DIR}/redis.conf <<EOF
# 基础网络
bind 0.0.0.0
port 6379
protected-mode no

# 认证安全 (生产环境请修改此密码!)
requirepass "StrongPass_ChangeMe"
masterauth "StrongPass_ChangeMe"

# 守护进程与 Systemd
daemonize no
supervised systemd
pidfile /run/redis/redis.pid

# 日志
logfile "${LOG_DIR}/redis.log"
loglevel notice

# 持久化 (RDB + AOF)
dir "${DATA_DIR}"
dbfilename dump.rdb
# RDB 策略: 3600秒1次修改, 300秒100次, 60秒10000次
save 3600 1 300 100 60 10000

appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 内存限制 (建议设置为物理内存的 75%)
# maxmemory 4gb
maxmemory-policy allkeys-lru

# 慢查询
slowlog-log-slower-than 10000
slowlog-max-len 1000
EOF
chown redis:redis ${CONFIG_DIR}/redis.conf
chmod 640 ${CONFIG_DIR}/redis.conf

# 6. 配置 Systemd 服务
echo "[6/6] Configuring Systemd..."
mkdir -p /run/redis
chown redis:redis /run/redis

cat > /etc/systemd/system/redis.service <<EOF
[Unit]
Description=Redis In-Memory Data Store
Documentation=https://redis.io/
After=network.target

[Service]
Type=notify
User=redis
Group=redis
ExecStart=${INSTALL_DIR}/bin/redis-server ${CONFIG_DIR}/redis.conf
ExecStop=${INSTALL_DIR}/bin/redis-cli -a StrongPass_ChangeMe shutdown
Restart=always
# 需要配合 redis.conf 中的 supervised systemd
RuntimeDirectory=redis
RuntimeDirectoryMode=0755

[Install]
WantedBy=multi-user.target
EOF

# 重新加载并启动
systemctl daemon-reload
systemctl enable --now redis

echo "=========================================="
echo "Redis ${REDIS_VERSION} installation completed!"
echo "Service status:"
systemctl status redis --no-pager
echo "=========================================="
```

-----

## 第三部分：标准化配置详解

### 1\. Systemd 服务文件标准模版

不要直接后台运行脚本，请使用以下标准模版管理服务。该模版已集成在上方脚本中，以下为详细说明：

**文件位置**: `/etc/systemd/system/redis.service`

```ini
[Unit]
Description=Redis In-Memory Data Store
Documentation=https://redis.io/
After=network.target

[Service]
# 使用 notify 类型，Redis 启动就绪后会发送信号给 systemd
Type=notify
User=redis
Group=redis
# 启动命令
ExecStart=/usr/local/redis/bin/redis-server /etc/redis/redis.conf
# 停止命令 (注意：如果设置了密码，必须加 -a 参数)
ExecStop=/usr/local/redis/bin/redis-cli -a StrongPass_ChangeMe shutdown
# 总是重启，防止异常退出
Restart=always
# 自动创建 /run/redis 目录用于存放 pid 文件
RuntimeDirectory=redis
RuntimeDirectoryMode=0755
# 增加文件打开数限制 (高并发必须)
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

### 2\. 生产环境 `redis.conf` 关键参数清单

在生成的配置文件基础上，以下参数需根据实际机器配置进行微调：

| 参数               | 推荐值                | 说明                                                         |
| :----------------- | :-------------------- | :----------------------------------------------------------- |
| `daemonize`        | **no**                | 使用 Systemd 管理时必须设为 no，否则 systemd 会无法追踪进程。 |
| `supervised`       | **systemd**           | 配合 Systemd 的 `Type=notify`，实现更好的进程管理。          |
| `maxmemory`        | **物理内存的 70-80%** | 预留内存给系统和 Redis 的额外开销，防止 OOM。                |
| `maxmemory-policy` | **allkeys-lru**       | 缓存场景首选；如果是持久存储场景，可设为 `noeviction`。      |
| `appendfsync`      | **everysec**          | 兼顾性能与数据安全，最多丢失 1 秒数据。                      |
| `replicaof`        | (从节点配置)          | 替代旧版的 `slaveof`，格式: `replicaof <MasterIP> <Port>`。  |

-----

## 第四部分：高可用架构配置指南

### 方案 A：Sentinel (哨兵) 模式配置

适用于 **1 主 2 从 + 3 哨兵** 的场景。

1.  **数据节点配置 (redis.conf)**

      * **Master**: 无需特殊配置。
      * **Slave**: 添加 `replicaof <MasterIP> 6379` 和 `masterauth <MasterPassword>`。

2.  **哨兵节点配置 (sentinel.conf)**
    所有哨兵节点配置相同：

    ```conf
    port 26379
    daemonize no
    # 监控 mymaster，至少 2 个哨兵同意才判定客观下线
    sentinel monitor mymaster 192.168.1.100 6379 2
    sentinel auth-pass mymaster StrongPass_ChangeMe
    sentinel down-after-milliseconds mymaster 30000
    sentinel parallel-syncs mymaster 1
    sentinel failover-timeout mymaster 180000
    ```

### 方案 B：Cluster (集群) 模式配置

适用于 **3 主 3 从** (至少 6 个节点) 的场景。

1.  **节点配置 (redis.conf)**
    所有 6 个节点开启以下配置：

    ```conf
    cluster-enabled yes
    cluster-config-file nodes-6379.conf
    cluster-node-timeout 15000
    masterauth "StrongPass_ChangeMe"
    requirepass "StrongPass_ChangeMe"
    ```

2.  **创建集群**
    使用 `redis-cli` 一键创建 (Redis 5+):

    ```bash
    /usr/local/redis/bin/redis-cli -a StrongPass_ChangeMe --cluster create \
    192.168.1.11:6379 192.168.1.12:6379 192.168.1.13:6379 \
    192.168.1.14:6379 192.168.1.15:6379 192.168.1.16:6379 \
    --cluster-replicas 1
    ```

-----

## 第五部分：常用维护命令

1.  **查看服务状态**

    ```bash
    systemctl status redis
    ```

2.  **查看 Redis 内部状态**

    ```bash
    # 交互式进入
    redis-cli -a StrongPass_ChangeMe
    > INFO Server       # 查看版本和运行参数
    > INFO Replication  # 查看主从状态
    > INFO Persistence  # 查看 RDB/AOF 状态
    > INFO Memory       # 查看内存使用
    ```

3.  **实时监控日志**

    ```bash
    tail -f /var/log/redis/redis.log
    ```

