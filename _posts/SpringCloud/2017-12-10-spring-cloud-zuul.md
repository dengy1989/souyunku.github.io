---
layout: post
title: Spring Cloud（六）服务网关 zuul 快速入门
categories: SpringCloud
description: Spring Cloud（六）服务网关 zuul 快速入门
keywords: SpringCloud 
---

服务网关是微服务架构中一个不可或缺的部分。通过服务网关统一向外系统提供REST API的过程中，除了具备服务路由、均衡负载功能之外，它还具备了权限控制等功能。Spring Cloud Netflix中的Zuul就担任了这样的一个角色，为微服务架构提供了前门保护的作用，同时将权限控制这些较重的非业务逻辑内容迁移到服务路由层面，使得服务集群主体能够具备更高的可复用性和可测试性。

路由在微服务体系结构的一个组成部分。例如，/可以映射到您的Web应用程序，`/api/users`映射到用户服务，并将`/api/shop`映射到商店服务。`Zuul`是`Netflix`的基于JVM的路由器和服务器端负载均衡器。

**Netflix使用Zuul进行以下操作：**

 - 认证  
 - 洞察  
 - 压力测试   
 - 金丝雀测试  
 - 动态路由  
 - 服务迁移  
 - 负载脱落  
 - 安全  
 - 静态响应处理    
 - 主动/主动流量管理  

Zuul的规则引擎允许基本上写任何JVM语言编写规则和过滤器，内置Java和Groovy。

# 什么是服务网关

**服务网关 = 路由转发 + 过滤器**

1、路由转发：接收一切外界请求，转发到后端的微服务上去；

2、过滤器：在服务网关中可以完成一系列的横切功能，例如权限校验、限流以及监控等，这些都可以通过过滤器完成（其实路由转发也是通过过滤器实现的）。

# 为什么需要服务网关

**上述所说的横切功能（以权限校验为例）可以写在三个位置：**

 - 每个服务自己实现一遍
 - 写到一个公共的服务中，然后其他所有服务都依赖这个服务
 - 写到服务网关的前置过滤器中，所有请求过来进行权限校验

**第一种，缺点太明显，基本不用；**
**第二种，相较于第一点好很多，代码开发不会冗余，但是有两个缺点：**

 - 由于每个服务引入了这个公共服务，那么相当于在每个服务中都引入了相同的权限校验的代码，使得每个服务的jar包大小无故增加了一些，尤其是对于使用docker镜像进行部署的场景，jar越小越好；
 - 由于每个服务都引入了这个公共服务，那么我们后续升级这个服务可能就比较困难，而且公共服务的功能越多，升级就越难，而且假设我们改变了公共服务中的权限校验的方式，想让所有的服务都去使用新的权限校验方式，我们就需要将之前所有的服务都重新引包，编译部署。
 
**而服务网关恰好可以解决这样的问题：**

 - 将权限校验的逻辑写在网关的过滤器中，后端服务不需要关注权限校验的代码，所以服务的jar包中也不会引入权限校验的逻辑，不会增加jar包大小；
 - 如果想修改权限校验的逻辑，只需要修改网关中的权限校验过滤器即可，而不需要升级所有已存在的微服务。
 
**所以，需要服务网关！！！**

# 服务网关技术选型

![服务网关][1]

**引入服务网关后的微服务架构如上，总体包含三部分：服务网关、open-service和service。**

**1、总体流程：**

 - 服务网关、open-service和service启动时注册到注册中心上去；
 - 用户请求时直接请求网关，网关做智能路由转发（包括服务发现，负载均衡）到open-service，这其中包含权限校验、监控、限流等操作
 - open-service聚合内部service响应，返回给网关，网关再返回给用户
 
**2、引入网关的注意点**

 - 增加了网关，多了一层转发（原本用户请求直接访问open-service即可），性能会下降一些（但是下降不大，通常，网关机器性能会很好，而且网关与open-service的访问通常是内网访问，速度很快）；
 - 网关的单点问题：在整个网络调用过程中，一定会有一个单点，可能是网关、nginx、dns服务器等。防止网关单点，可以在网关层前边再挂一台nginx，nginx的性能极高，基本不会挂，这样之后，网关服务就可以不断的添加机器。但是这样一个请求就转发了两次，所以最好的方式是网关单点服务部署在一台牛逼的机器上（通过压测来估算机器的配置），而且nginx与zuul的性能比较，根据国外的一个哥们儿做的实验来看，其实相差不大，zuul是netflix开源的一个用来做网关的开源框架；
 - 网关要尽量轻。
 
**3、服务网关基本功能**

 - 智能路由：接收外部一切请求，并转发到后端的对外服务open-service上去；
 - 注意：我们只转发外部请求，服务之间的请求不走网关，这就表示全链路追踪、内部服务API监控、内部服务之间调用的容错、智能路由不能在网关完成；当然，也可以将所有的服务调用都走网关，那么几乎所有的功能都可以集成到网关中，但是这样的话，网关的压力会很大，不堪重负。
 - 权限校验：只校验用户向open-service服务的请求，不校验服务内部的请求。服务内部的请求有必要校验吗？
 - API监控：只监控经过网关的请求，以及网关本身的一些性能指标（例如，gc等）；
 - 限流：与监控配合，进行限流操作；
 - API日志统一收集：类似于一个aspect切面，记录接口的进入和出去时的相关日志
 - 。。。后续补充
 
**4、技术选型**

