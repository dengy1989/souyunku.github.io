---
layout: post
title: Spring Boot 中使用 MongoDB 增删改查
categories: [MongoDB,SpringBoot]
description: Spring Boot 中使用 MongoDB 增删改查
keywords: MongoDB 
---

本文快速入门,MongoDB 结合SpringBoot starter-data-mongodb 进行增删改查

# 1、什么是MongoDB ?

MongoDB 是由C++语言编写的，是一个基于分布式文件存储的开源数据库系统。

在高负载的情况下，添加更多的节点，可以保证服务器性能。

MongoDB 旨在为WEB应用提供可扩展的高性能数据存储解决方案。

MongoDB 将数据存储为一个文档，数据结构由键值(key=>value)对组成。

MongoDB 文档类似于 JSON 对象。字段值可以包含其他文档，数组及文档数组。

![ ][1]

# 2、MongoDB 优缺点

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

# 3、优缺点详细解释

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


# 4、环境需要

64位操作系统，建议使用Linux / Unix / 

- CentOs 7.3
- MongoDB 3.6.2

# 5、环境安装

**请参考我的另一篇文章**

[搭建 MongoDB分片（sharding） / 分区 / 集群环境](http://www.ymq.io/2018/01/30/MongoDB-3/)

[http://www.ymq.io/2018/01/30/MongoDB-3](http://www.ymq.io/2018/01/30/MongoDB-3/)

# 6、新加项目

新建一个 maven 项目，这里就不详细操作了，大家都会的

不过也可以下载我的示例源码，下载地址如下

**GitHub：**[https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-mongodb](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-mongodb)

**码云：**[https://gitee.com/souyunku/spring-boot-examples/tree/master/spring-boot-mongodb](https://gitee.com/souyunku/spring-boot-examples/tree/master/spring-boot-mongodb)

MongoDB是一个开源NoSQL文档数据库，它使用类似JSON的模式而不是传统的基于表格的关系数据。Spring Boot为MongoDB提供了一些便利，包括 spring-boot-starter-data-mongodb “Starter”。

# 7、添加依赖

在POM 中添加如下依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

# 8、配置文件

在配置文件 `application.properties` 添加如下配置


```sh
spring.application.name=spring-boot-mongodb
# mongodb
spring.data.mongodb.uri=mongodb://192.168.252.121:20000,192.168.252.122:20000,192.168.252.12:20000/demo
```

多个IP集群的配置：

```sh
spring.data.mongodb.uri=mongodb://user:secret@mongo1.example.com:12345,mongo2.example.com:23456/test
```

# 9、DemoEntity

`spring-data-mongodb`中的实体映射是通过`MongoMappingConverter`这个类实现的。它可以通过注释把java类转换为mongodb的文档。

它有以下几种注释：

`@Id` - 文档的唯一标识，在mongodb中为ObjectId，它是唯一的，通过时间戳+机器标识+进程ID+自增计数器（确保同一秒内产生的Id不会冲突）构成。

`@Document` - 把一个java类声明为mongodb的文档，可以通过collection参数指定这个类对应的文档。@Document(collection="mongodb") mongodb对应表

`@DBRef` - 声明类似于关系数据库的关联关系。ps：暂不支持级联的保存功能，当你在本实例中修改了DERef对象里面的值时，单独保存本实例并不能保存DERef引用的对象，它要另外保存，如下面例子的Person和Account。

`@Indexed` - 声明该字段需要索引，建索引可以大大的提高查询效率。

`@CompoundIndex` - 复合索引的声明，建复合索引可以有效地提高多字段的查询效率。

`@GeoSpatialIndexed` - 声明该字段为地理信息的索引。

`@Transient` - 映射忽略的字段，该字段不会保存到mongodb。

`@PersistenceConstructor` - 声明构造函数，作用是把从数据库取出的数据实例化为对象。该构造函数传入的值为从DBObject中取出的数据


```java
package io.ymq.example.mongodb;

@Document(collection = "demo_collection")
public class DemoEntity implements Serializable {

    @Id
    private Long id;

    private String title;

    private String description;

    private String by;

    private String url;
	
	省略 getter setter
}
```

# 10、Demo DAO 接口

提供增删改查 MongoDB 接口
 
```java
package io.ymq.example.mongodb;

public interface DemoDao {

    void saveDemo(DemoEntity demoEntity);

    void removeDemo(Long id);

    void updateDemo(DemoEntity demoEntity);

    DemoEntity findDemoById(Long id);
}
```

# 11、Demo DAO 实现

提供增删改查 MongoDB 接口实现


Spring Data Mongo提供了一个 MongoTemplate类似于Spring的设计的类JdbcTemplate。和JdbcTemplateSpring Boot一样，自动配置一个bean来简单地注入：


 
```java
package io.ymq.example.mongodb;

/**
 * 描述: Demo DAO 实现
 *
 * @author yanpenglei
 * @create 2018-02-03 16:57
 **/
@Component
public class DemoDaoImpl implements DemoDao {

    @Resource
    private MongoTemplate mongoTemplate;

    @Override
    public void saveDemo(DemoEntity demoEntity) {
        mongoTemplate.save(demoEntity);
    }

    @Override
    public void removeDemo(Long id) {
        mongoTemplate.remove(id);
    }

    @Override
    public void updateDemo(DemoEntity demoEntity) {
        Query query = new Query(Criteria.where("id").is(demoEntity.getId()));

        Update update = new Update();
        update.set("title", demoEntity.getTitle());
        update.set("description", demoEntity.getDescription());
        update.set("by", demoEntity.getBy());
        update.set("url", demoEntity.getUrl());

        mongoTemplate.updateFirst(query, update, DemoEntity.class);
    }

    @Override
    public DemoEntity findDemoById(Long id) {
        Query query = new Query(Criteria.where("id").is(id));
        DemoEntity demoEntity = mongoTemplate.findOne(query, DemoEntity.class);
        return demoEntity;
    }

}
```

# 12、启动服务

运行启动类

```java
package io.ymq.example.mongodb;

@SpringBootApplication
public class SpringBootMongodbApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootMongodbApplication.class, args);
    }
}
```

# 13、单元测试

单元测试，提供`mongodb` 的 `demo` 库的 `demo_collection` 集合的增删改查

```java
package io.ymq.example.mongodb;

@RunWith(SpringRunner.class)
@SpringBootTest(classes = SpringBootMongodbApplication.class)
public class SpringBootMongodbApplicationTests {

    @Autowired
    private DemoDao demoDao;

    @Test
    public void saveDemoTest() {

        DemoEntity demoEntity = new DemoEntity();
        demoEntity.setId(1L);
        demoEntity.setTitle("Spring Boot 中使用 MongoDB");
        demoEntity.setDescription("关注公众号，搜云库，专注于开发技术的研究与知识分享");
        demoEntity.setBy("souyunku");
        demoEntity.setUrl("http://www.souyunku.com");

        demoDao.saveDemo(demoEntity);

        demoEntity = new DemoEntity();
        demoEntity.setId(2L);
        demoEntity.setTitle("Spring Boot 中使用 MongoDB");
        demoEntity.setDescription("关注公众号，搜云库，专注于开发技术的研究与知识分享");
        demoEntity.setBy("souyunku");
        demoEntity.setUrl("http://www.souyunku.com");

        demoDao.saveDemo(demoEntity);
    }

    @Test
    public void removeDemoTest() {
        demoDao.removeDemo(2L);
    }

    @Test
    public void updateDemoTest() {

        DemoEntity demoEntity = new DemoEntity();
        demoEntity.setId(1L);
        demoEntity.setTitle("Spring Boot 中使用 MongoDB 更新数据");
        demoEntity.setDescription("关注公众号，搜云库，专注于开发技术的研究与知识分享");
        demoEntity.setBy("souyunku");
        demoEntity.setUrl("http://www.souyunku.com");

        demoDao.updateDemo(demoEntity);
    }

    @Test
    public void findDemoByIdTest() {

        DemoEntity demoEntity = demoDao.findDemoById(1L);

        System.out.println(JSONObject.toJSONString(demoEntity));
    }
}
```

**扩展阅读**

手把手教你 MongoDB 的安装与详细使用（一）

[http://www.ymq.io/2018/01/26/MongoDB-1/](http://www.ymq.io/2018/01/26/MongoDB-1/)

手把手教你 MongoDB 的安装与详细使用（二）

[http://www.ymq.io/2018/01/29/MongoDB-2/](http://www.ymq.io/2018/01/29/MongoDB-2/)

# 源码下载

**GitHub：**[https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-mongodb](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-mongodb)

**码云：**[https://gitee.com/souyunku/spring-boot-examples/tree/master/spring-boot-mongodb](https://gitee.com/souyunku/spring-boot-examples/tree/master/spring-boot-mongodb)

# 福利

**[Spring Boot 学习教程](https://github.com/souyunku/spring-boot-examples) 会不定时更新技术, 欢迎 Star 和 Fork。**

[https://github.com/souyunku/spring-boot-examples](https://github.com/souyunku/spring-boot-examples)

![ ][2]

[https://github.com/souyunku/spring-boot-examples](https://github.com/souyunku/spring-boot-examples)

[2]: http://www.ymq.io/images/2018/SpringBoot/1.png

[1]: http://www.ymq.io/images/2018/MongoDB/1.png

**参考：**[https://docs.spring.io/spring-boot/docs/1.5.1.RELEASE/reference/html/boot-features-nosql.html#boot-features-mongodb](https://docs.spring.io/spring-boot/docs/1.5.1.RELEASE/reference/html/boot-features-nosql.html#boot-features-mongodb)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/02/05/spring-boot-mongodb-example](http://www.ymq.io/2018/02/05/spring-boot-mongodb-example)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

