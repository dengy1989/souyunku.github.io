---
layout: post
title: 离线部署 Cloudera Manager 5 和 CDH 5.12.1 及使用 CDH 部署 Hadoop 集群服务
categories: CDH
description: 离线部署 Cloudera Manager 5 和 CDH 5.12.1 及使用 CDH 部署 Hadoop 集群服务
keywords: CDH
---

# Cloudera Manager

Cloudera Manager 分为两个部分：CDH和CM。

CDH是Cloudera Distribution Hadoop的简称，顾名思义，就是cloudera公司发布的Hadoop版本，封装了Apache Hadoop，提供Hadoop所有的服务，包括HDFS,YARN,MapReduce以及各种相关的components:HBase, Hive, ZooKeeper,Kafka等。

CM是cloudera manager的简称，是CDH的管理平台，主要包括CM server, CM agent。通过CM可以对CDH进行配置，监测，报警，log查看，动态添加删除各种服务等。 


# 一、准备工作

## 环境

```sh
JDK:1.8  
centos:7.3

操作系统：CentOS 6
JDK 版本：1.7.0_80

所需安装包及版本说明：由于我们的操作系统为CentOS7，需要下载以下文件：

cloudera-manager-centos7-cm5.12.1_x86_64.tar.gz

CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel

CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel.sha1

manifest.json
```