笔者准备自建一个轻量级的服务网关，技术选型如下：

 - 开发语言：java + groovy，groovy的好处是网关服务不需要重启就可以动态的添加filter来实现一些功能；  
 - 微服务基础框架：springboot；  
 - 网关基础组件：netflix zuul；  
 - 服务注册中心：consul；   
 - 权限校验：jwt；  
 - API监控：prometheus + grafana；  
 - API统一日志收集：logback + ELK；  
 - 压力测试：Jmeter；  
 - 。。。后续补充  
 - 在后续的介绍中，会逐渐介绍各个知识点，并完成一个轻量级的服务网关！！！  
 
# Spring Cloud Zuul

## 简单使用

**新建项目** `spring-cloud-zuul-service`

### 添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```

### 开启服务注册

在程序的启动类 `ZuulApplication` 通过 `@EnableZuulProxy` 开启 Zuul 服务网关

```java
package io.ymq.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

@EnableZuulProxy
@SpringBootApplication
public class ZuulApplication {

	public static void main(String[] args) {
		SpringApplication.run(ZuulApplication.class, args);
	}
}
```

### 添加配置

配置文件 `application.yml`

```sh
spring:
  application:
    name: zuul-service

server:
  port: 9000

zuul:
  routes:
    blog:
        path: /ymq/**
        url: http://www.ymq.io/
```


### 测试访问

**配置说明：**

浏览器访问:[http://127.0.0.1:9000/ymq](http://127.0.0.1:9000/ymq) 重定向到我的博客

![浏览器访问][2]

## 服务转发

### 添加依赖

项目继续改造，添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

### 修改配置

配置文件 `application.yml`

```sh
zuul:
  routes:
    api:
        path: /**
        serviceId: eureka-provider

eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/
```

**配置说明：**

浏览器访问:[http://127.0.0.1:9000/](http://127.0.0.1:9000/) ,Zuul 会去 Eureka 服务注册中心，找到`eureka-provider`服务以均衡负载的方式访问

### 准备工作

在开始测试服务之前，我们先拿之前两篇博客，构建的两个微服务代码为基础，进行下面的操作，主要使用下面几个工程：

**建议先阅读以下两篇文章**

[Spring Cloud（四） 服务提供者 Eureka + 服务消费者 Feign](http://www.ymq.io/2017/12/06/spring-cloud-feign/)  
[Spring Cloud（三） 服务提供者 Eureka + 服务消费者（rest + Ribbon）](http://www.ymq.io/2017/12/05/spring-cloud-ribbon-rest/)  

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider)

复制项目`spring-cloud-eureka-provider` 改为`spring-cloud-eureka-provider-2`  为了更好的体现 Zuul 访问服务负载均衡

**修改** `EurekaProviderApplication.java`  home 方法的返回值

```java
@SpringBootApplication
@EnableEurekaClient
@RestController
public class EurekaProviderApplication {

    @Value("${server.port}")
    String port;

    @RequestMapping("/")
    public String home() {
        return "Hello Zuul ,port:" + port;
    }

    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class, args);
    }
}
```

**修改** `application.yml` 修改一下提供服务的端口,`8762` 改成8763

```sh
server:
  port: 8763
```

### 测试服务

依次启动四个服务：`spring-cloud-eureka-service`,`spring-cloud-eureka-provider`,`spring-cloud-eureka-provider-2`,`spring-cloud-zuul-service`

**查看 eureka 监控，看服务是否都注册成功**

![浏览器访问][3]

**浏览器访问**

访问:[http://127.0.0.1:9000/](http://127.0.0.1:9000/) ,Zuul 会去 Eureka 服务注册中心，找到`eureka-provider`服务以均衡负载的方式访问

`F5 刷新`

![F5刷新浏览器访问][4]
![F5刷新浏览器访问][5]

### 网关的默认路由规则

spring cloud zuul 默认情况下，`Zuul`会代理所有注册到`Eureka Server`的微服务，并且Zuul的路由规则如下：[http://ZUUL_HOST:ZUUL_PORT/]() 微服务在`Eureka`上的`serviceId/**`会被转发到`serviceId`对应的微服务。

我们注释 `spring-cloud-zuul-service`项目中关于路由的配置：

```sh
#zuul:
#  routes:
#    api:
#        path: /**
#        serviceId: eureka-provider
```

**浏览器访问**

访问:[http://127.0.0.1:9000/eureka-provider/](http://127.0.0.1:9000/eureka-provider/) ,Zuul 会去 Eureka 服务注册中心，找到`eureka-provider`服务以均衡负载的方式访问

![F5刷新浏览器访问][6]
![F5刷新浏览器访问][7]

## ZuulFilter

在下一章，会深入介绍 Zuul 高级功能使用，`ZuulFilter` ,支持下鹏磊，关注下屏幕下方的微信公众号
                                               
[1]: http://www.ymq.io/images/2017/SpringCloud/zuul/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/zuul/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/zuul/3.png
[4]: http://www.ymq.io/images/2017/SpringCloud/zuul/4.png
[5]: http://www.ymq.io/images/2017/SpringCloud/zuul/5.png
[6]: http://www.ymq.io/images/2017/SpringCloud/zuul/6.png
[7]: http://www.ymq.io/images/2017/SpringCloud/zuul/7.png
[8]: http://www.ymq.io/images/2017/SpringCloud/zuul/8.png
[9]: http://www.ymq.io/images/2017/SpringCloud/zuul/9.png


## 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider-2](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider-2)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-zuul-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-zuul-service)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")
