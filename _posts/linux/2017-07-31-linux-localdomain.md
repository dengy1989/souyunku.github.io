---
layout: post
title: CentOs6.5 修改主机名
categories: Linux
description: CentOs6.5 修改主机名
keywords: Linux
---

# CentOs6.5 修改主机名

 - CentOs6.5 系统安装好后，都会有默认的主机名：localhost.localdomain，修改主机名步骤如下


## 1.修改network文件

 - 修改HOSTNAME的值，改为要修改主机名

```
$ vi /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=node1
```


## 2.修改hosts文件

 - localhost.localdomain 要设置的主机名。

```
$ vi /etc/hosts
```

 - 注意：localhost.localdomain 修改后  node1

```
127.0.0.1   localhost node1 localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
```


## 3.重启服务器

```
$ reboot
```



## 4.重新连接服务器查看 hostname

 - 完成后用hostname命令查询系统主机名，可以看出系统主机名已经变更为node1

```
$ hostname
node1
```


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")