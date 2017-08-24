---
layout: post
title: CentOs7.3 搭建 SolrCloud 集群服务
categories: Solr
description: CentOs7.3 搭建 SolrCloud 集群服务
keywords: Solr
---

# CentOs7.3 搭建 SolrCloud 集群服务

## SolrCloud是什么？

Apache Solr包括设置一个组合容错和高可用性的Solr服务器集群的功能。称为SolrCloud，这些功能提供分布式索引和搜索功能，支持以下功能：

整个集群的中央配置

查询的自动负载平衡和故障切换

ZooKeeper集成，用于集群协调和配置。

SolrCloud是灵活的分布式搜索和索引，没有主节点分配节点，分片和副本。相反，Solr使用ZooKeeper来管理这些位置，这取决于配置文件和模式。查询和更新可以发送到任何服务器。Solr将使用ZooKeeper数据库中的信息来确定哪些服务器需要处理该请求。

[Apache SolrCloud 参考指南](http://lucene.apache.org/solr/guide/6_6/solrcloud.html)

[Apache Solr文档](https://cwiki.apache.org/confluence/display/solr/)

## 环境

VMware版本号：12.0.0  
CentOS版本：CentOS 7.3.1611  
Solr 版本：solr-6.6.0  
ZooKeeper版本：ZooKeeper-3.4.9.tar.gz 具体参考[《CentOs7.3 搭建 ZooKeeper-3.4.9 Cluster 集群服务》](https://segmentfault.com/a/1190000010807875)  
JDK环境：jdk-8u144-linux-x64.tar.gz  具体参考[《CentOs7.3 安装 JDK1.8》](https://segmentfault.com/a/1190000010716919)  


## 注意事项
 
 
关闭防火墙

```sh
$ systemctl stop firewalld.service 
```

Solr 6（和SolrJ客户端库）的Java支持的最低版本现在是Java 8。

# Solr 安装

## 下载 Solr

下载最新版本的Solr ，我在北京我就选择，清华镜像比较快 , 文件大概140M

清华镜像:[https://mirrors.tuna.tsinghua.edu.cn/apache/lucene/solr/6.6.0/](https://mirrors.tuna.tsinghua.edu.cn/apache/lucene/solr/6.6.0/)
 
阿里镜像:[https://mirrors.aliyun.com/apache/lucene/solr/6.6.0/](https://mirrors.aliyun.com/apache/lucene/solr/6.6.0/)


## 提取tar文件

```sh
$ cd /opt/
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/lucene/solr/6.6.0/solr-6.6.0.tgz
$ tar -zxf solr-6.6.0.tgz 
$ cd solr-6.6.0
```


# 集群配置

## 1.编辑 solr.in.sh

集群中的每台机器都要按照以下说明进行配置启动  
首先到 solr 安装目录的 bin 下,编辑 solr.in.sh 文件  
搜索 `SOLR_HOST`, 取消注释, 设置成自己的 ip  
搜索 `SOLR_TIMEZONE`, 取消注释, 设置成 `UTC+8`  

**把node1 的solr.in.sh 修改为一下配置**

建议设置Solr服务器的主机名，特别是在以SolrCloud模式运行时，因为它会在使用ZooKeeper注册时确定节点的地址 ，不建议用ip

```sh
SOLR_HOST="node1"
SOLR_TIMEZONE="UTC+8"
```

## 2.复制 Solr 配置

**1. 把 node1 编辑好的 Solr 文件及配置通过 `scp -r`  复制到集群 node2, node3**

```sh
$ for a in {2..3} ; do scp -r /opt/solr-6.6.0/ node$a:/opt/solr-6.6.0 ; done
```

**2. 然后修改 node2, node3 的上的 `solr.in.sh` 的`SOLR_HOST` 为机器的ip**

格式 `SOLR_HOST="ip"`

```sh
$ vi /opt/solr-6.6.0/bin/solr.in.sh
```

## 3.启动 ZooKeeper 集群

```sh
$ for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/zookeeper-3.4.9/bin/zkServer.sh start" ; done
```
 
## 4.启动 SolrCloud 集群

在任意一台机器，启动 SolrCloud 集群 并且关联 ZooKeeper 集群

```sh
$ for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/solr-6.6.0/bin/solr start -cloud -z node1:2181,node2:2181,node3:2181 -p 8983 -force" ; done
```

## 5.创建集群库

在任意一台机器

```sh
$ /opt/solr-6.6.0/bin/solr create_collection -c test_collection -shards 2 -replicationFactor 3 -force
```

`-c` 指定库(collection)名称  
`-shards` 指定分片数量,可简写为 -s ,索引数据会分布在这些分片上  
`-replicationFactor` 每个分片的副本数量,每个碎片由至少1个物理副本组成  

响应

```sh
Connecting to ZooKeeper at node3:2181 ...
INFO  - 2017-08-24 11:57:30.581; org.apache.solr.client.solrj.impl.ZkClientClusterStateProvider; Cluster at node3:2181 ready
Uploading /opt/solr-6.6.0/server/solr/configsets/data_driven_schema_configs/conf for config test_collection to ZooKeeper at node3:2181

Creating new collection 'test_collection' using command:
http://192.168.252.121:8983/solr/admin/collections?action=CREATE&name=test_collection&numShards=2&replicationFactor=3&maxShardsPerNode=2&collection.configName=test_collection

{
  "responseHeader":{
    "status":0,
    "QTime":11306},
  "success":{
    "192.168.252.123:8983_solr":{
      "responseHeader":{
        "status":0,
        "QTime":9746},
      "core":"test_collection_shard1_replica2"},
    "192.168.252.122:8983_solr":{
      "responseHeader":{
        "status":0,
        "QTime":9857},
      "core":"test_collection_shard1_replica3"},
    "192.168.252.121:8983_solr":{
      "responseHeader":{
        "status":0,
        "QTime":9899},
      "core":"test_collection_shard2_replica1"}}}

```

**SolrCloud状态 图表**

![SolrCloud状态 图表][1] 

[1]: /images/2017/Solr/solrCloud/solrCloud-Graph.png

**可以看到 solr 2个分片,个3个副本**

![可以看到 solr 2个分片,个3个副本][2]

 [2]: /images/2017/Solr/solrCloud/solrCloudShard.png
 
 
## 6.服务状态

如果您不确定SolrCloud状态

```sh
$ /opt/solr-6.6.0/bin/solr status
```

响应
```sh
Found 1 Solr nodes: 

Solr process 2926 running on port 8983
{
  "solr_home":"/opt/solr-6.6.0/server/solr",
  "version":"6.6.0 5c7a7b65d2aa7ce5ec96458315c661a18b320241 - ishan - 2017-05-30 07:32:53",
  "startTime":"2017-08-24T08:32:16.683Z",
  "uptime":"0 days, 0 hours, 34 minutes, 51 seconds",
  "memory":"63.8 MB (%13) of 490.7 MB",
  "cloud":{
    "ZooKeeper":"node1:2181,node2:2181,node3:2181",
    "liveNodes":"3",
    "collections":"1"}}

## 6.停止集群

在任意一台机器 ，停止 SolrCloud 集群 

在SolrCloud模式下停止Solr，可以使用 `-all`

```sh
$ /opt/solr-6.6.0/bin/solr stop -all
```

或者

```sh
$ for a in {1..3} ; do ssh node$a "source /etc/profile; /opt/solr-6.6.0/bin/solr stop -cloud -z node1:2181, -z node2:2181, -z node3:2181 -p 8983 -force" ; done
```


## 7.副本状态

`healthcheck` 命收集有关集合中每个副本的基本信息，例如副本数量，当前运行状态，是否正常，以及每个副本运行多长时间，内存 和地址（副本在群集中的位置）

```sh
$ /opt/solr-6.6.0/bin/solr healthcheck -c test_collection -z node1:2181,node2:2181,node3:2181 -p 8983 -force
```

响应

```sh
INFO  - 2017-08-24 16:34:26.906; org.apache.solr.client.solrj.impl.ZkClientClusterStateProvider; Cluster at node1:2181,node2:2181,node3:2181 ready
{
  "collection":"test_collection",
  "status":"healthy",
  "numDocs":0,
  "numShards":2,
  "shards":[
    {
      "shard":"shard1",
      "status":"healthy",
      "replicas":[
        {
          "name":"core_node3",
          "url":"http://node1:8983/solr/test_collection_shard1_replica1/",
          "numDocs":0,
          "status":"active",
          "uptime":"0 days, 0 hours, 2 minutes, 10 seconds",
          "memory":"58.6 MB (%12) of 490.7 MB",
          "leader":true},
        {
          "name":"core_node5",
          "url":"http://node2:8983/solr/test_collection_shard1_replica3/",
          "numDocs":0,
          "status":"active",
          "uptime":"0 days, 0 hours, 1 minutes, 58 seconds",
          "memory":"50.2 MB (%10.2) of 490.7 MB"},
        {
          "name":"core_node6",
          "url":"http://node3:8983/solr/test_collection_shard1_replica2/",
          "numDocs":0,
          "status":"active",
          "uptime":"0 days, 0 hours, 1 minutes, 46 seconds",
          "memory":"56.3 MB (%11.5) of 490.7 MB"}]},
    {
      "shard":"shard2",
      "status":"healthy",
      "replicas":[
        {
          "name":"core_node1",
          "url":"http://node1:8983/solr/test_collection_shard2_replica1/",
          "numDocs":0,
          "status":"active",
          "uptime":"0 days, 0 hours, 2 minutes, 10 seconds",
          "memory":"58.6 MB (%12) of 490.7 MB",
          "leader":true},
        {
          "name":"core_node2",
          "url":"http://node3:8983/solr/test_collection_shard2_replica2/",
          "numDocs":0,
          "status":"active",
          "uptime":"0 days, 0 hours, 1 minutes, 46 seconds",
          "memory":"58.8 MB (%12) of 490.7 MB"},
        {
          "name":"core_node4",
          "url":"http://node2:8983/solr/test_collection_shard2_replica3/",
          "numDocs":0,
          "status":"active",
          "uptime":"0 days, 0 hours, 1 minutes, 58 seconds",
          "memory":"51.9 MB (%10.6) of 490.7 MB"}]}]}

```

# Solr 文档

[Apache SolrCloud 参考指南](http://lucene.apache.org/solr/guide/6_6/solrcloud.html)  
[Apache Solr文档](https://cwiki.apache.org/confluence/display/solr/)  
[Solr 参数配置](https://cwiki.apache.org/confluence/display/solr/Format+of+solr.xml)  


