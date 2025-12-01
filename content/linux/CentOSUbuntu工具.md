---
title: Linux工具
description: 
date: 2025-11-09
categories:
    - 
    - 
---

# CentOS/Ubuntu工具

## 一、系统核心命令

### 1.1 系统信息侦查
```bash
# 综合信息（推荐）
hostnamectl                      # CentOS 7+/Ubuntu 16.04+
cat /etc/os-release              # 所有发行版标准
lscpu | grep "Model name"        # CPU型号
free -h                          # 内存
df -hT                           # 磁盘
uptime                           # 负载
dmesg | tail -50                 # 内核日志

# 快速诊断
alias sysinfo='echo "--- CPU ---"; lscpu | grep "Model name"; echo "--- Memory ---"; free -h; echo "--- Disk ---"; df -hT | grep -v tmpfs; echo "--- Network ---"; ip a | grep "inet "; echo "--- Load ---"; uptime'
```

### 1.2 LVM逻辑卷管理（完整流程）
```bash
# 1. 创建PV
pvcreate /dev/sdb /dev/sdc
pvs

# 2. 创建VG
vgcreate vg_data /dev/sdb /dev/sdc
vgs

# 3. 创建LV
lvcreate -L 50G -n lv_app vg_data
lvcreate -l 100%FREE -n lv_backup vg_data
lvs

# 4. 格式化
mkfs.xfs /dev/vg_data/lv_app      # CentOS推荐
mkfs.ext4 /dev/vg_data/lv_backup  # Ubuntu推荐

# 5. 挂载
mkdir /app /backup
mount /dev/vg_data/lv_app /app
mount /dev/vg_data/lv_backup /backup

# 6. 扩容（在线）
lvextend -L +20G /dev/vg_data/lv_app
xfs_growfs /app                   # XFS格式
# 或 resize2fs /dev/vg_data/lv_app  # ext4格式
```

---

## 二、文件与磁盘管理

### 2.1 磁盘管理
```bash
# 查看大文件
du -ah --max-depth=2 /var | sort -rh | head -20
find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | head

# 检查inode
df -i

# 磁盘健康
smartctl -a /dev/sda
yum install -y smartmontools     # CentOS
apt install -y smartmontools     # Ubuntu

# 磁盘清理
ncdu /                           # 交互式磁盘分析
yum install -y ncdu              # CentOS
apt install -y ncdu              # Ubuntu

# 临时文件清理
find /tmp -type f -atime +7 -delete  # 删除7天未访问文件
```

### 2.2 文件传输
```bash
# lrzsz（Xshell/SecureCRT必备）
yum install -y lrzsz            # CentOS
apt install -y lrzsz            # Ubuntu
rz -y                           # 上传
sz filename                     # 下载

# rsync高级用法
rsync -avz --progress --partial --bwlimit=10000 /src/ user@remote:/dst/  # 限速10MB/s

# 断点续传
rsync --partial --progress large_file.iso user@remote:/tmp/
```

---

## 三、网络诊断工具

### 3.1 基础网络
```bash
# 查看连接
ss -antp | grep ESTAB            # 已建立连接
ss -s                            # 连接统计
lsof -i :8080                    # 端口占用进程

# 路由追踪
mtr -r -c 10 baidu.com          # 10次报告模式
yum install -y mtr                # CentOS
apt install -y mtr-tiny           # Ubuntu

# 带宽测试
iperf3 -s                        # 服务端
iperf3 -c server_ip -P 8         # 客户端8线程
yum install -y iperf3             # CentOS (EPEL)
apt install -y iperf3             # Ubuntu
```

### 3.2 抓包分析
```bash
# tcpdump精确过滤
tcpdump -i eth0 -w capture.pcap port 80 and host 192.168.1.100
tcpdump -A -s0 port 3306 and 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x73656c65'  # 过滤"select"

# tshark实时分析
yum install -y wireshark          # CentOS
apt install -y tshark             # Ubuntu
tshark -i eth0 -Y "http.request" -T fields -e http.host -e http.request.uri
```

### 3.3 端口扫描
```bash
nmap -sP 192.168.1.0/24          # 存活主机扫描
nmap -sS -p 1-65535 -T4 target   # 全端口SYN扫描
nmap --script=vuln target        # 漏洞扫描
yum install -y nmap               # CentOS
apt install -y nmap               # Ubuntu

# Masscan超高速扫描
masscan 192.168.0.0/16 -p 80,443 --rate=10000
```

