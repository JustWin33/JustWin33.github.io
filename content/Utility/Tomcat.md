---
title: Tomcat
description: 
date: 2025-12-14
categories:
    - 
    - 
---
# Apache Tomcat 部署运维指南 (CentOS 7/Linux 通用版)

## 第一部分：核心知识点 (原理篇)

在动手之前，理解 Tomcat 的架构能让你在报错时迅速定位问题。

### 1\. Tomcat 是什么？

Tomcat 是 Apache 基金会开源的 **Servlet 容器**，也是 Java Web 应用（如 Spring Boot）的默认服务器。

  * **核心作用**：它充当“翻译官”，监听 HTTP 端口（如 8080），将收到的网络请求转换为 Java 对象（Request），交给你的 Java 代码处理，再将结果（Response）返回给浏览器。

### 2\. 关键目录结构 (部署必背)

部署完成后，你必须熟悉 `/opt/tomcat` 下的这几个目录：

| 目录         | 重要级 | 作用                                                         |
| :----------- | :----- | :----------------------------------------------------------- |
| **bin/**     | ⭐⭐⭐    | 存放脚本。`startup.sh` (启动), `shutdown.sh` (停止)。        |
| **conf/**    | ⭐⭐⭐⭐⭐  | **配置中心**。`server.xml` (端口/连接), `web.xml` (全局设置)。 |
| **webapps/** | ⭐⭐⭐⭐   | **代码存放处**。默认将 `.war` 包扔在这里会自动解压部署。     |
| **logs/**    | ⭐⭐⭐⭐⭐  | **排错第一现场**。`catalina.out` 是最重要的控制台日志。      |
| **work/**    | ⭐      | JSP 编译后的临时文件，清理缓存时可删除。                     |

-----

## 第二部分：生产环境部署实战 (操作篇)

> **环境假设**：CentOS 7 / RedHat 系列 Linux
> **用户权限**：需拥有 `sudo` 权限

### 步骤一：安装 Java 环境 (JDK)

Tomcat 是纯 Java 编写的，必须依赖 JDK 运行。

1.  **检查是否已安装**

    ```bash
    java -version
    # 如果显示 "command not found" 或版本过低，继续下一步
    ```

2.  **安装 OpenJDK 1.8 (生产环境最常用版本)**

    ```bash
    sudo yum install -y java-1.8.0-openjdk-devel
    ```

3.  **❗关键步骤：获取 Java 真实安装路径**
    Systemd 服务配置需要绝对路径，运行以下命令并记下输出结果：

    ```bash
    # 查找 java 命令的位置，并找出其软链接背后的真实路径
    readlink -f $(which java) | sed "s:bin/java::"
    ```

    > *预期输出示例（记下来，后面配置要用）：* `/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.382.b05-1.el7_9.x86_64/jre/`

### 步骤二：下载与规范化部署 Tomcat

我们将 Tomcat 部署在 `/opt` 目录，并使用专用用户运行，这是企业安全标准。

1.  **下载 Tomcat 9 (稳定版)**

    ```bash
    cd /tmp
    # 下载 9.0.x 最新版（建议去官网核对版本号）
    wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.80/bin/apache-tomcat-9.0.80.tar.gz
    ```

2.  **解压并规范目录**

    ```bash
    # 创建标准目录
    sudo mkdir -p /opt/tomcat

    # 解压并剥离首层文件夹（直接把内容解压到 /opt/tomcat 下，而不是 /opt/tomcat/apache-tomcat-x.x.x）
    sudo tar -zxvf apache-tomcat-9.0.80.tar.gz -C /opt/tomcat --strip-components=1
    ```

3.  **创建专用用户 (安全加固)**
    禁止 root 用户直接运行 Tomcat，防止黑客提权。

    ```bash
    # 创建一个没有登录权限(/bin/false)的用户组和用户
    sudo groupadd tomcat
    sudo useradd -s /bin/false -g tomcat -d /opt/tomcat tomcat

    # 移交文件所有权给 tomcat 用户
    sudo chown -R tomcat:tomcat /opt/tomcat/

    # 赋予脚本执行权限
    sudo chmod +x /opt/tomcat/bin/*.sh
    ```

### 步骤三：配置 Systemd 系统服务 (核心)

配置后，Tomcat 可以开机自启，崩溃自动重启。

1.  **创建服务文件**

    ```bash
    sudo vi /etc/systemd/system/tomcat.service
    ```

2.  **写入以下内容 (无误版)**
    *注意：将 `Environment="JAVA_HOME=..."` 修改为你步骤一里获取的真实路径！*

    ```ini
    [Unit]
    Description=Apache Tomcat 9 Web Application Container
    After=network.target syslog.target

    [Service]
    Type=forking

    # 指定运行用户
    User=tomcat
    Group=tomcat

    # ❗修改这里：填写你步骤一里查到的真实 Java 路径
    Environment="JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk" 
    Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
    Environment="CATALINA_HOME=/opt/tomcat"
    Environment="CATALINA_BASE=/opt/tomcat"

    # 内存优化参数 (可选，防止内存溢出)
    Environment='CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC'

    ExecStart=/opt/tomcat/bin/startup.sh
    ExecStop=/opt/tomcat/bin/shutdown.sh

    # 进程崩溃后自动重启
    Restart=on-failure

    [Install]
    WantedBy=multi-user.target
    ```

3.  **启动服务**

    ```bash
    # 重新加载服务配置
    sudo systemctl daemon-reload

    # 启动 Tomcat
    sudo systemctl start tomcat

    # 设置开机自启
    sudo systemctl enable tomcat
    ```

4.  **验证状态**

    ```bash
    sudo systemctl status tomcat
    ```

    *看到绿色的 `active (running)` 即表示成功。*

### 步骤四：防火墙设置

1.  **开放 8080 端口**

    ```bash
    sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
    sudo firewall-cmd --reload
    ```

    *(如果是阿里云/腾讯云服务器，还需要在控制台的“安全组”中放行 8080 端口)*

2.  **测试访问**
    浏览器访问：`http://服务器IP:8080`
    *出现 Apache Tomcat 的 logo 页面即部署完成。*

-----

## 第三部分：高级配置 (管理与优化)

### 1\. 配置 Tomcat 管理员 (Manager App)

默认情况下，Tomcat 的后台管理页面是被禁用的。

1.  **编辑用户配置**

    ```bash
    sudo vi /opt/tomcat/conf/tomcat-users.xml
    ```

    在 `<tomcat-users>` 标签内添加：

    ```xml
    <role rolename="manager-gui"/>
    <role rolename="admin-gui"/>
    <user username="admin" password="StrongPassword123" roles="manager-gui,admin-gui"/>
    ```

2.  **解除本地访问限制 (允许远程登录后台)**
    默认 Manager App 只允许 `127.0.0.1` 访问。

    ```bash
    # 修改 manager 应用的 context.xml
    sudo vi /opt/tomcat/webapps/manager/META-INF/context.xml
    ```

    **操作**：注释掉 `<Valve>` 标签（或者将你的 IP 加入 `allow` 列表）：

    ```xml
    
```
    
*对 `Host Manager` (/opt/tomcat/webapps/host-manager/META-INF/context.xml) 也做同样操作。*
    
3.  **重启生效**

    ```bash
    sudo systemctl restart tomcat
    ```

### 2\. 修改端口 (如改为 80)

如果你想直接输入 IP 访问而不加端口：

1.  编辑 `sudo vi /opt/tomcat/conf/server.xml`
2.  找到 `<Connector port="8080" ...>`
3.  将 `8080` 改为 `80`。
    *注意：Linux 下使用 1024 以下端口通常需要 root 权限，systemd 方式启动通常没问题，但建议保持 8080 或使用 Nginx 反向代理。*

-----

## 第四部分：常用运维命令与排错

这是你需要贴在工位上的**速查表**。

### 1\. 常用命令

| 动作             | 命令                            |
| :--------------- | :------------------------------ |
| **启动**         | `sudo systemctl start tomcat`   |
| **停止**         | `sudo systemctl stop tomcat`    |
| **重启**         | `sudo systemctl restart tomcat` |
| **查看状态**     | `sudo systemctl status tomcat`  |
| **查看进程**     | `ps -ef | grep tomcat`          |
| **查看端口占用** | `netstat -tlnp | grep 8080`     |

### 2\. 故障排查 (Log Analysis)

**场景：** 启动了但网页打不开，或者显示 500 错误。

**必杀技：实时查看日志**

```bash
# 实时滚动显示最新的日志，排错神器
tail -f /opt/tomcat/logs/catalina.out
```

  * **看到 `Caused by: ...`**：通常是你的 Java 代码报错了。
  * **看到 `Address already in use`**：端口被占用了，检查是否有另一个 Tomcat 在运行。
  * **看到 `OutOfMemoryError`**：内存爆了，需要调整 Systemd 文件中的 `CATALINA_OPTS` 参数。





 **Nginx + Tomcat 反向代理实战指南**。

这是生产环境中最经典、最标准的架构：**前端 Nginx (80端口)** 负责抗压和转发，**后端 Tomcat (8080端口)** 专注处理 Java 业务。

-----

# Nginx + Tomcat 反向代理配置指南 (CentOS 7 版)

### 核心架构图解

  * **用户** -\> 访问 `http://服务器IP` (默认80端口)
  * **Nginx** -\> 接收请求，通过**反向代理**转发给本地的 `127.0.0.1:8080`
  * **Tomcat** -\> 处理 Java 业务，将结果返回给 Nginx
  * **Nginx** -\> 将结果返回给用户

这样做的好处：**安全**（隐藏了 Tomcat 真实端口）、**快**（Nginx 处理静态资源速度极快）、**灵活**（以后可以轻松扩展成多台 Tomcat 负载均衡）。

-----

### 第一步：安装 Nginx

如果你的服务器上还没安装 Nginx，请执行以下命令：

1.  **添加 EPEL 源 (Nginx 在 EPEL 源中)**

    ```bash
    sudo yum install -y epel-release
    ```

2.  **安装 Nginx**

    ```bash
    sudo yum install -y nginx
    ```

3.  **启动并设置开机自启**

    ```bash
    sudo systemctl start nginx
    sudo systemctl enable nginx
    ```

    *此时访问服务器 IP，应该能看到 Nginx 的欢迎页面。*

-----

### 第二步：配置反向代理 (核心步骤)

我们需要修改 Nginx 的配置文件，让它把请求转交给 Tomcat。

1.  **新建/编辑配置文件**
    为了保持配置整洁，建议在 `conf.d` 目录下新建一个专门的 `.conf` 文件，而不是直接修改主文件。

    ```bash
    sudo vi /etc/nginx/conf.d/tomcat_proxy.conf
    ```

2.  **写入以下配置 (请直接复制并修改 server\_name)**

    ```nginx
    server {
        listen       80;
        # 将下面的 localhost 修改为你的 域名 或 服务器公网IP
        server_name  localhost;

        # 核心转发配置
        location / {
            # 将请求转发给本地的 Tomcat (8080)
            proxy_pass http://127.0.0.1:8080;

            # --- 必需的请求头设置 (这是重点) ---
            # 1. 传递域名给 Tomcat (否则 Tomcat 可能会重定向错)
            proxy_set_header Host $host;
            # 2. 传递用户的真实 IP (否则 Java 代码里读到的都是 127.0.0.1)
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            # 3. 传递协议 (http/https)
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        # 可选优化：让 Nginx 直接处理静态文件 (图片/css/js)，减轻 Tomcat 压力
        # location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|css|js)$ {
        #    root /opt/tomcat/webapps/ROOT;
        #    expires 30d;
        # }
    }
    ```

3.  **检查配置是否有语法错误**

    ```bash
    sudo nginx -t
    ```

    *必须看到 `syntax is ok` 和 `test is successful` 才能继续。*

4.  **重启 Nginx 使配置生效**

    ```bash
    sudo systemctl restart nginx
    ```

-----

### 第三步：解决 SELinux 权限问题 (CentOS 特有坑)

**这步非常关键！**
如果你是 CentOS 7，默认的 SELinux 安全策略会**禁止** Nginx 主动发起网络连接（即禁止它连接 8080 端口）。如果不做这步，你会看到 Nginx 报错 `502 Bad Gateway`。

1.  **方法一：告诉 SELinux 允许 Nginx 联网 (推荐)**

    ```bash
    # 打开 httpd_can_network_connect 开关
    sudo setsebool -P httpd_can_network_connect 1
    ```

    *(参数 `-P` 表示永久生效，执行可能需要几秒钟，请耐心等待)*

2.  **方法二：临时关闭 SELinux (暴力解法)**
    如果你不想折腾 SELinux，可以临时关闭它测试：

    ```bash
    sudo setenforce 0
    ```

-----

### 第四步：验证全链路

现在，你的架构已经部署完毕。

1.  确保 Tomcat 正在运行：`sudo systemctl status tomcat`

2.  确保 Nginx 正在运行：`sudo systemctl status nginx`

3.  **最终测试**：
    打开浏览器，直接访问 **`http://服务器IP`** (不需要加 :8080)。

      * **预期结果**：你应该能看到 **Tomcat** 的页面（虽然你访问的是 Nginx 的 80 端口）。
      * **如果看到 Nginx 欢迎页**：说明上面的 `server_name` 没匹配上，或者 `nginx.conf` 里有默认的 server 块抢占了 80 端口。建议注释掉 `/etc/nginx/nginx.conf` 里的默认 `server { ... }` 块。

-----

### 进阶场景：Tomcat 负载均衡 (集群)

如果你以后业务做大了，一台 Tomcat 扛不住，部署了三台 (8080, 8081, 8082)，Nginx 配置只需要改一点点即可实现**负载均衡**：

```nginx
# 定义一个上游服务器组
upstream my_java_app {
    server 127.0.0.1:8080 weight=1; # 权重1
    server 127.0.0.1:8081 weight=1; # 权重1
    server 127.0.0.1:8082 weight=2; # 权重2 (性能好的机器多扛点)
}

server {
    listen 80;
    server_name localhost;

    location / {
        # 这里的地址改成上面的 upstream 名字
        proxy_pass http://my_java_app;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

