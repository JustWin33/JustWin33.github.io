---
title: Linux脚本运行方式
description: 
date: 2025-11-06
categories:
    - 
    - 
---
# Linux 脚本运行方式

在 CentOS/RHEL 系统下，运行脚本共有 **20+ 种方法**，以下是详细分类说明：

下面脚本的名字是install_ops_tools_centos.sh作为参考

---

## 一、基础执行方式（8种）

### 1. **相对路径执行（最常用）**
```bash
./install_ops_tools_centos.sh
```
- **原理**：通过相对路径指定文件，OS 根据 shebang (`#!/bin/bash`) 调用解释器
- **要求**：必须拥有执行权限 `chmod +x`
- **注意**：当前目录 `./` 必须在 PATH 中，否则需明确指定

### 2. **绝对路径执行**
```bash
/root/install_ops_tools_centos.sh
```
- **原理**：从根目录开始的完整路径，避免依赖当前位置
- **适用**：在脚本中调用其他脚本，或从任意位置执行

### 3. **使用 bash 解释器直接执行**
```bash
bash install_ops_tools_centos.sh
bash /root/install_ops_tools_centos.sh
```
- **原理**：显式指定 bash 解释器，忽略 shebang 行
- **优点**：无需执行权限，适合临时测试脚本
- **缺点**：可能使用不同 bash 版本（如 `/bin/bash` vs `/usr/local/bin/bash`）

### 4. **使用 sh 解释器执行**
```bash
sh install_ops_tools_centos.sh
```
- **注意**：CentOS 中 `sh` 是 bash 的软链接，但会以 POSIX 模式运行，可能不支持 bash 特有语法

### 5. **使用其他 shell 解释器**
```bash
zsh install_ops_tools_centos.sh  # 使用 zsh
dash install_ops_tools_centos.sh  # 使用 dash（更严格的 POSIX）
ksh install_ops_tools_centos.sh   # 使用 Korn shell
```
- **场景**：测试脚本跨 shell 兼容性

### 6. **source 命令（当前 shell 执行）**
```bash
source install_ops_tools_centos.sh
. install_ops_tools_centos.sh
```
- **原理**：在当前 shell 进程读取执行，不创建子进程
- **影响**：脚本中的变量、函数、别名会在当前 shell 保留
- **风险**：可能污染当前 shell 环境
- **适用**：加载环境变量、配置文件（如 `.bashrc`）

### 7. **exec 命令（替换进程）**
```bash
exec ./install_ops_tools_centos.sh
```
- **原理**：用脚本进程替换当前 shell 进程，执行后终端会话结束
- **警告**：脚本退出后，当前终端会自动关闭！

### 8. **交互式输入重定向**
```bash
bash < install_ops_tools_centos.sh
```
- **原理**：将文件内容作为 bash 的标准输入
- **缺点**：无法传递参数，`$0` 会变成 `bash`

---

## 二、带参数执行方式（4种）

### 9. **带位置参数执行**
```bash
./install_ops_tools_centos.sh arg1 arg2 arg3
bash install_ops_tools_centos.sh arg1 arg2 arg3
```
- **访问**：脚本中通过 `$1`, `$2`, `$3` 获取参数

### 10. **带环境变量执行**
```bash
MY_VAR=value ./install_ops_tools_centos.sh
env MY_VAR=value ./install_ops_tools_centos.sh
```
- **区别**：第一种是当前 shell 临时变量，第二种是纯净环境

### 11. **带 sudo 执行（提升权限）**
```bash
sudo ./install_ops_tools_centos.sh
sudo -E ./install_ops_tools_centos.sh  # 保留当前用户环境变量
sudo -u apache ./script.sh             # 以特定用户身份执行
```
- **注意**：脚本和引用的文件需对 root 可读

### 12. **带调试模式执行**
```bash
bash -x install_ops_tools_centos.sh    # 显示每条命令及参数
bash -v install_ops_tools_centos.sh    # 显示原始命令行
bash -n install_ops_tools_centos.sh    # 仅语法检查不执行
bash -e install_ops_tools_centos.sh    # 遇到错误立即退出
```

---

## 三、远程与管道执行（5种）

### 13. **通过 SSH 远程执行**
```bash
ssh user@remote_host '/root/install_ops_tools_centos.sh'
ssh user@remote_host 'bash -s' < script.sh  # 本地脚本在远程执行
```
- **注意**：TTY 分配问题，可能需要 `-t` 参数

### 14. **通过 curl 直接执行（不推荐但有）**
```bash
curl -sSL https://example.com/script.sh | bash
curl -sSL https://example.com/script.sh | sh
```
- **警告**：极度危险！可能执行恶意代码
- **建议**：先下载检查再执行

### 15. **通过 wget 执行**
```bash
wget -qO- https://example.com/script.sh | bash
```
- **安全**：同 curl 方式，存在供应链攻击风险

### 16. **通过管道传递给 xargs**
```bash
echo "./install_ops_tools_centos.sh" | xargs bash
find . -name "*.sh" | xargs -I {} bash {}  # 批量执行
```

### 17. **进程替换执行**
```bash
bash <(curl -sSL https://example.com/script.sh)
```
- **原理**：将命令输出作为临时文件描述符

---

## 四、后台与定时执行（6种）

### 18. **后台执行**
```bash
./install_ops_tools_centos.sh &          # 直接后台运行
nohup ./install_ops_tools_centos.sh &    # 忽略挂断信号，日志写入 nohup.out
nohup ./script.sh > output.log 2>&1 &   # 自定义输出重定向
```
- **查看**：`jobs`（当前终端），`ps aux | grep script`（系统范围）

### 19. **screen 会话中执行**
```bash
screen -S install_session
./install_ops_tools_centos.sh
# Ctrl+A D 分离会话
screen -r install_session  # 重新连接
```

