layout: post
title: dockerfile指令介绍
author: Pyker
categories: docker
tags:
  - docker
date: 2018-03-21 23:12:00
---

# 什么是Dockerfile
Docker的镜像是通过对于在Dockerfile中的一系列指令的顺序解析实现自动的image的构建，通过使用`docker build`命令，根据Dockerfile的描述来构建镜像。Dockerfile指令只支持Docker自己定义的一套指令，不支持自定义指令，且大小写不敏感，但是建议全部使用大写，构建命令如：
```bash
# 分别为Dockerfile当前路径和指定Dockerfile路径
$ docker build -t image名称:tag标签 ./
$ docker build -t image名称:tag标签 -f <Dockerfile路径>
```
<div class="note danger"><p>注意：不要使用根("/")目录作为Dockfile PATH，因为它会导致构建将硬盘驱动器的全部内容传输到Docker守护程序。</p></div>

# BuildKit 
从18.09版本开始，Docker支持一种后端程序用于运行你的构建，它可以被moby/buildkit project提供。这个 BuildKit backend 提供很多与旧版本的好处。如：
* 捕获并跳过执行未使用的构建阶段
* 并行构建独立地阶段
* 在构建环境中，加强传输在构建阶段更改的文件
* 捕获并跳过传输没有用的文件
* 使用外部Dockerfile实现很多新的特性
* 避免REST API的副用作影响（中间image和container）
* 通过自动优化来提升构建缓存

要使用BuildKit后端，需要在调用docker build之前在CLI上设置环境变量`DOCKER_BUILDKIT = 1`。要了解基于BuildKit的构建可用的实验性Dockerfile语法，请参阅[BuildKit存储库中的文档](https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/experimental.md)。

# 基本结构
Dockerfile由一行行命令语句组成，并支持以#开头的注释行。
```bash
# This dockerfile uses the ubuntu image
# VERSION 2 - EDITION 1
# Author: docker_user
# Command format: Instruction [arguments / command ] ..

# Base image to use, this nust be set as the first line
FROM ubuntu

# Maintainer: docker_user <docker_user at email.com> (@docker_user)
MAINTAINER docker_user docker_user@email.com

# Commands to update the image
RUN echo "deb http://archive.ubuntu.com/ubuntu/ raring main universe" >> /etc/apt/sources.list
RUN apt-get update && apt-get install -y nginx
RUN echo "\ndaemon off;" >> /etc/nginx/nginx.conf

# Commands when creating a new container
CMD /usr/sbin/nginx
```
其中，开始必须指明所基于的镜像名称，接下来一般是说明维护者信息。后面则是镜像操作指令，例如RUN指令，RUN指令将对镜像执行跟随的命令。每运行一条RUN指令，镜像就添加新的一层，并提交。最后是CMD指令，用来指定运行容器时的操作命令。

在docker官网上有这样两个例子：
* 在debian:jessie基础镜像上安装nginx环境，从而创建一个新的nginx镜像：

```bash
FROM debian:jessie

MAINTAINER NGINX Docker Maintainers "docker-maint@nginx.com"

ENV NGINX_VERSION 1.10.1-1~jessie

RUN apt-key adv --keyserver hkp://pgp.mit.edu:80 --recv-keys 573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62 && \
echo "deb http://nginx.org/package/debian/ jessie nginx" >> /etc/apt/source.list && apt-get update && \
apt-get install --no-install-recommends --no-install-suggests -y ca-certificates nginx=$(NGINX_VERSION) \
nginx-module-xslt nginx-module-geoip nginx-module-image-filter nginx-module-perl nginx-module-njs gettext-base && \
rm -rf /var/lib/apt/lists/*

# forward request and error logs to docker log collector
RUN ln -sf /dev/stdout /var/log/nginx/access.log && ln -sf /dev/stderr /var/log/nginx/err.log

EXPOSE 80 443

CMD ["nginx","-g","daemon off;"]
```
* 基于buildpack-deps:jessie-scm基础镜像，安装golang相关环境，制作一个GO语言的运行环境。

