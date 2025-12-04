---
title: centos部署MHA
description: 
date: 2025-12-04
categories:
    - 
    - 
---
# CentOS 7 + MySQL 5.7 + MHA 高可用集群部署 

## 1. 环境规划

**前提条件**：所有机器操作系统均为 CentOS 7.6/7.9，网络互通。

| **主机名**      | **IP地址**      | **角色** | **Server ID** | **备注**     |
| --------------- | --------------- | -------- | ------------- | ------------ |
| **mha-master**  | 192.168.200.101 | Master   | 1             | 核心主库     |
| **mha-slave1**  | 192.168.200.102 | Slave    | 2             | 备用从库     |
| **mha-slave2**  | 192.168.200.103 | Slave    | 3             | 备用从库     |
| **mha-manager** | 192.168.200.104 | Manager  | -             | 监控管理节点 |
| **VIP**         | 192.168.200.200 | -        | -             | 业务连接IP   |

------

## 2. 基础环境初始化 (所有节点 101-104)

**注意：本节操作需在 4 台机器上全部执行。**

### 2.1 修复 Yum 源 (解决下载失败问题)

由于你的环境无法访问阿里云内网源，必须先切换为公网源，否则依赖包下载会报错。

Bash

```
# 1. 备份旧源
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
rm -rf /etc/yum.repos.d/epel*

# 2. 下载阿里云公网源
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo

# 3. 清理并生成缓存
yum clean all && yum makecache
```

### 2.2 系统配置 (防火墙/主机名/Hosts)

Bash

```
# 关闭防火墙与 SELinux
systemctl stop firewalld && systemctl disable firewalld
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 配置 Hosts 解析
cat >> /etc/hosts << EOF
192.168.200.101 mha-master
192.168.200.102 mha-slave1
192.168.200.103 mha-slave2
192.168.200.104 mha-manager
EOF
```

### 2.3 配置 SSH 免密互信 (避坑指南)

**重点**：MHA Manager 必须能免密登录所有节点。建议做全网互信。

1. **生成密钥**（在每台机器执行）：

   Bash

   ```
   ssh-keygen -t rsa
   # 【注意】出现提示时，不要输入任何字符，直接连续按 3 次回车！
   ```

2. **分发密钥**（建议在每台机器上都执行一遍以下 4 条命令）：

   Bash

   ```
   ssh-copy-id mha-master
   ssh-copy-id mha-slave1
   ssh-copy-id mha-slave2
   ssh-copy-id mha-manager
   # 输入 yes 并输入 root 密码
   ```

------

## 3. MySQL 集群部署 (节点 101, 102, 103)

### 3.1 准备安装包

在 **101, 102, 103** 机器上创建目录 `/opt/install`，上传以下两个文件：

1. `mysql_5.7.44.tar.bz2` (文件名必须精确匹配)

2. `mha4mysql-node-0.58-0.el7.centos.noarch.rpm`

   ```bash
   
   mkdir -p /opt/install 
   cd /opt/install
   将 mysql_5.7.44.tar.bz2 和 mha4mysql-node-0.58-0.el7.centos.noarch.rpm (如果有) 上传到该目录
   vim deploy_mysql_mha.sh
   chmod +x deploy_mysql_mha.sh
   ./deploy_mysql_mha.sh
   
   
   ```

   

### 3.2 创建自动化脚本

在 `/opt/install` 下创建脚本 `vim deploy_mysql.sh`，复制以下**V5 终极版代码**：

3.3 执行脚本

