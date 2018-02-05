---
layout: post
title: Spring Boot 中使用 Kafka
categories: [Kafka,SpringBoot]
description: Spring Boot 中使用 Kafka
keywords: Kafka 
---

Kafka 是一种高吞吐的分布式发布订阅消息系统，能够替代传统的消息队列用于解耦合数据处理，缓存未处理消息等，同时具有更高的吞吐率，支持分区、多副本、冗余，因此被广泛用于大规模消息数据处理应用。Kafka 支持Java 及多种其它语言客户端，可与Hadoop、Storm、Spark等其它大数据工具结合使用。

# 准备

## 环境安装 

[搭建高吞吐量 Kafka 分布式发布订阅消息 集群](https://segmentfault.com/a/1190000010896062)

# 测试用例

# Github 代码

代码我已放到 Github ，导入`spring-boot-kafka` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-kafka](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-kafka)

## 添加依赖

在项目中添加 `kafka-clients` 依赖

```xml
<dependency>
	<groupId>org.apache.kafka</groupId>
	<artifactId>kafka-clients</artifactId>
	<version>0.10.2.0</version>
</dependency>
<dependency>
	<groupId>org.springframework.kafka</groupId>
	<artifactId>spring-kafka</artifactId>
</dependency>
```

## 启用 kafka

```java
@Configuration
@EnableKafka
public class KafkaConfiguration {

}
```

## 消息生产者

```java
@Component
public class MsgProducer {

    private static final Logger log = LoggerFactory.getLogger(MsgProducer.class);

    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;

    public void sendMessage(String topicName, String jsonData) {
        log.info("向kafka推送数据:[{}]", jsonData);
        try {
            kafkaTemplate.send(topicName, jsonData);
        } catch (Exception e) {
            log.error("发送数据出错！！！{}{}", topicName, jsonData);
            log.error("发送数据出错=====>", e);
        }

        //消息发送的监听器，用于回调返回信息
        kafkaTemplate.setProducerListener(new ProducerListener<String, String>() {
            @Override
            public void onSuccess(String topic, Integer partition, String key, String value, RecordMetadata recordMetadata) {
            }

            @Override
            public void onError(String topic, Integer partition, String key, String value, Exception exception) {
            }

            @Override
            public boolean isInterestedInSuccess() {
                log.info("数据发送完毕");
                return false;
            }
        });
    }

}
```


## 消息消费者

```java
@Component
public class MsgConsumer {

    @KafkaListener(topics = {"topic-1","topic-2"})
    public void processMessage(String content) {

        System.out.println("消息被消费"+content);
    }
    
}
```

## 参数配置

**`application.properties`**

```
#kafka
# 指定kafka 代理地址，可以多个
spring.kafka.bootstrap-servers=YZ-PTEST-APP-HADOOP-02:9092,YZ-PTEST-APP-HADOOP-04:9092
# 指定listener 容器中的线程数，用于提高并发量
spring.kafka.listener.concurrency=3
# 每次批量发送消息的数量
spring.kafka.producer.batch-size=1000
# 指定默认消费者group id
spring.kafka.consumer.group-id=myGroup
# 指定默认topic id
spring.kafka.template.default-topic=topic-1
```

## 启动服务

```java
@SpringBootApplication
@ComponentScan(value = {"io.ymq.kafka"})
public class Startup {

    public static void main(String[] args) {
        SpringApplication.run(Startup.class, args);
    }
}
```

## 单元测试

```java
import io.ymq.kafka.MsgProducer;
import io.ymq.kafka.run.Startup;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;
/**
 * 描述: 测试 kafka
 *
 * @author yanpenglei
 * @create 2017-10-16 18:45
 **/
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Startup.class)
public class BaseTest {

    @Autowired
    private MsgProducer msgProducer;

    @Test
    public void test() throws Exception {

        msgProducer.sendMessage("topic-1", "topic--------1");
        msgProducer.sendMessage("topic-2", "topic--------2");
    }
}
```

消息生产者,响应

```
2017-10-17 15:54:44.814  INFO 2960 --- [           main] io.ymq.kafka.MsgProducer                 : 向kafka推送数据:[topic--------1]
2017-10-17 15:54:44.860  INFO 2960 --- [           main] io.ymq.kafka.MsgProducer                 : 向kafka推送数据:[topic--------2]
2017-10-17 15:54:44.878  INFO 2960 --- [ad | producer-1] io.ymq.kafka.MsgProducer                 : 数据发送完毕
2017-10-17 15:54:44.878  INFO 2960 --- [ad | producer-1] io.ymq.kafka.MsgProducer                 : 数据发送完毕
```

消息消费者,响应
```
消息被消费topic--------1
消息被消费topic--------2
```

代码我已放到 Github ，导入`spring-boot-kafka` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-kafka](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-kafka)


## 遇到一些坑

```sh
[2017-10-16 19:20:08.340] - 14884 严重 [main] --- org.springframework.kafka.support.LoggingProducerListener: Exception thrown when sending a message with key='null' and payload='topic--------2' to topic topic-2:
```
经调试发现 kafka 连接是用的主机名，所以修改 hosts

```sh
C:\Windows\System32\drivers\etc\hosts

10.32.32.149 YZ-PTEST-APP-HADOOP-02
10.32.32.154 YZ-PTEST-APP-HADOOP-04
```

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

 