---
layout: post
title: Spring Boot 中使用 RocketMQ
categories: [RocketMQ,SpringBoot]
description: Spring Boot 中使用 RocketMQ
keywords: RocketMQ
---

本文快速入门,RocketMQ消息系统的安装部署,发送,和接收消息,监控消息，的详细说明。

# 环境需要

64位操作系统，建议使用Linux / Unix / 

- CentOs7.3
- 64bit JDK 1.8+
- Maven 3.2.x
- Git 1.8.3.1

# 环境安装

**请参考我的另一篇文章**

[搭建 Apache RocketMQ 单机环境](http://www.ymq.io/2018/02/01/RocketMQ-install/)

[http://www.ymq.io/2018/02/01/RocketMQ-install](http://www.ymq.io/2018/02/01/RocketMQ-install/)

# 新加项目

新建一个 maven 项目，这里就不详细操作了，大家都会的

不过也可以下载我的示例源码，下载地址如下

GitHub 源码：[https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rocketmq](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rocketmq)

# 添加依赖

在POM 中添加如下依赖

```xml
<!-- RocketMq客户端相关依赖 -->
<dependency>
	<groupId>org.apache.rocketmq</groupId>
	<artifactId>rocketmq-client</artifactId>
	<version>4.1.0-incubating</version>
</dependency>

<dependency>
	<groupId>org.apache.rocketmq</groupId>
	<artifactId>rocketmq-common</artifactId>
	<version>4.1.0-incubating</version>
</dependency>
```

# 配置文件

在配置文件 `application.properties` 添加一下内容

```sh
# 消费者的组名
apache.rocketmq.consumer.PushConsumer=PushConsumer

# 生产者的组名
apache.rocketmq.producer.producerGroup=Producer

# NameServer地址
apache.rocketmq.namesrvAddr=192.168.252.121:9876
```

# 消息生产者

```java
@Component
public class Producer {

    /**
     * 生产者的组名
     */
    @Value("${apache.rocketmq.producer.producerGroup}")
    private String producerGroup;

    /**
     * NameServer 地址
     */
    @Value("${apache.rocketmq.namesrvAddr}")
    private String namesrvAddr;

    @PostConstruct
    public void defaultMQProducer() {

        //生产者的组名
        DefaultMQProducer producer = new DefaultMQProducer(producerGroup);

        //指定NameServer地址，多个地址以 ; 隔开
        producer.setNamesrvAddr(namesrvAddr);

        try {

            /**
             * Producer对象在使用之前必须要调用start初始化，初始化一次即可
             * 注意：切记不可以在每次发送消息时，都调用start方法
             */
            producer.start();

            for (int i = 0; i < 100; i++) {

                String messageBody = "我是消息内容:" + i;

                String message = new String(messageBody.getBytes(), "utf-8");

                //构建消息
                Message msg = new Message("PushTopic" /* PushTopic */, "push"/* Tag  */, "key_" + i /* Keys */, message.getBytes());

                //发送消息
                SendResult result = producer.send(msg);

                System.out.println("发送响应：MsgId:" + result.getMsgId() + "，发送状态:" + result.getSendStatus());

            }

        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            producer.shutdown();
        }

    }
}
```

# 消息消费者

```java
@Component
public class Consumer {

    /**
     * 消费者的组名
     */
    @Value("${apache.rocketmq.consumer.PushConsumer}")
    private String consumerGroup;

    /**
     * NameServer地址
     */
    @Value("${apache.rocketmq.namesrvAddr}")
    private String namesrvAddr;

    @PostConstruct
    public void defaultMQPushConsumer() {

        //消费者的组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer(consumerGroup);

        //指定NameServer地址，多个地址以 ; 隔开
        consumer.setNamesrvAddr(namesrvAddr);
        try {
            //订阅PushTopic下Tag为push的消息
            consumer.subscribe("PushTopic", "push");

            //设置Consumer第一次启动是从队列头部开始消费还是队列尾部开始消费
            //如果非第一次启动，那么按照上次消费的位置继续消费
            consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);
            consumer.registerMessageListener(new MessageListenerConcurrently() {

                @Override
                public ConsumeConcurrentlyStatus consumeMessage(List<MessageExt> list, ConsumeConcurrentlyContext context) {
                    try {
                        for (MessageExt messageExt : list) {

                            System.out.println("messageExt: " + messageExt);//输出消息内容

                            String messageBody = new String(messageExt.getBody(), "utf-8");

                            System.out.println("消费响应：Msg: " + messageExt.getMsgId() + ",msgBody: " + messageBody);//输出消息内容

                        }
                    } catch (Exception e) {
                        e.printStackTrace();
                        return ConsumeConcurrentlyStatus.RECONSUME_LATER; //稍后再试
                    }
                    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS; //消费成功
                }


            });
            consumer.start();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

}
```


# 启动服务

```sh
@SpringBootApplication
public class SpringBootRocketmqApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringBootRocketmqApplication.class, args);
	}
}

```

控制台会有响应

```sh
发送响应：MsgId:0AFF015E556818B4AAC208A0504F0063，发送状态:SEND_OK

messageExt: MessageExt [queueId=0, storeSize=195, queueOffset=113824, sysFlag=0, bornTimestamp=1517559124047, bornHost=/192.168.252.1:62165, storeTimestamp=1517559135052, storeHost=/192.168.252.121:10911, msgId=C0A8FC7900002A9F00000000056F499C, commitLogOffset=91179420, bodyCRC=1687852546, reconsumeTimes=0, preparedTransactionOffset=0, toString()=Message [topic=PushTopic, flag=0, properties={MIN_OFFSET=0, MAX_OFFSET=113825, KEYS=key_99, CONSUME_START_TIME=1517559124049, UNIQ_KEY=0AFF015E556818B4AAC208A0504F0063, WAIT=true, TAGS=push}, body=21]]

消费响应：Msg: 0AFF015E556818B4AAC208A0504F0063,msgBody: 我是消息内容:99

...

```

# 监控服务

RocketMQ web界面监控RocketMQ-Console-Ng部署

[https://github.com/apache/rocketmq-externals/tree/master/rocketmq-console](https://github.com/apache/rocketmq-externals/tree/master/rocketmq-console)

# 下载并且编译

下载并且 maven 编译

```sh
git clone https://github.com/apache/rocketmq-externals.git
cd rocketmq-externals/rocketmq-console/
mvn clean package -Dmaven.test.skip=true
```

# 启动监控服务

`rocketmq.config.namesrvAddr` `NameServer` 地址,默认启动端口8080

```sh
nohup java -jar target/rocketmq-console-ng-1.0.0.jar --rocketmq.config.namesrvAddr=127.0.0.1:9876  > /dev/null 2>&1 &
```

# 关于报错

关闭防火墙


```sh
org.apache.rocketmq.client.exception.MQClientException: No route info of this topic, PushTopic
See http://rocketmq.apache.org/docs/faq/ for further details.
```

开启服务器自动创建Topic功能

`autoCreateTopicEnable=true`


```sh
Caused by: org.apache.rocketmq.remoting.exception.RemotingConnectException: connect to <192.168.0.4:10911> failed
```

这个错，主要是启动的时候指定的ip 是 `-n localhost:9876`


**在服务器使用，不能使用连接rocketmq 解决步骤**

```sh
步骤一，启动 Name Server
nohup sh bin/mqnamesrv > /dev/null 2>&1 &

步骤二，指定 Broker 外网IP
添加
vi /opt/apache-rocketmq/conf/broker.conf
brokerIP1=116.196.97.159

输入终端执行
export NAMESRV_ADDR=116.196.97.159:9876

步骤三，启动 Broker
nohup sh bin/mqbroker -n 116.196.97.159:9876 > autoCreateTopicEnable=true -c /opt/apache-rocketmq/conf/broker.conf /dev/null 2>&1 &

步骤四，启动监控页面
nohup java -jar target/rocketmq-console-ng-1.0.0.jar --rocketmq.config.namesrvAddr=116.196.97.159:9876  > /dev/null 2>&1 &
```


# 访问监控服务

![ ][1]
![ ][2]
![ ][3]
![ ][4]
![ ][5]

[1]: http://www.ymq.io/images/2018/RocketMQ/1.png
[2]: http://www.ymq.io/images/2018/RocketMQ/2.png
[3]: http://www.ymq.io/images/2018/RocketMQ/3.png
[4]: http://www.ymq.io/images/2018/RocketMQ/4.png
[5]: http://www.ymq.io/images/2018/RocketMQ/5.png

GitHub 源码：[https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rocketmq](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rocketmq)

Gitee 源码：[https://gitee.com/souyunku/spring-boot-examples/tree/master/spring-boot-rabbitmq](https://gitee.com/souyunku/spring-boot-examples/tree/master/spring-boot-rabbitmq)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/02/02/spring-boot-rocketmq-example](http://www.ymq.io/2018/02/02/spring-boot-rocketmq-example)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

