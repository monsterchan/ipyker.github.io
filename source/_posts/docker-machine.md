layout: post
title: docker三剑客之docker-machine用法
author: Pyker
categories: docker
tags:
  - docker
date: 2018-03-22 17:12:00
---

# Docker Machine 概述
`docker-machine` 是docker官方提供的docker管理工具。简单来说就是给你快速创建一个docker容器环境的, 假如你要给100台阿里云ECS安装上docker，传统方式就是你一台一台ssh上去安装，但是有了docker-machine就不一样了，你可以快速给100台ecs安装上docker，怎么快速法呢，你看完这文章就知道了。还有就是你要在本地快速创建docker集群环境时，总不能一台一台创建虚拟机吧，所以docker-machine可以解决这个问题。总之docker-machine就是帮助你快速去创建安装docker环境的工具，这样说应该没什么问题吧！
通过`docker-machine`可以轻松的做到：
* 在Windows平台和MAC平台安装和运行docker
* 搭建和管理多个docker 主机
* 搭建swarm集群

# Docker Machine安装
可以手动前往`docker machine`的[github地址](https://github.com/docker/machine/releases)下载需要的版本,然后手动重命名且修改权限，或者也可以直接运行以下命令进行安装docker machine。
```bash
$ curl -L https://github.com/docker/machine/releases/download/v0.16.1/docker-machine-`uname -s`-`uname -m` >/tmp/docker-machine &&
    chmod +x /tmp/docker-machine &&
    sudo cp /tmp/docker-machine /usr/local/bin/docker-machine
```
检查是否安装成功:
```bash
#执行docker-machine version命令查看是否安装成功以及对应的版本
$ docker-machine version
  docker-machine version 0.16.1, build cce350d7
```
# 安装 bash completion scripts
为了得到更好的体验，我们可以安装 bash completion script，这样在 bash 能够通过 tab 键补全 docker-mahine 的子命令和参数。
>请先确认版本并运行以下命令将脚本保存到/etc/bash_completion.d目录中

```bash
base=https://raw.githubusercontent.com/docker/machine/v0.16.1
for i in docker-machine-prompt.bash docker-machine-wrapper.bash docker-machine.bash
do
  sudo wget "$base/contrib/completion/bash/${i}" -P /etc/bash_completion.d
done
```
然后，您需要在bash终端中运行
```bash
$ source /etc/bash_completion.d/docker-machine-prompt.bash
```
要启用docker-machine shell提示符，请将$(__docker_machine_ps1)添加到~/.bashrc中的PS1设置。
```bash
cat >> ~/.bashrc << EOF
PS1='[\u@\h \W$(__docker_machine_ps1)]\$ '
EOF
```
# Docker Machine命令介绍
```bash
docker-machine --help
Usage: docker-machine [OPTIONS] COMMAND [arg...]

Create and manage machines running Docker.

Version: 0.16.1, build cce350d7

Author:
  Docker Machine Contributors - <https://github.com/docker/machine>

Options:
  --debug, -D					Enable debug mode
  --storage-path, -s "/root/.docker/machine"	Configures storage path [$MACHINE_STORAGE_PATH]
  --tls-ca-cert 				CA to verify remotes against [$MACHINE_TLS_CA_CERT]
  --tls-ca-key 					Private key to generate certificates [$MACHINE_TLS_CA_KEY]
  --tls-client-cert 				Client cert to use for TLS [$MACHINE_TLS_CLIENT_CERT]
  --tls-client-key 				Private key used in client TLS auth [$MACHINE_TLS_CLIENT_KEY]
  --github-api-token 				Token to use for requests to the Github API [$MACHINE_GITHUB_API_TOKEN]
  --native-ssh					Use the native (Go-based) SSH implementation. [$MACHINE_NATIVE_SSH]
  --bugsnag-api-token 				BugSnag API token for crash reporting [$MACHINE_BUGSNAG_API_TOKEN]
  --help, -h					show help
  --version, -v					print the version
  
Commands:
  active		显示当前操作的是哪台machine
  config		显示已连接machine的配置信息
  create		创建一个machine主机
  env			显示为Docker客户机设置环境的命令
  inspect		检查machine的详细信息
  ip			获取machine的ip地址
  kill			停止一个machine
  ls			列出所有machine
  provision		重建现有的machines
  regenerate-certs	为计machine重新生成TLS证书
  restart		重启machine
  rm			剔除一个machine
  ssh			使用SSH服务登陆machine去执行命令
  scp			machine之间复制文件
  mount			使用SSHFS在machine中挂载和卸载目录卷
  start			启动一个machine
  status		获取machine的状态
  stop			停止一个machine
  upgrade		将machine升级到Docker的最新版本
  url			获取machine tcp地址的url
  version		显示Docker machine版本
  help			显示一个命令的命令列表或帮助
  
Run 'docker-machine COMMAND --help' for more information on a command.
```

# 创建docker-machine
对于 `Docker Machine` 来说，术语 Machine 就是运行 docker daemon 的主机。“创建 Machine” 指的就是在 host 上安装和部署 docker。先执行 docker-machine ls 查看一下当前的 machine：
```bash
$ docker-machine ls
NAME        ACTIVE   DRIVER    STATE     URL                         SWARM   DOCKER     ERRORS
```
当前还没有 machine，接下来我们创建第一个 machine： k8s-node1---192.168.20.213。创建 machine 要求能够无密码登录远程主机，所以需要确保docker-machine主机能无密码登陆machine主机。

## 创建docker-machine主机
```bash
$ docker-machine create   --driver generic   --generic-ip-address=192.168.20.213   --generic-ssh-key ~/.ssh/id_rsa  --generic-ssh-user=root k8s-node1

Running pre-create checks...
Creating machine...
(k8s-node2) Importing SSH key...
Waiting for machine to be running, this may take a few minutes...
Detecting operating system of created instance...
Waiting for SSH to be available...
Detecting the provisioner...
Provisioning with centos...
Copying certs to the local machine directory...
Copying certs to the remote machine...
Setting Docker configuration on the remote daemon...
Checking connection to Docker...
Docker is up and running!
To see how to connect your Docker Client to the Docker Engine running on this virtual machine, run: docker-machine env k8s-node1
```
<div class="note primary"><p>`docker-machine`详细命令参见：https://docs.docker.com/machine/overview/</p></div>

上述命令分析：
* {% label warning@create %}                              #创建docker主机
* {% label warning@--driver %} generic                    #驱动类型 generic 支持linux通用服务器，还支持很多种云主机
* {% label warning@--generic-ip-address %}=192.168.20.213 #指定主机
* {% label warning@--generic-ssh-key %}~/.ssh/id_rsa     #指定私钥
* {% label warning@--generic-ssh-user %}=root             #指定用户
* {% label warning@k8s-node1 %}                           #主机名称

<div class="note primary"><p>因为我们是往普通的 Linux 中部署 docker，所以使用 generic driver。,更多driver参考：https://docs.docker.com/machine/drivers/</p></div>

再次执行 docker-machine ls：
```bash
$ docker-machine ls
NAME        ACTIVE   DRIVER    STATE     URL                         SWARM   DOCKER     ERRORS
k8s-node1   -        generic   Running   tcp://192.168.20.213:2376           v18.09.6   
```
可以使用同样的方法创建多台docker machine主机。如我这里在创建一台k8s-node2。
```bash
$ docker-machine ls
NAME        ACTIVE   DRIVER    STATE     URL                         SWARM   DOCKER     ERRORS
k8s-node1   *        generic   Running   tcp://192.168.20.213:2376           v18.09.6   
k8s-node2   -        generic   Running   tcp://192.168.20.214:2376           v18.09.6 
```
# 管理machine
用 `docker-machine` 创建 machine 的过程很简洁，非常适合多主机环境。除此之外，Docker Machine 也提供了一些子命令方便对 machine 进行管理。其中最常用的就是无需登录到 machine 就能执行 docker 相关操作。
## 查看环境变量
Docker Machine 则让这个过程更简单。docker-machine env k8s-node1显示访问k8s-node1 需要的所有环境变量：
```bash
$ docker-machine env k8s-node1
export DOCKER_TLS_VERIFY="1"
export DOCKER_HOST="tcp://192.168.20.213:2376"
export DOCKER_CERT_PATH="/root/.docker/machine/machines/k8s-node1"
export DOCKER_MACHINE_NAME="k8s-node1"
# Run this command to configure your shell: 
# eval $(docker-machine env k8s-node1)
```
<div class="note primary"><p>根据提示，执行 `eval $(docker-machine env k8s-node1)` 会把当前shell环境设置为k8s-node1 machine的环境，之后管理docker的命令操作相当于在k8s-node1上。</p></div>

## 启动容器
在此状态下执行的所有 docker 命令其效果都相当于在 k8s-node1 上执行，可以通过`docker-machine active`查看当前shell对应哪个machine。例如启动一个 busybox 容器：
```bash
$ docker run --name b1 -itd busybox
7079c9d8283994ba77a93bddedd76856c74617e73eb374485a12247d61998e86

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
7079c9d82839        busybox             "sh"                2 seconds ago       Up 5 second                             b1
```
## docker-machine ssh
除了上面的使用方法外，我们还有更加简便的方式，就是使用docker-machine ssh，需要我们提前配置好 /etc/hosts，比如上面的命令还可以直接通过如下指令创建。
```bash
# 以ssh方式登陆k8s-node1 machine主机运行docker container run命令
$ docker-machine ssh k8s-node1 "docker container run --name b1 -itd busybox"
# 以ssh方式登陆k8s-node1 machine主机运行docker container ls命令
$ docker-machine ssh k8s-node1 "docker container ls" 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
7079c9d82839        busybox             "sh"                2 minutes ago       Up 2 minutes                            b1
```
## mount挂载卷
`docker-machine`Mount使用sshfs将目录从machine挂载到本地主机,方便我们管理。
<div class="note success"><p>前提需要安装`fuse-sshfs`，yum install -y fuse-sshfs</p></div>

例如，在docker-machine主机上如下操作：
```bash
$ mkdir /opt/foo
$ docker-machine ssh k8s-node1 mkdir foo
$ docker-machine mount k8s-node1:/root/foo /opt/foo
$ touch foo/bar
$ docker-machine ssh k8s-node1 ls foo
bar
```
以上命令为在k8s-node1上当前目录创建一个foo挂载目录，然后使用docker-machine mount将该目录挂载到本地/opt/foo目录下，我们可以在本地执行mount命令查看挂载情况：
```bash
#通过mount可以看到挂载的是machine目录到本地
$ mount | grep foo
root@k8s-node1:/root/foo on /opt/foo type fuse.sshfs (rw,nosuid,nodev,relatime,user_id=0,group_id=0)
```
那么现在，您可以使用machine机器上的目录来挂载到容器。而在当前docker-machine主机上就可以维护容器里的数据了。
```bash
#进入k8s-node1的docker shell环境，并且启动一个t1的容器并挂载/root/foo目录到容器/tmp/foo下
$ eval $(docker-machine env k8s-node1)
$ docker run --name t1 --rm -v /root/foo:/tmp/foo busybox ls /tmp/foo
bar
$ touch foo/baz
$ docker run --name t1 --rm -v /root/foo:/tmp/foo busybox ls /tmp/foo
bar
baz
```
这些文件实际上是通过sftp(通过ssh连接)传输的，所以这个程序(“sftp”)需要在机器上显示, 通常是这样的。

## 卸载挂载卷
要再次卸载该目录，可以使用相同的选项，但使用-u标志。您还可以直接调用fuserunmount(或fusermount -u)命令。
```bash
$ docker-machine mount -u k8s-node1:/root/foo /opt/foo
```