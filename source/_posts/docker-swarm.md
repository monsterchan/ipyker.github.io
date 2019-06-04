layout: post
title: docker三剑客之docker-swarm集群
author: Pyker
categories: docker
tags:
  - docker
date: 2018-03-23 14:12:00
---

# Docker Swarm 概念
`Docker Swarm`是Docker的群集。 在Docker 1.12及更高版本中，Swarm模式与Docker Engine集成在一起。它将若干单个的Docker Engine主机组成一个Docker集群。 由于Docker Swarm提供标准的Docker API，因此任何已与Docker守护程序通信的工具都可以使用Swarm透明地扩展到多个Docker Engine主机。 

## Docker 集群
从主机的层面来看，`Docker Swarm` 管理的是 Docker Host 集群。所以先来讨论一个重要的概念 - `集群化（Clustering）`。
服务器集群由一组网络上相互连接的服务器组成，它们一起协同工作。一个集群和一堆服务器最显著的区别在于：
* 群能够像单个系统那样工作，同时提供高可用、负载均衡和并行处理。
* 实现集群化后我们的思维方式就必须改变了：不再考虑一个一个的服务器，而是将集群看做是一个整体。
* 集群整体容量的调整是通过往集群中添加和删除主机节点实现的。但不管做怎样的操作，集群始终还是一个整体。

## swarm
* `swarm` 是运行 Docker Engine 的多个主机组成的集群。从 v1.12 开始，集群管理和编排功能已经集成进 Docker Engine。
* 当 Docker Engine 初始化了一个 swarm 或者加入到一个存在的 swarm 时，它就启动了 swarm mode。没启动 swarm mode 时，Docker 执行的是容器命令。
* 运行 swarm mode 后，Docker 增加了编排 service 的能力。Docker 允许在同一个 Docker 主机上既运行 swarm service，又运行单独的容器。

## node
* swarm 中的每个 Docker Engine 都是一个 node，有两种类型的node，`manager` 和 `worker`。
* 为了向 swarm 中部署应用，我们需要在 manager node 上执行部署命令，manager和node 会将部署任务拆解并分配给一个或多个worker node 完成部署。
* `manager node` 负责执行编排和集群管理工作，保持并维护 swarm 处于期望的状态。swarm 中如果有多个 manager node，它们会自动协商并选举出一个 leader 执行编排任务。
* `woker node` 接受并执行由 manager node 派发的任务。默认配置下 manager node 同时也是一个 worker node，不过可以将其配置成 manager-only node，让其专职负责编排和集群管理工作。
* work node 会定期向 manager node 报告自己的状态和它正在执行的任务的状态，这样 manager 就可以维护整个集群的状态。

## service
* service 定义了 worker node 上要执行的任务。
* swarm 的主要编排任务就是保证 service 处于期望的状态下。举一个 service 的例子：在 swarm 中启动一个 http 服务，使用的镜像是 httpd:latest，副本数为 3。manager node 负责创建这个 service，经过分析知道需要启动 3 个 httpd 容器，根据当前各 worker node 的状态将运行容器的任务分配下去，比如 worker1 上运行两个容器，worker2 上运行一个容器。运行了一段时间，worker2 突然宕机了，manager 监控到这个故障，于是立即在 worker1上启动了一个新的 httpd 容器。这样就保证了 service 处于期望的三个副本状态。

# 创建Docker swarm集群
## 环境准备
* 所有节点Docker版本均不低于 v1.12，我们是最新版的V18.09.6，实验环境node的操作系统为 Centos7.4，当然其他 Linux 也是可以的。
* 所有主机均配置了/etc/hosts对应的主机和ip解析。
* 我们所有的操作都在k8s-master1主机上，请配置ssh-keygen确保k8s-master1主机能无密码登陆其他主机。