```bash
#!/bin/bash

# ==========================================
# 脚本名称: MySQL 5.7.44 + MHA 部署脚本 
# ==========================================

# --- 步骤 0: 交互式配置 ---
clear
echo "========================================================"
echo "   MySQL 5.7.44 + MHA 一键部署脚本 (V5 终极清洗版)"
echo "========================================================"
echo "请根据规划输入当前机器的 Server ID:"
echo "  - Master (101) 请输入: 1"
echo "  - Slave1 (102) 请输入: 2"
echo "  - Slave2 (103) 请输入: 3"
echo "--------------------------------------------------------"
read -p "请输入 Server ID [1-3]: " SERVER_ID

if [[ ! "$SERVER_ID" =~ ^[1-9]+$ ]]; then
    echo "❌ 错误: Server ID 必须是数字！"
    exit 1
fi

MYSQL_PKG="mysql_5.7.44.tar.bz2"
MHA_NODE_PKG="mha4mysql-node-0.58-0.el7.centos.noarch.rpm"

# ==========================================
# 步骤 1: 预先安装基础依赖
# ==========================================
echo "[1/7] 安装基础依赖..."
yum install -y perl perl-devel libaio net-tools autoconf

# ==========================================
# 步骤 2: 彻底清理旧环境 (新增 /data/mysql 清理)
# ==========================================
echo "[2/7] 正在暴力清理旧环境..."
systemctl stop mysqld >/dev/null 2>&1
pkill -9 mysqld >/dev/null 2>&1
rpm -e --nodeps mariadb-libs >/dev/null 2>&1
yum remove -y mariadb-libs mysql-libs >/dev/null 2>&1
yum remove -y mysql-community-* mysql* MySQL* >/dev/null 2>&1

# 【V5 关键修改】强制删除旧数据目录，解决 "directory has files" 报错
rm -rf /var/lib/mysql /etc/my.cnf* /var/log/mysqld.log
rm -rf /data/mysql  

# ==========================================
# 步骤 3: 安装 MySQL (智能搜寻修复)
# ==========================================
echo "[3/7] 开始安装 MySQL 5.7.44..."

if [ ! -f "$MYSQL_PKG" ]; then
    echo "❌ 错误: 当前目录下未找到 $MYSQL_PKG"
    exit 1
fi

INSTALL_TEMP_DIR="mysql_install_temp"
rm -rf $INSTALL_TEMP_DIR 
mkdir -p $INSTALL_TEMP_DIR

echo "    正在解压..."
tar xf $MYSQL_PKG -C $INSTALL_TEMP_DIR

echo "    正在搜寻 RPM 包..."
RPM_LIST=$(find $INSTALL_TEMP_DIR -name "*.rpm")

if [ -z "$RPM_LIST" ]; then
    echo "❌ 严重错误: 解压后未找到任何 .rpm 文件！"
    ls -R $INSTALL_TEMP_DIR
    exit 1
fi

echo "    找到 RPM 包，正在安装..."
rpm -Uvh $RPM_LIST --force --nodeps

if [ ! -f "/usr/sbin/mysqld" ]; then
    echo "❌ 严重错误: MySQL 安装失败，未找到 /usr/sbin/mysqld 文件！"
    rm -rf $INSTALL_TEMP_DIR
    exit 1
fi

rm -rf $INSTALL_TEMP_DIR

# ==========================================
# 步骤 4: 安装 MHA Node 依赖
# ==========================================
echo "[4/7] 安装 MHA Node 依赖..."
yum install -y perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager
yum install -y perl-DBD-MySQL --skip-broken

if [ -f "$MHA_NODE_PKG" ]; then
    rpm -ivh $MHA_NODE_PKG --force --nodeps
fi

# ==========================================
# 步骤 5: 写入配置文件
# ==========================================
echo "[5/7] 写入配置文件 /etc/my.cnf..."
cat > /etc/my.cnf << EOF
[mysql]
port=3306
bind-address=0.0.0.0

[mysqld]
character_set_server=utf8
init_connect='SET NAMES utf8'
user=mysql
socket=/var/lib/mysql/mysql.sock
datadir=/data/mysql/data
max_connections=2000
skip-name-resolve

# --- MHA 核心配置 ---
server-id=${SERVER_ID}
log-bin=/data/mysql/data/mysql-bin
relay-log=/data/mysql/data/relay-log
binlog_format=ROW
expire_logs_days=7
max_binlog_size=500M
binlog_cache_size=128K
log_slave_updates=1
slave-skip-errors=1
relay_log_recovery=1
master_info_repository=TABLE
relay_log_info_repository=TABLE
gtid_mode=on
enforce_gtid_consistency=1

# --- 优化 ---
max_allowed_packet=100M
lower_case_table_names=1
slow_query_log=on
slow_query_log_file=/data/mysql/logs/slow_query_log.log
long_query_time=2
symbolic-links=0
innodb_buffer_pool_size=1G
innodb_open_files=800
innodb_file_per_table=1
innodb_flush_log_at_trx_commit=2
sync_binlog=1

[mysqld_safe]
pid-file=/var/run/mysqld/mysqld.pid
log-error=/data/mysql/logs/mysql.err
EOF

# ==========================================
# 步骤 6: 目录与初始化
# ==========================================
echo "[6/7] 配置目录与初始化..."
mkdir -pv /data/mysql/{data,logs}
mkdir -pv /var/lib/mysql
chown -R mysql:mysql /data/mysql/
chown -R mysql:mysql /var/lib/mysql/
chmod 750 /data/mysql/data

# 强制将日志输出到文件，确保能抓取到密码
/usr/sbin/mysqld --initialize --user=mysql --datadir=/data/mysql/data --log-error=/data/mysql/logs/mysql.err

if [ $? -ne 0 ]; then
    echo "❌ 错误: 数据库初始化失败！"
    tail -n 10 /data/mysql/logs/mysql.err
    exit 1
fi

# ==========================================
# 步骤 7: 启动服务
# ==========================================
echo "[7/7] 启动服务..."
systemctl start mysqld
systemctl enable mysqld

if systemctl is-active --quiet mysqld; then
    echo "✅ MySQL 部署成功！Server ID: $SERVER_ID"
    TEMP_PWD=$(grep 'temporary password' /data/mysql/logs/mysql.err | awk '{print $NF}')
    echo "----------------------------------------------------"
    echo "🔑 Root 初始密码: $TEMP_PWD"
    echo "----------------------------------------------------"
else
    echo "❌ MySQL 启动失败。"
    tail -n 20 /data/mysql/logs/mysql.err
fi

```



