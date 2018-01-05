---
layout: post
title: Docker Registry Server 搭建,配置免费HTTPS证书，及拥有权限认证、TLS 的私有仓库
categories: Docker
description: Docker Registry Server 搭建,配置免费HTTPS证书，及拥有权限认证、TLS 的私有仓库
keywords: Docker
---

上一篇文章搭建了一个具有基础功能的私有仓库，这次来搭建一个拥有权限认证、`TLS` 的私有仓库。

# 环境准备

 - 系统：Ubuntu 17.04 x64  
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

# 域名证书

[acme.sh 实现了 acme 协议, 可以从 letsencrypt 生成免费的证书. https://github.com/Neilpang/acme.sh](https://github.com/Neilpang/acme.sh)

[给acme.sh组织赞助：Acknowledgments](https://donate.acme.sh/)

很简单就两个步骤:

 - 安装 `acme.sh`
 - 生成证书，及验证证书

## 1.安装 acme.sh
 
安装很简单, 一个命令:
 
```sh
$ curl  https://get.acme.sh | sh
```

这条命令，会做的事情

1.把 `acme.sh` 安装到你的 `home` 目录下：
并创建 一个 `bash` 的 `alias`, 方便你的使用: `acme.sh=~/.acme.sh/acme.sh`

2.自动为你创建 `cronjob`, 每天 `0:00` 点自动检测所有的证书, 如果快过期了, 需要更新, 则会自动更新证书.

## 2.生成证书

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


# 搭建仓库

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

## 身份验证

为用户创建一个带有一个条目的密码文件`testuser`，密码为 `testpassword`：

```sh
$ mkdir auth
$ docker run \
  --entrypoint htpasswd \
  registry:2 -Bbn testuser testpassword > auth/htpasswd
```

## 创建仓库

启动注册表，指示它使用`TLS`证书。这个命令将`certs/`目录绑定到容器中`/certs/`，并设置环境变量来告诉容器在哪里找到`fullchain.cer` 和`hub.ymq.io.key`文件。注册表在端口`443`（默认的`HTTPS`端口）上运行。

```sh
docker run -d \
  --restart=always \
  --name registry \
  -v `pwd`/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v `pwd`/certs:/certs \
  -e REGISTRY_HTTP_ADDR=0.0.0.0:443 \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/fullchain.cer \
  -e REGISTRY_HTTP_TLS_KEY=/certs/hub.ymq.io.key \
  -p 443:443 \
  registry:2
```

查看日志

```sh
$ docker logs -f registry
```

## 登录仓库

```sh
$ docker login hub.ymq.io
Username (testuser): testuser
Password: 输入仓库密码
Login Succeeded
```

## 拉取镜像

从 `Docker Hub`拉取 `ubuntu:16.04` 镜像

```sh
$ docker pull ubuntu:16.04
```

## 标记镜像

将镜像标记为 `hub.ymq.io/my-ubuntu`，在推送时，`Docker`会将其解释为仓库的位置。

```sh
$ docker tag ubuntu:16.04 hub.ymq.io/my-ubuntu
```

## 推送镜像

将镜像推送到本地镜像标记的仓库`hub.ymq.io/my-ubuntu`

```sh
$ docker push hub.ymq.io/my-ubuntu
```

## 删除镜像

删除本地缓存`ubuntu:16.04`和`hub.ymq.io/my-ubuntu` 镜像，以便您可以测试从私有仓库中拉取镜像。这不会`hub.ymq.io/my-ubuntu` 从您的私有仓库中删除镜像。

```sh
$ docker image remove ubuntu:16.04
$ docker image remove hub.ymq.io/my-ubuntu
```
## 拉取镜像

拉取 `hub.ymq.io` 仓库的 `my-ubuntu` 镜像。

```sh
$ docker pull hub.ymq.io/my-ubuntu
```

## 查看镜像

```sh
$ docker images hub.ymq.io/my-ubuntu
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
hub.ymq.io/my-ubuntu   latest              00fd29ccc6f1        2 weeks ago         111MB
```

在浏览器中查看仓库中的镜像。需要输入账号密码

![ hub.ymq.io ][1]
![ hub.ymq.io ][2]

## 操作API

Docker Registry HTTP API V2 

仓库操作 API 官方文档：[https://docs.docker.com/registry/spec/api/](https://docs.docker.com/registry/spec/api/)

仓库搭建 官方文档：[https://docs.docker.com/registry/deploying/](https://docs.docker.com/registry/deploying/)

# Harbor

`Harbor`是`VMware`公司开源的企业级`DockerRegistry`项目，项目地址为：[https://github.com/vmware/harbor](https://github.com/vmware/harbor)

其目标是帮助用户迅速搭建一个企业级的`Dockerregistry`服务。它以`Docker`公司开源的`registry`为基础，提供了管理`UI`，基于角色的访问控制(`Role Based Access Control`)，`AD/LDAP`集成、以及审计日志(`Auditlogging`) 等企业用户需求的功能，同时还原生支持中文。`Harbor`的每个组件都是以Docker容器的形式构建的，使用`Docker Compose`来对它进行部署

![ https://hub.ymq.io/harbor/projects][3]

**Harbor 的搭建，及使用，正在整理中，会在下篇文章体现，关注公众号：“搜云库” 我会在微信公众号首发**

[1]: http://www.ymq.io/images/2018/docker/1.png
[2]: http://www.ymq.io/images/2018/docker/2.png
[3]: http://www.ymq.io/images/2018/docker/harbor/1.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

