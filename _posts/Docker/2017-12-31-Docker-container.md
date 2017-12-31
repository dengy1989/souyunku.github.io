---
layout: post
title: Docker 容器操作
categories: Docker
description: Docker 容器操作
keywords: Docker
---

容器是 Docker 又一核心概念。简单的说，容器是独立运行的一个或一组应用，以及它们的运行态环境。对应的，虚拟机可以理解为模拟运行的一整套操作系统（提供了运行态环境和其他系统环境）和跑在上面的应用。

本章将具体介绍如何来管理一个容器，包括创建、启动和停止等。

# Docker 容器操作

## 启动

### 启动容器

启动容器有两种方式，一种是基于镜像新建一个容器并启动，另外一个是将在终止状态（stopped）的容器重新启动。

因为 `Docker` 的容器实在太轻量级了，很多时候用户都是随时删除和新创建容器。

### 新建并启动

所需要的命令主要为 `docker run`。

例如，下面的命令输出一个 “`Hello World`”，之后终止容器。

```sh
$ docker run ubuntu:14.04 /bin/echo 'Hello world'
Unable to find image 'ubuntu:14.04' locally
14.04: Pulling from library/ubuntu
050aa9ae81a9: Pull complete 
1eb2c989bc04: Pull complete 
f5e83780ccda: Pull complete 
2dec31d7323c: Pull complete 
286f32949bdc: Pull complete 
Digest: sha256:084989eb923bd86dbf7e706d464cf3587274a826b484f75b69468c19f8ae354c
Status: Downloaded newer image for ubuntu:14.04
Hello world
```

这跟在本地直接执行 `/bin/echo 'hello world'` 几乎感觉不出任何区别。

下面的命令则启动一个 `bash` 终端，允许用户进行交互。

```sh
$ docker run -t -i ubuntu:14.04 /bin/bash
root@57eac9f84f5c:/#
```


`-t` 选项让`Docker`分配一个伪终端`（pseudo-tty）`并绑定到容器的标准输入上  
`-i` 则让容器的标准输入保持打开。  

在交互模式下，用户可以通过所创建的终端来输入命令，例如

```sh
root@57eac9f84f5c:/# pwd
/
root@57eac9f84f5c:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@57eac9f84f5c:/#
```

当利用 `docker run` 来创建容器时，`Docker` 在后台运行的标准操作包括：

 - 检查本地是否存在指定的镜像，不存在就从公有仓库下载
 - 利用镜像创建并启动一个容器
 - 分配一个文件系统，并在只读的镜像层外面挂载一层可读写层
 - 从宿主主机配置的网桥接口中桥接一个虚拟接口到容器中去
 - 从地址池配置一个 ip 地址给容器
 - 执行用户指定的应用程序
 - 执行完毕后容器被终止

### 启动已终止容器

可以利用 `docker container start` 命令，直接将一个已经终止的容器启动运行。

**查看终止状态的容器**

```sh
$ docker container ls -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS                NAMES
fcf39bb41624        ubuntu:17.10        "/bin/bash"              About an hour ago   Up 2 minutes                                    objective_wozniak
a9312ab25a6e        ubuntu:17.10        "/bin/sh -c 'while t…"   About an hour ago   Up 40 minutes                                   quizzical_neumann
6e63bcf5e44d        ubuntu:17.10        "/bin/sh -c 'while t…"   2 hours ago         Up 52 seconds                                   brave_sammet
57eac9f84f5c        ubuntu:14.04        "/bin/bash"              2 hours ago         Up 2 seconds                                    frosty_mayer
64835cfb8d6a        ubuntu:14.04        "/bin/echo 'Hello wo…"   2 hours ago         Exited (0) 2 hours ago                          dreamy_raman
5e629833e011        myweb:v1            "/bin/bash"              2 hours ago         Exited (100) 2 hours ago                        amazing_euler
3e3f0c8bb31f        myweb:v1            "nginx -g 'daemon of…"   3 hours ago         Created                                         web
d8ad862e6e0f        nginx               "nginx -g 'daemon of…"   3 hours ago         Up 3 hours                 0.0.0.0:80->80/tcp   myweb
24215366c6ad        hello-world         "/hello"                 3 hours ago         Exited (0) 3 hours ago                          inspiring_keller
```