分别在三台机器运行 `sh deploy_mysql.sh`，并根据提示输入 ID (1, 2, 3)。**记录下最后生成的密码**。

------

## 4. 配置 GTID 主从复制

### 4.1 Master 操作 (101)

登录数据库：

mysql -uroot -p'脚本生成的密码'

执行 SQL：

SQL

```
-- 1. 修改密码
ALTER USER 'root'@'localhost' IDENTIFIED BY '123123';
FLUSH PRIVILEGES;

-- 2. 创建用户 (复制用户 + MHA管理用户)
GRANT REPLICATION SLAVE ON *.* TO repl@'192.168.200.%' IDENTIFIED BY 'repl_user';
GRANT ALL ON *.* TO mha@'192.168.200.%' IDENTIFIED BY 'mha_user';
FLUSH PRIVILEGES;
```

### 4.2 Slave 操作 (102 和 103)

在两台从库分别执行：

mysql -uroot -p'脚本生成的密码'

执行 SQL：

SQL

```
-- 1. 修改密码
ALTER USER 'root'@'localhost' IDENTIFIED BY '123123';

-- 2. 配置 GTID 复制 (无需指定文件名和位置)
STOP SLAVE;
RESET SLAVE;
CHANGE MASTER TO 
  MASTER_HOST='192.168.200.101', 
  MASTER_USER='repl', 
  MASTER_PASSWORD='repl_user', 
  MASTER_AUTO_POSITION=1;
START SLAVE;

-- 3. 验证状态
SHOW SLAVE STATUS\G;
```

**验证标准**：确保 `Slave_IO_Running: Yes` 且 `Slave_SQL_Running: Yes`。

------

## 5. MHA Manager 部署 (节点 104)

### 5.1 安装软件

在 **mha-manager (104)** 上上传 RPM 包并安装：

Bash

```
# 1. 安装 Perl 依赖 (已修复 Yum 源，这里应很顺利)
yum install -y perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager

# 2. 安装 MHA 组件
rpm -ivh mha4mysql-node-0.58-0.el7.centos.noarch.rpm
rpm -ivh mha4mysql-manager-0.58-0.el7.centos.noarch.rpm
```

