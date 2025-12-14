---
title: Ansible
description: 
date: 2025-12-14
categories:
    - 
    - 
---
Ansible搭建部署

### 1. 安装必要的依赖和 EPEL 仓库

Ansible 不在 CentOS 默认的软件仓库中，需要先安装 EPEL 仓库：

bash

```bash
# 安装EPEL仓库
sudo yum install -y epel-release

# 更新系统包
sudo yum update -y
```

### 2. 安装 Ansible

bash

```bash
# 安装Ansible
sudo yum install -y ansible
```

### 3. 验证 Ansible 安装

bash

```bash
# 检查Ansible版本
ansible --version
```

成功安装后，会显示类似以下的信息：

plaintext

```plaintext
ansible 2.9.x
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/root/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/site-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.5 (default, Jun 20 2019, 20:27:34) [GCC 4.8.5 20150623 (Red Hat 4.8.5-36)]
```

### 4. 配置 Ansible

#### 4.1 配置主机清单 (inventory)

Ansible 使用 inventory 文件来管理它所控制的主机，默认位置是`/etc/ansible/hosts`

bash

```bash
# 编辑主机清单文件
sudo vi /etc/ansible/hosts
```

在文件中添加需要管理的主机，例如：

ini

```ini
# 定义一个名为webservers的主机组
[webservers]
192.168.1.100
192.168.1.101

# 定义一个名为databases的主机组
[databases]
192.168.1.200
db.example.com
```

#### 4.2 配置 SSH 免密码登录

为了让 Ansible 能够无密码登录到被管理节点，需要配置 SSH 密钥认证：

bash

```bash
# 生成SSH密钥对(如果还没有)
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# 将公钥复制到被管理节点
ssh-copy-id username@remote_host
```

#### 1. 主机密钥验证问题（最常见）

**错误信息**：

- `The authenticity of host 'xxx' can't be established.`（首次连接提示确认主机密钥）
- `Host key verification failed.`（主机密钥验证失败）

**原因**：
SSH 首次连接陌生主机时，会提示确认主机密钥（防止中间人攻击），但 Ansible 在非交互式模式下无法手动输入`yes`，导致连接中断；或主机密钥已变更，与本地存储的不一致。

**解决方法**：
临时关闭 Ansible 的主机密钥检查（适合测试环境）：

bash

```bash
# 临时生效（仅当前终端）
export ANSIBLE_HOST_KEY_CHECKING=False

# 永久生效（修改Ansible配置文件）
sudo vi /etc/ansible/ansible.cfg
# 找到并修改以下配置（去掉注释并设为False）
host_key_checking = False
```

#### 2. 权限拒绝（认证失败）

**错误信息**：
`Permission denied (publickey,gssapi-keyex,gssapi-with-mic,password).`

**原因**：
Ansible 默认使用 SSH 密钥认证，但目标主机未配置当前用户的公钥（或公钥配置错误），导致认证失败。



**解决方法**：
确保 Ansible 控制节点的公钥已正确复制到目标主机：

bash

```bash
# 1. 生成SSH密钥（如果还没有）
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa

# 2. 将公钥复制到目标主机（替换为实际的用户名和IP）
# 例如：复制到192.168.200.100，远程用户为root
ssh-copy-id root@192.168.200.100

# 3. 验证SSH是否能无密码登录
ssh root@192.168.200.100  # 无需输入密码即登录成功说明配置正确
```



**注意**：

- 目标主机的

  ```
  ~/.ssh
  ```

  目录权限必须为

  ```
  700
  ```

  ，

  ```
  ~/.ssh/authorized_keys
  ```

  权限必须为

  ```
  600
  ```

  ，否则会认证失败：

  bash

  ```bash
  # 在目标主机执行（修复权限）
  chmod 700 ~/.ssh
  chmod 600 ~/.ssh/authorized_keys
  ```

#### 3. 域名解析失败

**错误信息**：
`ssh: Could not resolve hostname db.example.com: Name or service not known`

