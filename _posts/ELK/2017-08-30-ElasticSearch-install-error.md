---
layout: post
title: ElasticSearch 安装报错整理
categories: ElasticSearch
description: ElasticSearch 安装报错整理
keywords: ElasticSearch
---

# ElasticSearch 安装报错整理

## 单机安装报错

### 初次启动服务

```sh
/opt/elasticsearch-5.5.2/bin/elasticsearch
```

当使用**root账户调用启动命令**出现错误信息,错误提示信息如下:

```sh
[2017-08-30T13:32:17,003][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [ELK-node1] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: java.lang.RuntimeException: can not run elasticsearch as root
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:127) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Elasticsearch.execute(Elasticsearch.java:114) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.cli.EnvironmentAwareCommand.execute(EnvironmentAwareCommand.java:67) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.cli.Command.mainWithoutErrorHandling(Command.java:122) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.cli.Command.main(Command.java:88) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:91) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Elasticsearch.main(Elasticsearch.java:84) ~[elasticsearch-5.5.2.jar:5.5.2]
Caused by: java.lang.RuntimeException: can not run elasticsearch as root
	at org.elasticsearch.bootstrap.Bootstrap.initializeNatives(Bootstrap.java:106) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Bootstrap.setup(Bootstrap.java:194) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Bootstrap.init(Bootstrap.java:351) ~[elasticsearch-5.5.2.jar:5.5.2]
	at org.elasticsearch.bootstrap.Elasticsearch.init(Elasticsearch.java:123) ~[elasticsearch-5.5.2.jar:5.5.2]
	... 6 more

```

### 创建新用户

由于Elasticsearch可以接收用户输入的脚本并且执行,为了系统安全考虑,**不允许root账号启动**,所以建议给`Elasticsearch`单独创建一个用户来运行`Elasticsearch`。

命令格式 `useradd  ymq(用户名) -g ymq(所属组名) -p ymq(密码)`

```sh 
groupadd ymq
useradd  ymq -g ymq -p ymq
```

### 授权访问组权限

命令格式: `chown -R ymq`(所属用户)  `:`  `ymq`(所属用户组名) `/opt/elasticsearch-5.5.2` (要更改的文件路径)

```sh
chown -R ymq:ymq /opt/elasticsearch-5.5.2
```

### 授权 root 权限

命令格式:  ymq 用户 root 权限 ` NOPASSWD `意思是 不用输密码

```sh
chmod 777 /etc/sudoers
vi /etc/sudoers
```
```sh
root    ALL=(ALL)       ALL

#添加ymq 用户 root 权限
ymq     ALL=(ALL)       NOPASSWD:ALL 
```

```sh
su ymq
/opt/elasticsearch-5.5.2/bin/elasticsearch
```

### 如果报如下错误

```sh
[2017-08-30T13:41:13,631][INFO ][o.e.n.Node               ] [ELK-node1] starting ...
[2017-08-30T13:41:14,093][INFO ][o.e.t.TransportService   ] [ELK-node1] publish_address {192.168.252.121:9300}, bound_addresses {[::]:9300}
[2017-08-30T13:41:14,121][INFO ][o.e.b.BootstrapChecks    ] [ELK-node1] bound or publishing to a non-loopback or non-link-local address, enforcing bootstrap checks
[2017-08-30T13:41:14,127][ERROR][o.e.b.Bootstrap          ] [ELK-node1] node validation exception
[2] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[2017-08-30T13:41:14,142][INFO ][o.e.n.Node               ] [ELK-node1] stopping ...
[2017-08-30T13:41:14,186][INFO ][o.e.n.Node               ] [ELK-node1] stopped
[2017-08-30T13:41:14,186][INFO ][o.e.n.Node               ] [ELK-node1] closing ...
[2017-08-30T13:41:14,204][INFO ][o.e.n.Node               ] [ELK-node1] closed
```

以下错误都切换到 `root` 用户进项修改

```sh
su root
```

```sh
错误[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536
```

编辑 `limits.conf` 在第一行加上如下内容

```sh
cat /etc/security/limits.conf 

* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
```

```sh
错误[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

编辑 `limits.conf` 在第一行加上如下内容

```sh
cat /etc/sysctl.conf 

vm.max_map_count = 655360
```

执行 `sysctl -p`

```sh
sysctl -p
```

错误 `IllegalStateException`
```sh
Caused by: java.lang.IllegalStateException: failed to obtain node locks, tried [[/opt/elasticsearch-5.5.2/data/ymq]] with lock id [0]; maybe thes
```

删除 安装目录下`/data`
```sh
cd /opt/elasticsearch-5.5.2/data
rm -rf nodes
```


错误 `ElasticsearchUncaughtExceptionHandler]`
上次启动失败，占用了端口
```sh
[2017-08-30T22:02:14,463][WARN ][o.e.b.ElasticsearchUncaughtExceptionHandler] [ELK-node2] uncaught exception in thread [main]
org.elasticsearch.bootstrap.StartupException: BindHttpException[Failed to bind to [9200]]; nested: BindException[Address already in use];
```

```sh
jps 

