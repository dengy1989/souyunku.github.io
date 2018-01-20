---
layout: post
title: Spring Cloud（十）高可用的分布式配置中心 Spring Cloud Config 中使用 Refresh
categories: SpringCloud
description: Spring Cloud（十）高可用的分布式配置中心 Spring Cloud Config 中使用 Refresh
keywords: SpringCloud 
---

上一篇文章讲了`SpringCloudConfig` 集成`Git`仓库，配和 `Eureka` 注册中心一起使用，但是我们会发现，修改了`Git`仓库的配置后，需要重启服务，才可以得到最新的配置，这一篇我们尝试使用 `Refresh` 实现主动获取 `Config Server` 配置服务中心的最新配置

# 准备工作

把上一篇，示例代码下载，才可以进行一下的操作，下载地址在文章末尾

`spring-cloud-eureka-service`  
`spring-cloud-config-server`  
`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  
`spring-cloud-feign-consumer`  

# Config Client

修改第九篇文章项目

`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  
 
# 添加依赖

```xml
<!-- actuator 监控 -->
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

# 安全认证

在 `application.properties` 添加以下配置.关闭安全认证

```sh
#关闭刷新安全认证
management.security.enabled=false
```

值是`false`的话，除开`health`接口还依赖`endpoints.health.sensitive`的配置外，其他接口都不需要输入用户名和密码了

# 开启 refresh

在程序的启动类 `EurekaProviderApplication` 通过 `@RefreshScope` 开启 SpringCloudConfig 客户端的 `refresh` 刷新范围，来获取服务端的最新配置，`@RefreshScope`要加在声明`@Controller`声明的类上，否则`refres`h之后`Conroller`拿不到最新的值，会默认调用缓存。

```java
package io.ymq.example.eureka.provider;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RefreshScope
@RestController
@EnableEurekaClient
@SpringBootApplication
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
 
# 测试服务

按照顺序依次启动项目

`spring-cloud-eureka-service`  
`spring-cloud-config-server`  
`spring-cloud-eureka-provider-1`  
`spring-cloud-eureka-provider-2`  
`spring-cloud-eureka-provider-3`  
`spring-cloud-feign-consumer`  

 启动该工程后，访问服务注册中心，查看服务是否都已注册成功：[http://localhost:8761/](http://localhost:8761/) 
 
![查看服务注册情况][11]



## 修改Git仓库

修改`Git`仓库配置，在 `content=hello dev` 后面加个 `123456`
  
![修改Git仓库][22]

## 访问服务

命令窗口，通过`curl http://127.0.0.1:9000/hello` 访问服务，或者在浏览器访问`http://127.0.0.1:9000/hello` F5 刷新

**发现没有得到最新的值**

![访问服务][33]

# 刷新配置

通过 `Postman` 发送 `POST`请求到：[http://localhost:8081/refresh](http://localhost:8081/refresh)，[http://localhost:8083/refresh](http://localhost:8083/refresh)，我们可以看到以下内容：

![刷新配置][44]

## 访问服务

命令窗口，通过`curl http://127.0.0.1:9000/hello` 访问服务，或者在浏览器访问`http://127.0.0.1:9000/hello` F5 刷新

**发现：服务8082 没有刷新到最新配置** 因为没有手动触发更新

![访问服务][55]

# 源码下载

**GitHub：**[https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-eureka-refresh](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-eureka-refresh)

**码云：**[https://gitee.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-eureka-refresh](https://gitee.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-eureka-refresh)

[11]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/11.png
[22]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/22.png
[33]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/33.png
[44]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/44.png
[55]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/55.png

# 下篇预告

留了一个悬念,`Config Client` 实现配置的实时更新，我们可以使用 `/refresh` 接口触发，如果所有配置的更改，都需要手动触发，那岂不是维护成本很高，而使用	`Spring Cloud Bus` 消息总线实现方案，可以优雅的解决以上问题，下篇文章我们讲`Spring Cloud Bus` 的使用，关注下文章末尾公众号，支持下作者，感谢

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2017/12/23/spring-cloud-config-eureka-refresh](http://www.ymq.io/2017/12/23/spring-cloud-config-eureka-refresh/o)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

