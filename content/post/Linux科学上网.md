---
title: Linux科学上网
description: 
date: 2025-11-10
categories:
    - 
    - 
---

# Linux科学上网

## 一、主流工具对比与选择

| 工具            | 特点                           | 适用场景   | 安装难度 |
| --------------- | ------------------------------ | ---------- | -------- |
| **Clash**       | 支持多协议、智能分流、图形界面 | 全能型首选 | 中等     |
| **Shadowsocks** | 轻量、快速、抗干扰能力强       | 日常浏览   | 简单     |
| **V2Ray**       | 高度可配置、复杂路由           | 高级用户   | 较高     |
| **OpenVPN**     | 传统VPN、稳定性好              | 企业环境   | 中等     |
| **WireGuard**   | 现代加密、性能卓越             | 追求速度   | 简单     |

---

## 二、Clash 配置教程（两种方案）

### 方案一：ShellCrash 一键安装（推荐新手）

ShellCrash 提供了自动化安装和Web管理界面，极大简化了配置过程。

#### 1. 前置准备
```bash
# 更新系统并安装必要工具
sudo apt update
sudo apt install -y bzip2 tar curl

# 确保有root权限（脚本需要）
sudo -i
```

#### 2. 安装 ShellCrash
```bash
# 使用Jsdelivr CDN（推荐）
export url='https://fastly.jsdelivr.net/gh/juewuy/ShellCrash@master' && wget -q --no-check-certificate -O /tmp/install.sh $url/install.sh && bash /tmp/install.sh

# 备用安装源（如果主源不可用）
export url='https://gh.jwsc.eu.org/master' && bash -c "$(curl -kfsSl $url/install.sh)" && source /etc/profile
```

#### 3. 配置管理面板
```bash
# 放行Web管理端口（默认9999）
iptables -I INPUT -p tcp --dport 9999 -j ACCEPT

# 保存iptables规则（Debian/Ubuntu）
apt-get install -y iptables-persistent
netfilter-persistent save
```

#### 4. 访问管理界面
在浏览器中访问：`http://<你的服务器IP>:9999/ui`（注意：不是192.168.1.1，请替换为实际服务器IP）

#### 5. 测试连接
```bash
# 测试能否访问Google
curl google.com

# 安装并运行测速工具
sudo apt install -y speedtest-cli
speedtest-cli
```

**提示**：首次访问Web面板后，您需要上传或订阅Clash配置文件。

---

### 方案二：手动安装Clash（适合自定义需求）

#### 1. 安装Clash核心
```bash
# 下载最新版本（以v1.18.0为例）
wget https://github.com/Dreamacro/clash/releases/download/v1.18.0/clash-linux-amd64-v1.18.0.gz

# 解压并安装
gunzip clash-linux-amd64-v1.18.0.gz
sudo mv clash-linux-amd64-v1.18.0 /usr/local/bin/clash
sudo chmod +x /usr/local/bin/clash
```

#### 2. 创建配置文件 `~/.config/clash/config.yaml`
```yaml
mixed-port: 7890  # HTTP和SOCKS5代理端口
allow-lan: false
mode: Rule  # Rule/Global/Direct
log-level: info

proxies:
  - name: "your-proxy"
    type: ss  # 或 vmess, trojan等
    server: your_server_ip
    port: 8388
    password: "your_password"
    cipher: aes-256-gcm

rules:
  - DOMAIN-SUFFIX,google.com,your-proxy
  - GEOIP,CN,DIRECT
  - MATCH,your-proxy
```

#### 3. 启动Clash
```bash
# 后台运行
nohup clash -d ~/.config/clash/ &

# 设置系统代理（临时）
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
```

**验证连接**：`curl -x http://127.0.0.1:7890 ifconfig.me`

---

## 三、Shadowsocks 快速配置

经典轻量级代理方案。

```bash
# Ubuntu/Debian
sudo apt-get install shadowsocks-libev

# 创建配置文件 /etc/shadowsocks.json
{
  "server": "your_server_ip",
  "server_port": 8388,
  "local_address": "127.0.0.1",
  "local_port": 1080,
  "password": "your_password",
  "timeout": 300,
  "method": "aes-256-gcm"
}

# 启动客户端
ss-local -c /etc/shadowsocks.json
```

**启动命令**：`ss-local -c /etc/shadowsocks.json`

---

## 四、V2Ray 高级配置

提供更强大的流量伪装和路由功能。

