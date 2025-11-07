---
title: MySQL
description: 不同安装方式
date: 2025-11-07
categories:
    - 
    - 
---

## 官方yum源安装

### 1. 安装 MySQL

CentOS 7 默认的 yum 源中没有 MySQL，需要先添加官方 yum 源

```bash
# 下载MySQL官方yum源
wget https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm

# 安装yum源
sudo rpm -ivh mysql80-community-release-el7-3.noarch.rpm

# 安装MySQL服务器
sudo yum install -y mysql-community-server
```

### 使用最新的 MySQL GPG 密钥并强制安装

#### 步骤 1：下载并导入最新的 MySQL GPG 密钥

```bash
# 下载最新的MySQL 8.0 GPG密钥
wget https://repo.mysql.com/RPM-GPG-KEY-mysql-2023

# 导入新密钥
sudo rpm --import RPM-GPG-KEY-mysql-2023
```

#### 步骤 2：手动指定密钥路径并安装（强制匹配）

如果步骤 1 仍失败，直接在安装命令中指定密钥文件路径：

```bash
# 强制使用新下载的密钥进行安装
sudo yum install -y mysql-community-server --nogpgcheck
```

> 注意：`--nogpgcheck` 会跳过 GPG 验证，仅作为临时解决方法，安装完成后建议重新配置密钥验证。

#### 步骤 3：（可选）手动编辑 MySQL 源配置文件

如果上述方法仍有问题，检查并修改 MySQL 的 yum 源配置：

```bash
# 编辑MySQL源配置
sudo vi /etc/yum.repos.d/mysql-community.repo
```

找到 `[mysql80-community]` 段落，确保 `gpgkey` 指向正确的密钥文件：

```ini
[mysql80-community]
name=MySQL 8.0 Community Server
baseurl=http://repo.mysql.com/yum/mysql-8.0-community/el/7/$basearch/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql-2023  # 修改为最新下载的密钥路径
```

保存后重新安装：

```bash
sudo yum clean all
sudo yum install -y mysql-community-server
```



### 2. 启动 MySQL 服务

```bash
# 启动MySQL服务
sudo systemctl start mysqld

# 设置开机自启动
sudo systemctl enable mysqld

# 查看服务状态
sudo systemctl status mysqld
```








## 本地RPM包安装MySQL5.7（CentOS/RHEL 7+ 适用）

### 前置说明
- 适用系统：CentOS 7/8、RHEL 7/8（兼容你的nginx-server环境）
- 安装方式：优先使用现有RPM包（避免编译麻烦，适配性更强）
- 核心流程：卸载残留→安装依赖→RPM包安装→解决端口冲突→初始化配置→开机自启

---

### 第一步：彻底卸载残留（含MySQL/MariaDB）
```bash
# 1. 查询已安装的MySQL/MariaDB包（避免漏卸）
rpm -qa | grep -E 'mysql|mariadb'

# 2. 卸载所有查询到的包（替换为上一步查询结果，或直接执行批量卸载）
yum remove -y $(rpm -qa | grep -E 'mysql|mariadb')

# 3. 清理残留文件/目录
rm -rf /var/lib/mysql  # 数据目录（彻底删除，无重要数据再执行）
rm -rf /etc/my.cnf     # 配置文件
rm -rf /var/log/mysqld.log  # 日志文件
rm -rf /var/run/mysqld  # PID目录
rm -rf /usr/local/mysql  # 若之前有二进制安装残留，一并删除
```

---

### 第二步：安装依赖包（避免RPM安装失败）
```bash
# 安装MySQL依赖的系统库（必装，解决大部分依赖缺失问题）
yum install -y libaio numactl-libs perl
```

---

### 第三步：安装MySQL RPM包（使用你现有包）
```bash
# 1. 进入RPM包所在目录（你的包在/root/1106，直接cd进入）
cd /root/1106

# 2. 批量安装所有MySQL RPM包（自动处理依赖关系）
yum localinstall mysql-community-*.rpm -y

# 3. 验证安装是否成功（显示mysql相关包即正常）
rpm -qa | grep mysql-community
```

---

### 第四步：解决端口冲突（3306端口被Docker占用处理）
```bash
# 1. 检查3306端口占用情况（确认是否为docker-proxy）
ss -tulpn | grep 3306

# 2. 方案A：停止占用3306的Docker容器（优先，保留3306端口）
# 查找映射3306的容器
docker ps | grep 3306
# 停止并删除容器（替换为实际容器ID/名称）
docker stop 容器ID/容器名
docker rm 容器ID/容器名

# 3. 方案B：修改MySQL端口（若Docker容器不能停，改用3307端口）
# 编辑MySQL配置文件
vi /etc/my.cnf
# 在[mysqld]段添加以下内容（替换端口为3307）
[mysqld]
port=3307
basedir=/usr
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
skip-name-resolve
log-error=/var/log/mysqld.log
```

