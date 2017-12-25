---
layout: post
title: CentOs7.3 搭建 RabbitMQ 3.6 单机服务
categories: RabbitMQ
description: CentOs7.3 搭建 RabbitMQ 3.6 单机服务
keywords: RabbitMQ
---

# CentOs7.3 搭建 RabbitMQ 3.6 单机服务

## RabbitMQ简介
 
RabbitMQ是一个开源的AMQP实现，服务器端用Erlang语言编写，支持多种客户端，如：Python、Ruby、.NET、Java、JMS、C、PHP、ActionScript、XMPP、STOMP等，支持AJAX。用于在分布式系统中存储转发消息，在易用性、扩展性、高可用性等方面表现不俗。

AMQP，即Advanced message Queuing Protocol，高级消息队列协议，是应用层协议的一个开放标准，为面向消息的中间件设计。消息中间件主要用于组件之间的解耦，消息的发送者无需知道消息使用者的存在，反之亦然。

AMQP的主要特征是面向消息、队列、路由（包括点对点和发布/订阅）、可靠性、安全。

## 环境
 - VMware版本号：12.0.0
 - CentOS版本：CentOS 7.3.1611
 - 虚拟机(IP)：192.168.252.101

## 注意事项
 

关闭防火墙

centos 6.x 关闭 iptables
```sh
$ service iptables stop # 关闭命令：
```

centos 7.x 关闭firewall

```sh
$ systemctl stop firewalld.service # 停止firewall
```

 
不想关闭防火墙，就开放15672端口,设置之后可以通过网页方式管理MQ

安装安装iptables防火墙

```sh
yum install iptables-services
```

编辑配置
```sh
$ vi /etc/sysconfig/iptables-config
```

添加配置
```sh
iptables -I INPUT -p tcp --dport 5672 -j ACCEPT
iptables -I INPUT -p tcp --dport 15672 -j ACCEPT
```

保存配置
```sh
$ service iptables save
```

重启
```sh
systemctl restart iptables.service
```

设置开机自启动
```sh
systemctl enable iptables.service 
```
 
