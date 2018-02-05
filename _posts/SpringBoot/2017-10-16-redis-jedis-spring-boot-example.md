---
layout: post
title: Spring Boot 中使用 Redis
categories: [Redis,SpringBoot]
description: Spring Boot 中使用 Redis
keywords: Redis 
---

Spring Boot中除了对常用的关系型数据库提供了优秀的自动化支持之外，对于很多NoSQL数据库一样提供了自动化配置的支持，包括：Redis, MongoDB, Elasticsearch, Solr和Cassandra。

# 准备

## 环境安装 

**任选其一**

[CentOs7.3 搭建 Redis-4.0.1 单机服务](https://segmentfault.com/a/1190000010709337)

[CentOs7.3 搭建 Redis-4.0.1 Cluster 集群服务](https://segmentfault.com/a/1190000010682551)

# 测试用例

# Github 代码

代码我已放到 Github ，导入 spring-boot-examples 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-redis](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-redis)

## 添加依赖

在项目中添加 `spring-boot-starter-data-redis` 依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

## 配置 RedisTemplate 实例

```java
@Configuration
public class RedisConfig {

    private Logger LOG = LoggerFactory.getLogger(RedisConfig.class);

    @Bean
    JedisConnectionFactory jedisConnectionFactory() {
        return new JedisConnectionFactory();
    }

    @Bean
    public RedisTemplate<String, String> redisTemplate() {
        RedisTemplate<String, String> template = new RedisTemplate<String, String>();
        template.setConnectionFactory(jedisConnectionFactory());
        template.setKeySerializer(new StringRedisSerializer());
        LOG.info("create RedisTemplate success");
        return template;
    }
}
```

## 配置参数

**`application.properties`**
```
# Redis数据库索引（默认为0）
spring.redis.database=0
# Redis服务器地址
spring.redis.host=127.0.0.1
# Redis服务器连接端口
spring.redis.port=6379
# Redis服务器连接密码（默认为空）
spring.redis.password=
# 连接池最大连接数（使用负值表示没有限制）
spring.redis.pool.max-active=8
# 连接池最大阻塞等待时间（使用负值表示没有限制）
spring.redis.pool.max-wait=-1
# 连接池中的最大空闲连接
spring.redis.pool.max-idle=8
# 连接池中的最小空闲连接
spring.redis.pool.min-idle=0
# 连接超时时间（毫秒）
spring.redis.timeout=0
```
## 操作 Redis 工具类

```java
public class CacheUtils {

    @Resource
    private RedisTemplate<String, String> redisTemplate;
    private static CacheUtils cacheUtils;

    @PostConstruct
    public void init() {
        cacheUtils = this;
        cacheUtils.redisTemplate = this.redisTemplate;
    }

    /**
     * 保存到hash集合中
     *
     * @param hName 集合名
     * @param key
     * @param value
     */
    public static void hashSet(String hName, String key, String value) {
        cacheUtils.redisTemplate.opsForHash().put(hName, key, value);
    }

    /**
     * 从hash集合里取得
     *
     * @param hName
     * @param key
     * @return
     */

    public static Object hashGet(String hName, String key) {
        return cacheUtils.redisTemplate.opsForHash().get(hName, key);
    }

    /**
     省略 N 多方法
     。。。。。。
     */
}
```


## 注册配置类到容器

```java
@Configuration
@Import({RedisConfig.class, CacheUtils.class})
public class RedisAutoConfiguration {

}
```

## 单元测试

```java
import io.ymq.redis.utils.CacheUtils;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import io.ymq.redis.run.Application;
import org.springframework.test.context.junit4.SpringRunner;

/**
 * 描述:测试类
 *
 * @author yanpenglei
 * @create 2017-10-16 13:18
 **/
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Application.class)
public class BaseTest {

    @Test
    public void test() throws Exception {

        CacheUtils.hashSet("test", "ymq", "www.ymq.io");

        System.out.println(CacheUtils.hashGet("test", "ymq"));
    }
    
}
```


代码我已放到 Github ，导入 spring-boot-examples 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-redis](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-redis)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

  
  
  
