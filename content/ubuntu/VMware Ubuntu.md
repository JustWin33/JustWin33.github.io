---
title: Ubuntu
description: 
date: 2025-12-04
categories:
    - 
    - 
---
VMware Ubuntu

1、安装粘贴复制功能

```bash
sudo apt install -y open-vm-tools open-vm-tools-desktop

#强制重启虚拟机（必须）
sudo reboot
```

2、启动SSH服务（Xshell7）

```bash
# 1. 安装SSH服务
sudo apt update
sudo apt install -y openssh-server

# 2. 启动服务（Ubuntu服务名为ssh）
sudo systemctl enable --now ssh

# 3. 验证安装
sudo systemctl status ssh --no-pager
sudo ss -tuln | grep :22
预期输出：
tcp 0 0 0.0.00:22 0.0.0.0:* LISTEN
```

3、设置root密码

```bash
# 设置root密码（必须）      8位数
sudo passwd root                  

# 测试root登录
su -

## 立即禁用root SSH登录（安全规范）  也可以不用管
sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config

#启动sshd
sudo systemctl restart sshd


```

4、系统更新与版本确认

```bash
# 查看系统版本（关键！不同版本命令有差异）
lsb_release -a

# 更新软件源并升级系统
sudo apt update && sudo apt upgrade -y && sudo apt autoremove -y

# 查看内核版本
uname -a
```



### **创建专用运维用户**

```bash
# 如果当前用户权限不足，创建运维专用账户
sudo useradd -m -s /bin/bash -G sudo ops
sudo passwd ops

# 为新用户配置免密sudo（生产环境慎用）
echo "ops ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ops
sudo chmod 440 /etc/sudoers.d/ops
```

### 4. **配置系统时区与NTP**

bash

复制

```bash
# 设置时区为上海
sudo timedatectl set-timezone Asia/Shanghai

# 安装并启用NTP同步
sudo apt install chrony -y
sudo systemctl enable chrony
sudo systemctl start chrony
timedatectl status  # 确认NTP synchronized: yes
```

------

## **Phase 2: 网络与安全配置（20分钟）**

### 5. **配置静态IP（企业环境必需）**

bash

复制

```bash
# 查看网卡名称（18.04+使用netplan）
ip a  # 记住网卡名如ens33

# 编辑netplan配置
sudo vim /etc/netplan/01-netcfg.yaml

# 写入以下内容（根据实际网卡名和IP修改）
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: no
      addresses: [192.168.1.100/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [223.5.5.5, 8.8.8.8]

# 应用配置
sudo netplan apply
ip a  # 验证IP是否生效
```

### 6. **SSH服务加固（生产级配置）**

bash

复制

```bash
# 安装SSH服务
sudo apt install openssh-server -y

# 备份原始配置
sudo cp /etc/ssh/sshd_config{,.bak}

# 关键配置项（逐条执行或编辑文件）
sudo sed -i 's/#Port 22/Port 2222/' /etc/ssh/sshd_config                    # 修改默认端口
sudo sed -i 's/#PermitRootLogin yes/PermitRootLogin no/' /etc/ssh/sshd_config
sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo sed -i 's/#MaxAuthTries 6/MaxAuthTries 3/' /etc/ssh/sshd_config
sudo sed -i '$a AllowUsers ops' /etc/ssh/sshd_config                        # 仅允许ops用户

# 重启SSH并设置开机自启
sudo systemctl restart sshd
sudo systemctl enable sshd

# 查看监听状态
ss -tuln | grep 2222
```

### 7. **配置主机名与hosts**

bash

复制

```bash
# 设置规范主机名（建议格式：业务-角色-序号）
sudo hostnamectl set-hostname dev-web-01

# 写入hosts文件（便于内部解析）
sudo vim /etc/hosts
# 添加：
127.0.0.1 localhost dev-web-01
192.168.1.100 dev-web-01
```

### 8. **配置防火墙（UFW）**

bash

复制

```bash
# 启用UFW（Uncomplicated Firewall）
sudo apt install ufw -y

# 默认拒绝所有入站
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 开放必要端口（根据实际服务调整）
sudo ufw allow 2222/tcp    # SSH
sudo ufw allow 80/tcp      # HTTP
sudo ufw allow 443/tcp     # HTTPS

# 启用防火墙
sudo ufw enable
sudo ufw status verbose
```

