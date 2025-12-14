---
title: zabbix
description: 
date: 2025-12-14
categories:
    - 
    - 
---

1.  **操作系统版本修正**：您提供的 Yum 源是 `el7` (RHEL 7/CentOS 7)，但 Zabbix 7.x 版本通常依赖较新的 PHP 和库文件。**建议使用 Rocky Linux 9 / AlmaLinux 9 / openEuler 22.03+** 等现代系统。本教程将基于 **Rocky Linux 9/CentOS Stream 9** 环境编写（目前最主流的企业级免费 Linux 环境）。
2.  **配置持久化**：补充了防火墙和 SELinux 的永久关闭方法。
3.  **数据库断点续接**：您在 `mysql_secure_installation` 处中断，我补充了后续的建库、授权、导入数据和配置文件修改步骤。
4.  **Nginx 配置**：补充了 Nginx 与 PHP-FPM 的联动配置。

-----

# Zabbix 7.0 LTS 自动化监控告警系统部署文档

**实验目的**：掌握 Zabbix 自动化监控系统的全流程部署、配置与基础使用。
**系统环境**：Rocky Linux 9 / CentOS Stream 9 / openEuler (推荐)
**软件版本**：Zabbix 7.0 LTS, Nginx, PHP 8.x, MariaDB/MySQL

-----

## 1\. 基础系统环境配置

### 1.1 配置 IP 地址

确保服务器网络通畅。

```bash
# 查看网卡名称（假设为 ens33）
ip addr

# 配置静态 IP (请根据实际网络环境修改 IP)
nmcli connect mod ens33 ipv4.addresses 192.168.200.100/24
nmcli connect mod ens33 ipv4.gateway 192.168.200.1
nmcli connect mod ens33 ipv4.dns "8.8.8.8 114.114.114.114"
nmcli connect mod ens33 ipv4.method manual
nmcli connect up ens33
```

### 1.2 关闭防火墙和 SELinux

这是为了避免实验过程中出现权限拦截问题。

```bash
# 1. 停止并禁用防火墙
systemctl stop firewalld
systemctl disable firewalld

# 2. 临时关闭 SELinux
setenforce 0

# 3. 永久关闭 SELinux (修改配置文件，防止重启失效)
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

### 1.3 安装基础工具

```bash
yum update -y
yum install vim tar net-tools wget curl -y
```

-----

## 2\. 安装 Zabbix 及相关组件

### 2.1 安装 Zabbix 官方源

**注意**：这里使用 Zabbix 7.0 LTS (长期支持版) 和 RHEL 9 (el9) 版本的源。如果您使用的是 CentOS 7，请务必升级系统，因为旧版 PHP 不支持新版 Zabbix。

```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.0/rhel/9/x86_64/zabbix-release-7.0-5.el9.noarch.rpm
yum clean all
```

### 2.2 安装 Zabbix Server, Web 前端, Agent 和 数据库

```bash
yum install zabbix-server-mysql zabbix-web-mysql zabbix-nginx-conf zabbix-sql-scripts zabbix-selinux-policy zabbix-agent mariadb-server langpacks-zh_CN -y
```

*注：这里推荐使用 `mariadb-server`，它是 RHEL/CentOS 系列默认且兼容性最好的数据库。*

-----

## 3\. 数据库初始化与配置

### 3.1 启动数据库

```bash
systemctl start mariadb
systemctl enable mariadb
```

### 3.2 数据库安全初始化

```bash
mariadb-secure-installation
```

  * `Enter current password for root`: 初次安装为空，直接**回车**。
  * `Switch to unix_socket authentication`: 输入 **n**。
  * `Change the root password?`: 输入 **y**，设置一个新的 root 密码（例如：`Root123!`）。
  * 后续选项（移除匿名用户、禁止远程 root 登录等）：建议全部输入 **y**。

### 3.3 创建 Zabbix 专用库和用户

登录数据库并执行 SQL 语句：

```bash
# 登录数据库 (输入刚才设置的密码)
mysql -u root -p