---

## 四、监控与性能分析

### 4.1 实时监控
```bash
# glances全能监控
yum install -y glances          # CentOS (EPEL)
apt install -y glances          # Ubuntu
glances -w                       # Web模式 http://ip:61208

# nethogs进程级流量
yum install -y nethogs          # CentOS (EPEL)
apt install -y nethogs          # Ubuntu
nethogs -d 5 -v 3               # 5秒刷新，显示MB

# iotop IO监控
yum install -y iotop            # CentOS (EPEL)
apt install -y iotop            # Ubuntu
iotop -oP                       # 只显示有IO的进程

# iftop 流量监控
yum install -y iftop            # CentOS (EPEL)
apt install -y iftop            # Ubuntu
iftop -i eth0 -n -P             # 显示端口和IP
```

### 4.2 性能分析
```bash
# strace系统调用
strace -c -p PID                # 统计耗时
strace -f -p PID                # 跟踪子进程

# perf内核级分析
yum install -y perf            # CentOS
apt install -y linux-tools-common # Ubuntu
perf top -p PID                 # 实时分析
perf record -p PID -g           # 记录采样
perf report                     # 生成报告

# lsof查看文件句柄
lsof -p PID | wc -l             # 统计句柄数
lsof -i TCP:3306                # 查看数据库连接
```

### 4.3 历史数据
```bash
# sar系统活动报告
yum install -y sysstat          # CentOS
apt install -y sysstat          # Ubuntu
systemctl enable --now sysstat  # 启用收集

sar -u 1 10                     # CPU
sar -r 1 10                     # 内存
sar -d 1 10                     # 磁盘
sar -n DEV 1 10                 # 网络
```

---

## 五、文本编辑器配置

### 5.1 Vim 运维终极配置
```bash
# 安装
yum install -y vim-enhanced vim-X11  # CentOS
apt install -y vim vim-gtk3          # Ubuntu

# 完整配置（~/.vimrc）
cat > ~/.vimrc << 'EOF'
" ==================== 运维专用 Vim 配置 ====================
set nocompatible
syntax on
set number relativenumber
set cursorline ruler showcmd
set incsearch hlsearch ignorecase smartcase
set wrap linebreak
set tabstop=4 shiftwidth=4 expandtab autoindent smartindent
set mouse=a clipboard=unnamedplus
set encoding=utf-8 fileencodings=utf-8,gbk,gb2312,gb18030
set pastetoggle=<F2>
set listchars=tab:>-,trail:·,space:·
nnoremap <F3> :set list!<cr>

" 快速编辑vimrc
nnoremap <leader>ev :vsplit $MYVIMRC<cr>
nnoremap <leader>sv :source $MYVIMRC<cr>

" 插件管理器
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
  https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim

call plug#begin('~/.vim/plugged')

" 文件浏览器
Plug 'preservim/nerdtree'
nnoremap <F5> :NERDTreeToggle<cr>
let NERDTreeShowHidden=1

" 状态栏
Plug 'vim-airline/vim-airline'
Plug 'vim-airline/vim-airline-themes'

" Git集成
Plug 'tpope/vim-fugitive'
nnoremap <leader>gs :Gstatus<cr>

" 智能补全
Plug 'neoclide/coc.nvim', {'branch': 'release'}

" 语法检查
Plug 'dense-analysis/ale'

" 快速注释
Plug 'preservim/nerdcommenter'

" 彩虹括号
Plug 'luochen1990/rainbow'

" 文件搜索
Plug 'ctrlpvim/ctrlp.vim'

" 缩进线
Plug 'Yggdroot/indentLine'

" 日志高亮
Plug 'MTDL9/vim-log-highlighting'

" Kubernetes
Plug 'andrewstuart/vim-kubernetes'

call plug#end()

" YAML缩进
autocmd FileType yaml setlocal ts=2 sts=2 sw=2 expandtab

" 大文件优化
let g:LargeFile=10
autocmd BufReadPre * let f=getfsize(expand('%'))
autocmd BufReadPre * if f > g:LargeFile*1024*1024 | set eventignore+=FileType | set syntax=OFF | set filetype=conf | endif
EOF

# 安装插件
vim +PlugInstall +qall
```

### 5.2 Nano 基础配置
```bash
cat > ~/.nanorc << EOF
include /usr/share/nano/*.nanorc
set tabsize 4
set tabstospaces
set autoindent
set linenumbers
EOF
```

