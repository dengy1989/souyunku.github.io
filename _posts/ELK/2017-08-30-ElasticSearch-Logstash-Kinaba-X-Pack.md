---
layout: post
title: ELK 集群 Kibana 使用 X-Pack  权限控制，监控集群状态，实时的生成，警报，监视,cpu，内存，磁盘空间，等等一系列，报告和的可视化图形
categories: ElasticSearch Logstash Kinaba X-Pack
description: ELK集群搭建 ElasticSearch Logstash Kinaba X-Pack
keywords: ElasticSearch Logstash Kinaba X-Pack
---

# 简述

**ELK**实际上是**三个工具**的集合，**ElasticSearch** + **Logstash** + **Kibana**

这三个工具组合形成了一套实用、易用的监控架构，很多公司利用它来搭建可视化的海量日志分析平台。  

**X-Pack**

[X-Pack Elastic Stack](https://www.elastic.co/guide/en/x-pack/current/index.html)
	
`X-Pack`是一个`Elastic Stack`的扩展，将安全，警报，监视，报告和图形功能包含在一个易于安装的软件包中

## 搭建集群

阅读本文之前，你需要有ELK,环境，请移步，我的另一篇文章

**[ELK 集群 + Redis 集群 + Nginx ,分布式的实时日志（数据）搜集和分析的监控系统搭建，简单上手使用](https://segmentfault.com/a/1190000010975383)**

# 1.X-Pack 安装

https://www.elastic.co/guide/en/x-pack/current/index.html

[Installing X-Pack](https://www.elastic.co/guide/en/x-pack/5.5/xpack-introduction.html)

在`Elasticsearch`，`Kibana`和`Logstash`上安装`X-Pack`

X-Pack是一个Elastic Stack的扩展，将安全，警报，监视，报告和图形功能包含在一个易于安装的软件包中

## 下载安装

`X-Pack` 安装方式有两种 

### `logstash` 安装 `x-pack`

[Installing X-Pack in Logstash](https://www.elastic.co/guide/en/logstash/5.5/installing-xpack-log.html)

#### 安装方式 一

默认一条命令就可以自动下载安装完成

```sh
bin/logstash-plugin install x-pack
```

#### 安装方式 二

如果您下载慢，或者机器不能联网，以下**提供两种离线安装方式**

上传在上传了所有本文章所有的安装包在 [ 百度云盘-点击下载链接：密码：3l8v](http://pan.baidu.com/s/1bpm2Dwb )---[官方下载地址](https://artifacts.elastic.co/downloads/packs/x-pack/x-pack-5.5.2.zip)  

**请勿将文件放在`Elasticsearch plugins`目录中**

**1.指定目录安装**

```sh
bin/logstash-plugin install /opt/file/x-pack-5.5.2.zip
```

**2.或者放在服务器，`/tmp` 目录下**  这样就不用指定目录了

```sh
bin/logstash-plugin install  x-pack
```

### `elasticsearch` 安装 `x-pack`

[Installing X-Pack in Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/5.5/installing-xpack-es.html)


跟上面`logstash` 安装 `x-pack`类似，只是把安装的脚本前缀修改下

格式 `<elasticsearch>`-plugin install x-pack

```sh
bin/elasticsearch-plugin install x-pack
```

### `kibana` 安装 `x-pack`

[Installing X-Pack in Kibana](https://www.elastic.co/guide/en/kibana/5.5/installing-xpack-kb.html)

跟上面`logstash` 安装 `x-pack`类似，只是把安装的脚本前缀修改下


格式 `<elasticsearch>`-plugin install x-pack

你可能会等待不知道多久才成功：（所以建议调大虚拟机的内存和处理器的核数）

```sh
bin/kibana-plugin install x-pack
```

如果安装失败

`Plugin x-pack already exists, please remove before installing a new version

```sh
bin/kibana-plugin remove x-pack
```

安装成功的样子

```sh
Found previous install attempt. Deleting...
Attempting to transfer from x-pack
Attempting to transfer from https://artifacts.elastic.co/downloads/kibana-plugins/x-pack/x-pack-5.5.2.zip
Transferring 119363535 bytes....................
Transfer complete
Retrieving metadata from plugin archive
Extracting plugin archive
Extraction complete
Optimizing and caching browser bundles...
Plugin installation complete
```

curl -XPUT -u elastic 'localhost:9200/_xpack/security/user/elastic/_password' -d '{"password" : "elastic"}'

## 启用和禁用

启用和禁用`X-Pack`功能

默认情况下，所有X-Pack功能都被启用。您可以启用或禁用特定的X-Pack功能`elasticsearch.yml`，`kibana.yml`以及`logstash.yml` 配置文件。

设置 |	描述 
----|------
`xpack.graph.enabled` | 设置为`false`禁用`X-Pack`图形功能。 
`xpack.ml.enabled` | 设置为`false`禁用`X-Pack`机器学习功能。
`xpack.monitoring.enabled` | 设置为`false`禁用`X-Pack`监视功能。
`xpack.reporting.enabled` | 设置为`false`禁用`X-Pack`报告功能。
`xpack.security.enabled` | 设置为`false`禁用`X-Pack`安全功能。
`xpack.watcher.enabled` | 设置`false`为禁用观察器。

有关每个配置文件中存在哪些设置的详细信息，请参阅 [X-Pack设置](https://www.elastic.co/guide/en/x-pack/5.5/xpack-settings.html)。


# 2.使用 X-Pack

## 初始用户名密码

用户名：changeme
密码为：changeme

## 修改密码

修改kibana密码：修改之前需要在`kibana.yml`中配置`elasticsearch`的用户名和密码后才能需改密码，否则会报错。

```sh
# If your Elasticsearch is protected with basic authentication, these settings provide
# the username and password that the Kibana server uses to perform maintenance on the Kibana
# index at startup. Your Kibana users still need to authenticate with Elasticsearch, which
# is proxied through the Kibana server.
elasticsearch.username: "elastic"
elasticsearch.password: "your password"
```

```sh
curl -XPUT -u elastic 'localhost:9200/_xpack/security/user/elastic/_password' -d '{
  "password" : "elastic"
}'
```


## 一些命令

**查询所有用户**

```sh
 curl -XGET -u elastic 'localhost:9200/_xpack/security/user?pretty'
```
**查询所有Roles**

```sh
curl -XGET -u elastic 'localhost:9200/_xpack/security/role'
```

(安全API)[https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api.html]

##  ElasticSearchHead

当你再次打开 浏览器 `ElasticSearchHead` 插件时候，会提示你输入密码
 
 ![][1] 
 
**登录成功的**
 
 ![][2] 

## Kibana
  
  当你再次打开 浏览器 `Kibana` 页面，会提示你输入密码
 
 ![][3] 
 
 **Kibana，登录成功的**
 
 ![][4] 

 **Kibana，登录成功的，发现菜单功能多了，这就是我们安装的`X-Pack` 插件所提供的**
 ![][5] 
 
JVM堆，索引内存（KB），CPU利用率（％），系统负载，延迟（ms）等等

 ![][6] 

 ![][7] 
 
 ![][8] 
 
 ![][9]
 
  ![][10]
  
 - **作者：Peng Lei** 
 - **出处：[Segment Fault PengLei `Blog  专栏](https://segmentfault.com/a/1190000010867488)**      
 - **版权归作者所有，转载请注明出处** 

[1]: /images/2017/ELK/Elasticsearch-head-pwd.png

[2]: /images/2017/ELK/Elasticsearch-head-pwd-succ.png

[3]: /images/2017/ELK/kibana-login.png

[4]: /images/2017/ELK/kibana-login-succ.png

[5]: /images/2017/ELK/kibana-tree.png

[6]: /images/2017/ELK/monitoring.png

[7]: /images/2017/ELK/monitoring2.png

[8]: /images/2017/ELK/monitoring3.png

[9]: /images/2017/ELK/elasticsearch-nodes.png

[10]: /images/2017/ELK/elasticsearch-indices.png