```bash
# 下载官方脚本
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)

# 配置 /usr/local/etc/v2ray/config.json
{
  "inbounds": [{
    "port": 1080,
    "protocol": "socks"
  }],
  "outbounds": [{
    "protocol": "vmess",
    "settings": {
      "vnext": [{
        "address": "your_server_ip",
        "port": 10086,
        "users": [{
          "id": "your-uuid",
          "alterId": 64
        }]
      }]
    }
  }]
}

# 启动服务
systemctl start v2ray
```

---

## 五、OpenVPN 传统方案

适合需要稳定加密隧道的场景。

```bash
# 1. 安装
sudo apt-get update && sudo apt-get install openvpn

# 2. 下载VPN商提供的 .ovpn 配置文件
sudo cp your-config.ovpn /etc/openvpn/client.conf

# 3. 启动连接
sudo openvpn --config /etc/openvpn/client.conf

# 4. 验证IP
curl ifconfig.me
```

---

## 六、系统级代理配置

### 1. 临时环境变量（当前终端有效）
```bash
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export ALL_PROXY="socks5://127.0.0.1:7890"
```

### 2. 永久配置
编辑 `~/.bashrc` 或 `~/.zshrc`：
```bash
# 代理设置
export http_proxy="http://127.0.0.1:7890"
export https_proxy="http://127.0.0.1:7890"
export no_proxy="localhost,127.0.0.1,192.168.0.0/16"
```

### 3. APT包管理器代理
创建 `/etc/apt/apt.conf.d/proxy.conf`：
```
Acquire::http::Proxy "http://127.0.0.1:7890";
Acquire::https::Proxy "http://127.0.0.1:7890";
```

---

## 七、应用场景配置

### 1. 命令行工具
```bash
# wget 使用代理
echo "http_proxy = http://127.0.0.1:7890/" >> ~/.wgetrc

# git 使用代理
git config --global http.proxy http://127.0.0.1:7890
git config --global https.proxy http://127.0.0.1:7890

# Docker 代理
sudo mkdir -p /etc/systemd/docker.service.d
sudo tee /etc/systemd/docker.service.d/proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:7890"
Environment="HTTPS_PROXY=http://127.0.0.1:7890"
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### 2. 浏览器配置
- **Firefox**: 设置 → 网络设置 → 手动代理配置 → SOCKS5 127.0.0.1:7890
- **Chrome**: 使用插件 SwitchyOmega 或启动参数 `--proxy-server="socks5://127.0.0.1:7890"`

---

## 八、高级技巧

### 1. 透明代理（全局流量）
使用iptables实现全局代理：
```bash
# 将本机流量重定向到Clash的7890端口
sudo iptables -t nat -A OUTPUT -p tcp -m owner ! --uid-owner root -j REDIRECT --to-port 7890
```

### 2. SSH隧道（临时方案）
```bash
ssh -D 1080 -C -N user@your_server
# 然后在浏览器中设置SOCKS5代理 127.0.0.1:1080
```

---

## 九、故障排查

1. **检查端口占用**: `netstat -tlnp | grep 7890`
2. **查看日志**: 
   - ShellCrash: `journalctl -u ShellCrash -f`
   - Clash: `journalctl -u clash -f`
3. **测试连通性**: `curl -v --proxy http://127.0.0.1:7890 http://google.com`
4. **防火墙设置**: `sudo ufw allow 7890/tcp`
5. **SSH连接问题**: 如果遇到"port 22: Connection refused"，需安装SSH服务：
   ```bash
   # Debian/Ubuntu
   sudo apt-get install openssh-server
   # CentOS/RHEL
   sudo yum install openssh-server
   ```

---

## 十、安全建议

1. **避免使用免费公共节点**，存在隐私泄露风险
2. **定期更新客户端**以获取最新的安全补丁
3. **使用强加密算法**（如aes-256-gcm、chacha20-poly1305）
4. **配置no_proxy**绕过国内网站，提升访问速度
5. **开启日志记录**，便于排查问题但不记录敏感信息
6. **谨慎使用root权限**，尽量使用sudo执行特定命令

---

## 总结

对于**新手用户**，强烈推荐使用**ShellCrash一键安装方案**，它提供了Web界面管理，操作简单直观。对于需要自定义配置的高级用户，可以选择**手动安装Clash**或其他工具。配置完成后，可通过`curl ifconfig.me`验证IP是否变更，确认科学上网是否生效。