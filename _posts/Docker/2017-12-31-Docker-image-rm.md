---
layout: post
title: Docker Image 解决镜像无法删除的问题
categories: Docker
description: Docker Image 解决镜像无法删除的问题
keywords: Docker
---

`Error response from daemon: conflict: unable to delete 4ac2d12f10cd (must be forced) - image is referenced in multiple repositories`

来自守护进程的错误响应：冲突：无法删除4ac2d12f10cd（必须强制） - 映像在多个存储库中被引用

# 1.删除镜像

## 查看镜像

```sh
root@souyunku:~/mydocker# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
nginx               v1                  4ac2d12f10cd        41 minutes ago      108MB
souyunku/nginx      v1                  4ac2d12f10cd        41 minutes ago      108MB
hello-world         latest              f2a91732366c        5 weeks ago         1.85kB
```

## 删除失败

删除其中一个镜像，这里的镜像有1个`repo`引用，并且没有容器使用


并且没有容器使用

```sh
root@souyunku:~/mydocker# docker container ls -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                         PORTS               NAMES
4c104074b3f4        hello-world         "/hello"            About an hour ago   Exited (0) About an hour ago                       priceless_hawking
```


镜像有1个`repo`引用

```sh
root@souyunku:~/mydocker# docker rmi 4ac
Error response from daemon: conflict: unable to delete 4ac2d12f10cd (must be forced) - image is referenced in multiple repositories
```

# 2.解决方法

## 删除`REPOSITORY`

被删除的`ImageID`，这里存在1个`REPOSITORY`名字引用，解决方法如下：

即删除时指定名称，而不是`IMAGE ID`。

```sh
root@souyunku:~/mydocker# docker rmi souyunku/nginx:v1
Untagged: souyunku/nginx:v1
```

## 再删除IMAGE ID就可以了：

```sh
root@souyunku:~/mydocker# docker rmi 4ac
Untagged: nginx:v1
Deleted: sha256:4ac2d12f10cdb99c099749432b7a450ee1c6958e0f2f964cd64c6b086ba3e622
Deleted: sha256:346164f732e08d72d1f64828acda4e5ca93f79473f443ce57d9cfe69d9b66b24
Deleted: sha256:3f8a4339aadda5897b744682f5f774dc69991a81af8d715d37a616bb4c99edf5
Deleted: sha256:bb528503f6f01b70cd8de94372e1e3196fad3b28da2f69b105e95934263b0487
Deleted: sha256:410204d28a96d436e31842a740ad0c827f845d22e06f3b1ff19c3b22706c3ed4
Deleted: sha256:2ec5c0a4cb57c0af7c16ceda0b0a87a54f01f027ed33836a5669ca266cafe97a
```

# 3.查看镜像

```sh
root@souyunku:~/mydocker# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              f2a91732366c        5 weeks ago         1.85kB
```


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

