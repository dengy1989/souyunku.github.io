---
layout: post
title: ELK 集群 + Redis + Nginx 日志分析平台(未完待续)
categories: ElasticSearch Logstash Kinaba 
description: ELK集群搭建 ElasticSearch Logstash Kinaba 
keywords: ElasticSearch Logstash Kinaba 
---

# ELK集群搭建

## 简述

ELK实际上是**三个工具**的集合，**ElasticSearch** + **Logstash** + **Kibana**，这三个工具组合形成了一套实用、易用的监控架构，很多公司利用它来搭建可视化的海量日志分析平台。  

[官网下载地址：https://www.elastic.co/downloads](https://www.elastic.co/downloads)

**Elasticsearch**

  是一个基于Apache Lucene(TM)的开源搜索引擎 ，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，RESTful web风格接口，多数据源，自动搜索负载等。

**Logstash**

  用于管理日志和事件的工具，你可以用它去收集日志、转换日志、解析日志并将他们作为数据提供给其它模块调用，例如搜索、存储等。  

**Kibana**

  是一个开源和免费的工具，它`Kibana`可以为 `Logstash` 和 `ElasticSearch` 提供的日志分析友好的 Web 界面，可以帮助您汇总、分析和搜索重要数据日志。

  
## 应用场景

在传统的应用运行环境中，收集、分析日志往往是非常必要且常见的，一旦应用出现问题、或者需要分析应用行为时，管理员将通过一些传统工具对日志进行检查，如cat、tail、sed、awk、perl以及grep。这种做法有助于培养在常用工具方面的优秀技能，但它的适用范围仅限于少量的主机和日志文件类型。


随着应用系统的复杂性与日俱增，日志的分析也越来越重要，常见的需求包括，团队开发过程中可能遇到一些和日志有关的问题：

 - 开发没有生产环境服务器权限，需要通过系统管理员获取详细日志，沟通成本高

 - 系统可能是有多个不同语言编写、分布式环境下的模块组成，查找日志费时费力

 - 日志数量巨大，查询时间很长
 
# 准备工作

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
node4 --> nginx: 192.168.252.125
  
node5 --> Kibana: 192.168.252.125

node6 --> Redis: 192.168.252.126
```

**修改主机名**

[CentOs7.3 修改主机名](https://segmentfault.com/a/1190000010723105)


## 安装依赖

本次搭建，ELK 集群 + Redis + Nginx 日志分析平台，需要安装以下软件


**安装 JDK**

[CentOs7.3 安装 JDK1.8](https://segmentfault.com/a/1190000010716919)

**安装 Nginx**

[CentOs7.3 编译安装 Nginx 1.9.9](https://segmentfault.com/a/1190000010721915)

**安装 Redis**

任选一个搭建

[CentOs7.3 搭建 Redis-4.0.1 单机服务](https://segmentfault.com/a/1190000010709337/edit)

[CentOs7.3 搭建 Redis-4.0.1 Cluster 集群服务](https://segmentfault.com/a/1190000010682551)

**关闭防火墙** 

节点之前需要开放指定端口，为了方便，生产不要禁用

```sh
systemctl stop firewalld.service
```

# Elasticsearch

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

[ElasticSearch 安装报错整理](http://127.0.0.1/2017/08/30/ElasticSearch-install-error/)


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

[《不推荐》如果你一台机器，已经安装好，可以参考我这个复制集群配置操作](http://127.0.0.1/2017/08/30/ElasticSearch-install-error/#elasticsearch-%E5%A4%8D%E5%88%B6%E9%9B%86%E7%BE%A4%E9%85%8D%E7%BD%AE)

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

```
discovery.zen.ping.unicast.hosts: ["192.168.252.121","192.168.252.122","192.168.252.123"]
```

[为了防止数据丢失](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/important-settings.html#minimum_master_nodes)

```
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

### ES 插架

**概要**

Chrome扩展程序包含优秀的`ElasticSearch Head`应用程序。  

**动机**

这是因为ElasticSearch 5 删除了将ElasticSearch Head作为弹性插件运行的功能而创建的。这为您自己的Web服务器提供了自主托管的替代方案。它还具有绕过CORS而不在您的Elastic服务器上配置CORS的优点。  

**使用**

单击Web浏览器工具栏中的扩展名图标。  
键入弹性节点的地址到打开的新选项卡的顶部。  
单击连接按钮。  

**安装**

在Chrome扩展程序里，搜索ElasticSearch Head 点击安装

![][5] 

# Logstash

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
		type => "nginx access log"
		path => "ccess.log"
    }
}
output {
        redis {
                host => "192.168.252.125"
                data_type => "list"
                port => "6379"
                key => "logstash:redis"
        }
}
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
                host => "192.168.252.125"
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

`启动 logstash agent` `logstash` 代理收集日志发送 `Redis`

```sh
nohup /opt/logstash-5.5.2/bin/logstash -f /opt/logstash-5.5.2/etc/logstash_agent_nginx.conf  --path.data=/opt/logstash-5.5.2/logs/log1   > /dev/null 2>&1 &
```

`启动 logstash indexer` `logstash`  读 `Redis`  日志发送到`Elasticsearch`

```sh
nohup /opt/logstash-5.5.2/bin/logstash -f /opt/logstash-5.5.2/etc/logstash_indexer.conf --path.data=/opt/logstash-5.5.2/logs/log2  > /dev/null 2>&1 &
```

**查看日志**

```sh
less /opt/logstash-5.5.2/logs/logstash-plain.log
```

# Kibana

[Installing Kibana](https://www.elastic.co/guide/en/kibana/current/targz.html)

## 下载解压
```
cd /opt
wget https://artifacts.elastic.co/downloads/kibana/kibana-5.5.2-linux-x86_64.tar.gz
tar -xzf kibana-5.5.2-linux-x86_64.tar.gz
```

## 编辑配置

```sh
cd kibana-5.5.2-linux-x86_64 
vi config/kibana.yml
```

```sh
server.port: 5601
server.host: "0.0.0.0"
elasticsearch.url: "http://192.168.252.121:9200"
```

## 启动服务

```sh
cd /opt/kibana-5.5.2-linux-x86_64/bin/

./kibana
```


**访问 `Kibana`**


访问 `Kibana` 地址 [http://192.168.252.125:5601/ ](http://192.168.252.125:5601/ )

![][1] 

[1]: /images/2017/ELK/kibana/kibana-0.png  

# 查看收集日志

## 刷新nginx

![][2] 



![][3] 

[3]: /images/2017/ELK/kibana/kibana-1.png  
[2]: /images/2017/ELK/kibana/kibana.png  

[5]: /images/2017/ELK/ElasticSearch/ElasticSearch-Head.png



http://192.168.252.121:9200/_search?pretty
