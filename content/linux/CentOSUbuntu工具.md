---
title: Linux实战手册
description: 
date: 2025-11-09
categories:
    - 
    - 
---

# Linux 运维实战手册 (CentOS/Ubuntu 通用版)

## 一、系统核心命令与资源管理

本节包含系统最基础的信息侦查与存储管理，其中 LVM 管理部分包含了从物理磁盘到逻辑卷挂载的完整闭环操作。

### 1.1 系统信息侦查 (System Info)

快速获取系统指纹、负载及内核状态。

Bash

```
# 综合信息（推荐）
hostnamectl                      # 查看内核版本、架构、虚拟化类型 (CentOS 7+/Ubuntu 16.04+)
cat /etc/os-release              # 查看发行版详细名称 (所有发行版标准)
lscpu | grep "Model name"        # 查看CPU具体型号
free -h                          # 查看内存使用情况 (人类可读格式)
df -hT                           # 查看磁盘挂载点及文件系统类型
uptime                           # 查看系统运行时间及平均负载
dmesg | tail -50                 # 查看最近50行内核启动/硬件日志

# 快速诊断别名 (直接复制到终端执行)
alias sysinfo='echo "--- CPU ---"; lscpu | grep "Model name"; echo "--- Memory ---"; free -h; echo "--- Disk ---"; df -hT | grep -v tmpfs; echo "--- Network ---"; ip a | grep "inet "; echo "--- Load ---"; uptime'
```

### 1.2 LVM 逻辑卷管理（完整全流程）

LVM 允许灵活调整分区大小。以下是从裸磁盘到文件系统的完整步骤。

**操作场景：** 假设有两块新硬盘 `/dev/sdb` and `/dev/sdc`。

1. **创建物理卷 (PV - Physical Volume)**

   ```
   pvcreate /dev/sdb /dev/sdc
   pvs  # 验证PV
   ```

2. **创建卷组 (VG - Volume Group)**

   ```
   # 将sdb和sdc加入名为 vg_data 的卷组
   vgcreate vg_data /dev/sdb /dev/sdc
   vgs  # 验证VG
   ```

3. **创建逻辑卷 (LV - Logical Volume)**

   ```
   # 创建固定大小 50G 的LV，命名为 lv_app
   lvcreate -L 50G -n lv_app vg_data
   # 将剩余所有空间创建为 lv_backup
   lvcreate -l 100%FREE -n lv_backup vg_data
   lvs  # 验证LV
   ```

4. **格式化文件系统**

   ```
   mkfs.xfs /dev/vg_data/lv_app      # CentOS 推荐使用 XFS
   mkfs.ext4 /dev/vg_data/lv_backup  # Ubuntu 推荐使用 EXT4
   ```

5. **挂载目录**

   ```
   mkdir /app /backup
   mount /dev/vg_data/lv_app /app
   mount /dev/vg_data/lv_backup /backup
   # 注意：永久挂载需编辑 /etc/fstab
   ```

6. **在线扩容 (无需停机)**

   ```
   # 1. 扩容逻辑卷
   lvextend -L +20G /dev/vg_data/lv_app
   
   # 2. 刷新文件系统大小
   xfs_growfs /app                   # 如果是 XFS 格式
   # resize2fs /dev/vg_data/lv_app   # 如果是 EXT4 格式
   ```

------

## 二、文件与磁盘高级管理

### 2.1 磁盘清理与分析

当磁盘报警时，使用以下命令快速定位问题。

```
# 快速定位大文件（排除虚拟文件系统）
du -ah --max-depth=2 /var | sort -rh | head -20

# 查找大于100M的文件
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | head

# 检查inode耗尽问题（小文件过多导致）
df -i

# 交互式磁盘分析工具 (强力推荐)
yum install -y ncdu || apt install -y ncdu
ncdu /  # 使用键盘上下键浏览和删除

# 自动化清理：删除 /tmp 下7天未访问的文件
find /tmp -type f -atime +7 -delete
```

### 2.2 文件传输技巧

```
# rsync 高级同步 (带进度条、断点续传、限速)
# 将本地 /src/ 同步到远程，限速 10MB/s
rsync -avz --progress --partial --bwlimit=10000 /src/ user@remote:/dst/

# lrzsz (Xshell/SecureCRT 拖拽上传工具)
yum install -y lrzsz || apt install -y lrzsz
rz -y           # 弹窗选择文件上传
sz filename     # 下载文件到本地
```

------

## 三、网络诊断与抓包分析

### 3.1 基础网络诊断

