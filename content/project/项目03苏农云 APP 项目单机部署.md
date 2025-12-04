---
title: 苏农云项目部署
description: 
date: 2025-12-04
categories:
    - 
    - 
---



# 苏农云 APP 项目单机部署手册 

**环境信息：**

  * **操作系统：** Kylin Server 10 SP2 (x86)
  * **服务器 IP：** `192.168.200.111` (已根据您的情况写死，如IP变更请替换)
  * **操作用户：** root
  * **验证状态：** 已通过验证

-----



```bash
ip a
注意看下面是ens33还是ens160
cat <<EOF > /etc/sysconfig/network-scripts/ifcfg-ens160
TYPE=Ethernet
BOOTPROTO=static
NAME=ens160
DEVICE=ens160
ONBOOT=yes
IPADDR=192.168.200.111
NETMASK=255.255.255.0
GATEWAY=192.168.200.2
DNS1=8.8.8.8
DNS2=114.114.114.114
EOF

nmcli connection reload
nmcli connection up ens160
ip addr show ens160  # 应该能看到192.168.200.111
ping 8.8.8.8         # 测试网络连通性
```



```bash
安装docker


# 1. 编辑Docker repo文件
vim /etc/yum.repos.d/docker-ce.repo

# 2. 将所有$releasever替换为8（共3处）
# 将 baseurl=https://download.docker.com/linux/centos/$releasever/...
# 改为 baseurl=https://download.docker.com/linux/centos/8/...

# 或使用sed一键修改
sed -i 's/$releasever/8/g' /etc/yum.repos.d/docker-ce.repo

# 3. 清理缓存并安装
yum clean all && yum makecache
yum -y install docker-ce docker-ce-cli containerd.io

docker --version

systemctl start docker
systemctl enable docker  # 设置开机自启

运行测试
docker run hello-world
验证docker-compose
docker compose version

```





### 第零步：彻底清理环境 (防止残留干扰)

为了确保万无一失，我们先清空之前所有可能出错的容器和文件。

```bash
# 1. 停止并强制删除所有相关容器
docker rm -f percona8 redis rabbitmq memcached project_a project_m project_api 2>/dev/null

# 2. 清理旧的数据目录 (这一步会清空数据库，确保权限重置)
rm -rf /data/percona
rm -rf /data/redis
rm -rf /data/rabbitmq
rm -rf /data/memcached
rm -rf /data/project
rm -rf /root/docker_build

# 3. 重新创建干净的目录结构
mkdir -pv /data/software
mkdir -pv /data/percona/{data,log,conf}
mkdir -pv /data/redis/{data,conf}
mkdir -pv /data/rabbitmq/{data,home}
mkdir -pv /data/memcached
mkdir -pv /data/project/{a,m,api}
mkdir -pv /root/docker_build/software
```

-----

### 第一步：系统底层初始化 (解决核心报错)

**解决历史问题：**

  * `OS errno 13 - Permission denied`: 通过关闭 SELinux 解决。
  * `unable to allocate file descriptor`: 通过 ulimit 解决（在后续启动命令中）。

<!-- end list -->

```bash
# 1. 核心：关闭 SELinux (必须执行，否则数据库必挂)
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config

# 2. 关闭防火墙 (确保外部能访问网站)
systemctl stop firewalld
systemctl disable firewalld

# 3. 配置 Host (解决应用内部域名解析)
hostnamectl set-hostname sny-app-dev
# 如果 hosts 里已经有旧IP，先删掉再加，防止冲突
sed -i '/sny-app-dev/d' /etc/hosts
echo "192.168.200.111 sny-app-dev" >> /etc/hosts
```

-----

### 第二步：资源解压 (解决参数错误)

**解决历史问题：**

  * `unzip -0` 报错: 修正为 `-O` (大写字母)。

<!-- end list -->

```bash
cd /data/software
# 确保 zip 包已上传。使用 -O (大写字母O) 指定 gbk 编码
# 如果提示是否替换(replace)，输入 A 回车
unzip -O gbk source_package_230221_v1.zip
```

-----

### 第三步：部署 MySQL 数据库 (Percona-8)

**解决历史问题：**

  * 无限重启: 修正配置文件权限为 644。
  * `cp` 找不到文件: 修正了 `.sql` 文件源路径。

**3.1 生成配置文件与权限修正**

```bash
# 写入配置
cat > /data/percona/conf/my.cnf <<EOF
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
character_set_server=gbk
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
innodb_log_file_size=1024M
innodb_strict_mode=0
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
!includedir /etc/my.cnf.d
EOF

# 【关键】修正权限：配置文件必须是 644，绝不能是 777
chmod 644 /data/percona/conf/my.cnf

# 【关键】修正数据目录归属 (Percona 容器内用户 ID 为 1001)
touch /data/percona/log/mysqld.log
chmod 666 /data/percona/log/mysqld.log
chown -R 1001:1001 /data/percona/data
chown -R 1001:1001 /data/percona/log
```

