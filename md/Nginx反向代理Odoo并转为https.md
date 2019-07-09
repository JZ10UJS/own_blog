---
title: Nginx反向代理Odoo并转为https
date: 2016-07-19 20:10:49
tags:
---

### 生成证书文件
生成自签名证书，并放在指定位置
```bash
$ openssl req -x509 -days 3650 -subj '/CN=odoo.youyun.com/' -nodes -newkey rsa:2048 -keyout server.key -out server.crt
$ sudo mkdir /etc/ssl/nginx
$ sudo mv server.key server.crt /etc/ssl/nginx
```
### 配置nginx

```
$ sudo rm /etc/nginx/sites-avaliable/default
$ sudo vim /etc/nginx/sites-avalibale/odoo.conf
```
删除默认的nginx default文件，并新建odoo.conf文件，内容如下

```
server {
	listen 443 default;
	server_name _;
	
	access_log /var/log/nginx/odoo.access.log;
	error_log  /var/log/nginx/odoo.error.log;

	ssl on;
	ssl_certificate     /etc/ssl/nginx/server.crt; # 之前生成的证书和key
	ssl_certificate_key /etc/ssl/nginx/server.key;
	ssl_ciphers             HIGH:!ADH:!MD5;
    ssl_protocols           SSLv3 TLSv1;
    ssl_prefer_server_ciphers on;
	
	location / {
        proxy_pass http://127.0.0.1:8069;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;

        proxy_buffer_size 128k;
        proxy_buffers 16 64k;
        proxy_redirect off;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto https;
    }

   location ~* /web/static/ {
        proxy_cache_valid 200 60m;
	    proxy_buffering    on;
	    expires 864000;
        proxy_pass http://127.0.0.1:8069;
    }
}

server {   # 将80端口转到443的https中
    listen 80;
    server_name __;

    add_header Strict-Transport-Security max-age=2592000;
    rewrite ^/.*$ https://$host$request_uri? permanent;
}

server {   # 将特定ip的8069端口转到443的https中
    listen 192.168.1.102:8069; # 这是虚机的ip
    server_name __;

    add_header Strict-Transport-Security max-age=2592000;
    rewrite ^/.*$ https://$host$request_uri? permanent;
}
```
### 配置访问源主机

 - 安装之前生成的server.crt证书
 - 修改hosts文件添加， 由于之前的证书使用的是该域名
 ```
192.168.1.102 odoo.youyun.com
```
![这里写图片描述](http://img.blog.csdn.net/20160804165239381)