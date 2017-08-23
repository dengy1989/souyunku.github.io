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

## 环境

VMware版本号：12.0.0

CentOS版本：CentOS 7.3.1611

Solr 版本：solr-6.6.0

ZooKeeper版本：ZooKeeper-3.4.9.tar.gz

JDK环境：jdk-8u144-linux-x64.tar.gz 


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


# JDK 1.8 安装

具体参考[《CentOs7.3 安装 JDK1.8》](https://segmentfault.com/a/1190000010716919)


# ZooKeeper 安装

具体参考[《CentOs7.3 搭建 ZooKeeper-3.4.9 Cluster 集群服务》](https://segmentfault.com/a/1190000010807875)



# Solr 安装

## 1.下载 Solr

下载最新版本的Solr ，我在北京我就选择，清华镜像比较快 , 文件大概140M

清华镜像:[https://mirrors.tuna.tsinghua.edu.cn/apache/lucene/solr/6.6.0/](https://mirrors.tuna.tsinghua.edu.cn/apache/lucene/solr/6.6.0/)
 
阿里镜像:[https://mirrors.aliyun.com/apache/lucene/solr/6.6.0/](https://mirrors.aliyun.com/apache/lucene/solr/6.6.0/)



## 2.提取tar文件

```sh
$ cd /opt/
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/lucene/solr/6.6.0/solr-6.6.0.tgz
$ tar -zxf solr-6.6.0.tgz 
$ cd solr-6.6.0
```


# 未完待续，睡觉