```bash
FROM buildpack-deps:jessie-scm

# gcc fo cgo
RUN apt-get update && apt-get install -y --no-install-recommends g++ gcc libc6-dev make && rm -rf /var/lib/apt/lists*

ENV GOLANG_VERSION 1.6.3
ENV GOLANG_DOWNLOAD_RUL https://golang.org/dl/go$GOLANG_VERSION.linux-amd64.tar.gz
ENV GOLANG_DOWNLOAD_SHA256 cdd5e08530c0579255d6153b08fdb3b8e47caabbe717bc7bcd7561275a87aeb

RUN curl -fssL "$GOLANG_DOWNLOAD_RUL" -o golang.tar.gz && \
echo "$GOLANG_DOWNLOAD_SHA256 golang.tar.gz" | sha256sum -c - && tar -C /usr/local -xzf golang.tar.gz && rm golang.tar.gz

ENV GOPATH $GOPATH/bin:/usr/local/go/bin:$PATH

RUN mkdir -p "$GOPATH/bin" && chmod -R 777 "$GOPATH"
WORKDIR $GOPATH

COPY go-wrapper /usr/local/bin
```
# 指令介绍
指令的一般格式为`INSTRUNCTION arguments`，指令包括FROM、MAINTAINER、RUN等。具体指令及说明如下：
* {% label primary@FROM %}
指定所创建的镜像的基础镜像，如果本地不存在，则默认会去Docker Hub下载指定镜像。
格式为：`FROM <image> [AS <name>]，或FROM<image>:<tag> [AS <name>]，或FROM<image>@<digest> [AS <name>]。`
任何Dockerfile中的第一条指令`必须为FROM指令`。并且，如果在同一个Dockerfile文件中创建多个镜像，可以使用多个FROM指令(每个镜像一次)。

* {% label primary@MAINTAINER %}
指定维护者信息，格式为MAINTAINER<name>。(已弃用，但仍兼容，使用LABEL代替) 例如：
```bash
MAINTAINER pyker <pyker@qq.com>
```
	该信息将会写入生成镜像的Author属性域中。可以通过docker inspect image名称查看。

* {% label primary@RUN %}
运行指定命令，如果有多个命令关联，建议用&&符号连接。格式为：`RUN <command>或RUN ["executable","param1","param2"]`。
<div class="note info"><p>注意：后一个指令会被解析为json数组，所以必须使用双引号。</p></div>

	前者默认将在shell终端中运行命令，即`/bin/sh -c`；后者则使用exec执行，不会启动shell环境。指定使用其他终端类型可以通过第二种方式实现，例如：`RUN ["/bin/bash","-c","echo hello"]` , 每条RUN指令将在FROM指定的镜像的基础上执行指定命令，并提交为新的镜像。当命令较长时可以使用\换行。例如：
	```bash
	RUN apt-get update \
        && apt-get install -y libsnappy-dev zliblg-dev libbz2-dev \
        && rm -rf /var/cache/apt
	```

* {% label primary@CMD %}
CMD指令用来指定启动容器时默认执行的命令。它支持三种格式：
```bash
1.CMD ["executable","param1","param2"] 使用exec执行，是推荐使用的方式；
2.CMD param1 param2 在/bin/sh中执行，提供给需要交互的应用；
3.CMD ["param1","param2"] 提供给ENTRYPOINT的默认参数。
```
	每个Dockerfile只能有一条CMD命令。如果指定了多条命令，只有最后一条会被执行。如果用户在docker run启动容器时指定了运行的命令(作为run的参数)，则会覆盖掉CMD指定的命令。

* {% label primary@LABEL %}
LABEL指令用来生成用于生成镜像的元数据的标签信息。格式为：`LABEL <key>=<value> <key>=<value> ...`。例如：
```bash
LABEL version="1.0"
LABEL description="This text illustrates \ that label-values can span multiple lines."
```

* {% label primary@EXPOSE %}
声明镜像内服务所监听的端口。格式为：`EXPOSE <port> [<port>...]`，例如：
```bash
EXPOSE 22 80/tcp 443/tcp 3306
```
	<div class="note info"><p>注意：该命令只是起到声明的作用，并不会自动完成端口映射。在容器启动时需要使用-P(大写P)，Docker主机会自动分配一个宿主机未被使用的临时端口转发到指定的端口；使用-p(小写p)，则可以具体指定哪个宿主机的本地端口映射过来。</p></div>


