---
layout: post
title: Ubuntu 17.04 编译安装 Nginx 1.9.9 配置 https 免费证书
categories: Nginx
description: Ubuntu 17.04 编译安装 Nginx 1.9.9 配置 https 免费证书
keywords: Nginx
---

# Ubuntu 17.04 编译安装 Nginx 1.9.9 配置 https 免费证书

## 安装 Nginx

### 安装依赖

```sh		  
$ apt-get update
$ apt-get install build-essential libtool libpcre3 libpcre3-dev zlib1g-dev
$ apt-get install openssl
$ apt-get install libssl-dev
```		

### 下载并解压

```sh	
$ cd /opt/
$ wget http://nginx.org/download/nginx-1.9.9.tar.gz
$ tar zxvf nginx-1.9.9.tar.gz
```	

### 编译

```sh
$ cd nginx-1.9.9
$ ./configure --prefix=/usr/local/nginx \--with-http_ssl_module 
```	

### 安装

```sh
$ make
$ make && make install
```	

默认安装在`/usr/local/nginx`

里面有四个目录：
 - conf: 配置文件夹，最重要文件是nginx.conf
 - html: 静态网页文件夹
 - logs: 日志文件夹
 - sbin: nginx 的可执行文件，启动、停止等操作

## 常用命令

### 正确性检查

每次修改nginx配置文件后都要进行检查

```sh
$ /usr/local/nginx/sbin/nginx -t
```

```sh
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```

### 启动

```sh
$ /usr/local/nginx/sbin/nginx
```

浏览器输入本机IP ，看到如下内容证明安装成功

```sh
Welcome to nginx!

If you see this page, the nginx web server is successfully installed and working. Further configuration is required.

For online documentation and support please refer to nginx.org.
Commercial support is available at nginx.com.

Thank you for using nginx.
```

### 停止

```sh
$ /usr/local/nginx/sbin/nginx -s stop
```

### 重启

```sh
$ /usr/local/nginx/sbin/nginx -s reload
```

## 配置证书

### 安装 acme.sh

安装很简单, 一个命令:

```sh
curl  https://get.acme.sh | sh
```

### 生成证书

```sh
cd ~/.acme.sh/
apt install socat
sh acme.sh  --issue -d docker.souyunku.com   --standalone
```

### 复制证书

```sh
mkdir -p /certs
cd /root/.acme.sh/docker.souyunku.com
cp docker.souyunku.com.cer /certs
cp docker.souyunku.com.key /certs
```

### 配置Nginx

```sh
vim /usr/local/nginx/conf/nginx.conf
```

```sh
server {
	listen 443;
	ssl on;
	ssl_certificate  /certs/docker.souyunku.com.cer;
	ssl_certificate_key  /certs/docker.souyunku.com.key;
}
```

每次修改nginx配置文件后都要进行检查

```sh
$ /usr/local/nginx/sbin/nginx -t
```

### 启动Nginx

```sh
$ /usr/local/nginx/sbin/nginx
```

### 测试证书

浏览器访问：[https://docker.souyunku.com/](https://docker.souyunku.com/)

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")



















