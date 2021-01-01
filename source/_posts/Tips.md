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
   
2. 解决matplotlib中文图例乱码问题

   > plt.rcParams['font.sans-serif']=['SimHei']     # 用来正常显示中文标签   
   > plt.rcParams['axes.unicode_minus']=False  # 用来正常显示负号

3. 想判断x类型的时候，建议用`isinstance(x, 预判类型)`而不用`type(x)==预判类型`因为isinstance可以判断子类对象是否继承于父类，而type不可以。不过对于基本类型而言，type还是很好用的。

   ```python
   >>> student = Student("Jone", "UESTC") # class Student(Person)
   >>> type(Student)==type(Person) # True
   >>> isinstance(Person, Student) # False
   
   >>> isinstance(student, Student) # True
   
   >>> issubclass(Person, Student) # False
   >>> issubclass(Student, Person) # True
   ```
4. 做大量数据循环的时候建议使用xrange而非range，xrange本质是个生成器，而range是个list，当数据量很大的时候，range需要提前开辟一堆内存空间。
5. Python无明确的公共、私有或受保护的关键字来定义成员函数或属性，而是使用“_"和"__"来标识。

   1. 单下划线的函数或属性，在类中可以调用和访问，类的实例可以直接访问，子类中可以访问；
   2. 双下划线的函数或属性，在类中可以调用和访问，类的实例不可以直接访问，子类不可访问。
   
   但是带`__`的还有内建方法，python中所有类默认继承object类，我们可以重写object类的原始属性和方法，比如在类中重写`def __str__(self):`后，再print这个类时会直接调用重写后的方法。

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
   
5. revoke时候出现ERROR 1141 (42000)是因为当初你怎么grant的就必须怎么revoke回来，无法有修改。

6. mysql默认root账户无法远程登录，哪怕你改名改密码也不行，会出现ERROR 1045 (28000) 

## PostgreSQL

1. 打开*pgAdmin*如果出现无法连接并要求输入`postgres`密码的情况，请打开任务管理器>服务，找到并启动`postgresql-x64-10`即可。

## Redis

1. WSL下安装Redis的话，要记得修改/etc/redis/redis.conf这个文件，将`supervised no`修改为`supervised systemd`即可，ubuntu用`systemd`初始系统。
2. 如果连不上，第一件事就是启动服务试试，`sudo service redis-server start`

## Vue.js

1. 刚开始我在修改代码中途发现自己没下`vue-router`强停后npm install结果errno -4048 npm ERR! syscall rename，解决方法`npm cache clean --force`后npm install即可。

## Hadoop

Hadoop启动没有ResourceManager，查看log发现了报错`NoClassDefFoundError: javax/activation/DataSource`，直接下载activation-1.1.1.jar到lib目录下，或者本地上传到${HADOOP_HOME}/share/hadoop/yarn/lib目录下后重新启动start-yarn.sh即可。

```shell
cd ${HADOOP_HOME}/share/hadoop/yarn/lib
wget https://repo1.maven.org/maven2/javax/activation/activation/1.1.1/activation-1.1.1.jar
```

## Terminal

1. 用tree直接导出当前目录下的目录结构：
   ```shell
   tree -L 2 > test.txt
   ```
   
2. 不想每次都输服务器密码（懒）怎么办：
   ```shell
   ssh-copy-id USER_NAME@IP_ADDRESS
   ```
   
3. linux用密钥登录出现 *Permissions 0777 are too open.* 时可以加个`sudo`
   ```shell
   sudo ssh -i keyfile <user>@ip
   ```
   
4. Ctrl+C 使当前命令Stopped并置于后台。`jobs`查看后台命令。`fg %1`使后台命令1在前台继续执行，`bg %1`使后台命令1在后台继续执行，相当于加上`&`。

5. `&`和`nohup`都可以使命令后台执行，nohup即no hang up，在用户退出后依旧可以继续执行。

6. linux用vim忘记sudo后无法退出，强制退出用`:q!`但还是先另存为`:w new_filename`好一点。

7. 网易云音乐，`npx @nondanee/unblockneteasemusic -p 16300`然后客户端设置代理localhost:16300

8. 找寻相关文件的命令，比如`find /hadoop -name "*hadoop*.log"`在hadoop文件夹下寻找相关log