---

## 六、安全审计工具

### 6.1 系统安全扫描
```bash
# Lynis 安全审计
yum install -y lynis            # CentOS (EPEL)
apt install -y lynis            # Ubuntu
lynis audit system --quick      # 快速扫描
lynis audit system --pentest    # 渗透测试模式

# chkrootkit/rkhunter
yum install -y rkhunter         # CentOS
apt install -y rkhunter chkrootkit  # Ubuntu
rkhunter --check --sk          # 检查rootkit

# ClamAV 杀毒
yum install -y clamav clamav-update  # CentOS
apt install -y clamav           # Ubuntu
freshclam                       # 更新病毒库
clamscan -r /var/www            # 扫描目录
```

### 6.2 入侵检测
```bash
# Fail2ban 防暴力破解
yum install -y fail2ban         # CentOS
apt install -y fail2ban         # Ubuntu

cat > /etc/fail2ban/jail.local << EOF
[sshd]
enabled = true
port = 22
filter = sshd
logpath = /var/log/secure      # Ubuntu为/var/log/auth.log
maxretry = 3
bantime = 3600
findtime = 600
EOF

systemctl enable --now fail2ban
fail2ban-client status sshd

# AIDE 文件完整性检查
yum install -y aide            # CentOS
apt install -y aide            # Ubuntu
aideinit                       # 初始化数据库
```

### 6.3 权限检查
```bash
# SUID/SGID文件（提权风险）
find / -perm -4000 -type f 2>/dev/null
find / -perm -2000 -type f 2>/dev/null

# 全局可写文件
find / -type f -perm -002 2>/dev/null

# 无属主文件
find / -nouser -o -nogroup 2>/dev/null
```

---

## 七、自动化运维平台

### 7.1 Ansible 快速部署
```bash
# 安装
pip3 install ansible --user

# 配置
cat > ~/.ansible.cfg << EOF
[defaults]
host_key_checking = False
inventory = ./hosts
remote_user = root
forks = 50
timeout = 30

[privilege_escalation]
become = True
become_method = sudo
become_user = root
EOF

# hosts清单示例
cat > hosts << EOF
[web]
web[01:10].example.com

[db]
db01.example.com
db02.example.com

[k8s]
k8s-master.example.com
k8s-node[01:05].example.com
EOF

# 常用模块
ansible all -m ping
ansible web -m shell -a "uptime"
ansible db -m yum -a "name=mysql state=present"      # CentOS
ansible db -m apt -a "name=mysql-server state=present"  # Ubuntu
```

### 7.2 SaltStack
```bash
# 安装
yum install -y salt-master salt-minion  # CentOS
apt install -y salt-master salt-minion  # Ubuntu

# 配置minion指向master
sed -i 's/#master: salt/master: salt-master.example.com/' /etc/salt/minion
systemctl enable --now salt-minion

# 接受key
salt-key -L                      # 列出待接受
salt-key -A                      # 接受全部

# 执行命令
salt '*' test.ping
salt '*' cmd.run 'uptime'
```

---

## 八、终端效率工具

### 8.1 tmux 终极配置
```bash
# 安装
yum install -y tmux             # CentOS
apt install -y tmux             # Ubuntu

# 配置（~/.tmux.conf）
cat > ~/.tmux.conf << EOF
# 修改前缀为Ctrl+A
unbind C-b
set -g prefix C-a
bind C-a send-prefix

# 鼠标支持
set -g mouse on

# 状态栏
set -g status-right "%Y-%m-%d %H:%M"
set -g status-interval 60

# 窗口/面板
bind c new-window -c "#{pane_current_path}"
bind % split-window -h -c "#{pane_current_path}"
bind '"' split-window -v -c "#{pane_current_path}"

# 重新加载
bind r source-file ~/.tmux.conf \; display "Reloaded!"
EOF

# 插件管理器
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# 常用操作
tmux new -s ops                 # 新建会话
tmux attach -t ops              # 连接
tmux ls                         # 列出
Ctrl+A d                        # 分离
Ctrl+A c                        # 新窗口
Ctrl+A %                        # 垂直分屏
Ctrl+A "                        # 水平分屏
```

### 8.2 fzf 模糊查找
```bash
# 安装
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install

# 使用示例
history | fzf
find . -type f | fzf
```

