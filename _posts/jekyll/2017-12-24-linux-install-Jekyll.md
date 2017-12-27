---
layout: post
title: 在 Linux 上搭建Jekyll静态博客
categories: Jekyll
description: 在 Linux 上搭建Jekyll静态博客
keywords: Jekyll
---

在`CentOS，Ubuntu` 按照同样步骤安装,`Ruby Gems` 往往都无法搭建成，每次都是依赖不对，各种奇葩原因，解决办法就是使用 `RVM` 安装，解决 `Ruby` 的环境依赖管理，而且每次安装`Jekyll`基本不会出错

本文主要介绍如何用一条靠谱的路子快速安装 `Ruby` 环境 搭建`Jekyll`博客。


# 一、Jekyll介绍

`jekyll`是一个简单的免费的`Blog`生成工具，类似`WordPress`。但是和`WordPress`又有很大的不同，原因是`Jekyll`只是一个生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务,例如`Disqus`。最关键的是jekyll可以免费部署在Github上，而且可以绑定自己的域名。

# 二、环境准备

```sh
CentOS 7.3 / Ubuntu 16.04  
rvm 1.29.3  
gem 2.5.1  
ruby 2.3.0  
jekyll 3.6.2  
```

# 三、系统需求

首先确定操作系统环境，不建议在 Windows 上面搞，如果你一定想在`Windows`上安装`Jekyll`

