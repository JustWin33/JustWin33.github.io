---
title: 若依系统
description: 
date: 2025-12-04
categories:
    - 
    - 
---


🚀 若依系统 (Ruoyi) Kubernetes 容器化部署完整实战手册 



> **版本说明**：本手册基于 `CentOS 7` + `Kubernetes v1.20` + `Docker 20.10` 环境编写，修正了 NFS 权限、Docker Java 环境缺失、数据库防火墙拦截及 Service 端口映射错误等问题。



## 📋 0. 核心 IP 规划表 (基准配置)



| **角色**       | **主机名**   | **IP 地址**         | **关键说明**                        |
| -------------- | ------------ | ------------------- | ----------------------------------- |
| **K8s Master** | `k8s-master` | **192.168.200.100** | 控制节点，执行 `kubectl` 的地方     |
| **K8s Node1**  | `k8s-node1`  | **192.168.200.101** | Ingress 入口节点 (外部访问流量入口) |
| **K8s Node2**  | `k8s-node2`  | **192.168.200.102** | 工作节点                            |
| **Database**   | `db-server`  | **192.168.200.103** | MySQL 物理机 (非 K8s 内)            |
| **Harbor**     | `harbor`     | **192.168.200.104** | 私有镜像仓库                        |
| **NFS**        | `nfs-server` | **192.168.200.106** | 共享存储服务器                      |

------



## 第一阶段：基础环境互通 & NFS 客户端



**操作目的**：确保机器互认，且所有 K8s 节点都能挂载 NFS 盘（**修正点：** 之前 Node 节点缺少 `nfs-utils` 导致挂载失败）。

**在所有 6 台机器上执行：**

Bash

```
# 1. 配置 Hosts 解析
cat >> /etc/hosts << EOF
192.168.200.100 k8s-master
192.168.200.101 k8s-node1
192.168.200.102 k8s-node2
192.168.200.103 db-server
192.168.200.104 harbor
192.168.200.106 nfs-server
EOF

# 2. 🚨 必须安装：防止 Pod 启动报 RunContainerError
yum install -y nfs-utils
```

------



## 第二阶段：MySQL 数据库部署 (IP: 192.168.200.103)



**操作目的**：部署数据库并允许 K8s 远程连接（**修正点：** 之前被防火墙拦截，且缺少用户授权）。

1. **安装配置**

   Bash

   ```
   tar -xvf mysql-5.7.40.tar.gz
   yum -y localinstall mysql-5.7.40/*
   
   mkdir -pv /data/mysql/{data,logs}
   chown -R mysql:mysql /data/mysql/
   
   # 写入配置
   cat > /etc/my.cnf <<EOF
   
   
   systemctl enable --now mysqld
   ```

2. **安全设置与授权 (🚨 重点修正)**

   Bash

   ```
   # 1. 获取密码并登录
   grep password /data/mysql/logs/mysql.err
   mysql -uroot -p'临时密码'
   
   # 2. 修改 Root 密码
   set global validate_password_policy=0;
   set global validate_password_length=1;
   alter user 'root'@'localhost' identified by '123123';
   flush privileges;
   
   # 3. 创建业务库与远程用户 (修复 Access denied)
   create database ruoyi_demo character set utf8;
   # 创建 ruoyi_demo 用户，允许从任意地方 (%) 登录
   grant all privileges on ruoyi_demo.* to ruoyi_demo@'%' identified by '123123';
   flush privileges;
   
   # 4. 导入数据
   use ruoyi_demo;
   source /root/ry_20240601.sql;
   exit
   ```

3. **防火墙放行 (🚨 重点修正：修复 No route to host)**

   Bash

   ```
   # 简单粗暴方案：直接关闭防火墙
   systemctl stop firewalld && systemctl disable firewalld
   ```

------



## 第三阶段：Harbor 与 Docker 信任配置 (IP: 192.168.200.104)



**操作目的**：让 K8s 能够从私有仓库拉取镜像（**修正点：** 确保所有节点信任 HTTP 仓库）。

1. **部署 Harbor** (在 104 执行)

   ```bash
   cd rpm_package/
   
   安装docker-compose
   chmod +x docker-compose
   mv docker-compose /usr/local/bin
   yum localinstall -y 03_docker_package/* docker_need_rpm_*/*
   cd docker_harbor/
   tar xvf harbor-offline-installer-v2.30.tgz
   mv harbor /usr/local/
   cd /usr/local/harbor/
   mv harbor.yml.tmpl harbor.yml
   vim harbor.yml
   ```

   

   - 修改 `harbor.yml`：hostname 改为 `192.168.200.104`，注释掉 `https` 部分。
   - 执行 `./install.sh`。
   - Web 界面新建公开项目：`car_prod`。

2. **配置 Docker 信任** (在 **Master, Node1, Node2, Harbor** 上执行)

   Bash

   ```
   vim /etc/docker/daemon.json
   # 内容：
   {
     "insecure-registries": ["192.168.200.104"],
     "exec-opts": ["native.cgroupdriver=systemd"],
     "registry-mirrors": ["https://docker.mirrors.ustc.edu.cn"],
     "storage-driver": "overlay2"
   }
   
   # 重启并验证登录
   systemctl daemon-reload && systemctl restart docker
   docker login -u admin -p Harbor12345 http://192.168.200.104
   ```

------



## 第四阶段：构建镜像 (在 Harbor 104 上操作)



**操作目的**：构建带 Java 环境的业务镜像（**修正点：** 修复了之前 Dockerfile 缺少 Java 环境导致的 `executable file not found` 错误，并改用本地 CentOS 源）。