**启动终止状态的容器 （NAMES） 为 dreamy_raman**

```sh
$ docker container start dreamy_raman
dreamy_raman
```

容器的核心为所执行的应用程序，所需要的资源都是应用程序运行所必需的。除此之外，并没有其它的资源。可以在伪终端中利用 `ps` 或 `top` 来查看进程信息。
 
```sh
$ docker run -t -i ubuntu:14.04 /bin/bash
root@8b8b04dd97cb:/# ps
  PID TTY          TIME CMD
    1 pts/0    00:00:00 bash
   14 pts/0    00:00:00 ps
root@8b8b04dd97cb:/#
root@8b8b04dd97cb:/# exit  
exit
```

可见，容器中仅运行了指定的 `bash` 应用。这种特点使得 `Docker` 对资源的利用率极高，是货真价实的轻量级虚拟化。

## 后台运行

更多的时候，需要让 `Docker` 在后台运行而不是直接把执行命令的结果输出在当前宿主机下。此时，可以通过添加 `-d` 参数来实现。

下面举两个例子来说明一下。

### 不使用 `-d`

如果不使用 `-d` 参数运行容器。

```sh
$ docker run ubuntu:17.10 /bin/sh -c "while true; do echo hello world; sleep 1; done"

Unable to find image 'ubuntu:17.10' locally
17.10: Pulling from library/ubuntu
0bd639347642: Pull complete 
15f827925d02: Pull complete 
8d4e9883d6b5: Pull complete 
c754e879539b: Pull complete 
85f5abd03ce7: Pull complete 
Digest: sha256:01421c4dccafd6d38272e8299f5a23019b7937bea8cc4e7fdfc1bf266a77f369
Status: Downloaded newer image for ubuntu:17.10
hello world
hello world
hello world
hello world
hello world
hello world
hello world
hello world
```

容器会把输出的结果 `(STDOUT)` 打印到宿主机上面

### 使用了 `-d`
 
如果使用了 `-d` 参数运行容器。

```sh
$ docker run -d ubuntu:17.10 /bin/sh -c "while true; do echo hello world; sleep 1; done"
a9312ab25a6e1f5a4d368acfd8126ce476d371a6fdbb08cfb6ad191f218b51ee
```

此时容器会在后台运行并不会把输出的结果 `(STDOUT)` 打印到宿主机上面(输出结果可以用 `docker logs` 查看)。

注： 容器是否会长久运行，是和 `docker run` 指定的命令有关，和 `-d` 参数无关。

使用 `-d` 参数启动后会返回一个唯一的 `id`，也可以通过 `docker container ls` 命令来查看容器信息。

```sh
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
a9312ab25a6e        ubuntu:17.10        "/bin/sh -c 'while t…"   5 minutes ago       Up 5 minutes                             quizzical_neumann
d8ad862e6e0f        nginx               "nginx -g 'daemon of…"   About an hour ago   Up About an hour    0.0.0.0:80->80/tcp   myweb
```

要获取容器的输出信息，可以通过 `docker container logs` 命令。


命令格式

```sh
$ docker container logs [container ID or NAMES]
```

`container ID`

```sh
$ docker container logs a9312ab25a6e
hello world
hello world
hello world
hello world
hello world
hello world
...
```

或者

`NAMES`

```sh
$ docker container logs quizzical_neumann
hello world
hello world
hello world
hello world
hello world
hello world
...
```

## 终止容器

可以使用 `docker container stop` 来终止一个运行中的容器。

此外，当 `Docker` 容器中指定的应用终结时，容器也自动终止。

