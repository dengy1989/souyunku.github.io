---
layout: post
title: 可能是最详细的部署：Docker Registry企业级私有镜像仓库Harbor管理WEB UI
categories: Docker
description: 可能是最详细的部署：Docker Registry企业级私有镜像仓库Harbor管理WEB UI
keywords: Docker
---

上一篇文章搭建了一个具有基础功能，权限认证、`TLS` 的私有仓库，但是`Docker Registry` 作为镜像仓库，连管理界面都没有，甚至连一些运维必备的功能都是缺失的，还有什么 `Docker` 镜像仓库管理工具呢？
这里有一个简单好用的企业级 `Registry` 服务器 - `Harbor`，推荐在生产环境上使用。

# Harbor 简介

`Harbor`是`VMware`公司开源的企业级`Docker Registry`项目，其目标是帮助用户迅速搭建一个企业级的`Docker registry`服务。

它以`Docker`公司开源的`registry`为基础，提供了管理`UI`，基于角色的访问控制(`Role Based AccessControl`)，`AD/LDAP`集成、以及审计日志(`Auditlogging`) 等企业用户需求的功能，通过添加一些企业必需的功能特性，例如安全、标识和管理等，扩展了开源 `Docker Distribution`。  


作为一个企业级私有 `Registry` 服务器，`Harbor` 提供了更好的性能和安全。提升用户使用 `Registry` 构建和运行环境传输镜像的效率。  


`Harbor` 支持安装在多个 `Registry` 节点的镜像资源复制，镜像全部保存在私有 `Registry` 中，确保数据和知识产权在公司内部网络中管控。另外，`Harbor` 也提供了高级的安全特性，诸如用户管理，访问控制和活动审计等。


`Harbor` 是由 `VMware` 中国研发团队负责开发的开源企业级 `Docker Registry`，不仅解决了我们直接使用 `Docker Registry` 的功能缺失，更解决了我们在生产使用 `Docker Registry` 面临的高可用、镜像仓库直接复制、镜像仓库性能等运维痛点。

# 环境准备

 - 系统：Ubuntu 17.04 x64  
 - Docker 17.12.0-ce ,Docker Compose
 - python3
 - IP:198.13.48.154  
 - 域名：hub.ymq.io，此域名需要dns 解析到198.13.48.154 作为私有仓库地址  
 
**本文出现的所有：`hub.ymq.io` 域名。使用时候请替换成自己的域名**
 
# Docker 环境

在部署私有仓库之前，需要在主机上安装`Docker`。私有仓库是 `registry images`，并在`Docker`中运行。

我是用的`vultr` 的服务器，所以，下面操作，就不用配置国内的，加速镜像库，直接用`Docker`官方的！

