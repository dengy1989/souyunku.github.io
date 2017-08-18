---
layout: post
title: CentOs7.3 搭建 ZooKeeper-3.4.9 集群
categories: JDK
description: CentOs7.3 安装 Jdk1.8
keywords: JDK
---

# CentOs7.3 搭建 ZooKeeper-3.4.9 集群

## 环境

VMware版本号：12.0.0

CentOS版本：CentOS 7.3.1611

ZooKeeper版本：ZooKeeper-3.4.9.tar.gz

虚拟机IP：192.168.252.101，192.168.252.102，192.168.252.103

集群主机名称：node1，node2，node3 具体参考[《CentOs7.3 修改主机名》](https://segmentfault.com/a/1190000010723105) 

集群主机用户：都是用root用户

集群JDK环境：jdk-8u144-linux-x64.tar.gz 具体参考[CentOs7.3 安装 JDK1.8](https://segmentfault.com/a/1190000010716919)

集群主机之间设置免密登陆：设置方式见：[《CentOs7.3 ssh 免密登录》](https://segmentfault.com/a/1190000010738165)


# 周末出去玩，下周接着搞