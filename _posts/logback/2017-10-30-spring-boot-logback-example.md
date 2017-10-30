---
layout: post
title: Spring Boot 中使用 LogBack 详解
categories: [LogBack,SpringBoot]
description: Spring Boot 中使用 LogBack 详解
keywords: LogBack 
---

LogBack是一个日志框架，它与Log4j可以说是同出一源，都出自Ceki Gülcü之手。（log4j的原型是早前由Ceki Gülcü贡献给Apache基金会的）下载地址 [https://logback.qos.ch/download.html](https://logback.qos.ch/download.html)

# LogBack、Slf4j和Log4j之间的关系

Slf4j是`The Simple Logging Facade for Java`的简称，是一个简单日志门面抽象框架，它本身只提供了日志Facade API和一个简单的日志类实现，一般常配合Log4j，LogBack，java.util.logging使用。Slf4j作为应用层的Log接入时，程序可以根据实际应用场景动态调整底层的日志实现框架(Log4j/LogBack/JdkLog…)。

LogBack和Log4j都是开源日记工具库，LogBack是Log4j的改良版本，比Log4j拥有更多的特性，同时也带来很大性能提升。详细数据可参照下面地址：Reasons to prefer logback over log4j。

LogBack官方建议配合Slf4j使用，这样可以灵活地替换底层日志框架。

**TIPS：为了优化log4j，以及更大性能的提升，Apache基金会已经着手开发了log4j 2.0, 其中也借鉴和吸收了logback的一些先进特性，目前log4j2还处于beta阶段**

# logback取代log4j的理由

1、更快的实现：Logback的内核重写了，在一些关键执行路径上性能提升10倍以上。而且logback不仅性能提升了，初始化内存加载也更小了。  
2、非常充分的测试：Logback经过了几年，数不清小时的测试。Logback的测试完全不同级别的。  
3、Logback-classic非常自然实现了SLF4j：Logback-classic实现了SLF4j。在使用SLF4j中，你都感觉不到logback-classic。而且因为logback-classic非常自然地实现了slf4j ， 所 以切换到log4j或者其他，非常容易，只需要提供成另一个jar包就OK，根本不需要去动那些通过SLF4JAPI实现的代码。  
4、非常充分的文档 官方网站有两百多页的文档。  
5、自动重新加载配置文件，当配置文件修改了，Logback-classic能自动重新加载配置文件。扫描过程快且安全，它并不需要另外创建一个扫描线程。这个技术充分保证了应用程序能跑得很欢在JEE环境里面。  
6、Lilith是log事件的观察者，和log4j的chainsaw类似。而lilith还能处理大数量的log数据 。  
7、谨慎的模式和非常友好的恢复，在谨慎模式下，多个FileAppender实例跑在多个JVM下，能 够安全地写道同一个日志文件。RollingFileAppender会有些限制。Logback的FileAppender和它的子类包括 RollingFileAppender能够非常友好地从I/O异常中恢复。  
8、配置文件可以处理不同的情况，开发人员经常需要判断不同的Logback配置文件在不同的环境下（开发，测试，生产）。而这些配置文件仅仅只有一些很小的不同，可以通过,和来实现，这样一个配置文件就可以适应多个环境。  
9、Filters（过滤器）有些时候，需要诊断一个问题，需要打出日志。在log4j，只有降低日志级别，不过这样会打出大量的日志，会影响应用性能。在Logback，你可以继续 保持那个日志级别而除掉某种特殊情况，如alice这个用户登录，她的日志将打在DEBUG级别而其他用户可以继续打在WARN级别。要实现这个功能只需加4行XML配置。可以参考MDCFIlter 。  
10、SiftingAppender（一个非常多功能的Appender）：它可以用来分割日志文件根据任何一个给定的运行参数。如，SiftingAppender能够区别日志事件跟进用户的Session，然后每个用户会有一个日志文件。  
11、自动压缩已经打出来的log：RollingFileAppender在产生新文件的时候，会自动压缩已经打出来的日志文件。压缩是个异步过程，所以甚至对于大的日志文件，在压缩过程中应用不会受任何影响。  
12、堆栈树带有包版本：Logback在打出堆栈树日志时，会带上包的数据。  
13、自动去除旧的日志文件：通过设置TimeBasedRollingPolicy或者SizeAndTimeBasedFNATP的maxHistory属性，你可以控制已经产生日志文件的最大数量。如果设置maxHistory 12，那那些log文件超过12个月的都会被自动移除。


# LogBack的结构

LogBack被分为3个组件，logback-core, logback-classic 和 logback-access。

**logback-core**提供了LogBack的核心功能，是另外两个组件的基础。

**logback-classic**则实现了Slf4j的API，所以当想配合Slf4j使用时，需要将logback-classic加入classpath。

**logback-access**是为了集成Servlet环境而准备的，可提供HTTP-access的日志接口。


# 配置详解

# Github 代码

代码我已放到 Github ，导入`spring-boot-logback` 项目 

github [spring-boot-logback](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-logback)

## Maven依赖

假如maven依赖中添加了`spring-boot-starter-logging`：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

那么，我们的Spring Boot应用将自动使用logback作为应用日志框架，Spring Boot启动的时候，由org.springframework.boot.logging.Logging-Application-Listener根据情况初始化并使用。

但是呢，实际开发中我们不需要直接添加该依赖，你会发现spring-boot-starter其中包含了 spring-boot-starter-logging，该依赖内容就是 Spring Boot 默认的日志框架 logback

![ 依赖关系 ][1]


# 未完待续


代码我已放到 Github ，导入`spring-boot-logback` 项目 

github [spring-boot-logback](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-logback)

[1]: /images/2017/logback/1.png
[2]: /images/2017/logback/2.png


