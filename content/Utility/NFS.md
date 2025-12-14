---
title: NFS
description: 
date: 2025-12-14
categories:
    - 
    - 
---
# NFS 文件共享服务部署指南

## 1\. 工作原理

NFS（Network File System）服务启动时会随机选择端口，因此客户端无法直接与NFS通信。NFS通过\*\*RPC（Remote Procedure Call）\*\*服务来解决这个问题：

1.  NFS启动时，将随机端口注册到RPC服务（默认端口111）。
2.  客户端连接NFS前，先向服务端的RPC服务查询NFS端口。
3.  RPC告知客户端NFS的端口，客户端再通过该端口建立连接进行数据传输。

> **注意**：因此在启动NFS之前，必须先启动RPC服务（`rpcbind`）。

-----

## 2\. 环境准备

为确保实验顺利，建议在服务端和客户端均执行以下操作（生产环境需通过防火墙放行特定端口）：

```bash
# 1. 关闭防火墙
systemctl stop firewalld
systemctl disable firewalld

# 2. 关闭SELinux
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

-----

## 3\. NFS 服务端配置

### 3.1 安装软件包

服务端需要安装 `nfs-utils`（核心包）和 `rpcbind`（RPC服务）。

```bash
yum -y install nfs-utils rpcbind
```

### 3.2 启动服务

**注意启动顺序**：必须先启动 `rpcbind`，再启动 `nfs`。

```bash
# 启动 rpcbind 并设置开机自启
systemctl start rpcbind
systemctl enable rpcbind

# 启动 nfs 并设置开机自启
systemctl start nfs
systemctl enable nfs
```

### 3.3 检查服务状态

```bash
# 查看RPC端口（111）是否开启
lsof -i:111

# 查看本机RPC注册情况
rpcinfo -p localhost
```

### 3.4 配置共享目录

NFS的主配置文件是 `/etc/exports`。

1.  **创建共享目录**

    ```bash
    mkdir /data
    ```

2.  **编辑配置文件**

    ```bash
    vim /etc/exports
    ```

    **写入内容示例：**

    ```text
    /data  192.168.200.0/24(rw,sync)
    ```

    *解释：共享 `/data` 目录，允许 `192.168.200.0` 网段的所有主机访问，权限为读写（rw）且同步写入（sync）。*

3.  **生效配置**
    修改配置文件后，无需重启服务，使用 `exportfs` 命令即可生效：

    ```bash
    exportfs -rv
    ```

      * `-r`: 重新挂载
      * `-v`: 显示详细信息

4.  **本地验证**

    ```bash
    showmount -e localhost
    # 预期输出：Export list for localhost: /data 192.168.200.0/24
    ```

-----

## 4\. NFS 客户端配置

### 4.1 安装软件包

客户端也需要安装相关工具才能挂载。

```bash
# 纠错：原文写成了 nfs-utlis，正确为 nfs-utils
yum -y install nfs-utils rpcbind
```

### 4.2 发现共享

查看服务端（假设IP为 192.168.200.100）共享了哪些目录：

```bash
showmount -e 192.168.200.100
```

### 4.3 挂载目录

将服务端的 `/data` 挂载到本地的 `/mnt` 目录。

```bash
mount -t nfs 192.168.200.100:/data /mnt

# 查看挂载情况
df -hT | grep nfs
```

### 4.4 权限问题处理（重点）

**现象**：客户端挂载成功，但在 `/mnt` 下创建文件时提示 `Permission denied`（权限不够）。

**原因**：
NFS默认配置了 `root_squash`（root用户压缩）。即：客户端使用 `root` 身份访问时，服务端会将其映射为匿名用户 `nfsnobody`（UID通常为65534）。如果服务端 `/data` 目录属于 `root` 用户，则 `nfsnobody` 没有写入权限。

**解决方法（二选一）**：

  * **方法A：修改服务端目录权限（推荐）**
    在**服务端**执行，将共享目录所有者改为 `nfsnobody`：

    ```bash
    chown -R nfsnobody /data/
    # 此时客户端即可正常写入，文件属主会自动变为 nfsnobody
    ```

  * **方法B：修改配置文件权限参数（不安全）**
    在 `/etc/exports` 中添加 `no_root_squash` 参数，允许客户端保留root权限：

    ```text
    /data 192.168.200.0/24(rw,sync,no_root_squash)
    ```

-----

## 5\. 卸载与强制卸载

正常卸载：

```bash
umount /mnt
```

**故障处理**：如果提示 `umount: /mnt: target is busy`（设备忙），说明有进程正在使用该目录，或者网络已断开导致挂死。
**强制卸载命令**：

```bash
umount -lf /mnt
```

  * `-l` (lazy): 立即从文件系统层次结构中分离文件系统，并在不再繁忙时清理所有引用。
  * `-f` (force): 强制卸载（主要用于NFS不可达时）。

-----

## 6\. 配置文件参数详解

`/etc/exports` 中常用参数说明：

| 参数                 | 说明           | 备注                                                  |
| :------------------- | :------------- | :---------------------------------------------------- |
| **rw**               | 读写权限       | Read-Write                                            |
| **ro**               | 只读权限       | Read-Only                                             |
| **sync**             | 同步写入       | 数据同时写入内存和硬盘，数据安全但速度稍慢            |
| **async**            | 异步写入       | 数据先暂存内存，稍后写入硬盘，速度快但断电可能丢数据  |
| **root\_squash**     | 压缩root权限   | **默认启用**。客户端root访问时，服务端映射为nfsnobody |
| **no\_root\_squash** | 不压缩root权限 | 客户端root访问时，服务端保留root权限（高风险）        |
| **all\_squash**      | 压缩所有用户   | 无论客户端用什么身份，服务端都将其映射为匿名用户      |
| **anonuid/anongid**  | 指定匿名ID     | 配合 `all_squash` 使用，指定映射后的具体UID/GID       |

-----

## 7\. 常用管理命令汇总

| 命令                | 作用                                       |
| :------------------ | :----------------------------------------- |
| `rpcinfo -p`        | 查看RPC注册的端口信息                      |
| `showmount -e [IP]` | 查看指定IP的NFS共享列表                    |
| `exportfs -rv`      | 重新加载配置文件并显示过程（无需重启服务） |
| `exportfs -auv`     | 暂停/卸载所有当前的共享                    |

