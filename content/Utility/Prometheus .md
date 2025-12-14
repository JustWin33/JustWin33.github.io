---
title: Prometheus
description: 
date: 2025-12-14
categories:
    - 
    - 
---

# Prometheus 生产环境部署(CentOS 7)

### 一、 环境准备与系统要求

Prometheus 对操作系统没有严格的强制限制，只要是 Linux 系统（包括 CentOS 7）几乎都可以运行。它是用 Go 语言编写的，编译后是独立的二进制文件，不依赖复杂的系统库。

在开始之前，请确保服务器满足以下基础条件：

1. **操作系统**：CentOS 7.x (内核版本 3.10+ 即可)。
2. **硬件建议**：
   - **测试环境**：2 CPU / 4GB 内存 / 20GB 磁盘。
   - **生产环境**：建议 4 CPU / 8GB 内存 / 100GB+ SSD (根据数据保留时间和采集量调整)。
3. **时间同步**：**非常重要**。Prometheus 对时间敏感，请确保服务器时间已同步 (使用 `ntpdate` 或 `chronyd`)。
4. **防火墙**：需开放 TCP 端口 **9090** (Prometheus Server) 和 **9100** (Node Exporter)。

------

### 二、 安装 Prometheus Server (服务端)

#### 1. 创建专用系统用户

为了系统安全，禁止使用 root 运行服务。创建一个没有登录权限的 `prometheus` 用户。

```
# 创建用户，不创建家目录，禁止登录 Shell
sudo useradd --no-create-home --shell /bin/false prometheus
```

#### 2. 创建标准目录结构

遵循 Linux 目录规范，将配置、数据和二进制文件分离。

```
# 存放配置文件
sudo mkdir -p /etc/prometheus
# 存放持久化监控数据 (TSDB)
sudo mkdir -p /var/lib/prometheus
```

#### 3. 下载并部署二进制文件

从官网https://prometheus.io/download/获取安装包（以 LTS 版本为例）。

```
# 1. 安装下载工具
sudo yum install -y wget

# 2. 进入临时目录下载
cd /tmp
wget https://github.com/prometheus/prometheus/releases/download/v2.54.1/prometheus-2.54.1.linux-amd64.tar.gz

# 3. 解压文件
tar -xvf prometheus-2.54.1.linux-amd64.tar.gz

# 4. 移动二进制执行文件到 /usr/local/bin (并重命名文件夹方便操作)
cd prometheus-2.54.1.linux-amd64
sudo cp prometheus /usr/local/bin/
sudo cp promtool /usr/local/bin/

# 5. 移动配置文件和依赖库到 /etc/prometheus
sudo cp -r consoles /etc/prometheus
sudo cp -r console_libraries /etc/prometheus
sudo cp prometheus.yml /etc/prometheus/
```

#### 4. 修正文件权限

确保 `prometheus` 用户拥有对配置和数据目录的读写权限。

```
# 修改配置目录权限
sudo chown -R prometheus:prometheus /etc/prometheus
# 修改数据目录权限 (关键，否则无法写入数据)
sudo chown -R prometheus:prometheus /var/lib/prometheus
# 修改二进制文件归属
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
```

#### 5. 配置 Systemd 服务管理 (开机自启)

创建服务文件，托管给 systemd 管理。

```
sudo vi /etc/systemd/system/prometheus.service
```

**写入以下内容：**

Ini, TOML

```
[Unit]
Description=Prometheus
Documentation=https://prometheus.io/docs/introduction/overview/
Wants=network-online.target
After=network-online.target

[Service]
# 指定运行用户
User=prometheus
Group=prometheus
Type=simple
# 启动命令：指定配置文件、数据存储路径、Web控制台库路径
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.console.templates=/etc/prometheus/consoles \
    --web.console.libraries=/etc/prometheus/console_libraries \
    --web.enable-lifecycle  
    # ⬆️ 添加 --web.enable-lifecycle 可开启通过 HTTP 请求热加载配置的功能

[Install]
WantedBy=multi-user.target
```

#### 6. 启动服务与验证

```
# 重载系统服务配置
sudo systemctl daemon-reload

# 启动 Prometheus
sudo systemctl start prometheus

# 设置开机自启
sudo systemctl enable prometheus

# 检查状态 (应显示 active (running))
sudo systemctl status prometheus
```

#### 7. 配置防火墙 (CentOS 7 Firewalld)

```
# 放行 9090 端口
sudo firewall-cmd --permanent --add-port=9090/tcp
# 重载防火墙规则
sudo firewall-cmd --reload
```

此时，访问 `http://<服务器IP>:9090` 即可看到 Prometheus 界面。

------

### 三、 安装 Node Exporter (数据采集端)

Prometheus 本身只是个数据库，要监控 CentOS 系统的 CPU、内存、磁盘等指标，需要安装 **Node Exporter**。

#### 1. 下载与部署

```
# 1. 下载
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

# 2. 解压
tar -xvf node_exporter-1.8.2.linux-amd64.tar.gz

# 3. 移动二进制文件
sudo cp node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/node_exporter
```

#### 2. 配置 Node Exporter 的 Systemd 服务

```
sudo vi /etc/systemd/system/node_exporter.service
```

**写入以下内容：**

Ini, TOML

```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

#### 3. 启动 Node Exporter

```
sudo systemctl daemon-reload
sudo systemctl start node_exporter
sudo systemctl enable node_exporter

# 防火墙放行 9100 端口
sudo firewall-cmd --permanent --add-port=9100/tcp
sudo firewall-cmd --reload
```

------

### 四、 关联配置 (让 Prometheus 抓取数据)

最后一步是告诉 Prometheus Server 去哪里抓取 Node Exporter 的数据。

#### 1. 编辑配置文件

```
sudo vi /etc/prometheus/prometheus.yml
```

#### 2. 添加监控目标

在文件末尾的 `scrape_configs` 模块中，添加一个新的 job。**注意 YAML 格式对缩进非常敏感（通常是2个空格）**。

YAML

```
scrape_configs:
  # ... 原有的 prometheus job ...

  # 新增：监控本机 CentOS 7
  - job_name: 'centos7_node'
    static_configs:
      - targets: ['localhost:9100']
```

#### 3. 热加载配置

无需重启服务，发送指令让配置立即生效：

```
sudo systemctl reload prometheus
# 或者使用 curl (如果开启了 lifecycle)
# curl -X POST http://localhost:9090/-/reload
```

------

### 五、 最终验证1

1. 打开浏览器访问：`http://<服务器IP>:9090/targets`

2. 在列表中，你应该能看到 `centos7_node` 状态为 **UP**（绿色）。

3. 点击导航栏的 **Graph**，在搜索框输入 `node_memory_MemTotal_bytes` 并点击 Execute，如果能看到数值，说明监控部署完全成功。

   

   