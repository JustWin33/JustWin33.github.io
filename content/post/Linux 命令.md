---
title: Linux命令
description: 
date: 2025-11-08
categories:
    - 
    - 
---

# Linux 命令

作为运维工程师，我为您深度挖掘全网权威资料，整理出这份**涵盖基础到内核级调试**的完整命令手册。每个命令都包含**生产环境实战技巧**和**避坑指南**。

---

## 一、基础命令深度解析（运维视角）

### 1.1 文件目录操作（含性能优化）

#### `ls` - 目录列表
```bash
# 运维必用组合：按大小排序，显示隐藏文件，人性化单位
ls -lhS --color=auto  # 按大小排序，人类可读
ls -lart              # 按修改时间倒序（最新在最后）
ls -i                 # 显示inode号（排查硬链接问题）
ls -Z                 # 显示SELinux上下文

# 生产环境技巧：快速找出大目录
du -h --max-depth=1 /var | sort -hr | head -10
```

#### `cd` - 切换目录
```bash
cd -                  # 快速返回上次目录（运维高频操作）
cd ..                 # 返回上级
cd ~                  # 返回home目录
cd /                  # 返回根目录

# 高级技巧：自动纠正拼写错误
shopt -s cdspell      # 开启cd命令自动纠错
```

#### `pwd` - 显示当前路径
```bash
pwd -P                # 显示物理路径（解析符号链接）
pwd -L                # 显示逻辑路径（默认）
```

#### `mkdir` - 创建目录
```bash
mkdir -p /path/to/dir # 递归创建（-p = parents）
mkdir -m 755 /path    # 直接指定权限
mkdir -v /path        # 显示创建过程
```

#### `rm` - 删除文件/目录（⚠️高危命令）
```bash
# 生产环境黄金法则：先ls确认，再rm删除
ls /path/to/delete && rm -rf /path/to/delete

# 安全删除：使用trash-cli替代rm
# rm -rf 的危险性：删除后无法恢复，操作前务必确认

# 运维技巧：限制rm别名防止误删
alias rm='rm -i'      # 删除前提示确认
alias rm='rm --preserve-root'  # 防止删除根目录
```

#### `cp` - 复制文件
```bash
cp -a source dest     # -a = -dR --preserve=all，保留所有属性（备份首选）
cp -r dir1 dir2       # 递归复制目录
cp -u source dest     # 只复制更新的文件（update）
cp -p source dest     # 保留文件属性（时间戳、权限等）
cp --sparse=always    # 优化稀疏文件复制（虚拟机镜像）
```

#### `mv` - 移动/重命名文件
```bash
mv -i file1 file2     # 交互式，覆盖前提示
mv -n file1 file2     # 不覆盖已存在文件
mv -t target_dir source_files  # 指定目标目录
```

#### `touch` - 修改时间戳
```bash
touch file            # 不存在则创建，存在则更新时间为当前
touch -c file         # 不创建新文件
touch -t 202501011200 file  # 指定时间戳
touch -r ref_file file  # 使用参考文件的时间
```

---

### 1.2 文件查看与编辑

#### `cat` - 连接文件并打印
```bash
cat -n file           # 显示行号
cat -A file           # 显示所有控制字符（包括换行符$）
cat file1 file2 > file3  # 合并文件
```

#### `more` - 分页查看
```bash
more +100 file        # 从第100行开始显示
more -d file          # 显示帮助提示
```

#### `less` - 增强分页查看（运维首选）
```bash
less -N file          # 显示行号
less +F file          # 实时跟踪文件更新（类似tail -f）
less -S file          # 不换行显示（查看宽日志）
# 内部命令：
# /pattern 搜索  n/N 下一个/上一个
# g 跳到行首  G 跳到行尾  q 退出
```

#### `head` - 查看文件头部
```bash
head -n 20 file       # 显示前20行
head -c 100 file      # 显示前100字节
head -n -5 file       # 显示除最后5行的所有内容
```

#### `tail` - 查看文件尾部（运维高频）
```bash
tail -f file          # 实时跟踪文件更新（日志监控）
tail -F file          # 跟踪文件，即使文件被旋转也继续
tail -n 100 file      # 显示最后100行
tail -c 100 file      # 显示最后100字节
tail -n +100 file     # 从第100行开始显示

# 生产技巧：同时查看多个日志
tail -f /var/log/*.log
```

#### `vim` - 文本编辑器（运维必备）
```bash
# 常用操作
vim +100 file         # 打开文件并定位到第100行
vim -R file           # 只读模式打开

# 内部命令
# :set nu             显示行号
# :set nonu           取消行号
# /pattern            搜索
# n                   下一个匹配
# :%s/old/new/g       全局替换
# :wq                 保存退出
# :q!                 强制退出不保存
```

---

## 二、系统监控与性能分析（生产级）

### 2.1 CPU性能监控

#### `top` - 实时监控系统
```bash
top                   # 默认按CPU排序
top -Hp <PID>         # 查看进程内所有线程（排查Java/Python多线程问题关键）
top -p 1,2,3          # 只监控指定PID
top -u username       # 只显示指定用户
top -b -n 1 > top.log # 批处理模式，运行一次后退出

# 内部交互命令
# P - 按CPU排序  M - 按内存排序  T - 按时间排序
# k - 杀死进程  r - 修改nice值  H - 显示线程
# 1 - 显示各CPU核心详情

# 生产案例：Java应用CPU飙高排查
# 1. top找到高CPU进程PID
# 2. top -Hp PID查看线程
# 3. printf "%xn" <TID> 转16进制
# 4. jstack PID | grep -A 20 <hex_TID>
```

#### `htop` - 增强版top（推荐）
```bash
htop                  # 彩色显示，支持鼠标操作
htop -d 10            # 刷新间隔10秒
htop -u root          # 只显示root用户进程

# 优点：支持树状显示、直接kill、renice、搜索过滤
```

#### `vmstat` - 系统资源综合监控
```bash
vmstat 2 10           # 每2秒采样一次，共10次
vmstat -s             # 显示内存统计摘要
vmstat -d             # 显示磁盘统计

# 输出解读：
# procs: r(运行队列) b(不可中断睡眠进程)
# memory: swpd(交换内存) free(空闲) buff/cache(缓存)
# swap: si(换入) so(换出) - 这两个值应该为0
# io: bi(读块) bo(写块)
# cpu: us(用户) sy(系统) id(空闲) wa(IO等待)

# 警戒线：wa持续>20%说明IO瓶颈
```

#### `mpstat` - 多核CPU分析
```bash
mpstat -P ALL 1 10    # 显示每个CPU核心的使用情况
mpstat 1 10           # 显示所有CPU平均情况

# 输出列：%usr %nice %sys %iowait %irq %soft %steal %guest %gnice %idle
```

