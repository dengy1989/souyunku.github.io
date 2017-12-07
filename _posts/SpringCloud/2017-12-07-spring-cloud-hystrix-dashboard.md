---
layout: post
title: Spring Cloud（五）断路器监控(Hystrix Dashboard)
categories: SpringCloud
description: Spring Cloud（五）断路器监控(Hystrix Dashboard)
keywords: SpringCloud 
---

在上两篇文章中讲了，服务提供者 Eureka + 服务消费者 Feign，服务提供者 Eureka + 服务消费者（rest + Ribbon），本篇文章结合，上两篇文章中代码进行修改加入 断路器监控(Hystrix Dashboard)

在微服务架构中，根据业务来拆分成一个个的服务，服务与服务之间可以相互调用（RPC），在Spring Cloud可以用RestTemplate+Ribbon和Feign来调用。为了保证其高可用，单个服务通常会集群部署。由于网络原因或者自身的原因，服务并不能保证100%可用，如果单个服务出现问题，调用这个服务就会出现线程阻塞，此时若有大量的请求涌入，Servlet容器的线程资源会被消耗完毕，导致服务瘫痪。服务与服务之间的依赖性，故障会传播，会对整个微服务系统造成灾难性的严重后果，这就是服务故障的“雪崩”效应。

针对上述问题，在Spring Cloud Hystrix中实现了线程隔离、断路器等一系列的服务保护功能。它也是基于Netflix的开源框架 Hystrix实现的，该框架目标在于通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备了服务降级、服务熔断、线程隔离、请求缓存、请求合并以及服务监控等强大功能。

# 什么是断路器

断路器模式源于Martin Fowler的Circuit Breaker一文。“断路器”本身是一种开关装置，用于在电路上保护线路过载，当线路中有电器发生短路时，“断路器”能够及时的切断故障电路，防止发生过载、发热、甚至起火等严重后果。

在分布式架构中，断路器模式的作用也是类似的，当某个服务单元发生故障（类似用电器发生短路）之后，通过断路器的故障监控（类似熔断保险丝），向调用方返回一个错误响应，而不是长时间的等待。这样就不会使得线程因调用故障服务被长时间占用不释放，避免了故障在分布式系统中的蔓延。

# 断路器示意图

SpringCloud Netflix实现了断路器库的名字叫Hystrix. 在微服务架构下，通常会有多个层次的服务调用. 下面是微服架构下, 浏览器端通过API访问后台微服务的一个示意图：

![ hystrix 1][8]

一个微服务的超时失败可能导致瀑布式连锁反映，下图中，Hystrix通过自主反馈实现的断路器， 防止了这种情况发生。

![ hystrix 2][9]

图中的服务B因为某些原因失败，变得不可用，所有对服务B的调用都会超时。当对B的调用失败达到一个特定的阀值(5秒之内发生20次失败是Hystrix定义的缺省值), 链路就会被处于open状态， 之后所有所有对服务B的调用都不会被执行， 取而代之的是由断路器提供的一个表示链路open的Fallback消息.  Hystrix提供了相应机制，可以让开发者定义这个Fallbak消息.

open的链路阻断了瀑布式错误， 可以让被淹没或者错误的服务有时间进行修复。这个fallback可以是另外一个Hystrix保护的调用, 静态数据，或者合法的空值. Fallbacks可以组成链式结构，所以，最底层调用其它业务服务的第一个Fallback返回静态数据.

# 准备工作

在开始加入断路器之前，我们先拿之前两篇博客，构建的两个微服务代码为基础，进行下面的操作，主要使用下面几个工程：

**建议先阅读以下两篇文章**

[Spring Cloud（四） 服务提供者 Eureka + 服务消费者 Feign](http://www.ymq.io/2017/12/06/spring-cloud-feign/)  
[Spring Cloud（三） 服务提供者 Eureka + 服务消费者（rest + Ribbon）](http://www.ymq.io/2017/12/05/spring-cloud-ribbon-rest/)  


- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider)

**主要修改以下项目**

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-feign-consumer](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-feign-consumer)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-ribbon-consumer](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-ribbon-consumer)




## Ribbon Hystrix

**首先启动，`spring-cloud-eureka-service`,`spring-cloud-eureka-provider` 项目**

### 修改依赖

