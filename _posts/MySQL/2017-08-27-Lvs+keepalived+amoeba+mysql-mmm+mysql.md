---
layout: post
title: CentOs7.3 安装 MySQL 5.7.19 负载均衡+读写分离+双主多从
categories: MySQL
description: CentOs7.3 安装 MySQL 5.7.19 负载均衡+读写分离+双主多从
keywords: MySQL
---

# keepalived + amoeba + mysql-mmm + mysql实现mysql 负载均衡+读写分离+双主多从


## 准备工作

关闭防火墙

```sh
$ systemctl stop firewalld.service
```

环境

| 		主机名	| 		物理IP 	| 虚拟IP 	| 		集群角色 					|server_id 	|
|---------------|---------------|-----------|-----------------------------------|-----------:  
|Moniter1		|192.168.252.121|			|keepalived / amoeba				|			|
|Moniter2		|192.168.252.122|			|keepalived / amoeba / mysql-mmm	|			|
|Master1		|192.168.252.123|			|Master 读 / 写      / mysql-mmm	|	1		|
|Master2		|192.168.252.124|			|Master 读 / 写      / mysql-mmm	|	2		|
|Slave1			|192.168.252.125|			|Slave 只读 / mysql-mmm				|	3		|
|Slave2			|192.168.252.126|			|Slave 只读/ mysql-mmm				|	4		|


### 安装MySQL

在Master1，Master2，Slave1，Slave2 都安装MySQL，为了篇幅，就不贴了安装过程了