2824 Elasticsearch
3165 Jps
```

```
kill -9 2824
```



## 再次启动服务

切到 `ymq` 用户尝试启动服务 加 	`-d` 后台启动

```sh
su ymq
/opt/elasticsearch-5.5.2/bin/elasticsearch  -d
```

### jvm 内存内存修改

```sh
vi /opt/elasticsearch-5.5.2/config/jvm.options

-Xms2g  --》修改为512m
-Xmx2g  --》修改为512m
```



### 查看日志  

`ymq` 及时集群名称

```sh
less /opt/elasticsearch-5.5.2/logs/ymq.log
```


### 查看端口

```sh
netstat -nltp
```

```sh
(Not all processes could be identified, non-owned process info
 will not be shown, you would have to be root to see it all.)
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -                   
tcp        0      0 127.0.0.1:25            0.0.0.0:*               LISTEN      -                   
tcp6       0      0 :::9200                 :::*                    LISTEN      2944/java           
tcp6       0      0 :::9300                 :::*                    LISTEN      2944/java           
tcp6       0      0 :::22                   :::*                    LISTEN      -                   
tcp6       0      0 ::1:25                  :::*                    LISTEN      - 
```

### 测试访问

```sh
curl -X GET http://localhost:9200/
```

```sh
{
  "name" : "ELK-node1",
  "cluster_name" : "ymq",
  "cluster_uuid" : "jxWzvSFNTCWtToD6wrVIpA",
  "version" : {
    "number" : "5.5.2",
    "build_hash" : "b2f0c09",
    "build_date" : "2017-08-14T12:33:14.154Z",
    "build_snapshot" : false,
    "lucene_version" : "6.6.0"
  },
  "tagline" : "You Know, for Search"
```

浏览器如果访问不成功，关闭防火墙，开放指定端9200，9300端口，为了方便，生产不要禁用防火墙

```sh
systemctl stop firewalld.service
```


[更多异常：请参考](http://www.cnblogs.com/sloveling/p/elasticsearch.html)



## Elasticsearch 复制集群配置

**不推荐按照以下步骤**，可以按照上面介绍的单机配置依次安装

### 在`node1` ROOT 用户下操作

```sh
su root
```

把本机配置的文件复制到 **node2**,**node3**集群 

```sh
for a in {2..3} ; do scp -r /opt/elasticsearch-5.5.2/ node$a:/opt/elasticsearch-5.5.2/ ; done
```

`ssh 登录` **node2**,**node3**集群，创建新用户，授权访问组权限，并且授权访问`/etc/sudoers` 给root权限 

```sh
for a in {2..3} ; do ssh node$a "source /etc/profile; groupadd ymq; useradd ymq -g ymq -p ymq; chown -R ymq:ymq /opt/elasticsearch-5.5.2; chmod 777 /etc/sudoers;" ; done
```

```sh
for a in {2..3} ; do scp -r /etc/sudoers node$a:/etc/sudoers ; done
```

复制 `limits.conf` 到 **node2**,**node3**集群 

```sh
for a in {2..3} ; do scp -r /etc/security/limits.conf node$a:/etc/security/limits.conf ; done
```

复制 `sysctl.conf` 到 **node2**,**node3**集群
 
```sh
for a in {2..3} ; do scp -r /etc/sysctl.conf  node$a:/etc/sysctl.conf ; done
```


复制 `jvm.options` 到 **node2**,**node3**集群
 
```sh
for a in {2..3} ; do scp -r /opt/elasticsearch-5.5.2/config/jvm.options  node$a:/opt/elasticsearch-5.5.2/config/jvm.options ; done
```

执行 `sysctl -p`操作


```sh
for a in {2..3} ; do ssh node$a "source /etc/profile; sysctl -p ; " ; done
```

```sh
vi /opt/elasticsearch-5.5.2/config/elasticsearch.yml
```

node.name: 换个名字，可以`ELK-node2`,`ELK-node3`

```
node.name: ELK-node1
```

### 后台启动

后台启动**node1**,**node2**,**node3** 服务

```sh
su ymq
/opt/elasticsearch-5.5.2/bin/elasticsearch -d
```

查看**node1**,**node2**,**node3**端口使用情况

```sh
 netstat -nltp
```

