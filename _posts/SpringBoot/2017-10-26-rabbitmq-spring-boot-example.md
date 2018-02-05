---
layout: post
title: Spring Boot 中使用 RabbitMQ
categories: [RabbitMQ,SpringBoot]
description: Spring Boot 中使用 RabbitMQ
keywords: RabbitMQ 
---

# 一、什么是MongoDB ?

MongoDB 是由C++语言编写的，是一个基于分布式文件存储的开源数据库系统。

在高负载的情况下，添加更多的节点，可以保证服务器性能。

MongoDB 旨在为WEB应用提供可扩展的高性能数据存储解决方案。

MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。

MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档，数组及文档数组。

![ ][1]

# 二、MongoDB 优缺点

**优点**

- 文档结构的存储方式，能够更便捷的获取数据
- 内置GridFS，支持大容量的存储
- 海量数据下，性能优越
- 动态查询
- 全索引支持,扩展到内部对象和内嵌数组
- 查询记录分析
- 快速,就地更新
- 高效存储二进制大对象 (比如照片和视频)
- 复制（复制集）和支持自动故障恢复
- 内置 Auto- Sharding 自动分片支持云级扩展性，分片简单
- MapReduce 支持复杂聚合
- 商业支持,培训和咨询


**缺点**

- 不支持事务操作
- MongoDB 占用空间过大 （不过这个确定对于目前快速下跌的硬盘价格来说，也不算什么缺点了）
- MongoDB没有如MySQL那样成熟的维护工具
- 无法进行关联表查询，不适用于关系多的数据
- 复杂聚合操作通过mapreduce创建，速度慢
- 模式自由,自由灵活的文件存储格式带来的数据错
- MongoDB 在你删除记录后不会在文件系统回收空间。除非你删掉数据库。但是空间没有被浪费

# 三、优缺点详细解释

**1.内置GridFS，支持大容量的存储：**

GridFS是一个出色的分布式文件系统，可以支持海量的数据存储。
内置了GridFS了MongoDB，能够满足对大数据集的快速范围查询。

**2.内置 Auto- Sharding 自动分片支持云级扩展性，分片简单**

提供基于Range的Auto Sharding机制：

一个collection可按照记录的范围，分成若干个段，切分到不同的Shard上。

Shards可以和复制结合，配合Replica sets能够实现Sharding+fail-over，不同的Shard之间可以负载均衡。  
查询是对客户端是透明的。客户端执行查询，统计，MapReduce等操作，这些会被MongoDB自动路由到后端的数据节点。  
这让我们关注于自己的业务，适当的 时候可以无痛的升级。MongoDB的Sharding设计能力最大可支持约20 petabytes，足以支撑一般应用。  
这可以保证MongoDB运行在便宜的PC服务器集群上。PC集群扩充起来非常方便并且成本很低，避免了“sharding”操作的复杂性和成本。  
 
**3.海量数据下，性能优越：**

在使用场合下，千万级别的文档对象，近10G的数据，对有索引的ID的查询不会比mysql慢，而对非索引字段的查询，则是全面胜出。 mysql实际无法胜任大数据量下任意字段的查询，而mongodb的查询性能实在让我惊讶。写入性能同样很令人满意，同样写入百万级别的数 据，mongodb比我以前试用过的couchdb要快得多，基本10分钟以下可以解决。补上一句，观察过程中mongodb都远算不上是CPU杀手。

**4.全索引支持,扩展到内部对象和内嵌数组**

索引通常能够极大的提高查询的效率，如果没有索引，MongoDB在读取数据时必须扫描集合中的每个文件并选取那些符合查询条件的记录。

这种扫描全集合的查询效率是非常低的，特别在处理大量的数据时，查询可以要花费几十秒甚至几分钟，这对网站的性能是非常致命的。

索引是特殊的数据结构，索引存储在一个易于遍历读取的数据集合中，索引是对数据库表中一列或多列的值进行排序的一种结构。

**5.MapReduce 支持复杂聚合**

MongoDB中聚合(aggregate)主要用于处理数据(诸如统计平均值,求和等)，并返回计算后的数据结果。有点类似sql语句中的 count(*)。

**与关系型数据库相比，MongoDB的缺点：**

**mongodb不支持事务操作：**

所以事务要求严格的系统（如果银行系统）肯定不能用它。

**mongodb不支持事务操作：**

所以事务要求严格的系统（如果银行系统）肯定不能用它。

**mongodb占用空间过大：**

关于其原因，在官方的FAQ中，提到有如下几个方面：

1、空间的预分配：为避免形成过多的硬盘碎片，mongodb每次空间不足时都会申请生成一大块的硬盘空间，而且申请的量从64M、128M、256M那 样的指数递增，直到2G为单个文件的最大体积。随着数据量的增加，你可以在其数据目录里看到这些整块生成容量不断递增的文件。

2、字段名所占用的空间：为了保持每个记录内的结构信息用于查询，mongodb需要把每个字段的key-value都以BSON的形式存储，如果 value域相对于key域并不大，比如存放数值型的数据，则数据的overhead是最大的。一种减少空间占用的方法是把字段名尽量取短一些，这样占用 空间就小了，但这就要求在易读性与空间占用上作为权衡了。

3、删除记录不释放空间：这很容易理解，为避免记录删除后的数据的大规模挪动，原记录空间不删除，只标记“已删除”即可，以后还可以重复利用。

4、可以定期运行db.repairDatabase()来整理记录，但这个过程会比较缓慢

MongoDB没有如MySQL那样成熟的维护工具，这对于开发和IT运营都是个值得注意的地方。

# 新建项目







[1]: http://www.ymq.io/images/2017/rabbit/example/1.png
[2]: http://www.ymq.io/images/2017/rabbit/example/2.png
[3]: http://www.ymq.io/images/2017/rabbit/example/3.png
[4]: http://www.ymq.io/images/2017/rabbit/example/4.png



代码我已放到 Github ，导入`spring-boot-rabbitmq` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rabbitmq](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-rabbitmq)


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2017/10/26/rabbitmq-spring-boot-example](http://www.ymq.io/2017/10/26/rabbitmq-spring-boot-example/)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")