```
# 查看已建立的TCP连接
ss -antp | grep ESTAB

# 查看端口占用的进程 (PID)
lsof -i :8080

# 路由追踪 (MTR 是 traceroute 的增强版，显示丢包率)
yum install -y mtr || apt install -y mtr-tiny
mtr -r -c 10 baidu.com
```

### 3.2 抓包分析 (Tcpdump/Wireshark)

生产环境排查网络问题的利器。

```
# 精确抓包：抓取 eth0 网卡，端口 80 且来源/目标是 192.168.1.100 的包
tcpdump -i eth0 -w capture.pcap port 80 and host 192.168.1.100

# SQL审计：抓取 MySQL 流量并过滤 "select" 语句
tcpdump -A -s0 port 3306 and 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x73656c65'

# Tshark (命令行版 Wireshark) - 实时分析 HTTP 请求
# 安装: yum install wireshark / apt install tshark
tshark -i eth0 -Y "http.request" -T fields -e http.host -e http.request.uri
```

### 3.3 端口与漏洞扫描

```
# 扫描网段存活主机
nmap -sP 192.168.1.0/24

# 快速全端口扫描
nmap -sS -p 1-65535 -T4 target_ip

# Masscan (超高速扫描，慎用，可能会打挂防火墙)
masscan 192.168.0.0/16 -p 80,443 --rate=10000
```

------

## 四、监控与性能分析

### 4.1 实时监控工具集

这些工具建议全部安装，分别对应不同维度的监控。

- **Glances**: 全能系统监控 (CPU/Mem/Disk/Net/Proc)。
  - 命令: `glances` (Web模式: `glances -w`)
- **Nethogs**: 查看**哪个进程**在偷跑流量。
  - 命令: `nethogs -d 5 -v 3`
- **Iotop**: 查看**哪个进程**在疯狂读写磁盘。
  - 命令: `iotop -oP`
- **Iftop**: 查看网卡实时的流量来源和去向 IP。
  - 命令: `iftop -i eth0 -n -P`

### 4.2 深度性能分析

```
# Strace: 跟踪进程的系统调用 (排查程序卡住、权限错误)
strace -c -p PID   # 统计系统调用耗时
strace -f -p PID   # 跟踪进程及其子进程

# Perf: Linux 内核级性能分析
# 安装: yum install perf / apt install linux-tools-common
perf top -p PID    # 实时查看函数级热点

# Sar: 查看历史性能数据 (sysstat 包)
sar -u 1 10        # 查看CPU历史
sar -n DEV 1 10    # 查看网络历史
```

------

## 五、文本编辑器终极配置

### 5.1 Vim 运维专用配置

这是本手册的核心资产之一。它将 Vim 变成了 IDE，支持文件树、Git 状态、代码高亮和智能补全。

**安装步骤：**

1. 安装 Vim：`yum install -y vim-enhanced` 或 `apt install -y vim`
2. 复制以下内容并执行：

```
cat > ~/.vimrc << 'EOF'
" ==================== 运维专用 Vim 配置 ====================
set nocompatible
syntax on
set number relativenumber       " 显示相对行号
set cursorline ruler showcmd    " 高亮当前行，显示状态
set incsearch hlsearch ignorecase smartcase " 智能搜索
set wrap linebreak              " 自动换行
set tabstop=4 shiftwidth=4 expandtab autoindent smartindent " 缩进设置
set mouse=a clipboard=unnamedplus " 启用鼠标和系统剪贴板
set encoding=utf-8 fileencodings=utf-8,gbk,gb2312,gb18030
set pastetoggle=<F2>            " F2 切换粘贴模式（防止缩进错乱）
set listchars=tab:>-,trail:·,space:· " 显示不可见字符
nnoremap <F3> :set list!<cr>    " F3 开关特殊字符显示

" 快速编辑/重载 vimrc
nnoremap <leader>ev :vsplit $MYVIMRC<cr>
nnoremap <leader>sv :source $MYVIMRC<cr>

" --- 插件管理 (Vim-Plug) ---
" 首次运行需执行: curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
call plug#begin('~/.vim/plugged')

Plug 'preservim/nerdtree'           " 文件树 (<F5> 开关)
nnoremap <F5> :NERDTreeToggle<cr>
let NERDTreeShowHidden=1

Plug 'vim-airline/vim-airline'      " 状态栏
Plug 'vim-airline/vim-airline-themes'
Plug 'tpope/vim-fugitive'           " Git 集成
Plug 'luochen1990/rainbow'          " 彩虹括号
Plug 'MTDL9/vim-log-highlighting'   " 日志文件高亮
Plug 'andrewstuart/vim-kubernetes'  " K8s yaml 支持
Plug 'Yggdroot/indentLine'          " 缩进参考线

call plug#end()

" --- 特定文件优化 ---
" YAML 自动2空格缩进 (Ansible/K8s 必备)
autocmd FileType yaml setlocal ts=2 sts=2 sw=2 expandtab

" 大文件性能优化 (>10MB 关闭高亮和语法)
let g:LargeFile=10
autocmd BufReadPre * let f=getfsize(expand('%'))
autocmd BufReadPre * if f > g:LargeFile*1024*1024 | set eventignore+=FileType | set syntax=OFF | set filetype=conf | endif
EOF
```

