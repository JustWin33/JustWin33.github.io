---
title: Linux系统管理
description: 
date: 2025-11-10
categories:
    - 
    - 
---


# Linux系统管理

## 一、存储管理基础

### 1.1 RAID磁盘阵列技术

#### 1.1.1 RAID级别对比与选择

| 级别        | 核心特点                 | 磁盘利用率 | 最少磁盘数 | 容错能力 | 适用场景                 |
| ----------- | ------------------------ | ---------- | ---------- | -------- | ------------------------ |
| **RAID 0**  | 条带化，性能翻倍，无冗余 | 100%       | 2          | 0        | 临时数据、性能测试       |
| **RAID 1**  | 完全镜像，100%冗余       | 50%        | 2          | 1        | 系统盘、关键数据         |
| **RAID 5**  | 分布式校验，读快写慢     | (N-1)/N    | 3          | 1        | 文件服务器、常规应用     |
| **RAID 6**  | 双校验，更高安全         | (N-2)/N    | 4          | 2        | 归档存储、高可靠性需求   |
| **RAID 10** | RAID1+0，镜像+条带       | 50%        | 4          | 每组1个  | 数据库、高性能需求       |
| **RAID 01** | RAID0+1，条带+镜像       | 50%        | 4          | 每组1个  | 不推荐，安全性低于RAID10 |

**关键区别**：
- RAID10先镜像后条带，安全性更高，允许不同镜像组各坏1块盘
- RAID01先条带后镜像，安全性低，一旦某个RAID0组坏1块盘，整个阵列失效

#### 1.1.2 软RAID搭建与维护（mdadm）

```bash
# 1. 准备磁盘分区（类型fd: Linux raid auto）
fdisk /dev/sdb
# 依次创建分区：n→p→1→回车→+20G→t→1→fd→w

partprobe /dev/sdb  # 重读分区表

# 2. 创建RAID5阵列（3数据盘+1热备）
mdadm -C /dev/md5 -l5 -n3 -x1 /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4
# 参数说明：
# -C: 创建模式
# -l: RAID级别（level）
# -n: 数据盘数量
# -x: 热备盘数量

# 3. 查看RAID状态
cat /proc/mdstat
mdadm -D /dev/md5  # 详细信息

# 4. 创建配置文件（必须操作，否则重启后RAID名称会变化）
mdadm -D -s > /etc/mdadm.conf
# 编辑确保包含：DEVICE /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4
# 和ARRAY信息

# 5. 格式化与挂载
mkfs.xfs /dev/md5
mkdir /raid5
mount /dev/md5 /raid5
echo "/dev/md5 /raid5 xfs defaults 0 0" >> /etc/fstab

# 6. 模拟故障与恢复
mdadm /dev/md5 --fail /dev/sdb2  # 模拟sdb2故障
cat /proc/mdstat  # 观察热备盘自动重建
mdadm /dev/md5 --remove /dev/sdb2  # 移除故障盘
mdadm /dev/md5 --add /dev/sdb5     # 添加新盘（自动成为热备）

# 7. 性能测试
dd if=/dev/zero of=/raid5/test.data bs=1M count=1024
hdparm -t /dev/md5

# 8. 停止与删除RAID
umount /raid5
mdadm -S /dev/md5              # 停止阵列
mdadm --zero-superblock /dev/sdb1 /dev/sdb2 /dev/sdb3 /dev/sdb4  # 清除超级块
rm -rf /etc/mdadm.conf
```

#### 1.1.3 硬RAID配置要点

硬RAID通过RAID卡BIOS配置：
1. 开机按Ctrl+R或Ctrl+A进入RAID卡配置界面
2. 选择Create Virtual Drive
3. 选择RAID级别，添加物理磁盘
4. 设置热备盘（Global Hot Spare）
5. 配置完成后，系统识别为单一/dev/sdX设备
6. 安装操作系统前需加载RAID卡驱动

### 1.2 LVM逻辑卷管理

#### 1.2.1 LVM三层架构

```
物理卷(PV) → 卷组(VG) → 逻辑卷(LV)
  ↓              ↓              ↓
物理磁盘      存储池        可用分区
(PE=4MB)   (PE集合)     (LE映射)
```

**关键术语**：
- **PE**（Physical Extent）：物理卷的基本单元，默认4MB
- **LE**（Logical Extent）：逻辑卷的基本单元，与PE一一对应
- **UUID**：每个PV、VG、LV的唯一标识符
- **元数据区域**：存储LVM配置信息，默认在每个PV头部

#### 1.2.2 完整操作步骤

```bash
# 1. 准备物理卷（以/dev/sdc为例）
pvcreate /dev/sdc1 /dev/sdc2 /dev/sdc3
# 批量创建：pvcreate /dev/sdc{1,2,3}

# 查看PV信息
pvscan                    # 扫描所有PV
pvs                       # 简要信息（推荐）
pvdisplay /dev/sdc1       # 详细信息

# 2. 创建卷组myvg
vgcreate myvg /dev/sdc1 /dev/sdc2
# 指定PE大小：vgcreate -s 16M myvg /dev/sdc1 /dev/sdc2

# 查看VG信息
vgscan
vgs
vgdisplay myvg            # 查看PE总数、空闲数

# 3. 创建逻辑卷
lvcreate -L 8G -n mylv myvg        # 指定大小
lvcreate -l 2048 -n mylv myvg      # 指定PE数（2048*4MB=8G）
lvcreate -l +100%FREE -n mylv myvg # 使用全部剩余空间

# 查看LV信息
lvscan
lvs
lvdisplay /dev/myvg/mylv

# 4. 格式化与挂载
mkfs.xfs /dev/myvg/mylv
# 或mkfs.ext4 /dev/myvg/mylv

mkdir /media/s1
mount /dev/myvg/mylv /media/s1
df -hT

# 设置开机挂载
echo "/dev/myvg/mylv /media/s1 xfs defaults 0 0" >> /etc/fstab
# 或使用UUID（推荐）
blkid /dev/myvg/mylv >> /etc/fstab  # 然后编辑替换

# 5. 在线扩展（LVM最大优势）
lvextend -L +1G /dev/myvg/mylv       # 增加1G
# 或lvextend -L 9G /dev/myvg/mylv    # 扩展到9G

# XFS文件系统调整（必须在线执行）
xfs_growfs /dev/myvg/mylv           # 自动扩展至LV大小

# ext4文件系统调整（可离线或在线）
resize2fs /dev/myvg/mylv            # 扩展到LV大小
# resize2fs /dev/myvg/mylv 5G       # 收缩到5G（需先卸载）

# 查看扩展结果
df -hT

# 6. 卷组扩容
vgextend myvg /dev/sdc3             # 添加新PV到VG

# 7. 迁移数据（pvmove）
# 将sdc1上的数据迁移到sdc3
pvmove /dev/sdc1 /dev/sdc3

# 8. 删除LVM（顺序：LV→VG→PV）
umount /media/s1
lvremove /dev/myvg/mylv             # 确认删除
vgremove myvg
pvremove /dev/sdc1 /dev/sdc2 /dev/sdc3

# 9. 创建快照（重要功能）
lvcreate -L 1G -s -n mylv_snap /dev/myvg/mylv
# 参数：-s创建快照，-n快照名称

# 恢复快照
umount /media/s1
lvconvert --merge /dev/myvg/mylv_snap  # 合并快照到原LV
mount /dev/myvg/mylv /media/s1

# 删除快照
lvremove /dev/myvg/mylv_snap
```

#### 1.2.3 重要注意事项

1. **/boot分区不能放在LVM上**
   - 原因：GRUB无法识别LVM，系统启动流程为BIOS→MBR→GRUB→内核→init，GRUB阶段无法读取LVM中的文件
   - 解决方案：/boot必须是普通分区或独立RAID1

2. **扩容后df看不到新空间**
   - 现象：lvextend成功，但df显示大小不变
   - 原因：逻辑卷扩容后，文件系统未同步更新
   - 解决：执行`xfs_growsfs`（XFS）或`resize2fs`（ext4）

3. **根分区在线扩容**
   ```bash
   # 添加新硬盘→创建PV→扩展VG→扩展LV→扩展文件系统
   pvcreate /dev/sdd1
   vgextend centos /dev/sdd1    # 假设VG名为centos
   lvextend -l +100%FREE /dev/centos/root
   xfs_growfs /
   ```

4. **PE大小选择**
   - 默认4MB适合大多数场景
   - 大数据存储可考虑16MB或更大，减少元数据开销
   - PE大小在VG创建后不可更改

## 二、Linux系统引导与启动管理

### 2.1 完整引导流程详解

```
1. 开机自检(POST, Power-On Self-Test)
   ↓
   BIOS/UEFI初始化硬件，加载CMOS设置
   ↓
   按启动顺序查找可引导设备（硬盘、U盘、PXE）

2. MBR引导（传统BIOS模式）
   ↓
   读取磁盘第一个扇区（512字节）的MBR
   - 446字节：引导加载程序（GRUB stage1）
   - 64字节：分区表（4个主分区，各16字节）
   - 2字节：55AA签名（有效性标志）
   ↓
   GRUB stage1加载/boot/grub2/i386-pc/boot.img

3. GRUB2菜单阶段
   ↓
   加载/boot/grub2/grub.cfg（由grub2-mkconfig生成）
   加载字体、背景、菜单项
   加载内核vmlinuz和initramfs到内存
   ↓
   按Shift或Esc可进入GRUB编辑模式

4. 内核加载阶段
   ↓
   解压vmlinuz到内存
   初始化硬件、挂载驱动
   加载initramfs（临时根文件系统）
   - 包含必要的驱动模块（ext4/xfs、RAID、LVM）
   - 包含udev、systemd等早期启动程序

5. systemd初始化（CentOS 7+）
   ↓
   systemd成为PID 1，取代传统SysVinit
   读取/etc/systemd/system/default.target（指向multi-user.target或graphical.target）
   并行启动系统服务（由systemctl管理）
   执行/etc/rc.local（兼容性）
   启动getty/login提示用户登录
```

### 2.2 GRUB2修复与配置

#### 2.2.1 grub.cfg丢失修复

**现象**：开机直接进入grub rescue模式

**修复步骤**：
```bash
# 1. 查找boot分区
grub rescue> ls
(hd0) (hd0,msdos1) (hd0,msdos2)

# 2. 尝试查看分区内容
grub rescue> ls (hd0,msdos1)/
# 如果能看到vmlinuz和initramfs，说明这是boot分区

# 3. 设置根和prefix
grub rescue> set root=(hd0,msdos1)
grub rescue> set prefix=(hd0,msdos1)/boot/grub2

# 4. 加载必要模块
grub rescue> insmod linux
grub rescue> insmod xfs   # 根据文件系统类型

# 5. 手动引导（关键步骤）
grub rescue> linux16 /vmlinuz-3.10.0-1160.el7.x86_64 root=/dev/mapper/centos-root ro
grub rescue> initrd16 /initramfs-3.10.0-1160.el7.x86_64.img
grub rescue> boot

# 6. 进入系统后重建配置
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 2.2.2 MBR引导程序损坏修复

**现象**：启动黑屏，无GRUB菜单

**修复方法**：
```bash
# 方法一：使用系统安装盘进入救援模式
# 1. 选择Troubleshooting → Rescue a CentOS system
# 2. 选择1) Continue，挂载原系统到/mnt/sysimage
chroot /mnt/sysimage
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg
exit
reboot

# 方法二：如果系统仍在运行
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg
```

#### 2.2.3 GRUB2密码保护

```bash
# 1. 生成密码哈希
grub2-mkpasswd-pbkdf2
# 输入密码后得到：grub.pbkdf2.sha512.10000.xxx

# 2. 编辑/etc/grub.d/00_header
cat <<EOF
set superusers='admin'
password_pbkdf2 admin grub.pbkdf2.sha512.10000.xxx
export superusers
EOF

# 3. 更新GRUB配置
grub2-mkconfig -o /boot/grub2/grub.cfg

# 4. 效果：重启后按e编辑需输入用户名admin和密码
```

### 2.3 root密码破解

#### 2.3.1 rd.break方法（推荐）

```bash
# 1. 重启系统，在GRUB菜单按e编辑
# 2. 找到linux16行，在行尾添加 rd.break
linux16 ... ro rd.break

# 3. Ctrl+x启动
switch_root:/# mount -o remount,rw /sysroot
switch_root:/# chroot /sysroot

# 4. 修改密码
sh-4.2# passwd
New password: 
Retype new password: 
sh-4.2# touch /.autorelabel  # 关键！重置SELinux标签

# 5. 退出重启
sh-4.2# exit
switch_root:/# exit
# 系统会自动重启
```

**注意**：CentOS 7.2+可能需要：
```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd
touch /.autorelabel
```

#### 2.3.2 init=/bin/sh方法

```bash
# 1. GRUB编辑，在linux16行末添加 init=/bin/sh
linux16 ... ro init=/bin/sh

# 2. Ctrl+x启动
/bin/sh# mount -o remount,rw /
/bin/sh# passwd
/bin/sh# exec /sbin/init  # 或exec /sbin/reboot
```

#### 2.3.3 单用户模式（传统方法）

```bash
# 1. GRUB编辑，添加 single 或 1
linux16 ... ro single

