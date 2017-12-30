---
layout: post
title: Ubuntu 17.04 x64 安装 Docker CE 初窥 Dockerfile 部署 Nginx
categories: Docker
description: Ubuntu 17.04 x64 安装 Docker CE 初窥 Dockerfile 部署 Nginx
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

1.更新`apt`软件包索引：

```sh
$ apt-get update
```

2.安装软件包以允许`apt`通过`HTTPS`使用存储库：

```sh
$ apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
```

3.添加`Docker`的官方`GPG`密钥：

```sh
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

`9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`通过搜索指纹的最后8个字符，确认您现在拥有指纹的密钥 。

```sh
$ sudo apt-key fingerprint 0EBFCD88
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```

# 安装Docker CE

1.更新apt软件包索引。

```sh
$ apt-get update
```

2.安装最新版本的Docker CE，或者转到下一步安装特定版本。任何现有的Docker安装都将被替换。

```sh
$ apt-get install docker-ce
```

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

1.创建docker组。

```sh
$ sudo groupadd docker
```
2.将您的用户添加到docker组中。

```sh
$ sudo usermod -aG docker $USER
```

3.注销并重新登录,如果在虚拟机上进行测试，则可能需要重新启动虚拟机才能使更改生效。

4.验证您可以不运行`docker`命令`sudo`。

```sh
$ docker run hello-world
```

# 卸载Docker CE

1.卸载`Docker CE`软件包：

```sh
$ sudo apt-get purge docker-ce
```

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

现在，假设我们非常不喜欢这个欢迎页面，我们希望改成欢迎 `Docker` 的文字，我们可以使用 `docker exec` 命令进入容器，修改其内容。

```sh
$ docker exec -it myweb bash
root@5ceb0c8274ca:/# echo '<h1>Welcome to Docker!</h1>' > /usr/share/nginx/html/index.html
root@5ceb0c8274ca:/# exit
exit
```

直接访问：[http://localhost](http://localhost)；

![ Welcome to Docker!][2]

我们修改了容器的文件，也就是改动了容器的存储层。我们可以通过 `docker diff` 命令看到具体的改动。`COMMENT`列有备注！

```sh
$  docker diff myweb
C /root
A /root/.bash_history
C /run
A /run/nginx.pid
C /usr/share/nginx/html/index.html
C /var/cache/nginx
D /var/cache/nginx/client_temp
D /var/cache/nginx/fastcgi_temp
D /var/cache/nginx/proxy_temp
D /var/cache/nginx/scgi_temp
D /var/cache/nginx/uwsgi_temp
root@souyunku:~/mydocker#
```

**我们可以用下面的命令将容器保存为镜像：**

```sh
$ docker commit \
    --author "penglei <admin@souyunku.com>" \
    --message "修改了默认网页" \
    myweb \
    nginx:v2
	
