---
title: Odoo部署
date: 2018-01-26 14:23:08
tags:
---

## dbfilter
Odoo是一个多租户的系统：一个单Odoo系统可以跑多个数据库实例，并且它是高度可定制化的，不同的database可以安装不同的modules。
对于那些需要登录web后台的用户来说，`dbfilter`的配置不存在任何问题，因为他们在登录的时候，需要选择对应的database。
但是对于那些不要登录的用户来说（如：module中的 portal, website），Odoo就需要知道使用哪个database的数据来显示website page。如果没有使用Odoo的多租户系统，那就没问题，因为只存在一个数据库实例。但是如果有多个数据库实例，就需要一些东西来帮助Odoo理解使用哪个数据库了。
这也就是`--db-filter`参数存在的意义了。通过配置这个参数，Odoo可以针对hostname来决定使用哪一个数据库里面的数据。参数的值为一个正则表达式，可以使用`%h` 和 `%d`来代替hostname和domain。
生产环境中，如果有多个数据库实例，尤其是`website`模块安装了的，一定要配置`dbfilter`，否则有很多特性都不能使用。

### 配置样例

- 只显示名字以`'mycompany'` 开头的数据库

在`/etc/odoo.conf`中，
```
[options]
dbfilter = ^mycompany.*$
```
- 只显示名字与`www`之后子域名匹配的数据库，比如在域名为`www.mycompany.com`，`mycompany.co.uk`中，`mycompany`数据库会被显示，而在域名`www2.mycompany.com`和`helpdesk.mycompany.com`中不会显示

在 `/etc/odoo.conf`中：
```
[options]
dbfilter = ^%d$
```
> 提示：
> 配置一个合适的 `--db-filter` 对与系统安全是有一定帮助的。如果你的配置正确，并且这个dbfilter只有一个数据库名字能满足，那么强烈推荐你使用`--no-database-list`参数，来隐藏所有数据库的显示和选择。


## PostgreSQL
默认情况下，PostgreSQL仅接受来自UNIX sockets和本地环路（也就是localhost）的连接。
如果你的Odoo和PostgreSQL泡在同一台服务器上，那么UNIX socket也是可以的，并且如果你没指定pg host的情况下，默认使用的也就是unix socket。如果你的Odoo和PostgreSQL不再同一台机器上，那么就需要配置pg，使其接受网络连接，方法如下：

