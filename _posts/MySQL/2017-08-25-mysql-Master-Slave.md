---
layout: post
title: CentOs7.3 搭建 MySQL 5.7.19 主从同步复制，以及复制实现细节分析
categories: MySQL
description: CentOs7.3 搭建 MySQL 5.7.19 主从同步复制，以及复制实现细节分析
keywords: MySQL
---

# CentOs7.3 搭建 MySQL 5.7.19 主从同步复制，以及复制实现细节分析


[参考 MySQL官网 - 第16章主从复制](https://dev.mysql.com/doc/refman/5.7/en/replication.html)

## 概念

主从复制可以使MySQL数据库主服务器的主数据库，复制到一个或多个MySQL从服务器从数据库，默认情况下，复制异步; 从服务器不需要长连接接收主站的更新。根据配置，可以复制数据库中的所有数据库，选定的数据库或甚至选定的表。

## MySQL中复制的优点包括：

 - 横向扩展解决方案
 
在多个从库之间扩展负载以提高性能。在这种环境中，所有写入和更新都必须在主库上进行。但是，读取可能发生在一个或多个从库上。该模型可以提高写入的性能（由于主库专用于更新），同时在多个从库上读取，可以大大提高读取速度。

 - 数据安全性  
 
由于主库数据被复制到从库，从库可以暂停复制过程，可以在从库上运行备份服务，而不会破坏对应的主库数据。

 - 分析 - 可以在主库上创建实时数据，而信息分析可以在从库上进行，而不会影响主服务器的性能。

 - 长距离数据分发
 
可以使用复制创建远程站点使用的数据的本地副本，而无需永久访问主库。

## 环境

CentOS版本：CentOS 7.3.1611
Mysql版本：MySQL 5.7.19  
Master-Server : 192.168.252.121  
Slave-Server : 192.168.252.122  

## 1.准备工作

### 关闭防火墙

```
$ systemctl stop firewalld.service
```

### 安装 MySQL 

[参考 - CentOs7.3 安装 MySQL 5.7.19 二进制版本](https://segmentfault.com/a/1190000010864818)

首先在两台机器上装上，保证正常启动，可以使用


### 关闭MySQL 服务

```sh
$ service mysql.server stop
```

## 2.Master 主服务器配置


### 修改 my.cnf

配置 Master 以使用基于二进制日志文件位置的复制，必须启用二进制日志记录并建立唯一的服务器ID,否则则无法进行主从复制。

```sh
$ vi /etc/my.cnf

[mysqld]
log-bin=mysql-bin
server-id=1
```

进行更改后，重新启动服务。

```sh
$ service mysql.server restart
```

登录MySQL 查看 Master 状态

```sh
$ /usr/local/mysql/bin/mysql -uroot -p

mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

### 创建用户进行复制


每个从设备使用MySQL用户名和密码连接到主设备，因此主设备上必须有用户帐户，从站可以连接。任何帐户都可以用于此操作，只要它已被授予 REPLICATION SLAVE权限。您可以选择为每个从站创建不同的帐户，或者使用每个从站的相同帐户连接到主站。

虽然您不必专门为复制创建帐户，但应注意复制用户名和密码以纯文本格式存储在主信息存储库文件或表中（请参见 第16.2.4.2节“从属状态日志”） 。因此，您可能需要创建一个单独的帐户，该帐户只具有复制过程的权限，以尽可能减少对其他帐户的危害。

要创建一个新帐户，请使用CREATE USER。要将此帐户授予复制所需的权限，请使用该GRANT 语句。如果您仅为了复制而创建一个帐户，该帐户只需要 REPLICATION SLAVE权限。例如，要设置一个新用户，repl可以连接到mydomain.com域内任何主机进行复制 ，请在主服务器上发出以下语句：


```sql
mysql> CREATE USER 'repl'@'192.168.252.122' IDENTIFIED BY 'slavepass';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.252.122';
```


## 3.Slave 从服务器配置

### 修改 my.cnf
```sh
$ vi /etc/my.cnf

[mysqld]
server-id=2
```
进行更改后，重新启动服务器。

```sh
$ service mysql.server restart
```

如果要设置多个从库，则每个从库必须具有server-id与主库和其他从库不同的唯一值。


### 配置主库通信

要设置从库与主库进行通信，进行复制，使用必要的连接信息配置从库

在从库上执行以下语句，将选项值替换为与系统相关的实际值

参数格式

```sh
mysql> CHANGE MASTER TO
    ->     MASTER_HOST='master_host_name',
    ->     MASTER_USER='replication_user_name',
    ->     MASTER_PASSWORD='replication_password',
    ->     MASTER_LOG_FILE='recorded_log_file_name',
    ->     MASTER_LOG_POS=recorded_log_position;
```

我编辑好的

```sh
mysql>  CHANGE MASTER TO
    -> MASTER_HOST='node1',
    -> MASTER_USER='repl',
    -> MASTER_PASSWORD='slavepass',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=0;
Query OK, 0 rows affected, 2 warnings (0.02 sec)
```

### 启动从服务器复制线程
```sh
mysql>  START SLAVE;
Query OK, 0 rows affected (0.00 sec)
```

### 查看从服务器复制状态 

```sh 
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: node1
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000003
          Read_Master_Log_Pos: 154
               Relay_Log_File: node2-relay-bin.000004
                Relay_Log_Pos: 367
        Relay_Master_Log_File: mysql-bin.000003
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 154
              Relay_Log_Space: 787
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
                  Master_UUID: 511dd5da-8a30-11e7-97ed-000c29129bb0
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
1 row in set (0.00 sec)

ERROR: 
No query specified

mysql> 
```

### 确认主从同步配置

Slave_IO_State：从站的当前状态  

**注意** 以下两项必须是 `yes`,不能是Connecting.


Slave_IO_Running：读取主程序二进制日志的I/O线程是否正在运行 ,确保是 yes  
Slave_SQL_Running：执行读取主服务器中二进制日志事件的SQL线程是否正在运行。与I/O线程一样，确保是 yes  

[检查复制状态](https://dev.mysql.com/doc/refman/5.7/en/replication-administration-status.html)

## 4.操作 Master 主服务器

**【注意】以下操作都在 Master-Server : `192.168.252.121` 执行**

### 登录MySQL

```sh
$ /usr/local/mysql/bin/mysql -uroot -p
```

### 创建测试库

```sh
mysql> CREATE DATABASE sync_www_ymq_io;
Query OK, 1 row affected (0.00 sec)
```
### 切换测试库
```sh
mysql> use sync_www_ymq_io
Database changed
```

### 创建测试表

这个表用于测试主从库的，表结构变更，数据变更，是否真的同步了  

以下slq 在命令行 逐行执行，或者把`->`去掉，放在一行执行
 
```sh
mysql> CREATE TABLE `sync_test` (
    -> `id` int(11) NOT NULL AUTO_INCREMENT,
    -> `name` varchar(255) NOT NULL,
    -> PRIMARY KEY (`id`)
    -> ) ENGINE=InnoDB AUTO_INCREMENT=2 DEFAULT CHARSET=utf8;
Query OK, 0 rows affected (0.01 sec)
```
## 5.操作 Slave 从服务器

### 检查主从同步复制

### 1.检查 DATABASE

检查在 Master 主服务器，创建的 DATABASE `sync_www_ymq_io` 在 Slave 从服务器，是否同步过来

**登录MySQL**

```sh
$ /usr/local/mysql/bin/mysql -uroot -p
```

**查看所有数据库，发现，已经有了，在Master 主服务器创建 `sync_www_ymq_io`库**

```sql
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sync_www_ymq_io    |
| sys                |
+--------------------+
5 rows in set (0.00 sec)
```

### 2.检查 TABLE

**选择数据库**

```sh
mysql> use sync_www_ymq_io;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```

**查看所有表，发现，已经有了，在Master 主服务器创建 `sync_test`表**

```
mysql> show tables;
+---------------------------+
| Tables_in_sync_www_ymq_io |
+---------------------------+
| sync_test                 |
+---------------------------+
1 row in set (0.00 sec)
```

## 6.测试数据同步

###  Master主服务器

在 Master 主服务器，执行以下语句
 
```sh
mysql> INSERT INTO `sync_www_ymq_io`.`sync_test` (`name`) VALUES ('测试数据');
Query OK, 1 row affected (0.01 sec)

mysql> select * from sync_www_ymq_io.sync_test;
+----+--------------+
| id | name         |
+----+--------------+
|  2 | 测试数据     |
+----+--------------+
1 row in set (0.00 sec)
```

###  Slave 从服务器

在 Slave 从服务器，执行以下语句,发现数据已经同步过来了

```sh
mysql> select * from sync_www_ymq_io.sync_test;
+----+--------------+
| id | name         |
+----+--------------+
|  2 | 测试数据     |
+----+--------------+
1 row in set (0.00 sec)
```

## 7.复制实现细节分析

MySQL主从复制功能使用**三个线程实现**，**一个在主服务器上**，**两个在从服务器上**

### Binlog转储线程。  

当从服务器与主服务器连接时，主服务器会创建一个线程将二进制日志内容发送到从服务器。  
该线程可以使用 语句 `SHOW PROCESSLIST`(下面有示例介绍) 在服务器 sql 控制台输出中标识为Binlog Dump线程。

二进制日志转储线程获取服务器上二进制日志上的锁，用于读取要发送到从服务器的每个事件。一旦事件被读取，即使在将事件发送到从服务器之前，锁会被释放。

### 从服务器I/O线程。 

当在从服务器sql 控制台发出 `START SLAVE`语句时，从服务器将创建一个I/O线程，该线程连接到主服务器，并要求它发送记录在主服务器上的二进制更新日志。

从机I/O线程读取主服务器Binlog Dump线程发送的更新 （参考上面 Binlog转储线程 介绍），并将它们复制到自己的本地文件二进制日志中。

该线程的状态显示详情 Slave_IO_running 在输出端 使用 命令`SHOW SLAVE STATUS`  

使用`\G`语句终结符,而不是分号,是为了，易读的垂直布局  

这个命令在上面 **查看从服务器状态** 用到过
```sh
mysql> SHOW SLAVE STATUS\G
```

### 从服务器SQL线程。  

从服务器创建一条SQL线程来读取由主服务器I/O线程写入的二级制日志，并执行其中包含的事件。

在前面的描述中，每个主/从连接有三个线程。主服务器为每个当前连接的从服务器创建一个二进制日志转储线程，每个从服务器都有自己的I/O和SQL线程。
从服务器使用两个线程将读取更新与主服务器更新事件，并将其执行为独立任务。因此，如果语句执行缓慢，则读取语句的任务不会减慢。  

例如，如果从服务器开始几分钟没有运行，或者即使SQL线程远远落后，它的I/O线程也可以从主服务器建立连接时，快速获取所有二进制日志内容。  

如果从服务器在SQL线程执行所有获取的语句之前停止，则I/O线程至少获取已经读取到的内容，以便将语句的安全副本存储在自己的二级制日志文件中，准备下次执行主从服务器建立连接，继续同步。 

使用命令 `SHOW PROCESSLIST\G`  可以查看有关复制的信息


### 命令 SHOW FULL PROCESSLIST\G

**在 Master 主服务器 执行的数据示例**

```sh
mysql>  SHOW FULL PROCESSLIST\G
*************************** 1. row ***************************
     Id: 22
   User: repl
   Host: node2:39114
     db: NULL
Command: Binlog Dump
   Time: 4435
  State: Master has sent all binlog to slave; waiting for more updates
   Info: NULL
*************************** 2. row ***************************
     Id: 30
   User: root
   Host: 192.168.252.1:54471
     db: NULL
Command: Sleep
   Time: 966
  State: 
   Info: NULL
*************************** 3. row ***************************
     Id: 32
   User: root
   Host: 192.168.252.1:58767
     db: sync_www_ymq_io
Command: Sleep
   Time: 860
  State: 
   Info: NULL
*************************** 4. row ***************************
     Id: 36
   User: root
   Host: localhost
     db: sync_www_ymq_io
Command: Query
   Time: 0
  State: starting
   Info: SHOW FULL PROCESSLIST
4 rows in set (0.00 sec)
```

Id: 22是Binlog Dump服务连接的从站的复制线程  
Host: node2:39114 是从服务，主机名 级及端口  
State: 信息表示所有更新都已同步发送到从服务器，并且主服务器正在等待更多更新发生。  
如果Binlog Dump在主服务器上看不到 线程，意味着主从复制没有配置成功; 也就是说，没有从服务器连接主服务器。

### 命令 SHOW PROCESSLIST\G

**在 Slave 从服务器 执行的数据示例**

```sh
mysql> SHOW PROCESSLIST\G
*************************** 1. row ***************************
     Id: 6
   User: system user
   Host: 
     db: NULL
Command: Connect
   Time: 6810
  State: Waiting for master to send event
   Info: NULL
*************************** 2. row ***************************
     Id: 7
   User: system user
   Host: 
     db: NULL
Command: Connect
   Time: 3069
  State: Slave has read all relay log; waiting for more updates
   Info: NULL
*************************** 3. row ***************************
     Id: 8
   User: root
   Host: 192.168.252.1:54005
     db: www.ymq.io
Command: Sleep
   Time: 5828
  State: 
   Info: NULL
*************************** 4. row ***************************
     Id: 13
   User: root
   Host: 192.168.252.1:54472
     db: NULL
Command: Sleep
   Time: 3336
  State: 
   Info: NULL
*************************** 5. row ***************************
     Id: 14
   User: root
   Host: localhost
     db: sync_www_ymq_io
Command: Query
   Time: 0
  State: starting
   Info: SHOW PROCESSLIST
*************************** 6. row ***************************
     Id: 15
   User: root
   Host: 192.168.252.1:58785
     db: sync_www_ymq_io
Command: Sleep
   Time: 3247
  State: 
   Info: NULL
*************************** 7. row ***************************
     Id: 17
   User: root
   Host: 192.168.252.1:58919
     db: sync_www_ymq_io
Command: Sleep
   Time: 3226
  State: 
   Info: NULL
7 rows in set (0.01 sec)

```

**Id: 6**是与主服务器通信的I/O线程  
**Id: 7**是正在处理存储在中继日志中的更新的SQL线程  

在 运行 `SHOW PROCESSLIST` 命令时，两个线程都空闲，等待进一步更新  

如果在主服务器上在设置的超时，时间内 Binlog Dump线程没有活动，则主服务器会和从服务器断开连接。超时取决于的 **服务器系统变量 ** 值 net_write_timeout(在中止写入之前等待块写入连接的秒数，默认10秒)和 net_retry_count;(如果通信端口上的读取或写入中断，请在重试次数，默认10次) 设置 [服务器系统变量](https://dev.mysql.com/doc/refman/5.7/en/server-system-variables.html)

该SHOW SLAVE STATUS语句提供了有关从服务器上复制处理的附加信息。请参见 第16.1.7.1节“检查复制状态”。


## 8.更多常见主从复制问题：

[常见主从复制问题](https://dev.mysql.com/doc/refman/5.7/en/faqs-replication.html)

 - **作者：Peng Lei** 
 - **出处：[Segment Fault PengLei `Blog  专栏](https://segmentfault.com/a/1190000010867488)**      
 - **版权归作者所有，转载请注明出处** 