### 8.3 autojump 目录跳转
```bash
yum install -y autojump          # CentOS (EPEL)
apt install -y autojump          # Ubuntu
# 使用: j myproject  # 自动匹配跳转
```

---

## 九、云原生工具集

### 9.1 Docker 全家桶
```bash
# 安装
curl -fsSL https://get.docker.com | bash

# 配置镜像加速
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

# 常用工具
docker stats                     # 容器资源
docker system df                 # 磁盘使用
docker exec -it container_id bash

# dive 镜像分析
curl -L https://github.com/wagoodman/dive/releases/download/v0.9.2/dive_0.9.2_linux_amd64.rpm -o dive.rpm && yum install -y dive
# Ubuntu用 https://github.com/wagoodman/dive/releases/download/v0.9.2/dive_0.9.2_linux_amd64.deb
dive nginx:latest
```

### 9.2 Kubernetes 工具
```bash
# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && mv kubectl /usr/local/bin/

# k9s 终端UI
curl -sS https://webinstall.dev/k9s | bash

# kubectx/kubens
curl -L https://raw.githubusercontent.com/ahmetb/kubectx/master/kubectx -o /usr/local/bin/kubectx
curl -L https://raw.githubusercontent.com/ahmetb/kubectx/master/kubens -o /usr/local/bin/kubens
chmod +x /usr/local/bin/kubectx /usr/local/bin/kubens

# Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 别名
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias ke='kubectl exec -it'
alias klog='kubectl logs -f'
```

---

## 十、一键安装脚本

### 10.1 CentOS 运维全家桶
```bash
cat > install_ops_tools_centos.sh << 'EOF'
#!/bin/bash
set -e

echo ">>> 安装EPEL源"
yum install -y epel-release

echo ">>> 安装基础工具"
yum install -y vim htop iotop iftop nethogs glances git telnet net-tools bind-utils traceroute mtr nc tcpdump nmap lsof strace ltrace sysstat dstat bash-completion chrony unzip zip psmisc screen tmux expect pv jq python3-pip ncdu tree smartmontools ipmitool dmidecode

echo ">>> 安装网络工具"
yum install -y nethogs iftop mtr nmap tcpdump

echo ">>> 安装监控工具"
yum install -y htop iotop glances dstat

echo ">>> 安装日志工具"
yum install -y lnav multitail

echo ">>> 安装Docker"
curl -fsSL https://get.docker.com | bash
systemctl enable --now docker

echo ">>> 安装Python工具"
pip3 install --user mycli pgcli ansible yq thefuck

echo ">>> 安装Tmux插件管理器"
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

echo ">>> 安装Vim插件管理器"
curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim

echo ">>> 安装fzf"
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install

echo ">>> 安装neofetch"
yum install -y neofetch

echo "完成！请手动运行 vim +PlugInstall 和 tmux插件安装(Ctrl+A I)"
EOF

chmod +x install_ops_tools_centos.sh
```

### 10.2 Ubuntu 运维全家桶
```bash
cat > install_ops_tools_ubuntu.sh << 'EOF'
#!/bin/bash
set -e

echo ">>> 更新源"
apt update

echo ">>> 安装基础工具"
apt install -y vim htop iotop iftop nethogs glances git telnet net-tools dnsutils traceroute mtr netcat-openbsd tcpdump nmap lsof strace ltrace sysstat dstat bash-completion chrony unzip zip psmisc screen tmux expect pv jq python3-pip ncdu tree smartmontools ipmitool dmidecode

echo ">>> 安装网络工具"
apt install -y nethogs iftop mtr nmap tcpdump

echo ">>> 安装监控工具"
apt install -y htop iotop glances dstat

echo ">>> 安装日志工具"
apt install -y lnav multitail

echo ">>> 安装Docker"
curl -fsSL https://get.docker.com | bash
systemctl enable --now docker

echo ">>> 安装Python工具"
pip3 install --user mycli pgcli ansible yq thefuck

echo ">>> 安装Tmux插件管理器"
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

echo ">>> 安装Vim插件管理器"
curl -fLo ~/.vim/autoload/plug.vim --create-dirs https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim

echo ">>> 安装fzf"
git clone --depth 1 https://github.com/junegunn/fzf.git ~/.fzf
~/.fzf/install

echo ">>> 安装neofetch"
apt install -y neofetch

echo "完成！请手动运行 vim +PlugInstall 和 tmux插件安装(Ctrl+A I)"
EOF

chmod +x install_ops_tools_ubuntu.sh
```

