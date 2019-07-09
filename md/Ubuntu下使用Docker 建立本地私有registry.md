---
title: Ubuntu下使用Docker 建立本地私有registry
date: 2016-07-19 12:36:41
tags:
---

### 版本信息

 1. ubuntu 16.04
 2. docker 1.11.2
 3. registry 2

### 安装docker
使用安装脚本安装docker，保证本机有安装curl。
```
$ sudo curl https://get.docker.com/ | sh
```
安装完成之后，可以使用docker info 或 docker version 查看docker的相关信息。
由于docker需要sudo命令，所以可以将当前用户加入docker组，然后**退出当前用户，再次登录**，就可以不用使用sudo了。

```
$ sudo gpasswd -a ${USER} docker
```

### 安装registry
由于docker中registry的latest版本为1.x， 所以我们指定安装版本2。

```
$ docker pull registry:2
```
通过这个image创建一个container

```
$ docker run -p 5000:5000 -d --name=local_registry registry:2
$ docker ps 
```
之后使用docker ps查看就能看到名为 local_registry的container已经在后台运行了。
![这里写图片描述](http://img.blog.csdn.net/20160719100122313)
#### 测试registry
从官方下载hello-world image 用以测试， 192.168.1.102是本机局域网IP。

```
$ docker pull hello-world:latest
$ dcoker tag hello-world:latest 127.0.0.1:5000/hello-world
$ docker tag hello-world:latest 192.168.1.102:5000/hello-world
```
尝试push

```
$ docker push 127.0.0.1:5000/hello-world
```
![这里写图片描述](http://img.blog.csdn.net/20160719103046552)
进入container容器查看
```
$ docker exec -i -t local_registry /bin/bash
```
![这里写图片描述](http://img.blog.csdn.net/20160719103522007)

这个就看到了我们的上传的image了， 由于使用registry:2所以在容器中上传的image存在 /var/lib/registry中，我们将其删除。

![这里写图片描述](http://img.blog.csdn.net/20160719103838977)
然后exit退出容器。
如果再按照这个地址push， 就会出现错误。
```
$ docker push 192.168.1.102:5000/hello-world
```

![这里写图片描述](http://img.blog.csdn.net/20160719104927991)

这是由于使用https原因，查了很多，都说在/etc/default/docker中修改 DOCKER_OPTS， 然后重启docker。 然而这并**没有什么卵用**。

![这里写图片描述](http://img.blog.csdn.net/20160719105222823)

所以，这种方式只能用于本地主机，不能用在局域网中。所以，我们采取官方推荐的，用https，自建证书。
### 搭建https regsitry
#### server 端：

 - 生成自签名证书
```
$ mkdir -p ~/registry/certs && cd ~/registry/certs
$ openssl req -x509 -days 3650 -subj '/CN=reg.domain.com/' -nodes -newkey rsa:2048 -keyout registry.key -out registry.crt
```
这个由于ca认证不支持ip地址，所以采用域名，之后在hosts中添加映射就可以了。

- 生成用户名密码

```
$ mkdir -p ~/registry/auth && cd ~/registry/auth
$ docker run --rm --entrypoint htpasswd registry:2 -Bbn testuser password > ./htpasswd
```
使用registry:2中的 htpasswd 生成用户名密码 testuser password 并存到本地。

- 创建证书目录

```
$ mkdir -p /etc/docker/certs.d/reg.domain.com:5000
$ cp ~/registry/certs/registry.crt /etc/docker/certs.d/reg.domain.com:5000
```

 - 启动registry , 删除之前的 local_registry

```
$ docker stop local_registry && docker rm local_registry
$ docker run -d -p 5000:5000 --name=registry --restart=always -v ~/registry/auth:/auth -v ~/registry/certs:/certs -v /data/registry:/var/lib/registry -e REGISTRY_AUTH=htpasswd -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/registry.crt -e REGISTRY_HTTP_TLS_KEY=/certs/registry.key registry:2
```
#### client 端：
 - 证书相关
通过scp将证书copy到本地
```
$ sudo mkdir -p /etc/docker/certs.d/reg.domain.com:5000
$ sudo scp -r todd@192.168.1.102:/home/todd/registry/certs/registry.crt /etc/docker/certs.d/reg.domain.com:5000
```
 - 添加域名
	在 /etc/hosts 添加一行 192.168.1.102 reg.domain.com
```
$ sudo vim /etc/hosts
```

 - 测试
登录输入之前设置的 testuser 和 password
```
$ docker login reg.domain.com:5000
Username: testuser
Password: password
$ docker pull hello-world
$ docker tag hello-world:latest reg.domain.com:5000/hello-world
$ docker push reg.domain.com:5000/hello-world
```

