---
layout: post
title: Spring Boot 中使用 RabbitMQ
categories: [RabbitMQ,SpringBoot]
description: Spring Boot 中使用 RabbitMQ
keywords: RabbitMQ 
---

RabbitMQ是一个开源的AMQP实现，服务器端用Erlang语言编写，支持多种客户端，如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持AJAX。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

AMQP，即Advanced message Queuing Protocol，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。

AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。

# 准备

## 环境安装 

**任选其一**

[CentOs7.3 搭建 RabbitMQ 3.6 单机服务与使用](https://segmentfault.com/a/1190000010693696)

[CentOs7.3 搭建 RabbitMQ 3.6 Cluster 集群服务与使用](https://segmentfault.com/a/1190000010702020)

# 测试用例

# Github 代码

代码我已放到 Github ，导入`ymq-rabbitmq-spring-boot` 项目 

github [https://github.com/souyunku/ymq-example/tree/master/ymq-rabbitmq-spring-boot](https://github.com/souyunku/ymq-example/tree/master/ymq-rabbitmq-spring-boot)

## 添加依赖

在项目中添加 `spring-boot-starter-amqp` 依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

## 配置队列

```java
@Configuration
public class YmqBaseQueue {

    @Bean
    public Queue helloQueue() {
        return new Queue("hello");
    }
}
```

## 发送消息


```java
@Component
public class Sender {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    public void send(String context) {

        this.rabbitTemplate.convertAndSend("hello", context + ":" + new Date());
    }
}
```


## 监听队列

```java
@Component
@RabbitListener(queues = "hello")
public class Receiver {

    @RabbitHandler
    public void process(String hello) {
        System.out.println("消费成功 Receiver : " + hello);
    }
}
```

## 参数配置

```java
spring.application.name=ymq-rabbitmq-spring-boot

spring.rabbitmq.host=10.4.98.15
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
```

## 单元测试

```java
import io.ymq.rabbitmq.Sender;
import io.ymq.rabbitmq.run.Startup;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * 描述:
 *
 * @author yanpenglei
 * @create 2017-10-16 17:06
 **/
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Startup.class)
public class BaseTest {

    @Autowired
    private Sender sender;

    @Test
    public void test() throws Exception {

        sender.send("www.ymq.io");
    }
}
```

响应

```
消费成功 Receiver : www.ymq.io:Mon Oct 16 17:39:58 CST 2017
```

代码我已放到 Github ，导入 `ymq-rabbitmq-spring-boot` 项目 

github [https://github.com/souyunku/ymq-example/tree/master/ymq-rabbitmq-spring-boot](https://github.com/souyunku/ymq-example/tree/master/ymq-rabbitmq-spring-boot)