[《CentOs7.3 安装 MySQL 5.7.19 二进制版本》](https://segmentfault.com/a/1190000010864818)

[《集群主机名称修改》](https://segmentfault.com/a/1190000010723105)

[《使用通用二进制文件在Unix / Linux上安装MySQL》](https://dev.mysql.com/doc/refman/5.7/en/binary-installation.html)

[《MySQL社区版 下载地址》](https://dev.mysql.com/downloads/mysql/)

[《参考 MySQL官网 - 第16章主从复制》](https://dev.mysql.com/doc/refman/5.7/en/replication.html)

## MySQL主主复制

主主复制即在两台MySQL主机内都可以变更数据，而且另外一台主机也会做出相应的变更。聪明的你也许已经想到该怎么实现了。对，就是将两个主从复制有机合并起来就好了。只不过在配置的时候我们需要注意一些问题，例如，主键重复，server-id，`auto.cnf`配置 server-uuid  不能重复等等。


### 1.修改 my.cnf 

> 在 Master1，Master2 上都修改 /etc/my.cnf 

**停止 Master1，Master2 上的MySQL服务**

```sh
$ service mysql.server stop
```

在 Master1 操作

```sh
[root@master1 ~]# cat /etc/my.cnf
[mysqld]
server-id=1
log-bin = mysql-bin

replicate-do-db=replication_wwww.ymq.io
replicate-ignore-db = information_schema.%
```

在 Master2 操作

```sh
[root@master2 ~]# cat /etc/my.cnf
[mysqld]
server-id=1
log-bin = mysql-bin

replicate-do-db=replication_wwww.ymq.io
replicate-ignore-db = information_schema.%
```
一些系统变量参考  
[《 MySQL官网 - Replication Slave Options and Variables》](https://dev.mysql.com/doc/refman/5.7/en/replication-options-slave.html)  
[《 MySQL官网 - Replication and Binary Logging Options and Variables》](https://dev.mysql.com/doc/refman/5.7/en/replication-options.html)  
[《 MySQL官网 - Replication and Binary Logging Option and Variable Reference》](https://dev.mysql.com/doc/refman/5.7/en/replication-options-table.html)  

下面对应每行的解释

```
每台设置不同的 server-id
开启binlog  
		        
同步的数据库，多个写多行  
不同步的数据库，多个写多行  					
```

### 2.创建用户

每个从库使用MySQL用户名和密码连接到主库，因此主库上必须有用户帐户，从库可以连接。任何帐户都可以用于此操作，只要它已被授予 `REPLICATION SLAVE`权限。可以选择为每个从库创建不同的帐户，或者每个从库使用相同帐户连接到主库

虽然不必专门为复制创建帐户，但应注意，复制用到的用户名和密码会以纯文本格式存储在主信息存储库文件或表中 。因此，需要创建一个单独的帐户，该帐户只具有复制过程的权限，以尽可能减少对其他帐户的危害。

> 在 Master1，Master2 上执行 sql 操作

启动MySQL服务

```sh
$ service mysql.server start
```

登录MySQL

```sh
$ /usr/local/mysql/bin/mysql -uroot -p
```

创建用户，授予`REPLICATION SLAVE`权限

```sql
mysql> CREATE USER 'replication'@'192.168.252.%' IDENTIFIED BY 'mima';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'192.168.252.%';
```

### 3.配置主主复制通信

**查看 Master1，Master2 ， binlog 文件名称和 Position值位置 并且记下来**，由于我这都是新环境所以 Master1，Master2 上的 `binlog File 名称` ，`Position`，都是`mysql-bin.000001` 和` 625`
```sql
mysql> show master status;
+------------------+----------+-------------------------+----------------------+-------------------+
| File             | Position | Binlog_Do_DB            | Binlog_Ignore_DB     | Executed_Gtid_Set |
+------------------+----------+-------------------------+----------------------+-------------------+
| mysql-bin.000001 |      625 | replication_wwww.ymq.io | information_schema.% |                   |
+------------------+----------+-------------------------+----------------------+-------------------+
1 row in set (0.00 sec)
```

要设置从库与主库进行通信，进行复制，使用必要的连接信息配置从库在从库上执行以下语句  
**将选项值替换为与系统相关的实际值**

**参数格式，请勿执行**

```sh
mysql> CHANGE MASTER TO
    ->     MASTER_HOST='master_host_name',
    ->     MASTER_USER='replication_user_name',
    ->     MASTER_PASSWORD='replication_password',
    ->     MASTER_LOG_FILE='recorded_log_file_name',
    ->     MASTER_LOG_POS=recorded_log_position;
```

### 4.配置 Master2 

> 在 Master2 操作,配置与主库Master1通信 

```sh
mysql> CHANGE MASTER TO
    -> MASTER_HOST='192.168.252.123',
    -> MASTER_USER='replication',
    -> MASTER_PASSWORD='mima',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=625;
Query OK, 0 rows affected, 2 warnings (0.02 sec)
```

`MASTER_LOG_POS=0` 写成0 也是可以的

放在一行执行方便
```sh
CHANGE MASTER TO MASTER_HOST='192.168.252.123', MASTER_USER='replication', MASTER_PASSWORD='mima', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=625;
```

启动从服务器复制线程
```sh
mysql> START SLAVE;
Query OK, 0 rows affected (0.00 sec)
```

查看从服务器复制状态 
```sh
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.252.123
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 625
               Relay_Log_File: master2-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: mysql,ysnc_wwww.ymq.io
          Replicate_Ignore_DB: information_schema
 .......
```


### 5.配置 Master1 

> 在 Master1 操作,配置与主库Master2通信 

```sh
mysql> CHANGE MASTER TO
    -> MASTER_HOST='192.168.252.124',
    -> MASTER_USER='replication',
    -> MASTER_PASSWORD='mima',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=625;
Query OK, 0 rows affected, 2 warnings (0.02 sec)
```

放在一行执行方便
```sh
CHANGE MASTER TO MASTER_HOST='192.168.252.124', MASTER_USER='replication', MASTER_PASSWORD='mima', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=625;
```

启动从服务器复制线程
```sh
mysql> START SLAVE;
Query OK, 0 rows affected (0.00 sec)
```

查看从服务器复制状态 
```sh
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.252.124
                  Master_User: replication
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 625
               Relay_Log_File: master1-relay-bin.000002
                Relay_Log_Pos: 320
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: mysql,ysnc_wwww.ymq.io
          Replicate_Ignore_DB: information_schema
 .......
```


### 6.测试主主复制

**检查主主复制通信状态**

这个语句，在上面 Master1,Master2  执行过

```sh
mysql> show slave status\G

	 Slave_IO_Running: Yes
	Slave_SQL_Running: Yes
```

`Slave_IO_State`	       #从站的当前状态  
`Slave_IO_Running： Yes`   #读取主程序二进制日志的I/O线程是否正在运行  
`Slave_SQL_Running： Yes`  #执行读取主服务器中二进制日志事件的SQL线程是否正在运行。与I/O线程一样  
`Seconds_Behind_Master `   #是否为0，0就是已经同步了  

**必须都是 Yes**

如果不是原因主要有以下 4 个方面：  
  
1、网络不通  
2、密码不对  
3、MASTER_LOG_POS 不对 ps  
4、mysql 的	`auto.cnf` server-uuid 一样（可能你是复制的mysql）  

```sh			
$ find / -name 'auto.cnf'
$ cat /var/lib/mysql/auto.cnf
[auto]
server-uuid=6b831bf3-8ae7-11e7-a178-000c29cb5cbc # 按照这个16进制格式，修改server-uuid，重启mysql即可
```


在 Master1 创建测试库
```sql
mysql> CREATE DATABASE `replication_wwww.ymq.io`;
```

在 Master2 查看库是否同步过来
```sql
mysql> show databases;
+-------------------------+
| Database                |
+-------------------------+
| information_schema      |
| mysql                   |
| performance_schema      |
| replication_wwww.ymq.io |
| sys                     |
+-------------------------+
5 rows in set (0.01 sec)
```

在 Master1 创建测试表
```sql
mysql> use `replication_wwww.ymq.io`;
mysql> CREATE TABLE `sync_test` (`id` int(11) NOT NULL AUTO_INCREMENT, `name` varchar(255) NOT NULL, PRIMARY KEY (`id`) ) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;
```


在 Master1 查看表是否同步过来
```sql
mysql> use replication_wwww.ymq.io
mysql> show tables;
+-----------------------------------+
| Tables_in_replication_wwww.ymq.io |
+-----------------------------------+
| sync_test                         |
+-----------------------------------+
1 row in set (0.00 sec)
```


## MySQL双主多从

#  睡觉明天搞

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
   
   
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")
