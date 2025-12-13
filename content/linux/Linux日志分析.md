---
title: linux日志分析
description: 
date: 2025-11-11
categories:
    - 
    - 
---
# Linux日志分析

## 一、核心日志文件

### 1\. 文本类系统日志 (Syslog)

这些文件通常由 `rsyslog` 或 `syslog-ng` 管理，是文本格式，可直接阅读。

| 发行版            | 主要日志文件        | 作用与分析重点                                               |
| :---------------- | :------------------ | :----------------------------------------------------------- |
| **CentOS/RHEL**   | `/var/log/messages` | **系统总览**：记录服务启动、内核报错、网络状态等非敏感信息。 |
| **Ubuntu/Debian** | `/var/log/syslog`   | 同上。                                                       |
| **CentOS/RHEL**   | `/var/log/secure`   | **安全认证**：SSH 登录、sudo 提权、添加用户等安全事件。      |
| **Ubuntu/Debian** | `/var/log/auth.log` | 同上。                                                       |
| **通用**          | `/var/log/cron`     | **定时任务**：记录 Crontab 任务的执行时间、命令及是否启动成功。 |
| **通用**          | `/var/log/boot.log` | **启动流程**：记录开机时服务的启动顺序和状态（OK/FAILED）。  |
| **通用**          | `/var/log/maillog`  | **邮件服务**：Postfix/Sendmail 的发送记录。                  |

### 2\. 二进制日志 (需专用命令查看)

这些文件无法直接用 `cat` 查看，记录了用户登录的历史数据，防篡改性更强。

| 文件路径           | 查看命令  | 作用                                               |
| :----------------- | :-------- | :------------------------------------------------- |
| `/var/log/wtmp`    | `last`    | 查看**成功登录**的历史记录（含重启记录）。         |
| `/var/log/btmp`    | `lastb`   | 查看**失败登录**的历史记录（暴力破解分析的核心）。 |
| `/var/log/lastlog` | `lastlog` | 查看系统中所有用户**最后一次登录**的时间。         |

### 3\. 特殊组件日志

  - **`/var/log/dmesg`**：内核环缓冲区日志，主要记录硬件检测、驱动加载信息（使用 `dmesg` 命令查看效果更好）。
  - **`/var/log/audit/audit.log`**：Linux 审计守护进程日志，记录极其详细的系统调用信息（SELinux 报错主要看这里）。

-----

## 二、日志级别标准 (Log Levels)

在分析日志时，通过级别过滤是最高效的方法。Syslog 标准定义了 0-7 级：

| 代码  | 级别 (Keyword) | 含义             | 建议                                 |
| :---- | :------------- | :--------------- | :----------------------------------- |
| **0** | `emerg`        | 系统不可用       | 系统级崩溃，需立即处理。             |
| **1** | `alert`        | 必须立即采取行动 | 数据库损坏等严重问题。               |
| **2** | `crit`         | 关键状况         | 硬件错误、磁盘写满。                 |
| **3** | `err`          | **错误**         | 软件运行报错，最常关注的级别。       |
| **4** | `warning`      | 警告             | 配置过时、资源紧张，但暂不影响运行。 |
| **5** | `notice`       | 普通但重要的消息 | 服务正常启动/停止。                  |
| **6** | `info`         | 信息性消息       | 常规运行日志。                       |
| **7** | `debug`        | 调试信息         | 开发调试用，数据量巨大。             |

-----

## 三、基础查看与搜索技巧

### 1\. 实时监控 (Tail)

```bash
# 基础监控：查看最后 20 行并持续跟踪
tail -n 20 -f /var/log/messages

# 多文件监控：同时对比系统日志和安全日志（排查服务报错引发的权限问题）
tail -f /var/log/messages /var/log/secure
```

### 2\. 高效浏览 (Less)

`less` 是查看大日志文件（GB级别）的最佳工具，因为它不会一次性加载整个文件。

  * **常用快捷键**：
      * `G`：跳到文件末尾（查看最新日志）。
      * `g`：跳到文件开头。
      * `/error`：向下搜索 "error"。
      * `?error`：向上搜索 "error"。
      * `Shift+F`：进入实时滚动模式（类似 `tail -f`），按 `Ctrl+C` 退出回普通浏览。

### 3\. 强力搜索 (Grep 进阶)