[CentOS7.3 安装 iptables 与详细使用 https://segmentfault.com/a/1190000010713423 ](https://segmentfault.com/a/1190000010713423)
 
# 安装

## 安装 Erlang
 
RabbitMQ 安装需要依赖 Erlang 环境

```sh
$ cd /opt
$ wget http://www.rabbitmq.com/releases/erlang/erlang-19.0.4-1.el7.centos.x86_64.rpm

$ yum install erlang-19.0.4-1.el7.centos.x86_64.rpm

```

## 安装 RabbitMQ

```sh
$ cd /opt
$ wget http://www.rabbitmq.com/releases/rabbitmq-server/v3.6.10/rabbitmq-server-3.6.10-1.el7.noarch.rpm
$ yum install rabbitmq-server-3.6.10-1.el7.noarch.rpm
```

## 启动服务

```sh
$ service rabbitmq-server start
```

## 服务状态
```sh
$ service rabbitmq-server status
```

```
# service rabbitmq-server status
Redirecting to /bin/systemctl status  rabbitmq-server.service
● rabbitmq-server.service - RabbitMQ broker
   Loaded: loaded (/usr/lib/systemd/system/rabbitmq-server.service; disabled; vendor preset: disabled)
   Active: active (running) since Wed 2017-08-16 11:43:33 CST; 8s ago
 Main PID: 17919 (beam)
   Status: "Initialized"
   CGroup: /system.slice/rabbitmq-server.service
           ├─17919 /usr/lib64/erlang/erts-8.0.3/bin/beam -W w -A 64 -P 1048576 -t 5000000 -stbt db -zdbbl 32000 -K true -- -root /usr/lib64/erlang -progname erl -- -home /var/lib/rabbitmq -- -pa /us...
           ├─18062 /usr/lib64/erlang/erts-8.0.3/bin/epmd -daemon
           ├─18160 erl_child_setup 1024
           ├─18165 inet_gethost 4
           └─18166 inet_gethost 4

Aug 16 11:43:32 localhost.localdomain rabbitmq-server[17919]: RabbitMQ 3.6.10. Copyright (C) 2007-2017 Pivotal Software, Inc.
Aug 16 11:43:32 localhost.localdomain rabbitmq-server[17919]: ##  ##      Licensed under the MPL.  See http://www.rabbitmq.com/
Aug 16 11:43:32 localhost.localdomain rabbitmq-server[17919]: ##  ##
Aug 16 11:43:32 localhost.localdomain rabbitmq-server[17919]: ##########  Logs: /var/log/rabbitmq/rabbit@localhost.log
Aug 16 11:43:32 localhost.localdomain rabbitmq-server[17919]: ######  ##        /var/log/rabbitmq/rabbit@localhost-sasl.log
Aug 16 11:43:32 localhost.localdomain rabbitmq-server[17919]: ##########
Aug 16 11:43:32 localhost.localdomain rabbitmq-server[17919]: Starting broker...
Aug 16 11:43:33 localhost.localdomain rabbitmq-server[17919]: systemd unit for activation check: "rabbitmq-server.service"
Aug 16 11:43:33 localhost.localdomain systemd[1]: Started RabbitMQ broker.
Aug 16 11:43:33 localhost.localdomain rabbitmq-server[17919]: completed with 0 plugins.

```

## 查看日志

```sh
$ less /var/log/rabbitmq/rabbit@localhost.log
```


```sh
=INFO REPORT==== 16-Aug-2017::11:43:32 ===
Starting RabbitMQ 3.6.10 on Erlang 19.0.4
Copyright (C) 2007-2017 Pivotal Software, Inc.
Licensed under the MPL.  See http://www.rabbitmq.com/

=INFO REPORT==== 16-Aug-2017::11:43:32 ===
node           : rabbit@localhost
home dir       : /var/lib/rabbitmq
config file(s) : /etc/rabbitmq/rabbitmq.config (not found)
cookie hash    : kuUba2xGLitNNO48qE0Hrg==
log            : /var/log/rabbitmq/rabbit@localhost.log
sasl log       : /var/log/rabbitmq/rabbit@localhost-sasl.log
database dir   : /var/lib/rabbitmq/mnesia/rabbit@localhost

=INFO REPORT==== 16-Aug-2017::11:43:33 ===
Memory limit set to 390MB of 976MB total.

=INFO REPORT==== 16-Aug-2017::11:43:33 ===
Enabling free disk space monitoring

=INFO REPORT==== 16-Aug-2017::11:43:33 ===
Disk free limit set to 50MB

=INFO REPORT==== 16-Aug-2017::11:43:33 ===
Limiting to approx 924 file handles (829 sockets)

=INFO REPORT==== 16-Aug-2017::11:43:33 ===
FHC read buffering:  OFF
FHC write buffering: ON

=INFO REPORT==== 16-Aug-2017::11:43:33 ===
Database directory at /var/lib/rabbitmq/mnesia/rabbit@localhost is empty. Initialising from scratch...

=INFO REPORT==== 16-Aug-2017::11:43:33 ===
Waiting for Mnesia tables for 30000 ms, 9 retries left

=INFO REPORT==== 16-Aug-2017::11:43:33 ===
Waiting for Mnesia tables for 30000 ms, 9 retries left
```


**这里显示的是没有找到配置文件，我们可以自己创建这个文件**

```sh
config file(s) : /etc/rabbitmq/rabbitmq.config (not found)
```

**创建`rabbitmq.config`**

```sh
$ cd /etc/rabbitmq/
$ vi rabbitmq.config
```

编辑内容如下：
```sh
[{rabbit, [{loopback_users, []}]}].
```

 - 这里的意思是开放使用，rabbitmq默认创建的用户guest，密码也是guest，这个用户默认只能是本机访问，localhost或者127.0.0.1，从外部访问需要添加上面的配置。


保存配置后重启服务

```sh
$ service rabbitmq-server restart
```

## 开启管理UI


```sh
$ /sbin/rabbitmq-plugins enable rabbitmq_management
```

重启服务

```sh
$ service rabbitmq-server restart
```

## 访问管理UI

通过 http://ip:15672 使用guest,guest 进行登陆了

**如果不能访问，请检查防火墙**

<img src="http://www.ymq.io/images/2017/rabbit/rabbit-login.png" />

# 授权操作


## 添加用户

处于安全的考虑，guest这个默认的用户只能通过`http://localhost:15672` 来登录，其他的IP无法直接使用这个账号。 这对于服务器上没有安装桌面的情况是无法管理维护的，除非通过在前面添加一层代理向外提供服务，这个又有些麻烦了，这里通过配置文件来实现这个功能

命令格式
```sh
rabbitmqctl add_user <username> <newpassword> 
```

```sh
$ rabbitmqctl add_user ymq 123456
Creating user "ymq"
```

## 删除用户

命令格式
```sh
rabbitmqctl delete_user <username>
```

```sh
$ rabbitmqctl  delete_user penglei
Deleting user "penglei"
```

## 修改密码

命令格式
```sh
rabbitmqctl  change_password  <username> <newpassword> 
```

```sh
$ rabbitmqctl  change_password  ymq 123456
Changing password for user "ymq"
```

## 用户授权

命令格式

```sh
rabbitmqctl set_permissions [-pvhostpath] {user} {conf} {write} {read}
```

该命令使用户ymq /(可以访问虚拟主机) 中所有资源的配置、写、读权限以便管理其中的资源

```sh
$ rabbitmqctl set_permissions -p "/" ymq ".*" ".*" ".*"
Setting permissions for user "ymq" in vhost "/"

```

## 查看用户授权

命令格式
```sh
rabbitmqctl  list_permissions  [-p  VHostPath]
```

```sh
$ rabbitmqctl list_permissions -p /
Listing permissions in vhost "/"
guest	.*	.*	.*
ymq	.*	.*	.*
```

## 查看当前用户列表

可以看到添加用户成功了，但不是`administrator`角色

```sh
$ rabbitmqctl list_users
Listing users
guest	[administrator]
ymq	[]
```

## 添加角色

这里我们也将ymq用户设置为`administrator`角色

命令格式

```sh
rabbitmqctl set_user_tags <username> <tag>
```

```sh
$ rabbitmqctl set_user_tags ymq administrator
Setting tags for user "ymq" to [administrator]
```

再次查看权限

```sh
$ rabbitmqctl list_users
Listing users
guest	[administrator]
ymq	[administrator]
```

清除权限信息

命令格式
```sh
rabbitmqctl  clear_permissions  [-p VHostPath]  ymq
```

```sh
rabbitmqctl  clear_permissions  -p /  ymq
Clearing permissions for user "ymq" in vhost "/"
```

## 官方文档

 - 安装：[https://www.rabbitmq.com/install-debian.html](https://www.rabbitmq.com/install-debian.html)
 - 访问控制：[https://www.rabbitmq.com/access-control.html](https://www.rabbitmq.com/access-control.html)
 - 网络：[https://www.rabbitmq.com/networking.html](https://www.rabbitmq.com/access-control.html)
 - 配置：[https://www.rabbitmq.com/configure.html](https://www.rabbitmq.com/configure.html)
 - 集群：[https://www.rabbitmq.com/clustering.html](https://www.rabbitmq.com/configure.html)
 - 命令：[https://www.rabbitmq.com/man/rabbitmqctl.1.man.html#set_user_tags](https://www.rabbitmq.com/man/rabbitmqctl.1.man.html#set_user_tags)
	


# 后台操作

## 登录新用户

可以看到 ymq 和 guest 的权限 一样

<img src="http://www.ymq.io/images/2017/rabbit/ymq-login.png" />


## 添加用户

鼠标点击，划红线的角色，选择一种

<img src="http://www.ymq.io/images/2017/rabbit/add-user.png" />

## 设置权限

该用户无权访问任何虚拟主机

<img src="http://www.ymq.io/images/2017/rabbit/update-user.png" />

点击 Set permission

 - 设置可以访问虚拟主机 中所有资源的配置、写、读权限以便管理其中的资源

<img src="http://www.ymq.io/images/2017/rabbit/setPermission.png" />


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

