---
layout: post
title: HBase-1.3.1 集群搭建 - 报错整理
categories: HBase
description: HBase-1.3.1 集群搭建 - 报错整理
keywords: HBase
---

# HBase 启动报错

## Waiting for dfs to exit safe mode...


**启动 Hadoop 和HBase之后，执行jps命令，已经看到有 HMaster，HRegionServer的进程**

**Maste**

```sh
$ jps
6100 SecondaryNameNode
6634 HMaster	#Master 进程
7354 Jps
6254 ResourceManager
5903 NameNode
```

**node2,node3 (salve)**

```sh
$ jps
2528 NodeManager
2417 DataNode
2687 HRegionServer		#hbase msalve进程
```

**进入到logs目录查看master的日志：发现一直显示面的内容：**

```sh
cd /logs/hbase-hadoop-master-node1.log
```

```sh
2017-09-21 11:20:19,249 INFO  [node1:16000.activeMasterManager] util.FSUtils: Waiting for dfs to exit safe mode...
2017-09-21 11:20:29,254 INFO  [node1:16000.activeMasterManager] util.FSUtils: Waiting for dfs to exit safe mode...
2017-09-21 11:20:39,256 INFO  [node1:16000.activeMasterManager] util.FSUtils: Waiting for dfs to exit safe mode...
2017-09-21 11:20:49,262 INFO  [node1:16000.activeMasterManager] util.FSUtils: Waiting for dfs to exit safe mode...
```

原来是Hadoop在刚启动的时候，还处在安全模式造成的。


## 解决方法

**可等Hadoop退出安全模式后再执行HBase命令，或者手动退出Hadoop的安全模式**


```sh
cd /home/hadoop/hadoop-2.7.4/sbin
./hadoop dfsadmin -safemode leave
```

```sh
DEPRECATED: Use of this script to execute hdfs command is deprecated.
Instead use the hdfs command for it.

Safe mode is OFF
```

**现在再执行HBase的命令就没有问题了。**

```sh
cd /home/hadoop/hbase-1.3.1/bin
./start-hbase.sh
```


**hadoop dfsadmin-safemode 命令参数说明：**

```sh
enter    - 进入安全模式
leave    - 强制NameNode离开安全模式
get      - 返回安全模式是否开启的信息
wait     - 等待，一直到安全模式结束。
```

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")