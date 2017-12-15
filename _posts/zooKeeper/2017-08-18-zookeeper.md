---
layout: post
title: CentOs7.3 搭建 ZooKeeper-3.4.9 单机服务
categories: ZooKeeper
description: CentOs7.3 搭建 ZooKeeper-3.4.9 单机服务
keywords: ZooKeeper
---

# 概述

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

JDK环境：jdk-8u144-linux-x64.tar.gz 

**JDK 1.8 安装**

具体参考[《CentOs7.3 安装 JDK1.8》](https://segmentfault.com/a/1190000010716919)

# ZooKeeper安装

## 1.下载ZooKeeper

下载最新版本的ZooKeeper ，我在北京我就选择，清华镜像比较快

清华镜像:[https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/ ](https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/)
 
阿里镜像:[https://mirrors.aliyun.com/apache/zookeeper/](https://mirrors.aliyun.com/apache/zookeeper/)

```sh
cd /opt/
wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.9/zookeeper-3.4.9.tar.gz
```

或者在浏览器下载上传至opt 目录

## 2.提取tar文件

```sh
cd /opt/
tar -zxf  zookeeper-3.4.9.tar.gz
cd zookeeper-3.4.9
```

创建`data`文件夹 用于存储数据文件

```sh
mkdir data  logs
```

创建`logs`文件夹 用于存储日志
```sh
mkdir logs  
```

## 3.创建配置文件

使用命令 `vi conf/zoo.cfg` 创建配置文件并打开，ps (其实目录`conf` 下有默认的配置文件，但是注释太多，英文一大堆，太乱)

```sh
vi  /opt/zookeeper-3.4.9/conf/zoo.cfg
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
```


> 配置文件描述


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

# ZooKeeper操作

## 启动服务

```sh
/opt/zookeeper-3.4.9/bin/zkServer.sh start
```

响应

```sh
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Starting zookeeper ... STARTED
```

## 连接服务

连接到ZooKeeper服务

```sh
/opt/zookeeper-3.4.9/bin/zkCli.sh
```

响应

```sh
Connecting to localhost:2181
2017-08-22 16:43:05,954 [myid:] - INFO  [main:Environment@100] - Client environment:zookeeper.version=3.4.9-1757313, built on 08/23/2016 06:50 GMT
2017-08-22 16:43:05,958 [myid:] - INFO  [main:Environment@100] - Client environment:host.name=node1
2017-08-22 16:43:05,958 [myid:] - INFO  [main:Environment@100] - Client environment:java.version=1.8.0_144
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:java.vendor=Oracle Corporation
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:java.home=/usr/lib/jvm/jre
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:java.class.path=/opt/zookeeper-3.4.9/bin/../build/classes:/opt/zookeeper-3.4.9/bin/../build/lib/*.jar:/opt/zookeeper-3.4.9/bin/../lib/slf4j-log4j12-1.6.1.jar:/opt/zookeeper-3.4.9/bin/../lib/slf4j-api-1.6.1.jar:/opt/zookeeper-3.4.9/bin/../lib/netty-3.10.5.Final.jar:/opt/zookeeper-3.4.9/bin/../lib/log4j-1.2.16.jar:/opt/zookeeper-3.4.9/bin/../lib/jline-0.9.94.jar:/opt/zookeeper-3.4.9/bin/../zookeeper-3.4.9.jar:/opt/zookeeper-3.4.9/bin/../src/java/lib/*.jar:/opt/zookeeper-3.4.9/bin/../conf:.:/lib/jvm/lib:/lib/jvm/jre/lib
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:java.library.path=/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:java.io.tmpdir=/tmp
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:java.compiler=<NA>
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:os.name=Linux
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:os.arch=amd64
2017-08-22 16:43:05,967 [myid:] - INFO  [main:Environment@100] - Client environment:os.version=3.10.0-514.26.2.el7.x86_64
2017-08-22 16:43:05,968 [myid:] - INFO  [main:Environment@100] - Client environment:user.name=root
2017-08-22 16:43:05,968 [myid:] - INFO  [main:Environment@100] - Client environment:user.home=/root
2017-08-22 16:43:05,968 [myid:] - INFO  [main:Environment@100] - Client environment:user.dir=/opt/zookeeper-3.4.9
2017-08-22 16:43:05,969 [myid:] - INFO  [main:ZooKeeper@438] - Initiating client connection, connectString=localhost:2181 sessionTimeout=30000 watcher=org.apache.zookeeper.ZooKeeperMain$MyWatcher@506c589e
Welcome to ZooKeeper!
2017-08-22 16:43:06,011 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1032] - Opening socket connection to server localhost/0:0:0:0:0:0:0:1:2181. Will not attempt to authenticate using SASL (unknown error)
JLine support is enabled
2017-08-22 16:43:06,164 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@876] - Socket connection established to localhost/0:0:0:0:0:0:0:1:2181, initiating session
2017-08-22 16:43:06,237 [myid:] - INFO  [main-SendThread(localhost:2181):ClientCnxn$SendThread@1299] - Session establishment complete on server localhost/0:0:0:0:0:0:0:1:2181, sessionid = 0x15e091bf2020000, negotiated timeout = 30000

WATCHER::

WatchedEvent state:SyncConnected type:None path:null

[zk: localhost:2181(CONNECTED) 0] 

```

## 服务状态

```sh
/opt/zookeeper-3.4.9/bin/zkServer.sh status
```

响应

```
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Mode: standalone
```

## 停止服务

```sh
/opt/zookeeper-3.4.9/bin/zkServer.sh stop
```

响应

```sh
bin/zkServer.sh stop
ZooKeeper JMX enabled by default
Using config: /opt/zookeeper-3.4.9/bin/../conf/zoo.cfg
Stopping zookeeper ... STOPPED
```

## 推荐阅读
[CentOs7.3 搭建 ZooKeeper-3.4.9 Cluster 集群服务](https://segmentfault.com/a/1190000010807875)


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - GitHub：[https://github.com/souyunku](https://github.com/souyunku)  
   
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