例如对于上一章节中只启动了一个终端的容器，用户通过 `exit` 命令或 `Ctrl+d` 来退出终端时，所创建的容器立刻终止。

### 查看终止状态的容器

终止状态的容器可以用 `docker container ls -a` 命令看到。例如

```sh
$ docker container ls -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                           PORTS                NAMES
fcf39bb41624        ubuntu:17.10        "/bin/bash"              40 minutes ago      Exited (0) 40 minutes ago                             objective_wozniak
a9312ab25a6e        ubuntu:17.10        "/bin/sh -c 'while t…"   43 minutes ago      Up 43 minutes                                         quizzical_neumann
6e63bcf5e44d        ubuntu:17.10        "/bin/sh -c 'while t…"   About an hour ago   Exited (0) 45 minutes ago                             brave_sammet
57eac9f84f5c        ubuntu:14.04        "/bin/bash"              About an hour ago   Exited (0) About an hour ago                          frosty_mayer
64835cfb8d6a        ubuntu:14.04        "/bin/echo 'Hello wo…"   About an hour ago   Exited (0) About an hour ago                          dreamy_raman
5e629833e011        myweb:v1            "/bin/bash"              About an hour ago   Exited (100) About an hour ago                        amazing_euler
3e3f0c8bb31f        myweb:v1            "nginx -g 'daemon of…"   2 hours ago         Created                                               web
d8ad862e6e0f        nginx               "nginx -g 'daemon of…"   2 hours ago         Up 2 hours                       0.0.0.0:80->80/tcp   myweb
24215366c6ad        hello-world         "/hello"                 2 hours ago         Exited (0) 2 hours ago                                inspiring_keller
root@souyunku:~/mydocker#
```

处于终止状态的容器，可以通过 `docker container start` 命令来重新启动

### 启动终止状态的容器

```sh
$ docker container start objective_wozniak
objective_wozniak
```

```sh
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
fcf39bb41624        ubuntu:17.10        "/bin/bash"              42 minutes ago      Up 5 seconds                             objective_wozniak
a9312ab25a6e        ubuntu:17.10        "/bin/sh -c 'while t…"   About an hour ago   Up About an hour                         quizzical_neumann
d8ad862e6e0f        nginx               "nginx -g 'daemon of…"   2 hours ago         Up 2 hours          0.0.0.0:80->80/tcp   myweb
```

### 重启运行态的容器

此外，`docker container restart` 命令会将一个运行态的容器终止，然后再重新启动它。

```sh
$ docker container restart quizzical_neumann
quizzical_neumann
```

```sh
$ ocker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
fcf39bb41624        ubuntu:17.10        "/bin/bash"              44 minutes ago      Up 2 minutes                             objective_wozniak
a9312ab25a6e        ubuntu:17.10        "/bin/sh -c 'while t…"   About an hour ago   Up 9 seconds                             quizzical_neumann
d8ad862e6e0f        nginx               "nginx -g 'daemon of…"   2 hours ago         Up 2 hours          0.0.0.0:80->80/tcp   myweb
root@souyunku:~/mydocker#
```

### 停止容器

```sh
$ docker container stop objective_wozniak
objective_wozniak
```

```sh
$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
a9312ab25a6e        ubuntu:17.10        "/bin/sh -c 'while t…"   About an hour ago   Up 19 minutes                            quizzical_neumann
d8ad862e6e0f        nginx               "nginx -g 'daemon of…"   2 hours ago         Up 2 hours          0.0.0.0:80->80/tcp   myweb
```

## 进入容器

在使用 `-d` 参数时，容器启动后会进入后台。

某些时候需要进入容器进行操作，包括使用 `docker attach` 命令或 `docker exec` 命令，推荐大家使用 `docker exec` 命令，原因会在下面说明。

### `attach` 命令

`docker attach` 是 `Docker` 自带的命令。下面示例如何使用该命令。`

```sh
$ docker run -dit ubuntu

