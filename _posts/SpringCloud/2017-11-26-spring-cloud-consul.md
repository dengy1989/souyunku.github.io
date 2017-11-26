---
layout: post
title:  SpringCloud 服务的注册与发现（Eureka Consul）
categories: SpringCloud
description: SpringCloud 服务的注册与发现（Eureka Consul）
keywords: SpringCloud 
---

Spring Cloud Consul项目是针对Consul的服务治理实现。Consul是一个分布式高可用的系统，它包含多个组件，但是作为一个整体，在微服务架构中为我们的基础设施提供服务发现和服务配置的工具。它包含了下面几个特性：

提供的模式包括：服务发现，控制总线和配置。智能路由（Zuul）和客户端负载平衡（Ribbon），断路器（Hystrix）通过与Spring Cloud Netflix的集成提供。


# 修改依赖

由于Spring Cloud Consul项目的实现，我们可以轻松的将基于Spring Boot的微服务应用注册到Consul上，并通过此实现微服务架构中的服务治理。

基于之前 Eureka的示例（[spring-cloud-eureka-client](http://www.ymq.io/2017/11/22/spring-cloud-eureka/) ）为基，如何把服务提供者注册到Consul上呢？只需要在pom.xml中将eureka的依赖，修改成如下依赖


```java
<!-- spring cloud starter consul discovery -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```

# 配置 Consul 服务

`application.yml`配置文件中增加如下信息：

```sh
spring:
  cloud:
    consul:
      host: localhost
      port: 8500
```

# 安装 Agent

所有Spring Cloud Consul应用程序要想利用Consul提供的服务实现服务的注册与发现, 必须使用Consul Agent客户端。默认情况下，代理客户端访问地址:[http://localhost:8500](http://localhost:8500)。安装后，您可以使用以下命令启动 `Consul Agent`

有关如何启动代理客户端以及如何连接到Consul Agent服务器集群的详细信息，请参阅 [代理文档](https://www.consul.io/docs/agent/basics.html)

在Consul方案中，每个提供服务的节点上都要部署和运行Consul Agent，所有运行Consul Agent节点的集合构成Consul Cluster。

Consul Agent有两种运行模式：server和client。这里的server和client只是consul集群层面的区分，与搭建在Cluster之上的应用服务无关。
以server模式运行的Consul Agentt节点用于维护consul集群的状态，
官方建议每个consul cluster至少有3个或以上的运行在server mode的agent，client节点不限。

我们这里以安装三个节点为例，环境配置如下



# 服务发现与Consul

服务发现是基于微服务架构的关键原则之一。尝试配置每个客户端或某种形式的约定可能非常困难，可以非常脆弱。Consul通过HTTP API和DNS提供服务发现服务。Spring Cloud Consul利用HTTP API进行服务注册和发现。这不会阻止非Spring Cloud应用程序利用DNS界面。Consul代理服务器在通过八卦协议进行通信的集群中运行，并使用Raft协议协议。


# 未完待续



## 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-client](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-client)

- **作者：鹏磊** 
- **出处：[http://www.ymq.io](http://www.ymq.io/)**      
- **版权归作者所有，转载请注明出处** 