# 2. 进入单用户后直接passwd
```

### 2.4 系统服务管理（systemd）

```bash
# 服务状态
systemctl status nginx    # 查看状态
systemctl start nginx     # 启动
systemctl stop nginx      # 停止
systemctl restart nginx   # 重启
systemctl reload nginx    # 重载配置
systemctl enable nginx    # 开机自启
systemctl disable nginx   # 禁用自启
systemctl is-enabled nginx # 查看是否自启
systemctl is-active nginx  # 查看是否运行

# 查看服务依赖
systemctl list-dependencies nginx

# 查看启动耗时
systemd-analyze blame

# 创建自定义服务
cat > /etc/systemd/system/myapp.service <<EOF
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/myapp
Restart=always
User=appuser

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl start myapp
```

## 三、进程与任务管理

### 3.1 进程查看命令详解

```bash
# ps命令（静态查看）
ps aux          # 显示所有进程的详细信息
# a: 所有终端  u: 用户信息  x: 无终端进程

# 字段解释：
USER: 进程所有者
PID: 进程ID
%CPU: CPU占用百分比
%MEM: 内存占用百分比
VSZ: 虚拟内存大小（KB）
RSS: 常驻内存大小（KB）
TTY: 终端设备（?表示无终端）
STAT: 状态码
  R: 运行中
  S: 睡眠（可中断）
  D: 不可中断睡眠（I/O等待）
  T: 停止
  Z: 僵尸进程
  +: 前台进程
  l: 多线程
  s: 会话领导者
START: 启动时间或日期
TIME: 累计CPU时间
COMMAND: 命令名

ps -elf         # 长格式显示
# -e: 所有进程  -l: 长格式  -f: 全格式

# 按条件筛选
ps -C sshd      # 按命令名精确查找
ps -u root      # 按用户筛选
pgrep -l sshd   # 仅显示PID和名称（轻量）
pstree -p       # 树状结构显示父子关系

# 动态查看
top             # 交互式实时查看
# 快捷键：
#   P: 按CPU排序  M: 按内存排序  T: 按时间排序
#   k: 杀死进程  r: 调整优先级  q: 退出

htop            # 增强版top（需安装：yum install htop）
# 支持鼠标操作，彩色显示，功能更强大

vmstat 1        # 每秒刷新系统资源统计
# procs: r(运行队列) b(阻塞)  
# memory: swpd free buff cache
# swap: si so
# io: bi bo
# cpu: us sy id wa st
```

### 3.2 进程控制信号

```bash
# 查看所有信号
kill -l
# 常用信号：
# 1  SIGHUP    : 重新加载配置（如nginx -s reload）
# 2  SIGINT    : Ctrl+C中断
# 9  SIGKILL   : 强制终止（不可捕获、阻塞）
# 15 SIGTERM   : 优雅终止（默认信号）
# 18 SIGCONT   : 继续运行（恢复）
# 19 SIGSTOP   : 停止进程（不可捕获）

# 终止进程
kill PID                    # 发送SIGTERM
kill -9 PID                 # 强制杀死（谨慎使用）
kill -KILL PID              # 同-9

# 按名称批量终止
killall httpd               # 杀死所有httpd进程
killall -9 java             # 强制杀死所有java进程

# 按条件终止
pkill -U user1              # 按用户杀死进程
pkill -t pts/0              # 按终端杀死进程
pkill -f "python manage.py" # 按完整命令名匹配

# 查看信号详情
man 7 signal

# 僵尸进程处理
# 僵尸进程（Z状态）无法kill，需杀死其父进程
ps -ef | grep defunct       # 查找僵尸进程
# 找到PPID，然后 kill -9 PPID
```

### 3.3 后台任务管理

```bash
# 直接放入后台（仍在当前终端）
command &                   # 后台运行（当前终端关闭后进程仍退出）
[1] 12345                   # 输出：任务号 PID

# nohup方式（推荐）nohup command &                    # 忽略SIGHUP信号（终端断开后继续运行）
# 输出重定向到nohup.out

# screen方式（会话管理）
yum install screen
screen -S session_name      # 创建会话
Ctrl+a d                    # 分离会话
screen -ls                  # 列出会话
screen -r session_name      # 恢复会话
screen -X -S session_name quit # 关闭会话

# tmux方式（现代选择）
tmux new -s session_name
Ctrl+b d                    # 分离
tmux ls                     # 列出
tmux attach -t session_name # 恢复

# 任务控制
Ctrl+Z                      # 暂停前台任务
jobs                        # 查看后台任务列表
[1]+  Stopped                 vim file1
[2]-  Running                 python server.py &

fg %1                       # 将任务1恢复到前台
bg %2                       # 让任务2在后台继续运行

# 任务调度优先级
nice -n 10 command          # 以nice值10启动（范围-20~19，默认0）
renice -n 5 -p 12345        # 调整正在运行的进程nice值
# nice值越高，CPU优先级越低
```

### 3.4 at一次性计划任务

```bash
# 1. 启动服务
systemctl start atd
systemctl enable atd
systemctl status atd

# 2. 设置任务
at 14:30                    # 今天14:30执行
at> /root/backup.sh
at> echo "backup completed" | mail -s "Backup" admin@company.com
at> <Ctrl+D>                # 提交任务

# 3. 时间格式支持
at 2pm + 3 days            # 3天后下午2点
at now + 5 minutes         # 5分钟后
at 23:00 2024-01-01        # 指定日期时间
at teatime                 # 下午4点
at noon                    # 中午12点
at midnight                # 午夜

# 4. 查看待执行任务
atq
# 输出：2   Mon Jan 15 14:30:00 2024 a root

# 5. 查看任务详情
at -c 2                    # 显示任务2的完整内容（包括环境变量）

# 6. 删除任务
atrm 2                     # 删除任务2
# 或
at -d 2

# 7. 限制at使用
/etc/at.deny               # 黑名单（默认存在）
/etc/at.allow              # 白名单（优先级更高）
# 如果at.allow存在，只有其中用户能使用at
# 如果都不存在，仅root能使用

# 8. 查看at日志
grep CROND /var/log/cron   # at由cron服务管理
```

### 3.5 crontab周期性计划任务

```bash
# 时间格式
# *    *    *    *    *    command
# 分   时   日   月   周   命令
# 范围：分(0-59) 时(0-23) 日(1-31) 月(1-12) 周(0-7,0和7都是周日)

# 特殊符号
*     # 任意值（每）
,     # 分隔列表：1,3,5
-     # 范围：1-5
*/n   # 每隔n单位：*/5

# 系统级任务
/etc/crontab               # 系统主配置文件
/etc/cron.d/               # 服务专用的cron文件
/etc/cron.hourly/          # 每小时执行
/etc/cron.daily/           # 每天执行
/etc/cron.weekly/          # 每周执行
/etc/cron.monthly/         # 每月执行

# 用户级任务
/var/spool/cron/username     # 用户crontab文件
crontab -e                  # 编辑当前用户任务
crontab -l                  # 列出任务
crontab -r                  # 删除所有任务
crontab -i                  # 删除前确认
crontab -u user -e          # root编辑其他用户任务

# 实用示例
# 每天3点备份
0 3 * * * /usr/local/bin/backup.sh

# 每周一三五10点
0 10 * * 1,3,5 /usr/bin/sync

# 每10分钟
*/10 * * * * /usr/bin/date >> /tmp/log

# 每小时第30分
30 * * * * /path/to/script

# 每天8-18点，每2小时
0 8-18/2 * * * /path/to/script

# 每月1号、15号的2:30
30 2 1,15 * * /path/to/script

# 每30秒执行（cron最小粒度为分钟，需配合sleep）
* * * * * /path/to/script
* * * * * sleep 30; /path/to/script

# crontab环境变量
# cron执行环境PATH有限，建议脚本中写绝对路径
# 或在crontab顶部设置
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=admin@company.com

# 查看cron日志
grep CRON /var/log/cron
tail -f /var/log/cron

# 权限问题
/etc/cron.deny            # 拒绝列表
/etc/cron.allow           # 允许列表（优先级更高）
```

---

## 四、网络基础与配置

### 4.1 TCP/IP协议栈详解

#### 4.1.1 OSI与TCP/IP模型对比

| OSI七层模型   | TCP/IP四层模型 | 协议              | 数据单元    | 设备          |
| ------------- | -------------- | ----------------- | ----------- | ------------- |
| 7. 应用层     | 应用层         | HTTP/FTP/DNS/SMTP | 数据        | 网关          |
| 6. 表示层     |                | TLS/SSL           | 数据        |               |
| 5. 会话层     |                | SSH/Socket        | 数据        |               |
| 4. 传输层     | 传输层         | TCP/UDP           | 段(Segment) | 防火墙        |
| 3. 网络层     | 网络层         | IP/ICMP/ARP       | 包(Packet)  | 路由器        |
| 2. 数据链路层 | 网络接口层     | Ethernet/PPP      | 帧(Frame)   | 交换机/网桥   |
| 1. 物理层     |                | IEEE 802.3        | 比特(Bit)   | 集线器/中继器 |

**关键概念**：
- **端到端**（传输层以上）：端口到端口，由操作系统处理
- **点到点**（网络层以下）：节点到节点，由硬件处理
- **封装与解封装**：数据每经过一层增加头部信息，接收方反向剥离

#### 4.1.2 TCP三次握手与四次挥手

**三次握手（建立连接）**：
```
Client → SYN=1, Seq=x → Server
Client ← SYN=1, ACK=1, Seq=y, Ack=x+1 ← Server
Client → ACK=1, Seq=x+1, Ack=y+1 → Server
```
**状态变迁**：
- LISTEN → SYN_SENT → SYN_RECEIVED → ESTABLISHED

**为什么不能两次握手？**：
防止已失效的连接请求突然到达服务器，造成资源浪费。三次握手可以确保双方的发送和接收能力都正常。

**四次挥手（断开连接）**：
```
Client → FIN=1, ACK=1 → Server    # Client主动关闭
Client ← ACK=1 ← Server           # Server确认
Client ← FIN=1, ACK=1 ← Server    # Server也准备关闭
Client → ACK=1 → Server           # Client最后确认
```
**状态变迁**：
- ESTABLISHED → FIN_WAIT1 → FIN_WAIT2 → TIME_WAIT → CLOSED
- ESTABLISHED → CLOSE_WAIT → LAST_ACK → CLOSED

**TIME_WAIT状态（2MSL）**：
- **MSL**（Maximum Segment Lifetime）：报文最大生存时间，通常为30秒到2分钟
- **2MSL等待目的**：确保最后一个ACK能到达；让网络中延迟的旧报文消失
- **端口耗尽问题**：高并发服务器可能出现大量TIME_WAIT，需通过`net.ipv4.tcp_tw_reuse=1`优化

#### 4.1.3 TCP状态机11种状态

| 状态             | 端类型 | 描述                  |
| ---------------- | ------ | --------------------- |
| **LISTEN**       | Server | 监听连接请求          |
| **SYN_SENT**     | Client | 已发送SYN，等待确认   |
| **SYN_RECEIVED** | Server | 收到SYN，发送SYN+ACK  |
| **ESTABLISHED**  | Both   | 连接已建立，数据传输  |
| **FIN_WAIT_1**   | Client | 发送FIN，等待ACK      |
| **FIN_WAIT_2**   | Client | 收到ACK，等待对方FIN  |
| **CLOSE_WAIT**   | Server | 收到FIN，等待应用关闭 |
| **CLOSING**      | Both   | 同时发送FIN           |
| **LAST_ACK**     | Server | 发送FIN，等待最后ACK  |
| **TIME_WAIT**    | Client | 等待2MSL              |
| **CLOSED**       | Both   | 连接完全关闭          |

**客户端特有状态**：SYN_SENT, FIN_WAIT_1, FIN_WAIT_2, CLOSING, TIME_WAIT
**服务器特有状态**：LISTEN, SYN_RECEIVED, CLOSE_WAIT, LAST_ACK

### 4.2 网络配置命令

#### 4.2.1 ifconfig（传统命令）

```bash
# 查看所有活动接口
ifconfig

# 查看指定接口（即使down）
ifconfig ens33

