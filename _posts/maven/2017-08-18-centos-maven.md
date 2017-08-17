---
layout: post
title: CentOs7.3 安装 maven3.5
categories: maven
description: CentOs7.3 安装 maven3.5
keywords: maven
---

## 下载解压

```sh
$ cd /opt
$ wget https://mirrors.tuna.tsinghua.edu.cn/apache/maven/maven-3/3.5.0/binaries/apache-maven-3.5.0-bin.tar.gz
$ tar xzf apache-maven-3.5.0-bin.tar.gz
```

## 重命名

```sh
$ cd /opt
$ mv apache-maven-3.5.0 /lib/maven
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
#maven
export MAVEN_HOME=/lib/maven
export PATH=${MAVEN_HOME}/bin:${PATH}
```

## 使环境变量生效

运行 `source /etc/profile`使`/etc/profile`文件生效

```sh
$ source /etc/profile
```

或者

```sh
$ source ~/.bashrc
```

## 验证

使用 `mvn -version`  命令查看jdk版本及其相关信息，不会出现`command not found`错误，且显示的版本信息与前面安装的一致

```sh
$ mvn -version
Apache Maven 3.5.0 (ff8f5e7444045639af65f6095c62210b5713f426; 2017-04-04T03:39:06+08:00)
Maven home: /lib/maven
Java version: 1.8.0_144, vendor: Oracle Corporation
Java home: /usr/lib/jvm/jre
Default locale: en_US, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-514.26.2.el7.x86_64", arch: "amd64", family: "unix"

```