#### `pidstat` - 进程级资源监控（sysstat工具）
```bash
pidstat -u 1 10       # 监控CPU使用
pidstat -r 1 10       # 监控内存使用
pidstat -d 1 10       # 监控磁盘IO
pidstat -t -p <PID>   # 监控指定进程的线程

# 生产技巧：监控Java应用所有指标
pidstat -u -r -d -t -p <PID> 2
```

#### `sar` - 系统活动报告（历史数据）
```bash
sar -u 1 10           # CPU使用情况
sar -r 1 10           # 内存使用情况
sar -b 1 10           # IO传输速率
sar -n DEV 1 10       # 网络设备流量
sar -q 1 10           # 负载队列
sar -f /var/log/sa/saXX  # 查看历史数据

# 安装：yum install sysstat / apt-get install sysstat
```

#### `perf` - 性能分析利器（内核级）
```bash
perf top              # 实时分析系统性能热点
perf stat -p <PID>    # 统计进程性能数据
perf record -g -p <PID>  # 记录性能数据（用于火焰图）
perf report           # 报告分析结果

# 生产案例：分析CPU热点函数
perf top -g -p <PID>
# 定位到具体函数后，可结合jstack/gdb分析
```

#### `strace` - 系统调用跟踪（调试神器）
```bash
strace -p <PID>       # 跟踪指定进程
strace -c -p <PID>    # 统计系统调用耗时
strace -e trace=file -p <PID>  # 只跟踪文件操作
strace -f -e trace=network -p <PID> # 跟踪子进程和网络调用
strace -T -tt -p <PID> # 显示时间戳和耗时

# 生产案例：进程卡住排查
strace -p <PID> -e trace=futex
# 发现卡在futex_wait，说明在等待锁
```

---

### 2.2 内存性能监控

#### `free` - 内存使用
```bash
free -h               # 人类可读格式
free -m               # MB单位
free -w               # 显示 buff 和 cache 分开

# 解读：
# available: 真正可用内存（最重要指标）
# free: 完全空闲内存
# buff/cache: 缓存（可被回收）

# 生产误区：不要被free值迷惑，关键看available
```

#### `pmap` - 进程内存映射
```bash
pmap -x <PID>         # 详细内存映射
pmap -x <PID> | tail -5  # 只显示汇总

# 排查内存泄漏：持续观察RSS列
watch -n 5 "pmap -x <PID> | grep total"
```

#### `smem` - 内存统计工具
```bash
smem -rs swap -p      # 按swap使用排序（%）
smem -rs rss -p       # 按物理内存排序
smem -u               # 按用户汇总

# 优势：能准确反映PSS（比例集大小），比RSS更有意义
```

#### `vmstat -s` - 内存统计
```bash
vmstat -s             # 显示详细内存统计
# 重点看：swap pages in/out，应该为0
```

---

### 2.3 磁盘IO性能监控

#### `iostat` - IO统计（sysstat）
```bash
iostat -x 1 10        # 详细IO统计（-x显示扩展信息）
iostat -dk 1 10       # 以KB/s显示
iostat -p sda 1 10    # 显示分区详细

# 关键指标：
# %util: 设备利用率，>80%说明瓶颈
# await: 平均IO响应时间，>10ms需关注
# svctm: 平均服务时间
# r/s w/s: 读写IOPS
```

#### `iotop` - IO实时监控
```bash
iotop                 # 需要root权限
iotop -oP -d 2        # 只显示有IO的进程(-o)，只显示进程(-P)，2秒刷新
iotop -a              # 显示累积IO量
```

#### `df` - 磁盘空间
```bash
df -h                 # 人类可读
df -i                 # 显示inode使用情况（inode满导致无法写入）
df -T                 # 显示文件系统类型
df -h /path           # 只显示指定路径的挂载点

# 生产警戒线：使用率>85%预警，>95%紧急
```

#### `du` - 目录大小统计
```bash
du -h --max-depth=1 /path  # 统计一级子目录大小
du -sh /path              # 只显示总大小(-s)
du -ah /path | sort -hr | head -20  # 找出最大的20个文件

# 生产技巧：快速定位磁盘满的原因
du -h --max-depth=1 / | sort -hr | head -10
```

#### `lsof` - 查看打开的文件
```bash
lsof                  # 列出所有打开的文件
lsof -p <PID>         # 指定进程打开的文件
lsof /path/to/file    # 查看文件被哪些进程使用
lsof -i :8080         # 查看端口占用进程
lsof -u username      # 指定用户
lsof +D /path         # 递归目录（慢，用lsof | grep /path更快）

# 生产案例：删除文件后空间不释放
lsof | grep deleted   # 查找已删除但仍被进程占用的文件
# 解决：重启对应进程或> /proc/<PID>/fd/<FD>
```

---

### 2.4 网络性能监控

#### `ss` - socket统计（替代netstat）
```bash
ss -tuln             # 显示所有监听端口（-t TCP, -u UDP, -l 监听, -n 数字）
ss -tan              # 显示所有TCP连接
ss -s                # 显示socket统计摘要
ss -p                # 显示进程信息
ss -o state established '( dport = :22 or sport = :22 )'  # 过滤特定端口

# 比netstat更快，适合高并发环境
```

#### `netstat` - 网络连接（传统工具）
```bash
netstat -tuln        # 显示监听端口
netstat -anp         # 显示所有连接和进程
netstat -i           # 显示网络接口统计
netstat -r           # 显示路由表
netstat -s           # 显示协议统计
```

#### `iftop` - 实时流量监控
```bash
iftop -i eth0        # 监控指定网卡
iftop -n             # 不解析主机名
iftop -P             # 显示端口
iftop -f "port 80"   # 过滤特定端口

# 需要root权限，适合排查带宽占用
```

#### `tcpdump` - 抓包分析（网络排障核心）
```bash
tcpdump -i eth0 -nn -s0 -v port 80  # 抓HTTP流量
tcpdump -i any -nn host 192.168.1.1  # 抓特定IP
tcpdump -w capture.pcap              # 写入文件（wireshark分析）
tcpdump -r capture.pcap              # 读取文件
tcpdump -A -s0 port 80               # ASCII显示HTTP内容
tcpdump -i eth0 -c 1000              # 只抓1000个包

# 生产案例：排查MySQL慢查询网络原因
tcpdump -i eth0 -s0 -nn -A dst port 3306 | grep -E "SELECT|INSERT"
```

#### `nethogs` - 按进程统计流量
```bash
nethogs              # 需要root，显示进程级流量
nethogs eth0         # 指定网卡
```

