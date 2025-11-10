---
title: cmd
description: 
date: 2025-11-10
categories:
    - 
    - 
---

# Windows CMD 命令

以下是分类整理的常用命令提示符（CMD）命令，包含功能说明和实用示例。

---

## 一、系统信息类命令

| 命令          | 功能说明                 | 示例             |
| ------------- | ------------------------ | ---------------- |
| `systeminfo`  | 显示详细的系统配置信息   | `systeminfo`     |
| `ver`         | 显示Windows版本号        | `ver`            |
| `hostname`    | 显示计算机名称           | `hostname`       |
| `whoami`      | 显示当前用户名及权限     | `whoami`         |
| `ipconfig`    | 显示网络配置信息         | `ipconfig /all`  |
| `driverquery` | 显示已安装的驱动程序列表 | `driverquery /v` |

---

## 二、文件和目录操作

### 基础命令
| 命令             | 功能说明               | 常用示例                        |
| ---------------- | ---------------------- | ------------------------------- |
| `dir`            | 列出目录内容           | `dir /a:h`（显示隐藏文件）      |
| `cd`             | 切换目录               | `cd /d D:\folder`（跨盘符切换） |
| `md` 或 `mkdir`  | 创建新目录             | `md "New Folder"`               |
| `rd` 或 `rmdir`  | 删除目录               | `rd /s /q folder`（强制删除）   |
| `copy`           | 复制文件               | `copy file.txt D:\backup`       |
| `xcopy`          | 增强复制（支持目录树） | `xcopy /s /e C:\src D:\dst`     |
| `robocopy`       | 高级复制（推荐）       | `robocopy /mir C:\src D:\dst`   |
| `move`           | 移动/重命名文件        | `move file.txt newname.txt`     |
| `del` 或 `erase` | 删除文件               | `del /s *.tmp`（递归删除）      |
| `type`           | 显示文件内容           | `type readme.txt`               |
| `find`           | 在文件中搜索字符串     | `find "error" log.txt`          |
| `findstr`        | 增强搜索（支持正则）   | `findstr /s "hello" *.txt`      |

### 实用示例
```batch
:: 批量重命名.txt为.log
ren *.txt *.log

:: 快速清空文件夹内容
del /q *.* && for /d %i in (*) do @rd /s /q "%i"

:: 查找大文件（按大小排序）
dir /s /b /a-d | findstr /r "^.*\\[^\\]*$" > files.txt
```

---

## 三、网络命令

| 命令       | 功能说明                   | 常用参数                        |
| ---------- | -------------------------- | ------------------------------- |
| `ping`     | 测试网络连通性             | `ping -t baidu.com`（持续测试） |
| `tracert`  | 路由追踪                   | `tracert 8.8.8.8`               |
| `nslookup` | DNS查询                    | `nslookup google.com`           |
| `netstat`  | 显示网络连接状态           | `netstat -ano`（显示PID）       |
| `arp`      | 显示ARP缓存表              | `arp -a`                        |
| `ipconfig` | 网络配置管理               | `ipconfig /release`（释放IP）   |
| `netsh`    | 网络配置工具               | `netsh wlan show profiles`      |
| `ftp`      | FTP客户端                  | `ftp 192.168.1.1`               |
| `telnet`   | Telnet客户端（需开启功能） | `telnet baidu.com 80`           |
| `pathping` | 高级路由追踪               | `pathping baidu.com`            |

---

## 四、进程与服务管理

| 命令       | 功能说明         | 示例                                |
| ---------- | ---------------- | ----------------------------------- |
| `tasklist` | 显示所有运行进程 | `tasklist /fi "memusage gt 100000"` |
| `taskkill` | 终止进程         | `taskkill /im chrome.exe /f`        |
| `start`    | 启动新进程       | `start notepad.exe`                 |
| `net`      | 服务管理         | `net start`（查看服务）             |
| `sc`       | 服务控制管理器   | `sc query`（查询服务状态）          |
| `wmic`     | WMI命令行工具    | `wmic process list brief`           |
| `shutdown` | 关机/重启/注销   | `shutdown /s /t 0`（立即关机）      |