### 20. **tmux 会话中执行**
```bash
tmux new -s install
./install_ops_tools_centos.sh
# Ctrl+B D 分离
tmux attach -t install
```

### 21. **at 定时执行**
```bash
echo "./install_ops_tools_centos.sh" | at 02:00
at -f /root/install_ops_tools_centos.sh 14:30 tomorrow
```
- **前提**：需启动 `atd` 服务：`systemctl start atd`

### 22. **cron 计划任务执行**
```bash
# 编辑 crontab
crontab -e
# 添加条目（每天凌晨2点执行）
0 2 * * * /root/install_ops_tools_centos.sh >> /var/log/install.log 2>&1
```
- **注意**：cron 环境变量极少，建议写绝对路径

### 23. **systemd timer 执行（现代方式）**
```bash
# 创建 service 文件 /etc/systemd/system/myscript.service
[Service]
ExecStart=/root/install_ops_tools_centos.sh

# 创建 timer 文件 /etc/systemd/system/myscript.timer
[Timer]
OnCalendar=daily
Persistent=true

# 启用并启动
systemctl enable --now myscript.timer
```

---

## 五、容器与虚拟环境（3种）

### 24. **Docker 中执行**
```bash
docker run --rm -v $(pwd):/scripts centos:7 /scripts/install_ops_tools_centos.sh
docker exec -it container_name /root/script.sh
```

### 25. **chroot 环境中执行**
```bash
chroot /mnt/centos /root/install_ops_tools_centos.sh
```
- **要求**：目标环境需包含所有依赖

### 26. **systemd-nspawn 容器**
```bash
systemd-nspawn -D /var/lib/container/centos /root/script.sh
```

---

## 六、图形界面方式（2种）

### 27. **Nautilus 文件管理器**
- 右键脚本 → 属性 → 权限 → 勾选"允许作为程序执行"
- 双击文件 → 选择"在终端中运行"

### 28. **KDE Dolphin 文件管理器**
- 右键 → 属性 → 权限 → 勾选"可执行"
- 双击 → 选择"在终端中运行"

---

## 七、特殊用途方式（4种）

### 29. **限制资源执行（ulimit）**
```bash
ulimit -t 300 -v 1048576  # 限制CPU时间300秒，内存1GB
./install_ops_tools_centos.sh
```

### 30. **特定环境变量执行**
```bash
env -i HOME=/root PATH=/usr/bin:/bin ./script.sh  # 纯净环境
```

### 31. **parallel 并行执行**
```bash
parallel ::: ./script1.sh ./script2.sh ./script3.sh
```

### 32. **expect 自动交互**
```bash
# 先编写 expect 脚本 auto.expect
spawn ./install_ops_tools_centos.sh
expect "password:"
send "mypassword\r"
interact
```

---

## 八、安全相关注意事项

### ⚠️ 关键安全要点

| 风险场景           | 安全建议                                              |
| ------------------ | ----------------------------------------------------- |
| **执行未知脚本**   | 先用 `less/more/vim` 查看内容，检查 `curl\|bash` 管道 |
| **sudo 执行**      | 使用 `sudo -E` 保留环境变量需谨慎，检查脚本所有权     |
| **SUID/SGID 位**   | 绝对不要给脚本设置 `chmod u+s`，无效且易绕过          |
| **World-writable** | 避免执行 `/tmp` 等公共目录下的脚本，可能被篡改        |
| **隐藏字符**       | 用 `cat -A script.sh` 检查是否有恶意字符              |
| **校验完整性**     | 对重要脚本使用 GPG 签名：`gpg --verify script.sh.sig` |

### 🔍 脚本检查命令
```bash
# 查看脚本基本信息
file install_ops_tools_centos.sh
stat install_ops_tools_centos.sh
ls -l install_ops_tools_centos.sh

# 语法检查
bash -n install_ops_tools_centos.sh

# 跟踪执行（调试）
bash -x install_ops_tools_centos.sh

# 查看不可见字符
cat -A install_ops_tools_centos.sh
od -c install_ops_tools_centos.sh
```

---

## 九、CentOS 系统特定建议

### 服务管理
```bash
# 传统 SysVinit
service crond start
chkconfig crond on

# 现代 systemd
systemctl start crond
systemctl enable crond
```

### SELinux 注意事项
```bash
# 检查 SELinux 状态
getenforce

# 如果阻止执行，可临时宽容模式
setenforce 0

# 或添加策略
ausearch -m avc --start recent
sealert -a /var/log/audit/audit.log
```

---

## 总结：如何选择？

| 需求              | 推荐方式                          | 不推荐                                     |
| ----------------- | --------------------------------- | ------------------------------------------ |
| **日常手动执行**  | `./script.sh`                     | `sh script.sh`（可能不兼容）               |
| **临时测试**      | `bash script.sh`                  | `source script.sh`（污染环境）             |
| **后台长期运行**  | `nohup ./script.sh &`             | 直接 `&`（终端关闭后可能挂）               |
| **定时任务**      | `cron` + 绝对路径                 | `at`（一次性）                             |
| **远程执行**      | `ssh user@host '/path/script.sh'` | `ssh user@host 'bash -s' < script`（复杂） |
| **加载函数/变量** | `source script.sh`                | `./script.sh`（子进程无效）                |
| **绝对安全**      | 先 `vim` 查看，再 `./`            | `curl\|bash`（无检查）                     |

**对于你的情况**，在 `/root` 目录下，最简单安全的方式是：
```bash
[root@k8s-master ~]# ./install_ops_tools_centos.sh
```

需要记录日志则：
```bash
[root@k8s-master ~]# ./install_ops_tools_centos.sh 2>&1 | tee install.log
```