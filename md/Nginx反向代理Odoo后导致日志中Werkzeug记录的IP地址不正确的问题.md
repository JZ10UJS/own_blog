---
title: Nginx反向代理Odoo后导致日志中Werkzeug记录的IP地址不正确的问题
date: 2016-08-04 16:36:57
tags:
---

### 使用环境

 - 主机（win7 192.168.1.78） 虚机 （ubuntu 192.168.1.102）
 - ubuntu 16.04
 - nginx 1.10.0
 - odoo 9.0c

### 问题描述

 1. 在odoo不使用代理的情况下，日志中记录的ip地址是正确的
	![这里写图片描述](http://img.blog.csdn.net/20160804150426082)
	
 2. 配置nginx
	``` bash
	$ sudo vim /etc/nginx/sites-available/odoo.conf
	# odoo.conf 配置如下
	# 也可以先删除/etc/nginx/sites-available/default, 因为监听80端口冲突了
	--- odoo.conf ---
	server {
		listen 80 default;
		server_name _;
		
		location / {
			proxy_pass http://127.0.0.1:8069;
			proxy_next_upstream error timeout invalid_header http_500 http_502 http_504;
			
			proxy_buffer_size 128k;
	        proxy_buffers 16 64k;
			proxy_redirect off;

			proxy_set_header Host $host;
		    proxy_set_header X-Real-IP $remote_addr;  # 这一行必须要有
			proxy_set_header X-Forwarded-HOST $host;  # 这一行必须要有
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		}
		
		location ~* /[a-zA-Z0-9_]*/static/ {
			proxy_cache_valid 200 60m;
	        proxy_buffering    on;
	        expires 864000;
	        proxy_pass http://127.0.0.1:8069;
		}
	}
	---
	$ sudo ln /etc/nginx/sites-available/odoo.conf /etc/nginx/sites-enabled/odoo.conf
	$ sudo service nginx start
	```

 3. 验证
	重新启动odoo, 在 ~/.openerp_serverrc的文件中，我并没有设定 xmlrpc_interface，**但是 proxy_mode = True 一定要设置**。odoo开启后，我们有两种方式可以从**宿主机**访问：
	-  访问 http://192.168.1.102/    这种情况，将由Nginx代理之后，转发请求给odoo
	-  访问 http://192.168.1.102:8069/ 这种情况，将直接访问odoo
		
![这里写图片描述](http://img.blog.csdn.net/20160804153154082)

  此时，werkzeug 的 log 就出现了错误，他并没有处理正确的 IP 地址。
  查看源码，我们可以在
  odoo-9.0/openerp/service/wsgi_server.py 中看到
  ![这里写图片描述](http://img.blog.csdn.net/20160804153938441)
	这也是为什么在前面在nginx配置中一定要 set header X-Forwarded-Host的原因。但是，即使调用了ProxyFix， Werkzeug也并没有 log 出正确的 IP地址， **所以我们有理由怀疑这个是Werkzeug的锅**

###测试 Werkzeug
抛开odoo, 我们使用最简单的wsgi app来测试werkzeug
在home目录下，创建 app.py 文件，内容如下
``` python
#!/usr/bin/python
# coding: utf-8

import logging
from werkzeug.serving import run_simple, WSGIRequestHandler
from werkzeug.contrib.fixers import ProxyFix

def application(environ, start_response):
	start_response('200 OK', [('Content-Type', 'text/plain')])
	return 'Hello World\n'

# 这个是按照wsgi_server.py中源码的样式写的
def dispatch_app(environ, start_response):
	# 去掉了对 config 的要求，因为我们没有config
	if 'HTTP_X_FORWARDED_HOST' in environ:
		return ProxyFix(application)(environ, start_response)
	else:
		return application(environ, start_response)


if __name__ == "__main__":
	logging.basicConfig(level=10)
	run_simple('0.0.0.0', 8080, dispatch_app)
```
我们让他跑起来， 并用宿主机进行访问 http://192.168.1.102:8080
```
$ python ~/app.py 
```
![这里写图片描述](http://img.blog.csdn.net/20160804160349281)
可以看到地址是正确的，

我们再来配置 Nginx， 使用反向代理来测试这个app

```
$ sudo vim /etc/nginx/sites-available/app.conf
# ----app.conf ------
server {
	listen 8044 default;
	server_name __;
	location / {
		proxy_pass http://127.0.0.1:8080;
		
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;  # 这一行必须要有
		proxy_set_header X-Forwarded-HOST $host;  # 这一行必须要有
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	}
}
# -------------------
$ sudo ln /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/app.conf
$ sudo service nginx restart
```
此时宿主机可以通过两种方式访问虚拟机
   

 - http://192.168.1.102:8080 直接访问 app
 - http://192.168.1.102:8044 通过Nginx反向代理后，访问 app

![这里写图片描述](http://img.blog.csdn.net/20160804162238240)

可以看出，抛出Odoo这个干扰项之后，Werkzeug还是不能记录出正确的 IP 地址， 所以，**我们决定肯定这锅就是werkzeug的了**。

### 解决方法

在google搜索一大堆之后，终于在搜索引擎中将odoo关键词去掉，进而专注搜索werkzeug的问题，鄙人终于找到了解决方法， 重写 WSGIRequestHandler.address_string方法

在app.py中添加

```
# 添加此函数
def fix_werkzeug_logging():
	from werkzeug.serving import WSGIRequestHandler
	
    def address_string(self):
	    # 这就是在nginx的config中，为什么一定要有X-Real-IP啦
        return self.headers.get('x-real-ip', self.client_address[0])
    WSGIRequestHandler.address_string = address_string

if __name__ == "__main__":
	fix_werkzeug_logging()  # 这是新增的行
	logging.basicConfig(level=10)
	run_simple('0.0.0.0', 8080, dispatch_app)
```

重新启动 python ~/app.py， 用宿主机访问，发现问题 解决啦！
![这里写图片描述](http://img.blog.csdn.net/20160804163159648)

那么在odoo中修改 odoo-9.0/openerp/service/wsgi_server.py 的 application 函数，新增此行即可！

```
def application(environ, start_response):
    if config['proxy_mode'] and 'HTTP_X_FORWARDED_HOST' in environ:
	    # 增加此行
        werkzeug.serving.WSGIRequestHandler.address_string = lambda self: self.headers.get('x-real-ip', self.client_address[0])
        return werkzeug.contrib.fixers.ProxyFix(application_unproxied)(environ, start_response)
    else:
        return application_unproxied(environ, start_response)
```

至此，问题解决！