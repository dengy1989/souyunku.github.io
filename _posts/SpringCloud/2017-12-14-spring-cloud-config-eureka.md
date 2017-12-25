---
layout: post
title: Spring Cloud（九）高可用的分布式配置中心 Spring Cloud Config 集成 Eureka 服务
categories: SpringCloud
description: Spring Cloud（九）高可用的分布式配置中心 Spring Cloud Config 集成 Eureka 服务
keywords: SpringCloud 
---

上一篇文章，讲了SpringCloudConfig 集成Git仓库，这一篇我们讲一下 SpringCloudConfig 配和 Eureka 注册中心一起使用

在分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。在Spring Cloud中，有分布式配置中心组件spring cloud config ，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程Git仓库中。在spring cloud config 组件中，分两个角色，一是config server，二是config client，业界也有些知名的同类开源产品，比如百度的disconf。

相比较同类产品，SpringCloudConfig最大的优势是和Spring无缝集成，支持Spring里面Environment和PropertySource的接口，对于已有的Spring应用程序的迁移成本非常低，在配置获取的接口上是完全一致，结合SpringBoot可使你的项目有更加统一的标准（包括依赖版本和约束规范），避免了应为集成不同开软件源造成的依赖版本冲突。

# 准备工作

##  Eureka Service

Eureka 注册中心，就使用第三篇文章的源码

**项目：spring-cloud-eureka-service** 下载地址在文章末尾

[Spring Cloud（八）高可用的分布式配置中心 Spring Cloud Config](http://www.ymq.io/2017/12/05/spring-cloud-ribbon-rest/#eureka-server)

[http://www.ymq.io/2017/12/05/spring-cloud-ribbon-rest/#eureka-server](http://www.ymq.io/2017/12/05/spring-cloud-ribbon-rest/#eureka-server)


# 服务端配置

## config Server Eureka


**复制上一篇的项目** `spring-cloud-config-server` 修改项目名称为：`spring-cloud-config-server-eureka-provider`

## 添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

## 开启服务注册

在程序的启动类 `ConfigApplication` 通过 `@EnableConfigServer` 开启 SpringCloudConfig 服务端，通过 `@EnableEurekaClient` 开启 Eureka 提供者服务

```java
package io.ymq.example.config.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@EnableConfigServer
@EnableEurekaClient
@SpringBootApplication
public class ConfigApplication {

    public static void main(String[] args) {
		SpringApplication.run(ConfigApplication.class, args);
	}
}

```

## 修改配置

修改配置文件 `application.properties` ，添加 eureka 注册中心地址 `http://localhost:8761/eureka/`

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

 - spring.cloud.config.server.git.uri：配置git仓库地址
 - spring.cloud.config.server.git.searchPaths：配置仓库路径
 - spring.cloud.config.label：配置仓库的分支
 - spring.cloud.config.server.git.username：访问git仓库的用户名
 - spring.cloud.config.server.git.password：访问git仓库的用户密码
 
 - eureka.client.serviceUrl.defaultZone：eureka注册中心地址

Git仓库如果是私有仓库需要填写用户名密码，示例是公开仓库，所以不配置密码。

**远程Git仓库**

`spring-cloud-config` 文件夹下有 `application-dev.properties`,`application-test.properties` 三个文件，内容依次是：`content=hello dev`,`content=hello test`,`content=hello pre`

![远程Git仓库][2]

## 测试服务

启动程序 `ConfigApplication` 类

访问 Spring Cloud Config 服务：[http://localhost:8888/springCloudConfig/dev/master](http://localhost:8888/springCloudConfig/dev/master)

![远程Git仓库 springCloudConfig 配置][3]
     
证明配置服务中心可以从远程程序获取配置信息。

http请求地址和资源文件映射如下:

 - `/{application}/{profile}[/{label}]`
 - `/{application}-{profile}.yml`
 - `/{label}/{application}-{profile}.yml`
 - `/{application}-{profile}.properties`
 - `/{label}/{application}-{profile}.properties`


# 客户端端配置

## config Client Eureka

**复制上一篇的项目** `spring-cloud-config-client`  修改项目名称为：`spring-cloud-config-client-consumer`

## 添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```

## 开启服务注册

在程序的启动类 `ConfigClientApplication` 通过 `@EnableConfigServer` 开启 SpringCloudConfig 服务端，通过 `@EnableEurekaClient` 开启 Eureka 提供者服务

```java
package io.ymq.example.config.client;

@RestController
@EnableEurekaClient
@SpringBootApplication
public class ConfigClientApplication {

    @Value("${content}")
    String content;

    @RequestMapping("/")
    public String home() {
        return "content:" + content;
    }

    public static void main(String[] args) {
        SpringApplication.run(ConfigClientApplication.class, args);
    }
}
```

## 添加配置

修改配置文件 `application.properties` 添加 Eureka 注册中心，配置从springCloudConfig 配置中心读取配置，指定springCloudConfigService 服务名称

```sh
spring.application.name=config-client
server.port=8088

spring.cloud.config.label=master
spring.cloud.config.profile=dev
#spring.cloud.config.uri=http://localhost:8888/

eureka.client.serviceUrl.defaultZone=http://localhost:8761/eureka/
spring.cloud.config.discovery.enabled=true
spring.cloud.config.discovery.serviceId=config-server
```

 - spring.cloud.config.label 指明远程仓库的分支
 - spring.cloud.config.profile
 - dev开发环境配置文件
 - test测试环境
 - pro正式环境
 - #spring.cloud.config.uri= http://localhost:8888/ 指明配置服务中心的网址**（注释掉）**

 - spring.cloud.config.discovery.enabled=true 是从配置中心读取文件。
 - spring.cloud.config.discovery.serviceId=config-server  配置中心的servieId，服务名称，通过服务名称去 Eureka注册中心找服务
 
## 测试服务

启动 `spring-cloud-eureka-service` ,`spring-cloud-config-server-eureka-provider` ,`spring-cloud-config-client-consumer`  三个项目

![查看服务注册情况][5]

访问服务:[http://localhost:8088/](http://localhost:8088/)


![访问服务][4]

[1]: http://www.ymq.io/images/2017/SpringCloud/config/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/config/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/config/3.png
[4]: http://www.ymq.io/images/2017/SpringCloud/config/4.png
[5]: http://www.ymq.io/images/2017/SpringCloud/config/5.png


# 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-server-eureka-provider](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-server-eureka-provider)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-client-consumer](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-client-consumer)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

