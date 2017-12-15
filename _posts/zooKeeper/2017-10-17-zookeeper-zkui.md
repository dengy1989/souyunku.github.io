---
layout: post
title: ZooKeeper 可视化监控 zkui
categories: ZooKeeper
description: ZooKeeper 可视化监控 zkui
keywords: ZooKeeper
---

# 概述
一、简介zkui它提供了一个管理界面，可以针对zookeepr的节点值进行CRUD操作，同时也提供了安全认证。 
 
 
二、下载安装 
 
1、下载地址  [https://github.com/DeemOpen/zkui](https://github.com/DeemOpen/zkui)

2、mvn clean install，执行前需要安装 java 环境，maven环境，执行成功后会生成一个jar文件。 

3、将config.cfg复制到上一步生成的jar文件所在目录，然后修改配置文件中的zookeeper地址。 
 
```
zkServer=localhost:2181
```
4、执行运行命令
 
```
java -jar zkui-2.0-SNAPSHOT-jar-with-dependencies.jar
```

5、测试，[http://localhost:9090](http://localhost:9090)，如能看到如下页面则代表zookeeper安装运行正常。 

6.用默认的账号和密码登录 

```
username:admin
password:manager
```
 
![][1] 

[1]: http://www.ymq.io/images/2017/zkui/1.png  

 
# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

