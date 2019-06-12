layout: post
title: docker三剑客之docker-compose用法
author: Pyker
categories: docker
tags:
  - docker
date: 2018-03-22 18:12:00
---

# docker-compose简介
`docker-compose`项目是Docker官方的开源项目，负责实现对Docker容器集群的快速编排。它是一个定义和运行多容器的docker应用工具。使用docker-compose，你能通过YMAL文件配置你自己的服务，然后通过一个命令，你能使用配置文件创建和运行所有的服务。docker-compose的定位是 “ 定义和运行多个Docker容器应用的工具 ”，docker-compose开源代码地址：https://github.com/docker/compose 。

我们知道在Docker中构建自定义的镜像是通过使用`Dockerfile`模板文件来实现的，从而可以让用户很方便定义一个单独的应用容器。而docker-compose使用的模板文件就是一个YAML格式文件，它允许用户通过一个docker-compose.yml来定义`一组相关联的应用容器`为一个项目(project)。

**Compose 中有两个重要的概念：**
* `服务(service)`: 一个应用的容器，实际上可以包括若干运行相同镜像的容器实例
* `项目(project)`: 由一组关联的应用容器组成的一个完整业务单元，在docker-compose.yml中定义

以上可以理解为：</Br>
`服务（service）`就是在定义应用需要的一些服务，代表配置文件中的每一项服务。每个服务都有自己的名字、使用的镜像、挂载的数据卷、所属的网络、依赖哪些其他服务等等，即以容器为粒度，用户需要Compose所完成的任务。

`项目（project）`代表用户需要完成的一个项目，即是Compose的一个配置文件可以解析为一个项目，即Compose通过分析指定配置文件，得出配置文件所需完成的所有容器管理与部署操作。

Compose的默认管理对象时项目，通过子命令对项目中的一组容器进行便捷地生命周期管理。