#### `ping` - 测试连通性
```bash
ping -c 4 -i 0.2 -W 1 8.8.8.8  # 发送4次，间隔0.2秒，超时1秒
ping -f 8.8.8.8       # 洪水ping（需root，测试网络极限）
ping -s 1400 8.8.8.8  # 指定包大小（测试MTU）
```

#### `traceroute` - 路由追踪
```bash
traceroute -n 8.8.8.8  # 不解析主机名（快）
traceroute -T -p 80 8.8.8.8  # TCP SYN方式（绕过防火墙ICMP限制）
traceroute -I 8.8.8.8  # ICMP方式
```

#### `mtr` - 综合网络诊断
```bash
mtr -r -c 10 8.8.8.8  # 报告模式，发送10次
mtr -w 8.8.8.8        # 宽格式（长主机名）
```

---

### 2.5 系统负载与运行时长

#### `uptime` - 系统运行时间与负载
```bash
uptime               # 显示：时间、运行时长、用户数、负载
uptime -p            # 只显示运行时长

# 负载解读：
# 1分钟、5分钟、15分钟平均负载
# 理想值：负载 < CPU核心数
# 警戒线：负载 > CPU核心数 * 0.7
```

#### `who` - 查看登录用户
```bash
who                  # 显示当前登录用户
who -b               # 显示上次启动时间
who -r               # 显示运行级别
```

#### `last` - 查看登录历史
```bash
last                 # 显示所有登录历史
last root            # 只显示root登录历史
last -n 10           # 只显示最近10条
last -f /var/log/btmp  # 查看失败登录尝试
```

#### `dmesg` - 内核日志
```bash
dmesg                # 查看内核环形缓冲区
dmesg -T             # 显示人类可读时间
dmesg -c             # 读取后清除
dmesg | grep -i error  # 搜索错误
dmesg | grep -i oom    # 查找OOM killer记录

# 生产案例：排查硬件故障
dmesg | grep -E " Hardware|Error|Fail"
```

---

## 三、进程管理（运维核心）

### 3.1 进程查看

#### `ps` - 进程快照
```bash
ps aux               # BSD风格，显示所有进程（常用）
ps -ef               # System V风格
ps aux --sort=-%cpu  # 按CPU排序（降序）
ps aux --sort=-%mem  # 按内存排序
ps -o pid,ppid,cmd,%cpu,%mem -p <PID>  # 自定义输出列

# 生产技巧：查找僵尸进程
ps aux | awk '$8=="Z"'
```

#### `pgrep` - 按名称查找PID
```bash
pgrep nginx          # 查找nginx进程PID
pgrep -f "python3 app.py"  # 匹配完整命令行
pgrep -u root sshd   # 查找root用户的sshd进程
pgrep -d, -f java    # 用逗号分隔输出（用于top -p）
```

#### `pidof` - 按名称查找PID
```bash
pidof nginx          # 返回所有nginx worker PID
```

#### `pstree` - 进程树
```bash
pstree               # 显示进程树
pstree -p            # 显示PID
pstree -u            # 显示用户
pstree -a            # 显示完整命令
pstree | grep -A 5 nginx  # 查找nginx相关进程树
```

---

### 3.2 进程控制

#### `kill` - 发送信号
```bash
kill -l              # 列出所有信号
kill -15 <PID>       # 发送TERM信号（优雅终止）
kill -9 <PID>        # 发送KILL信号（强制终止，谨慎使用）
kill -2 <PID>        # 发送INT信号（类似Ctrl+C）
kill -1 <PID>        # 发送HUP信号（重载配置）

# 批量杀进程
pkill -9 java        # 按名称杀进程
killall -9 java      # 杀所有名为java的进程
kill -9 $(pgrep -f "test.py")  # 组合使用

# 生产案例：优雅重启nginx
kill -USR1 $(cat /var/run/nginx.pid)  # 重新打开日志文件
kill -HUP $(cat /var/run/nginx.pid)   # 重载配置
```

#### `killall` - 按名称杀进程
```bash
killall -9 python3   # 杀所有python3进程
killall -w process   # 等待进程终止
```

#### `pkill` - 按模式杀进程
```bash
pkill -f "test.py"   # 匹配完整命令行
pkill -u username    # 杀死某用户的所有进程
pkill -t pts/0       # 杀死某个终端的所有进程
```

#### `nice`/`renice` - 调整优先级
```bash
nice -n 19 command   # 以最低优先级运行（19最低，-20最高）
renice -n 10 -p <PID>  # 调整已运行进程优先级

# 生产技巧：降低备份任务优先级
nice -n 19 tar -czf backup.tar.gz /data
```

#### `nohup` - 后台持久运行
```bash
nohup command &       # &放入后台，nohup忽略挂起信号
nohup java -jar app.jar > app.log 2>&1 &  # 标准推荐写法
```

#### `disown` - 从shell移除作业
```bash
disown %1             # 移除作业1
disown -a             # 移除所有作业
disown -h %1          # 标记为不接收SIGHUP
```

---

### 3.3 作业控制

#### `&` - 后台运行
```bash
command &            # 放入后台执行
```

#### `jobs` - 查看后台作业
```bash
jobs                 # 查看当前shell的后台作业
jobs -l              # 显示PID
jobs -r              # 只显示运行中的作业
jobs -s              # 只显示停止的作业
```

#### `fg`/`bg` - 前台/后台切换
```bash
fg %1                # 将作业1调到前台
bg %1                # 将作业1在后台继续运行
```

#### `wait` - 等待作业完成
```bash
wait <PID>           # 等待指定PID进程结束
wait                 # 等待所有后台作业结束
```

---

### 3.4 高级进程调试

#### `strace` - 系统调用跟踪
```bash
# 详见2.1节
```

#### `ltrace` - 库函数调用跟踪
```bash
ltrace -p <PID>      # 跟踪库函数调用
ltrace -c -p <PID>   # 统计调用耗时
```

#### `gdb` - 调试器
```bash
gdb -p <PID>         # 附加到运行进程
gdb /path/to/binary core  # 分析core dump

# 生产案例：分析Java core文件
gdb java core.12345
(gdb) bt              # 查看堆栈
(gdb) info threads    # 查看线程
(gdb) thread apply all bt  # 查看所有线程堆栈
```

#### `perf` - 性能分析
```bash
# 详见2.1节
```

---

## 四、网络管理（运维核心）

### 4.1 网络配置

