---
layout: post
title: Spring Cloud（二）Consul 服务治理实现
categories: SpringCloud
description: Spring Cloud（二）Consul 服务治理实现
keywords: SpringCloud 
---

Spring Cloud Consul 项目是针对Consul的服务治理实现。Consul是一个分布式高可用的系统，具有分布式、高可用、高扩展性。

# Consul 简介

Consul 是 HashiCorp 公司推出的开源工具，用于实现分布式系统的服务发现与配置。与其他分布式服务注册与发现的方案，Consul的方案更“一站式”
，内置了服务注册与发现框 架、**具有以下性质：**

- 分布一致性协议实现、
- 健康检查、
- Key/Value存储、
- 多数据中心方案，

不再需要依赖其他工具（比如ZooKeeper等）。

使用起来也较 为简单。Consul使用Go语言编写，因此具有天然可移植性(支持Linux、windows和Mac OS X)；安装包仅包含一个可执行文件，方便部署，与Docker等轻量级容器可无缝配合 。
基于 Mozilla Public License 2.0 的协议进行开源. Consul 支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对.
一致性协议采用 Raft 算法,用来保证服务的高可用. 使用 GOSSIP 协议管理成员和广播消息, 并且支持 ACL 访问控制.

## Consul 的使用场景

- docker 实例的注册与配置共享
- coreos 实例的注册与配置共享
- vitess 集群
- SaaS 应用的配置共享
- 与 confd 服务集成，动态生成 nginx 和 haproxy 配置文件

## Consul 的优势

使用 Raft 算法来保证一致性, 比复杂的 Paxos 算法更直接. 相比较而言, zookeeper 采用的是 Paxos, 而 etcd 使用的则是 Raft.
支持多数据中心，内外网的服务采用不同的端口进行监听。 多数据中心集群可以避免单数据中心的单点故障,而其部署则需要考虑网络延迟, 分片等情况等. zookeeper 和 etcd 均不提供多数据中心功能的支持.
支持健康检查. etcd 不提供此功能.
支持 http 和 dns 协议接口. zookeeper 的集成较为复杂, etcd 只支持 http 协议.
官方提供web管理界面, etcd 无此功能.

## Consul 的角色
client: 客户端, 无状态, 将 HTTP 和 DNS 接口请求转发给局域网内的服务端集群.server: 服务端, 保存配置信息, 高可用集群, 在局域网内与本地客户端通讯, 通过广域网与其他数据中心通讯. 每个数据中心的 server 数量推荐为 3 个或是 5 个.

由于Spring Cloud Consul项目的实现，我们可以轻松的将基于Spring Boot的微服务应用注册到Consul上，并通过此实现微服务架构中的服务治理。

# 搭建环境