**原因**：
Ansible 清单（`/etc/ansible/hosts`）中配置的`db.example.com`域名无法被解析（本地 DNS 或`/etc/hosts`中无记录）。

**解决方法**：

- 方法 1：用 IP 地址替换域名（简单直接）：
  编辑`/etc/ansible/hosts`，将`db.example.com`改为实际 IP（如`192.168.200.xxx`）。

- 方法 2：手动添加域名解析：

  bash

  ```bash
  sudo vi /etc/hosts
  # 添加一行（替换为实际IP和域名）
  192.168.200.xxx  db.example.com
  ```

#### 4. 网络不可达

**错误信息**：
`ssh: connect to host 192.168.200.200 port 22: No route to host`

**原因**：
目标主机（`192.168.200.200`）与控制节点网络不通，可能是：

- IP 地址错误（目标主机实际 IP 不是这个）；
- 子网掩码 / 网关配置错误；
- 目标主机防火墙阻止了 SSH 端口（22）；
- 目标主机未开机或网络未连接。

**解决方法**：

1. 先验证网络连通性：

   bash

   ```bash
   #  ping目标主机（检查是否在线）
   ping 192.168.200.200
   
   # 检查SSH端口是否开放（22端口是否可达）
   telnet 192.168.200.200 22
   ```

2. 若 ping 不通：确认目标主机 IP 是否正确、是否开机、网络是否连接。

3. 若 ping 通但 telnet 失败：检查目标主机防火墙是否允许 22 端口：

   bash

   ```bash
   # 在目标主机执行（开放22端口）
   sudo firewall-cmd --add-port=22/tcp --permanent
   sudo firewall-cmd --reload
   ```

### 修复后测试

解决上述问题后，重新执行测试命令验证：

bash

```bash
# 测试所有主机连接
ansible all -m ping

# 测试webservers组连接
ansible webservers -m ping
```

**测试 Ansible 对该主机的连接**：

bash

```bash
ansible 192.168.200.50 -m ping
```

若输出类似以下内容，说明连接成功：

plaintext

```plaintext
192.168.200.100 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

###  区分 “控制节点” 和 “目标主机”

- **控制节点**：你当前登录并运行 Ansible 命令的机器（比如你操作的`client2`），Ansible 程序安装在这里，负责发起管理操作。
- **目标主机**：`webservers`、`databases`组里列出的远程服务器（如`192.168.200.50`），它们是被管理的对象，Ansible 通过 SSH 远程控制它们。

### 验证 “目标主机是否有效”

如果不确定某个 IP 是否是 “可用的目标主机”，可以通过以下方式验证：

bash

```bash
# 1. 先测试网络连通性（控制节点ping目标主机）
ping 192.168.200.50

# 2. 测试SSH是否能连接（控制节点SSH登录目标主机）
ssh root@192.168.200.50  # 若免密配置正确，无需输密码即可登录

# 3. 用Ansible的ping模块测试（最直接的验证）
ansible 192.168.200.50 -m ping  # 返回"pong"说明是有效目标主机
```

总结：`webservers`组里的 IP 就是 Ansible 要管理的 “目标主机”（Web 服务器类型），你通过 Ansible 命令操作这些 IP 时，它们就是本次操作的目标。

### 5. 测试 Ansible 配置

bash

```bash
# 测试与所有主机的连接
ansible all -m ping

# 测试与webservers组主机的连接
ansible webservers -m ping

# 在所有主机上执行date命令
ansible all -a "date"
```

如果配置正确，会显示类似以下的成功信息：

plaintext

```plaintext
192.168.1.100 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

#### 1. 确认目标主机是否开机并正常运行

- 直接登录目标主机（192.168.200.200）的控制台，检查是否处于开机状态（最基础的排查）。
- 若无法直接访问控制台，可尝试通过其他同网段主机 ping 该 IP，确认是否仅 192.168.200.50 无法连接。

#### 2. 验证目标主机的 IP 地址配置

登录目标主机（若能访问），检查其 IP 地址是否正确：

bash

```bash
# 在目标主机（192.168.200.200）上执行
ip addr show
```

