---
layout: post
title: CentOs7.3 搭建 ZooKeeper-3.4.9 Cluster 集群服务
categories: ZooKeeper
description: CentOs7.3 搭建 ZooKeeper-3.4.9 Cluster 集群服务
keywords: ZooKeeper
---

#  CentOs7.3 搭建 ZooKeeper-3.4.9 Cluster 集群服务

# Zookeeper 概述

zookeeper实际上是yahoo开发的，用于分布式中**一致性处理的框架**。最初其作为研发Hadoop时的副产品。由于分布式系统中一致性处理较为困难，其他的分布式系统没有必要 费劲重复造轮子，故随后的分布式系统中大量应用了zookeeper，以至于zookeeper成为了各种分布式系统的基础组件，其地位之重要，可想而知。著名的**hadoop，kafka，dubbo 都是基于zookeeper而构建**。

要想理解zookeeper到底是做啥的，那首先得理解清楚，**什么是一致性？**。

所谓的一致性，实际上就是围绕着“看见”来的。谁能看见？能否看见？什么时候看见？举个例子：淘宝后台卖家，在后台上架一件大促的商品，通过服务器A提交到主数据库，假设刚提交后立马就有用户去通过应用服务器B去从数据库查询该商品，就会出现一个现象，卖家已经更新成功了，然而买家却看不到；而经过一段时间后，主数据库的数据同步到了从数据库，买家就能查到了。

假设卖家更新成功之后买家立马就能看到卖家的更新，则称为**强一致性**

如果卖家更新成功后买家不能看到卖家更新的内容，则称为**弱一致性**

而卖家更新成功后，买家经过一段时间最终能看到卖家的更新，则称为**最终一致性**


[Zookeeper 概述引用 http://blog.csdn.net/liweisnake/article/details/63251252](http://blog.csdn.net/liweisnake/article/details/63251252)

## 环境


VMware版本号：12.0.0

CentOS版本：CentOS 7.3.1611

ZooKeeper版本：ZooKeeper-3.4.9.tar.gz

虚拟机IP：192.168.252.101，192.168.252.102，192.168.252.103

集群主机名称：node1，node2，node3 具体参考[《CentOs7.3 修改主机名》](https://segmentfault.com/a/1190000010723105) 

集群主机用户：都是用root用户

集群JDK环境：jdk-8u144-linux-x64.tar.gz JDK 1.8 安装 具体参考[《CentOs7.3 安装 JDK1.8》](https://segmentfault.com/a/1190000010716919)

集群主机之间设置免密登陆：设置方式见：[《CentOs7.3 ssh 免密登录》](https://segmentfault.com/a/1190000010738165)


## 注意事项
 

关闭防火墙

centos 6.x 关闭 iptables
```sh
$ service iptables stop # 关闭命令：
```

centos 7.x 关闭firewall

```sh
$ systemctl stop firewalld.service # 停止firewall
```

# ZooKeeper 安装

## 1.下载ZooKeeper



下载最新版本的ZooKeeper ，我在北京我就选择，清华镜像比较快

清华镜像:[https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/ ](https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/)
 
阿里镜像:[https://mirrors.aliyun.com/apache/zookeeper/](https://mirrors.aliyun.com/apache/zookeeper/)

```sh
$ cd /opt/
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.9/zookeeper-3.4.9.tar.gz
```

或者在浏览器下载上传至opt 目录

## 2.提取tar文件

```sh
$ cd /opt/
$ tar -zxf  zookeeper-3.4.9.tar.gz
$ cd zookeeper-3.4.9
```

创建`data`文件夹 用于存储数据文件

```sh
$ mkdir /opt/zookeeper-3.4.9/data
```

创建`logs`文件夹 用于存储日志
```sh
$ mkdir /opt/zookeeper-3.4.9/logs  
```

## 3.创建配置文件

使用命令 `vi conf/zoo.cfg` 创建配置文件并打开，ps (其实目录`conf` 下有默认的配置文件，但是注释太多，英文一大堆，太乱)

```sh
$ vi /opt/zookeeper-3.4.9/conf/zoo.cfg
```

编辑内容如下

```sh
tickTime = 2000
dataDir =  /opt/zookeeper-3.4.9/data
dataLogDir = /opt/zookeeper-3.4.9/logs
tickTime = 2000
clientPort = 2181
initLimit = 5
syncLimit = 2

server.1=node1:2888:3888
server.2=node2:2888:3888
server.3=node3:2888:3888
```

## 4.创建myid 文件

server.X 构成ZooKeeper服务的服务器。当服务器启动时，它通过查找data目录中的文件myid来知道它是哪个服务器 

```sh
echo "1" > /opt/zookeeper-3.4.9/data/myid
```

这样一台node1机器就配置完了


## 5.复制集群配置

在集群node1 上执行,复制配置好的zookeeper到其他两台主机上 

```sh
for a in {2..3} ; do scp -r /opt/zookeeper-3.4.9/ node$a:/opt/zookeeper-3.4.9 ; done
```

在集群node1 上执行 ,批量修改myid 文件


```sh
for a in {1..3} ; do ssh node$a "source /etc/profile; echo $a > /opt/zookeeper-3.4.9/data/myid" ; done
```

## 6.启动集群

```sh
for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/zookeeper-3.4.9/bin/zkServer.sh start" ; done
```

响应
```sh
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED

```

## 7.集群状态

```sh
for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/zookeeper-3.4.9/bin/zkServer.sh status" ; done
```


## 7.停止集群

```sh
for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/zookeeper-3.4.9/bin/zkServer.sh stop" ; done
```

响应
```
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED

```