**参考**

 - [Spring Cloud 官方文档](http://cloud.spring.io/spring-cloud-consul/ ) 
 - [Consul 官方文档 ](https://www.consul.io/intro/getting-started/install.html)

要想利用Consul提供的服务实现服务的注册与发现，我们需要搭建Consul Cluster 环境。

在Consul方案中，每个提供服务的节点上都要部署和运行Consul的agent，所有运行Consul agent节点的集合构成Consul Cluster。

Consul agent有两种运行模式：Server和Client。这里的Server和Client只是Consul集群层面的区分，与搭建在Cluster之上 的应用服务无关。

以Server模式运行的Consul agent节点用于维护Consul集群的状态，官方建议每个Consul Cluster至少有3个或以上的运行在Server mode的Agent，Client节点不限。

**环境配置如下:**

Centos 7.3

| 主机名称| IP | 作用 | 是否允许远程访问
| --------	| -------- 	| -------- |-------- |
| node1     | 192.168.252.121| consul server  | 是 |
| node2  	| 192.168.252.122| consul client  | 否 |
| node3     | 192.168.252.123| consul client  | 是 |

**关闭防火墙**

```sh
systemctl stop firewalld.service
```

Consul 最新版的下载地址:   
[https://releases.hashicorp.com/consul/1.0.1/consul_1.0.1_linux_amd64.zip](https://releases.hashicorp.com/consul/1.0.1/consul_1.0.1_linux_amd64.zip)

**下载，然后unzip 解压，得到唯一，一个可执行文件**

```sh
cd /opt/
wget https://releases.hashicorp.com/consul/1.0.1/consul_1.0.1_linux_amd64.zip
unzip consul_1.0.1_linux_amd64.zip
cp consul /usr/local/bin/
```

**查看是否安装成功**

```sh
[root@node1 opt]# consul
```

出现如下结果，表示安装成功

```sh
Usage: consul [--version] [--help] <command> [<args>]

Available commands are:
    agent          Runs a Consul agent
    catalog        Interact with the catalog
    event          Fire a new event
    exec           Executes a command on Consul nodes
    force-leave    Forces a member of the cluster to enter the "left" state
    info           Provides debugging information for operators.
    join           Tell Consul agent to join cluster
    keygen         Generates a new encryption key
    keyring        Manages gossip layer encryption keys
    kv             Interact with the key-value store
    leave          Gracefully leaves the Consul cluster and shuts down
    lock           Execute a command holding a lock
    maint          Controls node or service maintenance mode
    members        Lists the members of a Consul cluster
    monitor        Stream logs from a Consul agent
    operator       Provides cluster-level tools for Consul operators
    reload         Triggers the agent to reload configuration files
    rtt            Estimates network round trip time between nodes
    snapshot       Saves, restores and inspects snapshots of Consul server state
    validate       Validate config files/directories
    version        Prints the Consul version
    watch          Watch for changes in Consul
```

**检查版本**

```sh
[root@node1 opt]# consul version
```

```sh
Consul v1.0.1
Protocol 2 spoken by default, understands 2 to 3 (agent will automatically use protocol >2 when speaking to compatible agents)
```

## Consul常用命令


| 命令| 解释 | 示例 | 
| --------	| -------- 	| -------- |
| agent     | 运行一个consul agent   				 | consul agent -dev  |
| join  	| 将agent加入到consul集群       		 |consul join IP  	 |
| members   | 列出consul cluster集群中的members      |consul members   |
| leave     | 将节点移除所在集群   					 | consul leave    |
	

## consul agent 命令的常用选项

**-data-dir**

- 作用：指定agent储存状态的数据目录
- 这是所有agent都必须的
- 对于server尤其重要，因为他们必须持久化集群的状态

**-config-dir**

- 作用：指定service的配置文件和检查定义所在的位置
- 通常会指定为”某一个路径/consul.d”（通常情况下，.d表示一系列配置文件存放的目录）

**-config-file**

- 作用：指定一个要装载的配置文件
- 该选项可以配置多次，进而配置多个配置文件（后边的会合并前边的，相同的值覆盖）

**-dev**

- 作用：创建一个开发环境下的server节点
- 该参数配置下，不会有任何持久化操作，即不会有任何数据写入到磁盘
- 这种模式不能用于生产环境（因为第二条）

**-bootstrap-expect**
- 作用：该命令通知consul server我们现在准备加入的server节点个数，该参数是为了延迟日志复制的启动直到我们指定数量的server节点成功的加入后启动。

**-node**

- 作用：指定节点在集群中的名称
- 该名称在集群中必须是唯一的（默认采用机器的host）
- 推荐：直接采用机器的IP

**-bind**

- 作用：指明节点的IP地址
- 有时候不指定绑定IP，会报`Failed to get advertise address: Multiple private IPs found. Please configure one.` 的异常

**-server**

- 作用：指定节点为server
- 每个数据中心（DC）的server数推荐至少为1，至多为5
- 所有的server都采用raft一致性算法来确保事务的一致性和线性化，事务修改了集群的状态，且集群的状态保存在每一台server上保证可用性
- server也是与其他DC交互的门面（gateway）

**-client**

- 作用：指定节点为client，指定客户端接口的绑定地址，包括：HTTP、DNS、RPC
- 默认是127.0.0.1，只允许回环接口访问
- 若不指定为-server，其实就是-client

**-join**

- 作用：将节点加入到集群

**-datacenter**（老版本叫-dc，-dc已经失效）

- 作用：指定机器加入到哪一个数据中心中

## 启动服务

我们尝试一下：

`-dev表示开发模式运行，使用-client 参数可指定允许客户端使用什么ip去访问，例如-client 192.168.252.121 表示可以使用`

[http://192.168.252.121:8500/ui/ 去访问。](http://192.168.252.121:8500/ui/)

```sh
consul agent -dev -client 192.168.252.121
```


![Consul Cluster][1]

## Consul 的高可用

Consul Cluster集群架构图如下： 

![Consul Cluster集群架构][2]

这边准备了三台Centos 7.3的虚拟机，主机规划如下，供参考： 

| 主机名称| IP | 作用 | 是否允许远程访问
| --------	| -------- 	| -------- |-------- |
| node1     | 192.168.252.121| consul server  | 是 |
| node2  	| 192.168.252.122| consul client  | 否 |
| node3     | 192.168.252.123| consul client  | 是 |

## 搭建步骤

命令参数，参看上面详细介绍

**在 node1 机器上启动 Consul**

```sh
cd /opt/
mkdir data
consul agent -data-dir /opt/data -node=192.168.252.121 -bind=0.0.0.0 -datacenter=dc1 -ui -client=192.168.252.121 -server -bootstrap-expect 1 > /dev/null 2>&1 &
```

**在 node2 机器上启动 Consul,并且将node2节点加入到node1节点上**
```sh
cd /opt/
mkdir data
consul agent -data-dir /opt/data -node=192.168.252.122 -bind=0.0.0.0 -datacenter=dc1 -ui -client=192.168.252.122 -join=192.168.252.121 > /dev/null 2>&1 &
```

**在 node3 机器上启动 Consul,并且将node3节点加入到node1节点上**

```sh
cd /opt/
mkdir data
consul agent -data-dir /opt/data -node=192.168.252.123 -bind=0.0.0.0 -datacenter=dc1 -ui  -client=192.168.252.123 -join=192.168.252.121 > /dev/null 2>&1 &
```


在node1上查看当前集群节点：

```sh
consul members -rpc-addr=192.168.252.123:8400  

consul leave -rpc-addr=192.168.252.123:8400  

```

[http://192.168.252.121:8500/ui/ 去访问。](http://192.168.252.121:8500/ui/)


![Consul Cluster集群 nodes][3]

# GitHub:代码
  
代码我已放到 Github ，导入`spring-cloud-consul-client` 项目 

github [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-consul-client](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-consul-client)

## 添加依赖

在项目 `spring-cloud-consul` `pom.xml`中引入需要的依赖内容：

```xml
<!-- spring cloud starter consul discovery -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

## 开启服务注册

客户端注册Consul时，它提供有关自身的元数据，如主机和端口，ID，名称和标签。默认情况下，将创建一个HTTP 检查，每隔10秒Consul命中/health端点。如果健康检查失败，则服务实例被标记为关键。

```java
package io.ymq.example.consul;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@EnableDiscoveryClient
@RestController
public class ConsulApplication {

    @RequestMapping("/")
    public String home() {
        return "Hello world";
    }

	public static void main(String[] args) {
		SpringApplication.run(ConsulApplication.class, args);
	}
}

```

## 配置文件

在`application.yml`配置文件中增加如下信息：如果Consul客户端位于localhost:8500以外，则需要配置来定位客户端

```sh
spring:
  application:
    name: consul-client
  cloud:
    consul:
      host: 192.168.252.121
      port: 8500
      discovery:
        healthCheckPath: /
        healthCheckInterval: 5s
```


如果Consul客户端位于localhost:8500以外的位置，则需要配置来定位客户端。例：

```
host: 192.168.252.121
port: 8500
```

**HTTP健康检查路径**

“10s”和“1m”分别表示10秒和1分

```
discovery:
	healthCheckPath: ${management.context-path}/health
	healthCheckInterval: 15s
```

## 访问服务

![Consul Cluster 集群 服务注册情况][4]

![Consul Cluster集群 服务注册情况][5]

## 源码下载

代码我已放到 Github ，导入`spring-cloud-consul-client` 项目 

github [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-consul-client](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-consul-client)


[1]: http://www.ymq.io/images/2017/SpringCloud/consul/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/consul/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/consul/3.png
[4]: http://www.ymq.io/images/2017/SpringCloud/consul/4.png
[5]: http://www.ymq.io/images/2017/SpringCloud/consul/5.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

