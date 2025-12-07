---
title: Linux命令
description: 
date: 2025-11-08
categories:
    - 
    - 
---

# 🐧 Linux 运维命令手册 

**适用场景**：系统管理、故障排查、性能调优、自动化运维
**核心原则**：生产环境安全第一，操作前确认，高危命令需谨慎。

-----

## 📂 第一章：核心文件与目录管理

### 1.1 目录列表与切换 (`ls`, `cd`, `pwd`)

  * **`ls` (List)**：查看目录内容

      * `ls -lhS --color=auto`：**[运维推荐]** 按文件大小排序，显示人类可读单位 (KB/MB)，带颜色区分。
      * `ls -lart`：按修改时间倒序排列（最新的在最下），常用于快速查看最近改动的文件。
      * `ls -i`：查看 inode 号（排查硬链接或文件名乱码无法删除时使用）。
      * `ls -Z`：查看 SELinux 上下文。

  * **`cd` (Change Directory)**

      * `cd -`：**[高频]** 快速回到上一次所在的目录。
      * `cd ~` / `cd`：回到当前用户家目录。

  * **`pwd` (Print Working Directory)**

      * `pwd -P`：显示真实的物理路径（解析软链接后的路径）。

### 1.2 文件增删改 (`mkdir`, `rm`, `cp`, `mv`, `touch`)

  * **`mkdir` (Make Directory)**

      * `mkdir -p /path/to/dir`：递归创建多级目录，若存在不报错。
      * `mkdir -m 755 /path`：创建目录时直接指定权限。

  * **`rm` (Remove) —— ⚠️ 高危命令**

      * **[黄金法则]**：先 `ls` 确认文件，再 `rm`。例如：`ls /tmp/garbage && rm -rf /tmp/garbage`。
      * **[安全技巧]**：在 `.bashrc` 中设置别名 `alias rm='rm -i'` (删除前确认) 或 `alias rm='rm --preserve-root'` (防删根目录)。
      * `rm -rf /path`：强制递归删除，不可恢复。

  * **`cp` (Copy)**

      * `cp -a source dest`：**[备份首选]** 等同于 `-dR --preserve=all`，完整保留权限、时间戳、链接等属性。
      * `cp -u source dest`：增量复制，仅当源文件比目标新或目标不存在时才复制。

  * **`mv` (Move)**

      * `mv -n file1 file2`：不覆盖已存在的目标文件。
      * `mv -t target_dir source_files`：将多个源文件移动到目标目录。

  * **`touch`**

      * `touch -t 202501011200 file`：修改文件时间戳为指定时间。
      * `touch -r ref_file file`：将 `file` 的时间戳改成与 `ref_file` 一致。

-----

## 📊 第二章：系统资源与性能深度监控

### 2.1 CPU 与 进程 (`top`, `htop`, `vmstat`, `mpstat`)

  * **`top` (实时监控)**

      * `top -Hp <PID>`：**[排查神器]** 查看指定进程内的所有**线程**资源占用（排查 Java/Go 高负载关键）。
      * `top -b -n 1 > top.log`：批处理模式，将快照保存到文件。
      * *交互快捷键*：`P` (按CPU排序), `M` (按内存排序), `1` (显示多核详情)。

  * **`vmstat` (虚拟内存统计)**

      * `vmstat 1 10`：每秒刷新1次，共10次。
      * **[关键指标解读]**：
          * `r` (Running)：运行队列，超过 CPU 核数表示 CPU 瓶颈。
          * `si`/`so` (Swap In/Out)：非 0 表示内存不足，正在使用交换分区。
          * `wa` (IO Wait)：持续 \>20% 说明磁盘 IO 是瓶颈。

  * **`mpstat` (多核 CPU 分析)**

      * `mpstat -P ALL 1`：显示每个 CPU 核心的独立利用率，排查单核被打满的情况。

### 2.2 内存分析 (`free`, `pmap`, `smem`)

  * **`free`**

      * `free -h`：人类可读格式。
      * **[避坑]**：不要只看 `free` 列，`available` 才是真正应用程序可用的内存。

  * **`pmap` (进程内存映射)**

      * `pmap -x <PID>`：查看进程详细内存分布，排查内存泄漏。