#### `ip` - 现代网络配置（推荐）
```bash
ip addr               # 显示IP地址（替代ifconfig）
ip addr add 192.168.1.100/24 dev eth0  # 添加IP
ip addr del 192.168.1.100/24 dev eth0  # 删除IP
ip link set eth0 up   # 启用网卡
ip link set eth0 down # 禁用网卡
ip route              # 显示路由表
ip route add default via 192.168.1.1  # 添加默认网关
ip neighbor           # 显示ARP表
ip -s link            # 显示统计信息

# 生产技巧：快速查看IP
ip -4 -o addr show scope global | awk '{print $4}'
```

#### `ifconfig` - 传统网络配置
```bash
ifconfig              # 显示所有网卡
ifconfig eth0         # 显示指定网卡
ifconfig eth0 up/down # 启用/禁用
ifconfig eth0 192.168.1.100 netmask 255.255.255.0  # 配置IP

# 已废弃，推荐使用ip命令
```

#### `route` - 路由管理
```bash
route -n              # 显示路由表
route add default gw 192.168.1.1  # 添加默认网关
route add -net 10.0.0.0/8 gw 10.0.0.1  # 添加静态路由
route del -net 10.0.0.0/8  # 删除路由
```

#### `nmcli` - NetworkManager命令行
```bash
nmcli device status   # 查看设备状态
nmcli connection show # 查看连接
nmcli connection up/down id "conn-name"  # 启用/禁用连接
nmcli device wifi list  # 查看WiFi
```

---

### 4.2 网络诊断

#### `ping` - 连通性测试
```bash
# 详见2.4节
```

#### `traceroute` - 路由追踪
```bash
# 详见2.4节
```

#### `mtr` - 综合诊断
```bash
# 详见2.4节
```

#### `nslookup`/`dig` - DNS查询
```bash
dig www.baidu.com     # 详细DNS查询
dig +short www.baidu.com  # 简短输出
dig @8.8.8.8 www.baidu.com  # 指定DNS服务器
dig +trace www.baidu.com  # 追踪查询过程
dig -x 8.8.8.8        # 反向解析

nslookup www.baidu.com 8.8.8.8  # 指定DNS
```

#### `host` - 简单DNS查询
```bash
host www.baidu.com
host -t mx baidu.com  # 查询MX记录
```

#### `telnet` - 测试端口连通性
```bash
telnet 192.168.1.1 22  # 测试SSH端口
# Ctrl+] 然后输入quit退出
```

#### `nc` (netcat) - 网络瑞士军刀
```bash
nc -zv 192.168.1.1 22  # 测试端口（-z 零IO，-v 详细）
nc -l 8080           # 监听8080端口
nc 192.168.1.1 8080  # 连接端口
echo "test" | nc 192.168.1.1 8080  # 发送数据

# 生产技巧：快速传输文件
# 接收端：nc -l 8888 > file
# 发送端：nc receiver_ip 8888 < file
```

---

### 4.3 高级网络工具

#### `tcpdump` - 抓包分析
```bash
# 详见2.4节
```

#### `wireshark` - 图形化抓包
```bash
wireshark            # 图形界面
tshark -i eth0       # 命令行版wireshark
```

#### `ss` - socket统计（推荐）
```bash
# 详见4.1节
```

#### `lsof` - 查看网络连接
```bash
lsof -i :8080        # 查看8080端口占用
lsof -i TCP          # 查看所有TCP连接
lsof -i @192.168.1.1  # 查看连接到该IP的连接
```

---

## 五、文本处理三剑客（运维核心）

### 5.1 `grep` - 文本搜索

```bash
# 基本用法
grep "error" file.log  # 搜索包含error的行
grep -i "error" file   # 忽略大小写
grep -v "error" file   # 反向匹配（不包含）
grep -r "error" /path  # 递归目录搜索
grep -E "error|warn" file  # 正则表达式（扩展）
grep -A 3 -B 3 "error" file  # 显示前后3行
grep -n "error" file   # 显示行号
grep -c "error" file   # 统计匹配行数
grep -o "error" file   # 只显示匹配部分
grep -w "error" file   # 整词匹配
grep -f patterns.txt file  # 从文件读取模式

# 生产案例：分析Nginx访问日志
grep "200" access.log | wc -l  # 统计成功请求
grep -E "5[0-9]{2}" access.log | awk '{print $7}' | sort | uniq -c | sort -nr | head  # Top错误URL

# 高级用法：排除多个模式
grep -vE "(debug|info)" app.log

# 查找IP地址
grep -oE "([0-9]{1,3}\.){3}[0-9]{1,3}" access.log
```

---

### 5.2 `sed` - 流编辑器

```bash
# 基本语法：sed [选项] '命令' 文件

# 替换
sed 's/old/new/' file  # 每行第一个匹配
sed 's/old/new/g' file  # 全局替换
sed -i 's/old/new/g' file  # 直接修改文件（-i危险，先备份）

# 删除
sed '/pattern/d' file  # 删除匹配行
sed '1,10d' file       # 删除1-10行
sed '$d' file          # 删除最后一行

# 插入
sed '1i\insert' file   # 第1行前插入
sed '$a\append' file   # 最后追加

# 打印
sed -n '1,10p' file    # 打印1-10行
sed -n '/pattern/p' file  # 打印匹配行

# 生产案例：批量修改配置文件
sed -i 's/Port 22/Port 2222/' /etc/ssh/sshd_config  # 修改SSH端口
sed -i '/^#ServerActive/s/#//' /etc/zabbix/zabbix_agentd.conf  # 取消注释

# 提取特定行
sed -n '/2024-01-01/,/2024-01-02/p' access.log  # 提取日期范围
```

---

### 5.3 `awk` - 文本处理语言

```bash
# 基本语法：awk '模式{动作}' 文件

# 打印列
awk '{print $1}' file  # 打印第1列（默认空格分隔）
awk -F: '{print $1}' /etc/passwd  # 指定分隔符:
awk '{print $NF}' file  # 打印最后一列

# 条件过滤
awk '$3 > 100' file    # 第3列大于100
awk '/error/' file     # 匹配模式
awk '$1 ~ /^192/ {print $2}' file  # 第1列匹配正则

# 生产案例：分析Nginx日志
awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -20  # Top IP
awk '{print $7}' access.log | grep -v '^$' | sort | uniq -c | sort -nr | head -20  # Top URL
awk '$9 ~ /^5/ {print $7}' access.log | sort | uniq -c | sort -nr  # 500错误的URL

# 统计
awk '{sum+=$1} END {print sum/NR}' file  # 求平均值
awk '{if($1>max) max=$1} END {print max}' file  # 求最大值

# 多分隔符
awk -F'[: ]' '{print $1}' file  # 分隔符为冒号或空格

# 内置变量
# NR: 行号  NF: 列数  $0: 整行  FILENAME: 文件名
```

