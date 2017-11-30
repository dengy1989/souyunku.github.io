---
layout: post
title: CentOs7.3 编译安装 Nginx 1.9.9
categories: Nginx
description: CentOs7.3 编译安装 Nginx 1.9.9
keywords: Nginx
---

# CentOs7.3 编译安装 Nginx 1.9.9

## 安装

### 安装依赖

```sh		  
$ yum install -y  gcc gcc-c++ autoconf automake zlib zlib-devel openssl openssl-devel pcre pcre-devel
```		

### 下载并解压

```sh	
$ cd /opt/
$ wget http://nginx.org/download/nginx-1.9.9.tar.gz
$ tar zxvf nginx-1.9.9.tar.gz
```	

### 编译

编译时候可以指定编译参数，参考文章尾部：**常用编译选项**

```sh
$ cd nginx-1.9.9
$ ./configure 
```	

### 安装

```sh
$ make
$ make && make install
```	

默认安装在`/usr/locale/nginx`

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

```
nginx: configuration file /usr/local/nginx/conf/nginx.conf test is successful
```

### 启动

```sh
$ /usr/local/nginx/sbin/nginx
```


如果不能访问，检查防火墙并关闭防火墙

centos 6.x 关闭 iptables

```sh
$ service iptables status # 查询防火墙状态命令
$ service iptables stop # 关闭命令
```
centos 7.x 关闭firewall

```sh
$ ssystemctl status  firewalld.service # 查看状态
$ systemctl stop firewalld.service # 停止firewall
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



## 常用编译选项

```sh
./configure \
--prefix=/home/nginx \
--sbin-path=/usr/sbin/nginx \
--user=nginx \
--group=nginx \
--conf-path=/etc/nginx/nginx.conf \
--error-log-path=/home/log/nginx/error.log \
--http-log-path=/home/log/nginx/access.log \
--with-http_ssl_module \
--with-http_gzip_static_module \
--with-http_stub_status_module \
--with-http_realip_module \
--pid-path=/home/run/nginx.pid \
--with-pcre=/home/software/pcre-8.35 \
--with-zlib=/home/software/zlib-1.2.8 \
--with-openssl=/home/software/openssl-1.0.1i
```

## 选项说明


```sh
--prefix=/home/nginx \ Nginx安装的根路径,所有其它路径都要依赖该选项
--sbin-path=/usr/sbin/nginx \ nginx的可执行文件的路径（nginx）
--user=nginx \ worker进程运行的用户
--group=nginx \ worker进程运行的组
--conf-path=/etc/nginx/nginx.conf \  指向配置文件（nginx.conf）
--error-log-path=/var/log/nginx/error.log \ 指向错误日志目录
--http-log-path=/var/log/nginx/access.log \  设置主请求的HTTP服务器的日志文件的名称
--with-http_ssl_module \  使用https协议模块。默认情况下，该模块没有被构建。前提是openssl与openssl-devel已安装
--with-http_gzip_static_module \  启用ngx_http_gzip_static_module支持（在线实时压缩输出数据流）
--with-http_stub_status_module \  启用ngx_http_stub_status_module支持（获取nginx自上次启动以来的工作状态）
--with-http_realip_module \  启用ngx_http_realip_module支持（这个模块允许从请求标头更改客户端的IP地址值，默认为关）
--pid-path=/var/run/nginx.pid \  指向pid文件（nginx.pid）

设置PCRE库的源码路径，如果已通过yum方式安装，使用–with-pcre自动找到库文件。使用–with-pcre=PATH时，需要从PCRE网站下载pcre库的源码（版本4.4 – 8.30）并解压，剩下的就交给Nginx的./configure和make来完成。perl正则表达式使用在location指令和 ngx_http_rewrite_module模块中。
--with-pcre=/home/software/pcre-8.35 \ 

指定 zlib（版本1.1.3 – 1.2.5）的源码解压目录。在默认就启用的网络传输压缩模块ngx_http_gzip_module时需要使用zlib 。
--with-zlib=/home/software/zlib-1.2.8 \

指向openssl安装目录
--with-openssl=/home/software/openssl-1.0.1i
```

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)  
 - GitHub：[https://github.com/souyunku](https://github.com/souyunku)  
 - Segment Fault：[https://sf.gg/blog/souyunku](https://sf.gg/blog/souyunku)  
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")



















