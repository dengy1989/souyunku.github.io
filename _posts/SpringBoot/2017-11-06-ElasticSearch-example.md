---
layout: post
title: Spring Boot 中使用  Java API 调用 Elasticsearch
categories: [ElasticSearch,SpringBoot]
description: Spring Boot 中使用  Java API 调用 Elasticsearch
keywords: ElasticSearch  
---

ElasticSearch 是一个高可用开源全文检索和分析组件。提供存储服务，搜索服务，大数据准实时分析等。一般用于提供一些提供复杂搜索的应用。

ElasticSearch 提供了一套基于restful风格的全文检索服务组件。前身是compass，直到2010被一家公司接管进行维护，开始商业化，并提供了ElasticSearch 一些相关的产品，包括大家比较熟悉的 kibana、logstash 以及 ElasticSearch 的一些组件，比如 安全组件shield 。当前最新的Elasticsearch Reference: 版本为 5.6 ，比较应用广泛的为2.X，直到 2016-12 推出了5.x 版本 ，将版本号调为 5.X 。这是为了和 kibana 和 logstash 等产品版本号进行统一 ElasticSearch 。


准实时：ElasticSearch 是一个准实时的搜索工具，在一般情况下延时少于一秒。

# 特点

支持物理上的水平扩展，并拥有一套分布式协调的管理功能

## 操作简单

单节点的ES，安装启动后，会默认创建一个名为elasticsearch的es集群。如果在局域网中存在该clustr.name,会自动加入该集群。形成一个ElasticSearch 集群 。

多节点ES，在同一个局域网内的ES服务，只需要配置为同一个clust.name 名称即可成为 
一个ES集群。集群能够将同一个索引的分片，自动分布到各个节点。并在高效的提供查询服务的同时，自动协调每个节点的下线以及上线情况。


## restful 风格的API

提供了一套关于索引以及状态查看的restful风格接口。至于什么是Restful风格服务，请移步

