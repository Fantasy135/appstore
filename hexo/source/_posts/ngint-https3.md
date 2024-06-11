---
title: CentOS 8 源码编译安装 Nginx 支持http3 quic
date: 2023-07-21 16:23:20
cover: https://pub-ea3158201e944f5085d78b75bbac59de.r2.dev/1704791649763.jpg?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20240516T043826Z&X-Amz-SignedHeaders=host&X-Amz-Expires=1800&X-Amz-Credential=057b3fe7414eab195d93341f7b4245db%2F20240516%2Fauto%2Fs3%2Faws4_request&X-Amz-Signature=5bdc832f8b771e9e983f084fe6d9b299ab2f36d4d9900cb454afb2314d1f06f8
tags:
  - 原创
  - 技术
---

这里安装的是1.25.1
## 下载源码

下载
``` bash
wget http://nginx.org/download/nginx-1.25.1.tar.gz
```
解压
``` bash
tar zxvf nginx-1.25.1.tar.gz
```

## 安装编译环境
``` bash
yum update
yum -y install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel
```

## 进入nginx源码目录
``` bash
cd nginx-1.25.1
```

## 编译和安装

配置
``` bash
 ./configure \
    --prefix=/usr/local\nginx \
    --pid-path=/var/run/nginx/nginx.pid \
    --lock-path=/var/lock/nginx.lock \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --with-http_gzip_static_module \
    --http-client-body-temp-path=/var/temp/nginx/client \
    --http-proxy-temp-path=/var/temp/nginx/proxy \
    --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
    --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
    --http-scgi-temp-path=/var/temp/nginx/scgi \
    --with-http_stub_status_module \
    --with-http_ssl_module \
    --with-http_v2_module \
    --with-http_v3_module
```

编译安装
``` bash
make && make install
```

## 验证
``` bash
/usr/local/nginx/sbin/nginx -v
```
输出如下
```angular2html
nginx version: nginx/1.25.1
```

## 创建软连接
``` bash
ln -s /usr/local/nginx /usr/bin/nginx
```

## 开机自启
``` bash
systemctl enable nginx.server
```

## nginx配置文件
打开配置文件
``` bash
vim /usr/local/nginx/nginx.conf
```

开启gzip压缩
在http 里添加
```angular2html
    gzip  on;
    gzip_comp_level 5;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_proxied any;
    gzip_vary on;
    gzip_types
      application/javascript
      application/x-javascript
      text/javascript
      text/css
      text/xml
      application/xhtml+xml
      application/xml
      application/atom+xml
      application/rdf+xml
      application/rss+xml
      application/geo+json
      application/json
      application/ld+json
      application/manifest+json
      application/x-web-app-manifest+json
      image/svg+xml
      text/x-cross-domain-policy;
    gzip_static on;
    gzip_disable "MSIE [1-6]\.";
```

网站使用http2 http3
``` dict
server {
        listen       80; #监听80端口
        listen       443  default  quic reuseport; #开启http3 quic
        listen       443  default  ssl; 开启ssl

        http2  on;  #开启http2支持

        server_name  fantasy.kim  www.fantasy.kim;  #监听地址

        ssl_certificate  /home/ssl/fantasy.crt; #ssl签名证书
        ssl_certificate_key  /home/ssl/fantasy.key; #ssl签名密钥

        ssl_session_cache  shared:SSL:10M;  #ssl缓存大小
        ssl_session_timeout  10M;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        ssl_buffer_size 4k;
        ssl_ciphers 'ECDHE+AESGCM:DHE+AESGCM:HIGH:!aNULL:!MD5;';
        ssl_prefer_server_ciphers on;

        access_log  /home/hexo/logs/access.log  main;  #log 文件地址
        error_log  /home/hexo/logs/error.log  error;   #error log 文件地址

        location / {
            if ($http_x_forwarded_proto != "https") {
                rewrite ^/(.*)$ https://fantasy.kim/$1 permanent;
            }  #http重定向到https
            gzip_static on; #使用gzip推送
            root  /home/hexo/public;  #项目根目录
            index  index.html;
            error_page  404              404.html;
        }
}
```
## 检查配置文件有没有问题
``` bash
nginx -t
```

## 开启nginx
``` bash
nginx
```