# ifconfig输出解读
ens33: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.200.100  netmask 255.255.255.0  broadcast 192.168.200.255
        inet6 fe80::20c:29ff:feb3:4c4a  prefixlen 64  scopeid 0x20<link>
        ether 00:0c:29:b3:4c:4a  txqueuelen 1000  (Ethernet)
        RX packets 12345  bytes 1234567 (1.1 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 6789  bytes 6789012 (6.4 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

# 字段说明：
UP: 接口已启用
BROADCAST: 支持广播
RUNNING: 正在运行
MULTICAST: 支持组播
mtu: 最大传输单元（1500字节）
inet: IPv4地址
netmask: 子网掩码
broadcast: 广播地址
ether: MAC地址
RX/TX: 接收/发送统计

# 临时配置IP
ifconfig ens33 192.168.200.100 netmask 255.255.255.0
# 或
ifconfig ens33 192.168.200.100/24

# 启用/禁用接口
ifconfig ens33 up
ifconfig ens33 down

# 添加虚拟接口
ifconfig ens33:0 192.168.200.200 netmask 255.255.255.0
```

#### 4.2.2 ip命令（推荐，iproute2工具集）

```bash
# 查看网络接口（数据链路层）
ip link
ip link show ens33

# 查看IP地址（网络层）
ip addr
ip addr show ens33

# 临时配置IP
ip addr add 192.168.200.100/24 dev ens33
ip addr del 192.168.200.100/24 dev ens33

# 启用/禁用接口
ip link set ens33 up
ip link set ens33 down

# 查看路由
ip route
ip route show

# 添加/删除路由
ip route add 10.0.0.0/8 via 192.168.200.1
ip route del 10.0.0.0/8

# 查看ARP缓存
ip neigh
ip neigh show

# 查看统计信息
ip -s link
```

#### 4.2.3 其他网络工具

```bash
# ethtool - 查看网卡详细信息（需root）
ethtool ens33
# 输出：
Settings for ens33:
        Supported ports: [ TP ]
        Supported link modes:   10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Supported pause frame use: No
        Supports auto-negotiation: Yes
        Advertised link modes:  10baseT/Half 10baseT/Full
                                100baseT/Half 100baseT/Full
                                1000baseT/Full
        Advertised pause frame use: No
        Advertised auto-negotiation: Yes
        Speed: 1000Mb/s          # 当前速率
        Duplex: Full             # 全双工
        Port: Twisted Pair
        PHYAD: 0
        Transceiver: internal
        Auto-negotiation: on
        MDI-X: off (auto)
        Supports Wake-on: d
        Wake-on: d
        Current message level: 0x00000007 (7)
        Link detected: yes       # 链路状态

# mii-tool - 简洁查看
mii-tool ens33
ens33: negotiated 1000baseT-FD flow-control, link ok

# ss - 现代netstat替代品（性能更好）
ss -tuln                  # 查看监听端口
ss -tun                   # 查看所有连接
ss -tun | grep :80        # 过滤80端口

# lsof - 查看端口占用
lsof -i :22               # 查看22端口
lsof -i TCP:80            # 查看TCP 80端口
lsof -p 1234              # 查看进程打开的文件

# ping - 连通性测试
ping -c 4 baidu.com       # 发送4个包
ping -i 2 baidu.com       # 间隔2秒
ping -w 10 baidu.com      # 10秒超时
ping -s 1024 baidu.com    # 指定包大小1024字节

# traceroute - 路由跟踪
traceroute baidu.com      # 跟踪路径
traceroute -n baidu.com   # 不解析主机名
```

#### 4.2.4 配置文件管理

**网络配置文件位置**：
- RHEL/CentOS 7: `/etc/sysconfig/network-scripts/ifcfg-ens33`
- RHEL/CentOS 8+: `/etc/NetworkManager/system-connections/`（推荐nmcli）

**典型配置**：
```bash
cat > /etc/sysconfig/network-scripts/ifcfg-ens33 <<EOF
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static          # static|dhcp|none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=ens33
UUID=5e2cfdc2-1234-4567-890a-bcdef1234567  # 唯一标识
DEVICE=ens33
ONBOOT=yes                # 开机启用
IPADDR=192.168.200.100
PREFIX=24                 # 或NETMASK=255.255.255.0
GATEWAY=192.168.200.2
DNS1=8.8.8.8
DNS2=114.114.114.114
EOF

# 重启网络
systemctl restart network
# 或
nmcli con reload; nmcli con up ens33
```

**UUID生成**：
```bash
uuidgen ens33              # 生成新UUID
# 将生成的UUID写入ifcfg文件
```

#### 4.2.5 路由配置

```bash
# 临时添加路由
route add -net 10.0.0.0/8 gw 192.168.200.1
route add -net 10.0.0.0 netmask 255.0.0.0 gw 192.168.200.1

# 删除路由
route del -net 10.0.0.0/8

# 添加默认网关
route add default gw 192.168.200.1
# 或
ip route add default via 192.168.200.1

# 查看路由表
route -n                   # 数字格式
ip route show

# 永久路由配置
# 方法1：/etc/rc.local
echo "route add -net 10.0.0.0/8 gw 192.168.200.1" >> /etc/rc.local

# 方法2：/etc/sysconfig/static-routes（不存在则创建）
# 格式：any net 10.0.0.0/8 gw 192.168.200.1
cat > /etc/sysconfig/static-routes <<EOF
any net 10.0.0.0 netmask 255.0.0.0 gw 192.168.200.1
any host 192.168.100.100 gw 192.168.200.1
EOF

# 方法3：/etc/sysconfig/network-scripts/route-ens33
cat > /etc/sysconfig/network-scripts/route-ens33 <<EOF
10.0.0.0/8 via 192.168.200.1
192.168.100.0/24 via 192.168.200.1
EOF
```

#### 4.2.6 IP转发（路由功能）

```bash
# 临时开启
echo "1" > /proc/sys/net/ipv4/ip_forward

# 永久开启
cat >> /etc/sysctl.conf <<EOF
net.ipv4.ip_forward = 1
EOF
sysctl -p                  # 立即生效

# 验证
sysctl net.ipv4.ip_forward
# 输出：net.ipv4.ip_forward = 1
```

### 4.3 DNS与主机名配置

#### 4.3.1 DNS解析流程（完整版）

```
用户输入www.baidu.com
  ↓
浏览器缓存检查（TTL过期时间）
  ↓
操作系统缓存检查（/etc/hosts, nscd缓存）
  ↓
本地DNS服务器（LDNS）查询
  ├→ 递归查询（客户端等待结果）
  └→ 迭代查询（LDNS逐级查询）
      ↓
      根域名服务器（.）→ 返回.com服务器地址
      ↓
      顶级域服务器（.com）→ 返回baidu.com授权服务器
      ↓
      权威域名服务器（baidu.com）→ 返回www.baidu.com的A记录
      ↓
LDNS缓存结果 → 返回给客户端
  ↓
浏览器获得IP → 发起TCP连接
```

**关键概念**：
- **递归查询**：DNS服务器为客户机完全解析域名，客户端只需等待最终结果
- **迭代查询**：DNS服务器返回下一个查询服务器地址，由客户端继续查询
- **TTL**：缓存生存时间，单位为秒
- **权威服务器**：拥有域名最终解析权的服务器

#### 4.3.2 /etc/hosts文件

```bash
# 格式：IP 主机名 别名
cat > /etc/hosts <<EOF
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.200.100    www.linux.com   www
192.168.200.100    ns1.linux.com   ns1
192.168.200.193    ns2.linux.com   ns2
EOF

# 优先级：/etc/hosts > DNS查询
# nsswitch.conf控制顺序
grep hosts /etc/nsswitch.conf
# 输出：hosts:      files dns myhostname
# files代表/etc/hosts，dns代表DNS服务器
```

#### 4.3.3 主机名配置

```bash
# 临时修改（立即生效，重启失效）
hostname new-hostname
# 或
hostnamectl set-hostname new-hostname

# 永久修改（RHEL 7+推荐）
hostnamectl set-hostname new-hostname --static

# 查看主机名
hostname
hostnamectl status

# 传统方式（编辑配置文件）
cat > /etc/hostname <<EOF
new-hostname
EOF

# 同时修改/etc/hosts中的127.0.0.1对应的主机名
```

---

## 五、用户与权限管理

### 5.1 用户账号体系

#### 5.1.1 账号文件结构

**/etc/passwd**（7个冒号分隔字段）：
```bash
root:x:0:0:root:/root:/bin/bash
用户名:密码占位符:UID:GID:注释:家目录:Shell

# 字段说明：
# 1. 用户名：登录名，不超过32字符
# 2. 密码：x表示密码在/etc/shadow中，空表示无密码
# 3. UID：0=root，1-999=系统用户，1000+=普通用户
# 4. GID：主组ID，对应/etc/group
# 5. 注释：用户全名或描述
# 6. 家目录：登录后默认目录
# 7. Shell：/bin/bash可登录，/sbin/nologin不可登录
```

**/etc/shadow**（9个冒号分隔字段）：
```bash
root:$6$rounds=5000$salt$hash:18000:0:99999:7:::
用户名:加密密码:最后修改天数:最小天数:最大天数:警告天数:过期宽限:失效日期:保留

# 字段说明：
# 1. 用户名：与passwd对应
# 2. 加密密码：$id$salt$hash
#    - $1$ MD5
#    - $2a$ Blowfish
#    - $5$ SHA-256
#    - $6$ SHA-512（CentOS 7+默认）
#    - !! 或 * 表示账户锁定
# 3. 最后修改：从1970-01-01算起的天数
# 4. 最小天数：密码修改最小间隔
# 5. 最大天数：密码有效最大天数
# 6. 警告天数：到期前警告天数
# 7. 过期宽限：密码过期后可用天数
# 8. 失效日期：账户失效日期（天数）
# 9. 保留：未使用
```

#### 5.1.2 用户管理命令

```bash
# 创建用户
useradd -u 1001 -g wheel -G admin,dev -c "Admin User" -s /bin/bash -d /home/admin admin
# -u: UID
# -g: 主组（必须已存在）
# -G: 附加组（逗号分隔）
# -c: 注释
# -s: Shell
# -d: 家目录
# -m: 自动创建家目录（默认）
# -M: 不创建家目录
# -e: 过期日期（YYYY-MM-DD）
# -f: 密码过期后宽限天数

# 创建系统用户（无登录Shell，无家目录）
useradd -r -s /sbin/nologin nginx

# 批量创建用户
for i in {1..10}; do useradd user$i; done

# 修改用户属性
usermod -l newname oldname          # 重命名用户
usermod -u 2000 admin               # 修改UID
usermod -g users admin              # 修改主组
usermod -aG docker admin            # 追加附加组（-a关键）
usermod -s /bin/zsh admin           # 修改Shell
usermod -d /newhome -m admin        # 修改家目录并移动文件
usermod -e 2024-12-31 admin         # 设置账户过期
usermod -L admin                    # 锁定账户（密码前加!）
usermod -U admin                    # 解锁账户（需执行两次）

# 删除用户
userdel admin                       # 仅删除用户
userdel -r admin                    # 删除用户及家目录、邮件

# 密码管理
passwd admin                        # 交互式设置密码
echo "password" | passwd --stdin admin  # 非交互式（需root）

# 密码策略
passwd -n 7 admin                   # 最小间隔7天
passwd -x 90 admin                  # 最大有效期90天
passwd -w 7 admin                   # 提前7天警告
passwd -i 7 admin                   # 过期后宽限7天
passwd -S admin                     # 查看密码状态
passwd -l admin                     # 锁定（密码前加!!）
passwd -u admin                     # 解锁
```

#### 5.1.3 组管理命令

```bash
# 创建组
groupadd -g 1000 admin

# 查看组信息
cat /etc/group
# 格式：组名:密码占位符:GID:成员列表（逗号分隔）

# 修改组
groupmod -n newadmin admin          # 重命名
groupmod -g 2000 admin              # 修改GID

# 删除组
groupdel admin                      # 组内不能有用户为主组

# 组成员管理
gpasswd -a user1 admin              # 添加成员
gpasswd -M user1,user2,user3 admin  # 批量设置成员（覆盖）
gpasswd -d user1 admin              # 删除成员

# 设置组管理员
gpasswd -A user1 admin              # user1可管理admin组成员

# 组密码（极少使用）
gpasswd admin                       # 设置组密码
```

### 5.2 权限管理模型

#### 5.2.1 文件权限表示

```
-rwxr-xr-x  1 root root  12345 Jan 10 10:00 script.sh
\_________/  \__/ \____/  \___/ \______________\____/
  权限      链接  属主 属组   大小      日期      文件名

权限段：- rwx r-x r-x
类型  所有者  组    其他
-: 普通文件
d: 目录
l: 软链接
b: 块设备
c: 字符设备
s: 套接字
p: 管道

权限位：
r = 4 (read)
w = 2 (write)
x = 1 (execute)
- = 0

组合示例：
rwx = 7 (4+2+1)
rw- = 6 (4+2+0)
r-x = 5 (4+0+1)
r-- = 4
```

#### 5.2.2 权限修改命令

```bash
# chmod命令
chmod u+x,g-w,o=r file              # 符号模式
# u: user(owner), g: group, o: other, a: all
# +: 添加, -: 移除, =: 设置

chmod 755 file                      # 数字模式
chmod 644 file.txt                  # 普通文件
chmod 755 script.sh                 # 可执行脚本
chmod 700 /home/user                # 家目录

chmod -R 755 /path                  # 递归修改（慎用！）

# 特殊权限
chmod u+s /usr/bin/myapp            # SUID (4)
chmod g+s /path/to/dir              # SGID (2)
chmod o+t /path/to/dir              # Sticky Bit (1)

# 数字表示特殊权限
chmod 4755 file                     # SUID rwsr-xr-x
chmod 2755 dir                      # SGID rwxr-sr-x
chmod 1755 dir                      # Sticky rwxr-xr-t

# chown命令
chown user file                     # 修改属主
chown :group file                   # 修改属组
chown user:group file               # 同时修改
chown -R user:group /path           # 递归修改

# chgrp命令（较少使用）
chgrp admin file                    # 修改属组
```

#### 5.2.3 特殊权限详解

**SUID (Set UID, 4)**：
```bash
chmod u+s /usr/bin/myapp
# 权限显示：-rwsr-xr-x
# 效果：普通用户执行时，以文件所有者的身份运行
# 典型应用：/usr/bin/passwd (suid root)
# 风险：安全漏洞可能导致权限提升
```

**SGID (Set GID, 2)**：
```bash
# 对文件：以文件所属组身份运行
chmod g+s /usr/bin/myapp
# 权限显示：-rwxr-sr-x

# 对目录：新建文件自动继承目录的属组
mkdir /shared
chgrp devgroup /shared
chmod 2775 /shared
# 所有用户在此目录创建的文件属组都是devgroup
```

**Sticky Bit (1)**：
```bash
chmod o+t /tmp
# 权限显示：drwxrwxrwt
# 效果：只有文件所有者、目录所有者、root能删除/重命名文件
# 典型应用：/tmp目录
```

#### 5.2.4 umask掩码

```bash
# 查看当前umask
umask
# 输出：0022

# 临时修改
umask 027

# 永久修改
echo "umask 027" >> /etc/profile

# umask计算
# 目录默认权限：777 - umask
# 文件默认权限：666 - umask（文件默认无执行位）

# 示例：umask=022
# 目录：777 - 022 = 755 (rwxr-xr-x)
# 文件：666 - 022 = 644 (rw-r--r--)

# umask=027
# 目录：777 - 027 = 750 (rwxr-x---)
# 文件：666 - 027 = 640 (rw-r-----)

# umask=002
# 目录：777 - 002 = 775 (rwxrwxr-x)
# 文件：666 - 002 = 664 (rw-rw-r--)
```

### 5.3 ACL访问控制列表（补充）

```bash
# 查看是否支持ACL
tune2fs -l /dev/sda1 | grep "Default mount options"
# 输出应包含：acl

# 挂载时启用ACL（ext4默认启用）
mount -o remount,acl /

# 查看ACL
getfacl /path/to/file

# 设置ACL
setfacl -m u:user1:rw file          # 给用户user1添加rw权限
setfacl -m g:group1:r file          # 给组group1添加r权限
setfacl -m o::r file                # 设置其他用户权限
setfacl -m m::rwx file              # 设置有效权限掩码

# 递归设置
setfacl -R -m u:user1:rw /path

# 删除ACL
setfacl -x u:user1 file             # 删除指定用户
setfacl -b file                     # 删除所有ACL

# 默认ACL（对目录有效，影响新建文件）
setfacl -m d:u:user1:rw /path       # 设置默认ACL
```

---

## 六、系统安全加固

### 6.1 sudo提权管理

#### 6.1.1 sudo配置详解

```bash
# 编辑sudo配置（必须使用visudo，会检查语法）
visudo

# 基本格式：用户 主机=(运行身份) 命令
root    ALL=(ALL)       ALL

# 用户别名
User_Alias ADMIN = admin,fang,jialiang
User_Alias DEV = dev1,dev2

# 主机别名
Host_Alias WEBSERVERS = www1,www2
Host_Alias DBSERVERS = db1,db2

# 命令别名
Cmnd_Alias SOFTWARE = /bin/rpm,/usr/bin/yum
Cmnd_Alias SERVICES = /usr/bin/systemctl start *, /usr/bin/systemctl stop *
Cmnd_Alias DELEGATING = /usr/sbin/visudo, /bin/chown, /bin/chmod

# 授权规则
ADMIN ALL=(ALL) ALL                      # ADMIN组可执行所有命令
DEV WEBSERVERS=(root) SOFTWARE           # DEV组可在WEBSERVERS上安装软件
%wheel ALL=(ALL) NOPASSWD:ALL            # wheel组无需密码
admin ALL=(ALL) NOPASSWD:SERVICES        # admin可免密管理服务

# 限制只能执行特定命令
user1 ALL=(root) /usr/bin/cat /var/log/messages
# user1只能cat这个文件，不能cat其他文件

# 日志记录
Defaults logfile=/var/log/sudo.log
Defaults loglinelen=0
```

#### 6.1.2 sudo命令使用

```bash
# 查看当前用户权限
sudo -l

# 以root身份执行
sudo command

# 切换到root（需配置）
sudo -i
sudo -s

# 以其他用户身份执行
sudo -u nginx /usr/sbin/nginx

# 编辑文件
sudo visudo
sudo vim /etc/shadow

# 查看sudo日志
grep sudo /var/log/secure
cat /var/log/sudo.log

# 权限不足时的错误
# user1 is not in the sudoers file.  This incident will be reported.

# 免密码时限
# Defaults timestamp_timeout=15  # 15分钟内免密
```

### 6.2 账户清理与锁定

```bash
# 删除无用系统账户（根据实际需求）
userdel adm
userdel lp
userdel sync
userdel shutdown
userdel halt
# 注意：不要删除系统关键用户如bin,daemon等

# 锁定配置文件（防篡改）
chattr +i /etc/passwd /etc/shadow /etc/group /etc/gshadow
lsattr /etc/passwd         # 查看是否加锁
# 解锁：chattr -i /etc/passwd

# 设置账户过期
usermod -e 2024-12-31 tempuser
chage -E 2024-12-31 tempuser

# 查看账户过期信息
chage -l username
```

### 6.3 密码策略强化

```bash
# 修改/etc/login.defs
cat > /etc/login.defs <<'EOF'
# 密码过期配置
PASS_MAX_DAYS   90      # 密码最长使用90天
PASS_MIN_DAYS   7       # 密码修改最小间隔7天
PASS_MIN_LEN    12      # 密码最小长度12位
PASS_WARN_AGE   14      # 提前14天警告
EOF

# 强制密码复杂度（安装libpwquality）
yum install libpwquality

# 修改/etc/security/pwquality.conf
cat > /etc/security/pwquality.conf <<EOF
minlen = 12                # 最小长度
dcredit = -1               # 至少1个数字
ucredit = -1               # 至少1个大写字母
lcredit = -1               # 至少1个小写字母
ocredit = -1               # 至少1个特殊字符
minclass = 3               # 至少3种字符类型
maxrepeat = 2              # 最多2个相同字符连续
difok = 3                  # 新密码与旧密码至少3字符不同
remember = 5               # 记住最近5个密码
EOF

# 历史密码记录
# 修改/etc/pam.d/system-auth
password    sufficient    pam_unix.so sha512 shadow nullok try_first_pass use_authtok remember=5

# 密码加密算法
authconfig --passalgo=sha512 --update
```

### 6.4 历史命令管理

```bash
# /etc/profile环境配置
cat >> /etc/profile <<EOF
# 历史命令控制
export HISTSIZE=1000              # 内存中保留历史命令数
export HISTFILESIZE=1000          # ~/.bash_history文件中保留数
export HISTTIMEFORMAT="%F %T "    # 显示时间戳
export HISTCONTROL=ignoredups     # 忽略连续重复命令
# 其他选项：
# ignorespace: 忽略以空格开头的命令
# ignoreboth: 同时忽略重复和空格开头

# 历史命令文件位置
export HISTFILE=/var/log/.hist/$(whoami)_$(date +%Y%m%d).log
# 按用户和日期分离存储

# 安全加固：命令执行后立即写入文件，不等待退出
export PROMPT_COMMAND="history -a"

# 禁止某些危险命令记录
export HISTIGNORE="ls:ll:pwd:exit"
EOF

# 立即生效
source /etc/profile

# 清理当前会话历史
history -c                  # 清空内存历史
> ~/.bash_history           # 清空历史文件
history -r                  # 重新读取历史文件

# 查看历史命令
history
history | grep "rm"

# 执行历史命令
!123                        # 执行第123条命令
!!                          # 执行上一条命令
!vim                        # 执行最近一条以vim开头的命令
```

### 6.5 自动注销与终端安全

```bash
# /etc/profile配置
cat >> /etc/profile <<EOF
# 10分钟无操作自动注销
TMOUT=600
readonly TMOUT
export TMOUT

# 限制su切换
# 只有wheel组用户可su到root
# /etc/pam.d/su
auth required pam_wheel.so use_uid

# 添加用户到wheel组
usermod -aG wheel admin
EOF

# 限制root登录终端
# 编辑/etc/securetty
# 只保留需要的终端，如：
tty1
# tty2-tty6注释掉

# 禁用Ctrl+Alt+Del重启
systemctl mask ctrl-alt-del.target
systemctl daemon-reload

# 使用StrongSWAN或OpenSSH保护远程访问
```

### 6.6 SSH服务加固

```bash
# 修改/etc/ssh/sshd_config
cat > /etc/ssh/sshd_config <<EOF
# 监听配置
Port 2222                   # 修改默认端口
ListenAddress 192.168.200.100  # 只监听内网
Protocol 2                  # 仅使用SSH v2

# 登录控制
PermitRootLogin no          # 禁止root登录
PermitEmptyPasswords no     # 禁止空密码
PasswordAuthentication yes  # 允许密码登录（可关闭，仅用密钥）
LoginGraceTime 2m           # 登录宽限时间2分钟
MaxAuthTries 3              # 最大尝试3次
MaxSessions 10              # 最大会话数

# 密钥认证
PubkeyAuthentication yes
AuthorizedKeysFile      .ssh/authorized_keys

# 安全增强
UseDNS no                   # 禁用DNS反向解析，加快连接
GSSAPIAuthentication no     # 禁用GSSAPI认证
AllowUsers admin user1 user2  # 允许登录的用户列表
AllowGroups wheel dev       # 允许登录的组
DenyUsers tempuser          # 拒绝登录的用户
DenyGroups test             # 拒绝登录的组

# 空闲断开
ClientAliveInterval 600     # 每600秒检测一次
ClientAliveCountMax 3       # 3次无响应断开

# 日志
SyslogFacility AUTHPRIV
LogLevel INFO
EOF

# 重启SSH服务
systemctl restart sshd
systemctl enable sshd

# 密钥认证配置
# 1. 客户端生成密钥对
ssh-keygen -t rsa -b 4096 -C "admin@company.com"
# 2. 复制公钥到服务器
ssh-copy-id -i ~/.ssh/id_rsa.pub admin@server_ip
# 3. 测试密钥登录
ssh admin@server_ip

# 禁用密码登录（仅密钥）
# /etc/ssh/sshd_config
PasswordAuthentication no
# 重启sshd
```

### 6.7 文件系统安全

```bash
# 锁定关键文件
chattr +i /etc/passwd /etc/shadow /etc/group /etc/gshadow
chattr +i /etc/fstab /etc/inittab /etc/crontab

# 系统日志保护
chattr +a /var/log/messages
chattr +a /var/log/secure
chattr +a /var/log/maillog

# 检查文件完整性
# 安装AIDE或Tripwire
yum install aide
aide --init
mv /var/lib/aide/aide.db.new.gz /var/lib/aide/aide.db.gz
aide --check

# 使用auditd审计
systemctl start auditd
auditctl -w /etc/passwd -p wa -k passwd_changes
ausearch -k passwd_changes
```

---

## 七、服务管理（NGINX/MySQL/Redis/DNS）

### 7.1 NGINX服务配置与管理

#### 7.1.1 NGINX核心优势
1. **轻量级高并发**：异步非阻塞架构，C10K问题解决方案
2. **反向代理**：隐藏后端服务器，实现负载均衡
3. **静态资源处理**：高效处理静态文件，CDN核心组件
4. **模块化设计**：核心+插件架构，灵活扩展
5. **低内存占用**：1万个非活动连接仅消耗2.5MB内存

#### 7.1.2 源码安装步骤（生产环境推荐）

```bash
# 1. 安装依赖包
yum install -y gcc gcc-c++ make automake autoconf libtool \
pcre pcre-devel zlib zlib-devel openssl openssl-devel

# 2. 创建运行用户
useradd -s /sbin/nologin -M nginx

# 3. 下载源码
cd /usr/src
wget http://nginx.org/download/nginx-1.24.0.tar.gz
tar zxf nginx-1.24.0.tar.gz
cd nginx-1.24.0

# 4. 配置编译参数
./configure \
--prefix=/usr/local/nginx \
--user=nginx \
--group=nginx \
--with-http_ssl_module \
--with-http_v2_module \
--with-http_realip_module \
--with-http_stub_status_module \
--with-http_gzip_static_module \
--with-pcre \
--with-stream \
--with-stream_ssl_module \
--with-stream_realip_module

# 5. 编译安装
make -j $(nproc) && make install

# 6. 创建软链接
ln -s /usr/local/nginx/sbin/nginx /usr/local/sbin/

# 7. 验证安装
nginx -t
# 输出：
# nginx: the configuration file /usr/local/nginx/conf/nginx.conf syntax is ok
# nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful

# 8. 创建systemd服务
cat > /etc/systemd/system/nginx.service <<EOF
[Unit]
Description=The NGINX HTTP and reverse proxy server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t
ExecStart=/usr/local/nginx/sbin/nginx
ExecReload=/bin/kill -s HUP \$MAINPID
ExecStop=/bin/kill -s QUIT \$MAINPID
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF

# 9. 启动服务
systemctl daemon-reload
systemctl start nginx
systemctl enable nginx
systemctl status nginx

# 10. 配置防火墙
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --reload

# 11. 浏览器访问测试
http://server_ip
# 应显示"Welcome to nginx!"
```

#### 7.1.3 NGINX配置结构

```nginx
# 全局配置段
user nginx;
worker_processes auto;  # 根据CPU核心数自动设置
worker_cpu_affinity auto;  # CPU亲和性
error_log /usr/local/nginx/logs/error.log notice;
pid /usr/local/nginx/logs/nginx.pid;
worker_rlimit_nofile 65535;  # 每个worker最大文件句柄

# events配置段
events {
    use epoll;  # Linux高效模型
    worker_connections 65535;  # 每个worker最大连接数
    multi_accept on;  # 批量接受新连接
}

# http配置段（核心）
http {
    include mime.types;  # 文件类型映射
    default_type application/octet-stream;

    # 日志格式
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    access_log /usr/local/nginx/logs/access.log main;

    # 性能优化
    sendfile on;  # 零拷贝传输
    tcp_nopush on;  # 合并数据包
    tcp_nodelay on;  # 禁用Nagle算法
    keepalive_timeout 65;
    keepalive_requests 100;
    reset_timedout_connection on;

    # 压缩配置
    gzip on;
    gzip_min_length 1k;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml text/javascript application/json application/javascript application/xml+rss application/rss+xml image/svg+xml;

    # 虚拟主机配置
    server {
        listen 80;
        server_name www.linux.com;
        root /data/www;
        index index.html index.htm;

        # 访问日志
        access_log /usr/local/nginx/logs/www.access.log main;

        # 错误页面
        error_page 404 /404.html;
        error_page 500 502 503 504 /50x.html;

        #  location匹配
        location / {
            try_files $uri $uri/ @rewrite;
        }

        location @rewrite {
            rewrite ^/(.*)$ /index.php?q=$1;
        }

        # 静态文件缓存
        location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
            expires 1d;
            add_header Cache-Control "public, immutable";
        }

        # 拒绝访问
        location ~ /\. {
            deny all;
        }

        # 反向代理
        location /api/ {
            proxy_pass http://backend_server;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # 目录别名
        location /doc/ {
            alias /usr/share/doc/;
            autoindex on;
        }

        # 访问控制
        location /admin/ {
            auth_basic "Admin Area";
            auth_basic_user_file /usr/local/nginx/conf/htpasswd;
            allow 192.168.200.0/24;
            deny all;
        }
    }

    # 多个虚拟主机
    server {
        listen 80;
        server_name blog.linux.com;
        root /data/blog;
        # ...
    }
}
```

#### 7.1.4 location匹配规则（重要）

```nginx
# 匹配优先级（从高到低）：
# 1. = 精确匹配
location = / {
    # 只匹配"/"
}

# 2. ^~ 前缀匹配（优先于正则）
location ^~ /images/ {
    # 匹配以/images/开头的请求，停止后续正则匹配
}

# 3. ~ 区分大小写正则
location ~ \.php$ {
    # 匹配以.php结尾的请求
}

# 4. ~* 不区分大小写正则
location ~* \.(jpg|png|gif)$ {
    # 匹配图片文件，不区分大小写
}

# 5. / 通用匹配
location / {
    # 匹配所有请求
}

# 匹配示例：
# URI: /images/logo.png
# 优先匹配 ^~ /images/，不检查正则

# URI: /index.php
# 精确匹配失败，前缀匹配失败，正则匹配 ~ \.php$ 成功

# 实际配置建议：
location = / {
    # 首页
}

location ^~ /static/ {
    # 静态资源
}

location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
    # 静态文件缓存
}

location ~ \.php$ {
    # PHP处理
}

location / {
    # 默认处理
}
```

#### 7.1.5 NGINX性能优化

```nginx
# worker进程数
worker_processes auto;  # 或设置为CPU核心数
# worker_cpu_affinity auto;  # CPU亲和性

# 最大连接数计算
# 最大连接数 = worker_processes * worker_connections
# 如果启用反向代理，需除以2（客户端和后端各一个连接）

# 文件句柄限制
# /etc/security/limits.conf
nginx soft nofile 65535
nginx hard nofile 65535

# /etc/systemd/system/nginx.service
[Service]
LimitNOFILE=65535

# 内核参数优化
# /etc/sysctl.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.tcp_fin_timeout = 30
net.ipv4.tcp_max_syn_backlog = 8192

# 零拷贝优化
sendfile on;
sendfile_max_chunk 512k;

# TCP优化
tcp_nopush on;
tcp_nodelay on;

# 连接复用
keepalive_timeout 65;
keepalive_requests 100;

# Gzip优化
gzip on;
gzip_min_length 1k;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript;

# 缓存优化
open_file_cache max=65535 inactive=60s;
open_file_cache_valid 80s;
open_file_cache_min_uses 1;
```

#### 7.1.6 日志管理与切割

```bash
# 1. 日志切割脚本（按天）
cat > /usr/local/nginx/sbin/logrotate.sh <<'EOF'
#!/bin/bash
LOG_DIR="/usr/local/nginx/logs"
YESTERDAY=$(date -d "yesterday" +%Y%m%d)

# 移动日志
mv ${LOG_DIR}/access.log ${LOG_DIR}/access.log-${YESTERDAY}
mv ${LOG_DIR}/error.log ${LOG_DIR}/error.log-${YESTERDAY}

# 重新打开日志
kill -USR1 $(cat ${LOG_DIR}/nginx.pid)

# 压缩旧日志（保留30天）
find ${LOG_DIR} -name "*.log-20*" -mtime +30 -exec gzip {} \;
EOF

chmod +x /usr/local/nginx/sbin/logrotate.sh

# 2. 配置crontab
crontab -e
0 0 * * * /usr/local/nginx/sbin/logrotate.sh

# 3. 日志分析示例
# 统计IP访问次数
awk '{print $1}' /usr/local/nginx/logs/access.log | sort | uniq -c | sort -nr | head -20

# 统计URL访问次数
awk '{print $7}' /usr/local/nginx/logs/access.log | sort | uniq -c | sort -nr | head -20

# 统计响应时间超过1秒的请求
awk '$NF > 1 {print $0}' /usr/local/nginx/logs/access.log
```

### 7.2 MySQL数据库管理

#### 7.2.1 MySQL SQL语句分类

```sql
-- 1. DDL (Data Definition Language) - 数据定义语言
-- 创建、修改、删除数据库对象（表、索引、视图）
CREATE TABLE users (id INT PRIMARY KEY, name VARCHAR(50));
CREATE INDEX idx_name ON users(name);
CREATE VIEW v_users AS SELECT id, name FROM users WHERE active=1;
ALTER TABLE users ADD COLUMN email VARCHAR(100);
DROP TABLE users;
DROP INDEX idx_name;

-- 2. DML (Data Manipulation Language) - 数据操作语言
-- 插入、更新、删除数据
INSERT INTO users (id, name) VALUES (1, 'admin');
UPDATE users SET name='root' WHERE id=1;
DELETE FROM users WHERE id=1;
TRUNCATE TABLE users;  -- 快速清空表

-- 3. DQL (Data Query Language) - 数据查询语言
-- 查询数据（最常用）
SELECT * FROM users;
SELECT id, name FROM users WHERE id > 10 ORDER BY id DESC;
SELECT COUNT(*) FROM users;
SELECT DISTINCT name FROM users;

-- 4. DCL (Data Control Language) - 数据控制语言
-- 权限管理
GRANT SELECT, INSERT ON db1.* TO 'user1'@'localhost';
REVOKE INSERT ON db1.* FROM 'user1'@'localhost';
FLUSH PRIVILEGES;

-- 5. TCL (Transaction Control Language) - 事务控制语言
START TRANSACTION;  -- 或 BEGIN;
INSERT INTO users VALUES (2, 'test');
UPDATE users SET name='temp' WHERE id=2;
COMMIT;  -- 提交
-- 或
ROLLBACK;  -- 回滚
```

#### 7.2.2 索引管理

```sql
-- 创建索引
CREATE INDEX idx_salary ON employees(salary);          -- 普通索引
CREATE UNIQUE INDEX idx_empid ON employees(emp_id);    -- 唯一索引
CREATE FULLTEXT INDEX idx_content ON articles(content);-- 全文索引

-- 查看索引
SHOW INDEX FROM employees;
SHOW INDEX FROM employees \G;  -- 垂直显示

-- 删除索引
DROP INDEX idx_salary ON employees;
ALTER TABLE employees DROP INDEX idx_salary;

-- 删除主键索引
ALTER TABLE employees DROP PRIMARY KEY;

-- 创建主键索引
ALTER TABLE employees ADD PRIMARY KEY (emp_id);

-- 复合索引
CREATE INDEX idx_name_age ON employees(name, age);

-- 索引使用建议
-- 1.  WHERE、ORDER BY、GROUP BY字段建立索引
-- 2.  区分度低的字段（如性别）不适合建立索引
-- 3.  索引会占用额外空间，降低写入速度
-- 4.  定期使用OPTIMIZE TABLE优化索引
```

#### 7.2.3 事务管理

**事务ACID特性**：
- **原子性（Atomicity）**：事务是不可分割的工作单位，要么全做，要么全不做
- **一致性（Consistency）**：事务前后，数据库从一个一致状态变到另一个一致状态
- **隔离性（Isolation）**：事务之间互不干扰，隔离级别控制可见性
- **持久性（Durability）**：事务提交后，对数据的修改是永久性的

```sql
-- 查看当前事务隔离级别
SHOW VARIABLES LIKE 'transaction_isolation';
-- 或
SELECT @@transaction_isolation;

-- MySQL四种隔离级别
-- READ UNCOMMITTED（读未提交）：最低级别，可能脏读
-- READ COMMITTED（读已提交）：Oracle默认，可能不可重复读
-- REPEATABLE READ（可重复读）：MySQL默认，可能幻读
-- SERIALIZABLE（可串行化）：最高级别，完全隔离

-- 事务操作
BEGIN;  -- 或 START TRANSACTION;
INSERT INTO accounts VALUES (1, 1000);
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;  -- 提交

-- 回滚示例
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- 发现错误
ROLLBACK;  -- 撤销所有操作

-- 保存点
BEGIN;
INSERT INTO users VALUES (1, 'user1');
SAVEPOINT sp1;
INSERT INTO users VALUES (2, 'user2');
ROLLBACK TO sp1;  -- 回滚到保存点
COMMIT;

-- 自动提交控制
SET autocommit = 0;  -- 关闭自动提交
SET autocommit = 1;  -- 开启自动提交（默认）

-- MySQL默认引擎对比
-- MyISAM：不支持事务，表级锁，读快写慢，支持全文索引
-- InnoDB：支持事务，行级锁，支持外键，5.6+支持全文索引
```

#### 7.2.4 备份与恢复

**备份策略**：
1. **完全备份**：备份所有数据，恢复简单，占用空间大
2. **增量备份**：备份自上次备份后变化的数据，节省空间，恢复复杂
3. **差异备份**：备份自上次完全备份后变化的数据，折中方案

```bash
# 1. 物理冷备份（需停库）
systemctl stop mysqld
tar zcf /backup/mysql-$(date +%Y%m%d).tar.gz /var/lib/mysql
systemctl start mysqld

# 2. 逻辑备份（mysqldump，推荐）
# 备份单个库
mysqldump -u root -p'password' db1 > /backup/db1-$(date +%Y%m%d).sql

# 备份多个库
mysqldump -u root -p'password' --databases db1 db2 > /backup/dbs.sql

# 备份所有库
mysqldump -u root -p'password' --all-databases > /backup/all-$(date +%Y%m%d).sql

# 仅备份表结构
mysqldump -u root -p'password' -d db1 > /backup/db1-structure.sql

# 备份并压缩
mysqldump -u root -p'password' db1 | gzip > /backup/db1.sql.gz

# 3. 恢复
# 方法1：mysql命令
mysql -u root -p'password' db1 < /backup/db1.sql

# 方法2：source（在MySQL客户端）
mysql> USE db1;
mysql> SOURCE /backup/db1.sql;

# 4. 增量备份（基于binlog）
# 启用binlog
cat >> /etc/my.cnf <<EOF
[mysqld]
log_bin = /var/log/mysql/mysql-bin
binlog_format = MIXED
expire_logs_days = 7
EOF

# 查看binlog
mysqlbinlog /var/log/mysql/mysql-bin.000001 | mysql -u root -p

# 5. 自动化备份脚本
cat > /usr/local/bin/mysql_backup.sh <<'EOF'
#!/bin/bash
BACKUP_DIR="/backup/mysql"
DATE=$(date +%Y%m%d_%H%M%S)
USER="root"
PASS="your_password"

# 创建目录
mkdir -p $BACKUP_DIR

# 备份所有数据库
mysqldump -u $USER -p$PASS --all-databases | gzip > $BACKUP_DIR/all-$DATE.sql.gz

# 删除30天前的备份
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete

# 记录日志
echo "$DATE: Backup completed" >> /var/log/mysql_backup.log
EOF

chmod +x /usr/local/bin/mysql_backup.sh
crontab -e
0 2 * * * /usr/local/bin/mysql_backup.sh
```

### 7.3 Redis缓存系统

#### 7.3.1 Redis核心优势
1. **极高性能**：纯内存操作，11万/秒读，8万/秒写
2. **丰富数据结构**：String, Hash, List, Set, Sorted Set
3. **持久化**：RDB快照+AOF日志，支持数据恢复
4. **原子性**：所有操作单线程，保证原子性
5. **高可用**：主从复制+哨兵+集群

#### 7.3.2 Redis安装部署

```bash
# 1. 编译安装
cd /usr/src
wget https://download.redis.io/releases/redis-6.2.11.tar.gz
tar zxf redis-6.2.11.tar.gz
cd redis-6.2.11

# 2. 编译
make MALLOC=libc
make install PREFIX=/usr/local/redis

# 3. 创建配置目录
mkdir -p /usr/local/redis/{conf,logs,data,run}

# 4. 初始化配置文件
cat > /usr/local/redis/conf/redis.conf <<EOF
# 绑定地址
bind 0.0.0.0

# 端口
port 6379

# 守护进程
daemonize yes

# PID文件
pidfile /usr/local/redis/run/redis_6379.pid

# 日志文件
logfile /usr/local/redis/logs/redis.log

# 数据目录
dir /usr/local/redis/data

# 数据文件
dbfilename dump.rdb

# RDB持久化策略
save 900 1
save 300 10
save 60 10000

# AOF持久化
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec

# 密码验证
requirepass your_strong_password

# 最大内存
maxmemory 1gb
maxmemory-policy allkeys-lru

# 慢查询日志
slowlog-log-slower-than 10000
slowlog-max-len 128

# 限制连接数
maxclients 10000

# 超时设置
timeout 300
tcp-keepalive 60
EOF

# 5. 创建启动脚本
cat > /etc/systemd/system/redis.service <<EOF
[Unit]
Description=Redis In-Memory Data Store
After=network.target

[Service]
Type=forking
User=redis
Group=redis
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/conf/redis.conf
ExecStop=/usr/local/redis/bin/redis-cli shutdown
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

# 6. 创建用户并授权
useradd -s /sbin/nologin -M redis
chown -R redis:redis /usr/local/redis

# 7. 启动服务
systemctl daemon-reload
systemctl start redis
systemctl enable redis

# 8. 客户端测试
/usr/local/redis/bin/redis-cli
127.0.0.1:6379> AUTH your_strong_password
OK
127.0.0.1:6379> SET name "redis"
OK
127.0.0.1:6379> GET name
"redis"
127.0.0.1:6379> INFO
```

#### 7.3.3 Redis主从复制

```bash
# 1. 主节点配置（192.168.200.100）
# redis.conf
bind 0.0.0.0
port 6379
requirepass MasterPass123

# 2. 从节点1配置（192.168.200.101）
# redis.conf
bind 0.0.0.0
port 6379
requirepass SlavePass123
masterauth MasterPass123
slaveof 192.168.200.100 6379

# 3. 从节点2配置（192.168.200.102）
# 同上

# 4. 验证主从
# 主节点执行
redis-cli -a MasterPass123
127.0.0.1:6379> INFO replication
# 输出：
# role:master
# connected_slaves:2
# slave0:ip=192.168.200.101,port=6379,state=online,offset=12345,lag=0
# slave1:ip=192.168.200.102,port=6379,state=online,offset=12345,lag=0

# 从节点执行
127.0.0.1:6379> INFO replication
# 输出：
# role:slave
# master_host:192.168.200.100
# master_port:6379
# master_link_status:up
```

#### 7.3.4 Redis哨兵集群

```bash
# 1. 创建Sentinel配置
cat > /usr/local/redis/conf/sentinel.conf <<EOF
# 端口
port 26379

# 守护进程
daemonize yes

# PID文件
pidfile /usr/local/redis/run/sentinel_26379.pid

# 日志
logfile /usr/local/redis/logs/sentinel.log

# 监控主节点
sentinel monitor mymaster 192.168.200.100 6379 2
# mymaster: 主节点名称
# 192.168.200.100: 主节点IP
# 6379: 主节点端口
# 2: 最少2个sentinel同意才判定故障

# 密码
sentinel auth-pass mymaster MasterPass123

# 主观下线时间（毫秒）
sentinel down-after-milliseconds mymaster 3000

# 故障转移超时
sentinel failover-timeout mymaster 180000

# 故障转移后，从节点并行同步数量
sentinel parallel-syncs mymaster 1

# 保护模式
protected-mode no
EOF

# 2. 启动Sentinel
/usr/local/redis/bin/redis-sentinel /usr/local/redis/conf/sentinel.conf

# 3. 查看Sentinel状态
redis-cli -p 26379
127.0.0.1:26379> sentinel masters
127.0.0.1:26379> sentinel slaves mymaster
127.0.0.1:26379> sentinel sentinels mymaster

# 4. 故障转移测试
# 在主节点执行
redis-cli -a MasterPass123 shutdown
# 观察sentinel日志，自动选举新主节点

# 5. 客户端连接
# 使用哨兵获取当前主节点
redis-cli -p 26379 sentinel get-master-addr-by-name mymaster
# 输出：1) "192.168.200.101" 2) "6379"
```

#### 7.3.5 Redis缓存问题与解决方案

**缓存击穿**：
- **原因**：热点key过期瞬间，大量请求打到数据库
- **解决方案**：
  ```python
  # 互斥锁方案
  import redis
  import threading
  
  def get_data(key):
      value = r.get(key)
      if value is None:
          # 尝试获取锁
          lock = r.set(lock_key, 1, ex=10, nx=True)
          if lock:
              # 从数据库加载
              value = load_from_db(key)
              r.set(key, value, ex=3600)
              r.delete(lock_key)
          else:
              # 等待并重试
              time.sleep(0.1)
              return get_data(key)
      return value
  ```
  
  ```nginx
  # Nginx配置：永不过期
  location /api/hot-data {
      proxy_cache_valid 200 1d;
      proxy_cache_use_stale error timeout updating;
  }
  ```

**缓存雪崩**：
- **原因**：大量key同时过期
- **解决方案**：
  ```bash
  # Redis配置：随机过期时间
  # 在设置缓存时添加随机值
  EXPIRE key $(($RANDOM % 300 + 3600))  # 3600-3900秒之间
  ```

**缓存穿透**：
- **原因**：查询不存在的数据，绕过缓存
- **解决方案**：
  ```python
  # 布隆过滤器
  from pybloom_live import BloomFilter
  
  bf = BloomFilter(capacity=1000000, error_rate=0.001)
  # 加载所有有效key到布隆过滤器
  
  def get_data(key):
      if key not in bf:
          return None  # 直接返回，不查询数据库
      value = r.get(key)
      if value is None:
          value = load_from_db(key)
          if value is None:
              r.set(key, "NULL", ex=60)  # 缓存空结果
      return value
  ```

#### 7.3.6 Redis淘汰策略

```bash
# 查看当前策略
redis-cli config get maxmemory-policy

# 八种淘汰策略
noeviction          # 不淘汰，内存满后写入报错
allkeys-lru         # 所有key中淘汰最近最少使用（推荐）
allkeys-lfu         # 所有key中淘汰最不常用
allkeys-random      # 随机淘汰
volatile-lru        # 过期key中淘汰LRU（需要设置TTL）
volatile-lfu        # 过期key中淘汰LFU
volatile-random     # 过期key中随机淘汰
volatile-ttl        # 淘汰即将过期的key

# 配置淘汰策略
maxmemory-policy allkeys-lru
maxmemory-samples 5  # 采样数量，越大越精确

# 监控内存
redis-cli info memory
# used_memory_human:1.23G
# used_memory_peak_human:1.50G
```

#### 7.3.7 Redis持久化机制

```bash
# 1. RDB（Redis Database）
# 优点：紧凑、恢复快
# 缺点：可能丢失数据

save 900 1        # 900秒内有1次修改就触发
save 300 10       # 300秒内有10次修改
save 60 10000     # 60秒内有10000次修改
dbfilename dump.rdb
dir /usr/local/redis/data

# 手动触发
SAVE              # 同步保存（阻塞）
BGSAVE            # 异步保存（fork子进程）

# 2. AOF（Append Only File）
# 优点：数据安全，最多丢失1秒
# 缺点：文件大，恢复慢

appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec    # 每秒同步
# appendfsync always    # 每次操作同步（最安全，最慢）
# appendfsync no        # 由操作系统决定（最快，不安全）

# AOF重写
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 混合持久化（Redis 4.0+）
aof-use-rdb-preamble yes
```

### 7.4 DNS服务配置（BIND）

#### 7.4.1 主域名服务器配置

```bash
# 1. 安装BIND
yum install bind bind-utils

# 2. 修改主配置
cat > /etc/named.conf <<EOF
options {
    listen-on port 53 { 192.168.200.100; };
    listen-on-v6 port 53 { ::1; };
    directory   "/var/named";
    dump-file   "/var/named/data/cache_dump.db";
    statistics-file "/var/named/data/named_stats.txt";
    memstatistics-file "/var/named/data/named_mem_stats.txt";
    recursing-file  "/var/named/data/named.recursing";
    secroots-file   "/var/named/data/named.secroots";
    allow-query     { 192.168.200.0/24; localhost; };
    recursion yes;
    dnssec-enable yes;
    dnssec-validation yes;
    bindkeys-file "/etc/named.root.key";
    managed-keys-directory "/var/named/dynamic";
    pid-file "/run/named/named.pid";
    session-keyfile "/run/named/session.key";
};

logging {
        channel default_debug {
                file "data/named.run";
                severity dynamic;
        };
};

zone "." IN {
    type hint;
    file "named.ca";
};

zone "linux.com" IN {
    type master;
    file "linux.com.zone";
    allow-transfer { 192.168.200.193; };
    allow-update { none; };
};

zone "200.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.200.zone";
    allow-transfer { 192.168.200.193; };
    allow-update { none; };
};