1. **准备文件结构**

   Plaintext

   ```
   /data/docker/car_pro/
   ├── Dockerfile
   ├── software/jdk-8u11-linux-x64.tar.gz  (必须存在)
   └── project/ruoyi-admin.jar             (必须是 75M 的真包，不能是假空壳)
   ```

2. **编写 Dockerfile (修正版)**

   Dockerfile

   ```
   # 使用本地 CentOS 7，不依赖外网
   FROM centos:7
   MAINTAINER bdqn
   
   # 解压 JDK
   ADD software/jdk-8u11-linux-x64.tar.gz /usr/local/
   
   # 配置环境变量 (路径必须对)
   ENV JAVA_HOME /usr/local/jdk1.8.0_11
   ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
   ENV PATH $JAVA_HOME/bin:$PATH
   
   # 复制 Jar 包到 /opt 目录
   ADD project/ /opt/
   
   # 设置工作目录
   WORKDIR /opt/
   EXPOSE 80
   
   # 启动命令
   CMD ["java", "-Duser.timezone=Asia/Shanghai", "-jar", "ruoyi-admin.jar"]
   ```

3. **构建与推送**

   Bash

   ```
   docker build -t 192.168.200.104/car_prod/ruoyi_demo:1.0 .
   docker push 192.168.200.104/car_prod/ruoyi_demo:1.0
   ```

------



## 第五阶段：NFS 存储服务端 (IP: 192.168.200.106)



Bash

```
mkdir -pv /data/store
echo "/data/store 192.168.200.0/24(rw,no_root_squash)" > /etc/exports
systemctl enable --now nfs
```

------



## 第六阶段：Kubernetes 部署 (在 Master 100 上执行)



**这是核心部分，包含了所有修复后的 YAML 文件。**



### 1. 部署 NFS 供应商 (🚨 修正权限 Bug)



使用修正后的 `01_services_account_nfs.yaml`，增加了 `endpoints` 权限，防止 `CrashLoopBackOff`。

**执行命令：**

Bash

```
# 请使用之前对话中提供的完整 01-04 YAML 内容
kubectl apply -f 01_services_account_nfs.yaml
kubectl apply -f 02_nfs-deployment.yaml
kubectl apply -f 03_nfs-storageclass.yaml
kubectl apply -f 04_nfs-pvc-project.yaml
```



### 2. 部署应用 (🚨 修正数据库连接 & 挂载覆盖问题)



修正点 1：通过 env 注入数据库连接信息，指定 192.168.200.103 和 ruoyi_demo 用户。

修正点 2：修改挂载路径为 /opt/store，防止覆盖 Jar 包。

```
vim 06_bdqn_project_car_prod.yaml
```

YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bdqn-project-car
spec:
  replicas: 1
  selector:
    matchLabels:
      app: carprod
  template:
    metadata:
      labels:
        app: carprod
    spec:
      containers:
      - name: harbor-project-car
        image: 192.168.200.104/car_prod/ruoyi_demo:1.0
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        
        # 👇 修正：配置环境变量连接物理机数据库
        env:
        - name: SPRING_DATASOURCE_DRUID_MASTER_URL
          value: "jdbc:mysql://192.168.200.103:3306/ruoyi_demo?useUnicode=true&characterEncoding=utf8&zeroDateTimeBehavior=convertToNull&useSSL=false&serverTimezone=GMT%2B8"
        - name: SPRING_DATASOURCE_DRUID_MASTER_PASSWORD
          value: "123123"
        - name: SPRING_DATASOURCE_DRUID_MASTER_USERNAME
          value: "ruoyi_demo"

        # 👇 修正：挂载到子目录，防止覆盖 Jar 包
        volumeMounts:
        - name: nfs-pvc
          mountPath: /opt/store

      volumes:
      - name: nfs-pvc
        persistentVolumeClaim:
          claimName: nfs-project-pvc
kubectl apply -f 06_bdqn_project_car_prod.yaml
```



### 3. 部署 Service (🚨 修正端口映射问题)



**修正点**：将 `targetPort` 改为 `80`，匹配容器内 Tomcat 端口，修复 503 错误。

```
vim 07_bdqn_project_car_prod_services.yaml
```

YAML

```
apiVersion: v1
kind: Service
metadata:
  name: bdqn-prod-svc
  namespace: default
spec:
  selector:
    app: carprod
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80  # 🚨 修正：必须是 80，不能是 8080
  type: ClusterIP
kubectl apply -f 07_bdqn_project_car_prod_services.yaml
```



### 4. 部署 Ingress



Bash

```
kubectl apply -f 08_install_ingress_nginx.yaml
kubectl apply -f 09_bdqn_project_car_prod_ingress.yaml

# 🚨 绑定外网 IP (必做)
kubectl edit svc ingress-nginx-controller -n ingress-nginx
# 在 spec 下添加：
# externalIPs:
# - 192.168.200.101
```

------



## 第七阶段：最终验证

```bash
# 1. Ingress Controller 必须运行在 Ready 的节点上
kubectl get pods -n ingress-nginx -o wide

# 2. Service 必须有 Endpoints
kubectl get endpoints bdqn-prod-svc

# 3. Ingress 规则已正确绑定
kubectl get ingress

# 4. 在 Node1 上测试本地访问（Pod IP 来自上面的 endpoints）
curl http://10.244.1.13

# 5. 在 Node1 上检查 80 端口是否监听
ss -tlnp | grep 80
```



1. Windows 配置 Hosts：

   192.168.200.101 bqdn.project.mslinux.com

2. 浏览器访问：

   http://bqdn.project.mslinux.com/

**至此，若依系统应能完美运行！**