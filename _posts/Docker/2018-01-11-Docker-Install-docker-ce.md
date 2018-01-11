---
layout: post
title: Ubuntu 17.04 x64 安装 Docker CE
categories: Docker
description: Ubuntu 17.04 x64 安装 Docker CE
keywords: Docker
---

`Docker` 是个划时代的开源项目，它彻底释放了计算虚拟化的威力，极大提高了应用的运行效率，降低了云计算资源供应的成本！使用 `Docker`，可以让应用的部署、测试和分发都变得前所未有的高效和轻松！

无论是应用开发者、运维人员、还是其他信息技术从业人员，都有必要认识和掌握 `Docker`，节约有限的时间。

# 系统要求

要安装`Docker CE`，您需要这些`Ubuntu`版本的64位版本：

- Artful 17.10（Docker CE 17.11 Edge及更高版本）
- ZESTY 17.04
- Xenial 16.04（LTS）
- Trusty 14.04（LTS）

`Ubuntu x86_64，Linux armhf，s390x（IBM Z）和ppc64le（IBM Power）架构上支持Docker CE` 。

# 卸载旧版本

老版本的`Docker`被称为`docker`或`docker-engine`。如果安装了这些，请将其卸载：

```sh
$ apt-get remove docker docker-engine docker.io
```

# 使用存储库进行安装

首次在新的主机上安装`Docker CE`之前，需要设置`Docker`存储库。之后，您可以从存储库安装和更新`Docker`。

- 设置存储库

## 1.更新软件包

1.更新`apt`软件包索引：

```sh
$ apt-get update
```

## 2.设置存储库

2.安装软件包以允许`apt`通过`HTTPS`使用存储库：

```sh
$ apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

## 3.添加`GPG`密钥

3.添加`Docker`的官方`GPG`密钥：

鉴于国内网络问题，强烈建议使用国内源，官方源请在注释中查看。

```sh
$ curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

# 官方源
# $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

## 4.设置镜像源

4.添加 Docker 软件源

鉴于国内网络问题，强烈建议使用国内源，官方源请在注释中查看。


然后，我们需要向 `source.list` 中添加 `Docker` 软件源

```sh
$ sudo add-apt-repository \
    "deb [arch=amd64] https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu \
    $(lsb_release -cs) \
    stable"
	

# 官方源
# $ sudo add-apt-repository \
#    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
#    $(lsb_release -cs) \
#    stable"
```

以上命令会添加稳定版本的 `Docker CE APT` 镜像源，如果需要最新或者测试版本的 `Docker CE` 请将 `stable` 改为 `edge` 或者 `test`。从 `Docker 17.06` 开始，`edge test` 版本的 `APT` 镜像源也会包含稳定版本的 `Docker`。

# 安装Docker CE

## 1.更新软件包

1.更新apt软件包索引。

```sh
$ apt-get update
```

## 2.安装Docker CE

2.安装最新版本的Docker CE，或者转到下一步安装特定版本。任何现有的Docker安装都将被替换。

```sh
$ apt-get install docker-ce
```

## 3.列出版本

3.在生产系统上，您应该安装特定版本的Docker CE，而不是始终使用最新版本。此输出被截断。列出可用的版本。

```sh
$ apt-cache madison docker-ce

docker-ce | 17.12.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu zesty/stable amd64 Packages
docker-ce | 17.09.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu zesty/stable amd64 Packages
docker-ce | 17.09.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu zesty/stable amd64 Packages
docker-ce | 17.06.2~ce-0~ubuntu | https://download.docker.com/linux/ubuntu zesty/stable amd64 Packages
docker-ce | 17.06.1~ce-0~ubuntu | https://download.docker.com/linux/ubuntu zesty/stable amd64 Packages
docker-ce | 17.06.0~ce-0~ubuntu | https://download.docker.com/linux/ubuntu zesty/stable amd64 Packages
```

列表的内容取决于启用了哪个存储库。选择一个特定的版本进行安装。第二列是版本字符串。第三列是存储库名称，它指出了软件包来自哪个存储库，并通过扩展其稳定性级别。要安装特定版本，请将版本字符串附加到包名称，并用等号（=）将它们分开：

```sh
$ sudo apt-get install docker-ce=<VERSION>
```

## 4.运行镜像

4.通过运行`hello-world` 映像验证是否正确安装了`Docker CE` 。

```sh
$ docker run hello-world
```

这个命令下载一个测试图像并在容器中运行。容器运行时，会打印一条信息消息并退出。

```sh
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
ca4f61b1923c: Pull complete 
Digest: sha256:445b2fe9afea8b4aa0b2f27fe49dd6ad130dfe7a8fd0832be5de99625dad47cd
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://cloud.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/engine/userguide/

```

# 以非root用户身份管理Docker

默认情况下，`docker` 命令会使用 `Unix socket` 与 `Docker` 引擎通讯。而只有 `root` 用户和 `docker` 组的用户才可以访问 `Docker` 引擎的 `Unix socket`。出于安全考虑，一般 `Linux` 系统上不会直接使用 `root` 用户。因此，更好地做法是将需要使用 `docker` 的用户加入 `docker` 用户组。

要创建`docker`组并添加您的用户：

## 1.创建用户组

1.创建docker组。

```sh
$ sudo groupadd docker
```

## 2.添加用户组

2.将您的用户添加到docker组中。

```sh
$ sudo usermod -aG docker $USER
```

## 3.注销系统

3.注销并重新登录,如果在虚拟机上进行测试，则可能需要重新启动虚拟机才能使更改生效。

## 4.运行镜像

4.验证您可以不运行`docker`命令`sudo`。

```sh
$ docker run hello-world
```

# 卸载Docker CE

## 1.卸载

1.卸载`Docker CE`软件包：

```sh
$ sudo apt-get purge docker-ce
```

## 2.删除

2.主机上的图像，容器，卷或自定义配置文件不会自动删除。删除所有图像，容器和卷：

```sh
$ sudo rm -rf /var/lib/docker
```

# 利用 commit 理解镜像构成

这条命令会用`nginx` 镜像启动一个容器，命名为 `myweb`，并且映射了 `80` 端口，这样我们可以用浏览器去访问这个 `nginx` 服务器。

```sh
$ docker run --name myweb -d -p 80:80 nginx
```

直接访问：[http://localhost](http://localhost)；如果使用的是 Docker Toolbox，或者是在虚拟机、云服务器上安装的 Docker，则需要将 localhost 换为虚拟机地址或者实际云服务器地址。

![ Welcome to nginx!][1]

参考：Docker 官网 Get Docker CE for Ubuntu

[https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-using-the-convenience-script](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-using-the-convenience-script)

[1]: http://www.ymq.io/images/2017/docker/1.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/01/11/Docker-Install-docker-ce](http://www.ymq.io/2018/01/11/Docker-Install-docker-ce/)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

