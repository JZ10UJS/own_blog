---
title: 第1章 Odoo安装与配置
date: 2016-06-15 19:38:04
tags:
---

##1 运行环境介绍

 1. IDE: PyCharm 
 2. 开发环境：window下，使用virtualbox安装ubuntu16.04（64-bit）的虚拟环境
 3. odoo版本： odoo-9.0
 4. python版本： python 2.7.11

##2 安装配置

  2.1 安装和配置postgreSQL
Terminal中执行`sudo apt-get install postgresql`，静待postgreSQL安装完成，执行如下步骤：

```
$ sudo –u postgres createuser $USER
$ sudo –u postgres createdb $USER
```

此时，我们用postgres(ubuntu用户)创建了一个名为todd的（postgresql用户）和名字为todd的postgresql database，由于odoo运行时需要用户有创建db的权限，所以执行下面命令，更改todd role的database权限：

```
$ sudo –u postgres psql postgres
postgres=# ALTER USER todd createdb;
```

此时，todd用户就有创建db的权限了

 2.2. ubuntu可能缺少的文件
	先安装python-dev 和python-pip， 并将pip升级：
```
$ sudo apt-get install python-dev python-pip
$ pip install –U pip
```
关于python-ldap 模块的安装，执行如下命令：
```
$ sudo apt-get install libldap2-dev 
$ sudo apt-get install libsasl2-dev
```
关于psycopg2模块，执行如下命令：
```
$ sudo apt-get install libpq-dev
```
关于libxml2和lxml模块，执行如下命令：
```
$ sudo apt-get install libxml2-dev
$ sudo apt-get install libxslt1-dev
```
关于pillow模块，执行如下命令：

```
$ sudo apt-get install libjpeg-dev
```
odoo在网页端显示的时候，可能会出现缺少lessc之类的提示，导致网页显示不正常
所以执行：
```
$ sudo apt-get install -y npm
$ sudo npm install -g less less-plugin-clean-css
$ sudo ln -s /usr/bin/nodejs /usr/bin/node
```

2.3 odoo安装
	完成上述步骤之后，下载odoo源码
```
$ git clone https://github.com/odoo/odoo.git -b 9.0
```

然后cd至odoo.py同目录，安装依赖， 
```
$ sudo pip install -r requirements.txt
```
基本安装就没有问题了。
2.4 wkhtmltopdf安装
主要用于odoo报表打印，在 http://wkhtmltopdf.org/downloads.html 选取适合自己的版本。
我是 Ubuntu x64，所以选择了linux 64-bit
```
$ wget http://download.gna.org/wkhtmltopdf/0.12/0.12.3/wkhtmltox-0.12.3_linux-generic-amd64.tar.xz 
$ tar -xvf wkhtmlox-0.12.3_linux-generic-amd64.tar.xz
$ sudo cp wkhtmltox/bin/wkhtmltopdf /usr/bin/wkhtmltopdf  # 将解压后的二进制文件复制到 
$ sudo chown root:root /usr/bin/wkhtmltopdf 
$ sudo chmod +x /usr/bin/wkhtmltopdf
# 测试 
$ wkhtmltopdf www.baidu.com 1.pdf 
# 如果出现缺少 libXrender.so.1 或者 缺少 libfontconfig.so.1
$ sudo apt-get install libxrender-dev fontconfig
```

##3.工具配置
	由于源码是在虚拟机中，而常用IDE都在windows中，所以使用samba将虚拟机映射到window的盘符中，然后是用PyCharm进行开发。
	安装samba, 执行：
```
sudo apt-get install samba
```
然后添加一个用户（todd）用以登录，输入密码：

```
$ sudo smbpasswd –a todd
[sudo] passwd for todd:
New SMB password:
Retype new SMB password:
```

然后修改samba配置文件：

```
$ sudo vim /etc/samba/smb.conf
```

在文件尾添加：

```
[homes]
  commnet = Home Directories
  browseable = yes
  read only = no
  create mask = 0640
  directory mask = 0750
```
重启samba服务，sudo /etc/init.d/smbd restart, 然后，在window中
![这里写图片描述](http://img.blog.csdn.net/20160615193657146)
点击后，输入\\192.168.1.102\homes 前面就是虚拟机的ip地址，homes就是刚刚在samba配置文件中添加的，确认后，在弹出的新对话框中输入刚刚添加的用户名，密码。
 ![这里写图片描述](http://img.blog.csdn.net/20160615193715146)

