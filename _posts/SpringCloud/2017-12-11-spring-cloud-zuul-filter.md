---
layout: post
title: Spring Cloud（七）服务网关 Zuul Filter 使用
categories: SpringCloud
description: Spring Cloud（七）服务网关 Zuul Filter 使用
keywords: SpringCloud 
---

上一篇文章中，讲了Zuul 转发，动态路由，负载均衡，等等一些Zuul 的特性，这个一篇文章，讲Zuul Filter 使用，关于网关的作用，这里就不再次赘述了，重点是zuul的Filter ，我们可以实现安全控制，比如，只有请求参数中有token和密码的客户端才能访问服务端的资源。那么如何来实现Filter了？

# Spring Cloud Zuul

## 准备工作

在开始测试服务之前，我们先拿之前两篇博客，构建的两个微服务代码为基础，进行下面的操作，主要使用下面几个工程：

**建议先阅读以下文章**

[Spring Cloud（六）服务网关 zuul 快速入门-http://www.ymq.io/2017/12/10/spring-cloud-zuul/](http://www.ymq.io/2017/12/10/spring-cloud-zuul/)  

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider-2](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider-2)


## 简单使用

**新建项目** `spring-cloud-zuul-filter`

### 添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-zuul</artifactId>
</dependency>
```

### 开启服务注册

在程序的启动类 `ZuulFilterApplication` 通过 `@EnableZuulProxy` 开启 Zuul 服务网关

```java
package io.ymq.example.zuul.filter;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
import org.springframework.context.annotation.Bean;

@EnableZuulProxy
@SpringBootApplication
public class ZuulFilterApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulFilterApplication.class, args);
    }
}
```

### 添加配置

配置文件 `application.yml`

```sh
spring:
  application:
    name: zuul-service-filter

server:
  port: 9000

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

### TokenFilter

`ZuulFilter` 是Zuul中核心组件，通过继承该抽象类，覆写几个关键方法达到自定义调度请求的作用

**TokenFilter 过滤器**

```java
package io.ymq.example.zuul.filter;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.http.HttpServletRequest;

/**
 * 描述: 过滤器 token
 *
 * @author yanpenglei
 * @create 2017-12-11 14:38
 **/
public class TokenFilter extends ZuulFilter {

    private final Logger LOGGER = LoggerFactory.getLogger(TokenFilter.class);

    @Override
    public String filterType() {
        return "pre"; // 可以在请求被路由之前调用
    }

    @Override
    public int filterOrder() {
        return 0; // filter执行顺序，通过数字指定 ,优先级为0，数字越大，优先级越低
    }

    @Override
    public boolean shouldFilter() {
        return true;// 是否执行该过滤器，此处为true，说明需要过滤
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        LOGGER.info("--->>> TokenFilter {},{}", request.getMethod(), request.getRequestURL().toString());

        String token = request.getParameter("token");// 获取请求的参数

        if (StringUtils.isNotBlank(token)) {
            ctx.setSendZuulResponse(true); //对请求进行路由
            ctx.setResponseStatusCode(200);
            ctx.set("isSuccess", true);
            return null;
        } else {
            ctx.setSendZuulResponse(false); //不对其进行路由
            ctx.setResponseStatusCode(400);
            ctx.setResponseBody("token is empty");
            ctx.set("isSuccess", false);
            return null;
        }
    }

}
```

### PasswordFilter

`ZuulFilter` 是Zuul中核心组件，通过继承该抽象类，覆写几个关键方法达到自定义调度请求的作用

**PasswordFilter 过滤器**


```java
package io.ymq.example.zuul.filter;

import com.netflix.zuul.ZuulFilter;
import com.netflix.zuul.context.RequestContext;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import javax.servlet.http.HttpServletRequest;

/**
 * 描述: 过滤器 Password
 *
 * @author yanpenglei
 * @create 2017-12-11 15:40
 **/
public class PasswordFilter extends ZuulFilter {

    private final Logger LOGGER = LoggerFactory.getLogger(TokenFilter.class);

    @Override
    public String filterType() {
        return "post"; // 请求处理完成后执行的filter
    }

    @Override
    public int filterOrder() {
        return 1; // 优先级为0，数字越大，优先级越低
    }

    @Override
    public boolean shouldFilter() {
        RequestContext ctx = RequestContext.getCurrentContext();
        return (boolean) ctx.get("isSuccess");
        // 判断上一个过滤器结果为true，否则就不走下面过滤器，直接跳过后面的所有过滤器并返回 上一个过滤器不通过的结果。
    }

    @Override
    public Object run() {
        RequestContext ctx = RequestContext.getCurrentContext();
        HttpServletRequest request = ctx.getRequest();

        LOGGER.info("--->>> PasswordFilter {},{}", request.getMethod(), request.getRequestURL().toString());

        String username = request.getParameter("password");
        if (null != username && username.equals("123456")) {
            ctx.setSendZuulResponse(true);
            ctx.setResponseStatusCode(200);
            ctx.set("isSuccess", true);
            return null;
        } else {
            ctx.setSendZuulResponse(false);
            ctx.setResponseStatusCode(400);
            ctx.setResponseBody("The password cannot be empty");
            ctx.set("isSuccess", false);
            return null;
        }
    }
}

```