### 2.3 磁盘 IO (`iostat`, `iotop`, `du`, `df`)

  * **`iostat`**

      * `iostat -x 1 10`：显示扩展统计信息。
      * **[关键指标]**：`%util` (设备利用率) \>80% 为瓶颈；`await` (平均响应时间) \>10ms 需关注。

  * **`du` / `df`**

      * `df -h`：查看磁盘挂载点使用率（生产红线：\>85% 预警）。
      * `df -i`：查看 inode 使用率（inode 耗尽会导致无法写入）。
      * `du -h --max-depth=1 /var | sort -hr | head -10`：**[实战]** 快速找出 `/var` 下最大的一级子目录。

### 2.4 网络流量 (`iftop`, `ss`, `tcpdump`)

  * **`ss` (Socket Statistics - 推荐替代 netstat)**

      * `ss -tuln`：显示所有监听的 TCP/UDP 端口。
      * `ss -s`：显示 Socket 统计摘要（快速查看连接总数）。
      * `ss -o state established '( dport = :22 or sport = :22 )'`：过滤特定端口的已建立连接。

  * **`tcpdump` (抓包 - 网络排障核心)**

      * `tcpdump -i eth0 -nn -s0 -v port 80`：抓取 HTTP 流量。
      * `tcpdump -i any -nn host 1.2.3.4`：抓取与特定 IP 的交互。
      * `tcpdump -w capture.pcap`：保存为文件供 Wireshark 分析。
      * `tcpdump -A -s0 port 80`：以 ASCII 码显示包内容（可看 HTTP Body）。

-----

## 📝 第三章：文本处理“三剑客” (运维必修)

### 3.1 `grep` (文本搜索)

  * **基础**：`grep -i "error" file` (忽略大小写)；`grep -v "info" file` (反选，排除)。
  * **正则**：`grep -E "error|warn" file` (匹配多个关键字)。
  * **上下文**：`grep -C 5 "exception" file` (显示匹配行前后 5 行，`-A` 后，`-B` 前)。
  * **统计**：`grep -c "404" access.log` (统计行数)。

### 3.2 `awk` (文本处理与统计)

  * **提取列**：`awk '{print $1, $NF}' file` (打印第1列和最后一列)。
  * **条件过滤**：`awk '$9=="500" {print $0}' access.log` (打印第9列为500的行)。
  * **[实战] Nginx 日志分析 Top IP**：
    ```bash
    awk '{print $1}' access.log | sort | uniq -c | sort -nr | head -10
    ```

### 3.3 `sed` (流编辑器)

  * **替换**：`sed 's/old/new/g' file` (全局替换输出，不改源文件)。
  * **修改文件**：`sed -i 's/PORT/8080/g' config.conf` (直接修改文件，建议先备份)。
  * **时间范围提取**：
    ```bash
    sed -n '/2024-01-01 10:00/,/2024-01-01 11:00/p' access.log
    ```

-----

## 💻 第四章：进程全生命周期管理

### 4.1 查找进程 (`ps`, `pgrep`)

  * `ps aux --sort=-%cpu`：显示所有进程并按 CPU 使用率降序排列。
  * `pgrep -f "java -jar app.jar"`：按完整命令行查找 PID。
  * `pstree -p`：以树状图显示进程关系及 PID。

### 4.2 控制进程 (`kill`, `nohup`)

  * **`kill`**

      * `kill -15 <PID>`：**[推荐]** 发送 TERM 信号，让进程优雅退出（保存数据、关闭句柄）。
      * `kill -9 <PID>`：强制杀死进程，可能导致数据丢失，仅在进程卡死时使用。
      * `kill -HUP <PID>`：让进程重载配置（如 Nginx 平滑重启）。

  * **`nohup`**

      * `nohup java -jar app.jar > app.log 2>&1 &`：标准后台运行写法，防挂断，重定向标准输出和错误。

-----

## 🛠️ 第五章：系统服务与日志 (Systemd)

### 5.1 服务管理 (`systemctl`)

  * `systemctl status/start/stop/restart nginx`：基本控制。
  * `systemctl enable nginx`：设置开机自启。
  * `systemctl list-units --type=service --state=failed`：**[实战]** 快速列出所有启动失败的服务。

### 5.2 日志查询 (`journalctl`)

  * `journalctl -u nginx -f`：实时跟踪特定服务的日志。
  * `journalctl --since "10 minutes ago"`：查看最近 10 分钟的日志。
  * `journalctl -p err`：只看错误级别的日志。
  * `journalctl --disk-usage`：查看日志占用了多少磁盘。

-----

## 📦 第六章：压缩、备份与传输

