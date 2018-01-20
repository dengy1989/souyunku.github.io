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

# 最终效果

![docker-maven-plugin][1]

# 环境准备

 - 系统：Ubuntu 17.04 x64  
 - Docker 17.12.0-ce

 **Ubuntu 17.04 x64 安装 Docker CE**
 
[http://www.ymq.io/2018/01/11/Docker-Install-docker-ce/](http://www.ymq.io/2018/01/11/Docker-Install-docker-ce/)

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

使用 maven 命令： `mvn clean package docker:build`

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
& root@souyunku:# docker images docker-spring-boot-demo-maven-plugin
REPOSITORY                             TAG                 IMAGE ID            CREATED             SIZE
docker-spring-boot-demo-maven-plugin   latest              378fd82432e0        3 minutes ago       659MB
```
## 3.启动镜像

```sh
root@souyunku:# docker run --name MySpringBootMavenPlugin -d -p 8080:80 docker-spring-boot-demo-maven-plugin
84ebb2ebb8c002d3935e6e31c6d2aab05c32c075036368228e84f818d20ded4a
```

## 4.查看容器

```sh
& root@souyunku:# docker container ps -a
CONTAINER ID        IMAGE                                  COMMAND                  CREATED             STATUS              PORTS                    NAMES
84ebb2ebb8c0        docker-spring-boot-demo-maven-plugin   "java -jar /docker-s…"   About an hour ago   Up About an hour    0.0.0.0:8080->80/tcp     MySpringBootMavenPlugin
```

## 5.访问服务

浏览器输入：[http://Docker宿主机IP:8080](http://Docker宿主机IP:8080)能够正常看到界面,文章开头的最终效果页面。

# 二、使用Dockerfile

## 1.新建Dockerfile

使用Dockerfile进行构建Docker镜像

上文讲述的方式是最简单的方式，很多时候，我们还是要借助`Dockerfile`进行构建的，
首先我们在`/docker-spring-boot-demo-maven-plugin/src/main/resources`目录下，建立文件`Dockerfile`

```sh
FROM java:8
VOLUME /tmp
ADD docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar app.jar
RUN bash -c 'touch /app.jar'
EXPOSE 9000
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]
```

## 2.修改POM

项目`pom.xml`修改为如下： 指定`Dockerfile`所在的路径

```sh
<build>
	<plugins>
		<!-- docker的maven插件，官网：https://github.com/spotify/docker-maven-plugin -->
		<plugin>
			<groupId>com.spotify</groupId>
			<artifactId>docker-maven-plugin</artifactId>
			<version>0.4.12</version>
			<configuration>
				<!-- 注意imageName一定要是符合正则[a-z0-9-_.]的，否则构建不会成功 -->
				<!-- 详见：https://github.com/spotify/docker-maven-plugin    Invalid repository name ... only [a-z0-9-_.] are allowed-->
				<imageName>docker-spring-boot-demo-maven-plugin</imageName>
				<!-- 指定Dockerfile所在的路径 -->
				<dockerDirectory>${basedir}/src/main/resources</dockerDirectory>
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

## 3.构建镜像

使用 maven 命令： `mvn clean package docker:build`

```sh
& cd /opt/other-projects/docker-spring-boot-demo-maven-plugin
& mvn clean package docker:build
```

## 4.启动镜像

```sh
root@souyunku:# docker run --name MySpringBootMavenPlugin -d -p 8080:80 docker-spring-boot-demo-maven-plugin
84ebb2ebb8c002d3935e6e31c6d2aab05c32c075036368228e84f818d20ded4a
```

其他步骤和上面一样。这样即可使用`Dockerfile`进行构建`Docker`镜像啦。

## 5.访问服务

