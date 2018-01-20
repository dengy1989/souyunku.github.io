---
layout: post
title: Docker Compose 1.18.0 之服务编排详解
categories: Docker Compose
description: Docker Compose 1.18.0 之服务编排详解
keywords: Docker Compose
---

一个使用Docker容器的应用，通常由多个容器组成。使用Docker Compose，不再需要使用shell脚本来启动容器。在配置文件中，所有的容器通过services来定义，然后使用docker-compose脚本来启动，停止和重启应用，和应用中的服务以及所有依赖服务的容器
Compose 通过一个配置文件来管理多个Docker容器，非常适合组合使用多个容器进行开发的场景。

服务编排工具使得Docker应用管理更为方便快捷。 

Docker Compose网站：[https://docs.docker.com/compose](https://docs.docker.com/compose/)

**使用`Compose`基本上是三个步骤：**

1.定义`Dockerfile` 

2.编写`docker-compose.yml`

3.最后运行 `docker-compose up` 启动服务

# 系统环境

Ubuntu 17.04 x64  
Docker CE: 17.12.0-ce  
Docker Compose: 1.18.0  

[参考-https://docs.docker.com/compose/install/#prerequisites](https://docs.docker.com/compose/install/#prerequisites)

[在Linux上，您可以从GitHub上的Compose存储库版本页面下载Docker Compose 最新二进制文件](https://github.com/docker/compose/releases)

# Compose 安装

**运行此命令下载最新版本的`Docker Compose`**

```sh
$ curl -L https://github.com/docker/compose/releases/download/1.18.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```

**对二进制文件应用可执行权限**

```sh
$ chmod +x /usr/local/bin/docker-compose
```

**验证**

```sh
$ docker-compose --version
docker-compose version 1.16.1, build 6d1ac21
```

**卸载**

要卸载 Docker Compose，如果使用 curl 以下安装：
 
```sh
$ rm /usr/local/bin/docker-compose
```

# 入门示例

## WordPress

使用`Docker Compose` 可以轻松地在`Docker`容器中，构建独立环境运行的`WordPress`，在开始之前必须安装`Docker Compose`。

## 编写配置

1.创建一个空的项目目录。

新建一个你能记住的目录，这个目录是应用镜像的上下文，该目录用于存放构建该镜像的资源

在这个目录里面将会新建一个`docker-compose.yml`文件

```sh
$ mkdir my_wordpress
```

2.进入`my_wordpress` 目录


```sh
$ cd my_wordpress
```

3.创建一个`docker-compose.yml`文件，将启动您的 `WordPress`博客和一个单独的`MySQL`实例并挂载数据持久化到宿主机

```sh
$ touch docker-compose.yml
$ vi docker-compose.yml
```

内容如下

```sh
version: '3'

services:
   db:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: somewordpress
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: wordpress

   wordpress:
     depends_on:
       - db
     image: wordpress:latest
     ports:
       - "8000:80"
     restart: always
     environment:
       WORDPRESS_DB_HOST: db:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: wordpress
volumes:
    db_data:
```


**image**

`image: ` 指定服务的镜像名称或镜像 `ID` `image: mysql`,`image: wordpress:latest`。如果镜像在本地不存在，Compose 将会尝试拉取这个镜像。

所以我们不需要先拉取镜像

**volumes**

`- db_data：` 指`MySQL`实例挂载数据持久化到宿主机`/var/lib/docker/volumes/mywordpress_db_data/_data`

**PS**

```sh
$ docker run -v /var/lib/mysql --name mywordpress_db_data -e MYSQL_ROOT_PASSWORD=wordpress -d mysql
$ docker run --name some-wordpress --link mywordpress_db_data:mysql -p 8002:80 -d wordpress
```

以上命令的意思是新建`mywordpress_db_data `和`some-wordpress`容器。等同于：`docker-compose.yml` 内容

## 启动服务

```sh
root@souyunku:/opt/my_wordpress# docker-compose up -d
```

如果看到如下信息就证明没毛病

```sh
Pulling db (mysql:5.7)...
5.7: Pulling from library/mysql
f49cf87b52c1: Pull complete
78032de49d65: Pull complete
837546b20bc4: Pull complete
9b8316af6cc6: Pull complete
1056cf29b9f1: Pull complete
86f3913b029a: Pull complete
f98eea8321ca: Pull complete
3a8e3ebdeaf5: Pull complete
4be06ac1c51e: Pull complete
920c7ffb7747: Pull complete
Digest: sha256:7cdb08f30a54d109ddded59525937592cb6852ff635a546626a8960d9ec34c30
Status: Downloaded newer image for mysql:5.7
Pulling wordpress (wordpress:latest)...
latest: Pulling from library/wordpress
e7bb522d92ff: Pull complete
75651f247827: Pull complete
dbcf8fd0150f: Pull complete
de80263f26f0: Pull complete
65be8ad4c5fd: Pull complete
239d5fed0dda: Pull complete
5ab39b683a9f: Pull complete
4a3f54f2d93a: Pull complete
28c970ad99e9: Pull complete
5d1e20c7c396: Pull complete
05f877a23903: Pull complete
e0a5c61bdaa6: Pull complete
d27d2d70a072: Pull complete
ba039fef4b7e: Pull complete
fd026e22f5c3: Pull complete
a523c6d55ab4: Pull complete
025590874132: Pull complete
d1f0ca983d7b: Pull complete
40d597c8be8b: Pull complete
Digest: sha256:573257b41e1c3554cfe3a856d3c329030a821194172e2aeb1d3a7f5dd896ccb4
Creating mywordpress_db_1        ... done
Creating mywordpress_db_1        ... 
Creating mywordpress_wordpress_1 ... done
root@souyunku:/opt/my_wordpress#
```

## 查看容器

```sh
root@souyunku:/opt/my_wordpress# docker container ps -a
CONTAINER ID        IMAGE                              COMMAND                  CREATED             STATUS              PORTS                    NAMES
d715012934dc        wordpress:latest                   "docker-entrypoint.s…"   2 hours ago         Up 19 seconds       0.0.0.0:8000->80/tcp     mywordpress_wordpress_1
ce956cf8d74b        mysql:5.7                          "docker-entrypoint.s…"   2 hours ago         Up 2 hours          3306/tcp                 mywordpress_db_1
```

## 查看镜像

```sh
root@souyunku:/opt/my_wordpress# docker images
REPOSITORY                         TAG                 IMAGE ID            CREATED             SIZE
mysql                              5.7                 f008d8ff927d        2 days ago          409MB
wordpress                          latest              28084cde273b        9 days ago          408MB
root@souyunku:/opt/my_wordpress#
```

## 访问服务

![my_wordpress][1]

![my_wordpress][2]

![my_wordpress][3]
                     

# 编写参考

每个`docker-compose.yml`必须定义`image`或者`build`中的一个，其它的是可选的。

## image

`image` 指定镜像`tag`或者`ID`。示例：

```sh
image: mysql
image: redis
image: ubuntu:14.04
image: tutum/influxdb
image: example-registry.com:4000/postgresql
image: a4bc65fd
```

**注意**，在`version 1`里同时使用image和build是不允许的，`version 2`则可以，如果同时指定了两者，会将`build`出来的镜像打上名为`image`标签。

## build

用来指定一个包含`Dockerfile`文件的路径。一般是当前目录.`build`并生成一个随机命名的镜像。

实例

```sh
├── app
│   ├── Dockerfile
│   └── docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar
├── docker-compose.yml
```

**Dockerfile** 内容

```sh
root@souyunku:/opt/app# cat Dockerfile 
FROM java:8
VOLUME /tmp
ADD docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar app.jar
RUN bash -c 'touch /app.jar'
EXPOSE 9000
```

**docker-compose.yml** 内容

```sh
root@souyunku:/opt# cat docker-compose.yml 
app:
  build: ./app
  ports:
    - "9090:80"
  expose:
    - 80
```

`./app` 是放`Dockerfile` 的路径
 
`ports`  用于暴露端口 同`docker run -p`
 
## command

用来覆盖缺省命令。示例：

```sh
command: bundle exec thin -p 3000
```

`command`也支持数组形式

```sh
command: [bundle, exec, thin, -p, 3000]
```

## links

用于链接另一容器服务，如需要使用到另一容器的mysql服务。可以给出服务名和别名；也可以仅给出服务名，这样别名将和服务名相同。

同`docker run --link`。示例：

```sh
links:
 - db
 - db:mysql
 - redis
```


使用了别名将自动会在容器的/etc/hosts文件里创建相应记录：

```sh
172.17.2.186  db
172.17.2.186  mysql
172.17.2.187  redis
```

所以我们在容器里就可以直接使用别名作为服务的主机名。

## ports

用于暴露端口。同`docker run -p`。

示例：

```sh
ports:
 - "3000"
 - "8000:8000"
 - "49100:22"
 - "127.0.0.1:8001:8001"
```

## expose

`expose`提供`container`之间的端口访问，不会暴露给主机使用。同`docker run --expose`。

```sh
expose:
 - "3000"
 - "8000"
```

## volumes

挂载数据卷。同`docker run -v`。

示例：

```sh
volumes:
 - /var/lib/mysql
 - cache/:/tmp/cache
 - ~/configs:/etc/configs/:ro
```

**进入MySQL容器**

查看容器ID

```sh
root@souyunku:# docker container ps -a
CONTAINER ID        IMAGE                              COMMAND                  CREATED             STATUS              PORTS                    NAMES
559e49f8dc01        wordpress:latest                   "docker-entrypoint.s…"   18 minutes ago      Up 18 minutes       0.0.0.0:8000->80/tcp     mywordpress_wordpress_1
3c207b3e16bd        mysql:5.7                          "docker-entrypoint.s…"   18 minutes ago      Up 18 minutes       3306/tcp                 mywordpress_db_1
```

通过容器ID进入MySQL容器

```sh
root@souyunku:# docker exec -it 3c207b3e16bd bash
```

进入MySQL容器的存储目录

```sh
root@3c207b3e16bd:/# cd var/lib/mysql
root@3c207b3e16bd:/var/lib/mysql# ls
auto.cnf    ca.pem	     client-key.pem  ib_logfile0  ibdata1  mysql	       private_key.pem	server-cert.pem  sys	   wordpress
ca-key.pem  client-cert.pem  ib_buffer_pool  ib_logfile1  ibtmp1   performance_schema  public_key.pem	server-key.pem	 test.txt
root@3c207b3e16bd:/var/lib/mysql# cat test.txt
1234
```

新建一个文本，用于测试MySQL容器的挂载目录，有没有同步到宿主机

```sh
root@3c207b3e16bd:/var/lib/mysql# touch test.txt
root@3c207b3e16bd:/var/lib/mysql# echo '1234' >test.txt 
```

**宿主机查看容器挂载是否同步**

```sh
root@souyunku:/var/lib/docker/volumes/mywordpress_db_data/_data# pwd
/var/lib/docker/volumes/mywordpress_db_data/_data

root@souyunku:/var/lib/docker/volumes/mywordpress_db_data/_data# ls
auto.cnf    ca.pem           client-key.pem  ibdata1      ib_logfile1  mysql               private_key.pem  server-cert.pem  sys       wordpress
ca-key.pem  client-cert.pem  ib_buffer_pool  ib_logfile0  ibtmp1       performance_schema  public_key.pem   server-key.pem   test.txt

root@souyunku:/var/lib/docker/volumes/mywordpress_db_data/_data# cat test.txt 
1234
root@souyunku:/var/lib/docker/volumes/mywordpress_db_data/_data#
```

## volumes_from

挂载数据卷容器，挂载是容器。同`docker run --volumes-from`。示例：

```sh
volumes_from:
 - service_name
 - service_name:ro
 - container:container_name
 - container:container_name:rw
```

`container:container_name`格式仅支持`version 2`。


## environment

添加环境变量。同`docker run -e`。可以是数组或者字典格式：

```sh
environment:
  RACK_ENV: development
  SESSION_SECRET:

environment:
  - RACK_ENV=development
  - SESSION_SECRET
```

## depends_on

用于指定服务依赖，一般是`mysql、redis`等。
指定了依赖，将会优先于服务创建并启动依赖。

`links`也可以指定依赖。

## external_links

链接搭配`docker-compose.yml`文件或者`Compose`之外定义的服务，通常是提供共享或公共服务。格式与`links`相似：

```sh
external_links:
 - redis_1
 - project_db_1:mysql
 - project_db_1:postgresql
```

注意，`external_links`链接的服务与当前服务必须是同一个网络环境。

## extra_hosts

添加主机名映射。

```sh
extra_hosts:
 - "somehost:162.242.195.82"
 - "otherhost:50.31.209.229"
```

将会在`/etc/hosts`创建记录：

```sh
162.242.195.82  somehost
50.31.209.229   otherhost
```

## extends

继承自当前`yml`文件或者其它文件中定义的服务，可以选择性的覆盖原有配置。

```sh
extends:
  file: common.yml
  service: webapp
```

`service`必须有，`file`可选。`service`是需要继承的服务，例如`web`、`database`。


## net

设置网络模式。同`docker`的`--net`参数。

```sh
net: "bridge"
net: "none"
net: "container:[name or id]"
net: "host"
```

## dns

自定义dns服务器。

```sh
dns: 8.8.8.8
dns:
  - 8.8.8.8
  - 9.9.9.9
```

## 更多

**cpu_shares, cpu_quota, cpuset, domainname, hostname, ipc, mac_address, mem_limit, memswap_limit, privileged, read_only, restart, shm_size, stdin_open, tty, user, working_dir**

这些命令都是单个值，含义请参考

**编写 docker-compose 请参考官方文档**

**Compose file version 3**

[https://docs.docker.com/compose/compose-file/](https://docs.docker.com/compose/compose-file/)

**Compose file version 2**

[https://docs.docker.com/compose/compose-file/compose-file-v2/](https://docs.docker.com/compose/compose-file/compose-file-v2/)

**Compose file version 1**

[https://docs.docker.com/compose/compose-file/compose-file-v1/](https://docs.docker.com/compose/compose-file/compose-file-v1/)

**参考**

[https://docs.docker.com/compose/overview/](https://docs.docker.com/compose/overview/)

[https://docs.docker.com/compose/](https://docs.docker.com/compose/)
				 
[1]: http://www.ymq.io/images/2018/docker/compose/1.png
[2]: http://www.ymq.io/images/2018/docker/compose/2.png
[3]: http://www.ymq.io/images/2018/docker/compose/3.png
[4]: http://www.ymq.io/images/2018/docker/compose/4.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/01/17/Docker-Compose-example](http://www.ymq.io/2018/01/17/Docker-Compose-example)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，"搜云库"，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")
