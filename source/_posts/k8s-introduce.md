layout: post
title: Kubernetes 核心概念介绍
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
* **`APIServer`**
APIServer负责对外提供RESTful的Kubernetes API服务，它是系统管理指令的统一入口，任何对资源进行`GET`、`PUT` 、`DELETE` 、`POST` 的操作都要交给APIServer处理后再提交给etcd。如架构图中所示，kubectl（Kubernetes提供的客户端工具，该工具内部就是对Kubernetes API的调用）是直接和APIServer交互的。
* **`scheduler`**
scheduler的职责很明确，就是负责调度pod到合适的Node上。如果把scheduler看成一个黑匣子，那么它的输入是pod和由多个Node组成的列表，输出是Pod和一个Node的绑定，即将这个pod部署到这个Node上。Kubernetes目前提供了调度算法，但是同样也保留了接口，用户可以根据自己的需求定义自己的调度算法。
* **`controller-manager`**
如果说APIServer做的是“前台”的工作的话，那controller-manager就是负责“后台”的。每个资源一般都对应有一个控制器，而controller-manager就是负责管理这些控制器的。比如我们通过APIServer创建一个pod，当这个pod创建成功后，APIServer的任务就算完成了。而后面保证Pod的状态始终和我们预期的一样的重任就由controller-manager去保证了。
* **`etcd`**
etcd是一个高可用的键值存储系统，Kubernetes使用它来存储各个资源的状态，从而实现了Restful的API。对于kubernetes集群来说他的客户端只有apiserver，任何资源的变动都要提交给apiserver，由apiserver当etcd的客户端写入etcd。

## Node节点组件
每个Node节点主要由三个模块组成：`kubelet`、`kube-proxy`、`runtime`。
* **`runtime`**
runtime指的是容器运行环境，目前Kubernetes最主流的runtime就是docker了。
* **`kube-proxy`**
该模块实现了Kubernetes中的`服务发现`和`反向代理`功能。反向代理方面：kube-proxy支持TCP和UDP连接转发，默认基于Round Robin算法将客户端流量转发到与service对应的一组后端pod。服务发现方面，kube-proxy使用etcd的watch机制，监控集群中service和endpoint对象数据的动态变化，并且维护一个service到endpoint的映射关系，从而保证了后端pod的IP变化不会对访问者造成影响。另外kube-proxy还支持session affinity。
* **`kubelet`**
Kubelet是Master在每个Node节点上面的agent，是Node节点上面最重要的模块，它负责维护容器的生命周期，同时也负责 `Volume（CVI）`和`网络（CNI）`的管理，并将状态信息反馈给master。但是如果容器不是通过Kubernetes创建的，它并不会管理。

## 插件（Addons）
部署 Kubernetes 集群后，还需要部署一系列的附加组件（addons），这些组件通常是保证集群功能正常运行必不可少的。通常使用`addon-manager`来管理集群中的附加组件。它运行在 Kubernetes集群Master节点中，管理着所有扩展，保证它们始终运行在期望状态。
必须安装的插件组件有：
* `coredns`: 他为整个kubernetes集群提供DNS解析，而且是动态的DNS解析，他会随着pod或者service的配置变动而动态解析。
* `flannel和calio`： 比较主流的两款为kubernetes提供跨主机节点的网络通信的`CNI` (container network interface)。
* `kubernetes-dashboard`：为kubernetes提供web界面的管理方式。
* `metrics-server`: 监控整个kubernetes集群资源状态数据信息，代替早期1.11版本前的`heapster`。

# Kubernetes对象资源
`对象资源`是Kubernetes集群中的管理操作单元。基本的资源对象有：`Pod` 、`Service` 、`Namespace` 、`Job`，更高级的资源对象有：`Deployment` 、`DaemonSet` 、`StatefulSet` 、`ConfigMap`、 `Volume` 等。下面我们将对这些资源进行说明。

> 在kubernetes中所有的配置定义都应该是声明式（Declarative）的而不是命令式（Imperative）的。

## Pod
Pod 是Kubernetes的最小基本操作单元，也是应用运行的载体。整个Kubernetes系统都是围绕着Pod展开的，比如如何部署运行Pod、如何保证Pod的数量、如何访问Pod等。另外，一个Pod是一个或多个紧密关联容器的集合，一个Pod也是一个隔离体，而Pod内部包含的一组容器又是共享的（包括PID、Network、IPC、UTS）。除此之外，一个Pod中的容器可以访问共同的数据卷来实现文件系统的共享。