### 开启过滤器

在程序的启动类 `ZuulFilterApplication` 添加 Bean

```java
@Bean
public TokenFilter tokenFilter() {
	return new TokenFilter();
}

@Bean
public PasswordFilter PasswordFilter() {
	return new PasswordFilter();
}
```

### filterType

**filterType：返回一个字符串代表过滤器的类型，在zuul中定义了四种不同生命周期的过滤器类型，具体如下:** 

- pre：路由之前
- routing：路由之时
- post： 路由之后
- error：发送错误调用
- filterOrder：过滤的顺序
- shouldFilter：这里可以写逻辑判断，是否要过滤，本文true,永远过滤。
- run：过滤器的具体逻辑。可用很复杂，包括查sql，nosql去判断该请求到底有没有权限访问。

### 测试服务

依次启动四个服务：`spring-cloud-eureka-service`,`spring-cloud-eureka-provider`,`spring-cloud-eureka-provider-2`,`spring-cloud-zuul-service`

**查看 eureka 监控，看服务是否都注册成功**

访问:[http://127.0.0.1:8761/](http://127.0.0.1:8761/)


![浏览器访问][1]

**步骤一** 提示 `token is empty` 

访问:[http://127.0.0.1:9000/](http://127.0.0.1:9000/)

![token is empty][2]

**步骤二** 加上token `?token=token-uuid` ，已经验证通过了，提示 `The password cannot be empty` 

访问:[http://127.0.0.1:9000/?token=token-uuid](http://127.0.0.1:9000/?token=token-uuid)

![The password cannot be empty][3]

**步骤三** 加上token 和 password `&password=123456` ，已经验证通过

访问:[http://127.0.0.1:9000/?token=token-uuid&password=123456](http://127.0.0.1:9000/?token=token-uuid&password=123456)

F5 刷新，每次都验证通过，并且负载均衡

![Hello Zuul ,port:8763][4]

![THello world ,port:8762][5]

### zuul 执行流程

![执行流程图][6]

**Zuul**大部分功能都是通过过滤器来实现的。Zuul中定义了四种标准过滤器类型，这些过滤器类型对应于请求的典型生命周期。

**PRE：**这种过滤器在请求被路由之前调用。我们可利用这种过滤器实现身份验证、在集群中选择请求的微服务、记录调试信息等。

**ROUTING：**这种过滤器将请求路由到微服务。这种过滤器用于构建发送给微服务的请求，并使用Apache HttpClient或Netfilx Ribbon请求微服务。

**OST：**这种过滤器在路由到微服务以后执行。这种过滤器可用来为响应添加标准的HTTP Header、收集统计信息和指标、将响应从微服务发送给客户端等。

**ERROR：**在其他阶段发生错误时执行该过滤器。

除了默认的过滤器类型，Zuul还允许我们创建自定义的过滤器类型。例如，我们可以定制一种STATIC类型的过滤器，直接在Zuul中生成响应，而不将请求转发到后端的微服务。

                                     
[1]: http://www.ymq.io/images/2017/SpringCloud/zuulFilter/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/zuulFilter/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/zuulFilter/3.png
[4]: http://www.ymq.io/images/2017/SpringCloud/zuulFilter/4.png
[5]: http://www.ymq.io/images/2017/SpringCloud/zuulFilter/5.png
[6]: http://www.ymq.io/images/2017/SpringCloud/zuulFilter/6.png

## 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider-2](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-provider-2)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-zuul-filter](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-zuul-filter)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - GitHub：[https://github.com/souyunku](https://github.com/souyunku)  
   
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

