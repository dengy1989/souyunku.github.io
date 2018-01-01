---
layout: post
title: Docker Hub 仓库使用，及搭建 Docker Registry
categories: Docker
description: Docker Hub 仓库使用
keywords: Docker
---

目前 `Docker` 官方维护了一个公共仓库 [Docker Hub](https://hub.docker.com/)，其中已经包括了数量超过 `15,000` 的镜像。大部分需求都可以通过在 `Docker Hub` 中直接下载镜像来实现。


# Docker Hub

## 注册&&登录

你可以在 [https://cloud.docker.com](https://cloud.docker.com) 免费注册一个 `Docker` 账号。

可以通过执行 `docker login` 命令交互式的输入用户名及密码来完成在命令行界面登录 `Docker Hub`。

你可以通过 `docker logout` 退出登录。

```sh
$ docker login
Login with your Docker ID to push and pull images from Docker Hub. If you don't have a Docker ID, head over to https://hub.docker.com to create one.
Username: souyunku
Password: 输入密码
Login Succeeded
```

## 拉取镜像

你可以通过 `docker search` 命令来查找官方仓库中的镜像，并利用 `docker pull` 命令来将它下载到本地。

例如以 `nginx` 为关键词进行搜索：


```sh
$ docker search nginx
NAME                                                   DESCRIPTION                                     STARS               OFFICIAL            AUTOMATED
nginx                                                  Official build of Nginx.                        7636                [OK]                
jwilder/nginx-proxy                                    Automated Nginx reverse proxy for docker con…   1214                                    [OK]
richarvey/nginx-php-fpm                                Container running Nginx + PHP-FPM capable of…   490                                     [OK]
jrcs/letsencrypt-nginx-proxy-companion                 LetsEncrypt container to use with nginx as p…   279                                     [OK]
kong                                                   Open-source Microservice & API Management la…   143                 [OK]                
webdevops/php-nginx                                    Nginx with PHP-FPM                              93                                      [OK]
kitematic/hello-world-nginx                            A light-weight nginx container that demonstr…   88                                      
```

可以看到返回了很多包含关键字的镜像，其中包括镜像名字、描述、收藏数（表示该镜像的受关注程度）、是否官方创建、是否自动创建。

官方的镜像说明是官方项目组创建和维护的，`automated` 资源允许用户验证镜像的来源和内容。

根据是否是官方提供，可将镜像资源分为两类。

一种是类似 `centos` 这样的镜像，被称为基础镜像或根镜像。这些基础镜像由 `Docker` 公司创建、验证、支持、提供。这样的镜像往往使用单个单词作为名字。

还有一种类型，比如 `jwilder/nginx-proxy` 镜像，它是由 `Docker` 的用户创建并维护的，往往带有用户名称前缀。可以通过前缀 `username/` 来指定使用某个用户提供的镜像，比如 `jwilder` 用户。

另外，在查找的时候通过 `--filter=stars=N` 参数可以指定仅显示收藏数量为 `N` 以上的镜像。

下载官方 `nginx` 镜像到本地。

```sh
$ docker pull nginx
Using default tag: latest
latest: Pulling from library/nginx
e7bb522d92ff: Pull complete 
6edc05228666: Pull complete 
cd866a17e81f: Pull complete 
Digest: sha256:cf8d5726fc897486a4f628d3b93483e3f391a76ea4897de0500ef1f9abcd69a1
Status: Downloaded newer image for nginx:latest
root@souyunku:~/mydocker#
```

## 推送镜像

我们先制作一个镜像

### 先制作一个镜像


创建`Dockerfile`文件

```sh
$ touch Dockerfile
```

`Dockerfile`内容如下

```sh
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

生成镜像

```sh
$ docker build -t nginx:v1 .
Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM nginx
 ---> 3f8a4339aadd
Step 2/2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Using cache
 ---> 4ac2d12f10cd
Successfully built 4ac2d12f10cd
Successfully tagged nginx:v1
```

查看镜像

```sh
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               v1                  4ac2d12f10cd        23 minutes ago      108MB
```

### 推送制作的镜像

用户也可以在登录后通过 `docker push` 命令来将自己的镜像推送到 `Docker Hub。`

以下命令中的 `souyunku` 请替换为你的 `Docker` 账号用户名。


**标记本地镜像，将其归入`souyunku`仓库**

```sh
$ docker tag nginx:v1 souyunku/nginx:v1
```

查看本地镜像

```sh
$ docker images souyunku/nginx:v1
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
souyunku/nginx      v1                  4ac2d12f10cd        41 minutes ago      108MB
```

推送镜像

```sh
$ docker push souyunku/nginx:v1
The push refers to repository [docker.io/souyunku/nginx]
241cbe531d78: Pushed 
a103d141fc98: Pushed 
73e2bd445514: Pushed 
2ec5c0a4cb57: Pushed 
v1: digest: sha256:aae4f5b270340907da80b220315a0e82a15a9debc4347023a4d6c7a96b9cee40 size: 1155
```

### 拉取推送的镜像

先把本地镜像删除

```sh
$ docker rmi souyunku/nginx:v1
Untagged: souyunku/nginx:v1

$ docker rmi e0b
Untagged: nginx:v1
Deleted: sha256:e0bd56806499c0cec4534fe5a85525e45a4d12d8be188d5d498385b0ac36f33e
Deleted: sha256:67d1bbe70151d306c0014d6e3f5c1734ba74849b8989bab46e11f560ae8ec46d

$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              3f8a4339aadd        5 days ago          108MB
```

拉取自己`docker hub`的镜像

```sh
$ docker pull souyunku/nginx:v1
v1: Pulling from souyunku/nginx
e7bb522d92ff: Already exists 
6edc05228666: Already exists 
cd866a17e81f: Already exists 
9c3032d48351: Pull complete 
Digest: sha256:aae4f5b270340907da80b220315a0e82a15a9debc4347023a4d6c7a96b9cee40
Status: Downloaded newer image for souyunku/nginx:v1
```

```sh
$ docker images souyunku/nginx:v1
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
souyunku/nginx      v1                  4ac2d12f10cd        2 hours ago         108MB
```

# 私有仓库

有时候使用 `Docker Hub` 这样的公共仓库可能不方便，用户可以创建一个本地仓库供私人使用。

本节介绍如何使用本地仓库。

`docker-registry` 是官方提供的工具，可以用于构建私有的镜像仓库。本文内容基于 `docker-registry v2.x` 版本。

## 安装运行 `docker-registry`

### 容器运行

你可以通过获取官方 `registry` 镜像来运行。

```sh
$ docker run -d -p 5000:5000 --restart=always --name registry registry
```

```sh
Unable to find image 'registry:latest' locally
latest: Pulling from library/registry
ab7e51e37a18: Pull complete 
c8ad8919ce25: Pull complete 
5808405bc62f: Pull complete 
f6000d7b276c: Pull complete 
f792fdcd8ff6: Pull complete 
Digest: sha256:9d295999d330eba2552f9c78c9f59828af5c9a9c15a3fbd1351df03eaad04c6a
Status: Downloaded newer image for registry:latest
10e12c6983d054da8dc85c017b93e64be0ed11858c0d43b6198bdb652a270d9e
root@souyunku:~/mydocker# docker run -d \
>     -p 5000:5000 \
>     -v /opt/data/registry:/var/lib/registry \
>     registry
469f1bbf2a25f6038795014b0d4bce5035c4c937b86f968a0bff8acd28a78720
docker: Error response from daemon: driver failed programming external connectivity on endpoint flamboyant_yalow (734bddc352cd5804aeafe4c940267954a70109eabd557481e3572adc7cc29e9c): Bind for 0.0.0.0:5000 failed: port is already allocated.
```

这将使用官方的 `registry` 镜像来启动私有仓库。默认情况下，仓库会被创建在容器的 `/var/lib/registry` 目录下。你可以通过 `-v` 参数来将镜像文件存放在本地的指定路径。例如下面的例子将上传的镜像放到本地的 `/opt/data/registry` 目录。

```sh
$ docker run -d \
    -p 5000:5000 \
    -v /opt/data/registry:/var/lib/registry \
    registry
```

## 私有仓库操作


### 查看本地镜像

创建好私有仓库之后，就可以使用 `docker tag` 来标记一个镜像，然后推送它到仓库。例如私有仓库地址为 `127.0.0.1:5000`。

先在本机查看已有的镜像。

```sh
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               latest              3f8a4339aadd        5 days ago          108MB
```

### 标记本地镜像

使用 `docker tag` 将 `nginx:latest` 这个镜像标记为 `127.0.0.1:5000/nginx:latest`。

格式为 docker tag IMAGE[:TAG] [REGISTRY_HOST[:REGISTRY_PORT]/]REPOSITORY[:TAG]。

```sh
$ docker tag nginx:latest 127.0.0.1:5000/nginx:latest
```

### 上传标记镜像

使用 `docker push` 上传标记的镜像,到仓库

```sh
$ docker push 127.0.0.1:5000/nginx:latest
The push refers to repository [127.0.0.1:5000/nginx]
a103d141fc98: Pushed 
73e2bd445514: Pushed 
2ec5c0a4cb57: Pushed 
latest: digest: sha256:926b086e1234b6ae9a11589c4cece66b267890d24d1da388c96dd8795b2ffcfb size: 948
```

```sh
$ docker image ls
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
127.0.0.1:5000/nginx   latest              3f8a4339aadd        5 days ago          108MB
```


用 curl 查看仓库中的镜像。

```sh
$ curl 127.0.0.1:5000/v2/_catalog
{"repositories":["nginx"]}
```

这里可以看到 `{"repositories":["ubuntu"]}`，表明镜像已经被成功上传了。


### 下载仓库镜像

先删除已有镜像，再尝试从私有仓库中下载这个镜像。

```sh
$ docker image rm 127.0.0.1:5000/nginx:latest
Untagged: 127.0.0.1:5000/nginx:latest
Untagged: 127.0.0.1:5000/nginx@sha256:926b086e1234b6ae9a11589c4cece66b267890d24d1da388c96dd8795b2ffcfb
```

下载镜像

```sh
$ docker pull 127.0.0.1:5000/nginx:latest
latest: Pulling from nginx
Digest: sha256:926b086e1234b6ae9a11589c4cece66b267890d24d1da388c96dd8795b2ffcfb
Status: Downloaded newer image for 127.0.0.1:5000/nginx:latest
```

参考：Docker — 从入门到实践

[https://www.gitbook.com/download/pdf/book/yeasy/docker_practice](https://www.gitbook.com/download/pdf/book/yeasy/docker_practice)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