Unable to find image 'ubuntu:latest' locally
latest: Pulling from library/ubuntu
50aff78429b1: Pull complete 
f6d82e297bce: Pull complete 
275abb2c8a6f: Pull complete 
9f15a39356d6: Pull complete 
fc0342a94c89: Pull complete 
Digest: sha256:ec0e4e8bf2c1178e025099eed57c566959bb408c6b478c284c1683bc4298b683
Status: Downloaded newer image for ubuntu:latest
74447e5bca608a88ef6dc136d228ec36d4dd16220b38b0b35a0a83572dee627d
```

```sh
$ docker attach 74447

root@74447e5bca60:/# 
root@74447e5bca60:/# exit
exit
```

注意： 如果从这个 `stdin` 中 `exit`，会导致容器的停止。

### `exec` 命令

**`-i` `-t` 参数**

`docker exec` 后边可以跟多个参数，这里主要说明 `-i -t` 参数。

只用 `-i` 参数时，由于没有分配伪终端，界面没有我们熟悉的 `Linux` 命令提示符，但命令执行结果仍然可以返回。

当 `-i -t` 参数一起使用时，则可以看到我们熟悉的 `Linux` 命令提示符。

```sh
$ docker run -dit ubuntu
1f1b0989bff915f1293971bf275fde8f197e34ba826bcb93903fd0c6236111ea
```

```sh
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                NAMES
1f1b0989bff9        ubuntu              "/bin/bash"              About a minute ago   Up About a minute                        reverent_meninsky
```

```sh
$ docker exec -it 1f1b0 bash

root@1f1b0989bff9:/# ps
  PID TTY          TIME CMD
   20 pts/1    00:00:00 bash
   28 pts/1    00:00:00 ps
root@1f1b0989bff9:/# ls
bin  boot  dev  etc  home  lib  lib64  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var
root@1f1b0989bff9:/# exit 
exit


$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
1f1b0989bff9        ubuntu              "/bin/bash"              6 minutes ago       Up 6 minutes                             reverent_meninsky
```

如果从这个 `stdin` 中 `exit`，不会导致容器的停止。这就是为什么推荐大家使用 `docker exec` 的原因。

更多参数说明请使用 `docker exec --help` 查看。

## 导出和导入容器

### 导出容器

如果要导出本地某个容器，可以使用 `docker export` 命令。

```sh
$ docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                         PORTS                NAMES
1f1b0989bff9        ubuntu              "/bin/bash"              9 minutes ago       Up 9 minutes                                        reverent_meninsky

$ docker export 1f1b0989bff9 > ubuntu.tar

$ ll

total 87720
drwxr-xr-x 2 root root     4096 Dec 31 13:51 ./
drwx------ 4 root root     4096 Dec 31 10:08 ../
-rw-r--r-- 1 root root      172 Dec 31 10:08 Dockerfile
-rw-r--r-- 1 root root 89811456 Dec 31 13:52 ubuntu.tar
```
这样将导出容器快照到本地文件。

### 导入容器快照

可以使用 docker import 从容器快照文件中再导入为镜像，例如

```sh
$ cat ubuntu.tar | docker import - test/ubuntu:v1.1

