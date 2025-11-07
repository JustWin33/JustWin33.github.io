---
title: Linux科学上网教程
description: 支持Clash和V2ray
categories:
    - Test
    - 教程
---


### 1、下载SSH工具

FinalShell：(https://www.hostbuf.com/t/988.html)

### 2、安装环境：
{{< highlight html >}}
sudo apt update
sudo apt install bzip2 tar
sudo apt-get install curl
{{< /highlight >}}
### 3、安装ShellCrash:

sudo -i #切换到root用户
{{< highlight html >}}
export url='https://fastly.jsdelivr.net/gh/juewuy/ShellCrash@master' && wget -q --no-check-certificate -O /tmp/install.sh $url/install.sh  && bash /tmp/install.sh && source /etc/profile &> /dev/null
{{< /highlight >}}



备用安装源：
{{< highlight html >}}
export url='https://gh.jwsc.eu.org/master' && bash -c "$(curl -kfsSl $url/install.sh)" && source /etc/profile &> /dev/null
{{< /highlight >}}


放行端口

`iptables -I INPUT -p tcp --dport 9999 -j ACCEPT`

Clash管理地址（将IP替换成自己服务器IP）：http://192.168.1.1:9999/ui

#### 测试打开goole

`curl google.com`

测速

`sudo apt install speedtest-cli`

运行：speedtest

### 常见问题

在连接SSH时提示端口22:连接被拒绝：ssh: connect to host 192.168.1.1 port 22: Connection refused
请安装SSH服务，使用以下命令安装：
Debian/Ubuntu：sudo apt-get install openssh-server
CentOS/RHEL：sudo yum install openssh-server

本教程使用的命令，使用的是Ubtuntu/Debian系统，如果你使用的是CentOS或者其它，就切换对应的命令。
另外 ShellCrash 安装源是通用的。



