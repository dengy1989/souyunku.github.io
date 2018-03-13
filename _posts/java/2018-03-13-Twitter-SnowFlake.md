---
layout: post
title: Twitter的分布式雪花算法 SnowFlake 每秒自增生成26个万个可排序的ID (Java版) 
categories: java
description: Twitter的分布式雪花算法 SnowFlake 每秒自增生成26个万个可排序的ID (Java版) 
keywords: java 
---

分布式系统中，有一些需要使用全局唯一ID的场景，这种时候为了防止ID冲突可以使用36位的UUID，但是UUID有一些缺点，首先他相对比较长，另外UUID一般是无序的。

有些时候我们希望能使用一种简单一些的ID，并且希望ID能够按照时间有序生成。

而twitter的SnowFlake解决了这种需求，最初Twitter把存储系统从MySQL迁移到Cassandra，因为Cassandra没有顺序ID生成机制，所以开发了这样一套全局唯一ID生成服务。

# 原理

Twitter的雪花算法SnowFlake，使用Java语言实现。

SnowFlake算法产生的ID是一个64位的整型，结构如下（每一部分用“-”符号分隔）：

```
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000
```

**1位标识部分**，在java中由于long的最高位是符号位，正数是0，负数是1，一般生成的ID为正数，所以为0；