参考：[http://www.ymq.io/2017/07/22/Windows-install-Jekyll/](http://www.ymq.io/2017/07/22/Windows-install-Jekyll/)

 - Mac OS X
 - 任意 Linux 发行版本(Ubuntu,CentOS, Redhat, ArchLinux ...)
 
**强烈新手使用 Ubuntu 省掉不必要的麻烦！**

# 四、RVM 安装

RVM 是干什么的这里就不解释了，自行Google，这里所有的命令都是再用户权限下操作的，任何命令最好都不要用 `sudo`

```sh
gpg --keyserver hkp://keys.gnupg.net --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3
curl -sSL https://get.rvm.io | bash -s stable

# 如果上面的连接失败，可以尝试: 
curl -L https://raw.githubusercontent.com/wayneeseguin/rvm/master/binscripts/rvm-installer | bash -s stable
```

期间可能会问你 `sudo` 管理员密码，以及自动通过 `Homebrew` 安装依赖包，等待一段时间后就可以成功安装好 `RVM`。

然后，载入 `RVM` 环境（新开 `Termal` 就不用这么做了，会自动重新载入的）

```sh
source /usr/local/rvm/scripts/rvm
```

修改 RVM 的 Ruby 安装源到 Ruby China 的 Ruby 镜像服务器，这样能提高安装速度

```sh
echo "ruby_url=https://cache.ruby-china.org/pub/ruby" >> /usr/local/rvm/user/db
```

检查一下是否安装正确

```sh
$ rvm -v

rvm 1.29.3 (latest) by Michal Papis, Piotr Kuczynski, Wayne E. Seguin [https://rvm.io]
```

# 五、安装 Ruby

用 RVM 安装 Ruby 环境

```sh
$ rvm requirements
$ rvm install 2.3.0
```

等待漫长的下载，编译过程，完成以后，`Ruby, Ruby Gems` 就安装好了，国内速度很慢，国外服务器，不到一分钟就下载完了，文件大概100兆


设置 Ruby 版本,同样，也可以用其他版本号，前提是你有用 rvm install 安装过那个版本

```sh
rvm use 2.3.0 --default
```

这个时候你可以测试是否正确

```sh
$ruby -v
ruby 2.3.0p0 (2015-12-25 revision 53290) [x86_64-linux]

$gem -v
2.5.1
```

# 六、安装 Bundler

```sh
gem install bundler
```

# 七、搭建 Jekyll

搭建Jekyll博客,需要找一套主题模板，这里可以参考:[https://www.zhihu.com/question/20223939](https://www.zhihu.com/question/20223939) ,以下以 mzlogin.github.io 的主题为例


## 1、安装 Git

CentOS
```sh
yum install git
```

Ubuntu
```sh
apt install git
```

## 2、克隆主题

```sh
git clone https://github.com/mzlogin/mzlogin.github.io.git
```

## 3、安装 Jekyll

```sh
cd souyunku.github.io/
bundle install
```

## 4、启动 jekyll

```sh
jekyll serve -H 0.0.0.0 -P 80
```

**效果如下**
![访问服务][1]

## 5、报错解决

**Ubuntu 16.04**  

```sh
## Configuration file: /root/mzlogin.github.io/_config.yml
  Dependency Error: Yikes! It looks like you don't have jekyll-remote-theme or one of its dependencies installed. In order to use Jekyll as currently configured, you'll need to install this gem. The full error message from Ruby is: 'Could not open library 'libcurl': libcurl: cannot open shared object file: No such file or directory. Could not open library 'libcurl.so': libcurl.so: cannot open shared object file: No such file or directory. Could not open library 'libcurl.so.4': libcurl.so.4: cannot open shared object file: No such file or directory' If you run into trouble, you can find helpful resources at https://jekyllrb.com/help/! 
jekyll 3.6.2 | Error:  jekyll-remote-theme
```

执行

```sh
apt-get install libcurl3
```


# 八、jekyll 文档
[jekyll 基本用法 官方中文文档](http://jekyll.com.cn/docs/usage/)

# 九、博客使用指南


博客搭建成功之后，还需要做一些事情才能让你的页面「正确」跑起来。

**以下内容摘自 [码志](https://github.com/mzlogin/mzlogin.github.io/blob/master/README.md) 博客主题的，Fork 指南**

1. 正确设置项目名称与分支。

   按照 GitHub Pages 的规定，名称为 `username.github.io` 的项目的 master 分支，或者其它名称的项目的 gh-pages 分支可以自动生成 GitHub Pages 页面。

2. 修改域名。

   如果你需要绑定自己的域名，那么修改 CNAME 文件的内容；如果不需要绑定自己的域名，那么删掉 CNAME 文件。

3. 修改配置。

   网站的配置基本都集中在 \_config.yml 文件中，将其中与个人信息相关的部分替换成你自己的，比如网站的 url、title、subtitle 和第三方评论模块的配置等。

   **评论模块：** 目前支持 disqus、gitment 和 gitalk，选用其中一种就可以了，推荐使用 gitalk。它们各自的配置指南链接在 \_config.yml 文件的 Comments 一节里都贴出来了。

   **注意：** 如果使用 disqus，因为 disqus 处理用户名与域名白名单的策略存在缺陷，请一定将 disqus.username 修改成你自己的，否则请将该字段留空。我对该缺陷的记录见 [Issues#2][3]。

4. 删除我的文章与图片。

   如下文件夹中除了 template.md 文件外，都可以全部删除，然后添加你自己的内容。

   * \_posts 文件夹中是我已发布的博客文章。
   * \_drafts 文件夹中是我尚未发布的博客文章。
   * \_wiki 文件夹中是我已发布的 wiki 页面。
   * images 文件夹中是我的文章和页面里使用的图片。

5. 修改「关于」页面。

   pages/about.md 文件内容对应网站的「关于」页面，里面的内容多为个人相关，将它们替换成你自己的信息，包括 \_data 目录下的 skills.yml 和 social.yml 文件里的数据。

## 贴心提示

1. 排版建议遵照一定的规范，推荐 [中文文案排版指北（简体中文版）](https://github.com/mzlogin/chinese-copywriting-guidelines)。

2. 在本地预览博客效果可以参考 [Setting up your Pages site locally with Jekyll][2]。

## 经验与思考

* 简约，尽量每个页面都不展示多余的内容。

* 有时一图抵千言，有时可能只会拖慢网页加载速度。

* 言之有物，不做无痛之呻吟。

* 如果写技术文章，那先将技术原理完全理清了再开始写，一边摸索技术一边组织文章效率较低。

* 杜绝难断句、难理解的长句子，如果不能将其拆分成几个简洁的短句，说明脑中的理解并不清晰。

* 可以学习一下那些高质量的博主，他们的行文，内容组织方式，有什么值得借鉴的地方。

## 致谢作者

我的个人博客外观基于 [DONGChuan](http://dongchuan.github.io) 的修改，感谢 Zhuang Ma ！

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)   
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，专注于开发技术的研究与知识分享
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")


[1]: http://www.ymq.io/images/2017/jekyll/1.png
