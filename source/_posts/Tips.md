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

学习路上总会遇到各种小坑和奇妙操作，之前一直搞完就算了总觉得很可惜，就有了记录~~踩的各种坑~~的想法......未完待续...<!-- More -->

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

## PostgreSQL

1. 打开*pgAdmin*如果出现无法连接并要求输入`postgres`密码的情况，请打开任务管理器>服务，找到并启动`postgresql-x64-10`即可。

## 命令行

1. 用tree直接导出当前目录下的目录结构：

   > tree -L 2 > test.txt