---

### 5.4 组合使用

```bash
# 案例1：查找Top内存进程
ps aux | sort -nk4 -r | head -10

# 案例2：日志分析完整流程
cat access.log | grep "200" | awk '{print $7}' | sort | uniq -c | sort -nr | head -10 > top_urls.txt

# 案例3：多条件过滤
cat app.log | grep -v "DEBUG" | grep -E "error|warn" | awk -F'[' '{print $2}' | awk -F']' '{print $1}'

# 案例4：时间范围提取
sed -n '/2024-01-01 10:00:00/,/2024-01-01 11:00:00/p' access.log

# 案例5：CSV处理
awk -F',' '{printf "%-20s %-10s %5.2f\n", $1, $2, $3}' data.csv
```

---

## 六、文件系统与存储管理

### 6.1 磁盘管理

#### `fdisk` - 分区管理
```bash
fdisk -l             # 列出所有磁盘
fdisk /dev/sdb       # 进入交互模式
# 内部命令：m(帮助) n(新建) p(打印) w(写入) q(退出) d(删除)
```

#### `gdisk` - GPT分区管理
```bash
gdisk -l /dev/sdb    # 查看GPT分区
```

#### `parted` - 分区管理（支持大磁盘）
```bash
parted -l            # 列出所有磁盘
parted /dev/sdb mklabel gpt  # 创建GPT标签
parted /dev/sdb mkpart primary 0% 100%  # 创建分区
```

#### `mkfs` - 创建文件系统
```bash
mkfs.ext4 /dev/sdb1  # 创建ext4文件系统
mkfs.xfs /dev/sdb1   # 创建xfs文件系统
mkfs -t ext4 /dev/sdb1  # 通用格式
```

#### `fsck` - 文件系统检查
```bash
fsck /dev/sda1       # 检查并修复（卸载后执行）
fsck -y /dev/sda1    # 自动修复
fsck -n /dev/sda1    # 只检查不修复

# 生产警告：不要在挂载的文件系统上运行！
```

#### `mount`/`umount` - 挂载/卸载
```bash
mount                 # 显示挂载信息
mount /dev/sdb1 /mnt  # 挂载
mount -t nfs 192.168.1.1:/data /mnt  # 挂载NFS
mount -o remount,rw /  # 重新挂载为读写（救援模式常用）
umount /mnt           # 卸载
umount -l /mnt        # 懒卸载（文件系统忙时使用）
umount -f /mnt        # 强制卸载

# 开机挂载：编辑/etc/fstab
```

---

### 6.2 LVM管理

#### `pvcreate` - 创建物理卷
```bash
pvcreate /dev/sdb1   # 将分区初始化为物理卷
pvdisplay            # 显示物理卷信息
pvs                  # 简要显示
```

#### `vgcreate` - 创建卷组
```bash
vgcreate vg_data /dev/sdb1 /dev/sdc1  # 创建卷组包含两个PV
vgdisplay
vgs
vgextend vg_data /dev/sdd1  # 扩展卷组
vgreduce vg_data /dev/sdd1  # 缩减卷组
```

#### `lvcreate` - 创建逻辑卷
```bash
lvcreate -L 100G -n lv_home vg_data  # 创建100G逻辑卷
lvcreate -l 100%FREE -n lv_opt vg_data  # 使用所有剩余空间
lvdisplay
lvs
lvextend -L +50G /dev/vg_data/lv_home  # 扩展逻辑卷
lvreduce -L -50G /dev/vg_data/lv_home  # 缩减逻辑卷（危险）
```

#### `resize2fs`/`xfs_growfs` - 调整文件系统大小
```bash
resize2fs /dev/vg_data/lv_home  # ext4在线扩容
xfs_growfs /mnt/home            # xfs在线扩容
```

---

### 6.3 RAID管理

#### `mdadm` - RAID管理
```bash
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sdb1 /dev/sdc1  # 创建RAID1
mdadm --detail /dev/md0  # 查看RAID详情
mdadm --manage /dev/md0 --add /dev/sdd1  # 添加磁盘
mdadm --manage /dev/md0 --fail /dev/sdb1 # 标记故障盘
watch -n 1 cat /proc/mdstat  # 监控RAID同步
```

---

## 七、系统服务管理

### 7.1 systemd（现代系统）

#### `systemctl` - 服务控制
```bash
systemctl start nginx      # 启动服务
systemctl stop nginx       # 停止服务
systemctl restart nginx    # 重启服务
systemctl reload nginx     # 重载配置
systemctl status nginx     # 查看状态
systemctl enable nginx     # 开机自启
systemctl disable nginx    # 取消开机自启
systemctl is-active nginx  # 检查是否运行
systemctl is-enabled nginx # 检查是否开机自启
systemctl list-units --type=service --state=active  # 列出所有运行服务
systemctl list-unit-files | grep enabled  # 列出所有开机自启服务
systemctl daemon-reload    # 重载配置（修改unit文件后）
systemctl show nginx       # 显示详细属性

# 生产技巧：查看服务启动失败原因
systemctl status nginx -l  # 显示完整日志
journalctl -u nginx -f      # 实时跟踪服务日志
journalctl -u nginx --since "10 minutes ago"  # 最近10分钟日志
```

#### `journalctl` - systemd日志
```bash
journalctl -f          # 实时跟踪
journalctl -u nginx    # 指定服务
journalctl -k          # 内核日志
journalctl --since "2024-01-01" --until "2024-01-02"  # 时间范围
journalctl -p err      # 只显示错误
journalctl -n 100      # 最后100行
journalctl --disk-usage  # 查看日志占用空间
journalctl --vacuum-size=100M  # 清理日志到100M
```

---

### 7.2 SysVinit（传统系统）

#### `service` - 服务控制
```bash
service nginx start
service nginx stop
service nginx restart
service nginx reload
service nginx status

# 已废弃，推荐systemctl
```

#### `chkconfig` - 开机自启管理
```bash
chkconfig --list     # 列出所有服务
chkconfig nginx on   # 开机自启
chkconfig nginx off  # 取消自启
```

---

### 7.3 定时任务

#### `crontab` - 计划任务
```bash
crontab -e           # 编辑当前用户的crontab
crontab -l           # 列出
crontab -r           # 删除
crontab -u user -e   # 编辑指定用户的

# 格式：分 时 日 月 周 命令
# */5 * * * * /path/to/script.sh  # 每5分钟
# 0 2 * * * /path/to/backup.sh     # 每天2点
```

#### `at` - 一次性任务
```bash
at 14:00 tomorrow    # 明天14点执行
at> /path/to/command
at> Ctrl+D

atq                  # 查看队列
atrm 1               # 删除任务1
```

