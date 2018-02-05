---
layout: post
title: Spring Boot 中使用 MyBatis 整合 Druid 多数据源
categories: [MyBatis,SpringBoot]
description: Spring Boot 中使用 MyBatis 整合 Druid 多数据源
keywords: MyBatis
---

本文将讲述 spring boot + mybatis + druid 多数据源配置方案。

# 环境

[CentOs7.3 安装 MySQL 5.7.19 二进制版本](https://segmentfault.com/a/1190000010864818)

# Github 代码

代码我已放到 Github ，导入`spring-boot-mybatis` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-mybatis](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-mybatis)

![ 项目结构 ][1] 

## 添加依赖

在项目中添加 `mybatis，druid ` 依赖

[点击预览 pom.xml](https://github.com/souyunku/ymq-example/blob/master/ymq-mybatis-spring-boot/pom.xml)
```xml
<dependency>
	<groupId>org.mybatis.spring.boot</groupId>
	<artifactId>mybatis-spring-boot-starter</artifactId>
</dependency>
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid</artifactId>
</dependency>
省略 更多
```

## 基础数据源

```java
@Configuration
@EnableConfigurationProperties(DruidDbProperties.class)
@Import({DruidMonitConfig.class})
public abstract class AbstractDruidDBConfig {

    private Logger logger = LoggerFactory.getLogger(AbstractDruidDBConfig.class);

    @Resource
    private DruidDbProperties druidDbProperties;

    public DruidDataSource createDataSource(String url, String username, String password) {
        if (StringUtils.isEmpty(url)) {
            System.out.println(
                    "Your database connection pool configuration is incorrect!" + " Please check your Spring profile");
            throw new ApplicationContextException("Database connection pool is not configured correctly");
        }

        DruidDataSource datasource = new DruidDataSource();

        datasource.setUrl(url);
        datasource.setUsername(username);
        datasource.setPassword(password);
        // datasource.setDriverClassName(
        // StringUtils.isEmpty(driverClassName) ?
        // druidDbProperties.getDriverClassName() : driverClassName);
        datasource.setInitialSize(druidDbProperties.getInitialSize());
        datasource.setMinIdle(druidDbProperties.getMinIdle());
        datasource.setMaxActive(druidDbProperties.getMaxActive());
        datasource.setMaxWait(druidDbProperties.getMaxWait());
        datasource.setTimeBetweenEvictionRunsMillis(druidDbProperties.getTimeBetweenEvictionRunsMillis());
        datasource.setMinEvictableIdleTimeMillis(druidDbProperties.getMinEvictableIdleTimeMillis());
        datasource.setValidationQuery(druidDbProperties.getValidationQuery());
        datasource.setTestWhileIdle(druidDbProperties.isTestWhileIdle());
        datasource.setTestOnBorrow(druidDbProperties.isTestOnBorrow());
        datasource.setTestOnReturn(druidDbProperties.isTestOnReturn());
        try {
            datasource.setFilters(druidDbProperties.getFilters());
        } catch (SQLException e) {
            logger.error("druid configuration initialization filter", e);
        }
        datasource.setConnectionProperties(druidDbProperties.getConnectionProperties());
        return datasource;

    }

    /**
     * 加载默认mybatis xml配置文件，并初始化分页插件
     *
     * @param dataSource
     * @return
     * @throws Exception
     */
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        return createSqlSessionFactory(dataSource, "classpath:mybatis/**/*.xml");
    }

    /**
     * 加载mybatis xml配置文件，并初始化分页插件
     *
     * @param dataSource      数据源
     * @param mapperLocations 自定义xml配置路径
     * @return
     * @throws Exception
     */
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource, String mapperLocations) throws Exception {
        return createSqlSessionFactory(dataSource, mapperLocations);
    }

    private SqlSessionFactory createSqlSessionFactory(DataSource dataSource, String mapperLocations) throws Exception {

        SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(dataSource);
        // mybatis分页
        PageHelper pageHelper = new PageHelper();
        Properties props = new Properties();
        props.setProperty("dialect", "mysql");
        props.setProperty("reasonable", "true");
        props.setProperty("supportMethodsArguments", "true");
        props.setProperty("returnPageInfo", "check");
        props.setProperty("params", "count=countSql");
        pageHelper.setProperties(props); // 添加插件
        sqlSessionFactoryBean.setPlugins(new Interceptor[]{pageHelper});
        PathMatchingResourcePatternResolver resolver = new PathMatchingResourcePatternResolver();
        sqlSessionFactoryBean.setMapperLocations(resolver.getResources(mapperLocations));
        return sqlSessionFactoryBean.getObject();

    }
}

```

## Druid 监控配置

```java
@EnableConfigurationProperties(DruidDbProperties.class)
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class DruidMonitConfig {

    @Resource
    private DruidDbProperties druidDbProperties;

    @Bean
    public ServletRegistrationBean druidServlet() {
        ServletRegistrationBean reg = new ServletRegistrationBean();
        reg.setServlet(new StatViewServlet());
        reg.addUrlMappings("/druid/*");

        if (!StringUtils.isEmpty(druidDbProperties.getAllow())) {
            reg.addInitParameter("allow", druidDbProperties.getAllow()); // 白名单
        }
        if (!StringUtils.isEmpty(druidDbProperties.getDeny())) {
            reg.addInitParameter("deny", druidDbProperties.getDeny()); // 黑名单
        }
        reg.addInitParameter("loginUsername", druidDbProperties.getUsername());
        reg.addInitParameter("loginPassword", druidDbProperties.getPassword());
        return reg;
    }

    @Bean
    public FilterRegistrationBean filterRegistrationBean() {
        FilterRegistrationBean filterRegistrationBean = new FilterRegistrationBean();
        filterRegistrationBean.setFilter(new WebStatFilter());
        filterRegistrationBean.addUrlPatterns("/*");
        filterRegistrationBean.addInitParameter("exclusions", "*.js,*.gif,*.jpg,*.png,*.css,*.ico,/druid/*");
        return filterRegistrationBean;
    }

    /**
     * 监听Spring 1.定义拦截器 2.定义切入点 3.定义通知类
     *
     * @return
     */
    @Bean
    public DruidStatInterceptor druidStatInterceptor() {
        return new DruidStatInterceptor();
    }

    @Bean
    public JdkRegexpMethodPointcut druidStatPointcut() {
        JdkRegexpMethodPointcut druidStatPointcut = new JdkRegexpMethodPointcut();
        String patterns = "io.ymq.mybatis*";
        druidStatPointcut.setPatterns(patterns);
        return druidStatPointcut;
    }

    @Bean
    public Advisor druidStatAdvisor() {
        return new DefaultPointcutAdvisor(druidStatPointcut(), druidStatInterceptor());
    }
}
```


## Druid 监控参数

```java

@ConfigurationProperties(prefix = "druid")
public class DruidDbProperties {

    private String driverClassName = "com.mysql.jdbc.Driver";

    /**
     * 初始化时建立物理连接的个数。初始化发生在显示调用init方法，或者第一次getConnection时
     */
    private int initialSize = 10;

    /**
     * 最小连接池数量
     */
    private int minIdle = 50;

    /**
     * 最大连接池数量
     */
    private int maxActive = 300;

    /**
     * 获取连接时最大等待时间，单位毫秒。配置了maxWait之后，缺省启用公平锁，并发效率会有所下降，如果需要可以通过配置useUnfairLock属性为true使用非公平锁。
     */
    private int maxWait = 60000;

    /**
     * 有两个含义： 1)
     * Destroy线程会检测连接的间隔时间，如果连接空闲时间大于等于minEvictableIdleTimeMillis则关闭物理连接。 2)
     * testWhileIdle的判断依据，详细看testWhileIdle属性的说明
     */
    private int timeBetweenEvictionRunsMillis = 60000;

    /**
     * 连接保持空闲而不被驱逐的最长时间
     */
    private int minEvictableIdleTimeMillis = 3600000;

    /**
     * 用来检测连接是否有效的sql，要求是一个查询语句，常用select
     * 'x'。如果validationQuery为null，testOnBorrow、testOnReturn、testWhileIdle都不会其作用。
     */
    private String validationQuery = "SELECT USER()";

    /**
     * 建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效。
     */
    private boolean testWhileIdle = true;

    /**
     * 申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
     */
    private boolean testOnBorrow = false;

    /**
     * 归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
     */
    private boolean testOnReturn = false;

    /**
     * 属性类型是字符串，通过别名的方式配置扩展插件，常用的插件有： 监控统计用的filter:stat 日志用的filter:log4j
     * 防御sql注入的filter:wall
     */
    private String filters = "mergeStat,config,wall";

    private String connectionProperties;

    /**
     * 白名单
     */
    private String allow;

    /**
     * 黑名单
     */
    private String deny;

    private String username = "admin";

    private String password = "admin";
	
	省略 get set
}
```



## 配置数据源 one

```java
@Configuration
@EnableTransactionManagement
public class DBOneConfiguration extends AbstractDruidDBConfig {

    @Value("${ymq.one.datasource.url}")
    private String url;

    @Value("${ymq.one.datasource.username}")
    private String username;

    @Value("${ymq.one.datasource.password}")
    private String password;

    // 注册 datasourceOne
    @Bean(name = "datasourceOne", initMethod = "init", destroyMethod = "close")
    public DruidDataSource dataSource() {
        return super.createDataSource(url, username, password);
    }

    @Bean(name = "sqlSessionFactorYmqOne")
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        return super.sqlSessionFactory(dataSource());
    }

    @Bean
    public PlatformTransactionManager transactionManager() throws SQLException {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

## 配置数据源 two

```java
@Configuration
@EnableTransactionManagement
public class DBOneConfiguration extends AbstractDruidDBConfig {

    @Value("${ymq.one.datasource.url}")
    private String url;

    @Value("${ymq.one.datasource.username}")
    private String username;

    @Value("${ymq.one.datasource.password}")
    private String password;

    // 注册 datasourceOne
    @Bean(name = "datasourceOne", initMethod = "init", destroyMethod = "close")
    public DruidDataSource dataSource() {
        return super.createDataSource(url, username, password);
    }

    @Bean(name = "sqlSessionFactorYmqOne")
    public SqlSessionFactory sqlSessionFactory() throws Exception {
        return super.sqlSessionFactory(dataSource());
    }

    @Bean
    public PlatformTransactionManager transactionManager() throws SQLException {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

## BaseDao one 

```java
@Repository
public class YmqOneBaseDao extends BaseDao {

    @Resource
    public void setSqlSessionFactorYmqOne(SqlSessionFactory sqlSessionFactory) {
        super.setSqlSessionFactory(sqlSessionFactory);
    }
}
```

## BaseDao two 

```java
@Repository
public class YmqTwoBaseDao extends BaseDao {

    @Resource
    public void setSqlSessionFactorYmqTwo(SqlSessionFactory sqlSessionFactory) {
        super.setSqlSessionFactory(sqlSessionFactory);
    }
}
```

## 测试 Controller

```
@RestController
public class IndexController {

    private static final Logger LOG = LoggerFactory.getLogger(IndexController.class);

    @Autowired
    private YmqOneBaseDao ymqOneBaseDao;

    @Autowired
    private YmqTwoBaseDao ymqTwoBaseDao;

    @RequestMapping("/")
    public String index() throws Exception {

        List<TestOnePo> testOnePoList = null;

        testOnePoList = ymqOneBaseDao.selectList(new TestOnePo());
        for (TestOnePo item : testOnePoList) {
            LOG.info("数据源 ymqOneBaseDao ：查询结果:{}", JSONObject.toJSONString(item));
        }

        List<TestTwoPo> testTwoPoList = null;

        testTwoPoList = ymqTwoBaseDao.selectList(new TestTwoPo());

        for (TestTwoPo item : testTwoPoList) {
            LOG.info("数据源 ymqTwoBaseDao：查询结果:{}", JSONObject.toJSONString(item));
        }

        String onePoList = JSONObject.toJSONString(testOnePoList);
        String twoPoList = JSONObject.toJSONString(testTwoPoList);

        return "数据源 ymqOneBaseDao ：查询结果:" + onePoList + "<br/> 数据源 ymqTwoBaseDao ：查询结果:" + twoPoList;
    }
}
```

## 参数配置

**`application.properties`**

```
#############SERVER CONFIG############
spring.application.name=ymq-mybatis-spring-boot

#数据源 one
ymq.one.datasource.url=jdbc:mysql://10.4.82.6:3306/ymq_one?useUnicode=true&characterEncoding=UTF-8
ymq.one.datasource.username=root
ymq.one.datasource.password=123456

#数据源 two
ymq.two.datasource.url=jdbc:mysql://10.4.82.6:3306/ymq_two?useUnicode=true&characterEncoding=UTF-8
ymq.two.datasource.username=root
ymq.two.datasource.password=123456

server.port=80
server.tomcat.max-threads=1000
server.tomcat.max-connections=2000
```


## 启动服务

```java
@SpringBootApplication
@ComponentScan(value = {"io.ymq.mybatis"})
public class Startup {

    public static void main(String[] args) {
        SpringApplication.run(Startup.class, args);
    }
}
```

在页面上输入 [http://localhost/](http://localhost/) 可以看到 Controller  执行情况：

```
数据源 ymqOneBaseDao ：查询结果:[{"id":1,"name":"测试","remark":"这是测试 ymq_one 数据库"}]
数据源 ymqTwoBaseDao ：查询结果:[{"id":1,"name":"测试","remark":"这是测试 ymq_two 数据库"}]
```

在页面上输入 [http://localhost/druid/](http://localhost/druid/)  可以看到监控到的sql语句执行情况：

![ druid 监控 ][2] 

代码我已放到 Github ，导入`spring-boot-mybatis` 项目 

github [https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-mybatis](https://github.com/souyunku/spring-boot-examples/tree/master/spring-boot-mybatis)

[1]: http://www.ymq.io/images/2017/mybatis/1.png
[2]: http://www.ymq.io/images/2017/mybatis/2.png


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")