- 确认是否存在`192.168.200.200`的 IP 地址（通常在`eth0`或`ens33`等网卡下）。

- 若 IP 地址错误，重新配置正确的 IP（以 CentOS 为例）：

  bash

  ```bash
  # 临时设置IP（重启失效）
  ip addr add 192.168.200.200/24 dev eth0  # 假设子网掩码为255.255.255.0，网卡为eth0
  
  # 永久配置（编辑网卡文件，不同系统路径可能不同）
  sudo vi /etc/sysconfig/network-scripts/ifcfg-eth0
  # 修改或添加：
  IPADDR=192.168.200.200
  NETMASK=255.255.255.0
  # 重启网络服务
  sudo systemctl restart network
  ```

#### 3. 检查子网与网关配置

确保 192.168.200.50 和 192.168.200.200 在同一子网：



- 在 192.168.200.50 上执行`ip addr show`，查看子网掩码（如`255.255.255.0`）。
- 若子网掩码为`255.255.255.0`，则两台主机的 IP 前三位（192.168.200）必须一致（当前符合），无需网关即可直接通信。
- 若子网掩码不同（如目标主机是`255.255.254.0`），则可能不在同一网段，需检查网关是否正确配置并确保网关可达。

#### 4. 排查物理连接与网络设备

- 检查目标主机的网线是否插紧，交换机 / 路由器对应端口是否亮灯（正常工作状态）。
- 尝试更换目标主机的网线或交换机端口，排除硬件故障。

#### 5. 检查防火墙是否禁止 ICMP（ping）

即使网络连通，防火墙也可能阻止 ICMP 协议导致 ping 失败：

- 在目标主机（192.168.200.200）上开放 ICMP

  ：

  bash

  ```bash
  # CentOS 7默认防火墙是firewalld，允许ICMP（ping）
  sudo firewall-cmd --add-icmp-block-inversion --permanent  # 允许所有ICMP请求
  sudo firewall-cmd --reload
  ```

- 若目标主机使用 iptables 防火墙：

  bash

  ```bash
  sudo iptables -A INPUT -p icmp --icmp-type echo-request -j ACCEPT
  sudo service iptables save
  ```

#### 6. 清除本地 ARP 缓存（可选）

若存在 ARP 缓存异常，可能导致 IP 与 MAC 地址映射错误：

bash

```bash
# 在192.168.200.50上执行
arp -d 192.168.200.200  # 清除目标主机的ARP缓存
ping 192.168.200.200   # 重新尝试ping
```

### 三、验证结果

完成上述排查后，重新在 192.168.200.50 上执行`ping 192.168.200.200`，若显示类似以下内容，说明网络连通：

plaintext

```plaintext
64 bytes from 192.168.200.200: icmp_seq=1 ttl=64 time=0.567 ms
64 bytes from 192.168.200.200: icmp_seq=2 ttl=64 time=0.452 ms
```

### 6. 基本 Ansible 配置文件修改 (可选)

Ansible 的主配置文件是`/etc/ansible/ansible.cfg`，可以根据需要修改：

bash

```bash
sudo vi /etc/ansible/ansible.cfg
```

常用的配置项：

- `remote_user`: 默认的远程用户名
- `host_key_checking`: 是否检查主机密钥，设置为 False 可以跳过检查
- `timeout`: SSH 连接超时时间

Ansible主机清单中自定义的主机分组两个分别是：webservers和databases

[webservers]组：运行Web服务的服务器（批量安装Web服务软件、统一配置网站文件、管理Web服务状态、部署网页代码）

[databases]组：运行数据库服务的服务器（批量安装数据库软件、配置数据库参数、管理数据库服务状态、执行数据库命令）

### 如何使用这两个分组？

#### 1. 基础用法：直接对组执行命令

通过 `ansible 组名` 替代 `ansible 单个IP`，即可批量操作组内所有主机。

示例 1：测试组内主机的连接性

bash

```bash
# 测试 webservers 组所有主机是否能被 Ansible 连接
ansible webservers -m ping

# 测试 databases 组所有主机是否能被 Ansible 连接
ansible databases -m ping
```

