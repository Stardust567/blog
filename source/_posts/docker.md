---
title: docker
comments: true
tags:
  - GAP
  - Docker
categories: Docker
abbrlink: c018
date: 2020-01-13 10:10:12
---

Docker 属于 Linux 容器的一种封装，提供简单易用的容器使用接口。它将应用程序与该程序的依赖，打包在一个文件里面。运行这个文件，就会生成一个虚拟容器。程序在这个虚拟容器里运行，就好像在真实的物理机上运行一样。<!-- More -->

## Docker

一般来说，先写Dockerfile文件并用此来创建image文件：

 ```bash
 $ docker image build -t demo:0.0.1
 ```

然后生成容器：

 ```bash
 $ docker container run -p 8000:3000 -it demo:0.0.1 /bin/bash
 ```

- `-p`参数：容器的 3000 端口映射到本机的 8000 端口。
- `-it`参数：容器的 Shell 映射到当前的 Shell，然后你在本机窗口输入的命令，就会传入容器。
- `demo:0.0.1`：image 文件的名字（如果有标签，还需要提供标签，默认是 latest 标签）。
- `/bin/bash`：容器启动以后，内部第一个执行的命令。这里是启动 Bash，保证用户可以使用 Shell。

不过Node 进程运行在 Docker 容器的虚拟环境里面，进程接触到的文件系统和网络接口都是虚拟的，与本机的文件系统和网络接口是隔离的，因此需要定义容器与物理机的端口映射（map）才能在本机localhost访问到。

 ```bash
 # 在本机的另一个终端窗口，查出容器的 ID
 $ docker container ls
 # 停止指定的容器运行
 $ docker container kill [containerID]
 ```

容器停止运行之后，并不会消失，用下面的命令删除容器文件。

 ```bash
 # 查出容器的 ID
 $ docker container ls --all
 # 删除指定的容器文件
 $ docker container rm [containerID]
 ```

也可以使用`docker container run`命令的`--rm`参数，在容器终止运行后自动删除容器文件。

 ```bash
 $ docker container run --rm -p 8000:3000 -it demo:0.0.1 /bin/bash
 ```

## Vue

对于Vue前端，Dockerfile位于项目的docker文件夹下，同时有nginx.conf配置文件。

### Dockerfile

`FROM nginx:alpine`

`COPY ./nginx.conf /etc/nginx/conf.d/default.conf`
`COPY ./dist /usr/share/nginx/html`

Docker 需要依赖于一个基础镜像进行构建运行容器，我们指定nginx的alpine版本作为基础镜像。使用`COPY` 指令从Docker主机复制内容和配置文件。不过记住，需要复制的目录一定要放在 Dockerfile 文件的同级目录下。

#### nginx.conf

> server {
>
>   listen 80;
>
>   location / {
>
>    root /usr/share/nginx/html;
>
>    index index.html;
>
>   }
>
> }

在80端口监听，访问`/`相当于访问容器的`/usr/share/nginx/html`

### shell

`cd ..`
`yarn build` 
`cd docker`
`mv ../dist ./dist`
`sudo docker login --username=stardust567 registry.cn-shanghai.aliyuncs.com`
`sudo docker build -t registry.cn-shanghai.aliyuncs.com/stardust567/infoweb:$1 ./`
`sudo docker push registry.cn-shanghai.aliyuncs.com/stardust567/infoweb:$1`

在根目录编译vue项目于默认的dist文件夹下并将dist文件夹移入docker文件夹下，创建镜像并push上阿里云（$1为标签名）

## Go

对于go后端，Dockerfile文件直接位于项目根目录下。

### Dockerfile

`FROM golang as build`

`ENV GOPROXY=https://goproxy.cn`

`ADD . /infoweb`

`WORKDIR /infoweb`

`RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -o api_server`

`FROM alpine:3.7`

`ENV REDIS_ADDR=""`
`ENV REDIS_PW=""`
`ENV REDIS_DB=""`
`ENV MysqlDSN=""`
`ENV GIN_MODE="release"`
`ENV PORT=3000`

`RUN echo "http://mirrors.aliyun.com/alpine/v3.7/main/" > /etc/apk/repositories && \`
  `apk update && \`
  `apk add ca-certificates && \`
  `echo "hosts: files dns" > /etc/nsswitch.conf && \`
  `mkdir -p /www/conf`

`WORKDIR /www`

`COPY --from=build /infoweb/api_server /usr/bin/api_server`
`ADD ./conf /www/conf`

`RUN chmod +x /usr/bin/api_server`

`ENTRYPOINT ["api_server"]`

将当前目录下文件拷入docker并作为工作目录，在linux环境下编译go文件生成api_server文件。

再生成一个基础镜像并初始化一下并创建拷贝conf文件夹，将上个镜像的api_server拷贝并赋予权限，最后在启动容器时执行api_server文件。