**进程操作示例：**
```batch
:: 强制结束占用端口的进程
netstat -ano | findstr ":8080"
taskkill /pid 1234 /f

:: 批量结束同类型进程
taskkill /im chrome.exe /t /f
```

---

## 五、系统管理与维护

| 命令       | 功能说明               | 常用示例                       |
| ---------- | ---------------------- | ------------------------------ |
| `sfc`      | 系统文件检查器         | `sfc /scannow`                 |
| `chkdsk`   | 磁盘检查工具           | `chkdsk C: /f /r`              |
| `diskpart` | 磁盘分区管理工具       | `diskpart`（进入交互模式）     |
| `format`   | 格式化磁盘             | `format E: /q /fs:ntfs`        |
| `cipher`   | 文件加密/擦除空闲空间  | `cipher /w:C`                  |
| `gpresult` | 显示组策略结果         | `gpresult /r`                  |
| `gpupdate` | 刷新组策略             | `gpupdate /force`              |
| `powercfg` | 电源配置管理           | `powercfg /energy`（生成报告） |
| `msconfig` | 系统配置（图形界面）   | `msconfig`                     |
| `reg`      | 注册表编辑器（命令行） | `reg query HKLM\Software`      |

---

## 六、用户与权限管理

| 命令             | 功能说明           | 示例                             |
| ---------------- | ------------------ | -------------------------------- |
| `net user`       | 用户账户管理       | `net user`（列出用户）           |
| `net localgroup` | 本地组管理         | `net localgroup administrators`  |
| `runas`          | 以其他用户身份运行 | `runas /user:admin cmd`          |
| `icacls`         | 文件权限管理       | `icacls file.txt /grant Users:F` |
| `takeown`        | 获取文件所有权     | `takeown /f C:\locked`           |

---

## 七、性能与调试工具

| 命令       | 功能说明               | 说明                |
| ---------- | ---------------------- | ------------------- |
| `perfmon`  | 性能监视器（GUI）      | `perfmon`           |
| `resmon`   | 资源监视器（GUI）      | `resmon`            |
| `eventvwr` | 事件查看器（GUI）      | `eventvwr`          |
| `dxdiag`   | DirectX诊断工具（GUI） | `dxdiag`            |
| `msinfo32` | 系统信息（GUI）        | `msinfo32`          |
| `wmic`     | 硬件信息查询           | `wmic cpu get name` |

---

## 八、其他实用命令

| 命令          | 功能说明           | 技巧                       |
| ------------- | ------------------ | -------------------------- |
| `cls`         | 清屏               | `cls`                      |
| `color`       | 更改控制台颜色     | `color 0a`（黑底绿字）     |
| `title`       | 更改窗口标题       | `title MyCMD`              |
| `prompt`      | 更改命令提示符     | `prompt $p$g`              |
| `date`/`time` | 显示/设置日期时间  | `date`                     |
| `vol`         | 显示卷标和序列号   | `vol C:`                   |
| `label`       | 更改磁盘卷标       | `label C: System`          |
| `tree`        | 以树形显示目录结构 | `tree /f > structure.txt`  |
| `fc`          | 文件比较           | `fc file1.txt file2.txt`   |
| `comp`        | 二进制文件比较     | `comp file1.bin file2.bin` |
| `where`       | 查找文件位置       | `where notepad.exe`        |

---

## 九、批处理常用技巧

```batch
:: 注释说明
@echo off          :: 关闭命令回显
setlocal           :: 本地化环境变量
cd /d "%~dp0"      :: 切换到脚本所在目录

:: 延时
timeout /t 5       :: 等待5秒

:: 条件判断
if exist file.txt (echo 存在) else (echo 不存在)

:: 循环
for %%i in (*.txt) do echo %%i

:: 获取管理员权限
net session >nul 2>&1 || (powershell start -verb runas '%0' & exit)
```

---

## 使用技巧

1. **快速复制粘贴**：在CMD窗口右键选择"标记"复制，右键粘贴
2. **历史命令**：按`F7`查看历史，使用`↑`/`↓`键浏览
3. **自动补全**：按`Tab`键自动补全文件名
4. **管道符**：`|` 将一个命令的输出作为另一个的输入
5. **重定向**：`>` 输出到文件，`>>` 追加到文件

