---
title: Tips
comments: true
tags:
  - GAP
  - Tips
categories: tips
abbrlink: 5f09
date: 2019-12-22 15:34:17
---

学习路上总会遇到各种坑，之前一直搞完就算了，总觉得很可惜，就有了记录~~踩的各种坑~~的想法......

未完待续（不断填坑）中......<!-- More -->

## Git

1. 如果你add & commit过完看命令行发现自己智障了还好没push上去。

   > git reset --mixed HEAD^

2. 如果commit&push后error: src refspec matser does not match any.

   > git push --set-upstream origin master

## Python

1. 打开项目所在文件目录导出python库依赖requirements.txt

   > pipreqs ./ --encoding=utf-8

   这样之后有人需要用这个项目的时候直接打开命令行输入以下即可：

   > pip install -r requirements.txt

## Go

1. 添加 Go Modules Replace的时候，replace配置项的路径只能是`~`根目录开头，`.`当前目录开头，`..`上级目录开头，这点在WSL上就还是安利后两者，毕竟在根目录下VScode估计找不到文件夹orz

2. 为变量给予附属属性`json`，这样子在`c.JSON`(c *gin.Context)的时候就会自动转换格式，非常的便利。
   ```go
   Name string `json:"name"`
   ```

## Docker

1. Windows下用docker端口进行映射，但打开`localhost:port`无法访问对应的服务。

    因为docker是运行在Linux上的，在Windows中运行实际上还是在Windows下Linux环境中运行的docker。即服务中使用的localhost指的是这个Linux环境的地址，而非宿主环境Windows。我们可以通过命令`docker-machine ip defaul`找到这个Linux环境的IP地址，一般是**192.168.99.100**（比如我）

## MySQL

1. 如果是WSL的话，MySQL被设计为可以使用大写字母跨平台正常工作，其中数据库名称/表名和/或列为小写字母可能会给操作系统带来麻烦，并且可能会生成相同的二进制表文件。简单点，会出bug不建议WSL建议直接windows上MySQL，如果真的想WSL的话（比如我），当Cannot stat file /proc/1/fd/5: Operation not permitted时，不要慌，重启下bash然后输入：

   > sudo service mysql start
   >
   > sudo dpkg --configure -a
   
   然后你可能会像我一样发现Error: Access denied for user 'root'@'localhost'莫慌：
   
   > sudo mysql
   >
   > ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'root';
   
   这样再进入的时候usr:root, psd:root 就ok了
   
   > mysql -u root -p

   接下来就可以愉快地创建新用户，然后赋予并刷新权限了：
   
   > mysql >CREATE USER 'newuser'@'localhost' IDENTIFIED BY 'password';
   > 
   > mysql> GRANT ALL PRIVILEGES ON * . * TO 'newuser'@'localhost';
   > 
   > mysql >FLUSH PRIVILEGES;
   
2. 如果发现ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/var/run/mysqld/mysqld.sock' (2)那是因为没开服务，打开服务`sudo service mysql start`即可。

3. 用pymysql时候(pymysql.err.InternalError) (1366, "Incorrect string value: '\\xE6\\xAD\\xA3\\xE5\\xB8\\xB8'中文编码问题，修改即可。 
   
   > mysql> alter table recruit convert to character set utf8mb4; 
   
4. 用pymysql时候，想借MySQL函数`STR_TO_DATE`传DATE类型数据，记得年月日用%%而不是%：

   > sql = 'insert into recruit_info(title,time) VALUES (%s,STR_TO_DATE(%s, "%%Y-%%m-%%d"))'

## PostgreSQL

1. 打开*pgAdmin*如果出现无法连接并要求输入`postgres`密码的情况，请打开任务管理器>服务，找到并启动`postgresql-x64-10`即可。

## Redis

1. WSL下安装Redis的话，要记得修改/etc/redis/redis.conf这个文件，将`supervised no`修改为`supervised systemd`即可，ubuntu用`systemd`初始系统。
2. 如果连不上，第一件事就是启动服务试试，`sudo service redis-server start`

## Vue.js

1. 刚开始我在修改代码中途发现自己没下`vue-router`强停后npm install结果errno -4048 npm ERR! syscall rename，解决方法`npm cache clean --force`后npm install即可。

## Terminal

1. 用tree直接导出当前目录下的目录结构：

   > tree -L 2 > test.txt

2. 不想每次都输服务器密码（懒）怎么办：

   > ssh-copy-id USER_NAME@IP_ADDRESS