include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
EOF

# 3. 创建正向解析区域文件
cat > /var/named/linux.com.zone <<EOF
\$TTL 86400
@       IN      SOA     ns1.linux.com. admin.linux.com. (
                        2024011501      ; serial
                        3600            ; refresh (1小时)
                        600             ; retry (10分钟)
                        86400           ; expire (1天)
                        86400 )         ; minimum (1天)

        IN      NS      ns1.linux.com.
        IN      NS      ns2.linux.com.
        IN      MX  10  mail.linux.com.

ns1     IN      A       192.168.200.100
ns2     IN      A       192.168.200.193
www     IN      A       192.168.200.100
mail    IN      A       192.168.200.111
ftp     IN      CNAME   www
*       IN      A       192.168.200.100  ; 泛解析
EOF

# 4. 创建反向解析区域文件
cat > /var/named/192.168.200.zone <<EOF
\$TTL 86400
@       IN      SOA     ns1.linux.com. admin.linux.com. (
                        2024011501
                        3600
                        600
                        86400
                        86400 )

        IN      NS      ns1.linux.com.
        IN      NS      ns2.linux.com.

100     IN      PTR     ns1.linux.com.
193     IN      PTR     ns2.linux.com.
100     IN      PTR     www.linux.com.
111     IN      PTR     mail.linux.com.
EOF