Cloudera Manager 下载目录  
[http://archive.cloudera.com/cm5/cm/5/](http://archive.cloudera.com/cm5/cm/5/)

CDH 下载目录  
[http://archive.cloudera.com/cdh5/parcels/5.12.1/](http://archive.cloudera.com/cdh5/parcels/5.12.1/)

manifest.json 下载  
[http://archive.cloudera.com/cdh5/parcels/5.12.1/manifest.json](http://archive.cloudera.com/cdh5/parcels/5.12.1/manifest.json)

CHD5 相关的 Parcel 包放到主节点的`/opt/cloudera/parcel-repo/`目录中    

`CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel.sha1` 重命名为 `CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel.sha`

**这点必须注意**，否则，系统会重新下载 `CDH-5.12.1-1.cdh5.12.1.p0.3-el6.parcel` 文件  

**本文采用离线安装方式，在线安装方式请参照官方文** 

| 主机名 | ip地址 | 安装服务 |
| --- | --- | --- |
| node1 (Master) | 192.168.252.121 | jdk、cloudera-manager、MySql |
| node2 (Agents) | 192.168.252.122 | jdk、cloudera-manager |
| node3 (Agents) | 192.168.252.123 | jdk、cloudera-manager |
| node4 (Agents) | 192.168.252.124 | jdk、cloudera-manager |
| node5 (Agents) | 192.168.252.125 | jdk、cloudera-manager |
| node6 (Agents) | 192.168.252.126 | jdk、cloudera-manager |
| node7 (Agents) | 192.168.252.127 | jdk、cloudera-manager |

# 二、系统环境搭建

## 1、网络配置(所有节点)

**修改 hostname**

命令格式

```sh
hostnamectl set-hostname <hostname>
```

依次修改所有节点 `node`[1-7]

```sh
hostnamectl set-hostname node1
```

重启服务器

```sh
reboot
```

**修改映射关系**

1.在 node1 的 `/etc/hosts` 文件下添加如下内容

```sh
$ vi /etc/hosts
```

2.查看修改后的`/etc/hosts` 文件内容

```sh
[root@node7 ~]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.252.121 node1
192.168.252.122 node2
192.168.252.123 node3
192.168.252.124 node4
192.168.252.125 node5
192.168.252.126 node6
192.168.252.127 node7
```

## 2、SSH 免密码登录

1.在集群node1的 `/etc/ssh/sshd_config ` 文件去掉以下选项的注释

```sh
vi /etc/ssh/sshd_config 
```

```sh
RSAAuthentication yes      #开启私钥验证
PubkeyAuthentication yes   #开启公钥验证
```

2.将集群node1 修改后的 `/etc/ssh/sshd_config ` 通过 `scp` 命令复制发送到集群的每一个节点

```sh
for a in {2..7} ; do scp /etc/ssh/sshd_config node$a:/etc/ssh/sshd_config ; done
```

3.生成公钥、私钥

1.在集群的每一个节点节点输入命令 `ssh-keygen -t rsa -P ''`，生成 key，一律回车

```sh
ssh-keygen -t rsa -P ''
```

4.在集群的node1 节点输入命令

将集群每一个节点的公钥`id_rsa.pub`放入到自己的认证文件中`authorized_keys`;

```sh
for a in {1..7}; do ssh root@node$a cat /root/.ssh/id_rsa.pub >> /root/.ssh/authorized_keys; done
```

5.在集群的node1 节点输入命令

将自己的认证文件 `authorized_keys` ` 通过 `scp` 命令复制发送到每一个节点上去: `/root/.ssh/authorized_keys`

```sh
for a in {1..7}; do scp /root/.ssh/authorized_keys root@node$a:/root/.ssh/authorized_keys ; done
```

6.在集群的每一个节点节点输入命令

接重启ssh服务

```sh
sudo systemctl restart sshd.service
```

7.验证 ssh 无密登录

开一个其他窗口测试下能否免密登陆

例如：在node3

```sh
ssh root@node2
```

`exit` 退出


## 3、关闭防火墙

```sh
systemctl stop firewalld.service
```


## 4、关闭 SELINUX

**查看**

```sh
[root@node1 ~]# getenforce
Enforcing
[root@node1 ~]# /usr/sbin/sestatus -v
SELinux status:  
```

**临时关闭**

```sh
## 设置SELinux 成为permissive模式
## setenforce 1 设置SELinux 成为enforcing模式
setenforce 0
```

**永久关闭**

```sh
vi /etc/selinux/config
```

将 `SELINUX=enforcing` 改为 `SELINUX=disabled` 

设置后需要重启才能生效


PS 我是修改`node1` 的 `/etc/selinux/config` 后，把配置文件复制到其他节点

```sh
for a in {2..7}; do scp /etc/selinux/config root@node$a:/etc/selinux/config ; done
```

重启所有节点
```sh
reboot
```
## 5、安装 JDK

[下载Linux环境下的jdk1.8，请去（官网）中下载jdk的安装文件](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

我在百度云盘分下的链接：[http://pan.baidu.com/s/1jIFZF9s](http://pan.baidu.com/s/1jIFZF9s) 密码：u4n4

上传在 `/opt` 目录

解压

```sh
cd /opt
tar zxvf jdk-8u144-linux-x64.tar.gz
mv jdk1.8.0_144/ /lib/jvm
```

配置环境变量

```sh
vi /etc/profile
```

```sh
#jdk
export JAVA_HOME=/lib/jvm
export JRE_HOME=${JAVA_HOME}/jre   
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib   
export PATH=${JAVA_HOME}/bin:$PATH 
```

使环境变量生效
```
source /etc/profile
```

验证

```sh
[root@localhost ~]# java -version
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)
```

## 6、设置 NTP

所有节点安装 NTP

```sh
yum install ntp
```

设置同步

```sh
ntpdate -d 182.92.12.11
```

## 7、安装配置 MySql

主节点 安装 MySql

MySQL依赖于libaio 库

```sh
yum search libaio
yum install libaio
```
 
下载，解压，重命名

通常解压在 `/usr/local/mysql` 

把`mysql-5.7.19-linux-glibc2.12-x86_64` 文件夹，重命名成`mysql`,这样就凑成`/usr/local/mysql`目录了

```sh
cd /opt/
wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.19-linux-glibc2.12-x86_64.tar.gz
tar -zxvf /opt/mysql-5.7.19-linux-glibc2.12-x86_64.tar.gz -C /usr/local/
mv /usr/local/mysql-5.7.19-linux-glibc2.12-x86_64/ /usr/local/mysql
```

**1. 新建用户组和用户**

```sh
groupadd mysql
useradd mysql -g mysql
```

**2. 创建目录并授权**

```sh
cd /usr/local/mysql/ 
mkdir data mysql-files
chmod 750 mysql-files
chown -R mysql .
chgrp -R mysql .
```

**3. 初始化MySQL**

```sh
bin/mysqld --initialize --user=mysql # MySQL 5.7.6 and up
```

**4. 注意密码 mysql 临时密码**

 > [注意]root@localhost生成临时密码：`;b;s;)/rn6A3`,也就是`root@localhost:`后的字符串

```sh
2017-09-24T08:34:08.643206Z 1 [Note] A temporary password is generated for root@localhost: D<qha)5gtr<!
```

**5. 授予读写权限**

```
chown -R root .
chown -R mysql data mysql-files
```

**6. 添加到MySQL 启动脚本到系统服务**

```
cp support-files/mysql.server /etc/init.d/mysql.server
```

**7. 给日志目录授予读写权限**

```sh
mkdir /var/log/mariadb
touch /var/log/mariadb/mariadb.log
chown -R mysql:mysql /var/log/mariadb
```

**8. 修改 /etc/my.cnf**

```sh
vi /etc/my.cnf
```

修改 `[mysqld]`组下的 `socket` 路径，注释掉`/var/lib/mysql/mysql.sock`，加一行为`tmp/mysql.soc`
```sh
[mysqld]
datadir=/var/lib/mysql
#socket=/var/lib/mysql/mysql.sock
socket=/tmp/mysql.sock
```

**9.启动MySQL服务**

```sh
service mysql.server start
```

或者

```sh
/usr/local/mysql/support-files/mysql.server start
```

**10. 登录MySQL**
```sh
/usr/local/mysql/bin/mysql -uroot -p
Enter password: 
```

**如果不知道密码**
密码在，安装MySQL步骤 4 ，有提到，怎么找初始化临时密码

**11. 设置MySQL密码**

登陆成功后，设置MySQL密码

```sh
mysql> ALTER USER   'root'@'localhost' identified by 'mima';
mysql> flush privileges;
```

**12. 开启远程登录**

```sql
mysql> grant all privileges on *.*  to  'root'@'%'  identified by 'mima'  with grant option;
mysql> flush privileges;
mysql> exit;
```

## 8、下载依赖包

```sh
yum -y install chkconfig
yum -y install bind-utils
yum -y install psmisc
yum -y install libxslt
yum -y install zlib
yum -y install sqlite
yum -y install cyrus-sasl-plain
yum -y install cyrus-sasl-gssapi
yum -y install fuse
yum -y install portmap
yum -y install fuse-libs
yum -y install redhat-lsb
```

# 三、cloudera manager Server & Agent 安装

## 1、安装 CM Server & Agent


在所有节点，创建`/opt/cloudera-manager`

```sh
mkdir /opt/cloudera-manager
```

把下载好的`cloudera-manager-centos7-cm5.12.1_x86_64.tar.gz`安装包上传至  node1 节点`/opt/`目录


在 node1 节点拷贝 `cloudera-manager-centos7-cm5.12.1_x86_64.tar.gz` 到所有 `Server、Agent` 节点创建 `/opt/cloudera-manager` 目录：

```sh
for a in {2..7}; do scp /opt/cloudera-manager-*.tar.gz root@node$a:/opt/ ; done
```

所有 `Server、Agent` 节点节点解压安装 Cloudera Manager Server & Agent

```sh
cd /opt
tar xvzf cloudera-manager*.tar.gz -C /opt/cloudera-manager
```

## 2、创建用户 cloudera-scm（所有节点）

**cloudera-scm 用户说明，摘自官网：**

Cloudera Manager Server and managed services are configured to use the user account cloudera-scm by default, creating a user with this name is the simplest approach. This created user, is used automatically after installation is complete.

Cloudera管理器服务器和托管服务被配置为在默认情况下使用用户帐户Cloudera-scm，创建具有这个名称的用户是最简单的方法。创建用户，在安装完成后自动使用。


执行：在所有节点创建cloudera-scm用户

```sh
useradd --system --home=/opt/cloudera-manager/cm-5.12.1/run/cloudera-scm-server/ --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm
```

## 3、配置 CM Agent

修改 node1 节点

`/opt/cloudera-manager/cm-5.12.1/etc/cloudera-scm-agent/config.ini`中的`server_host`为主节点的主机名。

```
cd /opt/cloudera-manager/cm-5.12.1/etc/cloudera-scm-agent/
vi config.ini
```

在node1 操作将 node1 节点修改后的 (复制到所有节点)  
```sh
for a in {1..7}; do scp /opt/cloudera-manager/cm-5.12.1/etc/cloudera-scm-agent/config.ini root@node$a:/opt/cloudera-manager/cm-5.12.1/etc/cloudera-scm-agent/config.ini ; done
```

## 4、配置 CM Server 的数据库

在主节点 node1 初始化CM5的数据库：

下载 mysql 驱动包

```sh
cd /opt/cloudera-manager/cm-5.12.1/share/cmf/lib
wget http://maven.aliyun.com/nexus/service/local/repositories/hongkong-nexus/content/Mysql/mysql-connector-java/5.1.38/mysql-connector-java-5.1.38.jar
```

启动MySQL服务
```sh
service mysql.server start
```

```sh
cd /opt/cloudera-manager/cm-5.12.1/share/cmf/schema/