[谈谈到底什么是rest风格架构设计？](http://blog.csdn.net/cfl20121314/article/details/48982857)


## 对比Solr

Solr与ES都是基于java/lucence来做一套面向文档结构的Nosql结构的数据库。


## 支持的数据结构

solr支持 xml json html 等多种数据结构，而ES 仅支持json这种结构。

## 性能

solr在新建索引时是IO阻塞的，所以如果在新建索引时同时进行搜索这时候相比ES来的相对较快。所以在实时性上，ElasticSearch 相比还是更好的选择。


# 基本概念

## Index

定义：类似于mysql中的database。索引只是一个逻辑上的空间，物理上是分为多个文件来管理的。

命名：必须全小写

ES中index可能被分为多个分片【对应物理上的lcenne索引】，在实践过程中每个index都会有一个相应的副 
本。主要用来在硬件出现问题时，用来回滚数据的。这也某种程序上，加剧了ES对于内存高要求

## Type

定义：类似于mysql中的table，根据用户需求每个index中可以新建任意数量的type。

## Document

定义：对应mysql中的row。有点类似于MongoDB中的文档结构，每个Document是一个json格式的文本。

## Mapping

更像是一个用来定义每个字段类型的语义规范在mysql中类似sql语句，在ES中经过包装后，都被封装为友好的Restful风格的接口进行操作。这一点也是为什么开发人员更愿意使用ES或者compass这样的框架而不是直接使用Lucene的一个原因。

## Shards & Replicas

定义：能够为每个索引提供水平的扩展以及备份操作。

描述：

**Shards:**在单个节点中，index的存储始终是有限制，并且随着存储的增大会带来性能的问题。为了解决这个问题，ElasticSearch提供一个能够分割单个index到集群各个节点的功能。你可以在新建这个索引时，手动的定义每个索引分片的数量。

**Replicas:**在每个node出现宕机或者下线的情况，Replicas能够在该节点下线的同时将副本同时自动分配到其他仍然可用的节点。而且在提供搜索的同时，允许进行扩展节点的数量，在这个期间并不会出现服务终止的情况。

默认情况下，每个索引会分配5个分片，并且对应5个分片副本，同时会出现一个完整的副本【包括5个分配的副本数据】。


言而总之，用一句话来总结下。ElasticSearch 是一个基于 lucence 可水平扩展的自动化近实时全文搜索服务组件。

[Elasticsearch 官方参考文档 ](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)

# 准备

## 环境安装 

只需要参考 Elasticsearch 安装部分

[ELK 集群 + Redis 集群 + Nginx ,分布式的实时日志（数据）搜集和分析的监控系统搭建，简单上手使用](https://segmentfault.com/a/1190000010975383)

# 测试用例

# Github 代码

代码我已放到 Github ，导入`spring-boot-elasticsearch-demo` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-elasticsearch-demo](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-elasticsearch-demo)

## 添加依赖

```xml
<dependency>
	<groupId>org.elasticsearch</groupId>
	<artifactId>elasticsearch</artifactId>
	<version>5.5.3</version>
</dependency>
<dependency>
	<groupId>org.elasticsearch.client</groupId>
	<artifactId>transport</artifactId>
	<version>5.5.3</version>
</dependency>
```

## 配置ES Client

```java
@Configuration
public class ElasticsearchConfig {

    private static final Logger LOGGER = LoggerFactory.getLogger(ElasticsearchConfig.class);

    /**
     * elk集群地址
     */
    @Value("${elasticsearch.ip}")
    private String hostName;
    /**
     * 端口
     */
    @Value("${elasticsearch.port}")
    private String port;
    /**
     * 集群名称
     */
    @Value("${elasticsearch.cluster.name}")
    private String clusterName;

    /**
     * 连接池
     */
    @Value("${elasticsearch.pool}")
    private String poolSize;

    @Bean
    public TransportClient init() {

        TransportClient transportClient = null;

        try {
            // 配置信息
            Settings esSetting = Settings.builder()
                    .put("cluster.name", clusterName)
                    .put("client.transport.sniff", true)//增加嗅探机制，找到ES集群
                    .put("thread_pool.search.size", Integer.parseInt(poolSize))//增加线程池个数，暂时设为5
                    .build();

            transportClient = new PreBuiltTransportClient(esSetting);
            InetSocketTransportAddress inetSocketTransportAddress = new InetSocketTransportAddress(InetAddress.getByName(hostName), Integer.valueOf(port));
            transportClient.addTransportAddresses(inetSocketTransportAddress);

        } catch (Exception e) {
            LOGGER.error("elasticsearch TransportClient create error!!!", e);
        }

        return transportClient;
    }
}

```

## 参数配置

**`application.properties`**

```java
# Elasticsearch
elasticsearch.cluster.name=ymq
elasticsearch.ip=192.168.252.121
elasticsearch.port=9300
elasticsearch.pool=5
```

## ES 工具类

**`Elasticsearch 工具类`**

```java
@Component
public class ElasticsearchUtils {

    private static final Logger LOGGER = LoggerFactory.getLogger(ElasticsearchUtils.class);

    @Autowired
    private TransportClient transportClient;

    private static TransportClient client;

    @PostConstruct
    public void init() {
        client = this.transportClient;
    }

    /**
     * 创建索引
     *
     * @param index
     * @return
     */
    public static boolean createIndex(String index) {
        if (!isIndexExist(index)) {
            LOGGER.info("Index is not exits!");
        }
        CreateIndexResponse indexresponse = client.admin().indices().prepareCreate(index).execute().actionGet();
        LOGGER.info("执行建立成功？" + indexresponse.isAcknowledged());

        return indexresponse.isAcknowledged();
    }

    /**
     * 删除索引
     *
     * @param index
     * @return
     */
    public static boolean deleteIndex(String index) {
        if (!isIndexExist(index)) {
            LOGGER.info("Index is not exits!");
        }
        DeleteIndexResponse dResponse = client.admin().indices().prepareDelete(index).execute().actionGet();
        if (dResponse.isAcknowledged()) {
            LOGGER.info("delete index " + index + "  successfully!");
        } else {
            LOGGER.info("Fail to delete index " + index);
        }
        return dResponse.isAcknowledged();
    }

    /**
     * 判断索引是否存在
     *
     * @param index
     * @return
     */
    public static boolean isIndexExist(String index) {
        IndicesExistsResponse inExistsResponse = client.admin().indices().exists(new IndicesExistsRequest(index)).actionGet();
        if (inExistsResponse.isExists()) {
            LOGGER.info("Index [" + index + "] is exist!");
        } else {
            LOGGER.info("Index [" + index + "] is not exist!");
        }
        return inExistsResponse.isExists();
    }

    /**
     * 数据添加，正定ID
     *
     * @param jsonObject 要增加的数据
     * @param index      索引，类似数据库
     * @param type       类型，类似表
     * @param id         数据ID
     * @return
     */
    public static String addData(JSONObject jsonObject, String index, String type, String id) {

        IndexResponse response = client.prepareIndex(index, type, id).setSource(jsonObject).get();

        LOGGER.info("addData response status:{},id:{}", response.status().getStatus(), response.getId());

        return response.getId();
    }

    /**
     * 数据添加
     *
     * @param jsonObject 要增加的数据
     * @param index      索引，类似数据库
     * @param type       类型，类似表
     * @return
     */
    public static String addData(JSONObject jsonObject, String index, String type) {
        return addData(jsonObject, index, type, UUID.randomUUID().toString().replaceAll("-", "").toUpperCase());
    }

    /**
     * 通过ID删除数据
     *
     * @param index 索引，类似数据库
     * @param type  类型，类似表
     * @param id    数据ID
     */
    public static void deleteDataById(String index, String type, String id) {

        DeleteResponse response = client.prepareDelete(index, type, id).execute().actionGet();

        LOGGER.info("deleteDataById response status:{},id:{}", response.status().getStatus(), response.getId());
    }

    /**
     * 通过ID 更新数据
     *
     * @param jsonObject 要增加的数据
     * @param index      索引，类似数据库
     * @param type       类型，类似表
     * @param id         数据ID
     * @return
     */
    public static void updateDataById(JSONObject jsonObject, String index, String type, String id) {

        UpdateRequest updateRequest = new UpdateRequest();

        updateRequest.index(index).type(type).id(id).doc(jsonObject);

        client.update(updateRequest);

    }

    /**
     * 通过ID获取数据
     *
     * @param index  索引，类似数据库
     * @param type   类型，类似表
     * @param id     数据ID
     * @param fields 需要显示的字段，逗号分隔（缺省为全部字段）
     * @return
     */
    public static Map<String, Object> searchDataById(String index, String type, String id, String fields) {

        GetRequestBuilder getRequestBuilder = client.prepareGet(index, type, id);

        if (StringUtils.isNotEmpty(fields)) {
            getRequestBuilder.setFetchSource(fields.split(","), null);
        }

        GetResponse getResponse =  getRequestBuilder.execute().actionGet();

        return getResponse.getSource();
    }


    /**
     * 使用分词查询
     *
     * @param index    索引名称
     * @param type     类型名称,可传入多个type逗号分隔
     * @param fields   需要显示的字段，逗号分隔（缺省为全部字段）
     * @param matchStr 过滤条件（xxx=111,aaa=222）
     * @return
     */
    public static List<Map<String, Object>> searchListData(String index, String type, String fields, String matchStr) {
        return searchListData(index, type, 0, 0, null, fields, null, false, null, matchStr);
    }

    /**
     * 使用分词查询
     *
     * @param index       索引名称
     * @param type        类型名称,可传入多个type逗号分隔
     * @param fields      需要显示的字段，逗号分隔（缺省为全部字段）
     * @param sortField   排序字段
     * @param matchPhrase true 使用，短语精准匹配
     * @param matchStr    过滤条件（xxx=111,aaa=222）
     * @return
     */
    public static List<Map<String, Object>> searchListData(String index, String type, String fields, String sortField, boolean matchPhrase, String matchStr) {
        return searchListData(index, type, 0, 0, null, fields, sortField, matchPhrase, null, matchStr);
    }


    /**
     * 使用分词查询
     *
     * @param index          索引名称
     * @param type           类型名称,可传入多个type逗号分隔
     * @param size           文档大小限制
     * @param fields         需要显示的字段，逗号分隔（缺省为全部字段）
     * @param sortField      排序字段
     * @param matchPhrase    true 使用，短语精准匹配
     * @param highlightField 高亮字段
     * @param matchStr       过滤条件（xxx=111,aaa=222）
     * @return
     */
    public static List<Map<String, Object>> searchListData(String index, String type, Integer size, String fields, String sortField, boolean matchPhrase, String highlightField, String matchStr) {
        return searchListData(index, type, 0, 0, size, fields, sortField, matchPhrase, highlightField, matchStr);
    }


    /**
     * 使用分词查询
     *
     * @param index          索引名称
     * @param type           类型名称,可传入多个type逗号分隔
     * @param startTime      开始时间
     * @param endTime        结束时间
     * @param size           文档大小限制
     * @param fields         需要显示的字段，逗号分隔（缺省为全部字段）
     * @param sortField      排序字段
     * @param matchPhrase    true 使用，短语精准匹配
     * @param highlightField 高亮字段
     * @param matchStr       过滤条件（xxx=111,aaa=222）
     * @return
     */
    public static List<Map<String, Object>> searchListData(String index, String type, long startTime, long endTime, Integer size, String fields, String sortField, boolean matchPhrase, String highlightField, String matchStr) {

        SearchRequestBuilder searchRequestBuilder = client.prepareSearch(index);
        if (StringUtils.isNotEmpty(type)) {
            searchRequestBuilder.setTypes(type.split(","));
        }
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();

        if (startTime > 0 && endTime > 0) {
            boolQuery.must(QueryBuilders.rangeQuery("processTime")
                    .format("epoch_millis")
                    .from(startTime)
                    .to(endTime)
                    .includeLower(true)
                    .includeUpper(true));
        }

        //搜索的的字段
        if (StringUtils.isNotEmpty(matchStr)) {
            for (String s : matchStr.split(",")) {
                String[] ss = s.split("=");
                if (ss.length > 1) {
                    if (matchPhrase == Boolean.TRUE) {
                        boolQuery.must(QueryBuilders.matchPhraseQuery(s.split("=")[0], s.split("=")[1]));
                    } else {
                        boolQuery.must(QueryBuilders.matchQuery(s.split("=")[0], s.split("=")[1]));
                    }
                }

            }
        }

        // 高亮（xxx=111,aaa=222）
        if (StringUtils.isNotEmpty(highlightField)) {
            HighlightBuilder highlightBuilder = new HighlightBuilder();

            //highlightBuilder.preTags("<span style='color:red' >");//设置前缀
            //highlightBuilder.postTags("</span>");//设置后缀

            // 设置高亮字段
            highlightBuilder.field(highlightField);
            searchRequestBuilder.highlighter(highlightBuilder);
        }


        searchRequestBuilder.setQuery(boolQuery);

        if (StringUtils.isNotEmpty(fields)) {
            searchRequestBuilder.setFetchSource(fields.split(","), null);
        }
        searchRequestBuilder.setFetchSource(true);

        if (StringUtils.isNotEmpty(sortField)) {
            searchRequestBuilder.addSort(sortField, SortOrder.DESC);
        }

        if (size != null && size > 0) {
            searchRequestBuilder.setSize(size);
        }

        //打印的内容 可以在 Elasticsearch head 和 Kibana  上执行查询
        LOGGER.info("\n{}", searchRequestBuilder);

        SearchResponse searchResponse = searchRequestBuilder.execute().actionGet();

        long totalHits = searchResponse.getHits().totalHits;
        long length = searchResponse.getHits().getHits().length;

        LOGGER.info("共查询到[{}]条数据,处理数据条数[{}]", totalHits, length);

        if (searchResponse.status().getStatus() == 200) {
            // 解析对象
            return setSearchResponse(searchResponse, highlightField);
        }

        return null;

    }

    /**
     * 使用分词查询,并分页
     *
     * @param index          索引名称
     * @param type           类型名称,可传入多个type逗号分隔
     * @param currentPage    当前页
     * @param pageSize       每页显示条数
     * @param startTime      开始时间
     * @param endTime        结束时间
     * @param fields         需要显示的字段，逗号分隔（缺省为全部字段）
     * @param sortField      排序字段
     * @param matchPhrase    true 使用，短语精准匹配
     * @param highlightField 高亮字段
     * @param matchStr       过滤条件（xxx=111,aaa=222）
     * @return
     */
    public static EsPage searchDataPage(String index, String type, int currentPage, int pageSize, long startTime, long endTime, String fields, String sortField, boolean matchPhrase, String highlightField, String matchStr) {
        SearchRequestBuilder searchRequestBuilder = client.prepareSearch(index);
        if (StringUtils.isNotEmpty(type)) {
            searchRequestBuilder.setTypes(type.split(","));
        }
        searchRequestBuilder.setSearchType(SearchType.QUERY_THEN_FETCH);

        // 需要显示的字段，逗号分隔（缺省为全部字段）
        if (StringUtils.isNotEmpty(fields)) {
            searchRequestBuilder.setFetchSource(fields.split(","), null);
        }

        //排序字段
        if (StringUtils.isNotEmpty(sortField)) {
            searchRequestBuilder.addSort(sortField, SortOrder.DESC);
        }

        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();

        if (startTime > 0 && endTime > 0) {
            boolQuery.must(QueryBuilders.rangeQuery("processTime")
                    .format("epoch_millis")
                    .from(startTime)
                    .to(endTime)
                    .includeLower(true)
                    .includeUpper(true));
        }

        // 查询字段
        if (StringUtils.isNotEmpty(matchStr)) {
            for (String s : matchStr.split(",")) {
                String[] ss = s.split("=");
                if (matchPhrase == Boolean.TRUE) {
                    boolQuery.must(QueryBuilders.matchPhraseQuery(s.split("=")[0], s.split("=")[1]));
                } else {
                    boolQuery.must(QueryBuilders.matchQuery(s.split("=")[0], s.split("=")[1]));
                }
            }
        }

        // 高亮（xxx=111,aaa=222）
        if (StringUtils.isNotEmpty(highlightField)) {
            HighlightBuilder highlightBuilder = new HighlightBuilder();

            //highlightBuilder.preTags("<span style='color:red' >");//设置前缀
            //highlightBuilder.postTags("</span>");//设置后缀

            // 设置高亮字段
            highlightBuilder.field(highlightField);
            searchRequestBuilder.highlighter(highlightBuilder);
        }

        searchRequestBuilder.setQuery(QueryBuilders.matchAllQuery());
        searchRequestBuilder.setQuery(boolQuery);

        // 分页应用
        searchRequestBuilder.setFrom(currentPage).setSize(pageSize);

        // 设置是否按查询匹配度排序
        searchRequestBuilder.setExplain(true);

        //打印的内容 可以在 Elasticsearch head 和 Kibana  上执行查询
        LOGGER.info("\n{}", searchRequestBuilder);

        // 执行搜索,返回搜索响应信息
        SearchResponse searchResponse = searchRequestBuilder.execute().actionGet();

        long totalHits = searchResponse.getHits().totalHits;
        long length = searchResponse.getHits().getHits().length;

        LOGGER.debug("共查询到[{}]条数据,处理数据条数[{}]", totalHits, length);

        if (searchResponse.status().getStatus() == 200) {
            // 解析对象
            List<Map<String, Object>> sourceList = setSearchResponse(searchResponse, highlightField);

            return new EsPage(currentPage, pageSize, (int) totalHits, sourceList);
        }

        return null;

    }

    /**
     * 高亮结果集 特殊处理
     *
     * @param searchResponse
     * @param highlightField
     */
    private static List<Map<String, Object>> setSearchResponse(SearchResponse searchResponse, String highlightField) {
        List<Map<String, Object>> sourceList = new ArrayList<Map<String, Object>>();
        StringBuffer stringBuffer = new StringBuffer();

        for (SearchHit searchHit : searchResponse.getHits().getHits()) {
            searchHit.getSource().put("id", searchHit.getId());

            if (StringUtils.isNotEmpty(highlightField)) {

                System.out.println("遍历 高亮结果集，覆盖 正常结果集" + searchHit.getSource());
                Text[] text = searchHit.getHighlightFields().get(highlightField).getFragments();

                if (text != null) {
                    for (Text str : text) {
                        stringBuffer.append(str.string());
                    }
                    //遍历 高亮结果集，覆盖 正常结果集
                    searchHit.getSource().put(highlightField, stringBuffer.toString());
                }
            }
            sourceList.add(searchHit.getSource());
        }

        return sourceList;
    }

}

```

**EsPage.java**


```java
public class EsPage {

    // 指定的或是页面参数
    private int currentPage; // 当前页
    private int pageSize; // 每页显示多少条

    // 查询es结果
    private int recordCount; // 总记录数
    private List<Map<String, Object>> recordList; // 本页的数据列表

    // 计算
    private int pageCount; // 总页数
    private int beginPageIndex; // 页码列表的开始索引（包含）
    private int endPageIndex; // 页码列表的结束索引（包含）

    /**
     * 只接受前4个必要的属性，会自动的计算出其他3个属性的值
     *
     * @param currentPage
     * @param pageSize
     * @param recordCount
     * @param recordList
     */
    public EsPage(int currentPage, int pageSize, int recordCount, List<Map<String, Object>> recordList) {
        this.currentPage = currentPage;
        this.pageSize = pageSize;
        this.recordCount = recordCount;
        this.recordList = recordList;

        // 计算总页码
        pageCount = (recordCount + pageSize - 1) / pageSize;

        // 计算 beginPageIndex 和 endPageIndex
        // >> 总页数不多于10页，则全部显示
        if (pageCount <= 10) {
            beginPageIndex = 1;
            endPageIndex = pageCount;
        }
        // >> 总页数多于10页，则显示当前页附近的共10个页码
        else {
            // 当前页附近的共10个页码（前4个 + 当前页 + 后5个）
            beginPageIndex = currentPage - 4;
            endPageIndex = currentPage + 5;
            // 当前面的页码不足4个时，则显示前10个页码
            if (beginPageIndex < 1) {
                beginPageIndex = 1;
                endPageIndex = 10;
            }
            // 当后面的页码不足5个时，则显示后10个页码
            if (endPageIndex > pageCount) {
                endPageIndex = pageCount;
                beginPageIndex = pageCount - 10 + 1;
            }
        }
    }
}

省略 get set
```

## 单元测试

### 创建索引

```java
@Test
public void createIndexTest() {
	ElasticsearchUtils.createIndex("ymq_index");
	ElasticsearchUtils.createIndex("ymq_indexsssss");
}
```

响应

```
Index [ymq_index] is not exist!
Index is not exits!
执行建立成功？true
Index [ymq_indexsssss] is not exist!
Index is not exits!
执行建立成功？true
```

### 删除索引

```java
@Test
public void deleteIndexTest() {
	ElasticsearchUtils.deleteIndex("ymq_indexsssss");
}
```

响应

```
Index [ymq_indexsssss] is exist!|
delete index ymq_indexsssss  successfully!
```


### 判断索引是否存在

```java
@Test
public void isIndexExistTest() {
	ElasticsearchUtils.isIndexExist("ymq_index");
}
```

响应

```
Index [ymq_index] is exist!
```


### 数据添加

```java
@Test
public void addDataTest() {

	for (int i = 0; i < 100; i++) {
		Map<String, Object> map = new HashMap<String, Object>();

		map.put("name", "鹏磊" + i);
		map.put("age", i);
		map.put("interests", new String[]{"阅读", "学习"});
		map.put("about", "世界上没有优秀的理念，只有脚踏实地的结果");
		map.put("processTime", new Date());

		ElasticsearchUtils.addData(JSONObject.parseObject(JSONObject.toJSONString(map)), "ymq_index", "about_test", "id=" + i);
	}
}
```

响应

```
addData response status:201,id:id=0
addData response status:201,id:id=1
addData response status:201,id:id=2
addData response status:201,id:id=3
addData response status:201,id:id=4
addData response status:201,id:id=5
addData response status:201,id:id=6
。。。。。。。
```

### 通过ID删除数据

```java
@Test
public void deleteDataByIdTest() {

	for (int i = 0; i < 10; i++) {
		ElasticsearchUtils.deleteDataById("ymq_index", "about_test", "id=" + i);
	}
}
```

响应

```
deleteDataById response status:200,id:id=0
deleteDataById response status:200,id:id=1
deleteDataById response status:200,id:id=2
deleteDataById response status:200,id:id=3
deleteDataById response status:200,id:id=4
deleteDataById response status:200,id:id=5
deleteDataById response status:200,id:id=6
deleteDataById response status:200,id:id=7
deleteDataById response status:200,id:id=8
deleteDataById response status:200,id:id=9
```



### 通过ID更新数据

```java
/**
 * 通过ID 更新数据
 * <p>
 * jsonObject 要增加的数据
 * index      索引，类似数据库
 * type       类型，类似表
 * id         数据ID
 */
@Test
public void updateDataByIdTest() {
	Map<String, Object> map = new HashMap<String, Object>();

	map.put("name", "鹏磊");
	map.put("age", 11);
	map.put("interests", new String[]{"阅读", "学习"});
	map.put("about", "这条数据被修改");
	map.put("processTime", new Date());

	ElasticsearchUtils.updateDataById(JSONObject.parseObject(JSONObject.toJSONString(map)), "ymq_index", "about_test", "id=11");
}
```


### 通过ID获取数据

```java
/**
 * 通过ID获取数据
 * <p>
 * index  索引，类似数据库
 * type   类型，类似表
 * id     数据ID
 * fields 需要显示的字段，逗号分隔（缺省为全部字段）
 */
@Test
public void searchDataByIdTest() {
	Map<String, Object> map = ElasticsearchUtils.searchDataById("ymq_index", "about_test", "id=11", null);
	System.out.println(JSONObject.toJSONString(map));
}
```

响应

```
{"name":"鹏磊","about":"这条数据被修改","interests":["阅读","学习"],"age":11,"processTime":1509966025972}
```


### 使用分词查询

```java
/**
 * 使用分词查询
 * <p>
 * index          索引名称
 * type           类型名称,可传入多个type逗号分隔
 * startTime      开始时间
 * endTime        结束时间
 * size           文档大小限制
 * fields         需要显示的字段，逗号分隔（缺省为全部字段）
 * sortField      排序字段
 * matchPhrase    true 使用，短语精准匹配
 * highlightField 高亮字段
 * matchStr       过滤条件（xxx=111,aaa=222）
 */
@Test
public void searchListData() {

	List<Map<String, Object>> list = ElasticsearchUtils.searchListData("ymq_index", "about_test", 1509959382607l, 1509959383865l, 0, "", "", false, "", "name=鹏磊");

	for (Map<String, Object> item : list) {

		System.out.println(JSONObject.toJSONString(item));
	}
}
```

响应

```
{
  "query" : {
    "bool" : {
      "must" : [
        {
          "match" : {
            "name" : {
              "query" : "鹏磊",
              "operator" : "OR",
              "prefix_length" : 0,
              "max_expansions" : 50,
              "fuzzy_transpositions" : true,
              "lenient" : false,
              "zero_terms_query" : "NONE",
              "boost" : 1.0
            }
          }
        }
      ],
      "disable_coord" : false,
      "adjust_pure_negative" : true,
      "boost" : 1.0
    }
  },
  "_source" : {
    "includes" : [ ],
    "excludes" : [ ]
  }
}
- [20171106 19:02:23.923] | [INFO] | [DESKTOP-VG43S0C] | [main] | [i.y.e.e.utils.ElasticsearchUtils] | --> 共查询到[90]条数据,处理数据条数[10]|
{"name":"鹏磊15","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=15","interests":["阅读","学习"],"age":15,"processTime":1509965846816}
{"name":"鹏磊18","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=18","interests":["阅读","学习"],"age":18,"processTime":1509965846849}
{"name":"鹏磊25","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=25","interests":["阅读","学习"],"age":25,"processTime":1509965846942}
{"name":"鹏磊47","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=47","interests":["阅读","学习"],"age":47,"processTime":1509965847143}
{"name":"鹏磊48","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=48","interests":["阅读","学习"],"age":48,"processTime":1509965847156}
{"name":"鹏磊55","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=55","interests":["阅读","学习"],"age":55,"processTime":1509965847212}
{"name":"鹏磊68","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=68","interests":["阅读","学习"],"age":68,"processTime":1509965847322}
{"name":"鹏磊73","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=73","interests":["阅读","学习"],"age":73,"processTime":1509965847375}
{"name":"鹏磊88","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=88","interests":["阅读","学习"],"age":88,"processTime":1509965847826}
{"name":"鹏磊89","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=89","interests":["阅读","学习"],"age":89,"processTime":1509965847872}
```




### 使用分词查询,并分页

```java
/**
 * 使用分词查询,并分页
 * <p>
 * index          索引名称
 * type           类型名称,可传入多个type逗号分隔
 * currentPage    当前页
 * pageSize       每页显示条数
 * startTime      开始时间
 * endTime        结束时间
 * fields         需要显示的字段，逗号分隔（缺省为全部字段）
 * sortField      排序字段
 * matchPhrase    true 使用，短语精准匹配
 * highlightField 高亮字段
 * matchStr       过滤条件（xxx=111,aaa=222）
 */
@Test
public void searchDataPage() {

	EsPage esPage = ElasticsearchUtils.searchDataPage("ymq_index", "about_test", 10, 5, 1509943495299l, 1509943497954l, "", "processTime", false, "about", "about=鹏磊");

	for (Map<String, Object> item : esPage.getRecordList()) {

		System.out.println(JSONObject.toJSONString(item));
	}

}
```

响应

```
- [20171106 19:10:15.738] | [DEBUG] | [DESKTOP-VG43S0C] | [main] | [i.y.e.e.utils.ElasticsearchUtils] | --> 共查询到[90]条数据,处理数据条数[5]|
遍历 高亮结果集，覆盖 正常结果集{name=鹏磊90, about=世界上没有优秀的理念，只有脚踏实地的结果, id=id=90, interests=[阅读, 学习], age=90, processTime=1509965847911}
遍历 高亮结果集，覆盖 正常结果集{name=鹏磊89, about=世界上没有优秀的理念，只有脚踏实地的结果, id=id=89, interests=[阅读, 学习], age=89, processTime=1509965847872}
遍历 高亮结果集，覆盖 正常结果集{name=鹏磊88, about=世界上没有优秀的理念，只有脚踏实地的结果, id=id=88, interests=[阅读, 学习], age=88, processTime=1509965847826}
遍历 高亮结果集，覆盖 正常结果集{name=鹏磊87, about=世界上没有优秀的理念，只有脚踏实地的结果, id=id=87, interests=[阅读, 学习], age=87, processTime=1509965847804}
遍历 高亮结果集，覆盖 正常结果集{name=鹏磊86, about=世界上没有优秀的理念，只有脚踏实地的结果, id=id=86, interests=[阅读, 学习], age=86, processTime=1509965847761}
{"name":"<em>鹏</em><em>磊</em>90","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=90","interests":["阅读","学习"],"age":90,"processTime":1509965847911}
{"name":"<em>鹏</em><em>磊</em>90<em>鹏</em><em>磊</em>89","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=89","interests":["阅读","学习"],"age":89,"processTime":1509965847872}
{"name":"<em>鹏</em><em>磊</em>90<em>鹏</em><em>磊</em>89<em>鹏</em><em>磊</em>88","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=88","interests":["阅读","学习"],"age":88,"processTime":1509965847826}
{"name":"<em>鹏</em><em>磊</em>90<em>鹏</em><em>磊</em>89<em>鹏</em><em>磊</em>88<em>鹏</em><em>磊</em>87","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=87","interests":["阅读","学习"],"age":87,"processTime":1509965847804}
{"name":"<em>鹏</em><em>磊</em>90<em>鹏</em><em>磊</em>89<em>鹏</em><em>磊</em>88<em>鹏</em><em>磊</em>87<em>鹏</em><em>磊</em>86","about":"世界上没有优秀的理念，只有脚踏实地的结果","id":"id=86","interests":["阅读","学习"],"age":86,"processTime":1509965847761}
```


代码我已放到 Github ，导入`spring-boot-elasticsearch-demo` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-elasticsearch-demo](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-elasticsearch-demo)


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")
