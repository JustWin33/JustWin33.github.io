---
title: Tcpdump
description: 
date: 2025-12-15
categories:
    - 
    - 
---
# Tcpdump 指南

> **前言：**
> 在 Linux 运维中，`tcpdump` 是上帝视角。它不撒谎，不推卸责任。当开发说“代码没问题”，网络说“交换机没问题”时，Tcpdump 的抓包文件是判定问题的唯一“法律证据”。

-----

## 第一章：核心原理与环境准备 (The Foundation)

### 1.1 工作原理 (Kernel Space)

Tcpdump 工作在**内核空间 (Kernel Space)**。它通过调用 `libpcap` 库，直接从**数据链路层**（Device Driver 之上）捕获数据。

  * **关键点**：它在 `iptables/netfilter` 之前（入站流量）或之后（出站流量）看到数据。这意味着：即使 `iptables` INPUT 链丢弃了包，`tcpdump` 通常依然能抓到入站包（证明流量到了网卡）。

### 1.2 安装与权限

大多数发行版默认不带，需手动安装：

  * **CentOS/RHEL**: `yum install -y tcpdump`
  * **Ubuntu/Debian**: `apt-get install -y tcpdump`
  * **权限**: 必须使用 `root` 用户或 `sudo`，因为它需要将网卡设置为混杂模式。

### 1.3 核心参数：防止“掉坑”的必选项

除了 `-nn`，生产环境必须了解以下参数，否则抓出来的包可能无法使用：

  * **`-s 0` (Snaplen)**: **极其重要**。默认 `tcpdump` 可能只抓取包头（如前 96 字节）。如果不加 `-s 0`（表示抓取完整包），你在 Wireshark 里看 HTTP Body 或 SQL 语句时会被截断，显示 "Packet size limited during capture"。
  * **`-c [count]`**: 抓取指定数量的包后自动停止。防止忘记关命令导致磁盘写满。
  * **`-B [size]`**: 设置内核捕获缓冲区大小（单位 KiB）。流量大时，默认缓冲区容易溢出导致丢包（Tcpdump 显示 `dropped by kernel`），可设置为 `-B 4096`。

-----

## 第二章：过滤的艺术 (The Art of Filtering)

**原则：** 过滤不仅是为了看得清，更是为了保护服务器 I/O。

### 2.1 BPF 基础语法 (回顾与补充)

| 维度      | 关键字               | 示例                  | 补充说明                 |
| :-------- | :------------------- | :-------------------- | :----------------------- |
| **Type**  | `host`               | `host 1.1.1.1`        | 同时匹配 src 和 dst      |
|           | `net`                | `net 192.168.0.0/16`  | 匹配整个网段             |
|           | `port`               | `port 80`             |                          |
|           | `portrange`          | `portrange 8000-8080` |                          |
| **Dir**   | `src`, `dst`         | `src 192.168.1.1`     |                          |
|           | `src and dst`        |                       | 限制流向                 |
| **Proto** | `tcp`, `udp`, `icmp` | `icmp`                | 还有 `ip6` (IPv6), `arp` |

### 2.2 高级位掩码过滤 (Advanced Bitmasking) —— 专家级技能

在某些极端场景（如遭受 DDoS 攻击或排查僵尸连接），你需要精准抓取 TCP 标志位。BPF 允许我们直接检查 TCP 报头。

**TCP 报头偏移量 13 是标志位 (Flags)：**
`CWR | ECE | URG | ACK | PSH | RST | SYN | FIN`

**实战公式：** `tcp[tcpflags] & tcp-syn != 0`

1.  **只抓 SYN 包（发起连接）：**
    ```bash
    tcpdump -i eth0 "tcp[tcpflags] & (tcp-syn) != 0"
    ```
2.  **只抓 RST 包（连接重置/拒绝）：**
    ```bash
    tcpdump -i eth0 "tcp[tcpflags] & (tcp-rst) != 0"
    ```
3.  **抓取 SYN-ACK 包（服务端响应连接）：**
    ```bash
    tcpdump -i eth0 "tcp[tcpflags] & (tcp-syn|tcp-ack) == (tcp-syn|tcp-ack)"
    ```
4.  **抓取 HTTP GET 请求 (匹配数据包内容)：**
    ```bash
    # 'G', 'E', 'T', ' ' 的十六进制是 47 45 54 20
    tcpdump -i eth0 "tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420"
    ```

### 2.3 基于包大小过滤

排除掉只有 Header 的小包（如 ACK 确认包），只看传输数据的包：

```bash
# 抓取长度大于 100 字节的包
tcpdump -i eth0 length > 100
```

-----

## 第三章：现场解读与应用层分析 (Live Decoding)

### 3.1 增强版输出参数

```bash
tcpdump -i eth0 -nn -v -s 0 -A
```

  * **`-A`**: 以 ASCII 码打印 payload。这是看 **HTTP Header、JSON、SQL 语句** 的神器。
  * **`-X`**: 十六进制 + ASCII 对照。当数据包含乱码或二进制时使用。
  * **`-l`**: 缓冲行模式。配合 `grep` 使用时必须加，否则 `grep` 无法实时收到数据。
      * *示例*：`tcpdump -i eth0 -nn -l -A | grep "User-Agent"`

### 3.2 常见应用层协议实战

#### 1\. 抓取 MySQL SQL 语句

MySQL 默认端口 3306，未加密连接下可直接抓取 SQL：

```bash
tcpdump -i eth0 -nn -s 0 -A port 3306 | grep -iE 'select|insert|update|delete'
```

*注意：如果是 MySQL 8.0 且开启了 SSL，只能看到乱码。*

#### 2\. 抓取 HTTP 真实请求 (REST API)