---

## 八、用户与权限管理

### 8.1 用户管理

#### `useradd` - 添加用户
```bash
useradd -m -s /bin/bash -g users -G wheel newuser  # 创建用户
# -m 创建home目录  -s 指定shell  -g 主组  -G 附加组

# 生产技巧：批量创建用户
for i in {1..10}; do useradd -m user$i; echo "user$i:password" | chpasswd; done
```

#### `usermod` - 修改用户
```bash
usermod -aG wheel newuser  # 追加到wheel组
usermod -s /bin/zsh newuser  # 修改shell
usermod -L newuser     # 锁定用户
usermod -U newuser     # 解锁用户
```

#### `userdel` - 删除用户
```bash
userdel newuser        # 删除用户
userdel -r newuser     # 同时删除home目录和邮件
```

#### `passwd` - 修改密码
```bash
passwd                 # 修改当前用户密码
passwd newuser         # 修改指定用户密码
passwd -l newuser      # 锁定用户
passwd -u newuser      # 解锁用户
passwd -e newuser      # 强制下次登录修改密码
```

#### `su` - 切换用户
```bash
su - username          # 完全切换（推荐）
su username            # 不切换环境变量
su -c "command" username  # 以某用户执行命令
```

#### `sudo` - 提权执行
```bash
sudo command           # 以root执行
sudo -u user command   # 以指定用户执行
sudo -i                # 切换到root shell
sudo -l                # 列出当前用户可执行命令

# 生产配置：/etc/sudoers
# visudo 安全编辑
# user ALL=(ALL) NOPASSWD:ALL  # 免密码sudo（谨慎）
```

---

### 8.2 组管理

#### `groupadd` - 添加组
```bash
groupadd developers    # 创建组
groupadd -g 1000 dev   # 指定GID
```

#### `groupmod` - 修改组
```bash
groupmod -n newname oldname  # 重命名
```

#### `groupdel` - 删除组
```bash
groupdel developers    # 删除组（不能有用户为主组）
```

---

### 8.3 权限管理

#### `chmod` - 修改权限
```bash
chmod 755 file         # rwxr-xr-x
chmod u+x file         # 用户添加执行权限
chmod g-w file         # 组移除写权限
chmod o=r-- file       # 其他用户只读
chmod -R 755 dir       # 递归修改

# 数字权限：r=4 w=2 x=1
# 755 = 7(rwx) 5(r-x) 5(r-x)
```

#### `chown` - 修改所有者
```bash
chown user:group file  # 同时修改用户和组
chown user file        # 只修改用户
chown :group file      # 只修改组
chown -R user:group dir  # 递归修改

# 生产技巧：批量修改
chown -R www-data:www-data /var/www/html
```

#### `chgrp` - 修改组
```bash
chgrp group file       # 修改文件组
chgrp -R group dir     # 递归修改
```

#### `umask` - 默认权限掩码
```bash
umask                  # 查看当前掩码
umask 022              # 设置掩码
# 默认文件权限：666-umask
# 默认目录权限：777-umask
```

---

### 8.4 高级权限

#### `setuid`/`setgid`/`sticky bit`
```bash
chmod u+s file         # setuid (4) 以文件所有者权限执行
chmod g+s dir          # setgid (2) 新建文件继承目录组
chmod +t dir           # sticky bit (1) 用户只能删除自己的文件

# 数字表示：chmod 4755 file (setuid)
#          chmod 2755 dir  (setgid)
#          chmod 1777 dir  (sticky)
```

#### `ACL` - 访问控制列表
```bash
setfacl -m u:user:rwx file  # 给用户添加权限
setfacl -m g:group:r-x file # 给组添加权限
setfacl -x u:user file      # 删除用户ACL
setfacl -b file             # 清除所有ACL
getfacl file                # 查看ACL

# 生产场景：多用户共享目录
setfacl -R -m u:user1:rwx,u:user2:rwx /shared
setfacl -R -d -m u:user1:rwx,u:user2:rwx /shared  # 默认ACL
```

---

## 九、压缩与备份

### 9.1 压缩工具

#### `tar` - 归档（运维必备）
```bash
tar -czf archive.tar.gz /path  # 压缩（c创建，z gzip，f文件）
tar -xzf archive.tar.gz        # 解压（x解压）
tar -tzf archive.tar.gz        # 查看内容（t列表）
tar -czf - /path | ssh user@host "tar -xzf - -C /dest"  # 远程压缩传输
tar --exclude='*.log' -czf archive.tar.gz /path  # 排除文件

# 参数：
# c: 创建  x: 解压  t: 查看
# z: gzip  j: bzip2  J: xz
# v: 显示过程  f: 指定文件
# C: 切换到目录
```

#### `gzip`/`gunzip`
```bash
gzip file              # 压缩为file.gz
gzip -9 file           # 最高压缩率（1-9）
gzip -d file.gz        # 解压
gunzip file.gz         # 解压
```

#### `zip`/`unzip`
```bash
zip -r archive.zip dir  # 压缩目录
unzip archive.zip      # 解压
unzip -l archive.zip   # 查看内容
```

#### `rsync` - 同步与备份（运维神器）
```bash
rsync -avz /src/ /dst/  # 同步目录（a归档，v详细，z压缩）
rsync -avz /src/ user@host:/dst/  # 远程同步
rsync -avz --delete /src/ /dst/  # 删除目标端多余文件
rsync -avz --exclude='*.log' /src/ /dst/  # 排除
rsync -avz --bwlimit=10240 /src/ /dst/  # 限速10MB/s
rsync -avz --partial --progress /src/ /dst/  # 断点续传
rsync -avzh --link-dest=/latest /src/ /dst/`date +%Y%m%d`/  # 硬链接增量备份

# 生产案例：每日备份脚本
rsync -avz --delete /data/ /backup/`date +%Y%m%d`/
```

---

### 9.2 备份策略

#### `dd` - 磁盘克隆
```bash
dd if=/dev/sda of=/dev/sdb bs=4M  # 磁盘对拷
dd if=/dev/sda of=disk.img        # 创建镜像
dd if=disk.img of=/dev/sda        # 恢复镜像
dd if=/dev/zero of=/swapfile bs=1M count=4096  # 创建swap文件

# 生产警告：dd是危险命令，if/of参数务必确认
```

#### `cpio` - 归档
```bash
find /path | cpio -o > archive.cpio  # 创建
cpio -i < archive.cpio               # 解压
```

---

## 十、系统信息与管理

### 10.1 系统信息

