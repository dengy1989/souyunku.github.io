---
layout: post
title: Spring Cloud（八）高可用的分布式配置中心 Spring Cloud Config
categories: SpringCloud
description: Spring Cloud（八）高可用的分布式配置中心 Spring Cloud Config
keywords: SpringCloud 
---

在分布式系统中，由于服务数量巨多，为了方便服务配置文件统一管理，实时更新，所以需要分布式配置中心组件。在Spring Cloud中，有分布式配置中心组件spring cloud config ，它支持配置服务放在配置服务的内存中（即本地），也支持放在远程Git仓库中。在spring cloud config 组件中，分两个角色，一是config server，二是config client，业界也有些知名的同类开源产品，比如百度的disconf。

相比较同类产品，SpringCloudConfig最大的优势是和Spring无缝集成，支持Spring里面Environment和PropertySource的接口，对于已有的Spring应用程序的迁移成本非常低，在配置获取的接口上是完全一致，结合SpringBoot可使你的项目有更加统一的标准（包括依赖版本和约束规范），避免了应为集成不同开软件源造成的依赖版本冲突。

# Spring Cloud Config 简介

SpringCloudConfig就是我们通常意义上的配置中心，把应用原本放在本地文件的配置抽取出来放在中心服务器，从而能够提供更好的管理、发布能力。SpringCloudConfig分服务端和客户端，服务端负责将git（svn）中存储的配置文件发布成REST接口，客户端可以从服务端REST接口获取配置。但客户端并不能主动感知到配置的变化，从而主动去获取新的配置，这需要每个客户端通过POST方法触发各自的/refresh。

SpringCloudBus通过一个轻量级消息代理连接分布式系统的节点。这可以用于广播状态更改（如配置更改）或其他管理指令。SpringCloudBus提供了通过POST方法访问的endpoint/bus/refresh，这个接口通常由git的钩子功能调用，用以通知各个SpringCloudConfig的客户端去服务端更新配置。

注意：这是工作的流程图，实际的部署中SpringCloudBus并不是一个独立存在的服务，这里单列出来是为了能清晰的显示出工作流程。

下图是SpringCloudConfig结合SpringCloudBus实现分布式配置的工作流

![SpringCloudConfig结合SpringCloudBus实现分布式配置的工作流][1]

# 服务端配置

## config Server
**新建项目** `spring-cloud-config-server`

## 添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

## 开启服务注册

在程序的启动类 `ConfigApplication` 通过 `@EnableConfigServer` 开启 SpringCloudConfig 服务端

```java
package io.ymq.example.config.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.config.server.EnableConfigServer;

@EnableConfigServer
@SpringBootApplication
public class ConfigApplication {

    public static void main(String[] args) {
		SpringApplication.run(ConfigApplication.class, args);
	}
}

```

## 添加配置

配置文件 `application.properties`

```sh
spring.application.name=config-server
server.port=8888
spring.cloud.config.label=master
spring.cloud.config.server.git.uri=https://github.com/souyunku/spring-cloud-config.git
spring.cloud.config.server.git.search-paths=spring-cloud-config

#spring.cloud.config.server.git.username=your username
#spring.cloud.config.server.git.password=your password
```

 - spring.cloud.config.server.git.uri：配置git仓库地址
 - spring.cloud.config.server.git.searchPaths：配置仓库路径
 - spring.cloud.config.label：配置仓库的分支
 - spring.cloud.config.server.git.username：访问git仓库的用户名
 - spring.cloud.config.server.git.password：访问git仓库的用户密码

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

## config Client

**新建项目** `spring-cloud-config-client`

## 添加依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-config-client</artifactId>
</dependency>

<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

## 开启服务注册

在程序的启动类 `ConfigClientApplication` 通过 `@Value` 获取服务端的	`content` 值的内容

```java
package io.ymq.example.config.client;

@RestController
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

配置文件 `application.properties`

```sh
spring.application.name=config-client
server.port=8088

spring.cloud.config.label=master
spring.cloud.config.profile=dev
spring.cloud.config.uri=http://localhost:8888/
```
 - spring.cloud.config.label 指明远程仓库的分支
 - spring.cloud.config.profile
 - dev开发环境配置文件
 - test测试环境
 - pro正式环境
 - spring.cloud.config.uri= http://localhost:8888/ 指明配置服务中心的网址。

## 测试服务

启动程序 `ConfigClientApplication` 类

访问服务：[http://localhost:8088/](http://localhost:8088/)

![访问服务][4]

**下一篇，继续Spring Cloud Config 整合 eureka, 等更多特性**	
 
[1]: http://www.ymq.io/images/2017/SpringCloud/config/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/config/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/config/3.png
[4]: http://www.ymq.io/images/2017/SpringCloud/config/4.png

# 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-server](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-server)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-client](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-client)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
   
   
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