* {% label primary@ENV %}
指定环境变量，在镜像生成过程中会被后续RUN指令使用，使用该镜像启动的容器中也会存在。格式为：`ENV <key> <value>或ENV <key>=<value> ...`。例如：
```bash
ENV GOLANG_VERSION 1.6.3
ENV GOLANG_DOWNLOAD_RUL https://golang.org/dl/go$GOLANG_VERSION.linux-amd64.tar.gz
ENV GOLANG_DOWNLOAD_SHA256 cdd5e08530c0579255d6153b08fdb3b8e47caabbe717bc7bcd7561275a87aeb

RUN curl -fssL "$GOLANG_DOWNLOAD_RUL" -o golang.tar.gz && echo "$GOLANG_DOWNLOAD_SHA256 golang.tar.gz" | sha256sum -c - && tar -C /usr/local -xzf golang.tar.gz && rm golang.tar.gz

ENV GOPATH $GOPATH/bin:/usr/local/go/bin:$PATH

RUN mkdir -p "$GOPATH/bin" && chmod -R 777 "$GOPATH"
```
	指令指定的环境变量在运行时可以被覆盖掉，如`docker run --env <key>=<value> built_image`。

* {% label primary@ADD %}
该指令将复制指定的`<src>`路径下的内容到容器中的`<dest>`路径下。格式为：`ADD <src>... <dest>` 或 `ADD ["<src>",... "<dest>"]`其中`<src>`可以使Dockerfile所在目录的一个相对路径(文件或目录)，也可以是一个URL，还可以是一个tar文件(如果是本地tar文件，会自动解压到`<dest>`路径下)。`<dest>`可以使镜像内的绝对路径，或者相当于工作目录(WORKDIR)的相对路径。路径支持正则表达式，例如：
```bash
ADD *.c /code/
```
* {% label primary@COPY %}
复制本地主机的`<src>`(为Dockerfile所在目录的一个相对路径、文件或目录)下的内容到镜像中的`<dest>`下。目标路径不存在时，会自动创建。路径同样支持正则。格式为：`COPY <src> <dest>`或 `["<src>",... "<dest>"]`当使用本地目录为源目录时，推荐使用COPY。

* {% label primary@ENTRYPOINT %}
指定镜像的默认入口命令，该入口命令会在启动容器时作为根命令执行，所有传入值作为该命令的参数（包括在docker run执行的命令都将成为参数）。支持两种格式：
```bash
1.ENTRYPOINT ["executable","param1","param2"] (exec调用执行)；
2.ENTRYPOINT command param1 param2 (shell中执行)。
```
	此时，CMD指令指定值将作为ENTRYPOINT命令的参数。每个Dockerfile中只能有一个ENTRYPOINT，当指定多个时，只有最后一个有效。在docker run时可以被--entrypoint参数覆盖掉。

* {% label primary@VOLUME %}
创建一个数据卷挂载点。式为：`VOLUME ["/data",...]`, 可以从本地主机或者其他容器挂载数据卷，一般用来存放数据库和需要保存的数据等。

* {% label primary@USER %}
指定运行容器时的用户名或UID，后续的RUN等指令也会使用特定的用户身份。格式为：`USER daemon`,  当服务不需要管理员权限时，可以通过该指令指定运行用户，并且可以在之前创建所需要的用户。例如：
```bash
RUN groupadd -r nginx && useradd -r -g nginx nginx
```

* {% label primary@WORKDIR %}
为后续的RUN、CMD和ENTRYPOINT指令配置工作目录。格式为：`WORKDIR /path/to/workdir`。可以使用多个WORKDIR指令，后续命令如果参数是相对的，则会基于之前命令指定的路径。例如：
```bash
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```
	则最终路径为/a/b/c

* {% label primary@ARG %}
指定一些镜像内使用的参数(例如版本号信息等)，这些参数在执行docker build命令时才以`--build-arg <varname>=<value>`格式传入。格式为：`ARG <name>=<value>...`。可以用`docker build --build-arg<name>=<value>`来覆盖Dockefile指定的参数值。

* {% label primary@ONBUILD %}
在配置当所创建的镜像作为其他镜像的基础镜像的时候，所执行创建操作指令。格式为：`ONBUILD [INSTRUCTION]`。例如Dockerfile使用如下的内容创建了镜像image-A：
```bash
[...]
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
[...]
```
	如果基于image-A镜像创建新的镜像时，新的Dockerfile中使用FROM image-A指定基础镜像，会自动执行ONBUILD指令的内容，等价于创建新的镜像时在后面添加了两条指令：
	```bash
	FROM image-A

	# Automatically run the following
	ONBUILD ADD . /app/src
	ONBUILD RUN /usr/local/bin/python-build --dir /app/src
	```
	使用ONBUILD指令的镜像，推荐在标签中注明，例如：ruby:1.9-onbuild。 ONBUILD指令在基础镜像中不会执行。