#### `uname` - 内核信息
```bash
uname -a             # 显示所有信息
uname -r             # 内核版本
uname -m             # 架构
uname -n             # 主机名
```

#### `hostnamectl` - 主机名管理
```bash
hostnamectl           # 显示主机信息
hostnamectl set-hostname newname  # 修改主机名
```

#### `uptime` - 运行时间与负载
```bash
# 详见2.5节
```

#### `who`/`w` - 登录用户
```bash
who                   # 显示登录用户
w                     # 显示登录用户及正在执行的命令
```

#### `last` - 登录历史
```bash
# 详见2.5节
```

### 10.2 硬件信息

#### `lspci` - PCI设备
```bash
lspci                 # 显示所有PCI设备
lspci -v              # 详细信息
lspci | grep -i vga   # 查找显卡
```

#### `lsusb` - USB设备
```bash
lsusb                 # 显示USB设备
lsusb -t              # 树状显示
```

#### `lscpu` - CPU信息
```bash
lscpu                 # 显示CPU架构信息
lscpu | grep "CPU(s)" # 核心数
```

#### `dmidecode` - 硬件DMI信息
```bash
dmidecode             # 显示所有硬件信息
dmidecode -t memory   # 显示内存信息
dmidecode -t system   # 显示系统信息
```

#### `hdparm` - 磁盘信息
```bash
hdparm -I /dev/sda    # 显示磁盘详细信息
hdparm -tT /dev/sda   # 测试磁盘性能
```

---

### 10.3 内核与模块

#### `lsmod` - 加载的模块
```bash
lsmod                 # 显示已加载内核模块
lsmod | grep kvm      # 查找特定模块
```

#### `modinfo` - 模块信息
```bash
modinfo kvm            # 显示模块详细信息
```

#### `modprobe` - 加载/卸载模块
```bash
modprobe kvm           # 加载模块
modprobe -r kvm        # 卸载模块
```

#### `insmod`/`rmmod` - 加载/卸载模块
```bash
insmod /path/to/module.ko
rmmod kvm
```

#### `sysctl` - 内核参数
```bash
sysctl -a              # 显示所有参数
sysctl -p              # 加载/etc/sysctl.conf
sysctl -w vm.swappiness=10  # 临时修改
echo "vm.swappiness=10" >> /etc/sysctl.conf && sysctl -p  # 永久修改
```

---

## 十一、性能调优（专家级）

### 11.1 CPU调优

#### CPU亲和性
```bash
# 详见10.1节
```

#### 中断均衡
```bash
cat /proc/interrupts   # 查看中断分布
echo 2 > /proc/irq/24/smp_affinity  # 将中断绑定到CPU2

# 自动均衡：irqbalance服务
systemctl status irqbalance
```

#### 调节器
```bash
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor  # 查看当前调节器
echo performance > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor  # 性能模式
echo powersave > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor   # 节能模式

# 可用调节器：performance powersave ondemand conservative
```

---

### 11.2 内存调优

#### swappiness
```bash
# 详见2.2节
```

#### 透明大页
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled  # 查看状态
echo never > /sys/kernel/mm/transparent_hugepage/enabled  # 禁用（数据库推荐）
```

#### 内存压缩
```bash
cat /proc/sys/vm/drop_caches  # 释放缓存
# echo 1 > /proc/sys/vm/drop_caches  # 释放pagecache
# echo 2 > /proc/sys/vm/drop_caches  # 释放dentries和inodes
# echo 3 > /proc/sys/vm/drop_caches  # 释放所有缓存（生产环境慎用）
```

---

### 11.3 磁盘调优

#### IO调度器
```bash
cat /sys/block/sda/queue/scheduler  # 查看当前调度器
echo deadline > /sys/block/sda/queue/scheduler  # 修改为deadline

# 调度器选择：
# noop: SSD/闪存
# deadline: 数据库（保证延迟）
# cfq: 桌面/通用
```

#### 文件系统调优
```bash
# ext4挂载选项
/dev/sdb1 /data ext4 defaults,noatime,nodiratime 0 0

# XFS调优
xfs_info /dev/sda1   # 查看XFS信息
```

---

### 11.4 网络调优

#### TCP参数
```bash
# 详见10.1节
```

#### 文件描述符限制
```bash
ulimit -n            # 查看当前限制
ulimit -n 65535      # 临时修改

# 永久修改：/etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
```

#### 端口范围
```bash
cat /proc/sys/net/ipv4/ip_local_port_range  # 查看端口范围
echo "1024 65535" > /proc/sys/net/ipv4/ip_local_port_range  # 扩大范围
```

---

## 十二、安全与审计

### 12.1 文件完整性

#### `md5sum`/`sha256sum` - 校验和
```bash
md5sum file          # 计算MD5
sha256sum file       # 计算SHA256
md5sum -c file.md5   # 校验文件
```

#### `chattr`/`lsattr` - 扩展属性
```bash
chattr +i file       # 设置不可变（无法修改删除）
chattr -i file       # 取消不可变
chattr +a file       # 只允许追加
lsattr file          # 查看扩展属性

# 生产应用：保护关键配置文件
chattr +i /etc/passwd /etc/shadow
```

---

### 12.2 安全工具

#### `iptables` - 防火墙（传统）
```bash
iptables -L -n       # 查看规则
iptables -A INPUT -p tcp --dport 22 -j ACCEPT  # 放行SSH
iptables -A INPUT -p tcp --dport 80 -j ACCEPT  # 放行HTTP
iptables -A INPUT -j DROP  # 默认拒绝（注意顺序）
service iptables save  # 保存规则

# 生产脚本：配置基础防火墙
#!/bin/bash
iptables -F
iptables -A INPUT -i lo -j ACCEPT
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
iptables -P INPUT DROP
```

#### `firewalld` - 动态防火墙
```bash
firewall-cmd --state   # 查看状态
firewall-cmd --list-all  # 查看所有规则
firewall-cmd --add-port=8080/tcp --permanent  # 永久添加端口
firewall-cmd --reload  # 重载规则
```

#### `ufw` - Ubuntu防火墙
```bash
ufw status           # 查看状态
ufw allow 22/tcp     # 放行SSH
ufw enable           # 启用防火墙
```

---

### 12.3 审计

#### `lastlog` - 最后登录
```bash
lastlog               # 显示所有用户最后登录
lastlog -u username   # 指定用户
```

#### `lastb` - 失败登录
```bash
lastb                 # 查看失败登录尝试
```

#### `ausearch` - SELinux审计
```bash
ausearch -m avc -ts today  # 查看今天SELinux拒绝
```

---

## 十三、自动化运维脚本模板

### 13.1 系统巡检脚本
```bash
#!/bin/bash
# 系统健康检查脚本

