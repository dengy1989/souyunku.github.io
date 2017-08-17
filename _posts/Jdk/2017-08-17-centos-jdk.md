---
layout: post
title: CentOs7.3 安装 JDK1.8
categories: JDK
description: CentOs7.3 安装 Jdk1.8
keywords: JDK
---

## 下载

[下载Linux环境下的jdk1.8，请去（官网）中下载jdk的安装文件](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)


我在百度云盘分下的链接：[http://pan.baidu.com/s/1jIFZF9s](http://pan.baidu.com/s/1jIFZF9s) 密码：u4n4

上传在 `/opt` 目录

## 解压

```sh
$ cd /opt
$ tar zxvf jdk-8u144-linux-x64.tar.gz
```

## 重命名

```sh
$ cd /opt
$ mv jdk1.8.0_144/ /lib/jvm
```

## 配置环境变量

如果是对所有的用户都生效就修改`vi /etc/profile` 文件

如果只针对当前用户生效就修改 `vi ~/.bahsrc `文件

在文件底部添加如下代码，如果上一步的路径和我的不一致要改一下

```sh
$ vi /etc/profile
```
环境变量配置内容

```sh
#jdk
export JAVA_HOME=/lib/jvm
export JRE_HOME=${JAVA_HOME}/jre   
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib   
export PATH=${JAVA_HOME}/bin:$PATH 
```

## 使环境变量生效

运行 `source /etc/profile`使`/etc/profile`文件生效

```
$ source /etc/profile
```
或者
```
$ source ~/.bashrc
```

## 验证

使用 java -version 和 javac -version 命令查看jdk版本及其相关信息，不会出现command not found错误，且显示的版本信息与前面安装的一致

```
[root@localhost ~]# java -version
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 1.8.0_144-b01)
Java HotSpot(TM) 64-Bit Server VM (build 25.144-b01, mixed mode)
```