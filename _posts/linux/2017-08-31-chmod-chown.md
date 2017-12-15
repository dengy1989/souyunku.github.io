---
layout: post
title: linux chmod,chown命令详解
categories: Linux
description: linux chmod,chown命令详解
keywords: Linux
---

# linux chmod,chown命令详解

## 指令名称:`chmod` 
  
使用权限 : 所有使用者 
  
使用方式 : `chmod [-cfvR] [--help] [--version] mode file...`  
说明 : Linux/Unix 的档案存取权限分为三级 : 档案拥有者、群组、其他。

利用 `chmod` 可以藉以控制档案如何被他人所存取。  

`mode` 权限设定字串，格式如下 : `[ugoa...][[+-=][rwxX]...][,...]`  

`u`:表示该档案的拥有者  
`g`: 表示与该档案的拥有者属于同一个群体`(group)`者  
`o`:表示其他以外的人  
`a`: 表示这三者皆是  

`+`: 表示增加权限、- 表示取消权限、= 表示唯一设定权限。   
`r`:表示可读取，w 表示可写入，x 表示可执行，X 表示只有当该档案是个子目录或者该档案已经被设定过为可执行。 
`-c`: 若该档案权限确实已经更改，才显示其更改动作   
`-f` : 若该档案权限无法被更改也不要显示错误讯息   
`-v`: 显示权限变更的详细资料   
`-R`: 对目前目录下的所有档案与子目录进行相同的权限变更(即以递回的方式逐个变更)   
`--help`: 显示辅助说明   
`--version`: 显示版本  

### 示例1： 

将档案 `ymq.txt` 设为所有人皆可读取 :  

```sh
chmod ugo+r ymq.txt   
```

将档案 ymq.txt 设为所有人皆可读取 :

```sh
chmod a+r ymq.txt   
```

将档案 ymq1.txt 与 ymq2.txt 设为该档案拥有者，与其所属同一个群体者可写入，但其他以外的人则不可写入 :  

```sh
chmod ug+w,o-w ymq1.txt ymq2.txt  
```

将 ymq.py 设定为只有该档案拥有者可以执行 :  

```sh
chmod u+x ymq.py   
```

将目前目录下的所有档案与子目录皆设为任何人可读取 :  

```sh
chmod -R a+r *   
```

此外`chmod`也可以用数字来表示权限如 `chmod 777 file`   

语法为：`chmod abc file`   
其中`a,b,c`各为一个数字，分别表示`User`、`Group`、及`Other`的权限  

`r=4，w=2，x=1`   

若要`rwx`属性则`4+2+1=7`  
若要`rw-`属性则`4+2=6`   
若要`r-x`属性则`4+1=5`  


### 示例2：

```sh
chmod a=rwx file
#和
chmod 777 file  
#效果相同
chmod ug=rwx,o=x file
#和
chmod 771 file
#效果相同
```

若用`chmod 4755 filename`可使此程式具有`root`的权限

## 指令名称:`chown` 

使用权限 : `root`

使用方式 : `chown [-cfhvR] [--help] [--version] user[:group] file...` 
说明 : `Linux/Unix` 是多人多工作业系统，所有的档案皆有拥有者。利用 `chown` 可以将档案的拥
有者加以改变。一般来说，这个指令只有是由系统管理者(`root`)所使用，一般使用者没有权限可以
改变别人的档案拥有者，也没有权限可以自己的档案拥有者改设为别人。只有系统管理者(`root`)才
有这样的权限。

`user`: 新的档案拥有者的使用者 ID  
`group`: 新的档案拥有者的使用者群体(group)  
`-c` 或 `-change`：作用与-v相似，但只传回修改的部分   
`-f` 或 `–quiet或–silent`：不显示错误信息   
`-h` 或 `–no-dereference`：只对符号链接的文件做修改，而不更改其他任何相关文件   
`-R` 或 `-recursive`：递归处理，将指定目录下的所有文件及子目录一并处理   
`-v` 或 `–verbose`：显示指令执行过程   
`–dereference`：作用和-h刚好相反   
`–help`：显示在线说明   
`–reference=<参考文件或目录>`：把指定文件或目录的所有者与所属组，统统设置成和参考文件或目录的所有者与所属组相同   
`–version`：显示版本信息  


