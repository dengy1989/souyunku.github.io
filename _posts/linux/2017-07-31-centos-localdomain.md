---
layout: post
title: CentOs7.3 修改主机名
categories: Linux
description: CentOs7.3 修改主机名
keywords: Linux
---

# CentOs7.3 修改主机名

## 第一种

命令格式 

```sh
hostname <hostname>
```

```sh
$ hostname node1  
```

这种方式，只能修改临时的主机名，当重启机器后，主机名称又变回来了。

## 第二种

命令格式

```sh
hostnamectl set-hostname <hostname>
```

```sh
$ hostnamectl set-hostname node1
```

## 重启服务器

```sh
$ reboot
```

## 重新连接服务器

```sh
$ hostname
node1
```


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - GitHub：[https://github.com/souyunku](https://github.com/souyunku)  
 - Segment Fault：[http://sf.gg/blog/souyunku](http://sf.gg/blog/souyunku)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")