**3.2 启动数据库**

```bash
docker run -d --name percona8 \
  --privileged=true \
  --restart=always \
  -p 3306:3306 \
  -e MYSQL_ROOT_PASSWORD="znxd@230220" \
  -v /etc/localtime:/etc/localtime:ro \
  -v /data/percona/data:/var/lib/mysql:rw \
  -v /data/percona/conf/my.cnf:/etc/my.cnf:rw \
  -v /data/percona/log/mysqld.log:/var/log/mysqld.log:rw \
  percona:8

echo "等待数据库启动 (15秒)..."
sleep 15
```

**3.3 导入数据**

```bash
# 1. 拷贝 SQL 文件 (注意路径包含 /source/)
cp /data/software/source/*.sql /data/percona/data/
chmod 644 /data/percona/data/*.sql

# 2. 执行导入 (自动执行，无需交互)
docker exec -i percona8 mysql -uroot -p'znxd@230220' <<EOF
create database if not exists cmscloud character set gbk;
create database if not exists erpcloud character set gbk;
create database if not exists appcloud character set gbk;

create user if not exists cmscloud@'%' identified by 'Cmscloud!230221';
grant all privileges on *.* to cmscloud@'%' with grant option;
flush privileges;

use cmscloud;
source /var/lib/mysql/cmscloud.sql;
use appcloud;
source /var/lib/mysql/appcloud.sql;
use erpcloud;
source /var/lib/mysql/erpcloud.sql;

alter user 'cmscloud'@'%' identified with mysql_native_password by 'Cmscloud!230221';
flush privileges;
EOF
```

-----

### 第四步：部署 RabbitMQ

**解决历史问题：**

  * 崩溃/Exit 15: 修正数据目录权限为 999:999 (严禁 777)。

<!-- end list -->

```bash
# 1. 修正权限 (容器内用户 ID 为 999)
chown -R 999:999 /data/rabbitmq

# 2. 启动容器
docker run -d --name rabbitmq \
  --privileged=true \
  --restart=always \
  -p 5672:5672 -p 15672:15672 \
  -v /data/rabbitmq/data:/var/lib/rabbitmq \
  -v /data/rabbitmq/home:/var/lib/rabbitmq/mnesia \
  rabbitmq:3.8.14

# 3. 等待启动 (必须等待，否则无法配置用户)
echo "等待 RabbitMQ 启动 (15秒)..."
sleep 15

# 4. 配置用户
docker exec -it rabbitmq bash -c "rabbitmqctl add_user admin 7548777 && rabbitmqctl set_user_tags admin administrator && rabbitmqctl set_permissions -p / admin '.*' '.*' '.*'"
```

-----

### 第五步：部署 Middleware (Redis & Memcached)

**解决历史问题：**

  * Memcached 版本报错: 使用 `latest` 替代不存在的 `3.1.3`。

<!-- end list -->

```bash
# Redis
cat > /data/redis/conf/redis.conf <<EOF
bind 0.0.0.0
protected-mode no
port 6379
requirepass znxd@230220
timeout 0
daemonize no
appendonly yes
EOF

docker run -d --name redis \
  --privileged=true \
  --restart=always \
  -p 6379:6379 \
  -v /data/redis/data:/data:rw \
  -v /data/redis/conf:/etc/redis/:rw \
  redis:6.0.5 \
  redis-server /etc/redis/redis.conf

# Memcached
docker run -d --name memcached --privileged=true --restart=always -p 11211:11211 memcached
```

-----

### 第六步：构建 Tomcat 镜像

**解决历史问题：**

  * Docker build 找不到文件: 创建专用构建目录。

<!-- end list -->

```bash
mkdir -p /root/docker_build/software
# 1. 准备文件
cp /data/software/apache-tomcat-9.0.71.tar.gz /root/docker_build/software/
cp /data/software/jdk-8u11-linux-x64.tar.gz /root/docker_build/software/

检查文件是否到位
ls -lh /root/docker_build/software
# 2. 生成 Dockerfile
cd /root/docker_build
cat > Dockerfile <<EOF
FROM debian
MAINTAINER mslinux<mslinux@163.com>
ADD software/jdk-8u11-linux-x64.tar.gz /data/
ADD software/apache-tomcat-9.0.71.tar.gz /data/
WORKDIR /data
ENV JAVA_HOME /data/jdk1.8.0_11
ENV PATH \$PATH:\$JAVA_HOME/bin
EXPOSE 8080
CMD /data/apache-tomcat-9.0.71/bin/startup.sh && tail -f /data/apache-tomcat-9.0.71/logs/catalina.out
EOF

# 3. 构建
docker build -t znxd_tomcat9_java8:1.0 .
```