# 5. 修改权限
chown named:named /var/named/linux.com.zone
chown named:named /var/named/192.168.200.zone
chmod 640 /var/named/linux.com.zone
chmod 640 /var/named/192.168.200.zone

# 6. 检查配置
named-checkconf /etc/named.conf
named-checkzone linux.com /var/named/linux.com.zone
named-checkzone 200.168.192.in-addr.arpa /var/named/192.168.200.zone

# 7. 启动服务
systemctl start named
systemctl enable named

# 8. 客户端测试
# 修改客户端DNS
cat > /etc/resolv.conf <<EOF
nameserver 192.168.200.100
EOF

# 测试解析
nslookup www.linux.com
dig www.linux.com
host www.linux.com

# 反向解析测试
nslookup 192.168.200.100
dig -x 192.168.200.100
```

#### 7.4.2 从域名服务器配置

```bash
# 1. 在从服务器上安装BIND
yum install bind bind-utils

# 2. 修改named.conf
cat > /etc/named.conf <<EOF
options {
    listen-on port 53 { 192.168.200.193; };
    allow-query { 192.168.200.0/24; };
    # ... 其他配置同主服务器
};

zone "linux.com" IN {
    type slave;
    masters { 192.168.200.100; };
    file "slaves/linux.com.zone";
};