sha256:33b2e2aefccbaba54021c85ef7966c7d488abaa0677728e1b057c56a7734a4f0
```

其中 `--author` 是指定修改的作者，而 `--message` 则是记录本次修改的内容。这点和 `git` 版本控制相似，不过这里这些信息可以省略留空。

我们可以在 `docker image ls` 中看到这个新定制的镜像：

```sh
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED              SIZE
nginx               v2                  33b2e2aefccb        About a minute ago   108MB
nginx               latest              3f8a4339aadd        3 days ago           108MB
hello-world         latest              f2a91732366c        5 weeks ago          1.85kB
```

```sh
$ docker history nginx:v2
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
33b2e2aefccb        2 minutes ago       nginx -g daemon off;                            325B                修改了默认网页
3f8a4339aadd        3 days ago          /bin/sh -c #(nop)  CMD ["nginx" "-g" "daemon…   0B                  
<missing>           3 days ago          /bin/sh -c #(nop)  STOPSIGNAL [SIGTERM]         0B                  
<missing>           3 days ago          /bin/sh -c #(nop)  EXPOSE 80/tcp                0B                  
<missing>           3 days ago          /bin/sh -c ln -sf /dev/stdout /var/log/nginx…   22B                 
<missing>           3 days ago          /bin/sh -c set -x  && apt-get update  && apt…   53.2MB              
<missing>           3 days ago          /bin/sh -c #(nop)  ENV NJS_VERSION=1.13.8.0.…   0B                  
<missing>           3 days ago          /bin/sh -c #(nop)  ENV NGINX_VERSION=1.13.8-…   0B                  
<missing>           2 weeks ago         /bin/sh -c #(nop)  LABEL maintainer=NGINX Do…   0B                  
<missing>           2 weeks ago         /bin/sh -c #(nop)  CMD ["bash"]                 0B                  
<missing>           2 weeks ago         /bin/sh -c #(nop) ADD file:f30a8b5b7cdc9ba33…   55.3MB 
```

新的镜像定制好后，我们可以来运行这个镜像。

```sh
$ docker run --name web2 -d -p 81:80 nginx:v2
ed8c54aeb3c540981b892c0cdfbf9330114ecc935149190e073f49295f2ae147
```

直接访问：[http://localhost:81](http://localhost:81)；如果使用的是 Docker Toolbox，或者是在虚拟机、云服务器上安装的 Docker，则需要将 localhost 换为虚拟机地址或者实际云服务器地址。

![ Welcome to Docker!][3]

# 使用 Dockerfile 定制镜像

从刚才的 `docker commit` 的学习中，我们可以了解到，镜像的定制实际上就是定制每一层所添加的配置、文件。如果我们可以把每一层修改、安装、构建、操作的命令都写入一个脚本，用这个脚本来构建、定制镜像，那么之前提及的无法重复的问题、镜像构建透明性的问题、体积的问题就都会解决。这个脚本就是 `Dockerfile`。

`Dockerfile` 是一个文本文件，其内包含了一条条的指令(`Instruction`)，每一条指令构建一层，因此每一条指令的内容，就是描述该层应当如何构建。

还以之前定制 `nginx` 镜像为例，这次我们使用 `Dockerfile` 来定制。

在一个空白目录中，建立一个文本文件，并命名为 `Dockerfile`：


```sh
$ mkdir mynginx
$ cd mynginx
$ touch Dockerfile
```

其内容为：

```sh
FROM nginx
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```

这个 `Dockerfile` 很简单，一共就两行。涉及到了两条指令，`FROM` 和 `RUN`。

# FROM 指定基础镜像

所谓定制镜像，那一定是以一个镜像为基础，在其上进行定制。就像我们之前运行了一个 `nginx` 镜像的容器，再进行修改一样，基础镜像是必须指定的。而 FROM 就是指定基础镜像，因此一个 `Dockerfile` 中 `FROM` 是必备的指令，并且必须是第一条指令。

在 `Docker Store` 上有非常多的高质量的官方镜像，有可以直接拿来使用的服务类的镜像，如 `nginx`、`redis`、`mongo`、`mysql`、`httpd`、`php`、`tomcat` 等；也有一些方便开发、构建、运行各种语言应用的镜像，如 `node`、`openjdk`、`python`、`ruby`、`golang` 等。可以在其中寻找一个最符合我们最终目标的镜像为基础镜像进行定制。

如果没有找到对应服务的镜像，官方镜像中还提供了一些更为基础的操作系统镜像，如 `ubuntu`、`debian`、`centos`、`fedora`、`alpine` 等，这些操作系统的软件库为我们提供了更广阔的扩展空间。

除了选择现有镜像为基础镜像外，`Docker` 还存在一个特殊的镜像，名为 `scratch`。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。

```sh
FROM scratch
...
```


如果你以 `scratch` 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。


不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如 `swarm`、`coreos/etcd`。对于 `Linux` 下静态编译的程序来说，并不需要有操作系统提供运行时支持，所需的一切库都已经在可执行文件里了，因此直接 `FROM` `scratch` 会让镜像体积更加小巧。使用 `Go` 语言 开发的应用很多会使用这种方式来制作镜像，这也是为什么有人认为 Go 是特别适合容器微服务架构的语言的原因之一。

## RUN 执行命令

`RUN` 指令是用来执行命令行命令的。由于命令行的强大能力，`RUN` 指令在定制镜像时是最常用的指令之一。其格式有两种：

 - `shell` 格式：`RUN` <命令>，就像直接在命令行中输入的命令一样。刚才写的 `Dockerfile` 中的 `RUN` 指令就是这种格式。

```sh
RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
```
 - exec 格式：RUN ["可执行文件", "参数1", "参数2"]，这更像是函数调用中的格式。
 
既然 `RUN` 就像 `Shell` 脚本一样可以执行命令，那么我们是否就可以像 `Shell` 脚本一样把每个命令对应一个 `RUN` 呢？比如这样：

```sh
FROM debian:jessie

