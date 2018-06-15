---
layout: post
title: Spring Boot 中使用 Dubbo 详解
categories: [dubbo,SpringBoot]
description: Spring Boot 中使用 Dubbo 详解
keywords: Dubbo 
---

Dubbo是阿里巴巴SOA服务化治理方案的核心框架，每天为2,000+个服务提供3,000,000,000+次访问量支持，并被广泛应用于阿里巴巴集团的各成员站点。Dubbo是一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。

# Dubbo 简介

## Dubbo 是什么？

Dubbo是一个分布式服务框架，致力于提供高性能和透明化的RPC远程服务调用方案，以及SOA服务治理方案。简单的说，dubbo就是个服务框架，如果没有分布式的需求，其实是不需要用的，只有在分布式的时候，才有dubbo这样的分布式服务框架的需求，并且本质上是个服务调用的东东，说白了就是个远程服务调用的分布式框架

**其核心部分包含:**

1.远程通讯: 提供对多种基于长连接的NIO框架抽象封装，包括多种线程模型，序列化，以及“请求-响应”模式的信息交换方式。  
2.集群容错: 提供基于接口方法的透明远程过程调用，包括多协议支持，以及软负载均衡，失败容错，地址路由，动态配置等集群支持。  
3.自动发现: 基于注册中心目录服务，使服务消费方能动态的查找服务提供方，使地址透明，使服务提供方可以平滑增加或减少机器。  

## Dubbo 能做什么？

1.透明化的远程方法调用，就像调用本地方法一样调用远程方法，只需简单配置，没有任何API侵入。   
2.软负载均衡及容错机制，可在内网替代F5等硬件负载均衡器，降低成本，减少单点。  
3.服务自动注册与发现，不再需要写死服务提供方地址，注册中心基于接口名查询服务提供者的IP地址，并且能够平滑添加或删除服务提供者。  

## Dubbo 的架构

![ 架构图 ][1]

节点角色说明

节点	 | 角色说明
---|---
Provider	| 暴露服务的服务提供方
Consumer	| 调用远程服务的服务消费方
Registry	| 服务注册与发现的注册中心
Monitor	| 统计服务的调用次调和调用时间的监控中心
Container	| 服务运行容器

**Dubbo提供三个关键功能，包括基于接口的远程呼叫，容错和负载平衡以及自动服务注册和发现**

**调用关系说明**

1.服务容器负责启动，加载，运行服务提供者。  
2.服务提供者在启动时，向注册中心注册自己提供的服务。  
3.服务消费者在启动时，向注册中心订阅自己所需的服务。  
4.注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。  
5.服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。  
6.服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。  

## Dubbo 特点

**Dubbo 架构具有以下几个特点，分别是连通性、健壮性、伸缩性、以及向未来架构的升级性**

**连通性**

* 注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小
* 监控中心负责统计各服务调用次数，调用时间等，统计先在内存汇总后每分钟一次发送到监控中心服务器，并以报表展示
* 服务提供者向注册中心注册其提供的服务，并汇报调用时间到监控中心，此时间不包含网络开销
* 服务消费者向注册中心获取服务提供者地址列表，并根据负载算法直接调用提供者，同时汇报调用时间到监控中心，此时间包含网络开销
* 注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外
* 注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者
* 注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表
* 注册中心和监控中心都是可选的，服务消费者可以直连服务提供者


**健状性**

* 监控中心宕掉不影响使用，只是丢失部分采样数据
* 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务
* 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
* 注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯
* 服务提供者无状态，任意一台宕掉后，不影响使用
* 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复
   
**伸缩性**

* 注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心
* 服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者


**升级性**

当服务集群规模进一步扩大，带动IT治理结构进一步升级，需要实现动态部署，进行流动计算，现有分布式服务架构不会带来阻力。下图是未来可能的一种架构：

![ 未来可能的一种架构 ][2]

节点角色说明

节点	 | 角色说明
---|---
Deployer	|自动部署服务的本地代理
Repository	|仓库用于存储服务应用发布包
Scheduler	|调度中心基于访问压力自动增减服务提供者
Admin	|统一管理控制台
Registry	|服务注册与发现的注册中心
Monitor	|统计服务的调用次调和调用时间的监控中心

# 快速开始

Dubbo 采用全 Spring 配置方式，透明化接入应用，对应用没有任何 API 侵入，只需用 Spring 加载 Dubbo 的配置即可，Dubbo 基于 Spring 的 Schema 扩展进行加载。

## 环境安装 

任选其一