### 创建Pod
通常把创建Pod分为两类：
* **`自主式Pod`**： 这种Pod本身是不能自我修复的，当Pod被创建后，会被Kuberentes调度到集群的Node上。直到Pod的进程终止、被删掉、因为缺少资源而被驱逐、Node故障或者调度器本身故障时，这个Pod就会被删除，不会自动重建。
* **`控制器管理的Pod`**： Kubernetes使用更高级的称为Controller的抽象层，来管理Pod实例。Controller可以创建和管理多个Pod，提供副本管理、滚动升级和集群级别的自愈能力。例如，如果一个Node故障，Controller就能自动将该节点上的Pod调度到其他健康的Node上。

<div class="note primary"><p>注: 每个Pod都有一个基础的`Pause`容器，Pause容器对应的镜像属于Kubernetes平台的一部分，而Pod中的业务容器正是共享了该基础容器的`PID` `UTS` `Network` `IPC`。</p></div>

### Pod生命周期
Pod被分配到一个Node上之后，就不会离开这个Node，直到被删除。当某个Pod失败，首先会被Kubernetes清理掉，之后Pod对应的控制器将会在其它机器上（或本机）重建Pod，重建之后Pod的ID发生了变化，那将会是一个新的Pod。所以，Kubernetes中Pod的迁移，实际指的是在新Node上重建Pod。它生命周期有如下阶段(phase)：

* `Pending`: API Server创建了Pod资源对象并已经存入了etcd中，但是它并未被调度完成，或者仍然处于从仓库下载镜像的过程中。
* `Running`: Pod已经被调度到了某个节点，所有容器已经被创建完成。
* `Successded`: Pod中的所有容器成功终止，也不会重启。
* `Failed`: 所有容器终止，至少有一个容器以失败方式终止。也就是说，这个容器要么已非 0 状态退出，要么被系统终止。
* `Unknown`: 由于一些原因，Pod 的状态无法获取，通常是与 Pod 通信时出错导致的。

下图是Pod的生命周期示意图，从图中可以看到Pod状态的变化。

![](/images/pic/k8s/pod-phase.jpg)

另外在pod生命周期中可以做的一些事情。主容器启动前可以完成初始化容器，初始化容器可以有多个，他们是串行执行的，执行完成后就退出了，在主容器刚刚启动的时候可以指定一个`post start` 主容器启动开始后执行一些操作，在主容器结束前可以指定一个`pre stop` 表示主容器结束前执行的一些操作。在容器启动后可以做两类检测 `livenessprobe（存活性探测）`和 `readinessprobe（就绪性探测）`。如下图：

![](/images/pic/k8s/pod-check.jpg)

# Service
为了适应快速的业务需求，微服务架构已经逐渐成为主流，微服务架构的应用需要有非常好的服务编排支持。Kubernetes中的核心要素Service便提供了一套简化的服务发现和负载均衡，天然适应微服务架构。
在Kubernetes中，当Deployment创建pod后，Pod副本是变化的，对于的Pod IP也是变化的，比如发生迁移或者伸缩的时候。这对于Pod的访问者来说是不可接受的。Kubernetes中的Service是一种抽象概念，它定义了一个Pod逻辑集合以及访问它们的策略，Service同Pod的关联同样是居于Label标签来完成的。Service的目标是提供一种桥梁， 它会为访问者提供一个固定访问地址，我们称它为`ClusterIP`或者`ServiceIP`，该IP是Kubernetes通过iptables规则做转发分配给Service的一个虚拟IP，用于Kube-proxy组件来实现的虚拟IP路由及负载均衡访问后端相应的endPoint。
外部访问Pod，Kubernetes提供了NodePort、LoadBalancer、Ingress三种转发方式：

* `NodePort`: Kubernetes会在每一个Node上暴露出一个端口：nodePort，与之关联serviceIP，外部网络可以通过（任一Node）[NodeIP]:[NodePort]访问到后端的Service。
* `LoadBalancer`： 在NodePort基础上，Kubernetes可以请求底层云平台创建一个负载均衡器，将每个Node作为后端，进行服务分发。该模式需要底层云平台（例如GCE）支持。
* `Ingress`: Ingress通过http代理服务器将外部的http请求转发到集群内部的后端服务。Ingress Controller实时监控Kubernetes API，实时更新HTTP代理服务器的转发规则。