国内加速仓库，我其他文章有提到:Ubuntu 17.04 x64 安装 Docker CE 初窥 Dockerfile 部署 Nginx  
[http://www.ymq.io/2017/12/30/Docker-Install/](http://www.ymq.io/2017/12/30/Docker-Install/)  

## 安装Docker CE

**使用存储库进行安装**

1.更新`apt`软件包索引：

```sh
$ sudo apt-get update
```

2.装软件包以允许`apt`通过`HTTPS`使用存储库：

```sh
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

3.添加`Docker`的官方`GPG`密钥：

```sh
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

4.使用以下命令来设置稳定的存储库

```sh
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

5.更新`apt`软件包索引。

```sh
$ sudo apt-get update
```

7.安装最新版本的`Docker CE`

```sh
$ sudo apt-get install docker-ce
```

8.通过运行`hello-world` 映像验证是否正确安装了`Docker CE` 。

```sh
$ sudo docker run hello-world
```

# Docker Compose

## 安装 Compose

在`Linux`上，您可以从`GitHub`上的`Compose`存储库版本页面下载`Docker Compose`二进制文件。按照链接中的说明进行操作，即`curl`在终端中运行命令以下载二进制文件。这些一步一步的说明也包括在下面。

`GitHub`上的`Compose`存储库版本页面下载地址：[https://github.com/docker/compose/releases](https://github.com/docker/compose/releases)

**1.运行此命令下载最新版本的`Docker Compose`：**

```sh
sudo curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

**2.对二进制文件应用可执行权限：**

```sh
sudo chmod +x /usr/local/bin/docker-compose
```

## 测试安装

```sh
$ docker-compose --version
docker-compose version 1.18.0, build 8dd22a9
```

# Python 环境

## 安装 Python
```sh
apt-get install python3
apt-get install python-minimal
apt-get install python3-setuptools
easy_install3 pip
apt-get install python-argparse
```

## 测试安装

```sh
$ python --version
Python 2.7.13

$ pip -V
pip 9.0.1 from /usr/local/lib/python3.5/dist-packages/pip-9.0.1-py3.5.egg (python 3.5)
```

# 域名证书

[acme.sh 实现了 acme 协议, 可以从 letsencrypt 生成免费的证书. https://github.com/Neilpang/acme.sh](https://github.com/Neilpang/acme.sh)

[给acme.sh组织赞助：Acknowledgments](https://donate.acme.sh/)

很简单就两个步骤:

 - 安装 `acme.sh`
 - 生成证书，及验证证书

## 安装 acme.sh
 
安装很简单, 一个命令:
 
```sh
$ curl  https://get.acme.sh | sh
```

这条命令，会做的事情

1.把 `acme.sh` 安装到你的 `home` 目录下：
并创建 一个 `bash` 的 `alias`, 方便你的使用: `acme.sh=~/.acme.sh/acme.sh`

2.自动为你创建 `cronjob`, 每天 `0:00` 点自动检测所有的证书, 如果快过期了, 需要更新, 则会自动更新证书.

## 生成证书

如果你还没有运行任何 `web` 服务, 且`80` 端口是空闲的, 那么 `acme.sh` 能假装自己是一个`webserver`, 临时听在`80` 端口, 完成验证:

**注意：**如果您使用的时候，请把，`hub.ymq.io` 替换成自己域名，此域名需要`dns` 解析到安装私有仓库的服务器`IP` 

```sh
$ cd ~/.acme.sh/
$ apt-get install socat
$ sh acme.sh  --issue -d hub.ymq.io   --standalone
```

如果看到如下信息，说明证书验证并生成成功,证书生成位置在：`/root/.acme.sh/hub.ymq.io/` 下

```sh
Success
Verify finished, start to sign.
Cert success.
-----BEGIN CERTIFICATE-----
```

```sh
[Wed Jan  3 14:36:25 UTC 2018] Standalone mode.
[Wed Jan  3 14:36:25 UTC 2018] Registering account
[Wed Jan  3 14:36:27 UTC 2018] Registered
[Wed Jan  3 14:36:27 UTC 2018] ACCOUNT_THUMBPRINT='7TpUIE5N--hq2nhk2ruKmHBfgKB-LX-pBCkWzzmHzVM'
[Wed Jan  3 14:36:27 UTC 2018] Creating domain key
[Wed Jan  3 14:36:28 UTC 2018] The domain key is here: /root/.acme.sh/hub.ymq.io/hub.ymq.io.key
[Wed Jan  3 14:36:28 UTC 2018] Single domain='hub.ymq.io'
[Wed Jan  3 14:36:28 UTC 2018] Getting domain auth token for each domain
[Wed Jan  3 14:36:28 UTC 2018] Getting webroot for domain='hub.ymq.io'
[Wed Jan  3 14:36:28 UTC 2018] Getting new-authz for domain='hub.ymq.io'
[Wed Jan  3 14:36:29 UTC 2018] The new-authz request is ok.
[Wed Jan  3 14:36:29 UTC 2018] Verifying:hub.ymq.io
[Wed Jan  3 14:36:29 UTC 2018] Standalone mode server
[Wed Jan  3 14:36:34 UTC 2018] Success
[Wed Jan  3 14:36:34 UTC 2018] Verify finished, start to sign.
[Wed Jan  3 14:36:35 UTC 2018] Cert success.
-----BEGIN CERTIFICATE-----
MIIE9zCCA9+gAwIBAgISA6WV4ZFi6lr/kngVGx7/FoPMMA0GCSqGSIb3DQEBCwUA
******************************************
...

-----END CERTIFICATE-----
[Wed Jan  3 14:36:35 UTC 2018] Your cert is in  /root/.acme.sh/hub.ymq.io/hub.ymq.io.cer 
[Wed Jan  3 14:36:35 UTC 2018] Your cert key is in  /root/.acme.sh/hub.ymq.io/hub.ymq.io.key 
[Wed Jan  3 14:36:35 UTC 2018] The intermediate CA cert is in  /root/.acme.sh/hub.ymq.io/ca.cer 
[Wed Jan  3 14:36:35 UTC 2018] And the full chain certs is there:  /root/.acme.sh/hub.ymq.io/fullchain.cer 
```

# Harbor 仓库

前提条件：域名的`dns` 解析到安装私有仓库的服务器`IP`上

## 复制证书

1.创建一个`certs`目录。

```sh
$ cd /opt/
$ mkdir -p certs
```

2.移动证书到`certs`目录。

```sh
$ cd ~/.acme.sh/
$ sh acme.sh  --installcert  -d  hub.ymq.io   \
        --key-file   /opt/certs/hub.ymq.io.key \
        --fullchain-file /opt/certs/fullchain.cer
```

## Harbor 下载

下载Harbour版本的二进制文件 [https://github.com/vmware/harbor/releases](https://github.com/vmware/harbor/releases)

目前最新版本 `V1.3.0`

```sh
$ wget https://storage.googleapis.com/harbor-releases/harbor-online-installer-v1.3.0.tgz
$ tar -zxvf harbor-offline-installer-v1.3.0-rc4.tgz 
```
## Harbor 配置

```sh
$ cd harbor
$ vim harbor.cfg
```

只需修改如下内容

```sh
hostname = hub.ymq.io
ui_url_protocol = https
customize_crt = off
ssl_cert = /opt/certs/fullchain.cer
ssl_cert_key = /opt/certs/hub.ymq.io.key
```

参数解释

```sh
hostname = 主机名：目标主机的主机名，用于访问UI和注册表服务。它应该是目标机器的IP地址或完全限定的域名（FQDN），例如198.13.48.154或 `hub.ymq.io`。不要使用localhost或127.0.0.1为主机名 - 注册表服务需要由外部客户端访问！
ui_url_protocol = （http或https，默认为http）用于访问UI和令牌/通知服务的协议。如果公证处于启用状态，则此参数必须为https。默认情况下，这是http。
customize_crt = （打开或关闭，默认打开）打开此属性时，准备脚本创建私钥和根证书，用于生成/验证注册表令牌。当由外部来源提供密钥和根证书时，将此属性设置为off
ssl_cert =SSL证书的路径，仅当协议设置为https时才应用
ssl_cert_key = SSL密钥的路径，仅当协议设置为https时才应用
```

## 默认安装

```sh
$ sudo ./install.sh
```

## Harbor 登录

如果一切正常，你应该可以打开浏览器访问`http://hub.ymq.io`的管理门户（将`hub.ymq.io`更改为在你的配置中的主机名`harbor.cfg`）。请注意，默认的管理员用户名/密码是`admin / Harbor12345`。

登录管理员门户并创建一个新项目，例如`myproject`。然后，您可以使用docker命令来登录和推送图像（默认情况下，注册表服务器在端口`80`上侦听）：  

![ https://hub.ymq.io/harbor][1]

![ 新项目 myproject ][2]

![ 查看项目列表 myproject][3]

# 测试服务

## 登录仓库

**Username:**`admin`   
**Password:**`Harbor12345`  

```sh
$ docker login hub.ymq.io
Username (testuser): admin
Password: 输入仓库密码
Login Succeeded
```

## 拉取镜像

从 `Docker Hub`拉取 `ubuntu:16.04` 镜像

```sh
$ docker pull ubuntu:16.04
```

## 标记镜像


将镜像标记为 `hub.ymq.io/myproject`，在推送时，`Docker`会将其解释为仓库的位置。

```sh
$ docker tag  ubuntu:16.04 hub.ymq.io/myproject/my-ubuntu
```

## 推送镜像

将镜像推送到本地镜像标记的仓库`hub.ymq.io/myproject/`

```sh
$ docker push hub.ymq.io/myproject/my-ubuntu
```

![ 推送镜像列表 myproject][4]

**在镜像列表：可以删除，复制，查看日志，及其他操作** 


## 删除镜像

删除本地缓存`ubuntu:16.04`和`hub.ymq.io/myproject/my-ubuntu` 镜像，以便您可以测试从私有仓库中拉取镜像。这不会`hub.ymq.io/myproject/my-ubuntu` 从您的私有仓库中删除镜像。

```sh
$ docker image remove ubuntu:16.04
$ docker image remove hub.ymq.io/myproject/my-ubuntu
```
## 拉取镜像

拉取 `hub.ymq.io` 仓库的 `/myproject/my-ubuntu` 镜像。

```sh
root@souyunku:~# docker pull hub.ymq.io/myproject/my-ubuntu
Using default tag: latest
latest: Pulling from myproject/my-ubuntu
50aff78429b1: Pull complete 
f6d82e297bce: Pull complete 
275abb2c8a6f: Pull complete 
9f15a39356d6: Pull complete 
fc0342a94c89: Pull complete 
Digest: sha256:f871d0805ee3ce1c52b0608108dbdf1b447a34d22d5c7278a3a9dd78fc12c663
Status: Downloaded newer image for hub.ymq.io/myproject/my-ubuntu:latest
```

## 查看镜像

```sh
root@souyunku:~# docker images hub.ymq.io/myproject/my-ubuntu
REPOSITORY                       TAG                 IMAGE ID            CREATED             SIZE
hub.ymq.io/myproject/my-ubuntu   latest              00fd29ccc6f1        3 weeks ago         111MB
```

## 操作日志

![ 镜像操作日志 myproject][5]


官方文档:
[https://github.com/vmware/harbor/blob/master/docs/installation_guide.md](https://github.com/vmware/harbor/blob/master/docs/installation_guide.md)

# Docker Compose

`Docker Compose` 是 `Docker` 官方编排（`Orchestration`）项目之一，负责快速在集群中部署分布式应用。

一个使用`Docker`容器的应用，通常由多个容器组成。使用`Docker Compose`，不再需要使用`shell`脚本来启动容器。在配置文件中，所有的容器通过`services`来定义，然后使用`docker-compose`脚本来启动，停止和重启应用，和应用中的服务以及所有依赖服务的容器

**Docker Compose 的搭建，及使用，发布 `spring boot`整合`redis`做访问计数`demo`，实战 `WordPress`，正在整理中，会在下篇文章体现，关注公众号：“搜云库” 我会在微信公众号首发**

[1]: http://www.ymq.io/images/2018/docker/harbor-https/1.png
[2]: http://www.ymq.io/images/2018/docker/harbor-https/2.png
[3]: http://www.ymq.io/images/2018/docker/harbor-https/3.png
[4]: http://www.ymq.io/images/2018/docker/harbor-https/4.png
[5]: http://www.ymq.io/images/2018/docker/harbor-https/5.png
[6]: http://www.ymq.io/images/2018/docker/harbor-https/6.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/01/06/Docker-harbor-registry/](http://www.ymq.io/2018/01/06/Docker-harbor-registry/)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，"搜云库"，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

我的博客即将搬运同步至腾讯云+社区，邀请大家一同入驻：[https://cloud.tencent.com/developer/support-plan](https://cloud.tencent.com/developer/support-plan)
