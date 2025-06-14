---
title: v2ray配置教程
author: Z.C Yang
date: 2025-06-13 11:00:00 -0500
categories: [服务器]
tags: [v2ray]
---

---
title: A Quick Tutorial on Use-package for Emacs
author: Ian Y.E. Pan
date: 2021-05-12 21:18:00 +0800
categories: [Emacs]
tags: [linux, emacs, tutorial, tips, workflow, programming]
---

# 1. 服务器环境准备


## 1.1 购买服务器
购买服务器，推荐 [零零壹云](https://www.001yun.net)，对比阿里云等厂商境外服务器购买门槛低、价格也很实惠。选择配置过程中，建议选用cenos7系统。

## 1.2 shell远程连接服务器
推荐finalshell，功能强大，大部分功能免费。

## 1.3 域名准备
实现流量伪装过程中，需要域名，推荐购买阿里云域名。



# 2. 服务端安装和配置
## 2.1 linux服务上安装v2ray程序
执行如下命令安装v2Ray:

```shell
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```



## 2.2 配置v2ray
在**/usr/local/etc/v2ray/**目录下对config文件进行编辑：

```shell
# vi命令打开配置文件
vi /usr/local/etc/v2ray/config.json
```



填写如下配置：

```shell
{
  "inbounds": [{
    "port": 监听端口,
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "用户id，生成方法见下面说明"
        }
      ]
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  }]
}
```

其中，port字段选择`1024-65535`中的某个数字，clients中的`id`可以执行下述命令得到：

```shell
/usr/local/bin/v2ray uuid
```



设置完端口和uuid的示例如下：

```shell
{
  "inbounds": [{
    "port": 33688,
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "e7e91bbb-0e6c-3fab-6b82-5f441e953169"
        }
      ]
    }
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  }]
}
```



**上述配置中的两个参数在客户端也会用到，需要保持一致！**



## 2.3 防火墙设置

cenos6/7推荐使用iptables进行端口配置，例如在上述port配置为33688时，开放端口配置如下;

```shell
iptables -I INPUT -p tcp --dport 33688 -j ACCEPT
```



## 2.4 自启动配置

下面一些命令用来配置v2ray启动服务：

```shell
# 设置开启启动
systemctl enable v2ray

# 运行v2ray
systemctl start v2ray

# 重启v2ray
systemctl restart v2ray

# 查看v2ray运行状态
systemctl status v2ray
```



## 2.5 客户端配置
详细pc v2ray客户端配置方法查看[v2ray客户端下载和使用](https://itlanyan.com/v2ray-tutorial/)，关键步骤就是将ip、端口代理数据进行填写。



# 3. 流量伪装

## 3.1 配置dns
可以使用[cloudflare](https://dash.cloudflare.com/)的dns免费解析服务，将域名和服务器ip进行映射。


## 3.2 安装nginx

cenos服务器安装nginx:

```shell
yum install -y epel-release && yum install -y nginx
```



打开`/etc/nginx/nginx.conf`配置文件调整：

```shell
vi /etc/nginx/nginx.conf
```

```shell
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        server_name  us.testip.cn;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
}
}
```



配置完成后，进行配置检查，无误后设置nginx为服务，启动nginx服务：

```shell
# 配置检查
nginx -t 

# 设置nginx为服务
systemctl enable nginx

# 启动nginx服务
systemctl start nginx

# nginx服务重启
systemctl restart nginx
```



针对一些异常状态，可以使用如下命令排查：

```shell
# 查看nginx状态
systemctl status nginx

# 异常状态下查看端口占用
yum install lsof -y && lsof -i :80

# 杀掉进程
pkill nginx
```



开发端口：

```shell
iptables -I INPUT -p tcp --dport 80 -j ACCEPT
```



浏览器访问个人域名（例如 [http://us.testip.cn/](http://us.testip.cn/)）域名进行测试，可以看到nginx欢迎页。



## 3.3 ssl证书申请

安装acme.sh脚本程序，用于后续签发证书：

```shell
curl https://get.acme.sh | sh -s email=my@example.com
```

注意上述email修改为个人邮箱，用于接收过期信息。



签发证书方式有多种，这里使用运行nginx的方式进行验证：

```shell
~/.acme.sh/acme.sh --issue -d 域名 --nginx
```

针对`us.testip.cn`域名，命令为：

```shell
~/.acme.sh/acme.sh --issue -d us.testip.cn --nginx
```

更多签发方式参考[https://itlanyan.com/use-acme-sh-get-free-cert/](https://itlanyan.com/use-acme-sh-get-free-cert/)



<font style="color:rgb(51, 51, 51);">申请好证书的证书位于</font><font style="color:#DF2A3F;">~/.acme.sh</font><font style="color:rgb(51, 51, 51);">目录内，不建议直接使用，而是将其安装到指定目录：</font>

```shell
~/.acme.sh/acme.sh --install-cert -d 域名 \
--key-file       密钥存放路径（例如/etc/nginx/ssl/域名.key）  \
--fullchain-file 证书存放路径（例如/etc/nginx/ssl/域名.pem） \
--reloadcmd     "service nginx force-reload"
```

针对`us.testip.cn`域名，命令为：

```shell
# nginx创建ssl证书存放目录
mkdir /etc/nginx/ssl

# 复制证书到目录
~/.acme.sh/acme.sh --install-cert -d us.testip.cn \
--key-file       /etc/nginx/ssl/us.testip.cn.key  \
--fullchain-file /etc/nginx/ssl/us.testip.cn.pem  \
--reloadcmd     "service nginx force-reload"
```



## 3.4 nginx ssl证书配置

修改`/etc/nginx/nginx.conf`配置文件，加入ssl证书相关配置:

```shell
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        server_name  us.testip.cn;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }
}

# Settings for a TLS enabled server.
server {
   listen       8443 ssl http2;
   server_name  us.testip.cn;
   root         /usr/share/nginx/html;

   # ssl证书关键配置
   ssl_protocols TLSv1.2 TLSv1.3;
   ssl_certificate "/etc/nginx/ssl/us.testip.cn.pem";
   ssl_certificate_key "/etc/nginx/ssl/us.testip.cn.key";
   ssl_session_cache shared:SSL:10m;
   ssl_session_timeout  1d;
   ssl_ciphers HIGH:!aNULL:!MD5;
   ssl_prefer_server_ciphers off;

   # Load configuration files for the default server block.
   include /etc/nginx/default.d/*.conf;

   error_page 404 /404.html;
       location = /40x.html {
   }

   error_page 500 502 503 504 /50x.html;
       location = /50x.html {
   }
}
}
```

由于`443`端口容易被墙，推荐使用8000~9000之间的端口，这里使用`8443`。



同时我们应该开放`8443`端口：

```shell
iptables -I INPUT -p tcp --dport 8443 -j ACCEPT
```



浏览器访问个人域名（例如 [https://us.testip.cn:8443](http://us.testip.cn:8443) ）进行测试，可以看到nginx欢迎页。



## 3.5 nginx与v2ray结合
配置nginx，将伪装路径的请求都转发到v2ray，编辑`/etc/nginx/nginx.conf`配置文件，在第二个server配置中加入路径转发配置：

```shell
# vim 编辑器打开文件
vi /etc/nginx/nginx.conf
```

```shell
# nginx中新增的转发配置
   location /science { # 与 V2Ray 配置中的 path 保持一致
      proxy_redirect off;
      proxy_pass http://127.0.0.1:33688; # 假设v2ray的监听地址是33688
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   }
```



完整配置为：

```shell
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/doc/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 4096;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80;
        server_name  us.testip.cn;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        error_page 404 /404.html;
        location = /404.html {
        }

        error_page 500 502 503 504 /50x.html;
        location = /50x.html {
        }

        location / {
	   proxy_pass http://127.0.0.1:8002;
	   }

	   location /resource {
	   proxy_pass http://127.0.0.1:8002/resource;
	   }

        # rewrite ^(.*) https://$server_name$1 permanent;
    }

# Settings for a TLS enabled server.
server {
   listen       8433 ssl http2;
   server_name  us.testip.cn;
   root         /usr/share/nginx/html;

   ssl_protocols TLSv1.2 TLSv1.3;
   ssl_certificate "/etc/nginx/ssl/us.testip.cn.pem";
   ssl_certificate_key "/etc/nginx/ssl/us.testip.cn.key";
   ssl_session_cache shared:SSL:10m;
   ssl_session_timeout  1d;
   ssl_ciphers HIGH:!aNULL:!MD5;
   ssl_prefer_server_ciphers off;

   # Load configuration files for the default server block.
   include /etc/nginx/default.d/*.conf;

   error_page 404 /404.html;
       location = /40x.html {
   }

   error_page 500 502 503 504 /50x.html;
       location = /50x.html {
   }


   location /science { # 与 V2Ray 配置中的 path 保持一致
      proxy_redirect off;
      proxy_pass http://127.0.0.1:33688; # 假设v2ray的监听地址是33688
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
      proxy_set_header Host $host;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
   }
}
}
```



配置后重启nginx，使转发配置生效：

```shell
systemctl restart nginx
```



接着编辑`/usr/local/etc/v2ray/config.json`文件，配置v2ray监听地址和传输协议：

```shell
# vim 编辑器打开config.js
vi /usr/local/etc/v2ray/config.json
```

```shell
{
  "inbounds": [{
    "port": 33688,
    "protocol": "vmess",
    "settings": {
      "clients": [
        {
          "id": "e7e91bbb-0e6c-3fab-6b82-5f441e953169"  # 参照上文生成uuid
        }
      ]
    },
    "streamSettings": {
        "network": "ws",      # 网络传输协议设置为websocket
        "wsSettings": {      
          "path": "/science"  # 和上述nginx中路径一致
        }
      },
    "listen": "127.0.0.1"  # 安全考虑，只接受本地链接
  }],
  "outbounds": [{
    "protocol": "freedom",
    "settings": {}
  }]
}
```



重启v2ray服务:

```shell
systemctl restart v2ray
```