# Deployment
Deployment中描述你所期望的集群状态，用于管理Pod，其实真正控制Pod的是`ReplicaSet`，而Deployment控制的是ReplicaSet。也就是说 Deployment 是通过 ReplicaSet 来管理 Pod 的多个副本，我们通常不需要直接使用 ReplicaSet。 当更新时`Deployment Controller`会将该控制器管理的Pod逐步更新成你所期望的集群状态。Deployment主要职责同样是为了保证pod的数量和健康，90%的功能与Replication Controller（弃用）完全一样，可以看做新一代的Replication Controller。但是，它又具备了Replication Controller之外的新特性：

* `Replication Controller全部功能`：Deployment继承了Replication Controller全部功能。
* `事件和状态查看`：可以查看Deployment的升级详细进度和状态。
* `回滚`：当升级pod镜像或者相关参数的时候发现问题，可以使用回滚操作回滚到上一个稳定的版本或者指定的版本。
* `版本记录`: 每一次对Deployment的操作，都能保存下来，给予后续可能的回滚使用。
* `暂停和启动`：对于每一次升级，都能够随时暂停和启动。比如金丝雀发布。
* `多种升级方案`：Recreate----删除所有已存在的pod,重新创建新的; RollingUpdate----滚动升级，逐步替换的策略，同时滚动升级时，支持更多的附加参数，例如设置最大不可用pod数量，最小升级间隔时间等等。

# ReplicaSet
ReplicaSet是用来来取代 ReplicationController。ReplicaSet 跟 ReplicationController 没有本质的不同，只是名字不一样，并且 ReplicaSet 支持集合式的 selector（ReplicationController 仅支持等式），虽然 ReplicaSet也可以独立使用，但是我们日常都使用 Deployment 来自动管理 ReplicaSet。 而当对Pod进行升级时，旧的Replicaset也一直存在，Deployment默认可以管理10个Replicaset用于回滚操作。

# DaemonSet
DaemonSet确保所有（或某些标记为污点）节点都只运行一个Pod副本。 Pod随着节点添加而自动添加一个Pod到该节点。 随着节点从群集中删除而自动从该节点删除。 删除DaemonSet将清除它创建的所有Pod。它和Deployment一样也是一个无状态的控制器。
DaemonSet的一些典型用法是：

* 运行集群存储后台守护程序，在每个节点上运行如：glusterd，ceph。
* 在每个节点上运行日志收集守护程序，例如：fluentd或logstash。
* 在每个节点上运行节点监视守护程序，例如：Prometheus Node Exporter，Sysdig Agent，collectd，Dynatrace OneAgent，AppDynamics Agent，Datadog proxy，New Relic proxy，Ganglia gmond或Instana Agent。

# StatefuleSet
在云原生应用的体系里，有下面两组近义词；第一组是无状态（stateless）、牲畜（cattle）、无名（nameless）、可丢弃（disposable）；第二组是有状态（stateful）、宠物（pet）、有名（having name）、不可丢弃（non-disposable）。Deployment和RS主要是控制提供无状态服务的，其所控制的Pod的名字是随机设置的，一个Pod出故障了就被丢弃掉，在另一个地方重启一个新的Pod，名字变了、名字和启动在哪儿都不重要，重要的只是Pod总数。而StatefuleSet是用来控制有状态服务，能够保证 Pod 的每个副本在整个生命周期中名称是不变的。且StatefuleSet 会保证副本在启动时按照固定的顺序启动，在更新或者删除会按照逆序进行。

那么适用于StatefulSet业务包括数据库服务MySQL和PostgreSQL，集群化管理服务Zookeeper、etcd等有状态服务。StatefuleSet的另一种典型应用场景是作为一种比普通容器更稳定可靠的模拟虚拟机的机制。传统的虚拟机正是一种有状态的宠物，运维人员需要不断地维护它，容器刚开始流行时，我们用容器来模拟虚拟机使用，所有状态都保存在容器里，而这已被证明是非常不安全、不可靠的。使用StatefuleSet，Pod仍然可以通过漂移到不同节点提供高可用，而存储也可以通过外挂的存储来提供高可靠性，StatefuleSet做的只是将确定的Pod与确定的存储关联起来保证状态的连续性。