复制 `spring-cloud-ribbon-consumer` 项目,修改名称为`spring-cloud-ribbon-consumer-hystrix` 在项目 `pom.xml`中引入需要的依赖内容：

```xml
<!-- hystrix 断路器 -->
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>
```

### 开启服务注册

在程序的启动类 `RibbonConsumerApplication` 通过 `@EnableHystrix` 开启 Hystrix 断路器监控

```java
package io.ymq.example.ribbon.consumer.hystrix;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@EnableHystrix
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

修改 `ConsumerController` 类的，`hello` 方法，加上注解`@HystrixCommand(fallbackMethod = "defaultStores")`  该注解对该方法创建了熔断器的功能
,并指定了`defaultStores`熔断方法，熔断方法直接返回了一个字符串， `"feign + hystrix ,提供者服务挂了"`


@HystrixCommand 表明该方法为hystrix包裹，可以对依赖服务进行隔离、降级、快速失败、快速重试等等hystrix相关功能 
该注解属性较多，下面讲解其中几个

 - fallbackMethod 降级方法
 - commandProperties 普通配置属性，可以配置HystrixCommand对应属性，例如采用线程池还是信号量隔离、熔断器熔断规则等等
 - ignoreExceptions 忽略的异常，默认HystrixBadRequestException不计入失败
 - groupKey() 组名称，默认使用类名称
 - commandKey 命令名称，默认使用方法名


```java
package io.ymq.example.ribbon.consumer.hystrix;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
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

    @HystrixCommand(fallbackMethod = "defaultStores")
    @GetMapping(value = "/hello")
    public String hello() {
        return restTemplate.getForEntity("http://eureka-provider/", String.class).getBody();
    }

    public String defaultStores() {
        return "feign + hystrix ,提供者服务挂了";
    }

}
```

### 测试断路器

启动工程后，访问：[http://127.0.0.1:9000/](http://127.0.0.1:9000/)

- 访问：[http://127.0.0.1:9000/](http://127.0.0.1:9000/) ,发现一切正常

![eureka-provider 提供者服务响应][1]


**停止 eureka-provider  服务**

- 再次访问[http://127.0.0.1:9000/](http://127.0.0.1:9000/) ,断路器已经生效

![feign + hystrix ,提供者服务挂了][2]

## Feign Hystrix

在 Feign中使用断路器

**首先启动，`spring-cloud-eureka-service`,`spring-cloud-eureka-provider` 项目**

### 修改配置

复制`spring-cloud-feign-consumer` 项目,修改名称为`spring-cloud-feign-consumer-hystrix` 修改配置文件，

Feign是自带断路器的，在D版本的Spring Cloud中，它没有默认打开。需要在配置文件中配置打开它

```sh
feign:
  hystrix:
    enabled: true
```

### 开启服务注册

修改 HomeClient 类 ，`@FeignClient` 注解，加上fallback的指定类就行了

在程序的启动类 `RibbonConsumerApplication` 通过 `@EnableHystrix` 开启 Hystrix

```java
package io.ymq.example.feign.consumer.hystrix;

import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;

/**
 * 描述: 指定这个接口所要调用的 提供者服务名称 "eureka-provider"
 *
 * @author yanpenglei
 * @create 2017-12-06 15:13
 **/
@FeignClient(value ="eureka-provider",fallbackFactory = HystrixClientFallbackFactory.class)
public interface  HomeClient {

    @GetMapping("/")
    String consumer();
}
```

**新加的类 `HystrixClientFallbackFactory`**

```java
package io.ymq.example.feign.consumer.hystrix;

import feign.hystrix.FallbackFactory;
import org.springframework.stereotype.Component;

/**
 * 描述:
 *
 * @author yanpenglei
 * @create 2017-12-07 20:37
 **/
@Component
public class HystrixClientFallbackFactory implements FallbackFactory<HomeClient> {

    @Override
    public HomeClient create(Throwable throwable) {
        return () -> "feign + hystrix ,提供者服务挂了";
    }
}