# 在 mysql> 提示符下执行以下命令：
create database zabbix character set utf8mb4 collate utf8mb4_bin;
create user zabbix@localhost identified by 'Zabbix@123';
grant all privileges on zabbix.* to zabbix@localhost;
set global log_bin_trust_function_creators = 1;
quit;
```

*注意：`Zabbix@123` 是 Zabbix 连接数据库的密码，请记住它。*

### 3.4 导入 Zabbix 初始数据

这一步将把 Zabbix 需要的表结构导入数据库。

```bash
zcat /usr/share/doc/zabbix-sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
```

  * 系统会提示输入密码，请输入上一步设置的 `Zabbix@123`。
  * 此过程可能需要几秒到一分钟，请耐心等待直到命令结束。

### 3.5 关闭 log\_bin\_trust 功能 (安全建议)

数据导入完成后，关闭此选项：

```bash
mysql -u root -p -e "set global log_bin_trust_function_creators = 0;"
```

-----

## 4\. 修改配置文件

### 4.1 修改 Zabbix Server 配置

告诉 Zabbix Server 数据库密码是什么。

```bash
vim /etc/zabbix/zabbix_server.conf
```

找到 `DBPassword=` 这一行（可以使用 `/DBPassword` 搜索），取消注释并修改：

```ini
DBPassword=Zabbix@123
```

*(保存退出: `Esc` -\> `:wq`)*

### 4.2 修改 Nginx 配置

配置前端访问端口和域名。

```bash
vim /etc/nginx/conf.d/zabbix.conf
```

找到 `listen` 和 `server_name` 所在行，取消注释（去掉 `#`）：

```nginx
listen 80;
server_name example.com;  # 这里也可以写服务器IP，或者保持 example.com
```

*(保存退出: `Esc` -\> `:wq`)*

-----

## 5\. 启动服务与最终设置

### 5.1 重启所有相关服务

```bash
systemctl restart zabbix-server zabbix-agent nginx php-fpm
systemctl enable zabbix-server zabbix-agent nginx php-fpm
```

### 5.2 检查服务状态

确保没有报错（Active: active (running)）：

```bash
systemctl status zabbix-server
```

-----

## 6\. Web 端初始化 (图形化界面)

1.  **访问页面**：
    打开浏览器，访问 `http://192.168.200.100` (替换为您服务器的实际IP)。
2.  **欢迎界面**：
    点击 "Next step"。
3.  **必要条件检查**：
    确保所有项都是 "OK"。如果有报错，通常是 PHP 扩展未安装或配置错误。点击 "Next step"。
4.  **配置数据库连接**：
      * Database type: MySQL
      * Database host: localhost
      * Database name: zabbix
      * User: zabbix
      * **Password**: 输入之前设置的 `Zabbix@123`
      * 点击 "Next step"。
5.  **设置 Zabbix Server 信息**：
      * Zabbix server name: 可以随便填，例如 "MyMonitor"
      * Time zone: 选择 **Asia/Shanghai**
      * 点击 "Next step"。
6.  **完成安装**：
    确认信息无误，点击 "Next step" -\> "Finish"。

-----

## 7\. 登录与基础使用

### 7.1 登录

  * **默认账号**：`Admin` (注意首字母大写)
  * **默认密码**：`zabbix`

### 7.2 修改界面语言为中文

1.  登录后，点击左侧菜单底部的 **User settings** (或者点击右上角的小人图标 -\> User profile)。
2.  在 **Language** 选项中，选择 **Chinese (zh\_CN)**。
3.  点击 **Update**。

### 7.3 解决中文乱码问题 (如果出现)

如果在图表中看到方块乱码，需要替换字体：

1.  在 Windows 电脑找到 `C:\Windows\Fonts\simkai.ttf` (楷体) 或其他中文字体。
2.  上传该字体文件到服务器 `/usr/share/zabbix/assets/fonts/` 目录。
3.  修改软链接或配置指向新字体。

### 7.4 添加被监控主机 (简单示例)

1.  点击左侧菜单 **数据采集 (Data collection)** -\> **主机 (Hosts)**。
2.  点击右上角 **创建主机 (Create host)**。
3.  **主机名称**：填写被监控服务器的主机名或 IP。
4.  **模板**：选择 `Linux by Zabbix agent`。
5.  **主机群组**：选择 `Linux servers`。
6.  **接口**：点击添加 -\> Agent，输入被监控机的 IP 地址，端口 10050。
7.  点击 **添加**。

