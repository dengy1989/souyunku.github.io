---
layout: post
title: Centos7.3 安装 RabbitMQ
categories: RabbitMQ
description: Centos7.3 安装 RabbitMQ
keywords: RabbitMQ
---

# 环境

 - VMware版本号：12.0.0
 - CentOS版本：CentOS 7.3.1611
 - 三台虚拟机(IP)：192.168.252.101,192.168.102..102,192.168.252.103

## 注意事项
 

关闭防火墙

centos 6.x
```sh
$ service iptables stop # 关闭命令：
```

centos 7.x
```sh
$ systemctl stop firewalld.service # 停止firewall
```

 
不想关闭，就开放15672端口
```sh
firewall-cmd --zone=public --add-port=15672/tcp --permanent
firewall-cmd --reload 
```

## 安装 Erlang
 
RabbitMQ 安装需要依赖 Erlang 环境

```sh
$ wget http://www.rabbitmq.com/releases/erlang/erlang-19.0.4-1.el7.centos.x86_64.rpm

$ yum install erlang-19.0.4-1.el7.centos.x86_64.rpm

```

## 安装 RabbitMQ

```sh
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


<img src="/images/2017/rabbit/rabbit-login.png" />

# 授权操作


## 添加用户

处于安全的考虑，guest这个默认的用户只能通过http://localhost:15672 来登录，其他的IP无法直接使用这个账号。 这对于服务器上没有安装桌面的情况是无法管理维护的，除非通过在前面添加一层代理向外提供服务，这个又有些麻烦了，这里通过配置文件来实现这个功能

```sh
$ rabbitmqctl add_user ymq 123456
Creating user "ymq"
```

## 用户授权

```sh
$ rabbitmqctl set_permissions -p "/" ymq ".*" ".*" ".*"
Setting permissions for user "ymq" in vhost "/"

```
















