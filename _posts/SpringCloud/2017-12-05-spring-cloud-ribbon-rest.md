---
layout: post
title:  Spring Cloud（三）服务提供者 Eureka + 服务消费者（rest + Ribbon）
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

`均为接口类型,有以下几个：`

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

## Ribbon Consumer

**服务消费者**

### 添加依赖

在项目 `spring-cloud-ribbon-consumer` `pom.xml`中引入需要的依赖内容：

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

### 开启服务负载均衡

在工程的启动类中,通过@EnableDiscoveryClient向服务中心注册；并且向程序的ioc注入一个bean: restTemplate;并通过@LoadBalanced注解表明这个restRemplate开启负载均衡的功能。

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

### 消费提供者方法

写一个 `controller`，调用提供者的 `home` 方法

```
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

### 配置 Ribbon 服务

完整配置 `application.yml`

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

### 启动消费者

启动该工程后，再次访问：[http://localhost:8761/](http://localhost:8761/) 


![查看 Ribbon 注册状态][6]

### 负载均衡响应


**浏览器 F5 刷新，发现已经实现负载均衡**

![测试提供者，负载均衡响应][7]

![测试提供者，负载均衡响应][8]


[1]: http://www.ymq.io/images/2017/SpringCloud/ribbon/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/ribbon/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/ribbon/3.png
[4]: http://www.ymq.io/images/2017/SpringCloud/ribbon/4.png
[5]: http://www.ymq.io/images/2017/SpringCloud/ribbon/5.png
[6]: http://www.ymq.io/images/2017/SpringCloud/ribbon/6.png
[7]: http://www.ymq.io/images/2017/SpringCloud/ribbon/7.png
[8]: http://www.ymq.io/images/2017/SpringCloud/ribbon/8.png


**注意：spring-cloud-eureka-provider** 项目，改成不同端口发布两次

## 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-ribbon-consumer](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-ribbon-consumer)



# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

 