./scm_prepare_database.sh mysql cm -h node1 -uroot -pmima --scm-host node1 scm scm scm
```

**看到如下信息，恭喜您，配置没毛病**

```sh
[main] DbCommandExecutor              INFO  Successfully connected to database.
All done, your SCM database is configured correctly!

```

格式:

```sh
scm_prepare_database.sh mysql cm -h <hostName> -u<username>  -p<password> --scm-host <hostName>  scm scm scm

对应于：数据库类型  数据库 服务器 用户名 密码  –scm-host  Cloudera_Manager_Server 所在节点……
```

## 5、创建 Parcel 目录

**Manager** 节点创建目录`/opt/cloudera/parcel-repo`，执行：

将下载好的文件

```sh
CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel
CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel.sha
manifest.json
```
 拷贝到该目录下。  

```sh
mkdir -p /opt/cloudera/parcel-repo
chown cloudera-scm:cloudera-scm /opt/cloudera/parcel-repo
cd /opt/cloudera/parcel-repo

```

重命名，`CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel.sha1` 否则，系统会重新下载  `CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel`

```sh
mv CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel.sha1 CDH-5.12.1-1.cdh5.12.1.p0.3-el7.parcel.sha
```

**Agent** 节点创建目录/opt/cloudera/parcels，执行：

```sh
mkdir -p /opt/cloudera/parcels
chown cloudera-scm:cloudera-scm /opt/cloudera/parcels
```


## 6、启动 CM Manager&Agent 服务

**注意，mysql 服务启动，防火墙关闭**


在 node1 (master) 执行：

Server

```sh
/opt/cloudera-manager/cm-5.12.1/etc/init.d/cloudera-scm-server start
```

在 node2-7 (Agents) 执行：

Agents

```sh
/opt/cloudera-manager/cm-5.12.1/etc/init.d/cloudera-scm-agent start
```

访问 [http://Master:7180](http://node1:7180) 若可以访问（用户名、密码：admin），则安装成功。

Manager 启动成功需要等待一段时间，过程中会在数据库中创建对应的表需要耗费一些时间。


# 四、CDH5 安装

CM Manager && Agent 成功启动后，登录前端页面进行 CDH 安装配置。

![][1]

![][2]

![][3]

免费版本的 CM5 已经去除 50 个节点数量的限制。

![][4]

各个 Agent 节点正常启动后，可以在当前管理的主机列表中看到对应的节点。

![][5]

选择要安装的节点，点继续。

![][6]

点击，继续，如果配置本地 Parcel 包无误，那么下图中的已下载，应该是瞬间就完成了，然后就是耐心等待分配过程就行了，大约 10 多分钟吧，取决于内网网速。

（若本地 Parcel 有问题，重新检查步骤三、5 是否配置正确）

![][7]


![][8]


点击，继续，如果配置本地Parcel包无误，那么下图中的已下载，应该是瞬间就完成了，然后就是耐心等待分配过程就行了，大约10多分钟吧，取决于内网网速。


![][9]

![][10]

## 遇到问题

**问题一**  
接下来是服务器检查，可能会遇到以下问题：

Cloudera 建议将 `/proc/sys/vm/swappiness` 设置为最大值 10。当前设置为 30。 
 
使用 `sysctl` 命令在运行时更改该设置并编辑 `/etc/sysctl.conf`，以在重启后保存该设置。  

您可以继续进行安装，但 Cloudera Manager 可能会报告您的主机由于交换而运行状况不良。以下主机将受到影响：node[2-7]  

```sh
echo 0 > /proc/sys/vm/swappiness
```

**问题二**  
已启用透明大页面压缩，可能会导致重大性能问题。请运行    
`echo never > /sys/kernel/mm/transparent_hugepage/defrag`和 `echo never > /sys/kernel/mm/transparent_hugepage/enabled`  
以禁用此设置，然后将同一命令添加到 /etc/rc.local 等初始化脚本中，以便在系统重启时予以设置。以下主机将受到影响: node[2-7]

```sh
echo never > /sys/kernel/mm/transparent_hugepage/defrag
echo never > /sys/kernel/mm/transparent_hugepage/enabled 
```

![][11]

![][12]

![][13]

![][14]

![][15]

![][16]

![][17]

![][18]

# 五、脚本

## MySql 建库&&删库

1、MySql 建库&&删库

amon

```sh
create database amon DEFAULT CHARACTER SET utf8; 
grant all on amon.* TO 'amon'@'%' IDENTIFIED BY 'amon';
```

hive

```sh
create database hive DEFAULT CHARACTER SET utf8; 
grant all on hive.* TO 'hive'@'%' IDENTIFIED BY 'hive';
```

oozie

```sh
create database oozie DEFAULT CHARACTER SET utf8; 
grant all on oozie.* TO 'oozie'@'%' IDENTIFIED BY 'oozie';
```

[1]: http://www.ymq.io/images/2017/CDH/1.png
[2]: http://www.ymq.io/images/2017/CDH/2.png
[3]: http://www.ymq.io/images/2017/CDH/3.png
[4]: http://www.ymq.io/images/2017/CDH/4.png
[5]: http://www.ymq.io/images/2017/CDH/5.png
[6]: http://www.ymq.io/images/2017/CDH/6.png
[7]: http://www.ymq.io/images/2017/CDH/7.png
[8]: http://www.ymq.io/images/2017/CDH/8.png
[9]: http://www.ymq.io/images/2017/CDH/9.png
[10]: http://www.ymq.io/images/2017/CDH/10.png
[11]: http://www.ymq.io/images/2017/CDH/11.png
[12]: http://www.ymq.io/images/2017/CDH/12.png
[13]: http://www.ymq.io/images/2017/CDH/13.png
[14]: http://www.ymq.io/images/2017/CDH/14.png
[15]: http://www.ymq.io/images/2017/CDH/15.png
[16]: http://www.ymq.io/images/2017/CDH/16.png
[17]: http://www.ymq.io/images/2017/CDH/17.png
[18]: http://www.ymq.io/images/2017/CDH/18.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")