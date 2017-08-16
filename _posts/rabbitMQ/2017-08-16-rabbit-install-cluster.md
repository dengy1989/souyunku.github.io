---
layout: post
title: RabbitMQ 3.6 Cluster 集群搭建
categories: RabbitMQ
description: RabbitMQ 3.6 Cluster 集群搭建
keywords: RabbitMQ
---

# RabbitMQ 3.6 Cluster 集群搭建

## 集群概述

通过 Erlang 的分布式特性（通过 magic cookie 认证节点）进行 RabbitMQ 集群，各 RabbitMQ 服务为对等节点，即每个节点都提供服务给客户端连接，进行消息发送与接收。
 
 
 
这些节点通过 RabbitMQ HA 队列（镜像队列）进行消息队列结构复制。本方案中搭建 3 个节点，并且都是磁盘节点（所有节点状态保持一致，节点完全对等），只要有任何一个节点能够工作，RabbitMQ 集群对外就能提供服务。

## 环境
 - VMware版本号：12.0.0
 - CentOS版本：CentOS 7.3.1611
 - 虚拟机(IP)：192.168.252.101，192.168.252.102，192.168.252.103

# 安装 RabbitMQ

## 单机安装

**首先在三台主机上都安装 `Erlang` 安装 `RabbitMQ Server`**


[参考 我的另一篇 Centos7.3 安装 RabbitMQ 3.6](https://segmentfault.com/a/1190000010693696#articleHeader4)

# 配置集群

## 修改 hostname

**临时生效**

命令格式

```sh
hostname <new hostname>
```

在三台机器`192.168.252.101，192.168.252.102，192.168.252.103`分别执行

```sh
hostname node1
hostname node2
hostname node3
```

## 修改 hosts


编辑/etc/hosts文件，添加
```
$ vi /etc/hosts
```
```
192.168.252.101 node1
192.168.252.102 node2
192.168.252.103 node3
```




**永久更改**

[参考：linux修改主机名](http://www.ymq.io/2017/07/31/linux-localdomain/)