LOGFILE="/var/log/system_check_$(date +%Y%m%d).log"
exec > $LOGFILE 2>&1

echo "===== System Check $(date) ====="

# CPU检查
echo "--- CPU Check ---"
top -bn1 | grep "Cpu(s)"
mpstat -P ALL 1 1

# 内存检查
echo "--- Memory Check ---"
free -h
vmstat -s | grep -E "swapped|used memory"

# 磁盘检查
echo "--- Disk Check ---"
df -h
df -i

# IO检查
echo "--- IO Check ---"
iostat -x 1 3

# 网络检查
echo "--- Network Check ---"
ss -s
netstat -i

# 负载检查
echo "--- Load Check ---"
uptime
cat /proc/loadavg

# 服务检查
echo "--- Service Check ---"
systemctl list-units --type=service --state=failed

echo "===== Check Complete $(date) ====="
```

### 13.2 日志分析脚本
```bash
#!/bin/bash
# Nginx日志分析脚本

LOGFILE="/var/log/nginx/access.log"
REPORT="/tmp/nginx_report_$(date +%Y%m%d).txt"

# Top 10 IP
echo "Top 10 IP:" > $REPORT
awk '{print $1}' $LOGFILE | sort | uniq -c | sort -nr | head -10 >> $REPORT

# Top 10 URL
echo -e "\nTop 10 URL:" >> $REPORT
awk '{print $7}' $LOGFILE | grep -v '^$' | sort | uniq -c | sort -nr | head -10 >> $REPORT

# 状态码统计
echo -e "\nStatus Code:" >> $REPORT
awk '{print $9}' $LOGFILE | sort | uniq -c | sort -nr >> $REPORT

# 响应时间分析（假设日志格式包含$request_time）
echo -e "\nResponse Time:" >> $REPORT
awk '{print $NF}' $LOGFILE | awk '{sum+=$1; count++} END {print "Avg: " sum/count}' >> $REPORT

mail -s "Nginx Daily Report" admin@example.com < $REPORT
```

### 13.3 自动扩容脚本
```bash
#!/bin/bash
# 磁盘空间监控与扩容（LVM）

THRESHOLD=85
VG_NAME="vg_data"
LV_NAME="lv_home"
MOUNT_POINT="/home"

CURRENT=$(df -h $MOUNT_POINT | awk 'NR==2 {print $5}' | sed 's/%//')

if [ $CURRENT -gt $THRESHOLD ]; then
    # 检查是否有可用空间
    FREE=$(vgs --noheadings -o vg_free $VG_NAME | sed 's/ //g')
    if [ "$FREE" != "0" ]; then
        # 扩展逻辑卷（增加10G）
        lvextend -L +10G /dev/$VG_NAME/$LV_NAME
        # 扩展文件系统
        resize2fs /dev/$VG_NAME/$LV_NAME  # ext4
        # xfs_growfs $MOUNT_POINT  # xfs
        echo "Expanded $LV_NAME by 10G" | mail -s "Disk Alert" admin@example.com
    else
        echo "No free space in VG" | mail -s "URGENT Disk Alert" admin@example.com
    fi
fi
```

---

## 十四、生产环境故障排查案例

### 案例1：CPU 100%但找不到进程
```bash
# 1. 检查系统负载
uptime
# 2. 查看CPU等待IO情况
top
# 发现wa很高（94%）
# 3. 定位IO瓶颈进程
iotop -oP
# 发现是rsync备份进程
# 4. 限制带宽
killall rsync
nohup rsync -avz --bwlimit=10240 /src/ /dst/ &

# 解决：将备份任务调整到凌晨2点执行
```

### 案例2：内存泄漏排查
```bash
#!/bin/bash
# 内存泄漏检测脚本

PID=$1
INTERVAL=60
LOG="mem_leak_${PID}.log"

echo "Monitoring PID $PID every $INTERVAL seconds..." > $LOG

while true; do
    echo "=== $(date) ===" >> $LOG
    pmap -x $PID | tail -1 >> $LOG
    ps -p $PID -o pid,rss,vsz,cmd >> $LOG
    sleep $INTERVAL
done

# 分析：观察RSS列是否持续增长
# 配合jmap分析Java堆：jmap -histo:live <PID>
```

### 案例3：网络连接数暴增
```bash
# 1. 查看连接数
ss -s
# 2. 查看TIME_WAIT
ss -tan | grep TIME_WAIT | wc -l
# 3. 查看ESTABLISHED
ss -tan | grep ESTABLISHED | wc -l
# 4. 查找异常IP
netstat -anp | grep ESTABLISHED | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head

# 解决：调整TCP参数
echo "net.ipv4.tcp_tw_reuse = 1" >> /etc/sysctl.conf
echo "net.ipv4.tcp_max_tw_buckets = 5000" >> /etc/sysctl.conf
sysctl -p
```

### 案例4：磁盘空间异常
```bash
# 1. 查看磁盘使用
df -h
# 发现/分区100%
# 2. 查找大文件
du -h --max-depth=1 / | sort -hr | head -10
# 3. 检查已删除但仍在占用的文件
lsof | grep deleted
# 发现日志文件被删除但进程未释放
# 4. 释放空间
> /proc/<PID>/fd/<FD>
# 或者重启进程
```

---

## 十五、运维最佳实践总结

### 15.1 生产环境黄金法则

1. **备份先行**：任何操作前备份配置文件和数据
2. **测试验证**：先在测试环境验证，再应用到生产
3. **灰度发布**：分批重启服务，避免全量操作
4. **监控告警**：所有操作都要监控，设置告警
5. **文档记录**：所有变更必须记录

### 15.2 命令使用原则

1. **rm**：使用`rm -i`别名，高危操作先`ls`确认
2. **dd**：操作磁盘前再三确认`if=`和`of=`
3. **vim**：编辑配置文件前先备份
4. **systemctl**：重启服务前检查配置语法
5. **iptables**：修改防火墙前保存当前规则

### 15.3 性能优化 checklist

- [ ] CPU：检查亲和性、中断均衡、调度器
- [ ] 内存：检查swappiness、泄漏、缓存
- [ ] 磁盘：检查IO调度器、使用率、inode
- [ ] 网络：检查连接数、TCP参数、带宽
- [ ] 服务：检查资源限制、日志级别、健康检查

### 15.4 推荐工具组合

- **监控**：Prometheus + Grafana + Alertmanager
- **日志**：ELK (Elasticsearch + Logstash + Kibana)
- **自动化**：Ansible + Jenkins/GitLab CI
- **追踪**：SkyWalking / Jaeger
- **CMDB**：NetBox / CMDBuild

---