-----

### 第七步：应用部署 (最关键的一步)

**解决历史问题：**

  * `cp` 报错: 预先创建了 `{a,m,api}` 目录。

  * **打不开网站**: 配置文件 IP 错误 (`172...` -\> `192...`)。

  * **Tomcat 崩溃**: 添加了 `ulimit` 和 `-Xmx512m` 内存限制。

    ```bash
    mkdir -pv /data/project/{a,m,api}
    ```

    

**7.1 拷贝代码**

```bash
cp -ar /data/software/source/a.hzd360.com/ROOT/* /data/project/a/
cp -ar /data/software/source/m.hzd360.com/ROOT/* /data/project/m/
cp -ar /data/software/source/api.hzd360.com/ROOT/* /data/project/api/


以输入 ls /data/project/a/，如果看到里面有很多文件（不是空的），说明拷贝成功了。
```

**7.2 一键修正所有配置文件 (使用您的真实IP)**

```bash
# A 项目
cd /data/project/a/WEB-INF/classes/
sed -i 's/192.168.3.77/192.168.200.111/g' config.xml
sed -i 's/172.18.8.138/192.168.200.111/g' config.xml
sed -i 's/cmscloud1111/Cmscloud!230221/g' config.xml
sed -i 's/host=.*/host=192.168.200.111/g' redis.properties redis_info.properties
sed -i 's/pass=.*/pass=znxd@230220/g' redis.properties redis_info.properties

# M 项目
cd /data/project/m/WEB-INF/classes/
sed -i 's/192.168.3.77/192.168.200.111/g' config.xml
sed -i 's/172.18.8.138/192.168.200.111/g' config.xml
sed -i 's/cmscloud1111/Cmscloud!230221/g' config.xml
sed -i 's/192.168.1.201/192.168.200.111/g' config.xml
sed -i 's/host=.*/host=192.168.200.111/g' redis.properties redis_info.properties
sed -i 's/pass=.*/pass=znxd@230220/g' redis.properties redis_info.properties

# API 项目
cd /data/project/api/WEB-INF/classes/
sed -i 's/192.168.3.77/192.168.200.111/g' config.xml
sed -i 's/172.18.8.138/192.168.200.111/g' config.xml
sed -i 's/192.168.1.148/192.168.200.111/g' config.xml
sed -i 's/cmscloud1111/Cmscloud!230221/g' config.xml
sed -i 's/<uid>erpcloud<\/uid>/<uid>cmscloud<\/uid>/g' config.xml
sed -i 's/erpcloud1111/Cmscloud!230221/g' config.xml
sed -i 's/host=.*/host=192.168.200.111/g' redis.properties redis_info.properties
sed -i 's/pass=.*/pass=znxd@230220/g' redis.properties redis_info.properties
```

**7.3 启动应用 (带内存保护和句柄限制)**

```bash
# 启动 Project A (后台)
docker run -d --name project_a \
  --privileged=true \
  --restart=always \
  -p 10001:8080 \
  --ulimit nofile=65535:65535 \
  -e JAVA_OPTS="-Xms256m -Xmx512m" \
  -v /data/project/a:/data/apache-tomcat-9.0.71/webapps/ROOT:rw \
  znxd_tomcat9_java8:1.0

# 启动 Project M (移动端)
docker run -d --name project_m \
  --privileged=true \
  --restart=always \
  -p 10002:8080 \
  --ulimit nofile=65535:65535 \
  -e JAVA_OPTS="-Xms256m -Xmx512m" \
  -v /data/project/m:/data/apache-tomcat-9.0.71/webapps/ROOT:rw \
  znxd_tomcat9_java8:1.0

# 启动 Project API (接口)
docker run -d --name project_api \
  --privileged=true \
  --restart=always \
  -p 10003:8080 \
  --ulimit nofile=65535:65535 \
  -e JAVA_OPTS="-Xms256m -Xmx512m" \
  -v /data/project/api:/data/apache-tomcat-9.0.71/webapps/ROOT:rw \
  znxd_tomcat9_java8:1.0
```

-----

### 第八步：最终访问

**等待 2-3 分钟** 让所有 Java 服务完成启动，然后访问：

  * **管理后台:** `http://192.168.200.111:10001` (验证通过)
  * **移动端:** `http://192.168.200.111:10002/app/?userid=13592052010`
  * **API:** `http://192.168.200.111:10003/api/myApp.jsp`