RUN apt-get update
RUN apt-get install -y gcc libc6-dev make
RUN wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz"
RUN mkdir -p /usr/src/redis
RUN tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1
RUN make -C /usr/src/redis
RUN make -C /usr/src/redis install
```

之前说过，`Dockerfile` 中每一个指令都会建立一层，`RUN` 也不例外。每一个 `RUN `的行为，就和刚才我们手工建立镜像的过程一样：新建立一层，在其上执行这些命令，执行结束后，commit 这一层的修改，构成新的镜像。

而上面的这种写法，创建了 7 层镜像。这是完全没有意义的，而且很多运行时不需要的东西，都被装进了镜像里，比如编译环境、更新的软件包等等。结果就是产生非常臃肿、非常多层的镜像，不仅仅增加了构建部署的时间，也很容易出错。 这是很多初学 Docker 的人常犯的一个错误。

`Union FS` 是有最大层数限制的，比如 `AUFS`，曾经是最大不得超过 42 层，现在是不得超过 127 层。

上面的 `Dockerfile` 正确的写法应该是这样：


```sh
FROM debian:jessie

RUN buildDeps='gcc libc6-dev make' \
    && apt-get update \
    && apt-get install -y $buildDeps \
    && wget -O redis.tar.gz "http://download.redis.io/releases/redis-3.2.5.tar.gz" \
    && mkdir -p /usr/src/redis \
    && tar -xzf redis.tar.gz -C /usr/src/redis --strip-components=1 \
    && make -C /usr/src/redis \
    && make -C /usr/src/redis install \
    && rm -rf /var/lib/apt/lists/* \
    && rm redis.tar.gz \
    && rm -r /usr/src/redis \
    && apt-get purge -y --auto-remove $buildDeps
```

首先，之前所有的命令只有一个目的，就是编译、安装 `redis` 可执行文件。因此没有必要建立很多层，这只是一层的事情。因此，这里没有使用很多个 `RUN` 对一一对应不同的命令，而是仅仅使用一个 `RUN` 指令，并使用 && 将各个所需命令串联起来。将之前的 7 层，简化为了 1 层。在撰写 `Dockerfile` 的时候，要经常提醒自己，这并不是在写 `Shell` 脚本，而是在定义每一层该如何构建。

并且，这里为了格式化还进行了换行。`Dockerfile` 支持 `Shell` 类的行尾添加 \ 的命令换行方式，以及行首 # 进行注释的格式。良好的格式，比如换行、缩进、注释等，会让维护、排障更为容易，这是一个比较好的习惯。

此外，还可以看到这一组命令的最后添加了清理工作的命令，删除了为了编译构建所需要的软件，清理了所有下载、展开的文件，并且还清理了 `apt` 缓存文件。这是很重要的一步，我们之前说过，镜像是多层存储，每一层的东西并不会在下一层被删除，会一直跟随着镜像。因此镜像构建时，一定要确保每一层只添加真正需要添加的东西，任何无关的东西都应该清理掉。

很多人初学 `Docker` 制作出了很臃肿的镜像的原因之一，就是忘记了每一层构建的最后一定要清理掉无关文件。

## 构建镜像

好了，让我们再回到之前定制的 `nginx` 镜像的 `Dockerfile` 来。现在我们明白了这个 `Dockerfile` 的内容，那么让我们来构建这个镜像吧。

在 `Dockerfile` 文件所在目录执行：

```sh

$ docker build -t nginx:v3 .

Sending build context to Docker daemon  2.048kB
Step 1/2 : FROM nginx
 ---> 3f8a4339aadd
Step 2/2 : RUN echo '<h1>Hello, Docker!</h1>' > /usr/share/nginx/html/index.html
 ---> Running in 5cdac6d29a52
Removing intermediate container 5cdac6d29a52
 ---> 56a1b67b2533
Successfully built 56a1b67b2533
Successfully tagged nginx:v3
```

从命令的输出结果中，我们可以清晰的看到镜像的构建过程。在 `Step 2` 中，如同我们之前所说的那样，`RUN` 指令启动了一个容器 `5cdac6d29a52`，执行了所要求的命令，并最后提交了这一层 `56a1b67b2533`，随后删除了所用到的这个容器 `5cdac6d29a52`

这里我们使用了 `docker build` 命令进行镜像构建。其格式为：

```sh
$ docker build [选项] <上下文路径/URL/->
```

**镜像构建上下文（Context）**如果注意，会看到 `docker build` 命令最后有一个 `.`。`.` 表示当前目录

## 其它 docker build 的用法

## 直接用 `Git repo` 进行构建

或许你已经注意到了，`docker build` 还支持从 `URL` 构建，比如可以直接从 `Git repo` 中构建：

**感谢：漠然提供 Git dockerfile repo**

```sh
$ docker build https://github.com/mritd/dockerfile.git#:alpine-glibc
```

```sh
...

Executing ca-certificates-20171114-r0.post-deinstall
Executing busybox-1.27.2-r6.trigger
OK: 11 MiB in 14 packages
Removing intermediate container baf41e622959
 ---> aded3329be1b
Step 3/3 : ENV LANG=C.UTF-8
 ---> Running in 8c34e0f4d25b
Removing intermediate container 8c34e0f4d25b
 ---> c493a5aa5eb7
Successfully built c493a5aa5eb7
```

这行命令指定了构建所需的 `Git repo`，并且指定默认的 `master` 分支，构建目录为 /alpine-glibc/，然后 `Docker` 就会自己去 `git clone` 这个项目、切换到指定分支、并进入到指定目录后开始构建。


## 用给定的 tar 压缩包构建

```sh
$ docker build http://server/context.tar.gz
```
如果所给出的 URL 不是个 `Git repo`，而是个 tar 压缩包，那么 Docker 引擎会下载这个包，并自动解压缩，以其作为上下文，开始构建。

## 从标准输入中读取 `Dockerfile` 进行构建

```sh
$ docker build - < Dockerfile
```

或

```sh
$ cat Dockerfile | docker build -
```

如果标准输入传入的是文本文件，则将其视为 `Dockerfile`，并开始构建。这种形式由于直接从标准输入中读取 `Dockerfile` 的内容，它没有上下文，因此不可以像其他方法那样可以将本地文件 COPY 进镜像之类的事情。

## 从标准输入中读取上下文压缩包进行构建

```sh
$ docker build - < context.tar.gz
```

如果发现标准输入的文件格式是 `gzip、bzip2` 以及` xz` 的话，将会使其为上下文压缩包，直接将其展开，将里面视为上下文，并开始构建。

参考：Docker — 从入门到实践

[https://www.gitbook.com/download/pdf/book/yeasy/docker_practice](https://www.gitbook.com/download/pdf/book/yeasy/docker_practice)

参考：Docker 官网 Get Docker CE for Ubuntu

[https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-using-the-convenience-script](https://docs.docker.com/engine/installation/linux/docker-ce/ubuntu/#install-using-the-convenience-script)

[1]: http://www.ymq.io/images/2017/docker/1.png
[2]: http://www.ymq.io/images/2017/docker/2.png
[3]: http://www.ymq.io/images/2017/docker/3.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

