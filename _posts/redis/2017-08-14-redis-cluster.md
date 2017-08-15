---
layout: post
title: Redis-4.0.1 Cluster 集群搭建与简单使用
categories: Redis
description: Redis-4.0.1 Cluster 集群搭建与简单使用
keywords: Redis
---

## 环境

 - VMware版本号：12.0.0
 - CentOS版本：CentOS release 6.5
 - 三台虚拟机(IP)：192.168.252.150,192.168.252..151,192.168.252..152

### 注意事项
 

安裝 GCC 编译工具 不然会有编译不过的问题

```sh
$ yum install -y gcc g++ gcc-c++ make
```

升级所有的包，防止出现版本过久不兼容问题

```sh
$ yum -y update
```

关闭防火墙 节点之前需要开放指定端口，为了方便，生产不要禁用

centos 6.5

```
$ service iptables stop # 关闭命令：
```

centos 7.0
```
$ systemctl stop firewalld.service # 停止firewall
```



## 集群搭建

### 安装 Redis

**下载，解压，编译安装**

```sh
$ wget http://download.redis.io/releases/redis-4.0.1.tar.gz
$ tar xzf redis-4.0.1.tar.gz
$ cd redis-4.0.1
$ make
```

**如果因为上次编译失败，有残留的文件**
 
```sh
$ make distclean
```


### 创建节点

1.首先在 192.168.252.150机器上 /opt/redis-4.0.1目录下创建 `redis_cluster` 目录

```sh
$ mkdir /opt/redis-4.0.1/redis_cluster
```

2.在 `redis_cluster` 目录下，创建名为`7000、7001、7002`的目录

```sh

$ mkdir 7000 7001 7002

$ vi redis_cluster/7000
$ vi redis_cluster/7001
$ vi redis_cluster/7002
```

3.分别修改这三个配置文件，把如下内容粘贴进去
 
```sh
port                  7000                        #端口7000,7002,7003        
bind                  本机ip                      #默认ip为127.0.0.1，需要改为其他节点机器可访问的ip，否则创建集群时无法访问对应的端口，无法创建集群
daemonize             yes                         #redis后台运行
pidfile               /var/run/redis_7000.pid     #pidfile文件对应7000，7001，7002
cluster-enabled       yes                         #开启集群，把注释#去掉
cluster-config-file   nodes_7000.conf             #集群的配置，配置文件首次启动自动生成 7000，7001，7002
cluster-node-timeout  15000                       #请求超时，默认15秒，可自行设置
appendonly            yes                         #aof日志开启，有需要就开启，它会每次写操作都记录一条日志　
```

**接着在另外两台机器上`(192.168.252.151，192.168.252.152)`重复以上三步，只是把目录改为`7003、7004、7005、7006、7007、7008`对应的配置文件也按照这个规则修改即可**



### 启动集群

```sh
#第一台机器上执行 3个节点
$ for((i=0;i<=3;i++)); do /opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/700$i/redis.conf; done

#第二台机器上执行 3个节点
$ for((i=3;i<=3;i++)); do /opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/700$i/redis.conf; done
                     
#第三台机器上执行 3个节点 
$ for((i=6;i<=3;i++)); do /opt/redis-4.0.1/src/redis-server /opt/redis-4.0.1/redis_cluster/700$i/redis.conf; done
```

### 检查服务

检查各 Redis 各个节点启动情况
 
```sh
$ ps -ef | grep redis           //redis是否启动成功
$ netstat -tnlp | grep redis    //监听redis端口
```

### 安装 Ruby

```sh
$ yum -y install ruby ruby-devel rubygems rpm-build
$ gem install redis
```

### 创建集群
 
Redis 官方提供了 `redis-trib.rb` 这个工具，就在解压目录的 src 目录中,每个节点正常开启后，在任意一台上运行
 
```sh
$ /opt/redis-4.0.1/src/redis-trib.rb create --replicas 1 192.168.252.150:7000 192.168.252.150:7001 192.168.252.150:7002 192.168.252.151:7003 192.168.252.151:7004 192.168.252.151:7005 192.168.252.152:7006 192.168.252.152:7007 192.168.252.152:7008
```

出现以下内容

```sh
Can I set the above configuration? (type 'yes' to accept): yes
```

**输入 yes**




### 关闭集群

这样也可以，推荐

```sh
$ pkill redis
```


循环节点逐个关闭

```sh
$ for((i=0;i<3;i++)); do /opt/redis-4.0.1/src/redis-cli -c -h 192.168.252.150 -p 700$i shutdown; done
```


## 集群验证

 
### 连接集群测试
 
参数 -C 可连接到集群，因为 redis.conf 将 bind 改为了ip地址，所以 -h 参数不可以省略，-p 参数为端口号
 
```sh
$ /opt/redis-4.0.1/src/redis-cli -h 192.168.252.150 -c -p 7000

192.168.252.150:7000> set name www.ymq.io
-> Redirected to slot [5798] located at 192.168.252.150:7000
OK
192.168.252.150:7000> get name
"www.ymq.io"
192.168.252.150:7000>
```

### 检查集群状态
 
```sh
$ /opt/redis-4.0.1/src/redis-trib.rb check 192.168.252.150:7000
```

### 列出集群节点

列出集群当前已知的所有节点（node），以及这些节点的相关信息
 
```sh
$ /opt/redis-4.0.1/src/redis-cli -h 192.168.252.150 -c -p 7000 cluster nodes_7000
```
### 打印集群信息 
 
```sh
$ /opt/redis-4.0.1/src/redis-cli -h 192.168.252.150 -c -p 7000 cluster info
```



