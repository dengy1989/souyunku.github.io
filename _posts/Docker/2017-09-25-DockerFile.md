---
layout: post
title: 构建 Docker WEB 应用程序  
categories: Docker
description: 构建 Docker WEB 应用程序 
keywords: Docker
---

# Docker 简介

Docker是一个开源的引擎，可以轻松的为任何应用创建一个轻量级的、可移植的、自给自足的容器。开发者在笔记本上编译测试通过的容器可以批量地在生产环境中部署，包括VMs（虚拟机）、bare metal、OpenStack 集群和其他的基础应用平台。 

**Docker通常用于如下场景：**

- web应用的自动化打包和发布；
- 自动化测试和持续集成、发布；
- 在服务型环境中部署和调整数据库或其他的后台应用；
- 从头编译或者扩展现有的OpenShift或Cloud Foundry平台来搭建自己的PaaS环境。


# Dockerfile

Dockerfile是由一系列命令和参数构成的脚本，这些命令应用于基础镜像并最终创建一个新的镜像。它们简化了从头到尾的流程并极大的简化了部署工作。Dockerfile从FROM命令开始，紧接着跟随者各种方法，命令和参数。其产出为一个新的可以用于创建容器的镜像。


# 环境

```sh
centos:7.3  
Docker CE: 17.06.2
Docker Compose: 1.16.1
```


# 安装 Docker

## 下载解压

```sh

```



# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - GitHub：[https://github.com/souyunku](https://github.com/souyunku)  
 - Segment Fault：[https://sf.gg/blog/souyunku](https://sf.gg/blog/souyunku)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

