---
layout: post
title: Spring Boot 中使用 LogBack 配置
categories: [LogBack,SpringBoot]
description: Spring Boot 中使用 LogBack 配置
keywords: LogBack 
---

`LogBack`是一个日志框架，它与Log4j可以说是同出一源，都出自`Ceki Gülcü`之手。（`log4j`的原型是早前由`Ceki Gülcü`贡献给`Apache`基金会的）下载地址 [https://logback.qos.ch/download.html](https://logback.qos.ch/download.html)

# LogBack、Slf4j和Log4j之间的关系

`Slf4j`是`The Simple Logging Facade for Java`的简称，是一个简单日志门面抽象框架，它本身只提供了日志Facade `API`和一个简单的日志类实现，一般常配合`Log4j，LogBack，java.util.logging`使用。`Slf4j`作为应用层的Log接入时，程序可以根据实际应用场景动态调整底层的日志实现框架`(Log4j/LogBack/JdkLog…)`。

`LogBack`和`Log4j`都是开源日记工具库，`LogBack`是`Log4j`的改良版本，比`Log4j`拥有更多的特性，同时也带来很大性能提升。详细数据可参照下面地址：`Reasons to prefer logback over log4j`。

`LogBack`官方建议配合`Slf4j`使用，这样可以灵活地替换底层日志框架。

**TIPS：为了优化`log4j`，以及更大性能的提升，Apache基金会已经着手开发了`log4j 2.0`, 其中也借鉴和吸收了`logback`的一些先进特性，目前`log4j2`还处于`beta`阶段**

# logback取代log4j的理由

1、更快的实现：`Logback`的内核重写了，在一些关键执行路径上性能提升10倍以上。而且`logback`不仅性能提升了，初始化内存加载也更小了。  
2、非常充分的测试：`Logback`经过了几年，数不清小时的测试。`Logback`的测试完全不同级别的。  
3、`Logback-classic`非常自然实现了`SLF4j：Logback-classic`实现了`SLF4j`。在使用SLF4j中，你都感觉不到`logback-classic`。而且因为`logback-classic`非常自然地实现了`slf4j` ， 所 以切换到`log4j`或者其他，非常容易，只需要提供成另一个`jar`包就OK，根本不需要去动那些通过`SLF4JAPI`实现的代码。  
4、非常充分的文档 官方网站有两百多页的文档。  
5、自动重新加载配置文件，当配置文件修改了，`Logback-classic`能自动重新加载配置文件。扫描过程快且安全，它并不需要另外创建一个扫描线程。这个技术充分保证了应用程序能跑得很欢在JEE环境里面。  
6、`Lilith`是`log`事件的观察者，和`log4j`的`chainsaw`类似。而`lilith`还能处理大数量的log数据 。  
7、谨慎的模式和非常友好的恢复，在谨慎模式下，多个`FileAppender`实例跑在多个JVM下，能 够安全地写道同一个日志文件。`RollingFileAppender`会有些限制。`Logback`的`FileAppender`和它的子类包括 `RollingFileAppender`能够非常友好地从I/O异常中恢复。  
8、配置文件可以处理不同的情况，开发人员经常需要判断不同的Logback配置文件在不同的环境下（开发，测试，生产）。而这些配置文件仅仅只有一些很小的不同，可以通过,和来实现，这样一个配置文件就可以适应多个环境。  
9、`Filters`（过滤器）有些时候，需要诊断一个问题，需要打出日志。在`log4j`，只有降低日志级别，不过这样会打出大量的日志，会影响应用性能。在`Logback`，你可以继续 保持那个日志级别而除掉某种特殊情况，如`alice`这个用户登录，她的日志将打在`DEBUG`级别而其他用户可以继续打在`WARN`级别。要实现这个功能只需加4行`XML`配置。可以参考`MDCFIlter `。  
10、`SiftingAppender`（一个非常多功能的`Appender`）：它可以用来分割日志文件根据任何一个给定的运行参数。如，`SiftingAppender`能够区别日志事件跟进用户的`Session`，然后每个用户会有一个日志文件。  
11、自动压缩已经打出来的`log：RollingFileAppender`在产生新文件的时候，会自动压缩已经打出来的日志文件。压缩是个异步过程，所以甚至对于大的日志文件，在压缩过程中应用不会受任何影响。  
12、堆栈树带有包版本：`Logback`在打出堆栈树日志时，会带上包的数据。  
13、自动去除旧的日志文件：通过设置`TimeBasedRollingPolicy`或者`SizeAndTimeBasedFNATP的maxHistory`属性，你可以控制已经产生日志文件的最大数量。如果设置`maxHistory 12`，那那些`log`文件超过`12`个月的都会被自动移除。


# LogBack的结构

LogBack被分为3个组件，`logback-core, logback-classic` 和 `logback-access`。

**logback-core**提供了LogBack的核心功能，是另外两个组件的基础。

**logback-classic**则实现了`Slf4j`的`API`，所以当想配合Slf4j使用时，需要将`logback-classic`加入`classpath`。

**logback-access**是为了集成`Servlet`环境而准备的，可提供`HTTP-access`的日志接口。


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

## 配置文件

**配置文件 logback-spring.xml**
```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- 日志根目录-->
    <springProperty scope="context" name="LOG_HOME" source="logging.path" defaultValue="/data/logs/spring-boot-logback"/>

    <!-- 日志级别 -->
    <springProperty scope="context" name="LOG_ROOT_LEVEL" source="logging.level.root" defaultValue="DEBUG"/>

    <!--  标识这个"STDOUT" 将会添加到这个logger -->
    <springProperty scope="context" name="STDOUT" source="log.stdout" defaultValue="STDOUT"/>

    <!-- 日志文件名称-->
    <property name="LOG_PREFIX" value="spring-boot-logback" />

    <!-- 日志文件编码-->
    <property name="LOG_CHARSET" value="UTF-8" />

    <!-- 日志文件路径+日期-->
    <property name="LOG_DIR" value="${LOG_HOME}/%d{yyyyMMdd}" />

    <!--对日志进行格式化-->
    <property name="LOG_MSG" value="- | [%X{requestUUID}] | [%d{yyyyMMdd HH:mm:ss.SSS}] | [%level] | [${HOSTNAME}] | [%thread] | [%logger{36}] | --> %msg|%n "/>

    <!--文件大小，默认10MB-->
    <property name="MAX_FILE_SIZE" value="50MB" />

    <!-- 配置日志的滚动时间 ，表示只保留最近 10 天的日志-->
    <property name="MAX_HISTORY" value="10"/>

    <!--输出到控制台-->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <!-- 输出的日志内容格式化-->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>${LOG_MSG}</pattern>
        </layout>
    </appender>

    <!--输出到文件-->
    <appender name="0" class="ch.qos.logback.core.rolling.RollingFileAppender">
    </appender>

    <!-- 定义 ALL 日志的输出方式:-->
    <appender name="FILE_ALL" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!--日志文件路径，日志文件名称-->
        <File>${LOG_HOME}/all_${LOG_PREFIX}.log</File>

        <!-- 设置滚动策略，当天的日志大小超过 ${MAX_FILE_SIZE} 文件大小时候，新的内容写入新的文件， 默认10MB -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">

            <!--日志文件路径，新的 ALL 日志文件名称，“ i ” 是个变量 -->
            <FileNamePattern>${LOG_DIR}/all_${LOG_PREFIX}%i.log</FileNamePattern>

            <!-- 配置日志的滚动时间 ，表示只保留最近 10 天的日志-->
            <MaxHistory>${MAX_HISTORY}</MaxHistory>

            <!--当天的日志大小超过 ${MAX_FILE_SIZE} 文件大小时候，新的内容写入新的文件， 默认10MB-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>

        </rollingPolicy>

        <!-- 输出的日志内容格式化-->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>${LOG_MSG}</pattern>
        </layout>
    </appender>

    <!-- 定义 ERROR 日志的输出方式:-->
    <appender name="FILE_ERROR" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <!-- 下面为配置只输出error级别的日志 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <OnMismatch>DENY</OnMismatch>
            <OnMatch>ACCEPT</OnMatch>
        </filter>
        <!--日志文件路径，日志文件名称-->
        <File>${LOG_HOME}/err_${LOG_PREFIX}.log</File>

        <!-- 设置滚动策略，当天的日志大小超过 ${MAX_FILE_SIZE} 文件大小时候，新的内容写入新的文件， 默认10MB -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">

            <!--日志文件路径，新的 ERR 日志文件名称，“ i ” 是个变量 -->
            <FileNamePattern>${LOG_DIR}/err_${LOG_PREFIX}%i.log</FileNamePattern>

            <!-- 配置日志的滚动时间 ，表示只保留最近 10 天的日志-->
            <MaxHistory>${MAX_HISTORY}</MaxHistory>

            <!--当天的日志大小超过 ${MAX_FILE_SIZE} 文件大小时候，新的内容写入新的文件， 默认10MB-->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>

        <!-- 输出的日志内容格式化-->
        <layout class="ch.qos.logback.classic.PatternLayout">
            <Pattern>${LOG_MSG}</Pattern>
        </layout>
    </appender>

    <!-- additivity 设为false,则logger内容不附加至root ，配置以配置包下的所有类的日志的打印，级别是 ERROR-->

    <logger name="org.springframework"     level="ERROR" />
    <logger name="org.apache.commons"      level="ERROR" />
    <logger name="org.apache.zookeeper"    level="ERROR"  />
    <logger name="com.alibaba.dubbo.monitor" level="ERROR"/>
    <logger name="com.alibaba.dubbo.remoting" level="ERROR" />

    <!-- ${LOG_ROOT_LEVEL} 日志级别 -->
    <root level="${LOG_ROOT_LEVEL}">

        <!-- 标识这个"${STDOUT}"将会添加到这个logger -->
        <appender-ref ref="${STDOUT}"/>

        <!-- FILE_ALL 日志输出添加到 logger -->
        <appender-ref ref="FILE_ALL"/>

        <!-- FILE_ERROR 日志输出添加到 logger -->
        <appender-ref ref="FILE_ERROR"/>
    </root>

</configuration>
```

**配置文件  application.properties**

```
#日志级别从低到高分为TRACE < DEBUG < INFO < WARN < ERROR < FATAL，如果设置为WARN，则低于WARN的信息都不会输出
logging.level.root=trace

logging.path=/data/logs/spring-boot-logback
```

## 节点介绍


**这里参考，嘟嘟独立博客，和 Albin 的文章**

[Spring Boot干货系列：（七）默认日志logback配置解析](http://tengj.top/2017/04/05/springboot7/)

[logback节点配置详解](http://www.cnblogs.com/DeepLearing/p/5663178.html)


日志会每天新建一个文件夹，日文文件配置的每50兆，一个文本文件，超过新写入一个

```
文件夹：20171031
文件夹内容：all_spring-boot-logback0.log 
文件夹内容：all_spring-boot-logback1.log
文件夹内容：all_spring-boot-logback2.log

文件夹内容：err_spring-boot-logback0.log
```



## MDC requestUUID

**一种多线程下日志管理实践方式**

[logback MDC(Mapped Diagnostic Context)与分布式系统的跟踪系统](http://blog.csdn.net/doctor_who2004/article/details/46974695)

[Slf4j MDC 使用和 基于 Logback 的实现分析](http://blog.csdn.net/liubo2012/article/details/46337063)

[MDC介绍 -- 一种多线程下日志管理实践方式](http://blog.csdn.net/sunzhenhua0608/article/details/29175283)

　　MDC（Mapped Diagnostic Context，映射调试上下文）是 log4j 和 logback 提供的一种方便在多线程条件下记录日志的功能。某些应用程序采用多线程的方式来处理多个用户的请求。在一个用户的使用过程中，可能有多个不同的线程来进行处理。典型的例子是 Web 应用服务器。当用户访问某个页面时，应用服务器可能会创建一个新的线程来处理该请求，也可能从线程池中复用已有的线程。在一个用户的会话存续期间，可能有多个线程处理过该用户的请求。这使得比较难以区分不同用户所对应的日志。当需要追踪某个用户在系统中的相关日志记录时，就会变得很麻烦。

一种解决的办法是采用自定义的日志格式，把用户的信息采用某种方式编码在日志记录中。这种方式的问题在于要求在每个使用日志记录器的类中，都可以访问到用户相关的信息。这样才可能在记录日志时使用。这样的条件通常是比较难以满足的。MDC 的作用是解决这个问题。

　　MDC 可以看成是一个与当前线程绑定的哈希表，可以往其中添加键值对。MDC 中包含的内容可以被同一线程中执行的代码所访问。当前线程的子线程会继承其父线程中的 MDC 的内容。当需要记录日志时，只需要从 MDC 中获取所需的信息即可。MDC 的内容则由程序在适当的时候保存进去。对于一个 Web 应用来说，通常是在请求被处理的最开始保存这些数据。

**自定义拦截器 logback requestUUID**

```java
/**
 * 描述: 自定义拦截器 logback requestUUID
 *
 * @author yanpenglei
 * @create 2017-10-30 16:15
 **/

public class ControllerInterceptor extends HandlerInterceptorAdapter {

    private Logger LOGGER = LoggerFactory.getLogger(ControllerInterceptor.class);

    //在请求处理之前回调方法
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {

        String requestUUID = MDC.get("requestUUID");
        if (requestUUID == null || "".equals(requestUUID)) {
            String uuid = UUID.randomUUID().toString();
            uuid = uuid.replaceAll("-", "").toUpperCase();
            MDC.put("requestUUID", uuid);
            LOGGER.info("ControllerInterceptor preHandle 在请求处理之前生成 logback requestUUID:{}", uuid);
        }

        return true;// 只有返回true才会继续向下执行，返回false取消当前请求
    }

    //请求处理之后回调方法
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        /* 线程结束后需要清除,否则当前线程会一直占用这个requestId值 */
        MDC.remove("requestUUID");
        LOGGER.info("ControllerInterceptor postHandle 请求处理之后清除 logback MDC requestUUID");
    }

    //整个请求处理完毕回调方法
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        /*整个请求线程结束后需要清除,否则当前线程会一直占用这个requestId值 */
        MDC.clear();
        LOGGER.info("ControllerInterceptor afterCompletion 整个请求处理完毕清除 logback MDC requestUUID");
    }
}
```

**对日志进行格式化,时候用到**
```
<property name="LOG_MSG" value="- | [%X{requestUUID}] | [%d{yyyyMMdd HH:mm:ss.SSS}] | [%level] | [${HOSTNAME}] | [%thread] | [%logger{36}] | --> %msg|%n "/>
```

```java
/**
 * 描述:拦截器配置
 *
 * @author yanpenglei
 * @create 2017-10-30 16:54
 **/
@Configuration
public class MyWebMvcConfigurer extends WebMvcConfigurerAdapter {

    @Override
    public void addInterceptors(InterceptorRegistry registry) {

        /**
         * 多个拦截器组成一个拦截器链
         * addPathPatterns 用于添加拦截规则
         * excludePathPatterns 用于排除拦截
         */
        registry.addInterceptor(new ControllerInterceptor()).addPathPatterns("/**");
        super.addInterceptors(registry);
    }
}
```



## 日志切面

```java
@Aspect
@Component
public class LogAspect {

    private static final Logger log = LoggerFactory.getLogger(LogAspect.class);
    private static final String dateFormat = "yyyy-MM-dd HH:mm:ss";
    private static final String STRING_START = "\n>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>\n";
    private static final String STRING_END   = "\n<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<\n";

    @Pointcut("execution(* io.ymq.logback.controller..*(..))")
    public void serviceLog() {
    }

	@Around("serviceLog()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        Class<?> targetClass = method.getDeclaringClass();

        StringBuffer classAndMethod = new StringBuffer();

        Log classAnnotation = targetClass.getAnnotation(Log.class);
        Log methodAnnotation = method.getAnnotation(Log.class);

        if (classAnnotation != null) {
            if (classAnnotation.ignore()) {
                return joinPoint.proceed();
            }
            classAndMethod.append(classAnnotation.value()).append("-");
        }

        if (methodAnnotation != null) {
            if (methodAnnotation.ignore()) {
                return joinPoint.proceed();
            }
            classAndMethod.append(methodAnnotation.value());
        }

        String target = targetClass.getName() + "#" + method.getName();

        String params = null;
             params = JSONObject.toJSONStringWithDateFormat(joinPoint.getArgs(), dateFormat, SerializerFeature.WriteMapNullValue);

        log.info(STRING_START + "{} 开始调用--> {} 参数:{}", classAndMethod.toString(), target, params);

        long start = System.currentTimeMillis();
        Object result = joinPoint.proceed();
        long timeConsuming = System.currentTimeMillis() - start;

        log.info("\n{} 调用结束<-- {} 返回值:{} 耗时:{}ms" + STRING_END, classAndMethod.toString(), target, JSONObject.toJSONStringWithDateFormat(result, dateFormat, SerializerFeature.WriteMapNullValue), timeConsuming);
        return result;
    }

}
```

## 测试 logback

浏览器访问：[http://127.0.0.1:8080/index/?content="我是测试内容"]( http://127.0.0.1:8080/index/?content="我是测试内容")

```java
@RestController
@RequestMapping(value = "/index")
public class IndexController {

    private final Logger LOGGER = LoggerFactory.getLogger(this.getClass());

    /**
     * http://127.0.0.1:8080/index/?content="我是测试内容"
     *
     * @param content
     * @return
     */
    @Log("首页IndexController")
    @RequestMapping(value="", method= RequestMethod.GET)
    public String index(@RequestParam String content) {
        LocalDateTime localDateTime = LocalDateTime.now();

        LOGGER.trace("请求参数：content:{}", content);
        LOGGER.debug("请求参数：content:{}", content);
        LOGGER.info("请求参数：content:{}", content);
        LOGGER.warn("请求参数：content:{}", content);
        LOGGER.error("请求参数：content:{}", content);

        return localDateTime + ",content:" + content;
    }
}
```

前面的`07E94BA525CF4C97851E4B9E4ABB4890` 就是通过`logback` 的 MDC 做到的

```
首页IndexController 开始调用--> io.ymq.logback.controller.IndexController#index 参数:["\"我是测试内容\""]|
- | [07E94BA525CF4C97851E4B9E4ABB4890] | [20171101 10:02:35.589] | [DEBUG] | [DESKTOP-VG43S0C] | [http-nio-8080-exec-1] | [i.y.l.controller.IndexController] | --> 请求参数：content:"我是测试内容"|
- | [07E94BA525CF4C97851E4B9E4ABB4890] | [20171101 10:02:35.589] | [INFO] | [DESKTOP-VG43S0C] | [http-nio-8080-exec-1] | [i.y.l.controller.IndexController] | --> 请求参数：content:"我是测试内容"|
- | [07E94BA525CF4C97851E4B9E4ABB4890] | [20171101 10:02:35.589] | [WARN] | [DESKTOP-VG43S0C] | [http-nio-8080-exec-1] | [i.y.l.controller.IndexController] | --> 请求参数：content:"我是测试内容"|
- | [07E94BA525CF4C97851E4B9E4ABB4890] | [20171101 10:02:35.590] | [ERROR] | [DESKTOP-VG43S0C] | [http-nio-8080-exec-1] | [i.y.l.controller.IndexController] | --> 请求参数：content:"我是测试内容"|
- | [07E94BA525CF4C97851E4B9E4ABB4890] | [20171101 10:02:35.606] | [INFO] | [DESKTOP-VG43S0C] | [http-nio-8080-exec-1] | [i.y.logback.config.commons.LogAspect] | --> 
首页IndexController 调用结束<-- io.ymq.logback.controller.IndexController#index 返回值:"2017-11-01T10:02:35.589,content:\"我是测试内容\"" 耗时:23ms
```

**从上图可以看到，日志输出内容元素具体如下：**

`requestUUID：一次请求是唯一的`    
`时间日期：精确到毫秒`   
`日志级别：ERROR, WARN, INFO, DEBUG or TRACE`  
`主机名：`    
`进程ID：`    
`类路径：`    
`分隔符： --> 标识实际日志的开始 `   
`日志内容：`  

**日志切面的响应：**

```
首页IndexController 开始调用--> io.ymq.logback.controller.IndexController#index 参数:["\"我是测试内容\""]|
首页IndexController 调用结束<-- io.ymq.logback.controller.IndexController#index 返回值:"2017-11-01T10:02:35.589,content:\"我是测试内容\"" 耗时:23ms
```

代码我已放到 Github ，导入`spring-boot-logback` 项目 

github [spring-boot-logback](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-logback)

[1]: http://www.ymq.io/images/2017/logback/1.png
[2]: http://www.ymq.io/images/2017/logback/2.png


[slf4j-logback 日志以json格式导入ELK](http://www.cnblogs.com/spec-dog/p/6281377.html)


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")