### 5.2 配置文件

创建目录并编辑文件：

Bash

```
mkdir -p /data/mha/{conf,logs,tools,tmp}
vim /data/mha/conf/mha.conf
```

**内容如下**：

Ini, TOML

```
[server default]
manager_workdir=/data/mha/tmp
manager_log=/data/mha/logs/manager.log
remote_workdir=/data/mha/tmp
ssh_user=root
# MHA 监控用户
user=mha
password=mha_user
# 复制用户
repl_user=repl
repl_password=repl_user
# 切换脚本 (注意文件名是 script 不是 scrips)
master_ip_failover_script=/data/mha/tools/master_ip_failover_script
master_binlog_dir=/data/mysql/data
ping_interval=1

[server1]
hostname=192.168.200.101
candidate_master=1

[server2]
hostname=192.168.200.102
candidate_master=1

[server3]
hostname=192.168.200.103
candidate_master=1
```

### 5.3 故障切换脚本

创建脚本 vim /data/mha/tools/master_ip_failover_script：

(已修复 Perl 语法错误)

Perl

```
#!/usr/bin/env perl
use strict;
use warnings FATAL => 'all';
use Getopt::Long;

my ($command, $ssh_user, $orig_master_host, $orig_master_ip, $orig_master_port, $new_master_host, $new_master_ip, $new_master_port);

# === VIP配置 ===
my $vip = '192.168.200.200/24';
my $key = '1';
my $iface = 'ens33'; # 确认你的网卡名
# ==============

my $ssh_start_vip = "/sbin/ifconfig $iface:$key $vip";
my $ssh_stop_vip  = "/sbin/ifconfig $iface:$key down";

GetOptions(
    'command=s'          => \$command,
    'ssh_user=s'         => \$ssh_user,
    'orig_master_host=s' => \$orig_master_host,
    'orig_master_ip=s'   => \$orig_master_ip,
    'orig_master_port=i' => \$orig_master_port,
    'new_master_host=s'  => \$new_master_host,
    'new_master_ip=s'    => \$new_master_ip,
    'new_master_port=i'  => \$new_master_port,
);

exit &main();

sub main {
    if ($command eq "stop" || $command eq "stopssh") {
        eval { &stop_vip(); };
        if ($@) { warn "Got Error: $@\n"; exit 1; }
        exit 0;
    }
    elsif ($command eq "start") {
        eval { &start_vip(); };
        if ($@) { warn "Got Error: $@\n"; exit 1; }
        exit 0;
    }
    elsif ($command eq "status") {
        print "Checking script.. OK \n";
        exit 0;
    }
    exit 0;
}
sub start_vip { system("ssh $ssh_user\@$new_master_host \" $ssh_start_vip \""); }
sub stop_vip  { system("ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \""); }
```

**赋予权限 (重要)**：

Bash

```
chmod +x /data/mha/tools/master_ip_failover_script
```

------

## 6. 启动与验证

### 6.1 绑定 VIP

在 **101 (Master)** 上手动绑定 VIP：

Bash

```
ifconfig ens33:1 192.168.200.200/24 up
```

### 6.2 最终检查 (在 104 上执行)

Bash

```
# 1. 检查 SSH
masterha_check_ssh --conf=/data/mha/conf/mha.conf
# 必须显示: All SSH connection tests passed successfully.

# 2. 检查复制
masterha_check_repl --conf=/data/mha/conf/mha.conf
# 必须显示: MySQL Replication Health is OK.
```

### 6.3 启动 MHA

Bash

```
nohup masterha_manager --conf=/data/mha/conf/mha.conf --remove_dead_master_conf --ignore_last_failover < /dev/null > /data/mha/logs/manager.log 2>&1 &
```

### 6.4 查看状态

Bash

```
masterha_check_status --conf=/data/mha/conf/mha.conf
```

**看到 `PING_OK`，即代表部署圆满成功！**