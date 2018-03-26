---
layout: post
title: Spring Boot 中使用 RabbitMQ
categories: [RabbitMQ,SpringBoot]
description: Spring Boot 中使用 RabbitMQ
keywords: RabbitMQ 
---

`RabbitMQ`是一个开源的`AMQP`实现，服务器端用Erlang语言编写，支持多种客户端，如：`Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP`等，支持AJAX。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

`AMQP`，即`Advanced message Queuing` `Protocol`，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。

`AMQP`的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。

RabbitMQ是一个开源的AMQP实现，服务器端用Erlang语言编写，支持多种客户端，如：`Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP`等，支持`AJAX`。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。


# 常用概念

通常我们谈到队列服务, 会有三个概念： 发消息者、队列、收消息者，`RabbitMQ` 在这个基本概念之上, 多做了一层抽象, 在发消息者和 队列之间, 加入了交换器 (`Exchange`). 这样发消息者和队列就没有直接联系, 转而变成发消息者把消息给交换器, 交换器根据调度策略再把消息再给队列。



# 准备

## 环境安装 

**任选其一**

[CentOs7.3 搭建 RabbitMQ 3.6 单机服务与使用](https://segmentfault.com/a/1190000010693696)

[CentOs7.3 搭建 RabbitMQ 3.6 Cluster 集群服务与使用](https://segmentfault.com/a/1190000010702020)

# Github 代码

代码我已放到 Github ，导入`spring-boot-rabbitmq` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rabbitmq](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rabbitmq)


![ 项目结构 ][1]

## 添加依赖

在项目中添加 `spring-boot-starter-amqp` 依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

## 参数配置

```java
spring.application.name=ymq-rabbitmq-spring-boot

spring.rabbitmq.host=10.4.98.15
spring.rabbitmq.port=5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
```

## 交换机(Exchange) 

**1.Direct Exchange** 根据route key 直接找到队列  
**2.Topic Exchange** 根据route key 匹配队列  
**3.Topic Exchange** 不处理route key 全网发送，所有绑定的队列都发送  


## Direct Exchange

![ Direct Exchange 图解][2]

`Direct Exchange` 是`RabbitMQ`默认的交换机模式，也是最简单的模式，根据`key`全文匹配去寻找队列。

**任何发送到`Direct Exchange`的消息都会被转发到`RouteKey`中指定的`Queue`。** 
 
1.一般情况可以使用`rabbitMQ`自带的`Exchange：""`(该`Exchange`的名字为空字符串，下文称其为`default Exchange`)。    
2.这种模式下不需要将`Exchange`进行任何绑定(`binding`)操作    
3.消息传递时需要一个`RouteKey`，可以简单的理解为要发送到的队列名字。    
4.如果`vhost`中不存在`RouteKey`中指定的队列名，则该消息会被抛弃。    
 

### 配置队列

```java
@Configuration
public class RabbitDirectConfig {

    @Bean
    public Queue helloQueue() {
        return new Queue("hello");
    }

    @Bean
    public Queue directQueue() {
        return new Queue("direct");
    }

    //-------------------配置默认的交换机模式，可以不需要配置以下-----------------------------------
    @Bean
    DirectExchange directExchange() {
        return new DirectExchange("directExchange");
    }

    //绑定一个key "direct"，当消息匹配到就会放到这个队列中
    @Bean
    Binding bindingExchangeDirectQueue(Queue directQueue, DirectExchange directExchange) {
        return BindingBuilder.bind(directQueue).to(directExchange).with("direct");
    }
    // 推荐使用 helloQueue（） 方法写法，这种方式在 Direct Exchange 模式 多此一举，没必要这样写
    //---------------------------------------------------------------------------------------------
}

```

### 监听队列

```java
@Component
@RabbitListener(queues = "hello")
public class helloReceiver {

    @RabbitHandler
    public void process(String message) {
        System.out.println("接收者 helloReceiver," + message);
    }
}
```

```java
@Component
@RabbitListener(queues = "direct")
public class DirectReceiver {

    @RabbitHandler
    public void process(String message) {
        System.out.println("接收者 DirectReceiver," + message);
    }
}

```
### 发送消息

```java
package io.ymq.rabbitmq.test;

import io.ymq.rabbitmq.run.Startup;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * 描述: 默认的交换机模式
 *
 * @author: yanpenglei
 * @create: 2017/10/25 1:03
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Startup.class)
public class RabbitDirectTest {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    @Test
    public void sendHelloTest() {

        String context = "此消息在，默认的交换机模式队列下，有 helloReceiver 可以收到";

        String routeKey = "hello";

        context = "routeKey:" + routeKey + ",context:" + context;

        System.out.println("sendHelloTest : " + context);

        this.rabbitTemplate.convertAndSend(routeKey, context);
    }

    @Test
    public void sendDirectTest() {

        String context = "此消息在，默认的交换机模式队列下，有 DirectReceiver 可以收到";

        String routeKey = "direct";

        String exchange = "directExchange";

        context = "context:" + exchange + ",routeKey:" + routeKey + ",context:" + context;

        System.out.println("sendDirectTest : " + context);

        // 推荐使用 sendHello（） 方法写法，这种方式在 Direct Exchange 多此一举，没必要这样写
        this.rabbitTemplate.convertAndSend(exchange, routeKey, context);
    }
}
```

按顺序执行：响应

```
接收者 helloReceiver,routeKey:hello,context:此消息在，默认的交换机模式队列下，有 helloReceiver 可以收到

接收者 DirectReceiver,context:directExchange,routeKey:direct,context:此消息在，默认的交换机模式队列下，有 DirectReceiver 可以收到
```

## Fanout Exchange 

![ Fanout Exchange 图解][3]

**任何发送到`Fanout Exchange` 的消息都会被转发到与该`Exchange`绑定`(Binding)`的所有`Queue上`。**
  
1.可以理解为路由表的模式  
2.这种模式不需要 `RouteKey`  
3.这种模式需要提前将`Exchange`与`Queue`进行绑定，一个`Exchange`可以绑定多个`Queue`，一个`Queue`可以同多个`Exchange`进行绑定。  
4.如果接受到消息的`Exchange`没有与任何`Queue`绑定，则消息会被抛弃。  


### 配置队列

```java
@Configuration
public class RabbitFanoutConfig {

    final static String PENGLEI = "fanout.penglei.net";

    final static String SOUYUNKU = "fanout.souyunku.com";
    @Bean
    public Queue queuePenglei() {
        return new Queue(RabbitFanoutConfig.PENGLEI);
    }

    @Bean
    public Queue queueSouyunku() {
        return new Queue(RabbitFanoutConfig.SOUYUNKU);
    }

    /**
     * 任何发送到Fanout Exchange的消息都会被转发到与该Exchange绑定(Binding)的所有队列上。
     */
    @Bean
    FanoutExchange fanoutExchange() {
        return new FanoutExchange("fanoutExchange");
    }

    @Bean
    Binding bindingExchangeQueuePenglei(Queue queuePenglei, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(queuePenglei).to(fanoutExchange);
    }

    @Bean
    Binding bindingExchangeQueueSouyunku(Queue queueSouyunku, FanoutExchange fanoutExchange) {
        return BindingBuilder.bind(queueSouyunku).to(fanoutExchange);
    }

}
```

### 监听队列

```java
@Component
@RabbitListener(queues = "fanout.penglei.net")
public class FanoutReceiver1 {

    @RabbitHandler
    public void process(String message) {
        System.out.println("接收者 FanoutReceiver1," + message);
    }
}
```

```java
@Component
@RabbitListener(queues = "fanout.souyunku.com")
public class FanoutReceiver2 {

    @RabbitHandler
    public void process(String message) {
        System.out.println("接收者 FanoutReceiver2," + message);
    }
}
```

### 发送消息

```java
package io.ymq.rabbitmq.test;

import io.ymq.rabbitmq.run.Startup;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * 描述: 广播模式或者订阅模式队列
 *
 * @author: yanpenglei
 * @create: 2017/10/25 1:08
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Startup.class)
public class RabbitFanoutTest {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    @Test
    public void sendPengleiTest() {

        String context = "此消息在，广播模式或者订阅模式队列下，有 FanoutReceiver1 FanoutReceiver2 可以收到";

        String routeKey = "topic.penglei.net";

        String exchange = "fanoutExchange";

        System.out.println("sendPengleiTest : " + context);

        context = "context:" + exchange + ",routeKey:" + routeKey + ",context:" + context;

        this.rabbitTemplate.convertAndSend(exchange, routeKey, context);
    }

    @Test
    public void sendSouyunkuTest() {

        String context = "此消息在，广播模式或者订阅模式队列下，有 FanoutReceiver1 FanoutReceiver2 可以收到";

        String routeKey = "topic.souyunku.com";

        String exchange = "fanoutExchange";

        context = "context:" + exchange + ",routeKey:" + routeKey + ",context:" + context;

        System.out.println("sendSouyunkuTest : " + context);

        this.rabbitTemplate.convertAndSend(exchange, routeKey, context);
    }
}

```


按顺序执行：响应

```
接收者 FanoutReceiver1,context:fanoutExchange,routeKey:topic.penglei.net,context:此消息在，广播模式或者订阅模式队列下，有 FanoutReceiver1 FanoutReceiver2 可以收到
接收者 FanoutReceiver2,context:fanoutExchange,routeKey:topic.penglei.net,context:此消息在，广播模式或者订阅模式队列下，有 FanoutReceiver1 FanoutReceiver2 可以收到


接收者 FanoutReceiver2,context:fanoutExchange,routeKey:topic.souyunku.com,context:此消息在，广播模式或者订阅模式队列下，有 FanoutReceiver1 FanoutReceiver2 可以收到
接收者 FanoutReceiver1,context:fanoutExchange,routeKey:topic.souyunku.com,context:此消息在，广播模式或者订阅模式队列下，有 FanoutReceiver1 FanoutReceiver2 可以收到

```

## Topic Exchange

![ Topic Exchange 图解][4]

**任何发送到`Topic Exchange`的消息都会被转发到所有关心`RouteKey`中指定话题的`Queue`上**

1.这种模式较为复杂，简单来说，就是每个队列都有其关心的主题，所有的消息都带有一个`标题``(RouteKey)`，`Exchange`会将消息转发到所有关注主题能与`RouteKey`模糊匹配的队列。  
2.这种模式需要`RouteKey`，也许要提前绑定`Exchange`与`Queue`。  
3.在进行绑定时，要提供一个该队列关心的主题，如`#.log.#`表示该队列关心所有涉及log的消息(一个RouteKey为`MQ.log.error`的消息会被转发到该队列)。  
4.`#`表示0个或若干个关键字，`*`表示一个关键字。如`topic.*`能与`topic.warn`匹配，无法与`topic.warn.timeout`匹配；但是`topic.#`能与上述两者匹配。  
5.同样，如果`Exchange`没有发现能够与`RouteKey`匹配的`Queue`，则会抛弃此消息。  

### 配置队列

```java
@Configuration
public class RabbitTopicConfig {

    final static String MESSAGE = "topic.message";

    final static String MESSAGES = "topic.message.s";

    final static String YMQ = "topic.ymq";

    @Bean
    public Queue queueMessage() {
        return new Queue(RabbitTopicConfig.MESSAGE);
    }

    @Bean
    public Queue queueMessages() {
        return new Queue(RabbitTopicConfig.MESSAGES);
    }

    @Bean
    public Queue queueYmq() {
        return new Queue(RabbitTopicConfig.YMQ);
    }

    /**
     * 交换机(Exchange) 描述：接收消息并且转发到绑定的队列，交换机不存储消息
     */
    @Bean
    TopicExchange topicExchange() {
        return new TopicExchange("topicExchange");
    }

    //綁定队列 queueMessages() 到 topicExchange 交换机,路由键只接受完全匹配 topic.message 的队列接受者可以收到消息
    @Bean
    Binding bindingExchangeMessage(Queue queueMessage, TopicExchange topicExchange) {
        return BindingBuilder.bind(queueMessage).to(topicExchange).with("topic.message");
    }

    //綁定队列 queueMessages() 到 topicExchange 交换机,路由键只要是以 topic.message 开头的队列接受者可以收到消息
    @Bean
    Binding bindingExchangeMessages(Queue queueMessages, TopicExchange topicExchange) {
        return BindingBuilder.bind(queueMessages).to(topicExchange).with("topic.message.#");
    }

    //綁定队列 queueYmq() 到 topicExchange 交换机,路由键只要是以 topic 开头的队列接受者可以收到消息
    @Bean
    Binding bindingExchangeYmq(Queue queueYmq, TopicExchange topicExchange) {
        return BindingBuilder.bind(queueYmq).to(topicExchange).with("topic.#");
    }

}

```

### 监听队列

```java
@Component
@RabbitListener(queues = "topic.message")
public class TopicReceiver1 {

    @RabbitHandler
    public void process(String message) {
        System.out.println("接收者 TopicReceiver1," + message);
    }

}
```

```java
@Component
@RabbitListener(queues = "topic.message.s")
public class TopicReceiver2 {

    @RabbitHandler
    public void process(String message) {
        System.out.println("接收者 TopicReceiver2," + message);
    }

}
```

```java
@Component
@RabbitListener(queues = "topic.ymq")
public class TopicReceiver3 {

    @RabbitHandler
    public void process(String message) {
        System.out.println("接收者 TopicReceiver3," + message);
    }

}

```


### 发送消息

```java
package io.ymq.rabbitmq.test;

import io.ymq.rabbitmq.run.Startup;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.amqp.core.AmqpTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * 描述: 配置转发消息模式队列
 *
 * @author: yanpenglei
 * @create: 2017/10/25 1:20
 */
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Startup.class)
public class RabbitTopicTest {

    @Autowired
    private AmqpTemplate rabbitTemplate;

    @Test
    public void sendMessageTest() {

        String context = "此消息在，配置转发消息模式队列下， 有 TopicReceiver1 TopicReceiver2 TopicReceiver3 可以收到";

        String routeKey = "topic.message";

        String exchange = "topicExchange";

        context = "context:" + exchange + ",routeKey:" + routeKey + ",context:" + context;

        System.out.println("sendMessageTest : " + context);

        this.rabbitTemplate.convertAndSend(exchange, routeKey, context);
    }

    @Test
    public void sendMessagesTest() {


        String context = "此消息在，配置转发消息模式队列下，有  TopicReceiver2 TopicReceiver3 可以收到";

        String routeKey = "topic.message.s";

        String exchange = "topicExchange";

        context = "context:" + exchange + ",routeKey:" + routeKey + ",context:" + context;

        System.out.println("sendMessagesTest : " + context);

        this.rabbitTemplate.convertAndSend(exchange, routeKey, context);
    }

    @Test
    public void sendYmqTest() {

        String context = "此消息在，配置转发消息模式队列下，有 TopicReceiver3 可以收到";

        String routeKey = "topic.ymq";

        String exchange = "topicExchange";

        context = "context:" + exchange + ",routeKey:" + routeKey + ",context:" + context;

        System.out.println("sendYmqTest : " + context);

        this.rabbitTemplate.convertAndSend(exchange, routeKey, context);
    }
}

```


按顺序执行：响应

```
接收者 TopicReceiver2,context:topicExchange,routeKey:topic.message,context:此消息在，配置转发消息模式队列下， 有 TopicReceiver1 TopicReceiver2 TopicReceiver3 可以收到
接收者 TopicReceiver1,context:topicExchange,routeKey:topic.message,context:此消息在，配置转发消息模式队列下， 有 TopicReceiver1 TopicReceiver2 TopicReceiver3 可以收到
接收者 TopicReceiver3,context:topicExchange,routeKey:topic.message,context:此消息在，配置转发消息模式队列下， 有 TopicReceiver1 TopicReceiver2 TopicReceiver3 可以收到


接收者 TopicReceiver3,context:topicExchange,routeKey:topic.message.s,context:此消息在，配置转发消息模式队列下，有  TopicReceiver2 TopicReceiver3 可以收到
接收者 TopicReceiver2,context:topicExchange,routeKey:topic.message.s,context:此消息在，配置转发消息模式队列下，有  TopicReceiver2 TopicReceiver3 可以收到


接收者 TopicReceiver3,context:topicExchange,routeKey:topic.ymq,context:此消息在，配置转发消息模式队列下，有 TopicReceiver3 可以收到

```

代码我已放到 Github ，导入`spring-boot-rabbitmq` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rabbitmq](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rabbitmq)

[1]: http://www.ymq.io/images/2017/rabbit/example/1.png
[2]: http://www.ymq.io/images/2017/rabbit/example/2.png
[3]: http://www.ymq.io/images/2017/rabbit/example/3.png
[4]: http://www.ymq.io/images/2017/rabbit/example/4.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2017/10/26/rabbitmq-spring-boot-example](http://www.ymq.io/2017/10/26/rabbitmq-spring-boot-example/)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")


