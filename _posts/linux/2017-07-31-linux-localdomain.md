---
layout: post
title: linux修改主机名
categories: Linux
description: linux修改主机名
keywords: Linux
---

# linux修改主机名


 - linux系统安装好后，都会有默认的主机名：localhost.localdomain，修改主机名步骤如下

## 1.修改network文件

 - 修改HOSTNAME的值，改为要修改主机名

```
[root@souyunku ~]# vi /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=node1
```


## 2.修改hosts文件

 - localhost.localdomain 要设置的主机名。

```
[root@souyunku ~]# vi /etc/hosts
```

 - 注意：localhost.localdomain 修改后  node1

```
127.0.0.1   localhost node1 localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```


## 3.重启服务器

```
[root@souyunku ~]# reboot
```



## 4.重新连接服务器

```
Connecting to 192.168.252.10:22...
Connection established.
To escape to local shell, press 'Ctrl+Alt+]'.

Last login: Sat Dec 24 02:53:25 2016
[root@node1 ~]#
```

## 5.查看 hostname
 - 完成后用hostname命令查询系统主机名，可以看出系统主机名已经变更为node1

```
[root@node1 ~]# hostname
node1
[root@node1 ~]#
```

