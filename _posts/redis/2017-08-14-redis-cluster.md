---
layout: post
title: Redis集群搭建与简单使用
categories: Redis
description: Redis集群搭建与简单使用
keywords: Redis
---

## 环境

VMware版本号：12.0.0
CentOS版本：CentOS release 6.5 (Final)
三台虚拟机(IP)：192.168.252.150，192.168.252.151，192.168.252.151

## 安装
 
**本文使用的最新稳定版本 redis-4.0.1** 

 
**安裝 GCC 编译工具 不然会有编译不过的问题**

```
yum install -y gcc g++ gcc-c++ make
```

**下载，解压，编译安装**

```sh
wget http://download.redis.io/releases/redis-4.0.1.tar.gz
tar xzf redis-4.0.1.tar.gz
cd redis-4.0.1
make
```
如果因为上次编译失败，有残留的文件
 
```
make distclean
```

## 创建 Redis 节点

首先在 192.168.252.150机器上 /opt/redis-4.0.1目录下创建 `redis_cluster` 目录

```
mkdir /opt/redis-4.0.1/redis_cluster
```

在 `redis_cluster` 目录下，创建名为`7000、7001、7002`的目录，并将 `redis.conf` 拷贝到这三个目录中

```
mkdir 7000 7001 7002

cp /opt/redis-4.0.1/redis.conf /opt/redis-4.0.1/redis_cluster/7000
cp /opt/redis-4.0.1/redis.conf /opt/redis-4.0.1/redis_cluster/7001
cp /opt/redis-4.0.1/redis.conf /opt/redis-4.0.1/redis_cluster/7002

```

分别修改这三个配置文件，修改如下内容
 
```
port                  7000                        //端口7000,7002,7003        
bind                  本机ip                      //默认ip为127.0.0.1，需要改为其他节点机器可访问的ip，否则创建集群时无法访问对应的端口，无法创建集群
daemonize             yes                         //redis后台运行
pidfile               /var/run/redis_7000.pid     //pidfile文件对应7000，7001，7002
cluster-enabled       yes                         //开启集群，把注释#去掉
cluster-config-file   nodes_7000.conf             //集群的配置，配置文件首次启动自动生成 7000，7001，7002
cluster-node-timeout  15000                       //请求超时，默认15秒，可自行设置
appendonly            yes                         //aof日志开启，有需要就开启，它会每次写操作都记录一条日志　
```

**接着在另外两台机器上(192.168.252.151，192.168.252.151)重复以上三步，只是把目录改为7003、7004、7005、7006、7007、7008对应的配置文件也按照这个规则修改即可**



## 关闭防火墙

```
service   iptables stop
```


## 启动各个节点

```

##第一台机器上执行
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7000/redis.conf
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7001/redis.conf
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7002/redis.conf

##第二台机器上执行
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7003/redis.conf
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7004/redis.conf
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7005/redis.conf
                     
##第三台机器上执行   
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7006/redis.conf
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7007/redis.conf
/opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/7008/redis.conf

```

## 检查各 Redis 启动情况

```
##第一台机器
$ ps -ef | grep redis           //redis是否启动成功
$ netstat -tnlp | grep redis    //监听redis端口
```



## 安装 Ruby

```
yum -y install ruby ruby-devel rubygems rpm-build
gem install redis
```

Redis 官方提供了 redis-trib.rb 这个工具，就在解压目录的 src 目录中
 
```
/opt/redis-4.0.1/src/redis-trib.rb create --replicas 1 192.168.252.150:7000 192.168.252.150:7001 192.168.252.150:7002 192.168.252.151:7003 192.168.252.151:7004 192.168.252.151:7005 192.168.252.152:7006 192.168.252.152:7007 192.168.252.152:7008
```

输入 yes，然后出现如下内容，说明安装成功
 
## 集群验证
在第一台机器上连接集群的7000节点，在另外一台连接7004节点，连接方式为：
 
```
##加参数 -C 可连接到集群，因为 redis.conf 将 bind 改为了ip地址，所以 -h 参数不可以省略，-p 参数为端口号

/opt/redis-4.0.1/src/redis-cli -h 192.168.252.152 -c -p 7006
```