sha256:055405712b98244e632944e96f00bd5e5f28da6c49e1b1ea24bd1d42438ca9c5
```

```sh
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
test/ubuntu         v1.1                055405712b98        21 seconds ago      85.8MB
```

## 删除

### 删除容器

可以使用 `docker container rm` 来删除一个处于终止状态的容器。例如

```sh
$ docker container ls -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                         PORTS                NAMES
1f1b0989bff9        ubuntu              "/bin/bash"              26 minutes ago      Up 26 minutes                                       reverent_meninsky
74447e5bca60        ubuntu              "/bin/bash"              33 minutes ago      Exited (0) 29 minutes ago                           competent_lumiere
```

```sh
$ docker container rm competent_lumiere
competent_lumiere
```

如果要删除一个运行中的容器，可以添加 -f 参数。Docker 会发送 SIGKILL 信号给容器。

```sh
$ docker container rm -f reverent_meninsky
reverent_meninsky
```

### 删除所有处于终止状态的容器

用 `docker container ls -a` 命令可以查看所有已经创建的包括终止状态的容器，如果数量太多要一个个删除可能会很麻烦，用下面的命令可以清理掉所有处于终止状态的容器。

```sh
$ docker container ls -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                         PORTS                NAMES
8b8b04dd97cb        ubuntu:14.04        "/bin/bash"              About an hour ago   Exited (127) 37 minutes ago                         vigilant_gates
f280238f5a92        ubuntu:14.04        "/bin/bash"              About an hour ago   Exited (0) About an hour ago                        friendly_curie
fcf39bb41624        ubuntu:17.10        "/bin/bash"              3 hours ago         Up About an hour                                    objective_wozniak
a9312ab25a6e        ubuntu:17.10        "/bin/sh -c 'while t…"   3 hours ago         Up 2 hours                                          quizzical_neumann
6e63bcf5e44d        ubuntu:17.10        "/bin/sh -c 'while t…"   3 hours ago         Up About an hour                                    brave_sammet
57eac9f84f5c        ubuntu:14.04        "/bin/bash"              3 hours ago         Up About an hour                                    frosty_mayer
64835cfb8d6a        ubuntu:14.04        "/bin/echo 'Hello wo…"   3 hours ago         Exited (0) About an hour ago                        dreamy_raman
5e629833e011        myweb:v1            "/bin/bash"              3 hours ago         Exited (100) 3 hours ago                            amazing_euler
3e3f0c8bb31f        myweb:v1            "nginx -g 'daemon of…"   4 hours ago         Created                                             web
d8ad862e6e0f        nginx               "nginx -g 'daemon of…"   4 hours ago         Up 4 hours                     0.0.0.0:80->80/tcp   myweb
24215366c6ad        hello-world         "/hello"                 4 hours ago         Exited (0) 4 hours ago                              inspiring_keller
```

**删除所有处于终止状态的容器**

```sh
$ docker container prune
WARNING! This will remove all stopped containers.
Are you sure you want to continue? [y/N] y
Deleted Containers:
8b8b04dd97cbbed268b24c419ba3ddaca7ab07ab85f7629004b3cc16d1509e3f
f280238f5a928b8048a88c235071e6baad2d9949bb5e85b73957d5485b26fdbd
64835cfb8d6a821ed4c941a32a767b88cdbcc4c0b322a86119810f866bbfa60e
5e629833e011dac82c93f1c37e0ac291e5ac3b039ceac7a58c4d3acf119bcafb
3e3f0c8bb31f0da5a6a9205aea73a8e4e1ff2d3c55a9a42ee1ab9537e08e8e1e
24215366c6ad2546eaf098839b28265e077ce3069779ec3a703ff400bc2b4dfa

Total reclaimed space: 131B
```

已经没有停止的容器了

```sh
root@souyunku:~/mydocker# docker container ls -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                NAMES
fcf39bb41624        ubuntu:17.10        "/bin/bash"              3 hours ago         Up About an hour                         objective_wozniak
a9312ab25a6e        ubuntu:17.10        "/bin/sh -c 'while t…"   3 hours ago         Up 2 hours                               quizzical_neumann
6e63bcf5e44d        ubuntu:17.10        "/bin/sh -c 'while t…"   3 hours ago         Up About an hour                         brave_sammet
57eac9f84f5c        ubuntu:14.04        "/bin/bash"              3 hours ago         Up About an hour                         frosty_mayer
d8ad862e6e0f        nginx               "nginx -g 'daemon of…"   4 hours ago         Up 4 hours          0.0.0.0:80->80/tcp   myweb
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

