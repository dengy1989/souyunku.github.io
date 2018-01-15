---
layout: post
title: 使用Maven插件构建SpringBoot项目,生成Docker镜像push到DockerHub上
categories: Docker
description: 使用Maven插件构建SpringBoot项目,生成Docker镜像push到DockerHub上
keywords: Docker
---

一个用于构建和推送`Docker`镜像的`Maven`插件。

使用`Maven`插件构建`Docker`镜像，将`Docker`镜像`push`到`DockerHub`上，或者私有仓库，上一篇文章是手写`Dockerfile`，这篇文章借助开源插件`docker-maven-plugin` 进行操作

以下操作。默认你已经阅读过我上一篇文章：

Docker 部署 SpringBoot 项目整合 Redis 镜像做访问计数Demo

[http://www.ymq.io/2018/01/11/Docker-deploy-spring-boot-Integrate-redis](http://www.ymq.io/2018/01/11/Docker-deploy-spring-boot-Integrate-redis/)


# 环境准备

 - 系统：Ubuntu 17.04 x64  
 - Docker 17.12.0-ce

# 插件地址

**docker-maven-plugin**

**GitHub 地址:**[https://github.com/spotify/docker-maven-plugin](https://github.com/spotify/docker-maven-plugin)
 

# 一、简单使用

## 1.修改POM

在`pom.xml`中添加下面这段，

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>

        <!-- docker的maven插件，官网：https://github.com/spotify/docker-maven-plugin -->
        <plugin>
            <groupId>com.spotify</groupId>
            <artifactId>docker-maven-plugin</artifactId>
            <version>0.4.12</version>
            <configuration>
                <!-- 注意imageName一定要是符合正则[a-z0-9-_.]的，否则构建不会成功 -->
                <!-- 详见：https://github.com/spotify/docker-maven-plugin    Invalid repository name ... only [a-z0-9-_.] are allowed-->
                <imageName>microservice-discovery-eureka</imageName>
                <baseImage>java</baseImage>
                <entryPoint>["java", "-jar", "/${project.build.finalName}.jar"]</entryPoint>
                <resources>
                    <resource>
                        <targetPath>/</targetPath>
                        <directory>${project.build.directory}</directory>
                        <include>${project.build.finalName}.jar</include>
                    </resource>
                </resources>
            </configuration>
        </plugin>

    </plugins>
</build>
```

## 2.构建镜像

```sh
& cd /opt/other-projects/docker-spring-boot-demo-maven-plugin
& mvn clean package docker:build
```

我们会发现控制台有类似如下内容：

```sh
Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] 
[INFO] --- maven-jar-plugin:2.6:jar (default-jar) @ docker-spring-boot-demo-maven-plugin ---
[INFO] Building jar: /opt/other-projects/docker-spring-boot-demo-maven-plugin/target/docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar
[INFO] 
[INFO] --- spring-boot-maven-plugin:1.5.9.RELEASE:repackage (default) @ docker-spring-boot-demo-maven-plugin ---
[INFO] 
[INFO] --- docker-maven-plugin:0.4.12:build (default-cli) @ docker-spring-boot-demo-maven-plugin ---
[INFO] Copying /opt/other-projects/docker-spring-boot-demo-maven-plugin/target/docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar -> /opt/other-projects/docker-spring-boot-demo-maven-plugin/target/docker/docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar
[INFO] Building image docker-spring-boot-demo-maven-plugin
Step 1/3 : FROM java

 ---> d23bdf5b1b1b
Step 2/3 : ADD /docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar //

 ---> b5d8f92756f2
Step 3/3 : ENTRYPOINT ["java", "-jar", "/docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar"]

 ---> Running in 6867f460b40c
Removing intermediate container 6867f460b40c
 ---> 378fd82432e0
ProgressMessage{id=null, status=null, stream=null, error=null, progress=null, progressDetail=null}
Successfully built 378fd82432e0
Successfully tagged docker-spring-boot-demo-maven-plugin:latest
[INFO] Built docker-spring-boot-demo-maven-plugin
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 20.568 s
[INFO] Finished at: 2018-01-15T09:21:39+00:00
[INFO] Final Memory: 37M/89M
[INFO] ------------------------------------------------------------------------
root@souyunku:/opt/other-projects/docker-spring-boot-demo-maven-plugin#
```

恭喜，构建成功了。

-我们执行`docker images` 会发现该镜像已经被构建成功：

```sh
root@souyunku:# docker images docker-spring-boot-demo-maven-plugin
REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
docker-spring-boot-demo-maven-plugin   latest              378fd82432e0        3 minutes ago       659MB
```
## 3.启动镜像

```sh
root@souyunku:# docker run --name MySpringBootMavenPlugin -d -p 8080:80 docker-spring-boot-demo-maven-plugin
```

```sh
root@souyunku:# docker run --name MySpringBootMavenPlugin -d -p 8080:80 docker-spring-boot-demo-maven-plugin
84ebb2ebb8c002d3935e6e31c6d2aab05c32c075036368228e84f818d20ded4a
```

## 4.查看容器

```sh
root@souyunku:# docker container ps -a
CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS              PORTS                    NAMES
84ebb2ebb8c0        docker-spring-boot-demo-maven-plugin   "java -jar /docker-s…"   About an hour ago   Up About an hour    0.0.0.0:8080->80/tcp     MySpringBootMavenPlugin
```

## 5.访问服务

浏览器输入：[http://Docker宿主机IP:8080](http://Docker宿主机IP:8080)能够正常看到界面。

![搜云库访客系统][1]

# 一、使用Dockerfile

使用Dockerfile进行构建



**GitHub :docker-spring-boot-demo-maven-plugin**

[https://github.com/souyunku/other-projects/tree/master/docker-spring-boot-demo-maven-plugin](https://github.com/souyunku/other-projects/tree/master/docker-spring-boot-demo-maven-plugin)

[1]: /images/2018/docker/maven-plugin/1.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/01/11/Docker-deploy-spring-boot-Integrate-redis](http://www.ymq.io/2018/01/11/Docker-deploy-spring-boot-Integrate-redis/)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