查看发往本机 8080 端口的 POST 请求内容：

```bash
tcpdump -i eth0 -nn -s 0 -A port 8080 and tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504F5354
```

或者简单点：

```bash
tcpdump -i eth0 -nn -s 0 -A port 8080 | grep "POST /"
```

#### 3\. 抓取 DNS 解析耗时

DNS 慢会导致所有服务都慢。

```bash
tcpdump -i eth0 -nn port 53
```

  * **关注点**：看 Request 和 Response 之间的时间差。

-----

## 第四章：容器与云原生环境 (Docker & Kubernetes)

**这是现代运维最头疼的地方：容器里的流量怎么抓？**

### 4.1 传统 Docker 方式

容器通常有自己的虚拟网卡（veth pair），在宿主机上看到的是 `vethXXXX`。

1.  **查找容器对应的 veth 网卡**（较麻烦）。
2.  **直接在宿主机抓 docker0 网桥**：
    ```bash
    tcpdump -i docker0 -nn port 8080
    ```
    *缺点：只能看到进出网桥的流量，看不到容器内部 localhost 通信。*

### 4.2 终极方案：进入容器网络命名空间 (Namespace)

这是最专业的方法，无需在容器内安装 tcpdump。

1.  **获取容器 PID**：
    ```bash
    docker inspect -f '{{.State.Pid}}' <container_name_or_id>
    # 假设 PID 为 12345
    ```
2.  **使用 `nsenter` 进入该 PID 的网络空间抓包**：
    ```bash
    nsenter -n -t 12345 tcpdump -i eth0 -nn -v
    ```
    *解释*：`nsenter` 让你“借用”宿主机的 `tcpdump` 二进制文件，直接运行在目标容器的“网络环境”里。**Kubernetes Pod 排查同理。**

-----

## 第五章：故障排查实战剧本 (Scenarios Expanded)

### 场景 D：排查 TCP 重传 (Retransmission) —— 网络质量差

**现象**：偶尔卡顿，传输速度不稳定。
**命令**：

```bash
tcpdump -i eth0 -nn tcp
```

**Wireshark 分析**：
在 Wireshark 过滤器输入 `tcp.analysis.retransmission`。

  * **原理**：如果看到大量红色的 "TCP Retransmission"，说明数据包丢失了，发送方在重试。
  * **原因**：物理链路拥塞、网线故障、中间设备丢包。

### 场景 E：排查 TCP 零窗口 (Zero Window) —— 接收端处理不过来

**现象**：上传文件到服务器突然中断或极慢。
**分析**：
在 tcpdump 输出中寻找 `win 0`。

  * **日志**：`Flags [.], seq 100, ack 200, win 0, length 0`
  * **含义**：接收端（服务器）告诉发送端：“我缓存满了，处理不过来了，别发了！”
  * **原因**：服务端应用（如 Tomcat/Nginx）负载过高或卡死，无法从内核缓冲区取走数据。

### 场景 F：排查 Keepalive 超时

**现象**：连接空闲一段时间后断开。
**策略**：抓取长时间的流量，观察最后的包是 FIN 还是 RST，以及发生的时间间隔。

-----

## 第六章：高级数据保存与分析流程 (Workflow)

### 6.1 生产环境“黑匣子”策略 (Rotating Capture)

在排查偶发故障（如每天凌晨 3 点报错）时，不能人工守着。

```bash
nohup tcpdump -i eth0 port 80 -C 100 -W 20 -w /data/pcap/trace.pcap &
```

  * **`-C 100`**: 每个文件 100MB。
  * **`-W 20`**: 保留最近 20 个文件（循环覆盖）。
  * **总容量**：占用 2GB 磁盘空间。
  * **结果**：你永远拥有“最近 2GB 流量”的快照。

### 6.2 结合 Wireshark 的解密 (HTTPS)

Tcpdump 抓到的 HTTPS 流量是加密的（TLS），看到的是乱码。
**破解方法**：

1.  在服务端设置环境变量 `SSLKEYLOGFILE=/tmp/sslkey.log`（支持 Nginx, Java, Curl 等）。
2.  应用会将对称加密密钥写入该文件。
3.  将 `.pcap` 和 `.log` 文件都下载到本地。
4.  **Wireshark 设置**：`Preferences` -\> `Protocols` -\> `TLS` -\> `(Pre)-Master-Secret log filename` 导入 log 文件。
5.  **见证奇迹**：Wireshark 直接显示解密后的 HTTP 明文。

-----

## 第七章：运维速查总结卡 (The Cheat Sheet)

| 场景             | 核心命令                                                     | 关键点                              |
| :--------------- | :----------------------------------------------------------- | :---------------------------------- |
| **基础调试**     | `tcpdump -i eth0 -nn`                                        | 禁用域名解析，快                    |
| **看数据内容**   | `tcpdump -i eth0 -nn -A -s 0`                                | `-A`看文本, `-s 0`防截断            |
| **抓特定IP交互** | `tcpdump host 1.1.1.1 and port 80`                           | 精准定位                            |
| **抓拒绝连接**   | `tcpdump "tcp[tcpflags] & tcp-rst != 0"`                     | 只看 RST 包                         |
| **容器抓包**     | `nsenter -n -t <PID> tcpdump`                                | 穿透命名空间                        |
| **长时间监控**   | `tcpdump -C 50 -W 10 -w /tmp/run.pcap`                       | 轮询保存文件                        |
| **排除 SSH**     | `tcpdump port not 22`                                        | 防止自己刷屏                        |
| **抓 ICMP 报错** | `tcpdump 'icmp[icmptype] != icmp-echo and icmp[icmptype] != icmp-echoreply'` | 抓取 Destination Unreachable 等错误 |