---

## 十一、运维别名库（~/.bashrc）

```bash
cat >> ~/.bashrc << 'EOF'
# ==================== 运维别名库 ====================

# 系统信息
alias sysinfo='echo "--- CPU ---"; lscpu | grep "Model name"; echo "--- Memory ---"; free -h; echo "--- Disk ---"; df -hT | grep -v tmpfs; echo "--- Network ---"; ip a | grep "inet "; echo "--- Load ---"; uptime'
alias ports='ss -tuln | grep LISTEN'
alias bigfiles='find / -type f -size +100M -exec ls -lh {} \; 2>/dev/null | head -20'
alias iplist='ip a | grep "inet " | awk \'{print $2}\''

# 包管理
alias yup='yum update -y'
alias yin='yum install -y'
alias yrm='yum remove -y'
alias ysearch='yum search'
alias aup='apt update && apt upgrade -y'
alias ain='apt install -y'
alias arm='apt remove -y'
alias asearch='apt search'

# Docker
alias dps='docker ps --format "table {{.ID}}\t{{.Names}}\t{{.Status}}\t{{.Ports}}"'
alias dsa='docker stop $(docker ps -aq)'
alias drma='docker rm $(docker ps -aq)'
alias dcl='docker system prune -a'

# K8s
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias ke='kubectl exec -it'
alias klog='kubectl logs -f'
alias kdel='kubectl delete'

# 日志
alias tailf='tail -f'
alias grepall='grep -r --include="*.log" --include="*.txt"'
alias journalerr='journalctl -p err -f'
alias journalboot='journalctl -b'

# 网络
alias nettest='ping 114.114.114.114 -c 3 && ping 8.8.8.8 -c 3 && ping baidu.com -c 3'
alias ports80='ss -ant | grep :80 | wc -l'
alias ports443='ss -ant | grep :443 | wc -l'

# 安全
alias ssh20='ssh -o ServerAliveInterval=60 -o TCPKeepAlive=yes'
alias sshproxy='ssh -D 1080 -C -N'

# 颜色
alias ls='ls --color=auto'
alias ll='ls -lhF'
alias grep='grep --color=auto'
alias ip='ip -c'               # Ubuntu

# 历史记录增强
export HISTTIMEFORMAT="%Y-%m-%d %H:%M:%S "
export HISTSIZE=10000
export HISTFILESIZE=10000
shopt -s histappend
export PROMPT_COMMAND="history -a; history -c; history -r; $PROMPT_COMMAND"

# 路径
export PATH=$PATH:/usr/local/bin:/usr/local/sbin:$HOME/.local/bin

# 安全删除
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'

# 解压万能函数
extract() {
    if [ -f $1 ]; then
        case $1 in
            *.tar.bz2) tar xjf $1 ;;
            *.tar.gz)  tar xzf $1 ;;
            *.bz2)     bunzip2 $1 ;;
            *.rar)     unrar x $1 ;;
            *.gz)      gunzip $1 ;;
            *.tar)     tar xf $1 ;;
            *.tbz2)    tar xjf $1 ;;
            *.tgz)     tar xzf $1 ;;
            *.zip)     unzip $1 ;;
            *.Z)       uncompress $1 ;;
            *.7z)      7z x $1 ;;
            *)         echo "'$1' cannot be extracted via extract()" ;;
        esac
    else
        echo "'$1' is not a valid file"
    fi
}

# 快速SSH key分发
ssh-copy-id() {
    if [ -z "$1" ]; then
        echo "Usage: ssh-copy-id user@host"
        return 1
    fi
    cat ~/.ssh/id_rsa.pub | ssh $1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
}

# 备份配置
backup_cfg() {
    local dest="/tmp/config_backup_$(date +%Y%m%d_%H%M%S).tar.gz"
    tar -czf $dest /etc/nginx /etc/httpd /etc/mysql /etc/redis /etc/sysconfig 2>/dev/null
    echo "配置已备份到: $dest"
}

# 加载配置
source ~/.bashrc
EOF
```

---

## 十二、应急响应手册

