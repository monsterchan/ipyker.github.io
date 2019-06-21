layout: post
title: kubernetes概念介绍
author: Pyker
categories: kubernetes
tags:
  - kubernetes
date: 2019-06-11 10:22:00
---

# Kubernetes是什么
Kubernetes是容器集群管理系统，是一个开源的平台，可以实现容器集群的`自动装箱`、`自动修复`、`水平扩展`、`服务发现`、`负载均衡`、`自动发布`和`回滚`等功能。
* `可移植` : 支持公有云，私有云，混合云，多重云（multi-cloud）
* `可扩展` : 模块化, 插件化, 可挂载, 可组合
* `自动化` : 自动部署，自动重启，自动复制，自动伸缩/扩展

# kubernetes基本架构
![](https://feisky.gitbooks.io/kubernetes/content/introduction/architecture.png)

# Kubernetes各个组件
kubernetes采用master/node的拓扑结构，因此master和node节点上的组件不一样，下面我们详细说明两者节点上的组件功能。

## Master节点组件
Master节点上的组件提供集群的管理控制中心。 在Master节点上面主要由四个组件组成：`APIServer`、`scheduler`、`controller-manager`、`etcd`。当然etcd不属于kubernetes的自带的组件，而是属于第三方的以`key/value`存储形式的存储组件，该组件也可以单独部署到其他独立的节点上运行。
* `APIServer`
APIServer负责对外提供RESTful的Kubernetes API服务，它是系统管理指令的统一入口，任何对资源进行`GET`、`PUT` 、`DELETE` 、`POST` 的操作都要交给APIServer处理后再提交给etcd。如架构图中所示，kubectl（Kubernetes提供的客户端工具，该工具内部就是对Kubernetes API的调用）是直接和APIServer交互的。
* `scheduler`
scheduler的职责很明确，就是负责调度pod到合适的Node上。如果把scheduler看成一个黑匣子，那么它的输入是pod和由多个Node组成的列表，输出是Pod和一个Node的绑定，即将这个pod部署到这个Node上。Kubernetes目前提供了调度算法，但是同样也保留了接口，用户可以根据自己的需求定义自己的调度算法。
* `controller-manager`
如果说APIServer做的是“前台”的工作的话，那controller-manager就是负责“后台”的。每个资源一般都对应有一个控制器，而controller-manager就是负责管理这些控制器的。比如我们通过APIServer创建一个pod，当这个pod创建成功后，APIServer的任务就算完成了。而后面保证Pod的状态始终和我们预期的一样的重任就由controller-manager去保证了。
* `etcd`
etcd是一个高可用的键值存储系统，Kubernetes使用它来存储各个资源的状态，从而实现了Restful的API。对于kubernetes集群来说他的客户端只有apiserver，任何资源的变动都要提交给apiserver，由apiserver当etcd的客户端写入etcd。

## Node节点组件
每个Node节点主要由三个模块组成：`kubelet`、`kube-proxy`、`runtime`。
* `runtime`
runtime指的是容器运行环境，目前Kubernetes最主流的runtime就是docker了。
* `kube-proxy`
该模块实现了Kubernetes中的`服务发现`和`反向代理`功能。反向代理方面：kube-proxy支持TCP和UDP连接转发，默认基于Round Robin算法将客户端流量转发到与service对应的一组后端pod。服务发现方面，kube-proxy使用etcd的watch机制，监控集群中service和endpoint对象数据的动态变化，并且维护一个service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响。另外kube-proxy还支持session affinity。
* `kubelet`
Kubelet是Master在每个Node节点上面的agent，是Node节点上面最重要的模块，它负责维护容器的生命周期，同时也负责 `Volume（CVI）`和`网络（CNI）`的管理，并将状态信息反馈给master。但是如果容器不是通过Kubernetes创建的，它并不会管理。

## 插件（Addons）
部署 Kubernetes 集群后，还需要部署一系列的附加组件（addons），这些组件通常是保证集群功能正常运行必不可少的。通常使用`addon-manager`来管理集群中的附加组件。它运行在 Kubernetes集群Master节点中，管理着所有扩展，保证它们始终运行在期望状态。
必须安装的插件组件有：
* `coredns`: 他为整个kubernetes集群提供DNS解析，而且是动态的DNS解析，他会随着pod或者service的配置变动而动态解析。
* `flannel和calio`： 比较主流的两款为kubernetes提供跨主机节点的网络通信的`CNI` (container network interface)。
* `kubernetes-dashboard`：为kubernetes提供web界面的管理方式。
* `metrics-server`: 监控整个kubernetes集群资源状态数据信息，代替早期1.11版本前的`heapster`。

# API对象资源
`API对象资源`是Kubernetes集群中的管理操作单元。基本的资源对象有：`Pod` 、`Service` 、`Namespace` 、`Job`，更高级的资源对象有：`Deployment` 、`DaemonSet` 、`StatefulSet` 、`ConfigMap`、 `Volume` 等。下面我们将对这些资源进行说明。

> 在kubernetes中所有的配置定义都应该是声明式（Declarative）的而不是命令式（Imperative）的。

## Pod
Pod 是Kubernetes的最小基本操作单元，也是应用运行的载体。整个Kubernetes系统都是围绕着Pod展开的，比如如何部署运行Pod、如何保证Pod的数量、如何访问Pod等。另外，一个Pod是一个或多个紧密关联容器的集合，一个Pod也是一个隔离体，而Pod内部包含的一组容器又是共享的（包括PID、Network、IPC、UTS）。除此之外，Pod中的容器可以访问共同的数据卷来实现文件系统的共享。

### Pod生命周期
Pod被分配到一个Node上之后，就不会离开这个Node，直到被删除。当某个Pod失败，首先会被Kubernetes清理掉，之后Deployment控制器将会在其它机器上（或本机）重建Pod，重建之后Pod的ID发生了变化，那将会是一个新的Pod。所以，Kubernetes中Pod的迁移，实际指的是在新Node上重建Pod。它生命周期有如下阶段：

* `Pending`: Pod已被 Kubernetes 接受，但尚未创建容器镜像。这包括被调度之前的时间以及通过网络下载镜像所花费的时间。
* `Running`: Pod已经被绑定到了一个节点，所有容器已被创建。至少一个容器正在运行，或者正在启动或重新启动。
* `Successded`: 所有容器成功终止，也不会重启。
* `Failed`: 所有容器终止，至少有一个容器以失败方式终止。也就是说，这个容器要么已非 0 状态退出，要么被系统终止。
* `Unknown`: 由于一些原因，Pod 的状态无法获取，通常是与 Pod 通信时出错导致的。