---

### 第五步：MySQL启动
```bash
# 1. 修复数据目录权限（避免启动时权限不足）
chown -R mysql:mysql /var/lib/mysql
chown -R mysql:mysql /var/log/mysqld.log
chmod 755 /var/lib/mysql
```
```bash
报错用`##`的方案

## 1. 停止所有残留MySQL进程
systemctl stop mysqld
pkill -9 mysqld

## 2. 清空数据目录（重新生成系统表）
rm -rf /var/lib/mysql/*  # 清空现有数据（无重要数据）
chown -R mysql:mysql /var/lib/mysql
chmod 755 /var/lib/mysql

## 3. 重新初始化（生成mysql.plugin等系统表）
mysqld --initialize --user=mysql --datadir=/var/lib/mysql --basedir=/usr
```


```bash
# 2. 启动MySQL服务
systemctl start mysqld

# 3. 检查服务状态（显示active (running)即成功）
systemctl status mysqld -l

# 4. 设置开机自启（避免服务器重启后服务失效）
systemctl enable mysqld
```







## 源码编译安装
通过编译 MySQL 源码包进行安装，可自定义编译参数（如指定安装路径、启用 / 禁用功能模块等），适合对 MySQL 有深度定制需求的场景。
大致步骤：

```bash
# 1. 安装编译依赖

yum install -y gcc gcc-c++ cmake make ncurses-devel openssl-devel

# 2. 下载源码包（以MySQL 8.0为例）
wget https://dev.mysql.com/get/Downloads/MySQL-8.0/mysql-8.0.34.tar.gz
tar -zxvf mysql-8.0.34.tar.gz
cd mysql-8.0.34

# 3. 配置编译参数（示例：指定安装路径、启用SSL等）
cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql \
        -DMYSQL_DATADIR=/var/lib/mysql \
        -DENABLED_LOCAL_INFILE=ON \
        -DWITH_SSL=system

# 4. 编译并安装（耗时较长，根据服务器性能而定）
make && make install

# 5. 后续配置（创建用户、初始化数据库、配置服务等）
useradd -r -s /sbin/nologin mysql
chown -R mysql:mysql /usr/local/mysql /var/lib/mysql
/usr/local/mysql/bin/mysqld --initialize --user=mysql --datadir=/var/lib/mysql
```







##  二进制包安装（免编译）
MySQL 官方提供预编译的二进制分发版（.tar.gz格式），解压后即可使用，无需编译，比源码安装更快捷，适合需要快速部署且有一定自定义需求的场景。
```bash
# 1. 下载二进制包（以MySQL 5.7为例）
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.43-linux-glibc2.12-x86_64.tar.gz
tar -zxvf mysql-5.7.43-linux-glibc2.12-x86_64.tar.gz -C /usr/local/
ln -s /usr/local/mysql-5.7.43-linux-glibc2.12-x86_64 /usr/local/mysql

# 2. 初始化配置（类似源码安装）
useradd -r -s /sbin/nologin mysql
chown -R mysql:mysql /usr/local/mysql /var/lib/mysql
/usr/local/mysql/bin/mysqld --initialize --user=mysql --datadir=/var/lib/mysql

# 3. 配置环境变量和服务启动脚本
echo 'export PATH=$PATH:/usr/local/mysql/bin' >> /etc/profile
source /etc/profile
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
systemctl enable mysqld
```


## Docker 容器化安装
通过 Docker 运行 MySQL 容器，隔离性强、部署快速，适合开发、测试环境或需要快速扩缩容的场景，避免与主机环境冲突。

```bash
# 1. 拉取MySQL镜像（指定版本，如8.0或5.7）
docker pull mysql:8.0

# 2. 启动容器（映射端口、挂载数据卷持久化数据）
docker run -d \
  --name mysql8 \
  -p 3306:3306 \
  -v /data/mysql8:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=RootPass123! \
  mysql:8.0

# 3. 进入容器操作MySQL
docker exec -it mysql8 mysql -u root -p
```


























## 初始化配置MySQL（获取临时密码+修改密码）
```bash
# 1. 查找MySQL临时密码（首次安装自动生成）
grep 'temporary password' /var/log/mysqld.log

# 2. 用临时密码登录MySQL（若修改了端口，添加-P3307参数）
# 默认端口登录：
mysql -uroot -pXXXX                   `XXXX`是你临时密码
# 改端口后登录（3307为例）：
mysql -uroot -p -P3307

# 3. 登录后修改root密码（替换为你的新密码，需包含大小写+数字+特殊字符）强密码，才行
1. 先按默认策略设置一个临时强密码（完成初始重置）
执行以下命令（密码需满足：大小写 + 数字 + 特殊字符，长度≥8 位）：
ALTER USER 'root'@'localhost' IDENTIFIED BY 'TempPass123!';
2. 再次查看密码策略变量（此时已可执行）
SHOW VARIABLES LIKE 'validate_password%';

3. （可选）降低密码策略（如需简单密码）
若需要设置简单密码，继续执行：
-- 降低策略为LOW（仅检查长度≥8位）
set global validate_password_policy=LOW;
set global validate_password_length=8; 后面数字可以自己修改

-- 此时可设置简单密码（如8位纯数字）
ALTER USER 'root'@'localhost' IDENTIFIED BY '12345678';



# 4. （可选）允许root远程登录（方便外部连接，生产环境谨慎开启）
use mysql;
update user set host='%' where user='root';
flush privileges;

# 5. 退出MySQL
exit
```

---

### 验证MySQL可用
```bash
# 重新登录MySQL（验证密码是否生效）
mysql -uroot -p'123123'  # 默认端口
# 或改端口后登录：
mysql -uroot -p'123123' -P3307

# 执行简单查询（显示数据库列表即正常）
show databases;
```



### 5. MySQL 基本使用

```bash
# 登录MySQL
mysql -u root -p

# 创建数据库
CREATE DATABASE mydb;

# 查看所有数据库
SHOW DATABASES;

# 使用指定数据库
USE mydb;

# 创建表
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

# 插入数据
INSERT INTO users (name, email) VALUES ('John Doe', 'john@example.com');

# 查询数据
SELECT * FROM users;

# 更新数据
UPDATE users SET name = 'Jane Doe' WHERE id = 1;

# 删除数据
DELETE FROM users WHERE id = 1;

# 退出MySQL
exit;
```



### 6. 常用 MySQL 管理命令

```bash
# 重启MySQL服务
 systemctl restart mysqld

# 停止MySQL服务
 systemctl stop mysqld

# 查看MySQL版本
mysql --version

# 备份数据库
mysqldump -u root -p mydb > mydb_backup.sql

# 恢复数据库
mysql -u root -p mydb < mydb_backup.sql
```















## 常见导致启动失败的日志提示及解决方法：

#### 常见问题排查（遇到报错直接用）
1. 启动失败：查看错误日志 `cat /var/log/mysqld.log | grep -i error`
2. 密码忘记：停止服务→跳过授权表启动→重置密码（模板外补充，需可联系）
3. 远程连接失败：检查防火墙开放3306/3307端口 `firewall-cmd --permanent --add-port=3307/tcp && firewall-cmd --reload`




#### 1. 日志中出现 `Permission denied`（权限问题）

```bash
# 修复MySQL数据目录权限（关键目录必须归mysql用户所有）
sudo chown -R mysql:mysql /var/lib/mysql
sudo chmod -R 755 /var/lib/mysql

# 重启服务
sudo systemctl start mysqld
```

#### 2. 日志中出现 `Initialize database` 失败（初始化问题）

```bash
# 先停止服务（如果处于启动中）
sudo systemctl stop mysqld

# 清空数据目录（注意：会删除所有数据，首次安装适用）
sudo rm -rf /var/lib/mysql/*

# 重新初始化数据库
sudo mysqld --initialize --user=mysql --basedir=/usr --datadir=/var/lib/mysql

# 启动服务
sudo systemctl start mysqld
```

#### 3. 日志中出现 `Invalid configuration`（配置文件错误）

```bash
# 检查并编辑MySQL配置文件（通常是/etc/my.cnf）
sudo vi /etc/my.cnf

# 注释或删除错误的配置项（例如：无效的参数、路径错误等）
# 保存后重启服务
sudo systemctl start mysqld
```

#### 4. 日志中出现 `Port 3306 is already in use`（端口占用

```bash
# 查找占用3306端口的进程
 netstat -tulpn | grep 3306

# 终止占用进程（假设进程ID为1234）
 kill -9 1234

# 重启服务
 systemctl start mysqld
```

### 3. 初始化配置

```bash
# 获取初始密码
sudo grep 'temporary password' /var/log/mysqld.log

# 登录MySQL（使用上面获取的初始密码）
mysql -u root -p

# 修改密码（MySQL8.0以上要求密码包含大小写字母、数字和特殊字符）
ALTER USER 'root'@'localhost' IDENTIFIED BY 'NewPassword@123';

# 允许远程访问（可选）
use mysql;
update user set host = '%' where user = 'root';
FLUSH PRIVILEGES;
```

### 4. 防火墙配置（允许远程访问）

```bash
# 开放3306端口
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent

# 重新加载防火墙配置
sudo firewall-cmd --reload
```





### 6. 安全加固（推荐）

```bash
# 运行安全脚本，按照提示进行配置
sudo mysql_secure_installation
```

这个脚本会引导你完成一系列安全设置，包括：

- 设置 root 密码
- 移除匿名用户
- 禁止 root 远程登录
- 移除测试数据库
- 重新加载权限表