### 示例1：
 
将档案 `file1.txt` 的拥有者设为 `users` 群体的使用者 `jessie` :  

```sh
chown jessie:users file1.txt   
```

将目前目录下的所有档案与子目录的拥有者设为

`chown -R ymq`(所属用户) `:` `ymqgroup`(所属用户组名) `*` (要更改的文件路径)

`chown [-R] [用户名称] [文件或目录]`  
`chown [-R] [用户名称:组名称] [文件或目录]`  

```sh
chown -R ymq:ymqgroup *   
```

`rwx`分别表示`User`、`Group`、及`Other`的权限。 

`-rw-------` `(`600`) -- 只有属主有读写权限   
`-rw-r--r--` `(`644`) -- 只有属主有读写权限；而属组用户和其他用户只有读权限   
`-rwx------` `(`700`) -- 只有属主有读、写、执行权限   
`-rwxr-xr-x` `(`755`) -- 属主有读、写、执行权限；而属组用户和其他用户只有读、执行权限   
`-rwx--x--x` `(`711`) -- 属主有读、写、执行权限；而属组用户和其他用户只有执行权限    
`-rw-rw-rw-` `(`666`) -- 所有用户都有文件读、写权限。这种做法不可取    
`-rwxrwxrwx` `(`777`) -- 所有用户都有读、写、执行权限。更不可取的做法   


### 示例2：

将`test3.txt`文件的属主改为test用户。

```sh
# ls -l test3.txt  
-rw-r–r– 1 test root 0 2017-08-20 9:59 test3.txt  
# chown test:root test3.txt  
# ls -l test3.txt  
-rw-r–r– 1 test root 0 2017-08-20 9:59  
```

### 示例3：

`chown`所接的新的属主和新的属组之间可以使用:连接，属主和属组之一可以为空。如果属主为空，应该是“:属组”；如果属组为空，“:”可以不用带上。

```sh
# ls -l test3.txt  
-rw-r–r– 1 test root 0 2017-08-20 9:59 test3.txt  
  
# chown :test test3.txt <==把文件test3.txt的属组改为test  
# ls -l test3.txt  
-rw-r–r– 1 test test 0 2017-08-20 9:59 test3.txt  
```


### 示例4：

`chown`也提供了`-R`参数，这个参数对目录改变属主和属组极为有用，可以通过加 `-R`参数来改变某个目录下的所有文件到新的属主或属组。 

```
# ls -l testdir <== 查看testdir目录属性  
drwxr-xr-x 2 usr root 0 2009-10-56 10:38 testdir/ <==文件属主是usr用户，属组是 root用户  
# ls -lr testdir <==查看testdir目录下所有文件及其属性  
total 0  
-rw-r–r– 1 usr root 0 2017-08-20 10:38 test1.txt  
-rw-r–r– 1 usr root 0 2017-08-20 10:38 test2.txt  
-rw-r–r– 1 usr root 0 2017-08-20 10:38 test3.txt  
# chown -R test:test testdir/ <==修改testdir及它的下级目录和所有文件到新的用户和用户组  
# ls -l testdir  
drwxr-xr-x 2 test test 0 2017-08-20 10:38 testdir/  
# ls -lr testdir  
total 0  
-rw-r–r– 1 test test 0 2017-08-20 10:38 test1.txt  
-rw-r–r– 1 test test 0 2017-08-20 10:38 test2.txt  
-rw-r–r– 1 test test 0 2017-08-20 10:38 test3.txt  
```


# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")