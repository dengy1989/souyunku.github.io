---
layout: post
title: 使用 Jedis 连接操作 Redis
categories: Redis
description: 使用 Jedis 连接操作 Redis
keywords: Redis 
---

# 使用 Jedis 连接操作 Redis

## Redis 简介

Redis 是完全开源免费的，遵守BSD协议，是一个高性能的key-value数据库。

Redis 与其他 key - value 缓存产品有以下三个特点：
Redis支持数据的持久化，可以将内存中的数据保持在磁盘中，重启的时候可以再次加载进行使用。
Redis不仅仅支持简单的key-value类型的数据，同时还提供list，set，zset，hash等数据结构的存储。
Redis支持数据的备份，即master-slave模式的数据备份。


## Redis 优势

性能极高 – Redis能读的速度是110000次/s,写的速度是81000次/s 。
丰富的数据类型 – Redis支持二进制案例的 Strings, Lists, Hashes, Sets 及 Ordered Sets 数据类型操作。
原子 – Redis的所有操作都是原子性的，同时Redis还支持对几个操作全并后的原子性执行。
丰富的特性 – Redis还支持 publish/subscribe, 通知, key 过期等等特性。

## Redis与其他key-value存储有什么不同？

Redis有着更为复杂的数据结构并且提供对他们的原子性操作，这是一个不同于其他数据库的进化路径。Redis的数据类型都是基于基本数据结构的同时对程序员透明，无需进行额外的抽象。
Redis运行在内存中但是可以持久化到磁盘，所以在对不同数据集进行高速读写时需要权衡内存，应为数据量不能大于硬件内存。在内存数据库方面的另一个优点是， 相比在磁盘上相同的复杂的数据结构，在内存中操作起来非常简单，这样Redis可以做很多内部复杂性很强的事情。 同时，在磁盘格式方面他们是紧凑的以追加的方式产生的，因为他们并不需要进行随机访问。

# 准备

## 环境安装 

**任选其一**

[CentOs7.3 搭建 Redis-4.0.1 单机服务](https://segmentfault.com/a/1190000010709337)

[CentOs7.3 搭建 Redis-4.0.1 Cluster 集群服务](https://segmentfault.com/a/1190000010682551)

# 测试用例

# Github 代码

代码我已放到 Github ，导入 ymq-redis 项目 

github [https://github.com/souyunku/ymq-example/tree/master/ymq-redis](https://github.com/souyunku/ymq-example/tree/master/ymq-redis)

## 添加依赖

在项目中添加 `jedis` 依赖

```xml
<dependency>
	<groupId>redis.clients</groupId>
	<artifactId>jedis</artifactId>
	<version>2.9.0</version>
</dependency>
```

## 获取集群连接

```java
private static JedisCluster jedisCluster = null;

static {
	JedisPoolConfig config = new JedisPoolConfig();

	//最大连接数, 默认8个
	config.setMaxTotal(1000);

	//大空闲连接数, 默认8个
	config.setMaxIdle(10);

	//获取连接时的最大等待毫秒数(如果设置为阻塞时BlockWhenExhausted),如果超时就抛异常, 小于零:阻塞不确定的时间,  默认-1
	config.setMaxWaitMillis(3000);

	//--------以下配置默认就可以-----------

	//最小空闲连接数, 默认0
	config.setMinIdle(0);

	//是否启用pool的jmx管理功能, 默认true
	config.setJmxEnabled(true);

	//是否启用后进先出, 默认true
	config.setLifo(true);

	//在获取连接的时候检查有效性, 默认false
	config.setTestOnBorrow(false);

	//在空闲时检查有效性, 默认false
	config.setTestWhileIdle(false);

	Set<HostAndPort> hps = new HashSet<HostAndPort>();

	String redisClusterIp = "10.4.89.161:6379";
	String[] ip = redisClusterIp.split(":");
	int port = Integer.valueOf(ip[1]);
	hps.add(new HostAndPort(ip[0], port));

	jedisCluster = new JedisCluster(hps, config);

	LOG.info("JedisPoolConfig:{}", JSONObject.toJSONString(config));

	Map<String, JedisPool> nodes = jedisCluster.getClusterNodes();

	LOG.info("Get the redis thread pool:{}", nodes.toString());
}
```


## 获取单实例连接


```java
private static Jedis jedis = null;

static {

	JedisPoolConfig config = new JedisPoolConfig();

	//最大连接数, 默认8个
	config.setMaxTotal(1000);

	//大空闲连接数, 默认8个
	config.setMaxIdle(10);

	//获取连接时的最大等待毫秒数(如果设置为阻塞时BlockWhenExhausted),如果超时就抛异常, 小于零:阻塞不确定的时间,  默认-1
	config.setMaxWaitMillis(3000);

	//--------以下配置默认就可以-----------

	//最小空闲连接数, 默认0
	config.setMinIdle(0);

	//是否启用pool的jmx管理功能, 默认true
	config.setJmxEnabled(true);

	//是否启用后进先出, 默认true
	config.setLifo(true);

	//在获取连接的时候检查有效性, 默认false
	config.setTestOnBorrow(false);

	//在空闲时检查有效性, 默认false
	config.setTestWhileIdle(false);

	JedisPool pool = new JedisPool(config, "127.0.0.1", 6379);

	LOG.info("JedisPoolConfig:{}", JSONObject.toJSONString(config));

	jedis = pool.getResource();
}
```

## Redis 工具类

**请查看源码**

代码我已放到 Github ，导入 ymq-redis 项目 

github [https://github.com/souyunku/ymq-example/tree/master/ymq-redis](https://github.com/souyunku/ymq-example/tree/master/ymq-redis)


```java
package io.ymq.redis.jedis.utils.CacheUtils;
```

## 单元测试

```java
/**
 * java JedisCluster 操作 redis 集群
 */
@Test
public void clusterTest() {

	JedisClusterUtils.saveString("cluster-key", "www.ymq.io");

	System.out.println(JedisClusterUtils.getString("cluster-key"));

}

/**
 * java Jedis 操作 redis 单实例
 */
@Test
public void sentineTest() {

	JedisSentinelUtils.saveString("sentine-key", "www.ymq.io");

	System.out.println(JedisSentinelUtils.getString("sentine-key"));

}

/**
 * cacheUtils 操作 redis 集群
 */
@Test
public void cacheUtilsTest() {

	CacheUtils.saveString("cluster-key", "www.ymq.io");

	System.out.println(CacheUtils.getString("cluster-key"));

}
```