```bash
# 1. 忽略大小写查找错误
grep -i "error" /var/log/syslog

# 2. 扩展正则：同时查找 error, fail, fatal, denied
grep -E -i "error|fail|fatal|denied" /var/log/messages

# 3. 上下文定位：显示匹配行及其前后各 5 行 (Context)
grep -C 5 -i "oom-killer" /var/log/messages
# -A 5 (After): 后5行, -B 5 (Before): 前5行

# 4. 反向排除：查找错误但排除 debug 信息
grep -i "error" /var/log/messages | grep -v "debug"
```

-----

## 四、Systemd 时代的日志神器：journalctl

`journalctl` 统一管理了所有由 systemd 启动的服务的日志，支持结构化查询。

### 1\. 时间过滤

```bash
# 查看从昨天到现在的日志
journalctl --since "yesterday"

# 查看最近1小时的日志
journalctl --since "1 hour ago"

# 查看特定时间段
journalctl --since "2024-11-10 10:00:00" --until "2024-11-10 12:00:00"
```

### 2\. 服务与级别过滤

```bash
# 查看 Nginx 服务的错误级别(err)日志
journalctl -u nginx -p err

# 实时跟踪 SSH 服务日志
journalctl -f -u sshd
```

### 3\. 启动记录分析

```bash
# 列出系统所有启动记录
journalctl --list-boots

# 查看上一次启动期间的日志（用于排查系统因故障重启前发生了什么）
journalctl -b -1
```

### 4\. 维护日志体积

```bash
# 查看日志占用的磁盘空间
journalctl --disk-usage

# 清理日志，只保留最近 1GB
journalctl --vacuum-size=1G

# 清理日志，只保留最近 7 天
journalctl --vacuum-time=7d
```

-----

## 五、深度实战分析案例

### 案例 1：SSH 暴力破解精准画像

**场景**：发现服务器变慢，`/var/log/secure` 疯涨。

```bash
# 1. 统计前 10 名攻击者 IP
grep "Failed password" /var/log/secure | awk '{print $(NF-3)}' | sort | uniq -c | sort -nr | head -10
# 注意：$(NF-3) 表示倒数第4列，这比固定 $11 更通用，适应不同日志格式。

# 2. 统计攻击者尝试猜测的用户名
grep "Failed password" /var/log/secure | awk '{print $(NF-5)}' | sort | uniq -c | sort -nr | head -10

# 3. 使用 lastb 查看二进制记录（更防篡改）
lastb | awk '{print $3}' | sort | uniq -c | sort -nr | head -10
```

### 案例 2：内存泄漏与 OOM (Out Of Memory)

**场景**：MySQL 或 Java 应用突然崩溃，没有产生应用日志。

```bash
# 搜索内核日志中的 OOM Killer 记录
grep -i "killed process" /var/log/messages
# 或者
dmesg | grep -i "oom"

# 使用 journalctl 查看详细上下文
journalctl -k | grep -i -C 5 "out of memory"

# 解读输出：
# kernel: Out of memory: Kill process 1234 (java) score 800 or sacrifice child
# 含义：进程 1234 (java) 占用内存过高，被内核强制杀死以保护系统。
```

### 案例 3：定位流量/请求异常

**场景**：Web 访问日志分析（以 Nginx 为例）。

```bash
# 1. 找出 HTTP 状态码非 200 的分布
awk '{print $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn
# $9 通常是状态码

# 2. 找出访问最慢的 10 个请求（假设日志配置了响应时间在最后）
awk '{print $NF " " $7}' /var/log/nginx/access.log | sort -nr | head -10
# $NF: 最后一列(响应时间), $7: 请求路径

# 3. 统计每分钟的请求数（查看并发波峰）
awk '{print $4}' /var/log/nginx/access.log | cut -c 14-18 | sort | uniq -c
# 截取时间字段中的分钟部分
```

-----

## 六、自动化日志分析脚本 (Enhanced)

保存为 `log_audit.sh`，赋予执行权限 `chmod +x log_audit.sh`。