-  一种是，pg还是只接受本地环路，但是在两台机器上使用 [SSH tunnel](https://www.postgresql.org/docs/9.3/static/ssh-tunnels.html)，然后配置Odoo连接tunnel
- 另一种，pg接受来自odoo机器的[网路连接](https://www.postgresql.org/docs/9.3/static/runtime-config-connection.html)，然后配置odoo

### 配置样例

- 接受localhost上的tcp连接
- 接受192.168.1.x网段的tcp连接

在 `/etc/postgresql/9.5/main/pg_hba.conf`中：
```
host all all 127.0.0.1/32 md5
host all all 192.168.1.0/24 md5
```
在`/etc/postgreql/9.5/main/postgresql.conf`中：
```
listen_address = 'localhost,192.168.1.2'
port = 5432
max_connections = 80
```

### 配置Odoo
默认的，Odoo通过5432端口连接本地的PostgreSQL，可以通过配置[db参数](http://odoo.doc.zj.com/reference/cmdline.html#reference-cmdline-server-database)来覆盖默认的配置。
如果使用deb或rpm包安装的odoo，默认会创建一个`odoo`用户来连接pg。

- 数据库的管理密码为`admin_passwd`配置项的值，在对数据库进行修改操作时，会校验此密码，此密码建议随机生成一个。
- 所有的数据库操作，都会使用[这些配置](http://odoo.doc.zj.com/reference/cmdline.html#reference-cmdline-server-database)，在也需要连接postgresql的user有`createdb`的权限。
- users可以删除他们自己的db。如果想要在/web/database/manager界面中，使那些创建db的按钮没有效果，那么连接pg的user就要`no-createdb`权限，并且没有所有数据库都要归属于不同的用户。

> 警告： pg user不能为superuser

#### 配置样例
- 连接在192.168.1.2上的pg
- 端口为 5432
- 使用`odoo`用户连接数据库
- 密码为`pwd`
- 只要数据库名称以‘mycompany'开头的数据库

在`/etc/odoo.conf`中：
```
[options]
admin_passwd = mysupersecretpassword
db_host = 192.168.1.2
db_port = 5432
db_user = odoo
db_password = pwd
dbfilter = ^mycompany.*$
```

## 内建server
Odoo提供了内建的 HTTP servers，多进程和多线程的都有。
真实生产环境，建议使用多进程模式。

- 开启多进程服务，只需要配置一个非零的进程数即可，具体的进程数应当基于CPU的核数
- 也可以对worker进行一定的限制，避免资源枯竭

> 警告：windows中多进程模式无法启用

### woker数量的计算

- (#CPU * 2) + 1
- Cron worker也需要CPU
- 1个worker能支持大约6个并发用户

### 内存的计算

- 通常来说20%的请求比较重型，80%的请求比较简单
- 一个重型的worker大约 1GB的内存
- 轻型的workder大约150MB的内存

需要的内存 = #worker * ( (0.8 * 150mb) + (0.2 * 1024mb) ) 

### 在线聊天
在多进程模式中，一个专门用于聊天的进程会自动启动，并且监听 `longpolling port` 默认8072端口。
所以，你必须配置一个反向代理，将`/longpolling/`的请求，转至longpolling端口中。其他请求则转移至正常的监听端口

### 配置样例

- 一个4核8线程的服务器
- 60个并发用户
- 60 / 6 = 10 <-（理论上，需要的worker数）
- (4 * 2 ) + 1 = 9 <- (理论上，服务器支持的最大worker数）
- 我们使用8个worker 外加 1个cron进程，也使用监听系统来统计CPU load,看它是否处在 7 - 7.5之间。
- RAM = 9 * ((0.8 * 150) + (0.2 * 1024)) ~= 3GB

在 `/etc/odoo.conf` 中：
```
[options]
limit_memory_hard = 1677721600
limit_memory_soft = 629145600
limit_request = 8192
limit_time_cpu = 600
limit_time_real = 1200
max_cron_threads = 1
workers = 8
```

## HTTPS
odoo在提交信息时，使用的都是明文。所以为了安全，HTTPS是很有必要的：

 - Odoo在反向代理之后，应当开启Odoo的`proxy mode`
 - 设置SSL temination proxy [(Nginx 样例）](http://nginx.com/resources/admin-guide/nginx-ssl-termination/)
 - 设置转移到odoo的代理 [(Nginx 样例）](http://nginx.com/resources/admin-guide/reverse-proxy/)
 - 将http自动redirect到https

### 配置样例
在 `/etc/odoo.conf` 中：
```
proxy_mode = True
```
在`/etc/nginx/sites-enabled/odoo.conf`中：
```
#odoo server
upstream odoo {
 server 127.0.0.1:8069;
}
upstream odoochat {
 server 127.0.0.1:8072;
}

# http -> https
server {
   listen 80;
   server_name odoo.mycompany.com;
   rewrite ^(.*) https://$host$1 permanent;
}

server {
 listen 443;
 server_name odoo.mycompany.com;
 proxy_read_timeout 720s;
 proxy_connect_timeout 720s;
 proxy_send_timeout 720s;

 # Add Headers for odoo proxy mode
 proxy_set_header X-Forwarded-Host $host;
 proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
 proxy_set_header X-Forwarded-Proto $scheme;
 proxy_set_header X-Real-IP $remote_addr;

 # SSL parameters
 ssl on;
 ssl_certificate /etc/ssl/nginx/server.crt;
 ssl_certificate_key /etc/ssl/nginx/server.key;
 ssl_session_timeout 30m;
 ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
 ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
 ssl_prefer_server_ciphers on;

 # log
 access_log /var/log/nginx/odoo.access.log;
 error_log /var/log/nginx/odoo.error.log;

 # Redirect requests to odoo backend server
 location / {
   proxy_redirect off;
   proxy_pass http://odoo;
 }
 location /longpolling {
     proxy_pass http://odoochat;
 }

 # common gzip
 gzip_types text/css text/less text/plain text/xml application/xml application/json application/javascript;
 gzip on;
}
```

## 5. Odoo作为一个普通的WSGI应用
把Odoo当做一个WSGI应用是可以的，它也提供了一个基本的样例`odoo-wsgi.example.py`。如果要进行额外配置，建议复制这个文件，并且在`odoo.tools.config`进行修改，而不是使用命令行参数或修改配置文件。
如果这样使用Odoo，那么启动的worker实例就是标准的HTTP服务，没有cron worker和 livechat worker了。

### 5.1 Cron workers
这种情况，需要如下设置：
- 一个典型的Odoo进程（run via `odoo-bin`)
- 指定需要cron的db (via `odoo-bin -d `)
- 这个进程不应当处理来自网络的请求，需要关闭网络监听，`odoo-bin --no-xmlrpc` 或者配置文件中`xmlrpc = False`

### LiveChat
odoo作为wsgi应用启动，导致的第二问题就是，绝大部分HTTP请求都是简短的，需要快速返回响应，并且处理另外的请求，但是LiveChat需要一个针对每一个人的long-lived connection，用来处理实时消息提醒。
这个与多进程模式的设计的相对立的，这种长连接将会阻塞进程，进而阻止其他用户的连接。但是，livechat的请求相当简单，进程大部分时间都只是在等待而已。
所以解决这个问题的方法就是下列之一：
- 部署一个线程模式的Odoo，然后反向代理只把`/longpolling/`的请求转发到这个Odoo中，这种方式也顺便解决了Cron worker的问题，一举两得
- 或者，使用`odoo-bin gevent` 启动Odoo, 然后反向代理也只把`/longpolling/`的请求转发给它。

## 静态文件