*注：配置生效需联网并在 Vim 中运行 `:PlugInstall`*

------

## 六、安全加固与审计

### 6.1 入侵防御 (Fail2ban)

自动封禁多次 SSH 密码错误的 IP。

```
# 1. 安装
yum install -y fail2ban || apt install -y fail2ban

# 2. 配置 (/etc/fail2ban/jail.local)
cat > /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
port = 22
filter = sshd
# CentOS日志路径: /var/log/secure, Ubuntu: /var/log/auth.log
logpath = /var/log/secure
maxretry = 3        # 尝试3次失败即封禁
bantime = 3600      # 封禁1小时
findtime = 600      # 统计10分钟内的失败次数
EOF

# 3. 启动
systemctl enable --now fail2ban
fail2ban-client status sshd
```

### 6.2 安全扫描

- **Lynis**: `lynis audit system --quick` (系统安全全面体检)
- **Rootkit检查**: `rkhunter --check --sk`
- **病毒扫描**: `clamscan -r /var/www`

------

## 七、自动化运维 (Ansible)

无需 Agent，通过 SSH 管理多台主机。

```
# 1. 安装 Ansible
pip3 install ansible --user

# 2. 基础配置 (~/.ansible.cfg) - 关闭HostKey检查以提高速度
cat > ~/.ansible.cfg << EOF
[defaults]
host_key_checking = False
inventory = ./hosts
forks = 50
[privilege_escalation]
become = True
become_method = sudo
become_user = root
EOF

# 3. 主机清单 (hosts)
cat > hosts << EOF
[web]
web01.example.com
web02.example.com
EOF

# 4. 测试命令
ansible all -m ping
ansible web -m shell -a "uptime"
```

------

## 八、终端效率神器

### 8.1 Tmux (终端复用器)

运维必备，防止 SSH 断开导致任务中断。

**终极配置 (~/.tmux.conf):**

```
# 将前缀键修改为 Ctrl+A (同 Screen 习惯)
unbind C-b
set -g prefix C-a
bind C-a send-prefix

set -g mouse on                 # 开启鼠标支持 (点击切换窗口，拖拽调整大小)
set -g status-right "%Y-%m-%d %H:%M"

# 快捷分屏
bind % split-window -h -c "#{pane_current_path}"  # 左右分屏
bind '"' split-window -v -c "#{pane_current_path}" # 上下分屏
```

**常用操作:**

- `tmux new -s name`: 新建会话
- `Ctrl+A` 然后 `d`: 暂时离开会话 (任务后台继续运行)
- `tmux attach -t name`: 回到会话

### 8.2 Fzf (模糊查找)

```
# 历史记录搜索增强
history | fzf
# 文件快速查找
find . -type f | fzf
```

------

## 九、云原生工具集 (Docker & K8s)

### 9.1 Docker 优化配置

安装后配置国内镜像源及日志轮转（防止日志撑爆磁盘）。

```
cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://mirror.gcr.io"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  }
}
EOF
```

### 9.2 Kubernetes 别名 (kubectl)

```
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias ke='kubectl exec -it'
alias klog='kubectl logs -f'
```

------

## 十、一键全家桶安装脚本

请直接在服务器上以 root 身份运行以下脚本，即可安装上述所有工具。

### 10.1 CentOS 7/8/9 专用脚本

```
cat > install_ops_tools_centos.sh << 'EOF'
#!/bin/bash
set -e
echo ">>> Step 1: 安装 EPEL 源"
yum install -y epel-release

echo ">>> Step 2: 安装基础系统工具"
yum install -y vim htop iotop iftop nethogs glances git telnet net-tools bind-utils traceroute mtr nc tcpdump nmap lsof strace ltrace sysstat dstat bash-completion chrony unzip zip psmisc screen tmux expect pv jq python3-pip ncdu tree smartmontools ipmitool dmidecode

echo ">>> Step 3: 安装 Docker"
curl -fsSL https://get.docker.com | bash
systemctl enable --now docker

echo ">>> Step 4: 安装 Python 运维工具"
pip3 install --user mycli pgcli ansible yq thefuck

echo ">>> Step 5: 配置 Vim 插件管理器"
curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim

echo ">>> 安装完成！请记得运行 source ~/.bashrc 加载配置"
EOF
chmod +x install_ops_tools_centos.sh
```