# docker-compose安装
二进制安装：
```bash
$ curl -L https://github.com/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```
> 更多系统环境安装请参考[docker-compose仓库](https://github.com/docker/compose/releases)。

# docker-compose命令
在我们使用Compose前，可以通过执行`docker-compose --help`来查看Compose基本命令用法。
```bash
$ docker-compose --help
Define and run multi-container applications with Docker.
使用Docker定义和运行多容器应用程序

Usage:
  docker-compose [-f <arg>...] [options] [COMMAND] [ARGS...]
  docker-compose -h|--help

Options:
  -f, --file FILE             指定模板文件，默认是docker-compose.yml模板文件,可以多次指定
  -p, --project-name NAME     指定项目名称，默认使用所在目录名称作为项目名称
  --verbose                   输入更多的调试信息
  --log-level LEVEL           设置日志等级，有 (DEBUG, INFO, WARNING, ERROR, CRITICAL)
  --no-ansi                   不打印ANSI控制字符
  -v, --version               打印docker-compose版本
  -H, --host HOST             要连接到的守护进程套接字

  --tls                       Use TLS; implied by --tlsverify
  --tlscacert CA_PATH         Trust certs signed only by this CA
  --tlscert CLIENT_CERT_PATH  Path to TLS certificate file
  --tlskey TLS_KEY_PATH       Path to TLS key file
  --tlsverify                 Use TLS and verify the remote
  --skip-hostname-check       Don't check the daemon's hostname against the
                              name specified in the client certificate
  --project-directory PATH    Specify an alternate working directory
                              (default: the path of the Compose file)
  --compatibility             If set, Compose will attempt to convert keys
                              in v3 files to their non-Swarm equivalent

Commands:
  build              Build or rebuild services (构建或重建项目中的服务容器)
  bundle             Generate a Docker bundle from the Compose file (从Compose文件生成分布式应用程序包)
  config             Validate and view the Compose file (验证并查看docker-compose.yml文件)
  create             Create services (为服务创建容器)
  down               Stop and remove containers, networks, images, and volumes (停止容器并删除由其创建的容器，网络，卷和镜像)
  events             Receive real time events from containers (为项目中的每个容器流式传输容器事件)
  exec               Execute a command in a running container (这相当于docker exec。在服务中运行任意命令)
  help               Get help on a command (获得一个命令的帮助)
  images             List images (显示项目中当前所有镜像)
  kill               Kill containers (通过发送SIGKILL信号来强制停止服务容器)
  logs               View output from containers (查看服务容器的输出)
  pause              Pause services (暂停一个容器)
  port               Print the public port for a port binding (打印某个容器端口所映射的公共端口)
  ps                 List containers (列出项目中目前所有的容器)
  pull               Pull service images (拉取服务依赖镜像)
  push               Push service images (推送服务镜像)
  restart            Restart services (重启项目中的服务)
  rm                 Remove stopped containers (删除所有停止状态的服务容器)
  run                Run a one-off command (在指定服务上执行一个命令，会生成一个退出的容器，docker ps -a查看)
  scale              Set number of containers for a service (设置指定服务执行的容器个数，只在swarm集群有效)
  start              Start services (启动已存在的服务容器)
  stop               Stop services (停止正在运行的服务容器)
  top                Display the running processes (显示容器正在运行的进程)
  unpause            Unpause services (恢复处于暂停状态的容器)
  up                 Create and start containers (自动完成包括构建镜像、创建服务、启动服务并关联服务相关容器的一系列操作)
  version            Show the Docker-Compose version information (显示compose及依赖的环境当前版本)
```
> 更多命令的子命令可以使用`docker-compose command --help`格式查看。也可以查看[官方说明](https://docs.docker.com/compose/reference/build/)

这里主要说一下docker-compose run指令：
```bash
$ docker-compose run ubuntu ping www.baidu.com
```
将会新启动一个ubuntu容器，并执行`ping www.baidu.com` 命令。默认情况下，如果该服务存在关联，则所有关联的服务将会自动被启动，除非这些服务已经在运行中或者指定`--no-deps`参数。该命令类似于启动容器后运行指定的命令，相关卷、链接等都会按照配置自动创建。有两个不同点：
* 给定命令将会覆盖原有的自动运行命令
* 不会自动创建端口，以避免冲突

```bash
$ docker-compose run --help
Run a one-off command on a service.

For example:

    $ docker-compose run web python manage.py shell

By default, linked services will be started, unless they are already
running. If you do not want to start linked services, use
`docker-compose run --no-deps SERVICE COMMAND [ARGS...]`.

Usage:
    run [options] [-v VOLUME...] [-p PORT...] [-e KEY=VAL...] [-l KEY=VALUE...]
        SERVICE [COMMAND] [ARGS...]

Options:
    -d, --detach          Detached mode: Run container in the background, print
                          new container name. (在后台运行服务容器)
    --name NAME           Assign a name to the container (为容器指定一个名字)
    --entrypoint CMD      Override the entrypoint of the image. (覆盖默认的容器启动指令)
    -e KEY=VAL            Set an environment variable (can be used multiple times) (设置环境变量值，可多次使用选项来设置多个环境变量)
    -l, --label KEY=VAL   Add or override a label (can be used multiple times) (添加或覆盖标签可以多次使用)
    -u, --user=""         Run as specified username or uid (指定运行容器的用户名或者uid)
    --no-deps             Don't start linked services. (不自动启动管理的服务容器)
    --rm                  Remove container after run. Ignored in detached mode. (运行命令后自动删除容器，d模式下将忽略)
    -p, --publish=[]      Publish a container's port(s) to the host (映射容器端口到本地主机)
    --service-ports       Run command with the service's ports enabled and mapped
                          to the host. (配置服务端口并映射到本地主机)
    --use-aliases         Use the service's network aliases in the network(s) the
                          container connects to. (在容器连接到的网络中使用服务的网络别名)
    -v, --volume=[]       Bind mount a volume (default []) (绑定一个数据卷，默认为空)
    -T                    Disable pseudo-tty allocation. By default `docker-compose run`
                          allocates a TTY. (不分配伪tty，意味着依赖tty的指令将无法运行)
    -w, --workdir=""      Working directory inside the container (为容器指定默认工作目录)
```

# docker-compose模版文件
模板文件是使用Compose的核心，涉及的指令关键字也比较多，大部分指令与docker run相关参数的含义都是类似的。默认的模板文件迷城为docker-compose.yml，格式为YAML格式。
> 我们这里主要说明的是3.x版本，当然你也可以参考[官方配置说明](https://docs.docker.com/compose/compose-file/)查看更多版本模块文件

首先我们要知道compose模版文件主要分为4个区域，分别是：
* `version`
    compose版本，指定你当前使用的 compose 文件版本
* `services`
    服务，在它下面可以定义应用需要的一些服务，每个服务都有自己的名字、使用的镜像、挂载的数据卷、所属的网络、依赖哪些其他服务等等。
* `volumes`
    数据卷，在它下面可以定义的数据卷（名字等等），然后挂载到不同的服务下去使用。
* `networks`
    应用的网络，在它下面可以定义应用的名字、使用的网络类型等等。
> 注意：每个服务都必须通过image指令指定镜像或build指令（需要Dockerfile）等来自动构建生成镜像。如果使用build指令，在Dockefile中设置的选项（例如：CMD、EXPOSE、VOLUME、ENV等）将会自动被获取，无需在docker-compose.yml中再次设置。

## image
指定image的ID，这个image ID可以是本地也可以是远程的，如果本地不存在，compose会尝试pull下来。如：
```
image: 1.17.0-alpine
```

## build
服务除了可以基于指定的镜像，还可以基于一份 Dockerfile，在使用 up 启动之时执行构建任务，这个构建标签就是 build，它可以指定 Dockerfile 所在文件夹的路径。Compose 将会利用它自动构建这个镜像，然后使用这个镜像启动服务容器。例如：
```
# 绝对路径
build: /path/to/build/dir

# 相对路径
build: ./dir
```
> 在swarm模式下使用（版本3）Compose文件部署stack时，将忽略此选项。 docker stack命令仅接受预先构建好的的镜像。

### context
context 选项可以是 Dockerfile 的文件路径，也可以是到链接到 git 仓库的 url, 当提供的值是相对路径时，它被解析为相对于撰写文件的路径。
```
build:
  context: ./dir
```

### dockerfile
使用此 dockerfile 文件来构建，必须指定构建路径
```
build:
  context: ./dir
  dockerfile: Dockerfile-tomcat
```

### args
添加构建参数，这些参数是仅在构建过程中可访问的环境变量,它和Dockerfile中ARG效果类似
首先， 在Dockerfile中指定参数：
```bash
ARG buildno
ARG gitcommithash

RUN echo "Build number: $buildno"
RUN echo "Based on commit: $gitcommithash"
```
然后指定 build 下的参数,可以传递映射或列表
```
build:
  context: .
  args:
    buildno: 1
    gitcommithash: cdc3b19
```
> 在Dockerfile中，如果在FROM指令之前指定了ARG，那么在FROM下面的构建指令中就不能使用ARG。如果您需要一个参数在两个地方都可用，也可以在FROM指令下指定它。有关使用细节，请参见[ARGS和FROM如何交互](https://docs.docker.com/engine/reference/builder/#understand-how-arg-and-from-interact)。

指定构建参数时可以省略该值，在这种情况下，构建时的值默认构成运行环境中的值
```
args:
  - buildno
  - gitcommithash
```
> Tip： YAML 布尔值（true，false，yes，no，on，off）必须使用引号括起来，以为了能够正常被解析为字符串

### cache_from
编写缓存解析镜像列表
```
build:
  context: .
  cache_from:
    - alpine:latest
    - corp/web_app:3.14
```

### labels
使用 Docker标签 将元数据添加到生成的镜像中，可以使用数组或字典。建议使用反向 DNS 标记来防止签名与其他软件所使用的签名冲突, 例如：
```
build:
  context: .
  labels:
    com.example.description: "Accounting webapp"
    com.example.department: "Finance"
    com.example.label-with-empty-value: ""
```

### shm_size
设置容器 /dev/shm 分区的大小，值为表示字节的整数值或表示字符的字符串
```
build:
  context: .
  shm_size: '2gb'
```

### target
根据对应的 Dockerfile 构建指定 Stage, 关于[Dockerfile创建多个stage参考官方说明](https://docs.docker.com/develop/develop-images/multistage-build/)
```
build:
    context: .
    target: prod
```

## cap_add, cap_drop
添加或删除容器功能，可查看 [man 7 capabilities](http://man7.org/linux/man-pages/man7/capabilities.7.html)
```
cap_add:
  - ALL

cap_drop:
  - NET_ADMIN
  - SYS_ADMIN
```
> 使用（版本3）Compose文件在swarm群集模式下部署stack时，该选项被忽略。因为 `docker stack` 命令只接受预先构建的镜像。

## cgroup_parent
为容器指定可选的父cgroup。
```
cgroup_parent: m-executor-abcd
```
> 使用（版本3）Compose文件在swarm群集模式下部署stack时，该选项被忽略。

## command
覆盖容器启动后默认执行的命令
```
command: bundle exec thin -p 3000
```
该命令也可以是一个列表，方法类似于 dockerfile:
```
command: ["bundle", "exec", "thin", "-p", "3000"]
```

## configs
使用服务 configs 配置为每个服务赋予相应的访问权限，支持两种不同的语法。
> 配置必须存在或在 configs 此堆栈文件的顶层中定义，否则堆栈部署失效

### SHORT 语法
SHORT 语法只能指定配置名称，这允许容器访问配置并将其安装在 /<config_name> 容器内，源名称和目标装入点都设为配置名称。
```
version: "3.7"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - my_config
      - my_other_config
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```
以上实例使用 SHORT 语法将 redis 服务访问授予 my_config 和 my_other_config ,并被 my_other_config 定义为外部资源，这意味着它已经在 Docker 中定义。可以通过 docker config create 命令或通过另一个堆栈部署。如果外部部署配置都不存在，则堆栈部署会失败并出现 config not found 错误。
### LONG 语法
LONG 语法提供了创建服务配置的更加详细的信息
* `source`: Docker 中存在的配置的名称
* `target`: 要在服务的任务中装载的文件的路径或名称。如果未指定则默认为 /<source>
* `uid 和 gid`: 在服务的任务容器中拥有安装的配置文件的数字 UID 或 GID。如果未指定，则默认为在Linux上。Windows不支持。
* `mode`: 在服务的任务容器中安装的文件的权限，以八进制表示法。例如，0444 代表文件可读的。默认是 0444。如果配置文件无法写入，是因为它们安装在临时文件系统中，所以如果设置了可写位，它将被忽略。可执行位可以设置。如果您不熟悉 UNIX 文件权限模式，Unix Permissions Calculator

下面示例在容器中将 my_config 名称设置为 redis_config，将模式设置为 0440（group-readable）并将用户和组设置为 103。该　｀redis　服务无法访问 my_other_config 配置。
```
version: "3.7"
services:
  redis:
    image: redis:latest
    deploy:
      replicas: 1
    configs:
      - source: my_config
        target: /redis_config
        uid: '103'
        gid: '103'
        mode: 0440
configs:
  my_config:
    file: ./my_config.txt
  my_other_config:
    external: true
```
可以同时授予多个配置的服务相应的访问权限，也可以混合使用 LONG 和 SHORT 语法。定义配置并不意味着授予服务访问权限。

## container_name
为自定义的容器指定一个名称，而不是使用默认的名称
```
container_name: my-web-container
```
因为 docker 容器名称必须是唯一的，所以如果指定了一个自定义的名称，不能扩展一个服务超过 1 个容器

## depends_on
此选项解决了启动顺序的问题, 在使用 Compose 时，最大的好处就是少打启动命令，但是一般项目容器启动的顺序是有要求的，如果直接从上到下启动容器，必然会因为容器依赖问题而启动失败。例如在没启动数据库容器的时候启动了应用容器，这时候应用容器会因为找不到数据库而退出，为了避免这种情况我们需要加入一个标签，就是 depends_on，这个标签解决了容器的依赖、启动先后的问题。指定服务之间的依赖关系，有两种效果:
* `docker-compose up` 以依赖顺序启动服务，下面例子中 redis 和 db 服务在 web 启动前启动
* `docker-compose up SERVICE` 自动包含 SERVICE 的依赖性，下面例子中，例如下面容器会先启动 redis 和 db 两个服务，最后才启动 web 服务：
```
version: "3.7"
services:
  web:
    build: .
    depends_on:
      - db
      - redis
  redis:
    image: redis
  db:
    image: postgres
```
注意的是，默认情况下使用 docker-compose up web 这样的方式启动 web 服务时，也会启动 redis 和 db 两个服务，因为在配置文件中定义了依赖关系

## deploy
指定与部署和运行服务相关的配置。
> 这仅在使用docker stack deploy部署到swarm时生效，单机编排或使用docker-compose up和docker-compose run命令将忽略该配置。

```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      replicas: 6
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
```
这里有几个子选项

### endpoint_mode
指定连接到群组外部客户端服务发现方法

* `endpoint_mode:vip` ：Docker 为该服务分配了一个虚拟 IP(VIP),作为客户端的 “前端“ 部位用于访问网络上的服务。
* `endpoint_mode: dnsrr` : DNS轮询（DNSRR）服务发现不使用单个虚拟 IP。Docker为服务设置 DNS 条目，使得服务名称的 DNS 查询返回一个 IP 地址列表，并且客户端直接连接到其中的一个。如果想使用自己的负载平衡器，或者混合 Windows 和 Linux 应用程序，则 DNS 轮询调度（round-robin）功能就非常实用。
```
version: "3.7"

services:
  wordpress:
    image: wordpress
    ports:
      - "8080:80"
    networks:
      - overlay
    deploy:
      mode: replicated
      replicas: 2
      endpoint_mode: vip

  mysql:
    image: mysql
    volumes:
       - db-data:/var/lib/mysql/data
    networks:
       - overlay
    deploy:
      mode: replicated
      replicas: 2
      endpoint_mode: dnsrr

volumes:
  db-data:

networks:
  overlay:
```

### labels
指定服务的标签，这些标签仅在服务上设置。
```
version: "3.7"
services:
  web:
    image: web
    deploy:
      labels:
        com.example.description: "This label will appear on the web service"
```
通过将 deploy 外面的 labels 标签来设置容器上的 labels
```
version: "3.7"
services:
  web:
    image: web
    labels:
      com.example.description: "This label will appear on all containers for the web service"
```

### mode
* global:每个集节点只有一个容器
* replicated:指定容器数量（默认）
```
version: "3.7"
services:
  worker:
    image: dockersamples/examplevotingapp_worker
    deploy:
      mode: global
```

### placement
指定 `constraints` 和 `preferences`,有关语法的完整描述以及约束和首选项的可用类型，[请参阅docker service create documentation](https://docs.docker.com/engine/reference/commandline/service_create/#specify-service-constraints---constraint)
```
version: "3.7"
services:
  db:
    image: postgres
    deploy:
      placement:
        constraints:
          - node.role == manager
          - engine.labels.operatingsystem == ubuntu 14.04
        preferences:
          - spread: node.labels.zone
```

### replicas
如果服务是 replicated（默认)，需要指定运行的容器数量
```
version: "3.7"
services:
  worker:
    image: dockersamples/examplevotingapp_worker
    networks:
      - frontend
      - backend
    deploy:
      mode: replicated
      replicas: 6
```

### resources
配置资源限制
```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 50M
        reservations:
          cpus: '0.25'
          memory: 20M
```
此例子中，redis 服务限制使用不超过 50M 的内存和 0.50（50％）可用处理时间（CPU），并且 保留 20M 了内存和 0.25 CPU时间

### restart_policy
配置容器的重新启动，代替 restart
* `condition`:值可以为 none 、on-failure 以及 any(默认)
* `delay`: 尝试重启的等待时间，默认为 0
* `max_attempts`: 在放弃之前尝试重新启动容器次数（默认：从不放弃）。如果重新启动在配置中没有成功 window，则此尝试不计入配置max_attempts 值。例如，如果 max_attempts 值为 2，并且第一次尝试重新启动失败，则可能会尝试重新启动两次以上。
* `windows`: 在决定重新启动是否成功之前的等时间，指定为持续时间（默认值：立即决定）。
```
version: "3.7"
services:
  redis:
    image: redis:alpine
    deploy:
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
```
### rollback_config
配置在更新失败的情况下应如何回滚服务。
* `parallelism`: 一次回滚的容器数。如果设置为0，则所有容器同时回滚
* `delay`: 每个容器组的回滚之间等待的时间（默认为0）
* `failure_action`: 如果回滚失败该怎么办。继续或暂停（默认暂停）
* `monitor`: 每次更新任务后监视失败的持续时间（ns | us | ms | s | m | h）（默认为0）
* `max_failure_ratio`: 回滚期间容忍的失败率（默认为0）
* `order`: 回滚期间的操作顺序。stop-first（旧任务在启动新任务之前停止）或start-first（新任务首先启动，运行任务暂时重叠）（默认stop-first）。

### update_config
配置更新服务，用于无缝更新应用（rolling update)
`parallelism`, `delay`, `failure_action`, `monitor`, `max_failure_ratio`, `order`指令和 rollback_config 意思一样，只是是更新而已。
```
version: '3.7'
services:
  vote:
    image: dockersamples/examplevotingapp_vote:before
    depends_on:
      - redis
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
        order: stop-first
```
### 不支持 Docker stack desploy 的几个子选项
build、cgroup_parent、container_name、devices、tmpfs、external_links、inks、network_mode、restart、security_opt、stop_signal、sysctls、userns_mode

## devices
设置映射列表，与 Docker 客户端的 --device 参数类似 :
```
devices:
  - "/dev/ttyUSB0:/dev/ttyUSB0"
```

## dns
自定义 DNS 服务器，与 --dns 具有一样的用途，可以是单个值或列表
```
dns: 8.8.8.8
dns:
  - 8.8.8.8
  - 9.9.9.9
```

## dns_search
自定义 DNS 搜索域，可以是单个值或列表
```
dns_search: example.com
dns_search:
  - dc1.example.com
  - dc2.example.com
```

## entrypoint
在 Dockerfile 中有一个指令叫做 ENTRYPOINT 指令，用于指定接入点。在 docker-compose.yml 中可以定义接入点，覆盖 Dockerfile 中的定义：
```
entrypoint: /code/entrypoint.sh
```
entrypoint 也可以是一个列表，方法类似于 dockerfile
```
entrypoint:
    - php
    - -d
    - zend_extension=/usr/local/lib/php/extensions/no-debug-non-zts-20100525/xdebug.so
    - -d
    - memory_limit=-1
    - vendor/bin/phpunit
```

## env_file
从文件中添加环境变量。可以是单个值或是列表,如果已经用 docker-compose -f FILE 指定了 Compose 文件，那么 env_file 路径值为相对于该文件所在的目录, 但 environment 环境中的设置的变量会会覆盖这些值，无论这些值未定义还是为 None.
```
env_file: .env
```
或者根据 docker-compose.yml 设置多个：
```
env_file:
  - ./common.env
  - ./apps/web.env
  - /opt/secrets.env
```
环境配置文件 env_file 中的声明每行都是以 VAR=VAL 格式，其中以 # 开头的被解析为注释而被忽略

## environment
添加环境变量，可以使用数组或字典。与上面的 env_file 选项完全不同，反而和 arg 有几分类似，这个标签的作用是设置镜像变量，它可以保存变量到镜像里面，也就是说启动的容器也会包含这些变量设置，这是与 arg 最大的不同。
一般 arg 标签的变量仅用在构建过程中。而 environment 和 Dockerfile 中的 ENV 指令一样会把变量一直保存在镜像、容器中，类似 docker run -e 的效果.
```
environment:
  RACK_ENV: development
  SHOW: 'true'
  SESSION_SECRET:
```
## expose
暴露端口，但不映射到宿主机，只被连接的服务访问。这个标签与 Dockerfile 中的 EXPOSE 指令一样，用于指定暴露的端口，但是只是作为一种参考，实际上 docker-compose.yml 的端口映射还得 ports 这样的标签.
```
expose:
 - "3000"
 - "8000"
```
## external_links
链接到 docker-compose.yml 外部的容器，甚至 并非 Compose 项目文件管理的容器。参数格式跟 links 类似.
```
external_links:
 - redis_1
 - project_db_1:mysql
 - project_db_1:postgresql
```
## extra_hosts
添加主机名的标签，就是往 /etc/hosts 文件中添加一些记录，与 Docker 客户端 中的 --add-host 类似：
```
extra_hosts:
 - "somehost:162.242.195.82"
 - "otherhost:50.31.209.229"
```
具有 IP 地址和主机名的条目在 /etc/hosts 内部容器中创建。启动之后查看容器内部 hosts ，例如：
```
162.242.195.82  somehost
50.31.209.229   otherhost
```
## healthcheck
用于检查测试服务使用的容器是否正常.
```
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s
```
`interval`，`timeout` 以及 `start_period` 都定为持续时间, test 必须是字符串或列表，如果它是一个列表，第一项必须是 NONE，CMD 或 CMD-SHELL ；如果它是一个字符串，则相当于指定CMD-SHELL 后跟该字符串。例如：
```
# Hit the local web app
test: ["CMD", "curl", "-f", "http://localhost"]
```
如上所述，但包装在/bin/sh中。而以下两种形式都是等同的。
```
test: ["CMD-SHELL", "curl -f http://localhost || exit 1"]

test: curl -f https://localhost || exit 1
```
如果需要禁用镜像的所有检查项目，可以使用 disable:true,相当于 test:["NONE"]
```
healthcheck:
  disable: true
```
## init
在容器内运行init，转发信号并重新获得进程。将此选项设置为true可为服务启用此功能。
```
version: "3.7"
services:
  web:
    image: alpine:latest
    init: true
```
## labels
使用 Docker 标签将元数据添加到容器，可以使用数组或字典。与 Dockerfile 中的 LABELS 类似：
```
labels:
  com.example.description: "Accounting webapp"
  com.example.department: "Finance"
  com.example.label-with-empty-value: ""

labels:
  - "com.example.description=Accounting webapp"
  - "com.example.department=Finance"
  - "com.example.label-with-empty-value"
```
## links
链接到其它服务的中的容器，可以指定服务名称也可以指定链接别名（SERVICE：ALIAS)，与 Docker 客户端的 --link 有一样效果，会连接到其它服务中的容器.
```
web:
  links:
   - db
   - db:database
   - redis
```
使用的别名将会自动在服务容器中的 /etc/hosts 里创建。相应的环境变量也将被创建。例如：
```
172.12.2.186  db
172.12.2.186  database
172.12.2.187  redis
```
## logging
配置日志服务
```
logging:
  driver: syslog
  options:
    syslog-address: "tcp://192.168.0.42:123"
```
该 driver值是指定服务器的日志记录驱动程序，默认值为 json-file,与 --log-diver 选项一样
```
driver: "json-file"
driver: "syslog"
driver: "none"
```
> 只有驱动程序 json-file 和 journald 驱动程序可以直接从 docker-compose up 和 docker-compose logs 获取日志。使用任何其他方式不会显示任何日志。

对于可选值，可以使用 options 指定日志记录中的日志记录选项
```
driver: "syslog"
options:
  syslog-address: "tcp://192.168.0.42:123"
```
默认驱动程序 json-file 具有限制存储日志量的选项，所以，使用键值对来获得最大存储大小以及最小存储数量
```
options:
  max-size: "200k"
  max-file: "10"
```
上面实例将存储日志文件，直到它们达到max-size:200kB，存储的单个日志文件的数量由该 max-file 值指定。随着日志增长超出最大限制，旧日志文件将被删除以存储新日志
docker-compose.yml 限制日志存储的示例:
```
services:
  some-service:
    image: some-service
    logging:
      driver: "json-file"
      options:
        max-size: "200k"
        max-file: "10"
```
## network_mode
可以指定使用服务或者容器的网络模式，用法类似于 Docke 客户端的 --net 选项，格式为：service:[service name]
```
network_mode: "bridge"
network_mode: "host"
network_mode: "none"
network_mode: "service:[service name]"
network_mode: "container:[container name/id]"
```
> 在群集模式下使用（版本3）Compose文件部署堆栈时，将忽略此选项。network_mode：“host”不能与links混合使用。

## networks
加入指定网络
```
services:
  some-service:
    networks:
     - some-network
     - other-network
```
### aliases
同一网络上的其他容器可以使用服务器名称或别名来连接到其他服务的容器
```
services:
  some-service:
    networks:
      some-network:
        aliases:
         - alias1
         - alias3
      other-network:
        aliases:
         - alias2
```
下面实例中，提供 web 、worker以及db 服务，伴随着两个网络 new 和 legacy 。相同的服务可以在不同的网络有不同的别名:
```
version: "3.7"

services:
  web:
    image: "nginx:alpine"
    networks:
      - new

  worker:
    image: "my-worker-image:latest"
    networks:
      - legacy

  db:
    image: mysql
    networks:
      new:
        aliases:
          - database
      legacy:
        aliases:
          - mysql

networks:
  new:
  legacy:
```
### ipv4_address、ipv6_address
为服务的容器指定一个静态 IP 地址
```
version: "3.7"

services:
  app:
    image: nginx:alpine
    networks:
      app_net:
        ipv4_address: 172.16.238.10
        ipv6_address: 2001:3984:3989::10

networks:
  app_net:
    ipam:
      driver: default
      config:
        - subnet: "172.16.238.0/24"
        - subnet: "2001:3984:3989::/64"
```
## PID
将 PID 模式设置为主机 PID 模式，可以打开容器与主机操作系统之间的共享 PID 地址空间。使用此标志启动的容器可以访问和操作宿主机的其他容器，反之亦然。
```
pid: "host"
```

## ports
映射容器端口到本地主机
### SHORT 语法
可以使用 HOST:CONTAINER 的方式指定端口，也可以指定容器端口（选择临时主机端口），宿主机会随机映射端口。
```
ports:
 - "3000"
 - "3000-3005"
 - "8000:8000"
 - "9090-9091:8080-8081"
 - "49100:22"
 - "127.0.0.1:8001:8001"
 - "127.0.0.1:5000-5010:5000-5010"
 - "6060:6060/udp"
```
> 注意：当使用 HOST:CONTAINER 格式来映射端口时，如果使用的容器端口小于 60 可能会得到错误得结果，因为YAML 将会解析 xx:yy 这种数字格式为 60 进制，所以建议采用字符串格式。

### LONG 语法
LONG 语法支持 SHORT 语法不支持的附加字段
* `target`：容器内的端口
* `published`：公开的端口
* `protocol`： 端口协议（tcp 或 udp）
* `mode`：通过host 用在每个节点还是哪个发布的主机端口或使用 ingress 用于集群模式端口进行平衡负载，
```
ports:
  - target: 80
    published: 8080
    protocol: tcp
    mode: host
```
## volumes
挂载一个目录或者一个已存在的数据卷容器，可以直接使用 `HOST:CONTAINER` 这样的格式，或者使用 `HOST:CONTAINER:ro` 这样的格式，后者对于容器来说，数据卷是只读的，这样可以有效保护宿主机的文件系统。
```
version: "3.7"
services:
  web:
    image: nginx:alpine
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

  db:
    image: postgres:latest
    volumes:
      - "/var/run/postgres/postgres.sock:/var/run/postgres/postgres.sock"
      - "dbdata:/var/lib/postgresql/data"

volumes:
  mydata:
  dbdata:
```

### SHORT 语法
可以选择在主机（HOST:CONTAINER）或访问模式（HOST:CONTAINER:ro）上指定路径。可以在主机上挂载相对路径，该路径相对于正在使用的 Compose 配置文件的目录进行扩展。相对路径应始终以 . 或 .. 开头
```
volumes:
  # Just specify a path and let the Engine create a volume
  - /var/lib/mysql

  # Specify an absolute path mapping
  - /opt/data:/var/lib/mysql

  # Path on the host, relative to the Compose file
  - ./cache:/tmp/cache

  # User-relative path
  - ~/configs:/etc/configs/:ro

  # Named volume
  - datavolume:/var/lib/mysql
```

### LONG 语法
LONG 语法有些附加字段
* `type`：安装类型，可以为 volume、bind 或 tmpfs
* `source`：安装源，主机上用于绑定安装的路径或定义在顶级 volumes密钥中卷的名称 ,不适用于 tmpfs 类型安装。
* `target`：卷安装在容器中的路径
* `read_only`：标志将卷设置为只读
* `bind`：配置额外的绑定选项
* `propagation`：用于绑定的传播模式
* `volume`：配置额外的音量选项
* `nocopy`：创建卷时禁止从容器复制数据的标志
* `tmpfs`：配置额外的 tmpfs 选项
* `size`：tmpfs 的大小，以字节为单位
```
version: "3.7"
services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - type: volume
        source: mydata
        target: /data
        volume:
          nocopy: true
      - type: bind
        source: ./static
        target: /opt/app/static

networks:
  webnet:

volumes:
  mydata:
```

#  创建一个compose实例
本实例是以单机编排进行说明，因此有些配置并不适宜，比如deploy配置，该配置适用于swarm集群。关于哪些配置能在集群中用哪些不能，上文有说明。或者前往官网查看。

## 试验环境
我们将使用docker-compose.yml搭建2个服务。分别为nginx和tomcat来进行说明。而其中nginx使用image指令pull镜像，tomcat使用build进行本地构建。（swarm集群会忽略build指令）
## 创建docker-compose项目
```bash
$ mkdir /webserver
```
## 创建docker-compose.yml
```bash
$ vi docker-compose.yml
version: "3.7"
services:
  nginx:      # 服务名称
    image: nginx:1.17.0-alpine  # 从dockerhub上拉取nginx镜像
    volumes:  # 定义容器挂载卷
      - nginx_conf:/etc/nginx:rw
      - nginx_db:/usr/share/nginx/html:ro
    ports:    # 定义容器端口映射到本地的端口
      - "80:80"
      - "443:443"
    depends_on:  # 启动顺序依赖
      - tomcat
    networks:    # 加入的网络
      - frontend
    deploy:   # 该语句块配置只能在swarm集群中使用，这里写出来只是用于参考，实际上这里我注释了。
      mode: replicated
      replicas: 3
      restart_policy:
        condition: no-failure
      resources:
        limits:
          cpus: '2'
          memory: 1024M
        reservations:
          cpus: '1'
          memory: 500M

  tomcat:
    build:
     context: ./tomcat_df
     dockerfile: Dockerfile-tomcat
#     args:
#       TOMCAT_PORT: 808
     labels:
       com.ipyker.descrioption: "Accounting webapp"
    networks:
       - frontend

networks:
  frontend:

volumes:
  nginx_db:
  nginx_conf:
```
可以看到tomcat使用build指令构建Dockerfile，Dockerfile-tomcat在当前撰写文件目录的tomcat_df目录中。
```bash
$ cat tomcat_df/Dockerfile-tomcat
FROM tomcat
MAINTAINER pykerzhang <pyker@qq.com>
VOLUME ["/tomcat_data"]
```
我们这里只是为了演示，所以使用了tomcat镜像，如果你需要自己配置镜像，可以自己编写Dockerfile编写自己需要的tomcat镜像。

## 启动服务
```bash
$ docker-compose up -d
Starting webserver_nginx_1 ... done
Starting webserver_tomcat_1 ... done
```
## 查看镜像
```bash
$ docker-compose images
WARNING: Some services (nginx, tomcat) use the 'deploy' key, which will be ignored. Compose does not support 'deploy' configuration - use `docker stack deploy` to deploy to a swarm.
    Container           Repository           Tag          Image Id      Size  
------------------------------------------------------------------------------
webserver_nginx_1    nginx              1.17.0-alpine   bfba26ca350c   19.5 MB
webserver_tomcat_1   webserver_tomcat   latest          516992dbffe3   498 MB
```
> 上述WARNING是因为在单机编排中使用了swarm集群模式的deploy指令，被忽略了所产生的。

## 查看运行的容器
```bash
$ docker-compose ps
       Name                Command          State                    Ports                  
--------------------------------------------------------------------------------------------
webserver_nginx_1    nginx -g daemon off;   Up      0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
webserver_tomcat_1   catalina.sh run        Up      8080/tcp
```
## 查看挂载卷和网络
```bash
$ docker network ls
NETWORK ID          NAME                 DRIVER              SCOPE
5b5654c5583f        bridge               bridge              local
c362e57d9526        host                 host                local
9ee0bba2c2bd        none                 null                local
2237dc005d9f        webserver_frontend   bridge              local

$ docker volume ls
DRIVER              VOLUME NAME
local               0bc45d250accfa394bbfcf669bd3564e3cf7eb6ffcac7224c65f78e07ae76eb5
local               webserver_nginx_conf
local               webserver_nginx_db
```
此时可以发现我们在docker-compose.yml文件中定义的networks和volumes均自动创建了。类似手动执行docker network create webserver_frontend 和docker volume create webserver_nginx_conf， docker volume create webserver_nginx_db。那么卷将会在/var/lib/docker/volumes下产生，并且会将容器挂载的目录映射到对应的本地目录。如：
```
$ ls /var/lib/docker/volumes/webserver_nginx_conf/_data/
conf.d  fastcgi.conf  fastcgi_params  koi-utf  koi-win  mime.types  modules  nginx.conf  scgi_params  uwsgi_params  win-utf
```
那么此时我们就可以通过修改配置nginx虚拟主机配置文件来反向代理tomcat
```
server {
    listen       80;
    server_name  localhost;
    location /tomcat/ {
        proxy_pass http://172.18.0.2:8080/;
     ｝
｝
```
## 重启nginx服务
```
$ docker-compose restart nginx
```
##访问 tomcat服务
现在可以在局域网任意设备上访问该宿主主机IP的80端口进行访问容器服务了。
```bash
$ curl -I  http://192.168.20.210/webtomcat/
HTTP/1.1 200
Server: nginx/1.17.0
Date: Wed, 12 Jun 2019 10:03:24 GMT
Content-Type: text/html;charset=UTF-8
Connection: keep-alive
```