### 12.1 系统被入侵应急
```bash
#!/bin/bash
# incident_response.sh

echo ">>> 1. 隔离网络（如必要）"
# iptables -A OUTPUT -j DROP

echo ">>> 2. 保存现场"
mkdir -p /tmp/incident_$(date +%Y%m%d)
cd /tmp/incident_$(date +%Y%m%d)

# 保存进程信息
ps aux > ps.aux.txt
lsof > lsof.txt

# 保存网络连接
ss -antp > ss.antp.txt
netstat -tulpn > netstat.tulpn.txt

# 保存用户信息
cat /etc/passwd > passwd.txt
cat /etc/shadow > shadow.txt
last > last.txt

# 保存日志
cp /var/log/secure . 2>/dev/null
cp /var/log/auth.log . 2>/dev/null
cp /var/log/nginx/access.log . 2>/dev/null

echo ">>> 3. 查找可疑文件"
find /tmp /var/tmp -type f -executable -exec ls -lh {} \; > suspicious_files.txt

echo ">>> 4. 查找可疑进程"
ps aux | awk '$3 > 50 {print $0}' > high_cpu.txt
ps aux | awk '$4 > 10 {print $0}' > high_mem.txt

echo ">>> 5. 查找可疑连接"
ss -antp | grep -v '127.0.0.1' > suspicious_conn.txt

echo ">>> 数据已保存到: /tmp/incident_$(date +%Y%m%d)"
```

### 12.2 DDoS攻击应对
```bash
# 查看SYN洪水
netstat -n | awk '/^tcp/ {++state[$6]} END {for(key in state) print key,state[key]}'

# 限制SYN
iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT
iptables -A INPUT -p tcp --syn -j DROP

# 封禁IP（使用firewalld）
firewall-cmd --permanent --add-rich-rule='rule source address="192.168.1.100" reject'
firewall-cmd --reload

# 封禁IP（使用ufw）
ufw deny from 192.168.1.100

# CloudFlare模式（获取真实IP）
# 在nginx中配置: set_real_ip_from 103.21.244.0/22;
```

---

## 十三、日常巡检脚本

```bash
cat > daily_check.sh << 'EOF'
#!/bin/bash
REPORT="/tmp/daily_report_$(hostname)_$(date +%Y%m%d).txt"

echo "===== 主机: $(hostname) | 时间: $(date) =====" > $REPORT
echo "" >> $REPORT

echo "--- CPU负载 ---" >> $REPORT
uptime >> $REPORT
echo "" >> $REPORT

echo "--- 内存使用 ---" >> $REPORT
free -h >> $REPORT
echo "" >> $REPORT

echo "--- 磁盘使用 ---" >> $REPORT
df -hT >> $REPORT
echo "" >> $REPORT

echo "--- 磁盘IO ---" >> $REPORT
iostat -x 1 5 >> $REPORT
echo "" >> $REPORT

echo "--- 网络连接 ---" >> $REPORT
ss -ant | grep ESTAB | wc -l >> $REPORT
echo "" >> $REPORT

echo "--- 监听端口 ---" >> $REPORT
ss -tuln | grep LISTEN >> $REPORT
echo "" >> $REPORT

echo "--- 登录失败 ---" >> $REPORT
grep "Failed password" /var/log/secure 2>/dev/null | tail -20 >> $REPORT
grep "Failed password" /var/log/auth.log 2>/dev/null | tail -20 >> $REPORT
echo "" >> $REPORT

echo "--- 僵尸进程 ---" >> $REPORT
ps aux | awk '$8~/Z/ {print $0}' >> $REPORT
echo "" >> $REPORT

echo "--- 服务状态 ---" >> $REPORT
systemctl list-units --state=failed --no-pager >> $REPORT
echo "" >> $REPORT

echo "--- 最近变更 ---" >> $REPORT
find /etc /usr/local/bin -type f -mtime -1 2>/dev/null | head -20 >> $REPORT

mail -s "Daily Check Report: $(hostname)" ops@company.com < $REPORT
echo "报告已发送到邮箱"
EOF

chmod +x daily_check.sh
# 加入crontab: 0 8 * * * /path/to/daily_check.sh
```

---

## 🎯 使用建议

1. **新服务器初始化**：运行对应系统的一键安装脚本
2. **个人环境配置**：复制所有配置文件（`.vimrc`, `.bashrc`, `.tmux.conf`）
3. **日常巡检**：部署daily_check.sh到所有服务器
4. **应急响应**：准备好incident_response.sh，随时可用
5. **持续学习**：每周尝试一个新工具，逐步替换低效操作

**完整配置已就绪，直接复制使用即可！祝运维工作顺利！**