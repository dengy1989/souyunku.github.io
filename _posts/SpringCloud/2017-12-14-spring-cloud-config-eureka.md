---
layout: post
title: Spring Cloud（九）高可用的分布式配置中心 Spring Cloud Config 集成 Eureka 服务
categories: SpringCloud
description: Spring Cloud（九）高可用的分布式配置中心 Spring Cloud Config 集成 Eureka 服务
keywords: SpringCloud 
---

上一篇文章，讲了`SpringCloudConfig` 集成`Git`仓库，这一篇我们讲一下`SpringCloudConfig` 配和 `Eureka` 注册中心一起使用

在分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。在`Spring Cloud`中，有分布式配置中心组件`spring cloud config` ，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程`Git`仓库中。在`spring cloud config` 组件中，分两个角色，一是`config server`，二是`config client`，业界也有些知名的同类开源产品，比如百度的`disconf`。

相比较同类产品，`SpringCloudConfig`最大的优势是和`Spring`无缝集成，支持`Spring`里面`Environment`和P`ropertySource`的接口，对于已有的`pring`应用程序的迁移成本非常低，在配置获取的接口上是完全一致，结合`SpringBoot`可使你的项目有更加统一的标准（包括依赖版本和约束规范），避免了应为集成不同开软件源造成的依赖版本冲突。

# 准备工作

我们先拿之前的代码为基础，进行下面的操作

[Spring Cloud（四） 服务提供者 Eureka + 服务消费者 Feign ](http://www.ymq.io/2017/12/06/spring-cloud-feign/)  

[http://www.ymq.io/2017/12/06/spring-cloud-feign/](http://www.ymq.io/2017/12/06/spring-cloud-feign/)

## Eureka Service

**导入第四篇文章中的项目：作为服务注册中心**

`spring-cloud-eureka-service`

## Eureka Provider

**导入第四篇文章中的项目：作为服务的提供者**

`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  


## Eureka Consumer

**导入第四篇文章中的项目：作为服务的消费者**

`spring-cloud-feign-consumer`  

# 服务端配置

## Config Server

**复制上一篇的项目** `spring-cloud-config-server`,添加 `eureka`依赖

[https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config/](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config/)

## 添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

## 开启服务注册

在程序的启动类 `ConfigServerApplication.java` 通过 `@EnableEurekaClient` 开启 `Eureka`  提供者服务

```java
package io.ymq.example.config.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@EnableConfigServer
@EnableEurekaClient
@SpringBootApplication
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

## 修改配置

修改配置文件 `application.properties` ，添加 `eureka` 注册中心地址 `http://localhost:8761/eureka/`

```sh
spring.application.name=config-server
server.port=8888
spring.cloud.config.label=master
spring.cloud.config.server.git.uri=https://github.com/souyunku/spring-cloud-config.git
spring.cloud.config.server.git.search-paths=spring-cloud-config
#spring.cloud.config.server.git.username=your username
#spring.cloud.config.server.git.password=your password

eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
```

 - `spring.cloud.config.server.git.uri：`配置`git`仓库地址
 - `spring.cloud.config.server.git.searchPaths：`配置仓库路径
 - `spring.cloud.config.label：`配置仓库的分支
 - `spring.cloud.config.server.git.username：`访问`git`仓库的用户名
 - `spring.cloud.config.server.git.password：`访问`git`仓库的用户密码
   
 - `eureka.client.serviceUrl.defaultZone：eureka`注册中心地址

Git仓库如果是私有仓库需要填写用户名密码，示例是公开仓库，所以不配置密码。

## 远程Git仓库

`spring-cloud-config` 文件夹下有 `application-dev.properties`,`application-test.properties` 三个文件，内容依次是：`content=hello dev`,`content=hello test`,`content=hello pre`

![远程Git仓库][3]


## 测试服务

启动程序 `ConfigServerApplication` 类

访问 `Config Server`  服务：[http://localhost:8888/springCloudConfig/dev/master](http://localhost:8888/springCloudConfig/dev/master)

```json
{
    "name": "springCloudConfig",
    "profiles": [
        "dev"
    ],
    "label": "master",
    "version": "b6fbc2f77d1ead41d5668450e2601a03195eaf16",
    "state": null,
    "propertySources": [
        {
            "name": "https://github.com/souyunku/spring-cloud-config.git/application-dev.properties",
            "source": {
                "content": "hello dev"
            }
        }
    ]
}
```

证明配置服务中心可以从远程程序获取配置信息。

http请求地址和资源文件映射如下:

 - `/{application}/{profile}[/{label}]`
 - `/{application}-{profile}.yml`
 - `/{label}/{application}-{profile}.yml`
 - `/{application}-{profile}.properties`
 - `/{label}/{application}-{profile}.properties`


# 客户端端配置

## config Client Eureka

**修改已经导入的，第四篇文章中的项目：配置客户端的一些配置**

`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  

## 添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-client</artifactId>
</dependency>
```

## 开启服务注册

在程序的启动类 `EurekaProviderApplication` ，通过 `@Value` 获取服务端的	`content` 值的内容

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

    @Value("${content}")
    String content;

    @Value("${server.port}")
    String port;

    @RequestMapping("/")
    public String home() {
        return "Hello world ,port:" + port+",content="+content;
    }

    public static void main(String[] args) {
        SpringApplication.run(EurekaProviderApplication.class, args);
    }
}
```

## 添加配置

修改配置文件 `application.properties` 添加 `Eureka` 注册中心，配置从`springCloudConfig` 配置中心读取配置，指定`springCloudConfigService` 服务名称

```sh
spring.application.name=eureka-provider
server.port=8081

spring.cloud.config.label=master
spring.cloud.config.profile=dev
#spring.cloud.config.uri=http://localhost:8888/

eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=config-server
```

- `spring.cloud.config.label` 指明远程仓库的分支
- `spring.cloud.config.profile`
- `dev`开发环境配置文件
- `test`测试环境
- `pro`正式环境
- `#spring.cloud.config.uri= http://localhost:8888/` 指明配置服务中心的网址**（注释掉）**
  
- `spring.cloud.config.discovery.enabled=true` 是从配置中心读取文件。
- `spring.cloud.config.discovery.serviceId=config-server`  配置中心的`servieId`，服务名称，通过服务名称去 `Eureka`注册中心找服务
 
## 测试服务

依次启动项目：

`spring-cloud-eureka-service`  
`spring-cloud-config-server`  
`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  
`spring-cloud-feign-consumer`  

启动该工程后，访问服务注册中心，查看服务是否都已注册成功：[http://localhost:8761/](http://localhost:8761/) 

![查看各个服务注册状态][1]

**查看 eureka 监控，看服务是否都注册成功**

命令窗口，通过`curl http://127.0.0.1:9000/hello` 访问服务，或者在浏览器访问`http://127.0.0.1:9000/hello` F5 刷新

![查看各个服务注册状态][2]

**修改了Git仓库的配置后，需要重启服务，才可以得到最新的配置，下一篇讲怎么解决配置的更新**

# 源码下载

**GitHub：**[https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-eureka](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-eureka)

**码云：**[https://gitee.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-eureka](https://gitee.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-eureka)

[1]: http://www.ymq.io/images/2017/SpringCloud/config-eureka/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/config-eureka/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/config-eureka/3.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

