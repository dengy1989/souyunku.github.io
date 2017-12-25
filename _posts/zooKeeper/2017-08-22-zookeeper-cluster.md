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

要想理解zookeeper到底是做啥的，那首先得理解清楚，**什么是一致性？**

所谓的一致性，实际上就是围绕着“看见”来的。谁能看见？能否看见？什么时候看见？举个例子：淘宝后台卖家，在后台上架一件大促的商品，通过服务器A提交到主数据库，假设刚提交后立马就有用户去通过应用服务器B去从数据库查询该商品，就会出现一个现象，卖家已经更新成功了，然而买家却看不到；而经过一段时间后，主数据库的数据同步到了从数据库，买家就能查到了。

假设卖家更新成功之后买家立马就能看到卖家的更新，则称为**强一致性**

如果卖家更新成功后买家不能看到卖家更新的内容，则称为**弱一致性**

而卖家更新成功后，买家经过一段时间最终能看到卖家的更新，则称为**最终一致性**


[《一致性协议》](http://www.cnblogs.com/leesf456/p/6001278.html)

[《ZooKeeper应用场景》](http://www.cnblogs.com/leesf456/p/6036548.html)

[《分布式架构》](http://www.cnblogs.com/leesf456/p/5992377.html)

[《分布式 ZooKeeper 系列》](http://www.cnblogs.com/leesf456/tag/%E5%88%86%E5%B8%83%E5%BC%8F/)


# 环境


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

**zoo.cfg**

zookeeper的主要配置文件，因为Zookeeper是一个集群服务，集群的每个节点都需要这个配置文件。为了避免出差错，zoo.cfg这个配置文件里没有跟特定节点相关的配置，所以每个节点上的这个zoo.cfg都是一模一样的配置。这样就非常便于管理了，比如我们可以把这个文件提交到版本控制里管理起来。其实这给我们设计集群系统的时候也是个提示：集群系统一般有很多配置，应该尽量将通用的配置和特定每个服务的配置(比如服务标识)分离，这样通用的配置在不同服务之间copy就ok了


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

### 配置文件描述


**tickTime** 

 - tickTime则是上述两个超时配置的基本单位，例如对于initLimit，其配置值为5，说明其超时时间为 2000ms * 5 = 10秒。

**dataDir**

 - 其配置的含义跟单机模式下的含义类似，不同的是集群模式下还有一个myid文件。myid文件的内容只有一行，且内容只能为1 - 255之间的数字，这个数字亦即上面介绍server.id中的id，表示zk进程的id。

**dataLogDir**

 - 如果没提供的话使用的则是dataDir。zookeeper的持久化都存储在这两个目录里。dataLogDir里是放到的顺序日志(WAL)。而dataDir里放的是内存数据结构的snapshot，便于快速恢复。为了达到性能最大化，一般建议把dataDir和dataLogDir分到不同的磁盘上，这样就可以充分利用磁盘顺序写的特性。

**initLimit**

 - ZooKeeper集群模式下包含多个zk进程，其中一个进程为leader，余下的进程为follower。 
当follower最初与leader建立连接时，它们之间会传输相当多的数据，尤其是follower的数据落后leader很多。initLimit配置follower与leader之间建立连接后进行同步的最长时间。

**syncLimit**

 - 配置follower和leader之间发送消息，请求和应答的最大时间长度。


**server.id=host:port1:port2**


`server.id` 其中id为一个数字，表示zk进程的id，这个id也是data目录下myid文件的内容

`host` 是该zk进程所在的IP地址

`port1` 表示follower和leader交换消息所使用的端口

`port2` 表示选举leader所使用的端口


## 4.创建myid 文件

在data里会放置一个myid文件，里面就一个数字，用来唯一标识这个服务。这个id是很重要的，一定要保证整个集群中唯一

ZooKeeper会根据这个id来取出server.x上的配置。比如当前id为1，则对应着zoo.cfg里的server.1的配置

```sh
$ echo "1" > /opt/zookeeper-3.4.9/data/myid
```

这样一台node1机器就配置完了


## 5.复制集群配置

在集群node1 上执行,复制配置好的zookeeper到其他两台主机上 

```sh
$ for a in {2..3} ; do scp -r /opt/zookeeper-3.4.9/ node$a:/opt/zookeeper-3.4.9 ; done
```

在集群node1 上执行 ,批量修改myid 文件


```sh
$ for a in {1..3} ; do ssh node$a "source /etc/profile; echo $a > /opt/zookeeper-3.4.9/data/myid" ; done
```

# 集群操作

**在集群任意一台机器上执行**

## 启动集群
```sh
$ for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/zookeeper-3.4.9/bin/zkServer.sh start" ; done
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

## 连接集群

```sh
$ /opt/zookeeper-3.4.9/bin/zkCli.sh -server node1:2181,node2:2181,node3:2181
```

响应
```sh
Connecting to node1:2181,node2:2181,node3:2181
2017-08-23 11:08:10,323 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.9-1757313, built on 08/23/2016 06:50 GMT
2017-08-23 11:08:10,328 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=node1
2017-08-23 11:08:10,329 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.8.0_144
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/usr/lib/jvm/jre
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/opt/zookeeper-3.4.9/bin/../build/classes:/opt/zookeeper-3.4.9/bin/../build/lib/*.jar:/opt/zookeeper-3.4.9/bin/../lib/slf4j-log4j12-1.6.1.jar:/opt/zookeeper-3.4.9/bin/../lib/slf4j-api-1.6.1.jar:/opt/zookeeper-3.4.9/bin/../lib/netty-3.10.5.Final.jar:/opt/zookeeper-3.4.9/bin/../lib/log4j-1.2.16.jar:/opt/zookeeper-3.4.9/bin/../lib/jline-0.9.94.jar:/opt/zookeeper-3.4.9/bin/../zookeeper-3.4.9.jar:/opt/zookeeper-3.4.9/bin/../src/java/lib/*.jar:/opt/zookeeper-3.4.9/bin/../conf:.:/lib/jvm/lib:/lib/jvm/jre/lib
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=3.10.0-514.26.2.el7.x86_64
2017-08-23 11:08:10,331 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=root
2017-08-23 11:08:10,332 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/root
2017-08-23 11:08:10,332 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/root
2017-08-23 11:08:10,333 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=node1:2181,node2:2181,node3:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@506c589e
2017-08-23 11:08:10,361 [myid:] - INFO  [main-SendThread(node3:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server node3/192.168.252.123:2181. Will not attempt to authenticate using SASL (unknown error)
Welcome to ZooKeeper!
JLine support is enabled
2017-08-23 11:08:10,474 [myid:] - INFO  [main-SendThread(node3:2181):ClientCnxn$SendThread@876] - Socket connection established to node3/192.168.252.123:2181, initiating session
[zk: node1:2181,node2:2181,node3:2181(CONNECTING) 0] 2017-08-23 11:08:10,535 [myid:] - INFO  [main-SendThread(node3:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server node3/192.168.252.123:2181, sessionid = 0x35e0d0716340000, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null

[zk: node1:2181,node2:2181,node3:2181(CONNECTED) 0]
```

从日志可以看出客户端成功连接的是node3 连接上哪台机器的zk进程是随机的

```sh

2017-08-23 11:08:10,361 [myid:] - INFO  [main-SendThread(node3:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server node3/192.168.252.123:2181. Will not attempt to authenticate using SASL (unknown error)
Welcome to ZooKeeper!
JLine support is enabled
```


## 集群状态

```sh
$ for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/zookeeper-3.4.9/bin/zkServer.sh status" ; done
```

响应
```sh
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Mode: follower
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Mode: leader
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Mode: follower
```

通过日志我可以看到  node2 leader (ps 是老大)，其他 node1 ,node2 follower   (ps 都是小弟)

Leader 怎么选举的可以参考[《Zookeeper的Leader选举》](http://www.cnblogs.com/leesf456/p/6107600.html)
## 停止集群

```sh
$ for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/zookeeper-3.4.9/bin/zkServer.sh stop" ; done
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

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

