---
title: Linuxclash
description: 
date: 2025-12-18
categories:
    - 
    - 
---
shellclash linux使用教程，Debian/Ubuntu安装clash linux server vpn

1、SSH工具下载
FinalShell
2、安装环境：
sudo apt update
sudo apt install bzip2 tar
sudo apt-get install curl
3、安装ShellCrash:
sudo -i    #切换为root用户



export url='https://fastly.jsdelivr.net/gh/juewuy/ShellCrash@master' && wget -q --no-check-certificate -O /tmp/install.sh $url/install.sh  && bash /tmp/install.sh && source /etc/profile &> /dev/null


备用安装源：

export url='https://gh.jwsc.eu.org/master' && bash -c "$(curl -kfsSl $url/install.sh)" && source /etc/profile &> /dev/null

1公测版
1 /etc/
1
crash
2
1
clash
7890

自己本地LinuxIP

clash
1
1
1
https://win1688888888.dpdns.org/03ea7121-577a-4c38-9f90-abf624de7e47/sub
1
1
1
9
4      安装面板
4     基础面板
0
1
复制地址，如果这个管理地址正常时能直接打开的如果不是你的Linux的IP，就需要手动替换，再在浏览器打开

0





放行端口

iptables -I INPUT -p tcp --dport 9999 -j ACCEPT

密钥：clash

Clash管理地址（将IP替换成自己服务器IP）：http://192.168.1.1:9999/ui

测速打开google
curl google.com

测速
sudo apt install speedtest-cli
运行：speedtest


添加/更新
确保是root用户
crash
6
1
1
粘贴你的最新V2ray连接
1





常见问题
在连接SSH时提示端口22:连接被拒绝：ssh:connetct to host 192.168.1.1 port 22:Connection refused

请安装SSH服务，使用以下命令安装：
Debian/Ubuntu: sudo apt-get install openssh-server
CentOS/RHEL: sudo yum install openssh-server

本教程使用的命令，使用的时Ubuntu/Debian系统，如果你使用的是CentOS或者其他，就切换对应的命令，别外ShellCrash安装源也是通用的。







https://url.v1.mk/sub?target=clash&url=https%3A%2F%2Fwin1688888888.dpdns.org%2F03ea7121-577a-4c38-9f90-abf624de7e47%2Fsub&insert=false&config=https%3A%2F%2Fraw.githubusercontent.com%2FbyJoey%2Ftest%2Frefs%2Fheads%2Fmain%2Ftist.ini&emoji=true&list=false&xudp=false&udp=false&tfo=false&expand=true&scv=false&fdn=false&new_name=true