### 10.2 Ubuntu/Debian 专用脚本

```
cat > install_ops_tools_ubuntu.sh << 'EOF'
#!/bin/bash
set -e
echo ">>> Step 1: 更新软件源"
apt update

echo ">>> Step 2: 安装基础系统工具"
apt install -y vim htop iotop iftop nethogs glances git telnet net-tools dnsutils traceroute mtr netcat-openbsd tcpdump nmap lsof strace ltrace sysstat dstat bash-completion chrony unzip zip psmisc screen tmux expect pv jq python3-pip ncdu tree smartmontools ipmitool dmidecode

echo ">>> Step 3: 安装 Docker"
curl -fsSL https://get.docker.com | bash
systemctl enable --now docker

echo ">>> Step 4: 安装 Python 运维工具"
pip3 install --user mycli pgcli ansible yq thefuck

echo ">>> Step 5: 配置 Vim 插件管理器"
curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim

echo ">>> 安装完成！请记得运行 source ~/.bashrc 加载配置"
EOF
chmod +x install_ops_tools_ubuntu.sh
```

------

## 十一、运维别名库 (.bashrc)

将以下内容追加到 `~/.bashrc`，可极大幅度提升工作效率。

```
# === 复制以下内容到 .bashrc 底部 ===

# 1. 实用信息别名
alias ports='ss -tuln | grep LISTEN'
alias bigfiles='find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | head -20'
alias iplist='ip a | grep "inet " | awk \'{print $2}\''

# 2. Docker 快捷指令
alias dps='docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"'
alias dsa='docker stop $(docker ps -aq)'  # 停止所有容器
alias drma='docker rm $(docker ps -aq)'   # 删除所有容器
alias dcl='docker system prune -a'        # 清理所有未使用镜像和缓存

# 3. 安全与历史记录
alias rm='rm -i' # 删除前必须确认
export HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S " # 历史记录带时间戳
export HISTSIZE=10000

# 4. 万能解压函数 (输入 extract 文件名 即可解压任何格式)
extract() {
    if [ -f $1 ]; then
        case $1 in
            *.tar.bz2) tar xjf $1 ;;
            *.tar.gz)  tar xzf $1 ;;
            *.rar)     unrar x $1 ;;
            *.gz)      gunzip $1 ;;
            *.tar)     tar xf $1 ;;
            *.zip)     unzip $1 ;;
            *)         echo "'$1' cannot be extracted via extract()" ;;
        esac
    else
        echo "'$1' is not a valid file"
    fi
}
```

------

## 十二、应急响应与巡检

### 12.1 系统被入侵应急脚本

当怀疑服务器被黑时，运行此脚本保存现场证据。

```
#!/bin/bash
# incident_response.sh
mkdir -p /tmp/incident_$(date +%Y%m%d)
cd /tmp/incident_$(date +%Y%m%d)

echo ">>> 保存进程与网络状态..."
ps aux > ps.aux.txt
netstat -tulpn > netstat.tulpn.txt
ss -antp > ss.antp.txt

echo ">>> 保存登录日志..."
last > last.txt
cp /var/log/secure . 2>/dev/null
cp /var/log/auth.log . 2>/dev/null

echo ">>> 查找高资源消耗进程..."
ps aux | awk '$3 > 50 {print $0}' > high_cpu.txt # CPU > 50%

echo ">>> 查找所有可执行的临时文件 (常见木马特征)..."
find /tmp /var/tmp -type f -executable -exec ls -lh {} \; > suspicious_files.txt

echo "现场数据已保存至当前目录"
```

### 12.2 日常巡检脚本 (Crontab)

部署该脚本到 Crontab (`0 8 * * *`)，每天早上8点发送报告。

```
#!/bin/bash
# daily_check.sh
REPORT="/tmp/daily_report_$(hostname)_$(date +%Y%m%d).txt"

echo "=== $(hostname) 日常巡检 ===" > $REPORT
uptime >> $REPORT
echo "--- 磁盘使用 ---" >> $REPORT
df -hT >> $REPORT
echo "--- 异常登录 ---" >> $REPORT
grep "Failed password" /var/log/secure 2>/dev/null | tail -10 >> $REPORT
echo "--- 服务状态 (Failed) ---" >> $REPORT
systemctl list-units --state=failed --no-pager >> $REPORT

# 如果配置了邮件服务，取消下面注释
# mail -s "Daily Report: $(hostname)" ops@company.com < $REPORT
```

