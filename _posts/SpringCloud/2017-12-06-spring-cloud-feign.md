---
layout: post
title:  Spring Cloud（四）服务提供者 Eureka + 服务消费者 Feign
categories: SpringCloud
description: Spring Cloud（四）服务提供者 Eureka + 服务消费者 Feign
keywords: SpringCloud 
---

上一篇文章，讲述了如何通过RestTemplate+Ribbon去消费服务，这篇文章主要讲述如何通过Feign去消费服务。

# 一、Feign简介

Feign是一个声明式的伪Http客户端，它使得写Http客户端变得更简单。使用Feign，只需要创建一个接口并注解。它具有可插拔的注解特性，可使用Feign 注解和JAX-RS注解。Feign支持可插拔的编码器和解码器。Feign默认集成了Ribbon，并和Eureka结合，默认实现了负载均衡的效果。

Feign 具有如下特性：

 - 可插拔的注解支持，包括Feign注解和JAX-RS注解
 - 支持可插拔的HTTP编码器和解码器
 - 支持Hystrix和它的Fallback
 - 支持Ribbon的负载均衡
 - 支持HTTP请求和响应的压缩 Feign是一个声明式的Web Service客户端，它的目的就是让Web Service调用更加简单。它整合了Ribbon和Hystrix，从而不再需要显式地使用这两个组件。Feign还提供了HTTP请求的模板，通过编写简单的接口和注解，就可以定义好HTTP请求的参数、格式、地址等信息。接下来，Feign会完全代理HTTP的请求，我们只需要像调用方法一样调用它就可以完成服务请求。

 
简而言之：Feign能干Ribbon和Hystrix的事情，但是要用Ribbon和Hystrix自带的注解必须要引入相应的jar包才可以。
 
## Eureka Server

**提供服务注册和发现服务**

### 添加依赖

在项目 `spring-cloud-eureka-service` `pom.xml`中引入需要的依赖内容：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

### 开启服务注册

通过 `@EnableEurekaServer` 注解启动一个服务注册中心提供给其他应用进行对话,这个注解需要在springboot工程的启动application类上加

```java
package io.ymq.example.eureka;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaApplication {

	public static void main(String[] args) {
		SpringApplication.run(EurekaApplication.class, args);
	}
}
```

### 配置 Eureka 服务

在默认设置下，该服务注册中心也会将自己作为客户端来尝试注册它自己，所以我们需要禁用它的客户端注册行为，只需要在`application.yml`配置文件中增加如下信息：

```sh
registerWithEureka: false
fetchRegistry: false
```

完整配置

```sh
server:
  port: 8761

eureka:
  instance:
    hostname: localhost
  client:
    registerWithEureka: false
    fetchRegistry: false
    serviceUrl:
      defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/
```

### 启动服务注册中心


启动工程后，访问：[http://localhost:8761/](http://localhost:8761/)

可以看到下面的页面，其中还没有发现任何服务。

![ System Status ][1]

## Eureka Provider

**服务提供者**

- 将自身服务注册到 `Eureka Service`，从而使服务消费方能够找到


### 添加依赖

在项目 `spring-cloud-eureka-provider` `pom.xml`中引入需要的依赖内容：

```xml
<!-- spring boot eureka server -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```

### 开启服务注册

在应用主类中通过加上 @EnableEurekaClient，但只有Eureka可用，你也可以使用@EnableDiscoveryClient。需要配置才能找到Eureka服务器

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

### 配置 Eureka Server 服务

需要配置才能找到Eureka服务器。例：

完整配置 `application.yml`

```sh
eureka:
  client:
    serviceUrl:
      defaultZone: http://localhost:8761/eureka/

spring:
  application:
    name: eureka-provider

server:
  port: 8762
```

### 启动提供者

启动该工程后，再次访问启动工程后：[http://localhost:8761/](http://localhost:8761/)

可以如下图内容，我们定义的服务被成功注册了。

![ eureka-provider][2]

## 多个 Eureka Provider

修改 `spring-cloud-eureka-provider` 配置文件`application.yml` 里的端口：`8762` 改成 `8763` maven 打包 后发布


**maven 打包**

```sh
F:\spring-cloud-examples\spring-cloud-eureka-provider> mvn clean package

....
Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO]
[INFO] --- maven-jar-plugin:2.6:jar (default-jar) @ spring-cloud-eureka-provider ---
[INFO] Building jar: F:\spring-cloud-examples\spring-cloud-eureka-provider\target\spring-cloud-eureka-provider-0.0.1-SNAPSHOT.jar
[INFO]
[INFO] --- spring-boot-maven-plugin:1.5.9.RELEASE:repackage (default) @ spring-cloud-eureka-provider ---
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 23.848 s
[INFO] Finished at: 2017-12-05T20:26:25+08:00
[INFO] Final Memory: 47M/421M
[INFO] ------------------------------------------------------------------------
F:\spring-cloud-examples\spring-cloud-eureka-provider>
```


**把 maven打好的包，放入不同的目录，本地发布**
```sh
java -jar spring-cloud-eureka-provider-0.0.1-SNAPSHOT.jar
```

![发布 eureka-provider 8762 端口][3]

![发布 eureka-provider 8763 端口][4]

### 查看提供者服务

启动该工程后，再次访问启动工程后：[http://localhost:8761/](http://localhost:8761/)


![查看提供者服务][5]

## Feign Consumer

**服务消费者**

### 添加依赖

在项目 `spring-cloud-feign-consumer` `pom.xml`中引入需要的依赖内容：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-feign</artifactId>
</dependency>
```

### 开启服务负载均衡

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


### 定义一个Feign接口

通过`@FeignClient（“服务名”）`，来指定调用哪个服务。比如在代码中调用了`eureka-provider`服务的 `/` 接口，`/` 就是调用：服务提供者项目：`spring-cloud-eureka-provider`  `home()` 方法，代码如下：

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

### 消费提供者方法

写一个 `controller`，调用提供者的 `home` 方法

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

### 配置 Feign 服务

完整配置 `application.yml`

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

### 启动消费者

启动该工程后，再次访问：[http://localhost:8761/](http://localhost:8761/) 


![查看 Ribbon 注册状态][6]

### 负载均衡响应


**浏览器 F5 刷新，发现已经实现负载均衡**

![测试提供者，负载均衡响应][7]

![测试提供者，负载均衡响应][8]

[1]: /images/2017/SpringCloud/feign/1.png
[2]: /images/2017/SpringCloud/feign/2.png
[3]: /images/2017/SpringCloud/feign/3.png
[4]: /images/2017/SpringCloud/feign/4.png
[5]: /images/2017/SpringCloud/feign/5.png
[6]: /images/2017/SpringCloud/feign/6.png
[7]: /images/2017/SpringCloud/feign/7.png
[8]: /images/2017/SpringCloud/feign/8.png


**注意：spring-cloud-eureka-provider** 项目，改成不同端口发布两次

## 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-feign-consumer](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-feign-consumer)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - GitHub：[https://github.com/souyunku](https://github.com/souyunku)  
   
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

 