（如果组内某台主机不可达，会单独显示错误，不影响组内其他主机）

**示例 2：批量执行系统命令**

bash

```bash
# 查看 webservers 组所有主机的系统版本（如 CentOS 版本）
ansible webservers -a "cat /etc/redhat-release"

# 查看 databases 组所有主机的磁盘空间（检查数据库存储是否充足）
ansible databases -a "df -h /var/lib/mysql"  # /var/lib/mysql 是 MySQL 数据默认目录
```

#### 2. 进阶用法：用模块批量管理服务 / 软件

结合 Ansible 模块（如 `yum` 安装软件、`service` 管理服务），对组内主机执行更复杂的操作。

**示例 1：给 webservers 组批量部署 Nginx**

bash

```bash
# 1. 批量安装 Nginx（确保所有 Web 服务器都有 Nginx）
ansible webservers -m yum -a "name=nginx state=installed"

# 2. 批量启动 Nginx 并设置开机自启
ansible webservers -m service -a "name=nginx state=started enabled=yes"

# 3. 批量修改 Nginx 配置（比如替换默认首页）
# 先在控制节点创建一个简单的 index.html
echo "<h1>Hello from Ansible</h1>" > /tmp/index.html
# 再用 copy 模块传到所有 Web 服务器的 Nginx 根目录
ansible webservers -m copy -a "src=/tmp/index.html dest=/usr/share/nginx/html/index.html mode=0644"
```

**示例 2：给 databases 组批量部署 MySQL**

bash

```bash
# 1. 批量安装 MySQL 服务器
ansible databases -m yum -a "name=mariadb-server state=installed"  # CentOS 中 MySQL 通常用 mariadb 替代

# 2. 批量启动 MySQL 并设置开机自启
ansible databases -m service -a "name=mariadb state=started enabled=yes"

# 3. 批量初始化 MySQL（比如设置 root 密码，需结合 shell 模块）
ansible databases -m shell -a "mysqladmin -u root password '123456'"  # 简单示例，生产环境需更安全的方式
```

#### 3. 高级用法：用 Playbook 实现自动化任务

对于复杂的批量操作（如 “部署 Web 服务 + 数据库服务的完整架构”），可以编写 YAML 格式的 Playbook，指定对哪个组执行哪些任务。

**示例：创建一个部署 Web 服务的 Playbook（`deploy_web.yml`）**

yaml

```yaml
---
- name: 给 webservers 组部署 Nginx 并配置网站
  hosts: webservers  # 明确指定对 webservers 组操作
  tasks:
    - name: 安装 Nginx
      yum:
        name: nginx
        state: installed

    - name: 启动 Nginx 并设置开机自启
      service:
        name: nginx
        state: started
        enabled: yes

    - name: 复制自定义首页到所有 Web 服务器
      copy:
        src: /tmp/index.html  # 控制节点上的文件
        dest: /usr/share/nginx/html/index.html
        mode: 0644

    - name: 重启 Nginx 使配置生效
      service:
        name: nginx
        state: restarted
```

执行这个 Playbook，会自动对 `webservers` 组所有主机执行上述 4 个任务：

bash

```bash
ansible-playbook deploy_web.yml
```

### 四、注意事项

1. **组内主机需可用**：如果组内有不可达的主机（如你提到的 `192.168.200.200`），执行组操作时会提示错误，建议先注释或删除无效主机（加 `#` 注释）。

2. **分组可自定义**：`webservers` 和 `databases` 只是示例，你可以根据需求创建其他组，比如 `appservers`（应用服务器）、`monitor`（监控服务器）等。

3. 组可以嵌套

   ：复杂场景下，组可以包含其他组，例如：

   ini

   ```ini
   [webservers]
   192.168.200.50
   
   [databases]
   192.168.200.201
   
   [all_servers:children]  # 嵌套组，包含上面两个组
   webservers
   databases
   ```

   之后可以用

   ```
   ansible all_servers -m ping
   ```

   测试所有服务器。

