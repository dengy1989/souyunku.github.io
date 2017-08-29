---
layout: post
title: ELK集群搭建 ElasticSearch Logstash Kinaba 
categories: ElasticSearch Logstash Kinaba 
description: ELK集群搭建 ElasticSearch Logstash Kinaba 
keywords: ElasticSearch Logstash Kinaba 
---

# ELK集群搭建


## 应用场景

在传统的应用运行环境中，收集、分析日志往往是非常必要且常见的，一旦应用出现问题、或者需要分析应用行为时，管理员将通过一些传统工具对日志进行检查，如cat、tail、sed、awk、perl以及grep。这种做法有助于培养在常用工具方面的优秀技能，但它的适用范围仅限于少量的主机和日志文件类型。


随着应用系统的复杂性与日俱增，日志的分析也越来越重要，常见的需求包括，团队开发过程中可能遇到一些和日志有关的问题：

 - 开发没有生产环境服务器权限，需要通过系统管理员获取详细日志，沟通成本高

 - 系统可能是有多个不同语言编写、分布式环境下的模块组成，查找日志费时费力

 - 日志数量巨大，查询时间很长


**ELK实际上是三个工具的集合**，`ElasticSearch + Logstash + Kibana`，这三个工具组合形成了一套实用、易用的监控架构，很多公司利用它来搭建可视化的海量日志分析平台。  

[官网下载地址：https://www.elastic.co/downloads](https://www.elastic.co/downloads)


**Elasticsearch**

  是一个基于Apache Lucene(TM)的开源搜索引擎 ，它的特点有：分布式，零配置，自动发现，索引自动分片，索引副本机制，RESTful web风格接口，多数据源，自动搜索负载等。

**Logstash**

  用于管理日志和事件的工具，你可以用它去收集日志、转换日志、解析日志并将他们作为数据提供给其它模块调用，例如搜索、存储等。  

**Kibana**

  是一个开源和免费的工具，它`Kiban`a可以为 `Logstash` 和 `ElasticSearch` 提供的日志分析友好的 Web 界面，可以帮助您汇总、分析和搜索重要数据日志。

[参考-elastic 官方安装教程](https://www.elastic.co/guide/en/elastic-stack/current/installing-elastic-stack.html)

# 安装 Elasticsearch

Elasticsearch 下载地址：[https://www.elastic.co/downloads/elasticsearch](https://www.elastic.co/downloads/elasticsearch)

下载，解压
```sh
cd /opt
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.5.2.tar.gz
tar -xzf elasticsearch-5.5.2.tar.gz
```


编辑配置
```sh
vi /opt/elasticsearch-5.5.2/config/elasticsearch.yml
```

只要集群名相同，且机器处于同一局域网同一网段，es会自动去发现其他的节点

```sh
cluster.name: ymq
node.name: ELK-node1
network.host: 0.0.0.0
http.port: 9200
```

启动Elasticsearch 由于不能以root 用户启动，那么我们就先建个ymq 用户

```sh
adduser ymq
passwd ymq
chmod 777 /etc/sudoers
vi /etc/sudoers
```

```sh
root    ALL=(ALL)       ALL
ymq     ALL=(ALL)       NOPASSWD:ALL  #` NOPASSWD:ALL ` 不用输密码
```

授权，并启动
```sh
chown -R ymq:ymq /opt/elasticsearch-5.5.2
su ymq 
/opt/elasticsearch-5.5.2/bin/elasticsearch
```

```
[2017-08-30T02:26:37,700][INFO ][o.e.n.Node               ] [ELK-node1] initialized
[2017-08-30T02:26:37,700][INFO ][o.e.n.Node               ] [ELK-node1] starting ...
[2017-08-30T02:26:37,886][INFO ][o.e.t.TransportService   ] [ELK-node1] publish_address {192.168.252.122:9300}, bound_addresses {[::]:9300}
[2017-08-30T02:26:37,909][INFO ][o.e.b.BootstrapChecks    ] [ELK-node1] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
ERROR: [2] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]

[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[2017-08-30T02:26:37,925][INFO ][o.e.n.Node               ] [ELK-node1] stopping ...
[2017-08-30T02:26:37,945][INFO ][o.e.n.Node               ] [ELK-node1] stopped
[2017-08-30T02:26:37,945][INFO ][o.e.n.Node               ] [ELK-node1] closing ...
[2017-08-30T02:26:37,964][INFO ][o.e.n.Node               ] [ELK-node1] closed
```

[参考](http://www.cnblogs.com/sloveling/p/elasticsearch.html)