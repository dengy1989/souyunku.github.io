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

 - `spring-cloud-eureka-service`
 - `spring-cloud-config-server-eureka-provider`
 - `spring-cloud-config-client-consumer`  

# Config Client

修改第九篇文章  Config Client **项目：spring-cloud-config-client-consumer** 


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
management.security.enabled=false
```

值是`false`的话，除开`health`接口还依赖`endpoints.health.sensitive`的配置外，其他接口都不需要输入用户名和密码了

# 开启 refresh

在程序的启动类 `ConfigClientApplication` 通过 `@RefreshScope` 开启 SpringCloudConfig 客户端的 `refresh` 刷新范围，来获取服务端的最新配置，`@RefreshScope`要加在声明`@Controller`声明的类上，否则`refres`h之后`Conroller`拿不到最新的值，会默认调用缓存。

```java
package io.ymq.example.config.client;

@RefreshScope
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
 
# 测试服务

启动 `spring-cloud-eureka-service` ,`spring-cloud-config-server-eureka-provider` ,`spring-cloud-config-client-consumer`  三个项目

![查看服务注册情况][1]

## 修改配置

修改`Git`仓库配置，在 `content=hello dev` 后面加个 `123456`

![修改git 仓库配置][3]

## 查看 Config Server

通过 `Postman` 发送 `GET` 请求到：[http://localhost:8888/springCloudConfig/dev/master](http://localhost:8888/springCloudConfig/dev/master) 查看 `Config Server` 是否是最新的值

![Config Server 已经是新的值][4]

## 查看 Config Client

访问：[http://localhost:8088/](http://localhost:8088/) ,发现没有任何改变,由此可见，配置资源的更新不能即时通知到 Server Client

![访问服务][2]

## 刷新配置

通过 `Postman` 发送 `POST`请求到：[http://localhost:8088/refresh](http://localhost:8088/refresh) ，我们可以看到以下内容：

**注意是 `PSOT` 请求**

![访问：http://localhost:8088/refresh][5]

## 再次查看 Config Client

访问：[http://localhost:8088/](http://localhost:8088/) 已经刷新了配置

![访问服务][6]

[1]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/1.png
[2]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/2.png
[3]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/3.png
[4]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/4.png
[5]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/5.png
[6]: http://www.ymq.io/images/2017/SpringCloud/config-refresh/6.png

# 源码下载

- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-eureka-service)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-server-eureka-provider](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-server-eureka-provider)
- [https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-client-consumer](https://github.com/souyunku/spring-cloud-examples/tree/master/spring-cloud-config-client-consumer)

# 下篇预告

留了一个悬念,`Config Client` 实现配置的实时更新，我们可以使用 `/refresh` 接口触发，如果所有配置的更改，都需要手动触发，那岂不是维护成本很高，而使用	`Spring Cloud Bus` 消息总线实现方案，可以优雅的解决以上问题，下篇文章我们讲`Spring Cloud Bus` 的使用，关注下文章末尾公众号，支持下作者，感谢


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

