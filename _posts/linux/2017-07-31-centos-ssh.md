---
layout: post
title: CentOs7.3 ssh 免密登录
categories: Linux
description: CentOs7.3 ssh 免密登录
keywords: Linux
---

## 修改所有主机 `/etc/hosts` 文件

三台虚拟机(IP)：192.168.252.101,192.168.102..102,192.168.252.103

```sh
$ vi /etc/hosts
```

添加如下内容

```sh
192.168.252.101 node1
192.168.252.102 node2
192.168.252.103 node3
```


## 启动 ssh 无密登录

CentOS 默认没有启动 ssh 无密登录,去掉 `/etc/ssh/sshd_config` 其中 2 行的注释，每台服务器都要设置。

```sh
$ vi /etc/ssh/sshd_config 
```

``` 
# 去掉这下满的 “#” 注释
#RSAAuthentication yes
#PubkeyAuthentication yes
```

## 生成公钥、私钥对

每台服务器下都输入命令 `ssh-keygen -t rsa`，生成 key，一律不输入密码，直接回车，/root 就会生成 `.ssh `文件夹。

```sh
$ ssh-keygen -t rsa
```

## 合并公钥

每台服务器下都输入命令,进入到生成密钥文件夹中，`/root/.ssh/` 一个隐藏的.ssh文件夹中。

```sh
$ cd /root/.ssh 
$ cat id_rsa.pub >> authorized_keys  
```

## 修改 hosts 文件

`vi /etc/hosts`

```
192.168.252.101 node1
192.168.252.102 node2
192.168.252.103 node2
```

## 复制公钥

把 node1 服务器的 `id_rsa` ，`authorized_keys` 复制到 `node2,node3` 服务器的 `/root/.ssh` 目录

```sh
$ scp id_rsa node2:/root/.ssh/
$ scp id_rsa node3:/root/.ssh/
```

```sh
$ scp authorized_keys node2:/root/.ssh/
$ scp authorized_keys node3:/root/.ssh/
```


## 验证登录

此时ssh登录 node2，node3 不用输入密码

```sh
$ ssh node2
$ ssh node3
```






