**41位时间戳部分**，这个是毫秒级的时间，一般实现上不会存储当前的时间戳，而是时间戳的差值（当前时间-固定的开始时间），这样可以使产生的ID从更小值开始；41位的时间戳可以使用69年，(1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69年；

**10位节点部分**，Twitter实现中使用前5位作为数据中心标识，后5位作为机器标识，可以部署1024个节点；

**12位序列号部分**，支持同一毫秒内同一个节点可以生成4096个ID；

SnowFlake算法生成的ID大致上是按照时间递增的，用在分布式系统中时，需要注意数据中心标识和机器标识必须唯一，这样就能保证每个节点生成的ID都是唯一的。或许我们不一定都需要像上面那样使用5位作为数据中心标识，5位作为机器标识，可以根据我们业务的需要，灵活分配节点部分，如：若不需要数据中心，完全可以使用全部10位作为机器标识；若数据中心不多，也可以只使用3位作为数据中心，7位作为机器标识。

snowflake生成的ID整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞（由datacenter和workerId作区分），并且效率较高。据说：snowflake每秒能够产生26万个ID。


# 源码

**本机实测：100万个ID 耗时5秒**

```java
/**
 * 描述: Twitter的分布式自增ID雪花算法snowflake (Java版)
 * https://github.com/souyunku/SnowFlake
 *
 * @author yanpenglei
 * @create 2018-03-13 12:37
 **/
public class SnowFlake {

    /**
     * 起始的时间戳
     */
    private final static long START_STMP = 1480166465631L;

    /**
     * 每一部分占用的位数
     */
    private final static long SEQUENCE_BIT = 12; //序列号占用的位数
    private final static long MACHINE_BIT = 5;   //机器标识占用的位数
    private final static long DATACENTER_BIT = 5;//数据中心占用的位数

    /**
     * 每一部分的最大值
     */
    private final static long MAX_DATACENTER_NUM = -1L ^ (-1L << DATACENTER_BIT);
    private final static long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
    private final static long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);

    /**
     * 每一部分向左的位移
     */
    private final static long MACHINE_LEFT = SEQUENCE_BIT;
    private final static long DATACENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
    private final static long TIMESTMP_LEFT = DATACENTER_LEFT + DATACENTER_BIT;

    private long datacenterId;  //数据中心
    private long machineId;     //机器标识
    private long sequence = 0L; //序列号
    private long lastStmp = -1L;//上一次时间戳

    public SnowFlake(long datacenterId, long machineId) {
        if (datacenterId > MAX_DATACENTER_NUM || datacenterId < 0) {
            throw new IllegalArgumentException("datacenterId can't be greater than MAX_DATACENTER_NUM or less than 0");
        }
        if (machineId > MAX_MACHINE_NUM || machineId < 0) {
            throw new IllegalArgumentException("machineId can't be greater than MAX_MACHINE_NUM or less than 0");
        }
        this.datacenterId = datacenterId;
        this.machineId = machineId;
    }

    /**
     * 产生下一个ID
     *
     * @return
     */
    public synchronized long nextId() {
        long currStmp = getNewstmp();
        if (currStmp < lastStmp) {
            throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
        }

        if (currStmp == lastStmp) {
            //相同毫秒内，序列号自增
            sequence = (sequence + 1) & MAX_SEQUENCE;
            //同一毫秒的序列数已经达到最大
            if (sequence == 0L) {
                currStmp = getNextMill();
            }
        } else {
            //不同毫秒内，序列号置为0
            sequence = 0L;
        }

        lastStmp = currStmp;

        return (currStmp - START_STMP) << TIMESTMP_LEFT //时间戳部分
                | datacenterId << DATACENTER_LEFT       //数据中心部分
                | machineId << MACHINE_LEFT             //机器标识部分
                | sequence;                             //序列号部分
    }

    private long getNextMill() {
        long mill = getNewstmp();
        while (mill <= lastStmp) {
            mill = getNewstmp();
        }
        return mill;
    }

    private long getNewstmp() {
        return System.currentTimeMillis();
    }

    public static void main(String[] args) {
        SnowFlake snowFlake = new SnowFlake(2, 3);

        long start = System.currentTimeMillis();
        for (int i = 0; i < 1000000; i++) {
            System.out.println(snowFlake.nextId());
        }

        System.out.println(System.currentTimeMillis() - start);


    }
}
```

循环生成的ID，运行结果如下：

```
170916032679263329
170916032679263330
170916032679263331
170916032679263332
170916032679263333
170916032679263334
170916032679263335
170916032679263336
170916032679263337
170916032679263338
170916032679263339
170916032679263340
170916032679263341
170916032679263342
```

# 开源地址

**Github**:[https://github.com/souyunku/SnowFlake](https://github.com/souyunku/SnowFlake)

# 推荐阅读

## Spring Cloud 系列教程

- [Spring Cloud（一）服务的注册与发现 Eureka ](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964403&idx=1&sn=73ad9f65e2530bf87dadd35a96658fd7)
- [Spring Cloud（二）Consul 服务治理实现 ](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964404&idx=1&sn=72676a761715bdd1e4711360dfa2c469)
- [Spring Cloud（三）服务提供者 Eureka + 服务消费者（rest + Ribbon）](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964405&idx=1&sn=9a85514edf8e8fbe301aea5c0a8fb55e)
- [Spring Cloud（四）服务提供者 Eureka + 服务消费者 Feign ](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964406&idx=1&sn=6884383251b4eb9ae8a9c9f3ea4f3c09)
- [Spring Cloud（五）断路器监控(Hystrix Dashboard)](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964412&idx=1&sn=5e5e5208aedf7324f1f0ecd7e3780344)
- [Spring Cloud（六）服务网关 zuul 快速入门](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964422&idx=1&sn=4feb1646baa0a2172cdf953f55824002)
- [Spring Cloud（七）服务网关 Zuul Filter 使用](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964425&idx=1&sn=f33137f5c312b2d938d17bb1125d411d)
- [Spring Cloud（八）高可用的分布式配置中心 Spring Cloud Config](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964427&idx=1&sn=9aed5b79885c8e70831b5219ba1571be)
- [Spring Cloud（九）高可用的分布式配置中心 Spring Cloud Config 集成 Eureka 服务](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964439&idx=1&sn=556c328bd756555ff2c3422277fb7ffc)
- [Spring Cloud（十）高可用的分布式配置中心 Spring Cloud Config 中使用 Refresh](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964444&idx=1&sn=b8826d3d314d5f64bd5c54bcc8c15dbe)
- [Spring Cloud（十一）高可用的分布式配置中心 Spring Cloud Bus 消息总线集成（RabbitMQ）](http://mp.weixin.qq.com/s/R9bghtpnLPtGBHjBJRR0Og)

## Spring Boot 系列教程

源码 + 教程

**Github**:[https://github.com/souyunku/spring-boot-examples](https://github.com/souyunku/spring-boot-examples)

![Spring Cloud 系列教程](http://www.ymq.io/images/2018/spring/SpringBoot.png "Spring Cloud 系列教程")

## Docker 容器

- [Docker Compose 1.18.0 之服务编排详解](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964477&idx=1&sn=3cfa2332fc3f5fa0cb7613987b48f449)
- [Docker CE 安装 初窥 Dockerfile 部署 Nginx](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964410&idx=1&sn=a1d189b731662f062ab5c8ae6c1d4662)
- [Docker Container 容器操作](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964415&idx=1&sn=a1485845724a3595cbeb22abfd19c6ad)
- [Docker Hub 仓库使用，及搭建 Docker Registry](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964418&idx=1&sn=9f7b142968b75eaa7d0934d1045e0ce9)
- [Docker Registry Server 搭建,配置免费 HTTPS 证书，及拥有权限认证、TLS 的私有仓库](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964421&idx=1&sn=8ac4f3fb5dc1828b67b5798ffc0a9753)
- [Docker Registry 企业级私有镜像仓库Harbor管理WEB UI, 可能是最详细的部署](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964426&idx=1&sn=0c6d7d1ae718c5fea9901e5321aba403)
- [Docker 部署 SpringBoot 项目整合 Redis 镜像做访问计数 Demo](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964438&idx=1&sn=f709b17011fa2713221f66aa28e5702e)
- [Docker Maven Plugin 生成 Docker 镜像 push 到 DockerHub上 ](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964451&idx=1&sn=4865f254dcfa69fe0e98e7dde4a6cd5b)

## 环境搭建
- [搭建 Apache RocketMQ 单机环境](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964513&idx=1&sn=3c926d8f96048ba17dee17541c69893f&chksm=88ede9c9bf9a60dfb530bc3f7d892c826e99fba5a6c533744dcd25ebc6ab8839cbf79e607c5d#rd)  
- [手把手教你 MongoDB 的安装与详细使用（一）](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964503&idx=1&sn=1335ece54312f9ed91232ce34d222dd5&chksm=88ede9ffbf9a60e9e03efc28fb4fb469b8212d2ea52208c3db4a1b37b4437b8accbff4a2ed86#rd)  
- [手把手教你 MongoDB 的安装与详细使用（二）](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964506&idx=1&sn=9f473a0797b6f091629a6e42dc870133&chksm=88ede9f2bf9a60e4d20402824eaf9803261276073a988659d3b45c82bf4f9bc7f589a08e2086#rd)  
- [搭建 MongoDB分片（sharding） / 分区 / 集群环境](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964509&idx=1&sn=9476cda64a51956ae7c7a10352ca31b5&chksm=88ede9f5bf9a60e32f03832ced791e6fd95defa18a3ccc854f2a0ad51527a3ffc7d892132fe8#rd)  
- [搭建 SolrCloud 集群服务](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964464&idx=2&sn=f944f824c58acc709aea4e86498af426)
- [搭建 Solr 单机服务](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964464&idx=3&sn=32f023390ebdfe1d30bdf8c109316f83&scene=19#wechat_redirect)
- [搭建 RabbitMQ 集群服务](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964457&idx=3&sn=abb501f40f3314ecf28b5a96ab028f5c)
- [搭建 RabbitMQ 单机服务](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964457&idx=2&sn=7d19d7067e33e262463cd70186b23946)
- [Mycat 读写分离 数据库分库分表 中间件 安装部署，及简单使用](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964443&idx=1&sn=ef6243304e80b32b300b36e88cb7dae9)
- [离线部署 CDH 5.12.1 及使用 CDH 部署 Hadoop 大数据平台集群服务](https://mp.weixin.qq.com/s?__biz=MzA3MTUzOTcxOQ==&mid=2452964430&idx=1&sn=f76c3b3e2fa41fe149fa21dd91061c10)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")