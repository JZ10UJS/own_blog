---
title: Virtualbox下Ubuntu扩展根分区
date: 2016-08-10 16:00:48
tags:
---



## virtualbox 扩展虚拟机硬盘大小
当虚拟机使用一段时间后，我们会发现硬盘容量不够了，所以我们来扩充之。
**关闭虚拟机**

![这里写图片描述](http://img.blog.csdn.net/20160811152205106)

- cd 至 virtualbox 安装目录
``` cmd
c:\Program Files\Oracle\VirtualBox\VBoxManage.exe list hdds
```
![这里写图片描述](http://img.blog.csdn.net/20160811152853006)

- 扩展硬盘大小

```
c:\Program Files\Oracle\VirtualBox\VBoxManage.exe modifyhd "c:\users\pc\virtualbox vms\basic-odoo\basic-odoo.vdi" --resize 20480
```
- 查看 重新打开VirtualBox

![这里写图片描述](http://img.blog.csdn.net/20160811153430325)

## ubuntu 扩展根分区
**参考 http://xuxuezhe.blog.51cto.com/1636138/1219601**

 - 新建分区
```
$ sudo df -h  # 查看
$ sudo fdisk -l 
```
![这里写图片描述](http://img.blog.csdn.net/20160811153950145)

![这里写图片描述](http://img.blog.csdn.net/20160811154035849)

这里看到 20G 已经加进来了，但是没有分配
```
$ sudo fdisk /dev/sda
#--------fdisk--------#
输入 n # 创建一个新分区（new）
	输入 p # 此分区为主分区（primary）
	输入 分区的开始和结束。 开始可以不填，使用默认； 结束可以填写 +10G 或不填写，直接分配完
输入 p 可以查看到之前创立的分区信息 # 假设为 /dev/sda3
输入 t 进入修改
	输入 3 # 因为创建的是3
	输入 8e # 修改此分区为 LVM 逻辑卷 
输入 w # 保存退出
#---------------------#
```
 - 创建物理卷
```
$ sudo pvcreate /dev/sda3 # 创建物理卷 physical volumes
$ sudo pvdisplay
$ sudo pvscan    # 这两个都是查看的命令
```
 - 扩容VG

```
$ df -h # 可以查看到 /dev/mapper/ubuntu--vg-root mount的 /
$ sudo vgdisplay # 查出组名为 ubuntu-vg 且 Free PE / Size  为 0 / 0
$ sudo vgextend ubuntu-vg /dev/sda3 # 扩容， Free PE / Size 会变为 10G （/dev/sda3的容量）
```

 - 扩容LV
 

```
$ sudo lvextend -L +10G /dev/mapper/ubuntu--vg-root # 可能会由于容量不足失败，具体看提示
$ sudo lvextend -L +9.996G /dev/mapper/ubuntu--vg-root # 10G对应2560PE块, 可能只有2559可以使用
$ sudo resize2fs /dev/ubuntu-vg/root # 使其生效
```