```bash
#!/bin/bash
# Linux 系统日志健康检查脚本
# 适用：CentOS/Ubuntu/Debian

LOG_FILE="/var/log/messages"
[ -f /var/log/syslog ] && LOG_FILE="/var/log/syslog"
SECURE_LOG="/var/log/secure"
[ -f /var/log/auth.log ] && SECURE_LOG="/var/log/auth.log"

DATE_TODAY=$(date "+%b %e") # 格式如: "Nov 11" (注意: %e单数字日期前有空格)
REPORT_FILE="/tmp/sys_audit_$(date +%Y%m%d).txt"

echo "====== 系统日志审计报告 [$(date)] ======" > "$REPORT_FILE"

# 1. 错误概览
echo -e "\n[1] 今日错误与警告统计 ($DATE_TODAY)" >> "$REPORT_FILE"
echo "---------------------------------" >> "$REPORT_FILE"
grep "$DATE_TODAY" "$LOG_FILE" | grep -E -i "error|fail|critical|alert|emerg" | awk '{$1=$2=$3=""; print $0}' | sort | uniq -c | sort -nr | head -10 >> "$REPORT_FILE"

# 2. 暴力破解检测
echo -e "\n[2] 今日 SSH 失败登录 TOP 10 IP" >> "$REPORT_FILE"
echo "---------------------------------" >> "$REPORT_FILE"
grep "$DATE_TODAY" "$SECURE_LOG" | grep "Failed password" | awk '{for(i=1;i<=NF;i++) if($i~/^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$/) print $i}' | sort | uniq -c | sort -nr | head -10 >> "$REPORT_FILE"

# 3. 关键硬件与内核事件
echo -e "\n[3] 硬件与内核异常 (dmesg)" >> "$REPORT_FILE"
echo "---------------------------------" >> "$REPORT_FILE"
dmesg | grep -E -i "error|fail|warn|temp|sector" | tail -10 >> "$REPORT_FILE"

# 4. 磁盘空间检查
echo -e "\n[4] 磁盘空间使用情况" >> "$REPORT_FILE"
echo "---------------------------------" >> "$REPORT_FILE"
df -h | grep -E "^/dev" | awk '$5 > "85%" {print "警告: " $1 " 使用率 " $5}' >> "$REPORT_FILE"

echo "审计完成，报告已生成: $REPORT_FILE"
cat "$REPORT_FILE"
```

-----

## 七、日志轮转 (Logrotate) 核心配置

防止日志占满磁盘的关键机制。配置文件位于 `/etc/logrotate.conf` 和 `/etc/logrotate.d/`。

**常用配置解释：**

```nginx
# /etc/logrotate.d/nginx
/var/log/nginx/*.log {
    daily                   # 每天切割
    rotate 14               # 保留最近14份
    missingok               # 日志不存在不报错
    notifempty              # 空文件不切割
    compress                # 切割后的旧日志使用 gzip 压缩
    delaycompress           # 延迟一天压缩（防止当前还有进程在写）
    sharedscripts           # 所有日志切完后只执行一次脚本
    postrotate
        # 发送信号给 Nginx 让其重新生成日志文件
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

**强制测试命令**（修改配置后验证用）：

```bash
logrotate -d -f /etc/logrotate.d/nginx
# -d: Debug 模式（不实际操作）
# -f: Force 强制立即执行
```

-----

## 八、故障排查速查表

| 现象             | 可能原因             | 排查命令 (优先级从高到低)                                    |
| :--------------- | :------------------- | :----------------------------------------------------------- |
| **服务器连不上** | 网络/SSHD死锁/防火墙 | 1. 控制台登录 <br> 2. `systemctl status sshd` <br> 3. `iptables -L -n` |
| **服务自动停止** | 内存溢出/配置错误    | 1. `journalctl -u <服务> -e` <br> 2. `dmesg | grep -i kill` (查OOM) |
| **磁盘空间满**   | 日志未轮转/大文件    | 1. `du -sh /*` (定位大目录) <br> 2. `lsof | grep deleted` (查已删但未释放的文件) |
| **系统负载高**   | 进程死循环/IO瓶颈    | 1. `top` <br> 2. `iostat -x 1` <br> 3. 查看日志中是否有大量重复报错 |
| **文件无法写入** | 权限/磁盘满/inode满  | 1. `ls -ld <目录>` <br> 2. `df -h` <br> 3. `df -i` (查inode节点) |
| **时间不准**     | 时区/NTP服务         | 1. `timedatectl` <br> 2. `grep "ntp" /var/log/messages`      |