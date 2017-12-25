---
layout: post
title: 使用 阿里巴巴 Canal 增量订阅&消费组件 同步 MySQL 数据到 Redis
categories: Canal Redis MySQL
description: 使用 阿里巴巴 Canal 增量订阅&消费组件 同步 MySQL 数据到 Redis
keywords: Canal
---

# 使用 阿里巴巴 Canal 增量订阅&消费组件 同步 MySQL 数据到 Redis
## 背景

[《阿里巴巴的增量订阅&消费组件》 https://github.com/alibaba/canal ](https://github.com/alibaba/canal)

早期，阿里巴巴B2B公司因为存在杭州和美国双机房部署，存在跨机房同步的业务需求。不过早期的数据库同步业务，主要是基于trigger的方式获取增量变更，不过从2010年开始，阿里系公司开始逐步的尝试基于数据库的日志解析，获取增量变更进行同步，由此衍生出了增量订阅&消费的业务，从此开启了一段新纪元。

### 项目介绍

名称：运河[kə'næl]

译意：水道/管道/沟渠

语言：纯java开发

定位：基于数据库增量日志解析，提供增量数据订阅＆消费，目前主要支持了mysql

关键词：mysql binlog解析器/实时/队列和主题



基于日志增量订阅&消费支持的业务：

1.数据库镜像  
2.数据库实时备份  
3.多级索引 (卖家和买家各自分库索引)  
4.search build  
5.业务cache刷新  
6.价格变化等重要业务消息  


### 工作原理

#### mysql主备复制实现

![master-slave][2] 


从上层来看，复制分成三步：

master将改变记录到二进制日志(binary log)中（这些记录叫做二进制日志事件，binary log events，可以通过show binlog events进行查看）；
slave将master的binary log events拷贝到它的中继日志(relay log)；
slave重做中继日志中的事件，将改变反映它自己的数据。


[CentOs7.3 搭建 MySQL 5.7.19 主从复制，以及复制实现细节分析](https://segmentfault.com/a/1190000010867488)


#### canal的工作原理

![canal][1] 


原理相对比较简单：

1.运河模拟mysql slave的交互协议，伪装自己为mysql slave，向mysql master发送dump协议  
2.mysql master收到dump请求，开始推送二进制日志给slave（也就是运河）  
3.运河解析二进制对象（原始为字节流）  


canal的原理是基于mysql binlog技术，所以这里一定需要开启mysql的binlog写入功能，建议配置binlog模式为row.

**针对阿里云RDS账号默认已经有binlog dump权限,不需要任何权限或者binlog设置,可以直接跳过这一步**

修改 etc/my.cnf

``` sh
$ cat /etc/my.cnf
[mysqld]
log-bin=mysql-bin 	#添加这一行就ok
binlog-format=ROW 	#选择row模式
server_id=1      	 #配置mysql replaction需要定义，不能和canal的slaveId重复
```

## 一.配置步骤

MySQL 安装

[《CentOs7.3 安装 MySQL 5.7.19 二进制版本》](https://segmentfault.com/a/1190000010864818)


### 1.下载canal

直接下载 访问：[https://github.com/alibaba/canal/releases](https://github.com/alibaba/canal/releases),会列出所有历史的发布版本包 下载方式，比如以1.0.24版本为例子：

```sh
$ ca /opt
$ wget https://github.com/alibaba/canal/releases/download/canal-1.0.24/canal.deployer-1.0.24.tar.gz
```

or 自己编译

```sh
$ git clone git@github.com:alibaba/canal.git
$ cd canal; 
$ mvn clean install -Dmaven.test.skip -Denv=release
```
编译完成后，会在根目录下产生target/canal.deployer-$version.tar.gz

### 2.解压缩

```sh
$ mkdir /opt/canal
$ tar zxvf canal.deployer-$version.tar.gz  -C /opt/canal
```

### 3.配置修改

应用参数：
```sh
$ vi conf/example/instance.properties
```

```sh
#################################################
## mysql serverId
canal.instance.mysql.slaveId = 1234

canal.instance.master.address 需要改成自己的数据库信息

canal.instance.master.address = 127.0.0.1:3306 
canal.instance.master.journal.name =
canal.instance.master.position =
canal.instance.master.timestamp =

#canal.instance.standby.address =
#canal.instance.standby.journal.name =
#canal.instance.standby.position =
#canal.instance.standby.timestamp =

username/password，需要改成自己的数据库信息

canal.instance.dbUsername = canal

canal.instance.dbPassword = canal

canal.instance.defaultDatabaseName =
canal.instance.connectionCharset = UTF-8
```

### 4.启动

```sh
$ sh bin/startup.sh
```

### 5.查看日志

```sh
$ less logs/canal/canal.log
```
```sh
$ less logs/example/example.log
```

### 6.停止

```sh
$ sh bin/stop.sh
```

## 二.安装 Redis


**本测试项目，选择的是Redis 单机服务。集群也支持** 

### Redis 单机

[CentOs7.3 搭建 Redis-4.0.1 单机服务](https://segmentfault.com/a/1190000010709337/edit)

### Redis 集群
[CentOs7.3 搭建 Redis-4.0.1 Cluster 集群服务](https://segmentfault.com/a/1190000010682551)

## 三.同步Redis

[alibaba / canal wiki](https://github.com/alibaba/canal/wiki)

[Canal提供的 ClientExample](https://github.com/alibaba/canal/wiki/ClientExample)


### 1.创建库表

```sql

CREATE DATABASE `test`;

use `test`;

DROP TABLE IF EXISTS `test`;
CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(1000) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=16 DEFAULT CHARSET=utf8;

-- ----------------------------
-- Records of test
-- ----------------------------
INSERT INTO `test` VALUES ('1', '同步MySQL数据到 Redis');
```

### 2.导入源码

克隆，同步MySQL数据到 Redis项目[https://github.com/souyunku/ymq-example.git)

```sh
$ git clone https://github.com/souyunku/ymq-example.git
```

### 3.运行测试类


打开 `ymq-alibaba-otter-canal` 项目,运行 `SimpleCanalTest` 测试类

建立canal客户端，从canal中获取数据，并将数据更新至Redis

```java
import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import io.ymq.example.util.AbstractCanalClientTest;
import org.apache.commons.lang.exception.ExceptionUtils;

import java.net.InetSocketAddress;

/**
 * 单机模式的测试例子
 *
 * @author jianghang 2013-4-15 下午04:19:20
 * @version 1.0.4
 */
public class SimpleCanalClientTest extends AbstractCanalClientTest {

    public SimpleCanalClientTest(String destination) {
        super(destination);
    }

    public static void main(String args[]) {
        // 根据ip，直接创建链接，无HA的功能
        String destination = "example";
        // String ip = AddressUtils.getHostIp();

        CanalConnector connector = CanalConnectors.newSingleConnector(new InetSocketAddress("192.168.252.125", 11111),
                destination,
                "",
                "");

        final SimpleCanalClientTest clientTest = new SimpleCanalClientTest(destination);
        clientTest.setConnector(connector);
        clientTest.start();

        Runtime.getRuntime().addShutdownHook(new Thread() {

            public void run() {
                try {
                    logger.info("## stop the canal client");
                    clientTest.stop();
                } catch (Throwable e) {
                    logger.warn("##something goes wrong when stopping canal:\n{}", ExceptionUtils.getFullStackTrace(e));
                } finally {
                    logger.info("## canal client is down.");
                }
            }

        });
    }

```

### 4.更新数据

```sql
UPDATE `penglei`.`test` SET `id`='1', `name`='使用 Alibaba Canal 增量订阅&消费组件,同步MySQL数据到 Redis' WHERE (`id`='1');
```

### 5.查看响应
```
****************************************************
* Batch Id: [27] ,count : [3] , memsize : [325] , Time : 2017-08-29 13:57:33
* Start : [mysql-bin.000005:13948:1503986259000(2017-08-29 13:57:39)] 
* End : [mysql-bin.000005:14295:1503986259000(2017-08-29 13:57:39)] 
****************************************************

================> binlog[mysql-bin.000005:13948] , executeTime : 1503986259000 , delay : -5057ms
 BEGIN ----> Thread id: 27
----------------> binlog[mysql-bin.000005:14076] , name[penglei,test] , eventType : UPDATE , executeTime : 1503986259000 , delay : -5057ms
id : 1    type=int(11)
name : 使用 阿里巴巴 Canal 增量订阅&消费 binlog 同步 MySQL 数据到 Redis 集群    type=varchar(1000)    update=true
-------> before
id : 1    type=int(11)
name : 使用 阿里巴巴 Canal 增量订阅&消费 binlog 同步 MySQL 数据到 Redis    type=varchar(1000)
-------> after
----------------
 END ----> transaction id: 307
================> binlog[mysql-bin.000005:14295] , executeTime : 1503986259000 , delay : -5056ms
```

### 6.查看Redis

查看Redis 是否已经同步

```sh
$ /opt/redis-4.0.1/src/redis-cli -h 192.168.252.101 -c -p 6379
192.168.252.104:6379> get ymq-group:1

{
    "name": "使用 Alibaba Canal 增量订阅&消费组件,同步MySQL数据到 Redis",
    "id": "1"
}
```


[1]: http://www.ymq.io/images/2017/canal/canal.jpg
[2]: http://www.ymq.io/images/2017/canal/master-slave.jpg

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")