```


### 测试断路器

启动工程后，访问：[http://127.0.0.1:9000/](http://127.0.0.1:9000/)

- 访问：[http://127.0.0.1:9000/](http://127.0.0.1:9000/) ,发现一切正常

![eureka-provider 提供者服务响应][3]

**停止 eureka-provider  服务**

- 再次访问[http://127.0.0.1:9000/](http://127.0.0.1:9000/) ,断路器已经生效

![feign + hystrix ,提供者服务挂了][4]


## Hystrix Dashboard

### Hystrix Dashboard简介

在微服务架构中为例保证程序的可用性，防止程序出错导致网络阻塞，出现了断路器模型。断路器的状况反应了一个程序的可用性和健壮性，它是一个重要指标。Hystrix Dashboard是作为断路器状态的一个组件，提供了数据监控和友好的图形化界面。


### 改造项目
 
修改 `spring-cloud-ribbon-consumer-hystrix`  在它的基础上进行改造。 Feign的改造和这一样。

在pom的工程文件引入相应的依赖：

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
</dependency>
```


修改 `RibbonConsumerApplication` 类

在程序的入口`RibbonConsumerApplication`类，加上@EnableHystrix注解开启断路器，这个是必须的，并且需要在程序中声明断路点@HystrixCommand；加上@EnableHystrixDashboard注解，开启HystrixDashboard

```java
package io.ymq.example.ribbon.consumer.hystrix;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@EnableHystrix
@EnableDiscoveryClient
@EnableHystrixDashboard
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

声明断路点 `@HystrixCommand(fallbackMethod = "defaultStores")`  

```java
package io.ymq.example.ribbon.consumer.hystrix;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
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

    @HystrixCommand(fallbackMethod = "defaultStores")
    @GetMapping(value = "/hello")
    public String hello() {
        return restTemplate.getForEntity("http://eureka-provider/", String.class).getBody();
    }

    public String defaultStores() {
        return "feign + hystrix Dashboard ,提供者服务挂了";
    }

}

```

@HystrixCommand 表明该方法为hystrix包裹，可以对依赖服务进行隔离、降级、快速失败、快速重试等等hystrix相关功能 
该注解属性较多，下面讲解其中几个

 - fallbackMethod 降级方法
 - commandProperties 普通配置属性，可以配置HystrixCommand对应属性，例如采用线程池还是信号量隔离、熔断器熔断规则等等
 - ignoreExceptions 忽略的异常，默认HystrixBadRequestException不计入失败
 - groupKey() 组名称，默认使用类名称
 - commandKey 命令名称，默认使用方法名


### 测试 Hystrix Dashboard

运行工程

**停止 eureka-provider 服务**

- 访问[http://127.0.0.1:9000/](http://127.0.0.1:9000/) ,断路器已经生效

![feign + hystrix Dashboard ,提供者服务挂了][5]

**Dashboard 监控**

- 可以访问 [http://127.0.0.1:9090/hystrix.stream](http://127.0.0.1:9090/hystrix) ,获取dashboard信息，默认最大打开5个终端获取监控信息，可以增加delay参数指定获取监控数据间隔时间

![ hystrix stream][6]

在界面依次输入：`http://127.0.0.1:9090/hystrix` 、`2000` 、`hello` 点确定。可以访问以下，图形化监控页面

![ hystrix 图形化监控页面][7]


[1]: http://www.ymq.io/images/2017/SpringCloud/hystrix/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/hystrix/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/hystrix/3.png
[4]: http://www.ymq.io/images/2017/SpringCloud/hystrix/4.png
[5]: http://www.ymq.io/images/2017/SpringCloud/hystrix/5.png
[6]: http://www.ymq.io/images/2017/SpringCloud/hystrix/6.png
[7]: http://www.ymq.io/images/2017/SpringCloud/hystrix/7.png
[8]: http://www.ymq.io/images/2017/SpringCloud/hystrix/8.png
[9]: http://www.ymq.io/images/2017/SpringCloud/hystrix/9.png


## 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-hystrix/spring-cloud-ribbon-consumer-hystrix](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-hystrix/spring-cloud-ribbon-consumer-hystrix)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-hystrix/spring-cloud-feign-consumer-hystrix](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-hystrix/spring-cloud-feign-consumer-hystrix)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-hystrix/spring-cloud-ribbon-consumer-hystrix-dashboard](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-hystrix/spring-cloud-ribbon-consumer-hystrix-dashboard)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - GitHub：[https://github.com/souyunku](https://github.com/souyunku)  
 - Segment Fault：[http://sf.gg/blog/souyunku](http://sf.gg/blog/souyunku)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

 