[CentOs7.3 搭建 ZooKeeper-3.4.9 单机服务](https://segmentfault.com/a/1190000010791627)  
[CentOs7.3 搭建 ZooKeeper-3.4.9 Cluster 集群服务](https://segmentfault.com/a/1190000010807875)   

# Github 代码


代码我已放到 Github ，导入`spring-boot-dubbo` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-dubbo](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-dubbo)

## Maven依赖

在项目中添加 `dubbo` 依赖

```xml
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>dubbo</artifactId>
    <version>2.5.6</version>
</dependency>
```

## 定义服务接口

**项目：`dubbo-api`** 

```java
public interface DemoService {
    String sayHello(String name);
}
```

## 服务提供方

**项目：`dubbo-provider`，在服务提供方实现接口**

```java
@Service("demoService")
public class DemoServiceImpl implements DemoService {

    @Override
    public String sayHello(String name) {
        System.out.println("[" + new SimpleDateFormat("HH:mm:ss").format(new Date()) + "] Hello " + name + ", request from consumer: " + RpcContext.getContext().getRemoteAddress());
        return "Hello " + name + ", response form provider: " + RpcContext.getContext().getLocalAddress();
    }
}
```

**加载 dubbo 配置**

```java
@Configuration
@PropertySource("classpath:dubbo.properties")
@ImportResource({"classpath:dubbo/*.xml"})
public class PropertiesConfig {
}
```

在提供方增加暴露服务配置 : `<dubbo:service>`

`dubbo-provider.xml`

```xml
<!-- 声明需要暴露的服务接口 -->
<dubbo:service interface="io.ymq.dubbo.api.DemoService" ref="demoService"/>
```

## 服务消费方

**项目：`dubbo-consumer` ，消费消费远程方法**

```java
@Service("consumerDemoService")
public class ConsumerDemoService {

    @Autowired
    private DemoService demoService;

    public void sayHello(String name) {
        String hello = demoService.sayHello(name); // 执行消费远程方法
        System.out.println(hello); // 显示调用结果
    }
	
}
```

**加载 dubbo 配置**

```java
@Configuration
@PropertySource("classpath:dubbo.properties")
@ImportResource({"classpath:dubbo/*.xml"})
public class PropertiesConfig {
}
```

在消费方增加引用服务配置: `<dubbo:reference>`

`dubbo-consumer.xml`

```xml
<!-- 增加引用远程服务配置 可以和本地bean一样使用demoService -->
<dubbo:reference id="demoService" check="false" interface="io.ymq.dubbo.api.DemoService"/>
```

## 远程服务 Dubbo 配置

**项目：`dubbo-provider`** ,**`dubbo-consumer`** 一样配置

**`dubbo.xml`**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
        http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd
        http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

    <!-- 提供方应用信息，用于计算依赖关系 -->
    <dubbo:application name="${spring.application.name}"  />

    <!-- 使用multicast广播注册中心暴露服务地址 -->
    <dubbo:registry protocol="zookeeper" address="${zookeeper.connect}"  file="${dubbo.cache}"/>

    <!-- 用dubbo协议在20880端口暴露服务 -->
    <dubbo:protocol name="dubbo" port="${dubbo.protocol.port}"  threadpool="${dubbo.protocol.threadpool}"  threads="${dubbo.protocol.threads}"/>

    <!-- 提供方的缺省值，当ProtocolConfig和ServiceConfig某属性没有配置时，采用此缺省值，可选。-->
    <dubbo:provider connections="${dubbo.provider.connections}" timeout="${dubbo.provider.timeout}" retries="${dubbo.provider.retries}" version="${dubbo.provider.version}" />

    <!-- 消费方缺省配置，当ReferenceConfig某属性没有配置时，采用此缺省值，可选。-->
    <dubbo:consumer version="${dubbo.provider.version}" />

    <!-- 监控中心配置，用于配置连接监控中心相关信息，可选。-->
    <dubbo:monitor protocol="registry"/>

</beans>
```

**`dubbo.properties`**

```
#########################################################
# dubbo config
#暴露服务端口
dubbo.protocol.port=20880
#提供方超时时间
dubbo.provider.timeout=10000
#提供方版本
dubbo.provider.version=1.0
#表示该服务使用独的五条条长连
dubbo.provider.connections=5
# 固定大小线程池，启动时建立线程，不关闭，一直持有。(缺省)
dubbo.protocol.threadpool=fixed
# 线程数量
dubbo.protocol.threads=500
#配置重试次数，最好只用于读的重试，写操作可能会引起多次写入  默认retries="0"
dubbo.provider.retries=0
# dubbo缓存文件
dubbo.cache=/data/dubbo/cache/dubbo-provider
#########################################################
# zookeeper config
zookeeper.connect=127.0.0.1:2181
```

## 测试 Dubbo

1. 该接口需单独打包，在服务提供方和消费方共享 ↩  
2. 对服务消费方隐藏实现 ↩  
3. 也可以使用 IoC 注入 ↩  

###  启动 ZooKeeper

启动服务

```sh
/opt/zookeeper-3.4.9/bin/zkServer.sh start
```

### 启动提供方服务

```
package io.ymq.dubbo.provider.run;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;

/**
 * 描述:启动提供方服务
 *
 * @author yanpenglei
 * @create 2017-10-27 11:49
 **/
@SpringBootApplication
@ComponentScan(value = {"io.ymq.dubbo"})
public class Startup {

    public static void main(String[] args) {
        SpringApplication.run(Startup.class, args);
    }
}

```

### 测试消费远程服务

```java
package io.ymq.dubbo.test;

import io.ymq.dubbo.consumer.run.Startup;
import io.ymq.dubbo.consumer.service.ConsumerDemoService;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * 描述: 测试消费远程服务
 *
 * @author yanpenglei
 * @create 2017-10-27 14:15
 **/
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Startup.class)
public class ConsumerTest {

    @Autowired
    private ConsumerDemoService consumerDemoService;

    @Test
    public void sayHello(){
        consumerDemoService.sayHello("Peng Lei");
    }
}
```
响应：

```
[15:54:00] Hello Peng Lei, request from consumer: /10.4.82.6:63993
```


代码我已放到 Github ，导入`spring-boot-dubbo` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-dubbo](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-dubbo)

[1]: http://www.ymq.io/images/2017/dubbo/dubbo-architecture.png
[2]: http://www.ymq.io/images/2017/dubbo/dubbo-architecture-future.jpg


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")