zone "200.168.192.in-addr.arpa" IN {
    type slave;
    masters { 192.168.200.100; };
    file "slaves/192.168.200.zone";
};
EOF

# 3. 启动从服务器
systemctl start named
systemctl enable named

# 4. 验证
# 在主服务器修改区域文件，更新serial
# 执行 rndc reload 或 systemctl reload named
# 观察从服务器日志
tail -f /var/log/messages | grep named
# 应显示：transfer of 'linux.com/IN' from 192.168.200.100#53: Transfer completed: 1 messages

# 5. 测试从服务器解析
dig @192.168.200.193 www.linux.com
```

#### 7.4.3 DNS工具使用

```bash
# nslookup（交互式）
nslookup
> server 192.168.200.100
> www.linux.com
> set type=mx
> linux.com
> exit

# dig（详细信息）
dig www.linux.com
dig +short www.linux.com          # 简短输出
dig +trace www.linux.com          # 跟踪解析过程
dig +nocmd www.linux.com          # 不显示命令
dig +noall +answer www.linux.com  # 只显示答案

# host
host www.linux.com
host -t mx linux.com              # 查询MX记录
host -t a www.linux.com           # 查询A记录
host -t ptr 192.168.200.100       # 反向解析

# rndc（BIND管理工具）
rndc status                       # 查看状态
rndc reload                       # 重载配置
rndc flush                        # 清空缓存
rndc sync                         # 同步区域文件
```

---

## 八、系统监控与故障排查

### 8.1 系统性能监控

```bash
# CPU监控
top                                  # 实时CPU使用
htop                                 # 更友好的top
mpstat 1                             # 每个CPU核心统计
vmstat 1                             # CPU、内存、I/O综合
sar -u 1 10                          # CPU历史数据

# 内存监控
free -h                              # 内存使用
vmstat -s | grep memory              # 内存统计
pmap -x <pid>                        # 进程内存映射
smem                                 # 内存使用分析工具