# ConfigMap
在很多业务环境中的应用程序配置较为复杂，可能需要多个配置文件、命令行参数和环境变量的组合。并且，这些配置信息应该从应用程序镜像中解耦出来，以保证镜像的可移植性以及配置信息不被泄露。社区引入ConfigMap这个API资源来满足这一需求。ConfigMap包含了一系列的键值对，用于存储被Pod或者系统组件（如controller）访问的信息。这与secret的设计理念有异曲同工之妙，它们的主要区别在于ConfigMap通常不用于存储敏感信息，而只存储简单的文本信息。

# Secret
Kubernetes的Secret对象允许您存储和管理敏感信息，例如：密码，OAuth令牌和ssh密钥。 将此信息存放于Secret中比将其逐字地放入Pod定义或容器映像中更安全，更灵活。 目前secret支持创建以下三种类型的secret：

* `docker-resgistry`: 创建一个给 Docker registry 使用的 secret
* `generic`: 从本地 file, directory 或者 literal value 创建一个 secret
* `tls`: 创建一个 TLS secret

# Job
Job是创建一个或多个Pod并确保该pod成功运行后终止。 当pod成功完成后，Job会跟踪成功的完成情况。 达到指定数量的成功完成时，Job（即作业）完成。 该Pod将自动删除。通常运行一次性的脚本或者备份数据可以使用该类型。

# Volume
在Docker的设计实现中，容器中的数据是临时的，即当容器被销毁时，其中的数据将会丢失。如果需要持久化数据，需要使用Docker数据卷挂载宿主机上的文件或者目录到容器中。在Kubernetes中，当Pod重建的时候，数据也是会丢失的，Kubernetes也是通过数据卷挂载来提供Pod数据的持久化的。Kubernetes数据卷是对Docker数据卷的扩展，Kubernetes数据卷是Pod级别的，可以用来实现Pod中容器的文件共享。
kubernetes支持的挂载卷使用以下命令查看：
```bash
$ kubectl explain pod.spec.volumes
```
## EmptyDir
如果Pod配置了EmpyDir数据卷，在Pod的生命周期内都会存在，当Pod被分配到 Node上的时候，会在Node上创建EmptyDir数据卷，并挂载到Pod的容器中。只要Pod 存在，EmpyDir数据卷都会存在, 但是如果Pod的生命周期终结（Pod被删除），EmpyDir数据卷也会被删除，并且永久丢失。

## HostPath
HostPath数据卷允许将容器宿主机上的文件系统挂载到Pod中。如果Pod需要使用宿主机上的某些文件，可以使用HostPath，此时Pod被删除数据不会被丢失，只有对应的Node节点挂掉，数据才丢失。或Pod重建时被分配到其他节点上后，该数据也不能使用。

## 网络数据卷
Kubernetes提供了很多类型的数据卷以集成第三方的存储系统，包括一些非常流行的分布式文件系统，也有在IaaS平台上提供的存储支持，这些存储系统都是分布式的，通过网络共享文件系统，因此我们称这一类数据卷为网络数据卷。
网络数据卷能够满足数据的持久化需求，Pod通过配置使用网络数据卷，每次Pod创建的时候都会将存储系统的远端文件目录挂载到容器中，数据卷中的数据将被水久保存，即使Pod被删除，只是除去挂载数据卷，数据卷中的数据仍然保存在存储系统中，且当新的Pod被创建的时候，仍是挂载同样的数据卷。网络数据卷包含以下几种：`NFS`、`iSCISI`、`GlusterFS`、`RBD（Ceph Block Device）`、`Flocker`、`AWS Elastic Block Store`、`GCE Persistent Disk`。

## PersistentVolume（PV）和PersistentVolumeClaim(PVC)
理解每个存储系统是一件复杂的事情，特别是对于普通用户来说，有时候并不需要关心各种存储实现，只希望能够安全可靠地存储数据。Kubernetes中提供了`Persistent Volume`和`Persistent Volume Claim`机制，这是存储消费模式。Persistent Volume是由系统管理员配置创建的一个数据卷，它代表了某一类存储插件实现；而对于普通用户来说，通过Persistent Volume Claim可请求并获得合适的Persistent Volume，而无须感知后端的存储实现。Persistent Volume和Persistent Volume Claim的关系其实类似于Pod和Node，Pod消费Node资源，Persistent Volume Claim则消费Persistent Volume资源。Persistent Volume和Persistent Volume Claim相互关联，有着完整的生命周期管理。