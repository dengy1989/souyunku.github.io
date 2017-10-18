---
layout: post
title: Spring Boot 中使用 SolrCloud 
categories: SolrCloud 
description: Spring Boot 中使用 SolrCloud 
keywords: SolrCloud  
---

Lucene是一个Java语言编写的利用倒排原理实现的文本检索类库；

Solr是以Lucene为基础实现的文本检索应用服务。Solr部署方式有单机方式、多机Master-Slaver方式、Cloud方式。

SolrCloud是基于Solr和Zookeeper的分布式搜索方案。当索引越来越大，一个单一的系统无法满足磁盘需求，查询速度缓慢，此时就需要分布式索引。在分布式索引中，原来的大索引，将会分成多个小索引，solr可以将这些小索引返回的结果合并，然后返回给客户端。

# 准备

## 环境安装 

[CentOs7.3 搭建 SolrCloud 集群服务](https://segmentfault.com/a/1190000010836061)

# 测试用例

# Github 代码

代码我已放到 Github ，导入`ymq-solr-cloud-spring-boot` 项目 

github [https://github.com/souyunku/ymq-example/tree/master/ymq-solr-cloud-spring-boot](https://github.com/souyunku/ymq-example/tree/master/ymq-solr-cloud-spring-boot)

## 添加依赖

在项目中添加 `kafka-clients` 依赖

```xml
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-data-solr</artifactId>
</dependency>
<dependency>
	<groupId>org.springframework.data</groupId>
	<artifactId>spring-data-jpa</artifactId>
</dependency>
```

## 启用 Solr

```java
@Configuration
@EnableSolrRepositories(basePackages = {"io.ymq.solr"}, multicoreSupport = true)
public class SolrConfig {

    @Value("${spring.data.solr.zk-host}")
    private String zkHost;

    @Bean
    public CloudSolrClient solrClient() {
        return new CloudSolrClient(zkHost);
    }
	
}
```

## 映射的实体类

```java
@SolrDocument(solrCoreName = "test_collection")
public class Ymq implements Serializable {

    @Id
    @Field
    private String id;

    @Field
    private String ymqTitle;

    @Field
    private String ymqUrl;

    @Field
    private String ymqContent;


  get 。。。

  set 。。。
}
```


## 继承 SolrCrudRepository

```java
public interface YmqRepository extends SolrCrudRepository<Ymq, String> {

    /**
     * 通过标题查询
     *
     * @param ymqTitle
     * @return
     */
    @Query(" ymqTitle:*?0* ")
    public List<Ymq> findByQueryAnnotation(String ymqTitle);
}
```

## 参数配置

**`application.properties`**

```
#SolrCloud zookeeper
spring.data.solr.zk-host=node1:2181,node2:2181,node3:2181
```

## 单元测试

```java
package io.ymq.solr.test;

import com.alibaba.fastjson.JSONObject;
import io.ymq.solr.YmqRepository;
import io.ymq.solr.po.Ymq;
import io.ymq.solr.run.Startup;
import org.apache.solr.client.solrj.SolrQuery;
import org.apache.solr.client.solrj.SolrServerException;
import org.apache.solr.client.solrj.impl.CloudSolrClient;

import org.apache.solr.client.solrj.response.QueryResponse;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import java.io.IOException;
import java.util.List;


/**
 * 描述: 测试 solr cloud
 *
 * @author yanpenglei
 * @create 2017-10-17 19:00
 **/
@RunWith(SpringRunner.class)
@SpringBootTest(classes = Startup.class)
public class BaseTest {

    @Autowired
    private YmqRepository ymqRepository;

    @Autowired
    private CloudSolrClient cloudSolrClient;

    /**
     * 使用 ymqRepository 方式新增
     *
     * @throws Exception
     */
    @Test
    public void testAddYmqRepository() {

        Ymq ymq1 = new Ymq();
        ymq1.setId("1");
        ymq1.setYmqTitle("penglei");
        ymq1.setYmqUrl("www_ymq_io");
        ymq1.setYmqContent("ymqContent");

        Ymq ymq2 = new Ymq();
        ymq2.setId("2");//
        ymq2.setYmqTitle("penglei");
        ymq2.setYmqUrl("www_ymq_io");
        ymq2.setYmqContent("ymqContent");

        ymqRepository.save(ymq1);
        ymqRepository.save(ymq2);
    }


    /**
     * 使用 cloudSolrClient 方式新增
     *
     * @throws Exception
     */
    @Test
    public void testAddCloudSolrClient() throws IOException, SolrServerException {

        Ymq ymq = new Ymq();
        ymq.setId("3");
        ymq.setYmqTitle("penglei");
        ymq.setYmqUrl("www_ymq_io");
        ymq.setYmqContent("ymqContent");

        cloudSolrClient.setDefaultCollection("test_collection");
        cloudSolrClient.connect();

        cloudSolrClient.addBean(ymq);
        cloudSolrClient.commit();
    }

    /**
     * 删除数据
     */
    @Test
    public void testDelete() {

        Ymq ymq = new Ymq();
        ymq.setId("4");
        ymq.setYmqTitle("delete_penglei");
        ymq.setYmqUrl("www_ymq_io");
        ymq.setYmqContent("ymqContent");

        // 添加一条测试数据，用于删除的测试数据
        ymqRepository.save(ymq);

        // 通过标题查询数据ID
        List<Ymq> list = ymqRepository.findByQueryAnnotation("delete_penglei");

        for (Ymq item : list) {

            System.out.println("查询响应 :"+JSONObject.toJSONString(item));

            //通过主键 ID 删除
            ymqRepository.delete(item.getId());
        }

    }

    /**
     * data JPA 方式查询
     *
     * @throws Exception
     */
    @Test
    public void testYmqRepositorySearch() throws Exception {

        List<Ymq> list = ymqRepository.findByQueryAnnotation("penglei");

        for (Ymq item : list) {
            System.out.println(" data JPA 方式查询响应 :"+JSONObject.toJSONString(item));
        }
    }

    /**
     * SolrQuery 语法查询
     *
     * @throws Exception
     */
    @Test
    public void testYmqSolrQuery() throws Exception {

        SolrQuery query = new SolrQuery();

        String ymqTitle = "penglei";

        query.setQuery(" ymqTitle:*" + ymqTitle + "* ");

        cloudSolrClient.setDefaultCollection("test_collection");
        cloudSolrClient.connect();
        QueryResponse response = cloudSolrClient.query(query);

        List<Ymq> list = response.getBeans(Ymq.class);

        for (Ymq item : list) {
            System.out.println("SolrQuery 语法查询响应 :"+JSONObject.toJSONString(item));
        }
    }

}

```

一些查询,响应

```
 data JPA 方式查询响应 :{"id":"1","ymqContent":"ymqContent","ymqTitle":"penglei","ymqUrl":"www_ymq_io"}
 data JPA 方式查询响应 :{"id":"2","ymqContent":"ymqContent","ymqTitle":"penglei","ymqUrl":"www_ymq_io"}
 data JPA 方式查询响应 :{"id":"3","ymqContent":"ymqContent","ymqTitle":"penglei","ymqUrl":"www_ymq_io"}
```


代码我已放到 Github ，导入`ymq-solr-cloud-spring-boot` 项目 

github [https://github.com/souyunku/ymq-example/tree/master/ymq-solr-cloud-spring-boot](https://github.com/souyunku/ymq-example/tree/master/ymq-solr-cloud-spring-boot)

