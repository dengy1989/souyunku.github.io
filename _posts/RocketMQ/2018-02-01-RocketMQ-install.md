---
layout: post
title: 搭建 Apache RocketMQ 单机环境
categories: RocketMQ
description: 搭建 Apache RocketMQ 单机环境
keywords: RocketMQ
---

# 快速开始

本快速入门指南是在您的本地机器上设置RocketMQ消息系统以发送和接收消息的详细说明。

# 环境需要

64位操作系统，建议使用Linux / Unix / 

- CentOs7.3
- 64bit JDK 1.8+;
- Maven 3.2.x

JDK Maven 的安装自行 Google 或者去我博客 [http://www.ymq.io](http://www.ymq.io) 搜索安装

# 下载和构建

**下载4.2.0源码版本:** [https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.2.0/rocketmq-all-4.2.0-source-release.zip](https://www.apache.org/dyn/closer.cgi?path=rocketmq/4.2.0/rocketmq-all-4.2.0-source-release.zip)

**下载4.2.0二进制版本:** [http://rocketmq.apache.org/release_notes/release-notes-4.2.0/](http://rocketmq.apache.org/release_notes/release-notes-4.2.0/)

现在执行以下命令来解压4.2.0源码版本并构建二进制文件

```sh
& unzip rocketmq-all-4.2.0-source-release.zip
& cd rocketmq-all-4.2.0/
& mvn -Prelease-all -DskipTests clean install -U
& mv distribution/target/apache-rocketmq /opt/apache-rocketmq
```

编译成功的响应

```sh
[INFO] Reactor Summary:
[INFO] 
[INFO] Apache RocketMQ 4.2.0 .............................. SUCCESS [04:21 min]
[INFO] rocketmq-remoting 4.2.0 ............................ SUCCESS [ 25.561 s]
[INFO] rocketmq-common 4.2.0 .............................. SUCCESS [  4.533 s]
[INFO] rocketmq-client 4.2.0 .............................. SUCCESS [  5.804 s]
[INFO] rocketmq-store 4.2.0 ............................... SUCCESS [  5.239 s]
[INFO] rocketmq-srvutil 4.2.0 ............................. SUCCESS [  2.177 s]
[INFO] rocketmq-filter 4.2.0 .............................. SUCCESS [  1.262 s]
[INFO] rocketmq-broker 4.2.0 .............................. SUCCESS [  3.129 s]
[INFO] rocketmq-tools 4.2.0 ............................... SUCCESS [  1.995 s]
[INFO] rocketmq-namesrv 4.2.0 ............................. SUCCESS [  1.322 s]
[INFO] rocketmq-logappender 4.2.0 ......................... SUCCESS [  1.549 s]
[INFO] rocketmq-openmessaging 4.2.0 ....................... SUCCESS [  1.560 s]
[INFO] rocketmq-example 4.2.0 ............................. SUCCESS [  1.242 s]
[INFO] rocketmq-filtersrv 4.2.0 ........................... SUCCESS [  0.680 s]
[INFO] rocketmq-test 4.2.0 ................................ SUCCESS [  3.047 s]
[INFO] rocketmq-distribution 4.2.0 ........................ SUCCESS [ 27.005 s]
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 06:09 min
[INFO] Finished at: 2018-02-01T15:51:26+08:00
[INFO] Final Memory: 73M/411M
[INFO] ------------------------------------------------------------------------
```

# Start Name Server

默认 RocketMQ Server 内存需要很大的

```sh
vi /bin/runserver.sh
```

```sh
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

我修改成

```sh
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

```sh
& cd /opt/apache-rocketmq
& nohup sh bin/mqnamesrv > /dev/null 2>&1 &
& tail -f ~/logs/rocketmqlogs/namesrv.log
```

会看到如下响应信息

```sh
2018-02-01 16:15:16 INFO main - The Name Server boot success... 
```

# Start Broker

默认 RocketMQ Broker 内存需要很大的

```sh
vi /bin/runbroker.sh
```

```sh
JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g -Xmn4g"
```

我修改成

```sh
JAVA_OPT="${JAVA_OPT} -server -Xms1g -Xmx1g -Xmn512m"
```

**启动Broker**

```sh
& nohup sh bin/mqbroker -n localhost:9876 > /dev/null 2>&1 &
& tail -f ~/logs/rocketmqlogs/broker.log
```

会看到如下响应信息

```sh
2018-02-01 17:37:48 INFO main - The broker[node1, 192.168.252.121:10911] boot success...
```

# 查看进程

```sh
[root@node1 apache-rocketmq]# jps
2374 BrokerStartup
2350 NamesrvStartup
```

```sh
[root@node1 apache-rocketmq]# netstat -ntlp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name          
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      1508/master         
tcp6       0      0 :::9876                 :::*                    LISTEN      2350/java                    
tcp6       0      0 ::1:25                  :::*                    LISTEN      1508/master         
tcp6       0      0 :::10909                :::*                    LISTEN      2374/java           
tcp6       0      0 :::10911                :::*                    LISTEN      2374/java           
tcp6       0      0 :::10912                :::*                    LISTEN      2374/java           
[root@node1 apache-rocketmq]#
```

# 发送并接受 消息

**Send & Receive Messages**

**发送消息**

发送/接收消息之前,我们需要告诉客户端 NameServer 地址。RocketMQ 提供了多种方式来实现这一目标。为简单起见,我们使用环境变量 NAMESRV_ADDR。

```sh
export NAMESRV_ADDR=localhost:9876
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
```

响应

```sh
SendResult [sendStatus=SEND_OK, msgId= ...
```

**消费消息**

```sh
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

响应

```sh
ConsumeMessageThread_%d Receive New Messages: [MessageExt...
```

# 停止服务

**停止 broker**

```sh
> sh bin/mqshutdown broker
The mqbroker(36695) is running...
Send shutdown request to mqbroker(36695) OK
```

**停止 namesrv**

```
> sh bin/mqshutdown namesrv
The mqnamesrv(36664) is running...
Send shutdown request to mqnamesrv(36664) OK
```


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/01/29/MongoDB-2](http://www.ymq.io/2018/01/29/MongoDB-2/)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