------

## **Phase 3: 运维核心工具安装（20分钟）**

### 9. **必备命令行工具集**

bash

复制

```bash
# 网络诊断工具
sudo apt install -y net-tools iputils-ping traceroute dnsutils tcpdump nmap telnet netcat

# 文本处理与查看
sudo apt install -y vim tree htop iotop sysstat lsof strace

# 压缩解压
sudo apt install -y unzip gzip bzip2 tar

# 版本控制
sudo apt install -y git subversion
git config --global user.name "ops"
git config --global user.email "ops@company.com"

# 下载工具
sudo apt install -y wget curl axel
```

### 10. **系统监控与性能分析**

bash

复制

```bash
# 实时监控
sudo apt install -y htop iotop glances

# 日志分析
sudo apt install -y logwatch

# 进程监控
sudo apt install -y atop

# 磁盘与IO
sudo apt install -y iostat

# 服务管理（systemd自带，但可加强）
sudo apt install -y systemd-coredump
```

### 11. **Docker与容器环境（现代运维必备）**

bash

复制

```bash
# 卸载旧版本
sudo apt remove -y docker docker.io containerd runc

# 安装依赖
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# 添加Docker官方GPG密钥和仓库
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 配置用户权限（避免每次sudo）
sudo usermod -aG docker $USER
newgrp docker

# 验证安装
docker --version
docker run hello-world
```

------

## **Phase 4: 系统加固与优化（15分钟）**

### 12. **内核参数优化（/etc/sysctl.conf）**

```bash
# 备份
sudo cp /etc/sysctl.conf{,.bak}

# 追加核心参数
sudo tee -a /etc/sysctl.conf <<EOF
# 网络性能优化
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15

# 文件句柄限制
fs.file-max = 65535

# 禁用IPv6（根据企业规范）
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1

# 安全加固
net.ipv4.icmp_echo_ignore_broadcasts = 1
net.ipv4.conf.all.accept_redirects = 0
EOF

# 生效
sudo sysctl -p
```

### 13. limits.conf 配置（进程资源限制）

```bash
sudo vim /etc/security/limits.conf

# 添加以下内容
* soft nofile 65535
* hard nofile 65535
* soft nproc 65535
* hard nproc 65535
```

### 14. **日志轮转优化**

```bash
# 查看当前logrotate配置
sudo vim /etc/logrotate.d/rsyslog

# 建议修改：减小单个日志文件大小，加快轮转
# 将weekly改为daily，rotate 4改为rotate 7
```

### 15. **安装配置fail2ban（防暴力破解）**

```bash
sudo apt install -y fail2ban

# 创建SSH防护规则
sudo vim /etc/fail2ban/jail.local
# 写入：
[sshd]
enabled = true
port = 2222
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 3600

# 启动服务
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

------

## **Phase 5: 备份与文档（5分钟）**

### 16. **创建系统快照（黄金规则）**

```
VMware菜单 → 虚拟机 → 快照 → 拍摄快照
```

**命名规范**：`Prod-Ready-Ubuntu-2024-11-13` **描述**：基础系统+SSH加固+监控工具+Docker环境

### 17. **生成配置文档**

```bash
# 记录系统配置
sudo mkdir -p /opt/docs
sudo vim /opt/docs/system-init-report.md

# 写入关键信息模板：
## 系统初始化报告
- 主机名: dev-web-01
- IP地址: 192.168.1.100
- SSH端口: 2222
- 系统版本: Ubuntu 22.04.3 LTS
- 内核版本: 5.15.x
- 初始化时间: 2024-11-13
- Docker版本: 24.0.x
- 允许登录用户: ops

## 安全加固项
- [x] 禁用Root SSH登录
- [x] 修改SSH端口
- [x] 配置UFW防火墙
- [x] 安装fail2ban

## 已安装核心工具
- net-tools, tcpdump, nmap
- docker, docker-compose
- htop, iotop, glances
```

------

## **运维工程师每日检查清单（添加到alias）**

```bash
# 添加到~/.bashrc
echo "alias sys-check='echo ===系统检查===; uptime; free -h; df -h; ss -tuln; docker ps --format \"table {{.Names}}\\t{{.Status}}\"'" >> ~/.bashrc
source ~/.bashrc

# 使用
sys-check
```