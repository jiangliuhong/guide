---
title: centos7把home目录空间转移到其他目录
date: 2020-04-08 00:17:00:00
categories: 随笔
tags:
---

## 步骤

1.重启电脑
以root用户直接登陆（这是为了解决/home目录被占用的情况，也可以使用其它方式终止/home被占用，不过这样最直接）
2.卸载/home
umount /home
​3.删除/home所在的lv逻辑卷
lvremove /dev/centos00/home
小提示：如果不知道你的/home目录的路径，可以使用lvscan命令查看逻辑卷都有哪些，例如我的查
5.扩展/root所在的lv，增加100G
lvextend -L +100G  /dev/centos00/root
​6.扩展/root文件系统
xfs_growfs  /dev/centos00/root
7.重新创建home lv
lvcreate -L 70G -n home centos00
home：代表新建lv的名字
centos00：代表vg卷组的名字
而创建好之后，访问它的路径应该是：/dev/centos00/home（这个是路径的名字）
​8.创建文件系统
mkfs.xfs  /dev/centos00/home
9.挂载
​mount  /dev/centos00/home  /home

设置时间
timedatectl 命令
(1) 读取时间
timedatectl //等同于 timedatectl status
(2) 设置时间
timedatectl set-time "YYYY-MM-DD HH:MM:SS"
(3) 列出所有时区
timedatectl list-timezones
(4) 设置时区
timedatectl set-timezone Asia/Shanghai
(5) 是否NTP服务器同步
timedatectl set-ntp yes //yes或者no
(6) 将硬件时钟调整为与本地时钟一致
timedatectl set-local-rtc 1
hwclock --systohc --localtime //与上面命令效果一致