* {% label primary@STOPSIGNAL %}
指定所创建镜像启动的容器接收退出的信号值。例如：
```bash
STOPSIGNAL singnal
```

* {% label primary@HEALTHCHECK %}
配置所启动容器如何进行健康检查(如何判断是否健康)，自Docker 1.12开始支持。格式有两种：
```bash
1.HEALTHCHECK [OPTIONS] CMD command    ：根据所执行命令返回值是否为0判断；
2.HEALTHCHECK NONE    　　　　　　　　　　:禁止基础镜像中的健康检查。
```
	[OPTION]支持：
	```bash
	1.--inerval=DURATION  (默认为：30s)：多久检查一次；
	2.--timeout=DURATION  (默认为：30s)：每次检查等待结果的超时时间；
	3.--retries=N 　　     (默认为：3)：如果失败了，重试几次才最终确定失败。
	```
    例如：
    ```bash
    HEALTHCHECK --interval=1m --timeout=3s CMD curl -f http://localhost/ || exit 1
    ```

* {% label primary@SHELL %}
SHELL指令允许覆盖用于命令SHELL形式的默认SHELL。格式为： `SHELL ["executable","parameters"]`默认值为 `["bin/sh","-c"]`。Linux上的默认shell是`["bin/sh","-c"]`,Windows上是["cmd", "/S", "/C"]。SHELL指令必须以JSON格式写入Dockerfile
>注意：对于Windows系统，建议在Dockerfile开头添加# escape=`来指定转移信息。例如：

    ```bash
    # escape=`
    
    FROM microsoft/nanoserver
    SHELL ["powershell","-command"]
    RUN New-Item -ItemType Directory C:\Example
    ADD Execute-MyCmdlet.ps1 c:\example\
    RUN c:\example\Execute-MyCmdlet -sample 'hello world'
    ```

# 用法
## 创建镜像
编写完Dockerfile之后，可以通过docker build命令来创建镜像。基本的`docker build [选项] 内容路径`，该命令将读取指定路径下(包括子目录)的Dockerfile，并将该路径下的所有内容发送给Docker服务端，由服务端来创建镜像。因此除非生成镜像需要，否则一般建议放置Dockerfile的目录为空目录。
<div class="note primary"><p> * 1. 如果使用非当前路径下的Dockerfile，可以通过-f选项来指定其路径。<Br />
* 2. 要指定生成镜像的标签信息，可以使用-t选项。</p></div>

例如：指定Dockerfile所在路径为 /tmp/docker_builder/，并且希望生成镜像标签为build_repo:first_image，可以使用下面的命令：
```bash
docker build -t build_repo:first_image /tmp/docker_builder
```

## 使用 .dockerignore文件
.dockerignore文件可以想github的.gitingore文件一样，可以通过 .dockeringore文件(每一行添加一条匹配模式)来让Docker忽略匹配模式路径下的目录和文件。例如：
```bash
# comment
*/tmp*
*/*/tmp*
tmp?
~*
```

## Dockerfile编写小结
从需求出发，定制适合自己需求、高效方便的镜像，可以参考他人优秀的Dockerfile文件，在构建中慢慢优化Dockerfile文件：
```
1.精简镜像用途：                 尽量让每个镜像的用途都比较集中、单一，避免构造大而复杂、多功能的镜像；
2.选用合适的基础镜像：            过大的基础镜像会造成构建出臃肿的镜像，一般推荐比较小巧的镜像作为基础镜像；
3.提供详细的注释和维护者信息：     Dockerfile也是一种代码，需要考虑方便后续扩展和他人使用；
4.正确使用版本号：               使用明确的具体数字信息的版本号信息，而非latest，可以避免无法确认具体版本号，统一环境；
5.减少镜像层数：                减少镜像层数建议尽量合并RUN指令，可以将多条RUN指令的内容通过&&连接；
6.及时删除临时和缓存文件：        这样可以避免构造的镜像过于臃肿，并且这些缓存文件并没有实际用途；
7.提高生产速度：                合理使用缓存、减少目录下的使用文件，使用.dockeringore文件等；
8.调整合理的指令顺序：           在开启缓存的情况下，内容不变的指令尽量放在前面，这样可以提高指令的复用性；
9.减少外部源的干扰：             如果确实要从外部引入数据，需要制定持久的地址，并带有版本信息，让他人可以重复使用而不出错。
```












