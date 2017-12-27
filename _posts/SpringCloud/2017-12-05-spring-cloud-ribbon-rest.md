---
layout: post
title: Spring Cloud（三）服务提供者 Eureka + 服务消费者（rest + Ribbon）
categories: SpringCloud
description: Spring Cloud 服务提供者 Eureka + 服务消费者（rest + Ribbon）
keywords: SpringCloud 
---

Spring Cloud Ribbon是基于Netflix Ribbon实现的一套客户端负载均衡的工具。它是一个基于HTTP和TCP的客户端负载均衡器。它可以通过在客户端中配置ribbonServerList来设置服务端列表去轮询访问以达到均衡负载的作用。

# Ribbon是什么？

Ribbon是Netflix发布的开源项目，主要功能是提供客户端的软件负载均衡算法，将Netflix的中间层服务连接在一起。Ribbon客户端组件提供一系列完善的配置项如连接超时，重试等。简单的说，就是在配置文件中列出Load Balancer（简称LB）后面所有的机器，Ribbon会自动的帮助你基于某种规则（如简单轮询，随即连接等）去连接这些机器。我们也很容易使用Ribbon实现自定义的负载均衡算法。

# LB方案分类

目前主流的LB方案可分成两类：一种是集中式LB, 即在服务的消费方和提供方之间使用独立的LB设施(可以是硬件，如F5, 也可以是软件，如nginx), 由该设施负责把访问请求通过某种策略转发至服务的提供方；另一种是进程内LB，将LB逻辑集成到消费方，消费方从服务注册中心获知有哪些地址可用，然后自己再从这些地址中选择出一个合适的服务器。Ribbon就属于后者，它只是一个类库，集成于消费方进程，消费方通过它来获取到服务提供方的地址。

# Ribbon的主要组件与工作流程

微服务架构的核心思想是，一个应用是由多个小的、相互独立的、微服务组成，这些服务运行在自己的进程中，开发和发布都没有依赖。
不同服务通过一些轻量级交互机制来通信，例如 RPC、HTTP 等，服务可独立扩展伸缩，每个服务定义了明确的边界，不同的服务甚至可以采用不同的编程语言来实现，由独立的团队来维护。
简单的来说，一个系统的不同模块转变成不同的服务！而且服务可以使用不同的技术加以实现！

# Ribbon的核心组件

**均为接口类型,有以下几个**

**ServerList** 

 - 用于获取地址列表。它既可以是静态的(提供一组固定的地址)，也可以是动态的(从注册中心中定期查询地址列表)。

**ServerListFilter** 

 - 仅当使用动态ServerList时使用，用于在原始的服务列表中使用一定策略过虑掉一部分地址。

**IRule** 

 - 选择一个最终的服务地址作为LB结果。选择策略有轮询、根据响应时间加权、断路器(当Hystrix可用时)等。

Ribbon在工作时首选会通过ServerList来获取所有可用的服务列表，然后通过ServerListFilter过虑掉一部分地址，最后在剩下的地址中通过IRule选择出一台服务器作为最终结果。
 
# Ribbon提供的主要负载均衡策略介绍

**简单轮询负载均衡（RoundRobin）**

以轮询的方式依次将请求调度不同的服务器，即每次调度执行i = (i + 1) mod n，并选出第i台服务器。

**随机负载均衡 （Random）**

随机选择状态为UP的Server

**加权响应时间负载均衡 （WeightedResponseTime）**

根据相应时间分配一个weight，相应时间越长，weight越小，被选中的可能性越低。

**区域感知轮询负载均衡（ZoneAvoidanceRule）**

复合判断server所在区域的性能和server的可用性选择server


# 准备工作

本次项目示例，改造第一篇文章中的项目,使用`spring-cloud-eureka-service`作为服务注册中心，`spring-cloud-eureka-provider`,复制三分，项目名称依次修改为`spring-cloud-eureka-provider-1` `[1-3]`

## 改造 Provider

**服务提供者**

在项目：`spring-cloud-eureka-provider-1`，`spring-cloud-eureka-provider-2`，`spring-cloud-eureka-provider-3` 的启动类，都加入` @Value("${server.port}")`，修改`home()`方法， 来区分不同端口的`Controller` 响应，因为接下来，使用`ribbon`做均衡需要测试需要使用到

```java
package io.ymq.example.eureka.provider;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@EnableEurekaClient
@RestController
public class EurekaProviderApplication {

    @Value("${server.port}")
    String port;

    @RequestMapping("/")
    public String home() {
        return "Hello world ,port:" + port;
    }

    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class, args);
    }
}

```

## 修改配置

在项目：`spring-cloud-eureka-provider-1`，`spring-cloud-eureka-provider-2`，`spring-cloud-eureka-provider-3`,修改`server: port:`端口依次为`8081`,`8082`,`8083`

```sh
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: eureka-provider

server:
  port: 8081
```

# Ribbon Consumer

**服务消费者**

## 添加依赖

新建 `spring-cloud-ribbon-consumer` 

```xml
<!-- 客户端负载均衡 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>

<!-- eureka客户端 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

## 开启服务负载均衡

在工程的启动类中,通过`@EnableDiscoveryClient`向服务注册中心注册；并且向程序的`ioc`注入一个`bean: restTemplate`并通过`@LoadBalanced`注解表明这个`restRemplate`开启负载均衡的功能。

```java
package io.ymq.example.ribbon.consumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@EnableDiscoveryClient
@SpringBootApplication
public class RibbonConsumerApplication {

    @LoadBalanced
    @Bean
    RestTemplate restTemplate() {
        return new RestTemplate();
    }

	public static void main(String[] args) {
		SpringApplication.run(RibbonConsumerApplication.class, args);
	}
}

```

## 消费提供者方法

新建 `ConsumerController` 类，调用提供者的 `hello` 方法

```java
package io.ymq.example.ribbon.consumer;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

/**
 * 描述:调用提供者的 `home` 方法
 *
 * @author yanpenglei
 * @create 2017-12-05 18:53
 **/
@RestController
public class ConsumerController {

    @Autowired
    private RestTemplate restTemplate;

    @GetMapping(value = "/hello")
    public String hello() {
        return restTemplate.getForEntity("http://eureka-provider/", String.class).getBody();
    }
}

```

## 添加配置

完整配置 `application.yml`

指定服务的注册中心地址，配置自己的服务端口，服务名称

```sh
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: ribbon-consumer

server:
  port: 9000
```

# 测试服务 

## 启动服务

依次启动项目：

`spring-cloud-eureka-service`  
`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  
`spring-cloud-ribbon-consumer`  

启动该工程后，访问服务注册中心，查看服务是否都已注册成功：[http://localhost:8761/](http://localhost:8761/) 

![查看服务注册状态][1]

## 负载均衡

**在命令窗口`curl http://localhost:9000/hello`，发现Ribbon已经实现负载均衡**

或者浏览器`get` 请求`http://localhost:9000/hello` F5 刷新

![测试Ribbon，负载均衡响应][2]

[1]: http://www.ymq.io/images/2017/SpringCloud/ribbon/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/ribbon/2.png

## 源码下载

**GitHub：**[https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-ribbon](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-ribbon)  

**码云：**[https://gitee.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-ribbon](https://gitee.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-ribbon)  

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

 
