---
title: docker源
description: 
date: 2025-12-04
categories:
    - 
    - 
---
安装部署

```bash
#删除清理

sudo yum remove docker \
docker-client \
docker-client-latest \
docker-latest \
docker-latest-logrotate \
docker-logrotate \
docker-engine



#安装依赖工具
sudo yum install -y yum-utils

添加阿里云官方仓库
sudo yum-config-manager \
--add-repo \
https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum install -y docker-ce docker-ce-cli containerd.io

systemctl start docker
systemctl enable docker

vim /etc/docker/daemon.json
```



```bash
{
   "registry-mirrors": [
       "https://docker.m.daocloud.io",
       "https://dockerproxy.com",
       "https://docker.mirrors.ustc.edu.cn",
       "https://docker.nju.edu.cn",
       "https://iju9kaj2.mirror.maliyuncs.com",
       "http://hub-mirror.c.163.com",
       "https://cr.console.aliyun.com",
       "https://hub.docker.com",
       "http://mirrors.ustc.edu.cn"
   ]
}
```



```basg
systemctl daemon-reload
systemctl restart docker
```


或者

```
更新Linux系统软件源
bash <(curl -sSL https://linuxmirrors.cn/main.sh)

docker安装与换源
bash <(curl -sSL https://linuxmirrors.cn/docker.sh)

docker更换镜像加速器
bash <(curl -sSL https://linuxmirrors.cn/docker.sh) --only-registry

```


