# 磁盘监控
df -hT                               # 磁盘空间
du -sh /*                            # 目录大小
iostat -x 1                          # 磁盘I/O统计
iotop                                # I/O进程排名
dstat                                # 综合I/O监控

# 网络监控
iftop                                # 实时流量
nethogs                              # 按进程显示流量
ss -s                                # 套接字统计
netstat -i                           # 网络接口统计
cat /proc/net/dev                    # 原始网络数据

# 系统负载
uptime                               # 1、5、15分钟负载
cat /proc/loadavg                    # 详细负载信息

# 进程监控
ps aux --sort=-%cpu                  # 按CPU排序
ps aux --sort=-%mem                  # 按内存排序
pidstat 1                            # 进程资源使用

# 综合监控工具
dstat -cdngy 1                       # CPU、磁盘、网络、系统、分页
glances                              # 跨平台监控（需安装）
nmon                                 # AIX/Linux性能分析
```

### 8.2 日志管理

#### 8.2.1 日志文件说明

```bash
# 核心日志
/var/log/messages                    # 系统通用日志（内核、服务）
/var/log/secure                      # 安全相关（SSH、su、sudo）
/var/log/cron                        # 计划任务日志
/var/log/dmesg                       # 内核环形缓冲区
/var/log/boot.log                    # 系统启动日志
/var/log/audit/audit.log             # SELinux审计日志

# 应用日志
/var/log/nginx/access.log            # Nginx访问日志
/var/log/nginx/error.log             # Nginx错误日志
/var/log/mysql/mysqld.log            # MySQL日志
/var/log/redis/redis.log             # Redis日志

# 登录日志
/var/log/wtmp                        # 登录历史（二进制）
/var/log/btmp                        # 失败登录尝试（二进制）
/var/log/lastlog                     # 最近登录（二进制）
/var/log/utmp                        # 当前登录用户（二进制）

# 查看工具
last                                 # 读取wtmp
lastb                                # 读取btmp
lastlog                              # 读取lastlog
who                                  # 读取utmp
w                                    # 增强版who
```

#### 8.2.2 syslog与rsyslog

```bash
# rsyslog配置
cat > /etc/rsyslog.conf <<EOF
# 模块加载
$ModLoad imuxsock
$ModLoad imjournal

# 日志级别
# emerg, alert, crit, err, warning, notice, info, debug
# 格式：facility.priority   action

# 内核日志
kern.*                          /var/log/kern.log

# 邮件日志
mail.*                          /var/log/maillog

# 认证日志
authpriv.*                      /var/log/secure

# 计划任务
cron.*                          /var/log/cron

# 本地应用
local0.*                        /var/log/app1.log
local1.*                        /var/log/app2.log

# 远程日志服务器
*.* @192.168.200.100:514        # UDP
*.* @@192.168.200.100:514       # TCP

# 模板
$template CustomFormat,"%timestamp% %hostname% %syslogtag%%msg%\n"
$ActionFileDefaultTemplate CustomFormat
EOF

systemctl restart rsyslog

# 日志级别测试
logger -p local0.info "This is a test message"
tail -f /var/log/app1.log
```

#### 8.2.3 journalctl（systemd日志）

```bash
# 查看所有日志
journalctl

# 实时跟踪
journalctl -f

# 按时间范围
journalctl --since "2024-01-10 10:00:00"
journalctl --since "1 hour ago"
journalctl --until "2024-01-10 12:00:00"

# 按服务
journalctl -u nginx.service
journalctl -u nginx.service --since today

# 按优先级
journalctl -p err               # 错误级别及以上
# 0: emerg, 1: alert, 2: crit, 3: err, 4: warning, 5: notice, 6: info, 7: debug

# 按进程ID
journalctl _PID=12345

# 按用户
journalctl _UID=1000

# 查看占用空间
journalctl --disk-usage

# 清理旧日志
journalctl --vacuum-size=100M   # 保留100MB
journalctl --vacuum-time=7d     # 保留7天
journalctl --vacuum-files=5     # 保留5个文件
```

### 8.3 故障排查方法论

#### 8.3.1 故障排查步骤

1. **收集信息**：症状、日志、监控数据、最近变更
2. **定位范围**：网络/系统/应用/数据库
3. **分析原因**：日志分析、进程跟踪、资源检查
4. **制定方案**：临时解决+长期优化
5. **实施验证**：测试验证，监控观察
6. **文档记录**：记录过程，更新知识库

#### 8.3.2 常见问题排查

**场景1：系统无法启动**

```bash
# 进入救援模式
# 1. 使用安装盘启动，选择Troubleshooting → Rescue
# 2. 检查文件系统
fsck -y /dev/sda1

# 3. 检查/etc/fstab
cat /mnt/sysimage/etc/fstab
# 确认UUID、挂载点正确

# 4. 检查GRUB
chroot /mnt/sysimage
grub2-install /dev/sda
grub2-mkconfig -o /boot/grub2/grub.cfg

# 5. 检查SELinux
# 临时禁用
echo "SELINUX=disabled" > /mnt/sysimage/etc/selinux/config
touch /mnt/sysimage/.autorelabel

# 6. 重启
exit
reboot
```

**场景2：磁盘空间不足**

```bash
# 1. 检查空间
df -h
# 发现某分区使用率100%

# 2. 查找大文件
du -sh /* 2>/dev/null | sort -rh | head -10
# 或
ncdu /  # 交互式分析（需安装）

# 3. 清理日志
journalctl --vacuum-size=100M
> /var/log/messages
logrotate -f /etc/logrotate.conf

# 4. 清理临时文件
find /tmp -type f -atime +7 -delete
find /var/tmp -type f -atime +30 -delete

# 5. 清理包缓存
yum clean all
apt-get clean

# 6. 查找大文件
find / -type f -size +100M -exec ls -lh {} \;
# 检查是否为日志文件，可考虑压缩或删除
```

**场景3：内存不足（OOM）**

```bash
# 1. 查看内存
free -h
vmstat 1

# 2. 查看进程内存
ps aux --sort=-%mem | head
# 或使用smem
smem

# 3. 检查OOM Killer日志
dmesg | grep -i "oom-killer"
journalctl -k | grep -i "oom"

# 4. 调整OOM分数
# OOM优先级：-1000到1000，值越高越容易被杀
echo -500 > /proc/$PID/oom_score_adj  # 保护关键进程

# 5. 释放内存
sync
echo 3 > /proc/sys/vm/drop_caches

# 6. 扩展swap（应急）
dd if=/dev/zero of=/swapfile bs=1G count=4
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo "/swapfile swap swap defaults 0 0" >> /etc/fstab
```

**场景4：网络不通**

```bash
# 1. 检查物理链路
ip link show                   # 查看接口状态
ethtool eth0                   # 查看网卡状态
mii-tool eth0                  # 快速检查

# 2. 检查IP配置
ip addr show
route -n
# 确认IP、网关、子网掩码正确

# 3. 检查连通性
ping 127.0.0.1                 # 本地回环
ping 192.168.200.1             # 网关
ping 8.8.8.8                   # 外网
ping www.baidu.com             # DNS

# 4. 检查DNS
cat /etc/resolv.conf
nslookup www.baidu.com
dig www.baidu.com

# 5. 检查防火墙
iptables -L -n
firewall-cmd --list-all
# 临时禁用测试
systemctl stop firewalld
iptables -F

# 6. 检查路由
traceroute www.baidu.com
mtr www.baidu.com              # 更强大的traceroute

# 7. 检查端口监听
ss -tuln
netstat -tuln
# 确认服务监听在正确地址
```

**场景5：CPU使用率100%**

```bash
# 1. 查看CPU占用
top
# 按P按CPU排序
# 按H查看线程

# 2. 查看进程详情
pidstat 1 -p <PID>

# 3. 生成火焰图（高级）
# 安装perf
perf record -ag -- sleep 30
perf script | flamegraph.pl > cpu.svg

# 4. 查看系统调用
strace -p <PID> -c              # 统计系统调用
strace -p <PID> -o trace.log    # 详细日志

# 5. GDB调试
gdb -p <PID>
(gdb) thread apply all bt       # 查看所有线程堆栈

# 6. 查看Java进程
jstack <PID> > jstack.log
jvisualvm                       # GUI工具
```

### 8.4 核心转储分析

```bash
# 1. 启用core dump
ulimit -c unlimited
echo "/corefiles/core-%e-%p-%t" > /proc/sys/kernel/core_pattern
mkdir /corefiles && chmod 777 /corefiles

# 2. 程序崩溃后生成core文件
# 使用gdb分析
gdb /path/to/program /corefiles/core-program-12345-1673423401

# 3. gdb常用命令
(gdb) bt                      # 查看堆栈
(gdb) info threads            # 查看线程
(gdb) thread 2                # 切换到线程2
(gdb) bt                      # 查看该线程堆栈
(gdb) frame 3                 # 切换到第3帧
(gdb) list                    # 查看代码
(gdb) print var               # 打印变量
(gdb) info registers          # 查看寄存器

# 4. 分析内存泄漏
valgrind --leak-check=full ./program
# 或
valgrind --tool=memcheck --leak-check=yes ./program
```

---

## 九、Shell脚本编程

### 9.1 脚本基础与执行方式

```bash
# 1. 标准脚本格式（shebang）
#!/bin/bash
# Description: Backup script
# Author: Admin
# Date: 2024-01-15
# Version: 1.0

# 2. 执行方式对比
# 方式1：添加执行权限
chmod +x script.sh
./script.sh                # 相对路径
/path/to/script.sh         # 绝对路径

# 方式2：解释器执行（无需执行权限）
bash script.sh
sh script.sh
source script.sh           # 在当前shell执行（影响环境变量）
. script.sh                # 同source

# 区别：
# ./script.sh 或 bash script.sh：子shell执行，不影响当前环境
# source 或 .：当前shell执行，影响当前环境

# 3. 脚本调试
bash -x script.sh          # 显示执行过程
bash -n script.sh          # 检查语法错误
set -x                     # 脚本内开启调试
set +x                     # 关闭调试
```

### 9.2 重定向与管道

```bash
# 1. 重定向
# 标准输出重定向（1为stdout）
echo "hello" > file.txt    # 覆盖
echo "world" >> file.txt   # 追加

# 标准错误重定向（2为stderr）
command 2> error.log       # 错误重定向
command > out.log 2>&1     # 标准输出和错误都重定向（推荐）
command &> all.log         # Bash 4+ 简写

# 输入重定向
mysql -u root -p'pass' < backup.sql
# 或
mysql -u root -p'pass' -e "source backup.sql"

# 多行输入
cat > config.conf <<EOF
[Section]
key1=value1
key2=value2
EOF

# Here Document（忽略变量替换）
cat > config.conf <<'EOF'
PATH=$HOME/bin:$PATH
EOF

# Here String
grep "error" <<< "This is an error message"

# 2. 管道
ps aux | grep nginx
cat file.txt | sort | uniq -c | sort -nr
df -h | awk '{print $4}' | tail -n +2

# 管道与重定向组合
ps aux | tee processes.txt | grep ssh
# tee：同时输出到屏幕和文件
```

### 9.3 变量与数据类型

```bash
# 1. 变量定义（无类型）
name="John"
age=25
pi=3.14

# 注意：等号两边不能有空格
# name = "John"  # 错误

# 2. 变量引用
echo $name
echo ${name}      # 推荐，避免歧义
echo "My name is $name"
echo 'My name is $name'  # 单引号不解析变量

# 3. 只读变量
readonly PI=3.14159
PI=3.1416         # 报错：readonly variable

# 4. 删除变量
unset name

# 5. 特殊变量
$0                # 脚本名称
$1-$9             # 位置参数
$#                # 参数个数
$*                # 所有参数（整体）
$@                # 所有参数（个体）
$$                # 当前进程PID
$?                # 上条命令退出状态（0成功）
$!                # 最后一个后台进程PID

# 6. 环境变量
export PATH="/usr/local/bin:$PATH"  # 设置环境变量

# 7. 内置变量
$HOME             # 用户家目录
$PWD              # 当前目录
$OLDPWD           # 上一个目录
$UID               # 用户ID
$SHELL            # 当前Shell
$RANDOM           # 随机数（0-32767）

# 8. 数值运算
# 方式1：expr（旧方式）
a=10
b=20
c=$(expr $a + $b)  # 注意空格
# 支持：+ - * / %

# 方式2：$(( ))（推荐）
c=$((a + b))
c=$((a * b))
c=$((a / b))

# 方式3：let
let c=a+b
let a++           # 自增
let a--           # 自减
let a+=10         # 复合赋值

# 方式4：bc（浮点运算）
echo "scale=2; 10/3" | bc   # 3.33
```

### 9.4 流程控制

#### 9.4.1 条件判断

```bash
# if语句格式
if condition; then
    commands
elif condition2; then
    commands
else
    commands
fi

# 条件测试（[ ]是test的别名）
# 文件测试
[ -f file.txt ]      # 文件存在且为普通文件
[ -d /path ]         # 目录存在
[ -e file ]          # 文件/目录存在
[ -r file ]          # 可读
[ -w file ]          # 可写
[ -x file ]          # 可执行
[ -s file ]          # 文件大小大于0
[ -L link ]          # 是符号链接

# 字符串比较
[ -z "$str" ]        # 字符串为空
[ -n "$str" ]        # 字符串非空
[ "$str1" = "$str2" ] # 相等
[ "$str1" != "$str2" ]# 不等
[ "$str1" < "$str2" ] # 字典序小于（需转义：\<）
[ "$str1" > "$str2" ] # 字典序大于

# 数值比较
[ $a -eq $b ]        # 等于
[ $a -ne $b ]        # 不等于
[ $a -gt $b ]        # 大于
[ $a -ge $b ]        # 大于等于
[ $a -lt $b ]        # 小于
[ $a -le $b ]        # 小于等于

# 逻辑运算
[ condition1 -a condition2 ]  # 与（and）
[ condition1 -o condition2 ]  # 或（or）
! [ condition ]               # 非（not）

# 复合条件
if [ -f file.txt ] && [ -r file.txt ]; then
    echo "File exists and readable"
fi

# 示例：检查磁盘空间
DISK_USAGE=$(df / | tail -1 | awk '{print $5}' | tr -d '%')
if [ $DISK_USAGE -gt 80 ]; then
    echo "Warning: Disk usage is ${DISK_USAGE}%"
    # 发送告警
fi
```

#### 9.4.2 循环语句

```bash
# for循环（常用）
# 格式1：遍历列表
for i in 1 2 3 4 5; do
    echo $i
done

# 格式2：遍历范围
for i in {1..10}; do
    echo $i
done

# 格式3：C语言风格
for ((i=1; i<=10; i++)); do
    echo $i
done

# 格式4：遍历数组
arr=(apple banana cherry)
for fruit in "${arr[@]}"; do
    echo $fruit
done

# while循环
count=1
while [ $count -le 5 ]; do
    echo $count
    ((count++))
done

# 读取文件
while read line; do
    echo $line
done < file.txt

# until循环（条件为假时执行）
until [ $count -gt 5 ]; do
    echo $count
    ((count++))
done

# select菜单
PS3="Please select: "
select opt in "Option 1" "Option 2" "Quit"; do
    case $opt in
        "Option 1") echo "You selected 1" ;;
        "Option 2") echo "You selected 2" ;;
        "Quit") break ;;
        *) echo "Invalid option" ;;
    esac
done
```

#### 9.4.3 case分支

```bash
case $variable in
    pattern1)
        commands ;;
    pattern2|pattern3)
        commands ;;
    *)
        default_commands ;;
esac

# 示例：服务管理脚本
case "$1" in
    start)
        echo "Starting service..."
        /usr/local/bin/service start
        ;;
    stop)
        echo "Stopping service..."
        /usr/local/bin/service stop
        ;;
    restart)
        $0 stop
        sleep 2
        $0 start
        ;;
    status)
        /usr/local/bin/service status
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

### 9.5 函数定义

```bash
# 函数格式
function_name() {
    commands
}

# 或
function function_name {
    commands
}

# 示例
greet() {
    echo "Hello, $1!"
}

# 调用
greet "World"  # 输出：Hello, World!

# 返回值（0-255）
check_file() {
    if [ -f "$1" ]; then
        return 0  # 成功
    else
        return 1  # 失败
    fi
}

check_file "/etc/passwd"
if [ $? -eq 0 ]; then
    echo "File exists"
fi

# 返回字符串（使用echo）
get_date() {
    echo $(date +%Y-%m-%d)
}

today=$(get_date)
echo $today

# 局部变量
func() {
    local local_var="only in function"
    echo $local_var
}
func
echo $local_var  # 为空
```

### 9.6 数组操作

```bash
# 数组定义
arr1=(1 2 3 4 5)
arr2=("apple" "banana" "cherry")
arr3=([0]="first" [2]="third" [5]="sixth")  # 稀疏数组

# 访问元素
echo ${arr1[0]}        # 第一个元素
echo ${arr1[-1]}       # 最后一个元素

# 获取长度
echo ${#arr1[@]}       # 元素个数
echo ${#arr1[*]}       # 同上

# 获取所有元素
echo ${arr1[@]}        # 每个元素作为独立字符串
echo ${arr1[*]}        # 所有元素作为单个字符串

# 遍历数组
for i in "${arr2[@]}"; do
    echo $i
done

# 添加元素
arr1+=(6 7 8)          # 追加

# 删除元素
unset arr1[2]          # 删除索引2的元素

# 切片
echo ${arr1[@]:1:3}    # 从索引1开始，取3个元素

# 关联数组（类似Map，Bash 4+）
declare -A assoc_arr
assoc_arr[name]="John"
assoc_arr[age]=25
assoc_arr[city]="Beijing"

echo ${assoc_arr[name]}
for key in "${!assoc_arr[@]}"; do
    echo "$key: ${assoc_arr[$key]}"
done
```

### 9.7 正则表达式

```bash
# 基础正则（BRE）
# . 任意单个字符
# * 前面字符0次或多次
# ^ 行首
# $ 行尾
# [abc] 字符集
# [^abc] 取反
# \{n,m\} 重复n到m次

# 扩展正则（ERE，grep -E或egrep）
# + 前面字符1次或多次
# ? 前面字符0次或1次
# | 或
# () 分组
# {} 重复

# 示例
grep "error" log.txt               # 简单匹配
grep "^192.168" access.log         # 匹配行首
grep "200$" access.log             # 匹配行尾
grep "ab\{2,4\}" file.txt           # 匹配abbb或abbbb
egrep "error|fail" log.txt          # 匹配error或fail
egrep "(ab)+" file.txt              # 匹配ab, abab, ababab...

# 在变量中匹配
if [[ $email =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,4}$ ]]; then
    echo "Valid email"
fi

# 替换
${var/old/new}                     # 替换第一个
${var//old/new}                    # 替换全部
${var/#old/new}                    # 替换行首
${var/%old/new}                    # 替换行尾

# 示例
filename="file.txt.txt"
echo ${filename/.txt/.log}         # file.log.txt
echo ${filename//.txt/.log}        # file.log.log
```

### 9.8 错误处理与调试

```bash
# 1. 错误处理
set -e                           # 遇到错误立即退出
set -u                           # 使用未定义变量时报错
set -o pipefail                  # 管道中任意命令失败则整体失败
set -x                           # 显示执行命令

# 组合使用
#!/bin/bash
set -euo pipefail

# 2. 自定义错误处理
trap 'echo "Error at line $LINENO"' ERR

# 3. 检查命令是否存在
command -v nginx &>/dev/null || { echo "nginx not found"; exit 1; }

# 4. 日志记录
exec > >(tee -a /var/log/myscript.log)  # 标准输出重定向
exec 2>&1                              # 标准错误重定向到标准输出

# 5. 返回值检查
if ! command; then
    echo "Command failed"
    exit 1
fi

# 或
command || { echo "Failed"; exit 1; }
```

---

## 十、系统服务脚本编写

### 10.1 Systemd服务单元

```bash
# 示例：自定义应用服务
cat > /etc/systemd/system/myapp.service <<EOF
[Unit]
Description=My Application Service
After=network.target mysql.service redis.service
Wants=mysql.service redis.service

[Service]
Type=forking
User=appuser
Group=appgroup
WorkingDirectory=/opt/myapp
PIDFile=/run/myapp.pid
ExecStartPre=/opt/myapp/bin/check_config.sh
ExecStart=/opt/myapp/bin/start.sh
ExecReload=/bin/kill -s HUP \$MAINPID
ExecStop=/bin/kill -s TERM \$MAINPID
TimeoutStartSec=30
TimeoutStopSec=30
Restart=on-failure
RestartSec=10
LimitNOFILE=65536
Environment="PATH=/usr/local/bin:/usr/bin:/bin"
Environment="APP_ENV=production"

# 日志配置
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

[Install]
WantedBy=multi-user.target
EOF

# 服务管理命令
systemctl daemon-reload
systemctl start myapp
systemctl enable myapp
systemctl status myapp
systemctl stop myapp
systemctl restart myapp
systemctl reload myapp

# 查看服务日志
journalctl -u myapp -f
journalctl -u myapp --since "1 hour ago"

# 查看服务依赖
systemctl list-dependencies myapp
```

### 10.2 传统SysVinit脚本（兼容）

```bash
#!/bin/bash
# chkconfig: 2345 85 15
# description: My Application Service

# Source function library
. /etc/rc.d/init.d/functions

APP_NAME="myapp"
APP_BIN="/opt/myapp/bin/start.sh"
APP_PID="/run/myapp.pid"

start() {
    echo -n "Starting $APP_NAME: "
    if [ -f $APP_PID ]; then
        echo "Already running"
        return 1
    fi
    daemon $APP_BIN
    RETVAL=$?
    echo
    [ $RETVAL -eq 0 ] && touch /var/lock/subsys/$APP_NAME
    return $RETVAL
}

stop() {
    echo -n "Stopping $APP_NAME: "
    killproc $APP_NAME
    RETVAL=$?
    echo
    [ $RETVAL -eq 0 ] && rm -f /var/lock/subsys/$APP_NAME $APP_PID
    return $RETVAL
}

restart() {
    stop
    sleep 2
    start
}

case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    restart)
        restart
        ;;
    status)
        status $APP_NAME
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac

exit $?
```

---

## 十一、系统备份与灾难恢复

### 11.1 备份策略制定

```bash
# 3-2-1备份原则
# 3份备份：原始 + 2份备份
# 2种介质：磁盘 + 磁带/云
# 1份异地：物理隔离

# 备份类型
# 全量备份：周日，凌晨2点
# 增量备份：周一至周六，凌晨2点
# 实时备份：对关键数据（binlog、应用日志）

# RTO（恢复时间目标）和RPO（恢复点目标）
# RTO：系统恢复所需时间
# RPO：可接受的数据丢失量

# 备份工具选择
# 系统级：tar, rsync, dd, dump/restore
# 文件级：rsnapshot, Bacula, Amanda
# 虚拟化：Veeam, vSphere Data Protection
# 云备份：AWS S3, Azure Backup, Google Cloud Storage
```

### 11.2 rsync备份方案

```bash
# 1. 备份脚本
cat > /usr/local/bin/backup.sh <<'EOF'
#!/bin/bash
SOURCE_DIR="/data"
BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d_%H%M%S)
LOG_FILE="/var/log/backup.log"

# 创建备份目录
mkdir -p $BACKUP_DIR

# 排除文件
EXCLUDE_LIST="/tmp/backup_exclude.txt"
cat > $EXCLUDE_LIST <<EOT
*.log
*.tmp
cache/
temp/
EOT

# 执行备份
rsync -avz --delete --exclude-from=$EXCLUDE_LIST \
      $SOURCE_DIR/ $BACKUP_DIR/full-$DATE/

# 检查备份结果
if [ $? -eq 0 ]; then
    echo "[$DATE] Backup successful" >> $LOG_FILE
else
    echo "[$DATE] Backup failed" >> $LOG_FILE
    exit 1
fi

# 删除30天前的备份
find $BACKUP_DIR -type d -name "full-*" -mtime +30 -exec rm -rf {} \;

# 发送通知
echo "Backup completed" | mail -s "Backup Report" admin@company.com
EOF

chmod +x /usr/local/bin/backup.sh

# 2. rsync关键参数
# -a: 归档模式，保留所有属性
# -v: 详细输出
# -z: 压缩传输
# -r: 递归
# -P: 显示进度，支持断点续传
# -n: 试运行（dry run）
# --delete: 删除目标目录多余文件
# --exclude: 排除特定文件/目录
# --link-dest: 硬链接增量备份（rsnapshot原理）

# 3. rsync远程备份
rsync -avz -e ssh /data/ user@backup-server:/backup/
# 使用SSH密钥认证
rsync -avz -e "ssh -i /root/.ssh/backup_key" /data/ user@backup-server:/backup/

# 4. 增量备份实现
# 基于时间戳
rsync -avz --link-dest=/backup/latest /data/ /backup/$(date +%Y%m%d)/
ln -sf /backup/$(date +%Y%m%d) /backup/latest

# 5. 自动化
crontab -e
0 2 * * * /usr/local/bin/backup.sh
```

### 11.3 MySQL数据备份

```bash
# 物理备份（Percona XtraBackup，热备）
# 安装
yum install percona-xtrabackup-24

# 全量备份
xtrabackup --backup --user=root --password='pass' \
           --target-dir=/backup/mysql/full-$(date +%Y%m%d)

# 准备备份
xtrabackup --prepare --target-dir=/backup/mysql/full-$(date +%Y%m%d)

# 恢复
systemctl stop mysqld
rm -rf /var/lib/mysql/*
xtrabackup --copy-back --target-dir=/backup/mysql/full-$(date +%Y%m%d)
chown -R mysql:mysql /var/lib/mysql
systemctl start mysqld

# 增量备份
xtrabackup --backup --user=root --password='pass' \
           --target-dir=/backup/mysql/inc-$(date +%Y%m%d) \
           --incremental-basedir=/backup/mysql/full-20240101

# 逻辑备份（mysqldump + binlog）
# 全量备份
mysqldump --all-databases --single-transaction --master-data=2 \
          --quick --lock-tables=false | gzip > /backup/mysql/all-$(date +%Y%m%d).sql.gz

# 备份binlog
mysql -e "SHOW MASTER STATUS;" > /backup/mysql/master-pos.txt
cp /var/log/mysql/mysql-bin.* /backup/mysql/

# 恢复
gunzip < all-20240101.sql.gz | mysql
# 然后应用binlog
mysqlbinlog mysql-bin.000001 mysql-bin.000002 | mysql
```