主机名 | ip | 角色
:-----: | :---: | :---:
k8s-master1 | 192.168.20.210 | swarm manager1
k8s-master2 | 192.168.20.211 | swarm manager2
k8s-master3 | 192.168.20.212 | swarm manager3
k8s-node1 | 192.168.20.213 | swarm worker1
k8s-node2 | 192.168.20.214 | swarm worker2

## 安装docker engine
环境准备好后，我们需要为所有主机安装docker engine，你可以用传统方法登陆每台主机手动安装docker，也可以使用docker-machine统一管理和安装docker engine。我们这里使用docker-machine方法为所有主机安装docker engine。步骤我们已经在[上一篇](https://www.ipyker.com/2018/03/22/docker-machine.html)文章配置过，安装完后如下：
```bash
$ docker-machine ls
NAME          ACTIVE   DRIVER    STATE     URL                         SWARM   DOCKER     ERRORS
k8s-master1   -        generic   Running   tcp://192.168.20.210:2376           v18.09.6   
k8s-master2   -        generic   Running   tcp://192.168.20.211:2376           v18.09.6   
k8s-master3   -        generic   Running   tcp://192.168.20.212:2376           v18.09.6   
k8s-node1     -        generic   Running   tcp://192.168.20.213:2376           v18.09.6   
k8s-node2     -        generic   Running   tcp://192.168.20.214:2376           v18.09.6
```

## 创建swarm
在 k8s-master1 主机上执行如下命令创建 swarm，使用命令docker swarm init --advertise-addr 192.168.20.210。
```bash
$ docker swarm init --advertise-addr 192.168.20.210
Swarm initialized: current node (tlviocryph6mlw8yqp2m1nmv9) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1fdudt9by9zsvvr480thnxzsbbd66d50q5dh0swxraa8szp7s5-awrgjg1n17793vif828xsp63x 192.168.20.210:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```
从结果输出我们可以看出 manager 已经初始化完成，swarm-manager 成为 manager node，可以看到添加 worker node 和 manager node 的执行指令。
>`--advertise-addr`：指定与其他 node 通信的地址。

## 添加 node
执行 docker node ls 查看当前 swarm 的 node，目前只有一个 manager。
```bash
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
tlviocryph6mlw8yqp2m1nmv9 *   k8s-master1         Ready               Active               Leader              18.09.6
```
>如果当时没有记录下 docker swarm init 提示的添加 worker 的完整命令，可以通过docker swarm join-token worker查看。

复制前面的 docker swarm join命令，分别在 swarm-worker1 和 swarm-worker2 上执行，将它们添加到 swarm 中。
```bash
# 在k8s-node1和k8s-node2上都执行
$ docker swarm join --token SWMTKN-1-1fdudt9by9zsvvr480thnxzsbbd66d50q5dh0swxraa8szp7s5-awrgjg1n17793vif828xsp63x 192.168.20.210:2377
This node joined a swarm as a worker.
```
## 查看添加node结果
docker node ls可以看到两个 worker node 已经添加进来了。
```bash
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
tlviocryph6mlw8yqp2m1nmv9 *   k8s-master1         Ready               Active               Leader              18.09.6
odkkwwrfdowu2szjxo5yejuex     k8s-node1           Ready               Active                                  18.09.6
uokltoj11sdw9ffov7c1vb1lp     k8s-node2           Ready               Active                                  18.09.6
```
## 添加manager
通过上面的步骤我们已经创建了一个swarm manager两个 swarm worker，那么为了高可用动态选举Leader，我们这里在创建2个manager。
>如果当时没有记录下 docker swarm init 提示的添加 worker 的完整命令，可以通过docker swarm join-token manager查看。

```bash
# 显示对swarm集群加入manager节点的命令
$ docker swarm join-token manager
To add a manager to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-1fdudt9by9zsvvr480thnxzsbbd66d50q5dh0swxraa8szp7s5-353ufwphwu11rkokojlte8fzt 192.168.20.210:2377
```
复制这条 docker swarm join命令，在k8s-master1 上执行，将它们添加到 swarm 中。
```bash
# 把k8s-master2加入集群
$ docker-machine ssh k8s-master2 docker swarm join --token SWMTKN-1-1fdudt9by9zsvvr480thnxzsbbd66d50q5dh0swxraa8szp7s5-353ufwphwu11rkokojlte8fzt 192.168.20.210:2377

# 把k8s-master3加入集群
$ docker-machine ssh k8s-master3 docker swarm join --token SWMTKN-1-1fdudt9by9zsvvr480thnxzsbbd66d50q5dh0swxraa8szp7s5-353ufwphwu11rkokojlte8fzt 192.168.20.210:2377
```
>这里使用的命令和加入worker节点不一样，使用的是docker-machine命令，它是通过ssh远程执行docker swarm join命令，和直接在对应主机上执行效果一样。当然还可以使用ansible。

## 查看添加manager结果
docker node ls可以看到两个 manager node 已经添加进来了。
```bash
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
tlviocryph6mlw8yqp2m1nmv9 *   k8s-master1         Ready               Active               Leader              18.09.6
cgacdrgz7g80bdujq6x66ze7v     k8s-master2         Ready               Active               Reachable           18.09.6
n40j3686zd4f5w0vrfjb0p8be     k8s-master3         Ready               Active               Reachable           18.09.6
odkkwwrfdowu2szjxo5yejuex     k8s-node1           Ready               Active                                  18.09.6
uokltoj11sdw9ffov7c1vb1lp     k8s-node2           Ready               Active                                  18.09.6
```
至此，三manager节点和两worker节点的 swarm 集群就已经搭建好了，操作还是相当简单的。

# 部署Service
通过开头对Service的介绍，我们已经知道什么是Service了，现在我们就来创建它。
## 创建Service
我们创建好了 Swarm 集群， 现在部署一个运行nginx镜像的 service，执行如下命令：
```bash
$ docker service create --name httpd nginx
5vk0p382fudcwkf82ksf20s
overall progress: 1 out of 1 tasks 
1/1: running   [==================================================>] 
verify: Service converged 
```
> 请docker service create --help 查看更多关于配置service的选项。

此时我们已经创建好了一个名为httpd的nginx镜像服务，查看一下：
```bash
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
5vk0p382fudc        httpd               replicated          1/1                 nginx:latest
```
REPLICAS 显示当前副本信息，1/1 的意思是 httpd 这个 service 期望的容器副本数量为 1，目前已经启动的副本数量为 1。也就是当前 service 已经部署完成。
命令 docker service ps可以查看 service 每个副本的状态。
```bash
$ docker service ps httpd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
p3p9ld1k3xq9        httpd.1             nginx:latest        k8s-master1           Running             Running 1 minutes ago                           
```
我们可以看到 service 被分配到了 k8s-master1 上面。

## 设置服务副本数
前面部署了只有一个副本的 Service，不过对于 web 服务，我们通常会运行多个实例。这样可以负载均衡，同时也能提供高可用。swarm 要实现这个目标非常简单，增加 service 的副本数就可以了。在 swarm-manager 上执行如下命令：
```bash
$ docker service scale httpd=3
web_server scaled to 3
overall progress: 3 out of 3 tasks 
1/3: running   [==================================================>] 
2/3: running   [==================================================>] 
3/3: running   [==================================================>] 
verify: Service converged
```
副本数增加到 3，通过 docker service ls和 docker service ps httpd查看副本的详细信息。
```bash
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
5vk0p382fudc        httpd               replicated          3/3                 nginx:latest
#
$ docker service ps httpd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
p3p9ld1k3xq9        httpd.1             nginx:latest        k8s-master1         Running             Running 2 minutes ago                       
2ojzjk66n986        httpd.2             nginx:latest        k8s-node1           Running             Running 2 minutes ago                           
41pkpuzqq6rs        httpd.3             nginx:latest        k8s-node2           Running             Running 2 minutes ago
```
我们可以看到 k8s-master1 上面运行了一个副本，默认配置下 manager node 也是 worker node，所以 swarm-manager 上也运行了副本。如果不希望在 manager 上运行 service，可以执行如下命令：
```bash
$ docker node update --availability drain k8s-master1
$ docker node update --availability drain k8s-master2
$ docker node update --availability drain k8s-master3
```
在此查看swarm node状态：
```bash
# 发现AVAILABILITY为Drain状态，表示该节点不接受service的任务分配
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
tlviocryph6mlw8yqp2m1nmv9 *   k8s-master1         Ready               Drain               Leader              18.09.6
cgacdrgz7g80bdujq6x66ze7v     k8s-master2         Ready               Drain               Reachable           18.09.6
n40j3686zd4f5w0vrfjb0p8be     k8s-master3         Ready               Drain               Reachable           18.09.6
odkkwwrfdowu2szjxo5yejuex     k8s-node1           Ready               Active                                  18.09.6
uokltoj11sdw9ffov7c1vb1lp     k8s-node2           Ready               Active                                  18.09.6

```
经过上面的配置后，我们在查看一下service任务的分配情况：
```bash
$ docker service ps httpd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
u724moiz30wk        httpd.1             nginx:latest        k8s-node2           Running             Running 1 minutes ago                             
p3p9ld1k3xq9         \_ httpd.1         nginx:latest        k8s-master1         Shutdown            Shutdown 1 minutes ago                       
2ojzjk66n986        httpd.2             nginx:latest        k8s-node1           Running             Running 5 minutes ago                              
41pkpuzqq6rs        httpd.3             nginx:latest        k8s-node2           Running             Running 5 minutes ago
```
我们可以看到 k8s-master1 上面的副本已经转移到k8s-node2上了。

前面已经对副本数量进行了增加，那么减少也是一样。执行：
```bash
$ docker service scale httpd=2
web_server scaled to 2
overall progress: 2 out of 2 tasks 
1/2: running   [==================================================>] 
2/2: running   [==================================================>]
verify: Service converged 
```
查看减少副本数service的任务状态：
```bash
$ docker service ps httpd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
u724moiz30wk        httpd.1             nginx:latest        k8s-node2           Running             Running 3 minutes ago                             
p3p9ld1k3xq9         \_ httpd.1         nginx:latest        k8s-master1         Shutdown            Shutdown 3 minutes ago                       
2ojzjk66n986        httpd.2             nginx:latest        k8s-node1           Running             Running 7 minutes ago
```
我们可以看到目前 k8s-node1和k8s-node2 上面各运行了一个副本。NAME为httpd.3的已经删除了。

# 故障转移
故障是在所难免的，容器可能崩溃，Docker Host 可能宕机，不过幸运的是，Swarm 已经内置了 failover 策略。创建 service 的时候，我们没有告诉 swarm 发生故障时该如何处理，只是说明了我们期望的状态（比如运行3个副本），swarm 会尽最大的努力达成这个期望状态，无论发生什么状况。以前面部署的 Service 为例，当前 2 个副本分布在 k8s-node1 和 k8s-node2 上，现在我们测试 swarm 的 failover 特性，关闭 k8s-node1。
```bash
$ docker node ls
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
tlviocryph6mlw8yqp2m1nmv9 *   k8s-master1         Ready               Drain               Reachable           18.09.6
cgacdrgz7g80bdujq6x66ze7v     k8s-master2         Ready               Drain               Reachable           18.09.6
n40j3686zd4f5w0vrfjb0p8be     k8s-master3         Ready               Drain               Leader              18.09.6
odkkwwrfdowu2szjxo5yejuex     k8s-node1           Down                Active                                  18.09.6
uokltoj11sdw9ffov7c1vb1lp     k8s-node2           Ready               Active                                  18.09.6
```
Swarm 会将 k8s-node1 上的副本调度到其他可用节点。我们可以通过 docker service ps httpd查看。
```bash
$ docker service ps httpd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
u724moiz30wk        httpd.1             nginx:latest        k8s-node2           Running             Running 1 hours ago                              
p3p9ld1k3xq9         \_ httpd.1         nginx:latest        k8s-master1         Shutdown            Shutdown about an hour ago                       
ip79r8o6423n        httpd.2             nginx:latest        k8s-node2           Running             Running 51 seconds ago                           
2ojzjk66n986         \_ httpd.2         nginx:latest        k8s-node1           Shutdown            Running about a minute ago 
```
可以看到，副本已经从 k8s-node1 迁移到了k8s-node2，之前运行在故障节点 k8s-node1 上的副本状态被标记为 Shutdown。

# 访问Service
前面我们已经学习了如何部署 service，也验证了 swarm 的 failover 特性。不过截止到现在，有一个重要问题还没有涉及：如何访问 service？
通过上面的配置我们运行了一个httpd的服务，服务副本数有2个，都运行在k8s-node2的worker节点上。为了方便演示，我们将其scale副本数扩大到5个。
```bash
$ docker service scale httpd=5
httpd scaled to 5
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
#
$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
5vk0p382fudc        httpd               replicated          5/5                 nginx:latest        
#
$ docker service ps httpd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE                ERROR               PORTS
u724moiz30wk        httpd.1             nginx:latest        k8s-node2           Running             Running 3 hours ago                              
p3p9ld1k3xq9         \_ httpd.1         nginx:latest        k8s-master1         Shutdown            Shutdown about an hour ago                       
ip79r8o6423n        httpd.2             nginx:latest        k8s-node2           Running             Running 10 minutes ago                           
2ojzjk66n986         \_ httpd.2         nginx:latest        k8s-node1           Shutdown            Shutdown 3 minutes ago                           
glfr7knev7rc        httpd.3             nginx:latest        k8s-node1           Running             Running 15 seconds ago                           
qrpl6tcv0dud        httpd.4             nginx:latest        k8s-node1           Running             Running 15 seconds ago                           
3vnh68jq0a34        httpd.5             nginx:latest        k8s-node2           Running             Running 15 seconds ago
```
要访问 http 服务，最起码网络得通吧，服务的 IP 我们得知道吧，但这些信息目前我们都不清楚。不过至少我们知道每个副本都是一个运行的容器，要不先看看容器的网络配置吧。
```bash
$ docker-machine ssh k8s-node1 docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES
0f1eab42a2e3        nginx:latest        "nginx -g 'daemon of…"   3 minutes ago       Up 3 minutes        80/tcp              httpd.4.qrpl6tcv0dudfmt3tj1n8pxta
5852b6a01d0f        nginx:latest        "nginx -g 'daemon of…"   3 minutes ago       Up 3 minutes        80/tcp              httpd.3.glfr7knev7rcj5dpjnztd1pp1
```
在 k8s-node1 上运行了两个容器，是 httpd 的副本，容器监听了 80 端口，但并没有映射到 Docker Host，所以只能通过容器的 IP 访问。查看一下容器的 IP。
```bash
$ docker-machine ssh k8s-node1 docker inspect -f {{.NetworkSettings.IPAddress}} 0f1eab42a2e3
172.17.0.3
```
那么此时我们可以在k8s-node1节点上访问该地址是没问题的。
```bash
$ wget -O - -q 172.17.0.3
<h1>Welcome to nginx!</h1>
```
但这样的访问也仅仅是容器层面的访问，服务并没有暴露给外部网络，只能在 Docker 主机上访问。换句话说，当前配置下，我们无法访问 service httpd。

## 从外部访问 service
要将 service 暴露到外部，方法其实很简单，执行下面的命令：
```bash
$ docker service update --publish-add 8080:80 httpd
httpd
overall progress: 5 out of 5 tasks 
1/5: running   [==================================================>] 
2/5: running   [==================================================>] 
3/5: running   [==================================================>] 
4/5: running   [==================================================>] 
5/5: running   [==================================================>] 
verify: Service converged 
```
>如果是新建 service，可以直接用使用 --publish 参数，比如：docker service create --name httpd --publish 8080:80 --replicas=2 nginx

容器在 80 端口上监听 http 请求，--publish-add 8080:80 将容器的 80 映射到主机的 8080 端口，这样外部网络就能访问到 service 了。
当执行完`docker service update --publish-add 8080:80 httpd`命令后，现有已运行的容器全部会停止，并重启启动指定副本数的容器。此时我们可以netstat查看8080端口是否正常，以及对应的docker host通过8080能否访问容器web。答案是：可以的！
```bash
$ docker service ps httpd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE             ERROR               PORTS
nlfur26tb52u        httpd.1             nginx:latest        k8s-node1           Running             Running 4 minutes ago                         
u724moiz30wk         \_ httpd.1         nginx:latest        k8s-node2           Shutdown            Shutdown 4 minutes ago                        
p3p9ld1k3xq9         \_ httpd.1         nginx:latest        k8s-master1         Shutdown            Shutdown 2 hours ago                          
opdevklkud0z        httpd.2             nginx:latest        k8s-node2           Running             Running 4 minutes ago                         
ip79r8o6423n         \_ httpd.2         nginx:latest        k8s-node2           Shutdown            Shutdown 4 minutes ago                        
2ojzjk66n986         \_ httpd.2         nginx:latest        k8s-node1           Shutdown            Shutdown 20 minutes ago                       
7hnckjv791yp        httpd.3             nginx:latest        k8s-node2           Running             Running 4 minutes ago                         
glfr7knev7rc         \_ httpd.3         nginx:latest        k8s-node1           Shutdown            Shutdown 4 minutes ago                        
k03wuojgarr3        httpd.4             nginx:latest        k8s-node1           Running             Running 4 minutes ago                         
qrpl6tcv0dud         \_ httpd.4         nginx:latest        k8s-node1           Shutdown            Shutdown 4 minutes ago                        
zjriixku92t4        httpd.5             nginx:latest        k8s-node2           Running             Running 4 minutes ago                         
3vnh68jq0a34         \_ httpd.5         nginx:latest        k8s-node1           Shutdown            Shutdown 4 minutes ago
```
```bash
# 访问httpd service
$ curl 192.168.20.210:8080
<h1>Welcome to nginx!</h1>

$ curl 192.168.20.211:8080
<h1>Welcome to nginx!</h1>

$ curl 192.168.20.212:8080
<h1>Welcome to nginx!</h1>

$ curl 192.168.20.213:8080
<h1>Welcome to nginx!</h1>

$ curl 192.168.20.214:8080
<h1>Welcome to nginx!</h1>
```
其实为什么 curl 集群中任何一个节点的 8080 端口，都能够访问到 web_server？这实际上就是使用 swarm 的好处了，这个功能叫做 routing mesh。
我们可以在任意主机上查看docker网络情况！
```bash
$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
ed5feb91f2b5        bridge              bridge              local
af66f21b53bd        docker_gwbridge     bridge              local
8dd3b514208e        host                host                local
5bvz86l4evlq        ingress             overlay             swarm
918b30962f2a        none                null                local
```
很明显容器现在有两块网卡，每块网卡连接不同的 Docker 网络。
* eth0 连接的是一个 overlay 类型的网络，名字为 ingress，其作用是让运行在不同主机上的容器可以相互通信。
* eth1 连接的是一个 bridge 类型的网络，名字为 docker_gwbridge，其作用是让容器能够访问到外网。

`ingress 网络`是 swarm 创建时 Docker 为自动我们创建的，swarm 中的每个 node 都能使用 ingress。
通过 overlay 网络，主机与容器、容器与容器之间可以相互访问；同时，routing mesh 将外部请求路由到不同主机的容器，从而实现了外部网络对 service 的访问。