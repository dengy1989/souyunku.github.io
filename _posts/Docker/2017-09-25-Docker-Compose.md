---
layout: post
title: Docker Compose 1.16.1 安装
categories: Docker Compose
description: Docker Compose 1.16.1 安装
keywords: Docker Compose
---

# Docker Compose 简介

一个使用Docker容器的应用，通常由多个容器组成。使用Docker Compose，不再需要使用shell脚本来启动容器。在配置文件中，所有的容器通过services来定义，然后使用docker-compose脚本来启动，停止和重启应用，和应用中的服务以及所有依赖服务的容器

**完整的命令列表如下：**

- `build` 构建或重建服务
- `help` 命令帮助
- `kill` 杀掉容器
- `logs` 显示容器的输出内容
- `port` 打印绑定的开放端口
- `ps` 显示容器
- `pull` 拉取服务镜像
- `restart` 重启服务
- `rm` 删除停止的容器
- `run` 运行一个一次性命令
- `scale` 设置服务的容器数目
- `start` 开启服务
- `stop` 停止服务
- `up` 创建并启动容器


## 环境

```sh
centos:7.3  
Docker CE: 17.06.2
Docker Compose: 1.16.1
```

[参考-https://docs.docker.com/compose/gettingstarted/#prerequisites](https://docs.docker.com/compose/gettingstarted/#prerequisites)



[在Linux上，您可以从GitHub上的Compose存储库版本页面下载Docker Compose 最新二进制文件](https://github.com/docker/compose/releases)

## 下载

运行此命令下载最新版本的Docker Compose：

```sh
sudo curl -L https://github.com/docker/compose/releases/download/1.16.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

## 授权

对二进制文件应用可执行权限：

```sh
sudo chmod +x /usr/local/bin/docker-compose
```

## 验证

```sh
$ docker-compose --version
docker-compose version 1.16.1, build 6d1ac21
```

## 卸载

要卸载 Docker Compose，如果使用 curl 以下安装：
 
```sh
sudo rm /usr/local/bin/docker-compose
```


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
   
   
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")




