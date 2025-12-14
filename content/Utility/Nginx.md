---
title: nginx
description: 
date: 2025-12-14
categories:
    - 
    - 
---
### Nginx 完整部署与配置 

#### 1. 环境准备与安装

基于 CentOS 7 环境

```
# 1. 安装 EPEL 源
sudo yum install epel-release -y

# 2. 安装 Nginx
sudo yum install nginx -y

# 3. 启动并设置开机自启
sudo systemctl start nginx
sudo systemctl enable nginx

# 4. 配置防火墙 (开放 80/443)
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

**

------

#### 2. 全局主配置文件 (`nginx.conf`)

建议保留默认结构，仅在 `http` 块中添加全局优化配置。

**编辑文件**：`sudo vi /etc/nginx/nginx.conf`

Nginx

```
user nginx;
worker_processes auto; # 自动根据 CPU 核心数调整
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 日志格式
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    # --- 安全与优化配置 (源自文件2的修正) ---
    
    # 隐藏 Nginx 版本号，防止扫描
    server_tokens off; 

    # 开启目录索引 (可选，全局开启)
    # autoindex on; 

    # 限制并发连接数 (定义内存区域)
    # 限制每个 IP 的并发连接数为 10m 大小的空间
    limit_conn_zone $binary_remote_addr zone=addr_limit:10m;
    
    # 限制请求速率
    limit_req_zone $binary_remote_addr zone=req_limit:10m rate=1r/s;

    include /etc/nginx/conf.d/*.conf;
}
```

------

#### 3. 站点详细配置 (虚拟主机)

我们将所有高级功能（动静分离、SSL、认证、限速）集成到一个完整的虚拟主机配置文件中。

**创建文件**：`sudo vi /etc/nginx/conf.d/web_full.conf`

Nginx

```
server {
    listen 80;
    server_name example.com www.example.com;

    # 网站根目录
    root /var/www/example.com;
    index index.html index.htm index.php;

    # --- 1. 动静分离配置 ---
    
    # 静态资源：图片、CSS、JS 本地处理并设置缓存
    location ~* \.(png|jpg|jpeg|gif|bmp|ico|css|js)$ {
        root /var/www/example.com/images; # 指定单独存放图片的路径
        expires 1h;                       # 设置缓存时间为1小时 (修正了原文档的 lh)
        access_log off;                   # 不记录静态资源日志以减少 I/O
    }

    # 动态资源：PHP 转发给后端 (例如 LAMP 服务器 192.168.200.50)
    location ~ \.php$ {
        proxy_pass http://192.168.200.50;
        # 必须传递真实的 Host 和 IP，否则后端无法获取客户端信息
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # --- 2. 目录访问控制与别名 ---

    # 别名测试：访问 /abc 实际指向 /var/www/html
    location /abc {
        alias /var/www/html/; # 注意：alias 末尾通常需要加 /
        index index.html;
    }

    # 禁止访问特定目录 /a/
    location ^~ /a/ {
        deny all;
    }

    # --- 3. 密码访问控制 (Auth Basic) ---
    # 需要先生成密码文件：htpasswd -c /etc/nginx/.htpasswd admin
    location /status {
        stub_status on;
        auth_basic "Restricted Area";
        auth_basic_user_file /etc/nginx/.htpasswd;
    }

    # --- 4. 限速与下载限制 ---
    location /downloads/ {
        alias /data/apps/;
        limit_conn addr_limit 1;  # 每个IP限制1个连接
        limit_rate 512k;          # 限制下载速度为 512KB/s
    }

    # --- 5. 自定义错误页面 ---
    # 将 404 错误指向自定义页面
    error_page 404 /404.html;
    location = /404.html {
        root /usr/share/nginx/html;
    }
}
```

------

#### 4. SSL/HTTPS 配置 (可选)

如果需要开启 HTTPS，请在 `conf.d` 下添加如下配置（基于文件 2 的 SSL 部分优化）：

Nginx

```
server {
    listen 443 ssl;
    server_name example.com;

    # 证书路径 (需先申请或生成证书)
    ssl_certificate     /etc/nginx/ssl/server.crt; 
    ssl_certificate_key /etc/nginx/ssl/server.key;

    # SSL 优化配置
    ssl_session_cache shared:SSL:1m;
    ssl_session_timeout  5m;
    ssl_ciphers HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers on;

    location / {
        root /var/www/example.com;
        index index.html;
    }
}
```

------

### 常用维护命令与操作

1. **生成密码文件 (用于访问控制)**

   ```
   sudo yum install httpd-tools -y
   # 创建用户 linux，密码 666666
   sudo htpasswd -c /etc/nginx/.htpasswd linux
   ```

2. **生成自签名 SSL 证书 (测试用)**

   ```
   sudo mkdir -p /etc/nginx/ssl
   cd /etc/nginx/ssl
   # 生成私钥
   sudo openssl genrsa -out server.key 2048
   # 生成证书请求 (CSR)
   sudo openssl req -new -key server.key -out server.csr
   # 生成自签名证书 (CRT)
   sudo openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt
   ```

3. 日志切割 (推荐方式)

   原文档使用 Shell 脚本切割日志。在 CentOS/RedHat 系统中，推荐使用系统自带的 logrotate，Nginx 安装时通常会自动生成配置文件 /etc/logrotate.d/nginx，无需手动编写脚本。如果必须手动切割，原脚本逻辑可行，但请确保脚本有执行权限。

------

### 注意事项 (Precautions)

1. 配置语法检查 (最重要的步骤)

   每次修改配置文件后，必须运行以下命令检查语法，否则重启可能会导致网站宕机：

   ```
   sudo nginx -t
   ```

   只有看到 `syntax is ok` 和 `test is successful` 才能重启服务。

2. SELinux 权限问题

   如果您更改了默认的网站根目录（例如改到了 /data/web 或 /var/www/example.com），SELinux 可能会阻止 Nginx 读取文件，导致 403 Forbidden 错误。

   - *临时解决*：`setenforce 0`
   - *永久解决*：`chcon -Rt httpd_sys_content_t /您的自定义目录`

3. 文件权限

   确保 Nginx 的运行用户（通常是 nginx）对网站目录有读取权限，对上传目录有写入权限。

   ```
   sudo chown -R nginx:nginx /var/www/example.com
   sudo chmod -R 755 /var/www/example.com
   ```

4. **Alias 与 Root 的区别**

   - `root`：会将 location 后面的路径拼接到 root 路径之后。
   - `alias`：会将 location 后面的路径**替换**为 alias 路径。
   - **注意**：使用 `alias` 时，目录路径末尾通常需要加 `/`，否则容易出错。

5. 安全隐患

   原文档中提到将 404 错误状态码强制改为 200 (error_page 404 =200 /404.jpg)。

   - **警告**：这在 SEO（搜索引擎优化）中是极其糟糕的做法（称为“软 404”），会让搜索引擎误以为错误的页面是正常的。除非是为了特殊的安全伪装，否则**不要**在生产环境中使用 `=200`。

通过以上步骤，您可以搭建一个功能完善、安全且易于维护的 Nginx 服务器。