浏览器输入：[http://Docker宿主机IP:8080](http://Docker宿主机IP:8080)能够正常看到界面,文章开头的最终效果页面。

# 三、push 镜像

将`Docker`镜像`push`到`DockerHub`上

## 1.修改Maven配置

首先修改`Maven`的全局配置文件`settings.xml`，

查看`settings.xml` 所在位置

```sh
root@souyunku:# find / -name settings.xml
/etc/maven/settings.xml
```

添加以下段落

```sh
vi /etc/maven/settings.xml
```

```xml
<servers>
    <server>
        <id>docker-hub</id>
        <username>DockerHub 的账号</username>
        <password>DockerHub 的密码</password>
        <configuration>
            <email>admin@souyunku.com</email>
        </configuration>
    </server>
</servers>
```

## 2.创建Repository

注册个账号：[https://hub.docker.com/](https://hub.docker.com/)

在`DockerHub`上创建`Create Repository`  ,例如：`docker-spring-boot-demo-maven-plugin`，如下图

![docker-spring-boot-demo-maven-plugin][3]

## 3.修改POM

项目`pom.xml`修改为如下：注意`imageName`的路径要和repo的路径一致

镜像名称

```xml
<properties>
	<docker.image.prefix>souyunku</docker.image.prefix>
</properties>
```

将`Docker`镜像`push`到`DockerHub`上

```xml
<!--3：将Docker镜像push到DockerHub上-->

<!-- docker的maven插件，官网：https://github.com/spotify/docker-maven-plugin -->
<plugin>
	<groupId>com.spotify</groupId>
	<artifactId>docker-maven-plugin</artifactId>
	<version>0.4.12</version>
	<configuration>
		<!-- 注意imageName一定要是符合正则[a-z0-9-_.]的，否则构建不会成功 -->
		<!-- 详见：https://github.com/spotify/docker-maven-plugin Invalid repository
			name ... only [a-z0-9-_.] are allowed -->
		<!-- 如果要将docker镜像push到DockerHub上去的话，这边的路径要和repo路径一致 -->
		<imageName>${docker.image.prefix}/${project.artifactId}</imageName>
		<!-- 指定Dockerfile所在的路径 -->
		<dockerDirectory>${basedir}/src/main/resources</dockerDirectory>
		<resources>
			<resource>
				<targetPath>/</targetPath>
				<directory>${project.build.directory}</directory>
				<include>${project.build.finalName}.jar</include>
			</resource>
		</resources>
		<!-- 以下两行是为了docker push到DockerHub使用的。 -->
		<serverId>docker-hub</serverId>
		<registryUrl>https://index.docker.io/v1/</registryUrl>
	</configuration>
</plugin>
```		

## 4.构建镜像

使用 maven 命令： `mvn clean package docker:build  -DpushImage`

```sh
& cd /opt/other-projects/docker-spring-boot-demo-maven-plugin
& mvn clean package docker:build  -DpushImage
```

**看到类似这样的数据，就证明构建镜像没毛病**

```sh
[INFO] Building image souyunku/docker-spring-boot-demo-maven-plugin
Step 1/6 : FROM java:8

 ---> d23bdf5b1b1b
Step 2/6 : VOLUME /tmp

 ---> Using cache
 ---> cb237cc84527
Step 3/6 : ADD docker-spring-boot-demo-maven-plugin-0.0.1-SNAPSHOT.jar app.jar

 ---> 7fb5e3363ed5
Step 4/6 : RUN bash -c 'touch /app.jar'

 ---> Running in ab5d10dd64ad
Removing intermediate container ab5d10dd64ad
 ---> 05d96fe59da4
Step 5/6 : EXPOSE 9000

 ---> Running in d63e20122d8e
Removing intermediate container d63e20122d8e
 ---> 55ba378141fd
Step 6/6 : ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/app.jar"]

 ---> Running in 962d476363a3
Removing intermediate container 962d476363a3
 ---> 654b596fe91f
ProgressMessage{id=null, status=null, stream=null, error=null, progress=null, progressDetail=null}
Successfully built 654b596fe91f
Successfully tagged souyunku/docker-spring-boot-demo-maven-plugin:latest
[INFO] Built souyunku/docker-spring-boot-demo-maven-plugin
[INFO] Pushing souyunku/docker-spring-boot-demo-maven-plugin
The push refers to repository [docker.io/souyunku/docker-spring-boot-demo-maven-plugin]
464800d90790: Pushed 
d52b146f9147: Pushed 
35c20f26d188: Mounted from souyunku/docker-spring-boot-demo 
c3fe59dd9556: Mounted from souyunku/docker-spring-boot-demo 
6ed1a81ba5b6: Mounted from souyunku/docker-spring-boot-demo 
a3483ce177ce: Mounted from souyunku/docker-spring-boot-demo 
ce6c8756685b: Mounted from souyunku/docker-spring-boot-demo 
30339f20ced0: Mounted from souyunku/docker-spring-boot-demo 
0eb22bfb707d: Mounted from souyunku/docker-spring-boot-demo 
a2ae92ffcd29: Mounted from souyunku/docker-spring-boot-demo 
latest: digest: sha256:8d78ced0034f38be8086c8f812817ec4c12b178470b4cea668046906c825c9ee size: 2424
null: null 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 41.764 s
[INFO] Finished at: 2018-01-16T09:56:23+00:00
[INFO] Final Memory: 36M/88M
[INFO] ------------------------------------------------------------------------
root@souyunku:/opt/other-projects/docker-spring-boot-demo-maven-plugin# 
```

## 5.查看镜像

```sh
root@souyunku:# docker images souyunku/docker-spring-boot-demo-maven-plugin 
REPOSITORY                                      TAG                 IMAGE ID            CREATED             SIZE
souyunku/docker-spring-boot-demo-maven-plugin   latest              654b596fe91f        27 minutes ago      674MB
```

```sh
root@souyunku:# docker images souyunku/docker-spring-boot-demo-maven-plugin 
REPOSITORY                                      TAG                 IMAGE ID            CREATED             SIZE
souyunku/docker-spring-boot-demo-maven-plugin   latest              654b596fe91f        27 minutes ago      674MB
```

`Docker Hub` 查看镜像，看到已经上传成功

![ docker hub 查看镜像][4]


## 6.启动镜像

```sh
root@souyunku:# docker run --name MySpringBootMavenPlugin -d -p 8080:80 docker-spring-boot-demo-maven-plugin
84ebb2ebb8c002d3935e6e31c6d2aab05c32c075036368228e84f818d20ded4a
```

其他步骤和上面一样。这样即可使用`Dockerfile`进行构建`Docker`镜像啦。

## 7.访问服务

浏览器输入：[http://Docker宿主机IP:8080](http://Docker宿主机IP:8080)能够正常看到界面,文章开头的最终效果页面。

# 四、绑定phase执行

将插件绑定在某个phase执行

在很多场景下，我们有这样的需求，例如执行`mvn clean package` 时，自动地为我们构建docker镜像，可以吗？答案是肯定的。我们只需要将插件的`goal` 绑定在某个`phase`即可。

所谓的`phase`和`goal`，可以这样理解：`maven`命令格式是：`mvn phase:goal` ，例如`mvn package docker:build` 那么，`package` 和 `docker` 都是`phase，build` 则是`goal` 。


## 1.修改POM

下面是示例：

首先配置属性：

```xml
<properties>
	<docker.image.prefix>souyunku</docker.image.prefix>
</properties>
```

```xml
<plugin>
	<groupId>com.spotify</groupId>
	<artifactId>docker-maven-plugin</artifactId>
	<version>0.4.12</version>
	
	<executions>
		<execution>
			<id>build-image</id>
			<phase>package</phase>
			<goals>
				<goal>build</goal>
			</goals>
		</execution>
	</executions>
	
	<configuration>
		<!-- 注意imageName一定要是符合正则[a-z0-9-_.]的，否则构建不会成功 -->
		<!-- 详见：https://github.com/spotify/docker-maven-plugin Invalid repository
			name ... only [a-z0-9-_.] are allowed -->
		<!-- 如果要将docker镜像push到DockerHub上去的话，这边的路径要和repo路径一致 -->
		<imageName>${docker.image.prefix}/${project.artifactId}</imageName>
		<!-- 指定Dockerfile所在的路径 -->
		<dockerDirectory>${basedir}/src/main/resources</dockerDirectory>
		<resources>
			<resource>
				<targetPath>/</targetPath>
				<directory>${project.build.directory}</directory>
				<include>${project.build.finalName}.jar</include>
			</resource>
		</resources>
		<!-- 以下两行是为了docker push到DockerHub使用的。 -->
		<serverId>docker-hub</serverId>
		<registryUrl>https://index.docker.io/v1/</registryUrl>
	</configuration>
</plugin>
```

**新加内容**
	
```xml
<executions>
	<execution>
		<id>build-image</id>
		<phase>package</phase>
		<goals>
			<goal>build</goal>
		</goals>
	</execution>
</executions>
```

本例指的是讲`docke`r的`build`目标，绑定在`package`这个`phase`上。
也就是说，用户只需要执行`mvn package` ，就自动执行了`mvn docker:build` 。

## 2.构建镜像

使用 maven 命令： `mvn package`

```sh
& cd /opt/other-projects/docker-spring-boot-demo-maven-plugin
& mvn package
```

## 3.启动镜像

```sh
root@souyunku:# docker run --name MySpringBootMavenPlugin -d -p 8080:80 docker-spring-boot-demo-maven-plugin
84ebb2ebb8c002d3935e6e31c6d2aab05c32c075036368228e84f818d20ded4a
```

## 4.访问服务

浏览器输入：[http://Docker宿主机IP:8080](http://Docker宿主机IP:8080)能够正常看到界面,文章开头的最终效果页面。

![docker-maven-plugin][1]

**推荐阅读：Docker Hub 仓库使用，及搭建 Docker Registry**

[http://www.ymq.io/2017/12/31/Docker-dockerHub/](http://www.ymq.io/2017/12/31/Docker-dockerHub/)


**GitHub :docker-spring-boot-demo-maven-plugin**

[https://github.com/souyunku/other-projects/tree/master/docker-spring-boot-demo-maven-plugin](https://github.com/souyunku/other-projects/tree/master/docker-spring-boot-demo-maven-plugin)

[1]: http://www.ymq.io/images/2018/docker/maven-plugin/1.gif
[2]: http://www.ymq.io/images/2018/docker/maven-plugin/2.png
[3]: http://www.ymq.io/images/2018/docker/maven-plugin/3.png
[4]: http://www.ymq.io/images/2018/docker/maven-plugin/4.png

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io/2018/01/15/Docker-maven-plugin](http://www.ymq.io/2018/01/15/Docker-maven-plugin)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")