### 6.1 打包与压缩 (`tar`)

  * **压缩**：`tar -czf backup.tar.gz /data`
      * `-c`: 创建, `-z`: gzip压缩, `-f`: 指定文件名。
  * **解压**：`tar -xzf backup.tar.gz -C /opt`
      * `-x`: 解压, `-C`: 指定解压目录。
  * **查看**：`tar -tzf backup.tar.gz` (不解压查看内容)。

### 6.2 同步与传输 (`rsync`)

  * **`rsync` (运维神器)**
      * `rsync -avz /src/ /dst/`：同步目录（`-a` 归档模式保留属性, `-v` 详细, `-z` 压缩传输）。
      * `rsync -avz --delete /src/ /dst/`：**[慎用]** 镜像同步，目标端多余文件会被删除。
      * `rsync -avz --bwlimit=10240 /src/ user@host:/dst/`：限速 10MB/s 传输，防止占满带宽。

-----

## 🔐 第七章：用户权限与安全

### 7.1 权限控制 (`chmod`, `chown`)

  * `chmod 755 file` (rwx-r-x-r-x) / `chmod 644 file` (rw-r--r--)。
  * `chmod +x script.sh`：赋予执行权限。
  * `chown -R user:group /data`：递归修改目录的所有者和所属组。

### 7.2 特殊权限与安全

  * `lsattr / chattr`：
      * `chattr +i file`：**[安全加固]** 锁定文件，root 也无法修改或删除（适用于 `/etc/passwd` 等关键文件）。
      * `chattr -i file`：解锁文件。
  * `w` / `last`：查看当前登录用户及历史登录记录，排查非法入侵。

-----

## ⚙️ 第八章：内核调优与硬件信息 (专家级)

### 8.1 硬件查看

  * `lscpu`：查看 CPU 架构、核数。
  * `lsblk` / `fdisk -l`：查看磁盘分区情况。
  * `dmidecode -t memory`：查看内存硬件条数、频率。

### 8.2 内核参数 (`sysctl`)

  * `sysctl -a`：查看所有内核参数。
  * `sysctl -p`：加载 `/etc/sysctl.conf` 配置。
  * **[常见调优]**：
      * 减少 Swap 使用：`vm.swappiness = 10`
      * 开启端口复用：`net.ipv4.tcp_tw_reuse = 1` (解决大量 TIME\_WAIT)。

-----

## 🧩 第九章：生产环境故障排查实战案例

### 案例 1：CPU 100% 但找不到高负载进程

**现象**：`top` 显示 CPU 使用率高，但进程列表无异常。
**分析**：可能是短命进程（频繁启动退出）或 IO 等待。
**排查**：

1.  看 `top` 中的 `wa` (IO Wait) 是否高。
2.  `perf top` 查看热点函数。
3.  如果 `wa` 高，用 `iotop -oP` 抓出正在进行磁盘读写的进程。

### 案例 2：磁盘满了但 `du` 统计不到大文件

**现象**：`df -h` 显示 100%，但 `du -sh /` 显示占用很少。
**原因**：文件被删除（rm），但进程仍持有句柄，空间未释放。
**排查**：

```bash
lsof | grep deleted
```

**解决**：重启对应进程，或 `echo > /proc/<PID>/fd/<FD>` 清空文件描述符。

### 案例 3：网络连接数暴增

**排查**：

1.  `ss -s` 查看总连接数。
2.  `netstat -anp | grep ESTABLISHED | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -nr | head` 找出连接数最多的来源 IP。

-----

## 📜 第十章：自动化巡检脚本模板

将以下脚本保存为 `check.sh` 即可快速进行系统体检：

```bash
#!/bin/bash
# 系统健康检查简易脚本

echo "===== System Check $(date) ====="

# 1. 负载检查
echo "--- Load Average ---"
uptime

# 2. 内存检查 (重点看 available)
echo "--- Memory ---"
free -h

# 3. 磁盘空间 (>85% 需警惕)
echo "--- Disk Usage ---"
df -h | grep -E '^/dev'

# 4. 磁盘 IO 检查 (查看是否有设备利用率超标)
echo "--- Disk IO ---"
iostat -x 1 1 | awk '$NF > 80 {print "WARNING: High IO on " $1}'

# 5. 网络连接概况
echo "--- Network Connections ---"
ss -s

# 6. 检查失败的服务
echo "--- Failed Services ---"
systemctl list-units --type=service --state=failed

echo "===== Check Complete ====="
```