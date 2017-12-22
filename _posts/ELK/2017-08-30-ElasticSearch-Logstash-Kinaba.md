---
layout: post
title: ELK 集群 + X-Pack + Redis 集群 + Nginx ,实时日志（数据）搜集和分析的监控系统，简单上手使用
categories: ElasticSearch Logstash Kinaba 
description: ELK集群搭建 ElasticSearch Logstash Kinaba 
keywords: ElasticSearch Logstash Kinaba 
---

# ELK 集群 + X-Pack + Redis 集群 + Nginx ,实时日志（数据）搜集和分析的监控系统，简单上手使用

## 简述

**ELK**实际上是**三个工具**的集合，**ElasticSearch** + **Logstash** + **Kibana**，这三个工具组合形成了一套实用、易用的监控架构，很多公司利用它来搭建可视化的海量日志分析平台。  

[官网下载地址：https://www.elastic.co/downloads](https://www.elastic.co/downloads)

**Elasticsearch**

是一个基于Apache Lucene(TM)的开源搜索引擎 ，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，`RESTful` `web`风格接口，多数据源，自动搜索负载等。

**Logstash**

是一个用来搜集、分析、过滤日志的工具。它支持几乎任何类型的日志，包括系统日志、错误日志和自定义应用程序日志。它可以从许多来源接收日志，这些来源包括 `syslog`、消息传递（例如 RabbitMQ）和JMX，它能够以多种方式输出数据，包括电子邮件、`websockets`和 `Elasticsearch`。

**Kibana**

是一个基于`Web`的图形界面，用于搜索、分析和可视化存储在 `Elasticsearch`指标中的日志数据。它利用`Elasticsearch`的`REST`接口来检索数据，不仅允许用户创建他们自己的数据的定制仪表板视图，还允许他们以特殊的方式查询和过滤数据。
	

**X-Pack**
	
`X-Pack`是一个`Elastic Stack`的扩展，将安全，警报，监视，报告和图形功能包含在一个易于安装的软件包中

**Redis**

**Redis优势性能极高** – `Redis`能读的速度是`110000次/s`,写的速度是`81000`次/s 。
**丰富的数据类型** – `Redis`支持二进制案例的 `Strings, Lists, Hashes, Sets` 及 `Ordered Sets` 数据类型操作。
**原子** – `Redis`的所有操作都是原子性的，同时`Redis`还支持对几个操作全并后的原子性执行。
**丰富的特性** – `Redis`还支持 `publish/subscribe`, 通知, `key` 过期等等特性。

  
**Nginx**

是一个高性能的 Web 和反向代理服务器, 它具有有很多非常优越的特性:

`作为 Web 服务器`：相比 Apache，Nginx 使用更少的资源，支持更多的并发连接，体现更高的效率，这点使 Nginx 尤其受到虚拟主机提供商的欢迎。能够支持高达 50,000 个并发连接数的响应，感谢 Nginx 为我们选择了 epoll and kqueue 作为开发模型.

`作为负载均衡服务器`：Nginx 既可以在内部直接支持 Rails 和 PHP，也可以支持作为 HTTP代理服务器 对外进行服务。Nginx 用 C 编写, 不论是系统资源开销还是 CPU 使用效率都比 Perlbal 要好的多。


## 应用场景

在传统的应用运行环境中，收集、分析日志往往是非常必要且常见的，一旦应用出现问题、或者需要分析应用行为时，管理员将通过一些传统工具对日志进行检查，如cat、tail、sed、awk、perl以及grep。这种做法有助于培养在常用工具方面的优秀技能，但它的适用范围仅限于少量的主机和日志文件类型。


随着应用系统的复杂性与日俱增，日志的分析也越来越重要，常见的需求包括，团队开发过程中可能遇到一些和日志有关的问题：

 - 开发没有生产环境服务器权限，需要通过系统管理员获取详细日志，沟通成本高

 - 系统可能是有多个不同语言编写、分布式环境下的模块组成，查找日志费时费力

 - 日志数量巨大，查询时间很长
 
# 1.准备工作

## 环境

ElasticSearch : elasticsearch-5.5.2  
Logstash : logstash-5.5.2  
kibana：kibana-5.5.2  
JDK: 1.8  
Redis: Redis-4.0.1  
Nginx: 1.9.9  

```sh
node1 --> ElasticSearch: 192.168.252.121:9200  
node2 --> ElasticSearch: 192.168.252.122:9200  
node3 --> ElasticSearch: 192.168.252.123:9200  
  
node4 --> Logstash: 192.168.252.124
node4 --> nginx: 192.168.252.124
  
node5 --> Kibana: 192.168.252.125

node6 --> Redis: 192.168.252.126
node7 --> Redis: 192.168.252.127
node8 --> Redis: 192.168.252.128
```




**修改主机名**

[CentOs7.3 修改主机名](https://segmentfault.com/a/1190000010723105)


## 安装依赖

本次搭建，ELK 集群 + Redis + Nginx 日志分析平台，需要安装以下软件


**安装 JDK**

[CentOs7.3 安装 JDK1.8](https://segmentfault.com/a/1190000010716919)

**安装 Nginx**

[CentOs7.3 编译安装 Nginx 1.9.9](https://segmentfault.com/a/1190000010721915)

**安装 Redis集群**

[CentOs7.3 搭建 Redis-4.0.1 Cluster 集群服务](https://segmentfault.com/a/1190000010682551)

**关闭防火墙** 

节点之前需要开放指定端口，为了方便，生产不要禁用

```sh
systemctl stop firewalld.service
```

# 2.Elasticsearch

## ES单机安装

[Installing Elasticsearch](https://www.elastic.co/guide/en/elastic-stack/current/installing-elastic-stack.html)

### 下载解压

Elasticsearch 下载地址：[https://www.elastic.co/downloads/elasticsearch](https://www.elastic.co/downloads/elasticsearch)

```sh
cd /opt
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.2.tar.gz
tar -xzf elasticsearch-5.5.2.tar.gz
```

### 编辑配置

只要集群名相同ps(`cluster.name:`),且机器处于同一局域网同一网段,`Elasticsearch` 会自动去发现其他的节点

```sh
vi /opt/elasticsearch-5.5.2/config/elasticsearch.yml
```

```sh
cluster.name: ymq
node.name: ELK-node1
network.host: 0.0.0.0
http.port: 9200
```

通过jvm.options 编辑设置JVM堆大小

```sh
vi /opt/elasticsearch-5.5.2/config/jvm.options

-Xms2g  --》修改为512m
-Xmx2g  --》修改为512m
```

### 启动服务

```sh
/opt/elasticsearch-5.5.2/bin/elasticsearch -d
```

[ElasticSearch 安装报错整理](http://www.ymq.io/2017/08/30/ElasticSearch-install-error/)


**查看日志**  

日志名称`ymq`就是 `cluster.name: ymq` 的名称

```sh
less /opt/elasticsearch-5.5.2/logs/ymq.log
```

**查看端口**

```sh
netstat -nltp
```
```sh
tcp6       0      0 :::9200                 :::*                    LISTEN      2944/java           
tcp6       0      0 :::9300                 :::*                    LISTEN      2944/java         
```

**测试访问**

```sh
curl -X GET http://localhost:9200/
```

```sh
{
  "name" : "ELK-node1",
  "cluster_name" : "ymq",
  "cluster_uuid" : "jxWzvSFNTCWtToD6wrVIpA",
  "version" : {
    "number" : "5.5.2",
    "build_hash" : "b2f0c09",
    "build_date" : "2017-08-14T12:33:14.154Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
```

**健康状态**

```sh
curl -X GET http://192.168.252.123:9200/_cat/health?v 
```

```sh
epoch      timestamp cluster status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1504102874 22:21:14  ymq     green           1         1      0   0    0    0        0             0                  -                100.0%
```

**关闭服务**

直接杀掉进程

```sh
jps

2968 Elasticsearch
3135 Jps
```

```
kill -9 2968
```

## ES集群安装


依次安装nide2,node3,集群

[《不推荐》如果你一台机器，已经安装好，可以参考我这个复制集群配置操作](http://www.ymq.io/2017/08/30/ElasticSearch-install-error/)

### 编辑配置

[集群配置](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/important-settings.html)

集群名必须相同ps(`cluster.name:`),且机器处于同一局域网同一网段,`Elasticsearch` 会自动去发现其他的节点

```sh
vi /opt/elasticsearch-5.5.2/config/elasticsearch.yml
```

`node.name` 换个名字，可以`ELK-node2`,`ELK-node3`

```sh
node.name: ELK-node2
```

```sh
node.name: ELK-node3
```

配置集群时，必须设置集群中与其他的节点通信的列表，如果没有指定`端口`，该端口将默认为9300

```sh
discovery.zen.ping.unicast.hosts: ["192.168.252.121","192.168.252.122","192.168.252.123"]
```

[为了防止数据丢失](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/important-settings.html#minimum_master_nodes)

```sh
discovery.zen.minimum_master_nodes: 2
```

`discovery.zen.minimum_master_nodes`（默认是1）：这个参数控制的是，一个节点需要看到的具有master节点资格的最小数量，然后才能在集群中做操作。官方的推荐值是(N/2)+1，其中N是具有master资格的节点的数量（我们的情况是3，因此这个参数设置为2，但对于只有2个节点的情况，设置为2就有些问题了，一个节点DOWN掉后，你肯定连不上2台服务器了，这点需要注意）。

### 启动服务

依次启动各个集群

```sh
/opt/elasticsearch-5.5.2/bin/elasticsearch -d
```

[ElasticSearch 安装报错整理](http://127.0.0.1/2017/08/30/ElasticSearch-install-error/)


### 集群操作

[集群操作 API ](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster.html)

**节点列表**
```sh
curl 'localhost:9200/_cat/nodes?v'
```
```sh
ip              heap.percent ram.percent cpu load_1m load_5m load_15m node.role master name
192.168.252.121           11          92   0    0.00    0.04     0.05 mdi       *      ELK-node1
192.168.252.122           16          94   0    0.00    0.04     0.09 mdi       -      ELK-node2
192.168.252.123           13          92   0    0.02    0.05     0.07 mdi       -      ELK-node3
```

**集群健康**

[集群健康](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html)

```sh
curl -XGET 'http://localhost:9200/_cluster/health?pretty' 
```

```sh
{
  "cluster_name" : "ymq",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 3,
  "number_of_data_nodes" : 3,
  "active_primary_shards" : 0,
  "active_shards" : 0,
  "relocating_shards" : 0,
  "initializing_shards" : 0,
  "unassigned_shards" : 0,
  "delayed_unassigned_shards" : 0,
  "number_of_pending_tasks" : 0,
  "number_of_in_flight_fetch" : 0,
  "task_max_waiting_in_queue_millis" : 0,
  "active_shards_percent_as_number" : 100.0
}
```

**群集状态**

获得整个集群的综合状态信息。

```sh
curl -XGET 'http://localhost:9200/_cluster/state?pretty' 
```

```sh
{
  "cluster_name" : "ymq",
  "version" : 3,
  "state_uuid" : "uWsrNkiFTOSItF5LFZTl1A",
  "master_node" : "118aXxRfQIaYQ9Y-mDIfSg",
  "blocks" : { },
  "nodes" : {
    "118aXxRfQIaYQ9Y-mDIfSg" : {
      "name" : "ELK-node1",
      "ephemeral_id" : "9pT8VWO5SjyRCYMe6qUdgA",
      "transport_address" : "192.168.252.121:9300",
      "attributes" : { }
    },
    "R2aiceefQ9aoSqY0_SXZew" : {
      "name" : "ELK-node2",
      "ephemeral_id" : "1tAgHHrORP65KmHwAkKYaw",
      "transport_address" : "192.168.252.122:9300",
      "attributes" : { }
    },
    "P9XQhjaYQrSWpke8rzrgeg" : {
      "name" : "ELK-node3",
      "ephemeral_id" : "LSjPP11PSx-7YaUw3a-CSw",
      "transport_address" : "192.168.252.123:9300",
      "attributes" : { }
    }
  },
  "metadata" : {
    "cluster_uuid" : "-aqZbTMNTHGzPIm_KoLiiw",
    "templates" : { },
    "indices" : { },
    "index-graveyard" : {
      "tombstones" : [ ]
    }
  },
  "routing_table" : {
    "indices" : { }
  },
  "routing_nodes" : {
    "unassigned" : [ ],
    "nodes" : {
      "P9XQhjaYQrSWpke8rzrgeg" : [ ],
      "118aXxRfQIaYQ9Y-mDIfSg" : [ ],
      "R2aiceefQ9aoSqY0_SXZew" : [ ]
    }
  }
}
```

## ES 插架

**概要**

Chrome扩展程序包含优秀的`ElasticSearch Head`应用程序。  

**动机**

这是因为ElasticSearch 5 删除了将ElasticSearch Head作为弹性插件运行的功能而创建的。这为您自己的Web服务器提供了自主托管的替代方案。它还具有绕过CORS而不在您的Elastic服务器上配置CORS的优点。  

**使用**

单击Web浏览器工具栏中的扩展名图标。  
键入弹性节点的地址到打开的新选项卡的顶部。  
单击连接按钮。  

**安装**

需要翻墙安装  

在Chrome扩展程序里，搜索ElasticSearch Head 点击安装

![][1] 

# 3.Logstash

[Installing Logstash](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html)

## 下载解压

```sh
cd /opt
wget https://artifacts.elastic.co/downloads/logstash/logstash-5.5.2.tar.gz
tar -xzf logstash-5.5.2.tar.gz
```

**测试一下**

测试　logstash　是否正常运行


```sh
/opt/logstash-5.5.2/bin/logstash -e 'input{stdin{}}output{stdout{codec=>rubydebug}}'
```

敲入 Hello World，回车　　　

结果　　

```sh
Hello World
{
    "@timestamp" => 2017-08-30T17:24:25.553Z,
      "@version" => "1",
          "host" => "node4",
       "message" => "Hello World"
}
```
## 编辑配置

[Logstash配置示例](https://www.elastic.co/guide/en/logstash/current/config-examples.html)

### 开启 logstash agent

编辑  Nginx 配置，解开默认的 Nginx `#`注释

```sh
vi /usr/local/nginx/conf/nginx.conf
```

在nginx.conf 中设置日志格式：logstash

```sh
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
```

负责收集nginx访问日志信息传送到redis队列上

```sh
cd /opt/logstash-5.5.2
mkdir etc
vi /opt/logstash-5.5.2/etc/logstash_agent_nginx.conf
```


```sh
input {
    file {
                type => "nginx_access_log"
                path => "/usr/local/nginx/logs/access.log"
    }
}
output {
        redis {
                host => "192.168.252.126"
                data_type => "list"
                port => "6379"
                key => "logstash:redis"
        }
}
```

多日志文件例如

```sh
path => [ "/usr/local/nginx/logs/*.log", "/var/log/messages.log", "/var/log/syslog.log" ]
```

**`input {}`解释**

读取nginx访问日志`access.log`，Logstash 使用一个名叫 FileWatch 的 Ruby Gem 库来监听文件变化　FileWatch 插件只支持文件的绝对路径，而且会不自动递归目录。所以有需要的话，请用数组方式都写明具体哪些文件。

**`output {}`解释**

发送,Logstash 收集的 Nginx 访问日志信息传送到 redis 服务器上

### 开启 logstash indexer

```sh
cd /opt/logstash-5.5.2
mkdir etc
vi /opt/logstash-5.5.2/etc/logstash_indexer.conf 
```

```sh
input {
        redis {
                host => "192.168.252.126"
                data_type => "list"
                port => "6379"
                key => "logstash:redis"
                type => "redis-input"
        }
}

output {
    elasticsearch {
        hosts => ["192.168.252.121:9200"]
        index => "logstash-%{type}-%{+YYYY.MM.dd}"
        document_type => "%{type}"
        flush_size => 20000
        idle_flush_time => 10
        sniffing => true
        template_overwrite => true
    }
}
```

**`input {}`解释**

读取`Redis` key `logstash:redis`  的数据

**`filter {}`解释**

[过滤器 example ](https://www.elastic.co/guide/en/logstash/current/config-examples.html#filter-example)  

过滤器是一种在线处理机制，可提供切割和切割数据以满足您的需求的灵活性。我们来看看一些有效的过滤器。以下配置文件设置grok和date过滤器。

**`output {}`解释**

批量发送Elasticsearch，本插件的 flush_size 和 idle_flush_time 两个参数共同控制 Logstash 向 Elasticsearch 发送批量数据的行为。以上面示例来说：Logstash 会努力攒到 20000 条数据一次性发送出去，但是如果 10 秒钟内也没攒够 20000 条，Logstash 还是会以当前攒到的数据量发一次。 默认情况下，flush_size 是 500 条，idle_flush_time 是 1 秒。这也是很多人改大了 flush_size 也没能提高写入 ES 性能的原因——Logstash 还是 1 秒钟发送一次。

### 启动服务

```
cd /opt/logstash-5.5.2/logs/
mkdir log1 log2
```

启动 `logstash agent` `logstash` 代理收集日志发送 `Redis`

```sh
nohup /opt/logstash-5.5.2/bin/logstash -f /opt/logstash-5.5.2/etc/logstash_agent_nginx.conf  --path.data=/opt/logstash-5.5.2/logs/log1   > /dev/null 2>&1 &
```

启动 `logstash indexer` `logstash`  读 `Redis`  日志发送到`Elasticsearch`

```sh
nohup /opt/logstash-5.5.2/bin/logstash -f /opt/logstash-5.5.2/etc/logstash_indexer.conf --path.data=/opt/logstash-5.5.2/logs/log2  > /dev/null 2>&1 &
```

**查看日志**

```sh
less /opt/logstash-5.5.2/logs/logstash-plain.log
```

# 4.Kibana

[Installing Kibana](https://www.elastic.co/guide/en/kibana/current/targz.html)

## 下载解压
```
cd /opt
wget https://artifacts.elastic.co/downloads/kibana/kibana-5.5.2-linux-x86_64.tar.gz
tar -xzf kibana-5.5.2-linux-x86_64.tar.gz
```

## 编辑配置

```sh
cd /opt/kibana-5.5.2-linux-x86_64
vi config/kibana.yml
```

```sh
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.url: "http://192.168.252.121:9200"
```

## 启动服务

```sh
/opt/kibana-5.5.2-linux-x86_64/bin/kibana
```

**访问 `Kibana`**


访问 `Kibana` 地址 [http://192.168.252.125:5601/ ](http://192.168.252.125:5601/ )

# 5.测试日志分析平台

**如果之前启动过就`kill`掉**

## 启动以下服务

**启动 `ES` 集群**

切换到，新建用户 `ymq`，依次启动各个 elasticsearch 集群
```sh
su ymq
/opt/elasticsearch-5.5.2/bin/elasticsearch -d
```

**启动 `kibana`**

```sh
/opt/kibana-5.5.2-linux-x86_64/bin/kibana
```

**启动 `Redis`**

```sh
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis-sentinel/redis.conf
```

**启动 `Nginx`**

```sh
/usr/local/nginx/sbin/nginx
```

## 测试步骤

### 1.检查各服务

检查各服务，**是否启动成功** ，**是否日志异常** 

### 2.启动`logstash agent`

启动 `logstash agent` `logstash` 代理收集日志发送 `Redis`

```sh
nohup /opt/logstash-5.5.2/bin/logstash -f /opt/logstash-5.5.2/etc/logstash_agent_nginx.conf  --path.data=/opt/logstash-5.5.2/logs/log1   > /dev/null 2>&1 &
```
### 3.生成测试数据

在浏览器 `Nginx` 页面 **刷新 2 次**，生成测试数据
![][2] 

### 4.查看测试数据

打开`RedisDesktopManager` 页面看看Nginx 刷新页面的访问成功日志，看到 **写进 Redis 2 条数据** 

**ps 这只是个可视化Redis界面而已，你也可以使用命令行查看**

![][3] 

如果没有 `RedisDesktopManager`,就在安装Redis 服务器上 通过命令行查看  

```sh
[root@node6 opt]# /opt/redis-4.0.1/src/redis-cli -h 192.168.252.126 -c -p 6379
192.168.252.126:6379> LRANGE logstash:redis 0 9
```

[Redis 相关](https://segmentfault.com/a/1190000010721915)

### 5.使用`kibana`

在浏览器打开`kibana Dev Tools` [http://192.168.252.125:5601](http://192.168.252.125:5601)

可以看到`ElasticSearch` 是没有 `Nginx` 刷新页面的访问成功日志数据的，是因为，我没有启动 `logstash indexer` 读取`Redis`数据写入`ElasticSearch` 的服务

![][4] 

### 6.启动`logstash indexer`

启动 `logstash indexer` `logstash`  读 `Redis`  日志发送到`Elasticsearch`

```sh
nohup /opt/logstash-5.5.2/bin/logstash -f /opt/logstash-5.5.2/etc/logstash_indexer.conf --path.data=/opt/logstash-5.5.2/logs/log2  > /dev/null 2>&1 &
```

先看`Redis`，已经是空的了

```sh
192.168.252.126:6379> LRANGE logstash:redis 0 9
(empty list or set)
```
是不是已经写到 `ElasticSearch` 呢? 往下看


### 7.使用`kibana DevTools`

我们看看，`DevTools `，开发工具数据查询，相比上次已经看到多了 2 条，就是我们访问 Nginx` 刷新页面的访问成功日志数据

![][5] 

### 8.使用`ElasticSearchHead`

**通过 ElasticSearchHead 插件 查看数据**

![][6] 

### 9.使用`kibana Discover`

首先使用`kibana Discover`配置索引 `logstash-*` 

ps ( 索引名称是按照 `elasticsearch` 写入时创建的索引  `index => "logstash-%{type}-%{+YYYY.MM.dd}"` 这个配置在上面**开启 logstash indexer**  有提到

![][7] 


### 10.使用`kibana Search`

点击`Available Fields` 下选择咱们日志中的字段 `key`  `add` 可添加更多显示字段，我为了博客显示效果，没有让显示全部，默认显示全部字段，也可由删除显示字段

![][8] 

### 11.使用`kibana Search 条件查询`

点击`Discover 工具栏`   
下选择 指定时间 `2017-09-01 23:45:00.000 ----2017-09-01 23:50:00.000`的日志，  
搜索关键词`AppleWebKit`  
图形表示 Hourly 每小时  

![][9] 

### 12.使用`kibana Timelion`

**最后6分钟的时间轴**
 
 ![][10] 

[1]: http://www.ymq.io/images/2017/ELK/Elasticsearch-head.png
[2]: http://www.ymq.io/images/2017/ELK/nginx.png
[3]: http://www.ymq.io/images/2017/ELK/redis.png
[4]: http://www.ymq.io/images/2017/ELK/kibana-tool.png
[5]: http://www.ymq.io/images/2017/ELK/kibana-tool-run.png
[6]: http://www.ymq.io/images/2017/ELK/Elasticsearch-head-data.png
[7]: http://www.ymq.io/images/2017/ELK/kibaba-creata_index.png
[8]: http://www.ymq.io/images/2017/ELK/kibana-search.png
[9]: http://www.ymq.io/images/2017/ELK/kibana-date-to-date.png
[10]: http://www.ymq.io/images/2017/ELK/kibana-last6m.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")
