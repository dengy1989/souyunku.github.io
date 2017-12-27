---
layout: post
title: Spring Cloud（四）服务提供者 Eureka + 服务消费者 Feign
categories: SpringCloud
description: Spring Cloud（四）服务提供者 Eureka + 服务消费者 Feign
keywords: SpringCloud 
---

上一篇文章，讲述了如何通过`RestTemplate + Ribbon`去消费服务，这篇文章主要讲述如何通过`Feign`去消费服务。

# Feign简介

`Feign`是一个声明式的伪`Http`客户端，它使得写`Http`客户端变得更简单。

使用`Feign`，只需要创建一个接口并注解，它具有可插拔的注解特性，可使用`Feign` 注解和`JAX-RS`注解，`Feign`支持可插拔的编码器和解码器，`Feign`默认集成了`Ribbon`，并和`Eureka`结合，默认实现了负载均衡的效果。

**`Feign` 具有如下特性：**

- 可插拔的注解支持，包括`Feign`注解和`JAX-RS`注解
- 支持可插拔的`HTTP`编码器和解码器
- 支持`Hystrix`和它的`Fallback`
- 支持`Ribbon`的负载均衡
- 支持`HTTP`请求和响应的压缩`Feign`是一个声明式的`Web Service`客户端，它的目的就是让`Web Service`调用更加简单。它整合了`Ribbon`和`Hystrix`，从而不再需要显式地使用这两个组件。`Feign`还提供了`HTTP`请求的模板，通过编写简单的接口和注解，就可以定义好`HTTP`请求的参数、格式、地址等信息。接下来，`Feign`会完全代理`HTTP`的请求，我们只需要像调用方法一样调用它就可以完成服务请求。

简而言之：`Feign`能干`Ribbon`和`Hystrix`的事情，但是要用`Ribbon`和`Hystrix`自带的注解必须要引入相应的`jar`包才可以。
 
# 准备工作

## Eureka Service

**导入第三篇文章中的项目：作为服务注册中心**

`spring-cloud-eureka-service`

## Eureka Provider

**导入第三篇文章中的项目：作为服务的提供者**

`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  


# Feign Consumer

**服务消费者**

## 添加依赖

新建项目 `spring-cloud-feign-consumer` `pom.xml`中引入需要的依赖内容：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

## 开启Feign

在工程的启动类中,通过`@EnableFeignClients` 注解开启Feign的功能：

```java
package io.ymq.example.feign.consumer;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.feign.EnableFeignClients;

@EnableFeignClients
@EnableDiscoveryClient
@SpringBootApplication
public class FeignConsumerApplication {

	public static void main(String[] args) {
		SpringApplication.run(FeignConsumerApplication.class, args);
	}
}

```

## 定义接口

通过`@FeignClient（"服务名"）`，来指定调用哪个服务。  
比如在代码中调用了`eureka-provider`服务的 `/` 接口，`/` 就是调用：服务提供者项目：`spring-cloud-eureka-provider-1`，`spring-cloud-eureka-provider-2`，`spring-cloud-eureka-provider-3` 的  `home()` 方法，代码如下：

```java
package io.ymq.example.feign.consumer;

import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

/**
 * 描述: 指定这个接口所要调用的 提供者服务名称 "eureka-provider"
 *
 * @author yanpenglei
 * @create 2017-12-06 15:13
 **/
@FeignClient("eureka-provider")
public interface  HomeClient {

    @GetMapping("/")
    String consumer();
}
```

## 消费方法

写一个 `Controller`，消费提供者的 `home` 方法

```
package io.ymq.example.feign.consumer;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * 描述:调用提供者的 `home` 方法
 *
 * @author yanpenglei
 * @create 2017-12-06 15:26
 **/
@RestController
public class ConsumerController {

    @Autowired
    private HomeClient homeClient;

    @GetMapping(value = "/hello")
    public String hello() {
        return  homeClient.consumer();
    }
}

```

## 添加配置

完整配置 `application.yml`

指定注册中心地址，配置自己的服务名称

```sh
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: feign-consumer

server:
  port: 9000
```

# 测试服务

依次启动项目：

`spring-cloud-eureka-service`  
`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  
`spring-cloud-feign-consumer`  

启动该工程后，访问服务注册中心，查看服务是否都已注册成功：[http://localhost:8761/](http://localhost:8761/) 

![查看各个服务注册状态][1]

## 负载均衡响应

**在命令窗口`curl http://localhost:9000/hello`，发现Ribbon已经实现负载均衡**

或者浏览器`get` 请求`http://localhost:9000/hello` F5 刷新

![测试 Feign 负载均衡响应][2]

# 源码下载

**GitHub：**[https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-feign](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-feign)  

**码云：**[https://gitee.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-feign](https://gitee.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-feign)

[1]: http://www.ymq.io/images/2017/SpringCloud/feign/11.png
[2]: http://www.ymq.io/images/2017/SpringCloud/feign/22.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

 
