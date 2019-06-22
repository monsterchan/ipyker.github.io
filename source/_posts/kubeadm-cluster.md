layout: post
title: kubeadm安装Kubernetes1.14.3单master集群
author: Pyker
categories: kubernetes
tags:
  - kubernetes
date: 2019-06-12 18:12:00
---

`kubeadm`是Kubernetes官方提供的用于快速安装Kubernetes集群的工具，伴随Kubernetes每个版本的发布都会同步更新，kubeadm会对集群配置方面的一些实践做调整，通过实验kubeadm可以学习到Kubernetes官方在集群配置上一些新的最佳实践。

# 集群环境
本次构建kubernetes集群是在ESXI主机上创建4个VM虚拟机进行演示的，docker-ce版本为`18.09.6`，kubernetes组件均为`1.14.3`版本，环境信息如表所示：

系统 | 节点角色 | ip | Hostname | 安装组件
--- | --- | --- | --- | ---
centos7.4 | master | 192.168.20.210 | master1 | docker-ce、 kubeadm、 kubelet
centos7.4 | worker | 192.168.20.213 | node1 | docker-ce、 kubeadm、 kubelet
centos7.4 | worker | 192.168.20.214 | node2 | docker-ce、 kubeadm、 kubelet
centos7.4 | worker | 192.168.20.215 | node3 | docker-ce、 kubeadm、 kubelet

# 安装前准备
{% note primary %}注： 在所有节点上进行以下操作。{% endnote %}

## 配置SSH免密钥登陆
```bash
# master1上生成密钥对
$ ssh-keygen -t rsa 
$ ssh-copy-id -i ~/.ssh/id_rsa.pub node1
$ ssh-copy-id -i ~/.ssh/id_rsa.pub node2
$ ssh-copy-id -i ~/.ssh/id_rsa.pub node3
```

## 设置主机名
```bash
$ cat >> /etc/hosts << EOF
192.168.20.210   master1.ipyker.com  master1
192.168.20.211   master2.ipyker.com  master2
192.168.20.212   master3.ipyker.com  master3
192.168.20.213   node1.ipyker.com  node1
192.168.20.214   node2.ipyker.com  node2
192.168.20.215   node3.ipyker.com  node3
EOF
```

## 禁用防火墙
```bash
# 禁用iptables和firewalld开机自启动
$ systemctl disable iptables
$ systemctl disable firewalld
```

## 时间同步
```bash
$ systemctl restart chronyd
```

## 禁用Selinux
```bash
$ sed -i "s/^SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
```

## 关闭Swap
```bash
$ swapoff -a
$ sed -i 's/.*swap.*/#&/' /etc/fstab
```
> 如果你的集群有其他业务不能关闭swap时，需要在安装kubelet后，修改/etc/sysconfig/kubelet内容为：`KUBELET_EXTRA_ARGS=--fail-swap-on=false`

## 添加kubernetes内核参数
```bash
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF
```

## 加载ipvs模块
```bash
$ vi /etc/sysconfig/modules/ipvs.modules
#!/bin/sh
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs/"
for mod in `ls ${ipvs_mods_dir} | grep -o "^[^.]*"`; do
    /sbin/modprobe $mod
done

$ chmod +x /etc/sysconfig/modules/ipvs.modules && ./ipvs.modules
```
检查模块是否加载生效,如果如下所示表示已经加载ipvs模版到内核了。
```bash
$ lsmod | grep ip_vs
ip_vs_wrr              12697  0 
ip_vs_wlc              12519  0 
ip_vs_sh               12688  0 
ip_vs_sed              12519  0 
ip_vs_rr               12600  20 
ip_vs_pe_sip           12740  0 
nf_conntrack_sip       33860  1 ip_vs_pe_sip
ip_vs_nq               12516  0 
ip_vs_lc               12516  0 
ip_vs_lblcr            12922  0 
ip_vs_lblc             12819  0 
ip_vs_ftp              13079  0 
nf_nat                 26787  4 ip_vs_ftp,nf_nat_ipv4,nf_nat_ipv6,nf_nat_masquerade_ipv4
ip_vs_dh               12688  0 
ip_vs                 141432  44 ip_vs_dh,ip_vs_lc,ip_vs_nq,ip_vs_rr,ip_vs_sh,ip_vs_ftp,ip_vs_sed,ip_vs_wlc,ip_vs_wrr,ip_vs_pe_sip,ip_vs_lblcr,ip_vs_lblc
nf_conntrack          133053  10 ip_vs,nf_nat,nf_nat_ipv4,nf_nat_ipv6,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_netlink,nf_conntrack_sip,nf_conntrack_ipv4,nf_conntrack_ipv6
libcrc32c              12644  4 xfs,ip_vs,nf_nat,nf_conntrack
```
> 请确保ipset也已经安装了，如未安装请执行`yum install -y ipset`安装。

# 安装docker-ce
<div class="note primary"><p>注： 在所有节点上进行以下操作。</p></div>

## 配置docker-ce的yum源
```bash
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```

## 安装docker-ce
```bash
# 这里默认为18.09.6最新版本
$ yum install docker-ce -y
```
<div class="note danger"><p>如果要安装指定版本的docker-ce，请先查看版本`yum list docker-ce --showduplicates | sort -r`</p></div>

## 配置docker加速
```bash
$ cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://yuife6vp.mirror.aliyuncs.com"]
}
EOF
```

## 配置iptables FORWARD
默认情况下docker-ce将iptables的FORWARD默认规则设置为DROP，这样会引起Kubernetes集群中跨Node的Pod无法通信。我们需要将此规则设置为默认ACCEPT。
```bash
# 命令行设置，重启失效
$ iptables -P FORWARD ACCEPT
# 在docker的systemctl文件[Service]中添加ExecStartPost
$ vi /usr/lib/systemd/system/docker.service  
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
```

## 配置docker科学上网
```bash
# 在docker的systemctl文件[Service]中添加Environment
$ vi /usr/lib/systemd/system/docker.service  
Environment="HTTP_PROXY=192.168.20.199:1080"
Envrionment="NO_PROXY=127.0.0.0/8,192.168.20.0/23"
```
<div class="note warning"><p>这里需要你有科学上网的代理服务器，配置科学上网的目的是使docker能够pull gcr.io上的镜像，如果没有也可以在其他地方下载，然后通过docker tag成kubernetes需要的镜像名。</p></div>

## 启动docker
```bash
$ systemctl daemon-reload
$ systemctl start docker
$ systemctl enable docker
```

# 安装kubeadm 和 kubelet
<div class="note primary"><p>注： 在所有节点上进行以下操作。</p></div>

```bash
$ cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes Repository
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
gpgcheck=1
gpgkey=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
       https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
EOF
```

```bash
$ yum install -y kubelet kubeadm kubectl
```
```bash
# 设置开机自启动kubelet
$ systemctl enable kubelet
$ systemctl start kubelet
```
`kubelet`负责与其他节点集群通信，并进行本节点Pod和容器生命周期的管理。`kubeadm`是Kubernetes的自动化部署工具，降低了部署难度，提高效率。`kubectl`是Kubernetes集群客户端管理工具。

# 部署kubernetes master节点
## 查看kubeadm初始化配置
```bash
$ kubeadm config print init-defaults
apiVersion: kubeadm.k8s.io/v1beta1
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 1.2.3.4
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: master1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: ""
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.14.0
networking:
  dnsDomain: cluster.local
  podSubnet: ""
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```
这里可以查看到当执行`kubeadm init`初始化集群时的配置信息，其中`kubernetesVersion`版本为v1.14.0，并不是我们需要的v1.14.3，所以需要手动指定，`serviceSubnet`指定pod的网络，我们这里等会使用flannel网络，默认是10.244.0.0/16，所以也要手动指定。`imageRepository`是获取镜像的仓库地址，该地址需要科学上网才能访问, 也可以在`kubeadm init`时使用`--image-repository`参数指定仓库地址。

## 获取依赖镜像
这里手动拉取master所需要的镜像是为了等会kubeadm init时能够快些，也可以不手动拉取。在kubeadm init时也会自动拉取。
```bash
$ kubeadm config images pull
```
如果你能科学上网，执行`kubeadm config images pull`后将自动拉取kubeadm对应版本的master节点需要的镜像，拉取完需要的镜像如下：
```bash
$ docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.3             004666307c5b        8 days ago          82.1MB
k8s.gcr.io/kube-controller-manager   v1.14.3             ac2ce44462bc        8 days ago          158MB
k8s.gcr.io/kube-apiserver            v1.14.3             9946f563237c        8 days ago          210MB
k8s.gcr.io/kube-scheduler            v1.14.3             953364a3ae7a        8 days ago          81.6MB
quay.io/coreos/flannel               v0.11.0-amd64       ff281650a721        4 months ago        52.6MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        5 months ago        40.3MB
k8s.gcr.io/etcd                      3.3.10              2c4adeb21b4f        6 months ago        258MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        18 months ago       742kB
```
当你不能科学上网时，可以手动从别的镜像仓库中下载，这样也能达到我们的要求，如下操作：
```bash
$ vi /root/kubernetes-images.sh
#!/bin/sh
PACKAGE=(kube-apiserver:v1.14.3 kube-controller-manager:v1.14.3 kube-scheduler:v1.14.3 kube-proxy:v1.14.3 pause:3.1 etcd:3.3.10)
for pack in "${PACKAGE[@]}"; do
        docker pull mirrorgooglecontainers/${pack}
        docker tag docker.io/mirrorgooglecontainers/${pack} k8s.gcr.io/${pack}
        docker rmi docker.io/mirrorgooglecontainers/${pack}
done
docker pull coredns/coredns:1.3.1
docker tag docker.io/coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1
docker rmi docker.io/coredns/coredns:1.3.1

$ sh /root/kubernetes-images.sh
```

## 初始化master集群
```bash
$ kubeadm init --kubernetes-version="v1.14.3" --pod-network-cidr="10.244.0.0/16"
[init] Using Kubernetes version: v1.14.3
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Activating the kubelet service
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [master1 localhost] and IPs [192.168.20.210 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [master1 localhost] and IPs [192.168.20.210 127.0.0.1 ::1]
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [master1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.96.0.1 192.168.20.210]
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 15.501532 seconds
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --experimental-upload-certs
[mark-control-plane] Marking the node master1 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: ankpak.4mjrj58zunfd79mo
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.20.210:6443 --token ankpak.4mjrj58zunfd79mo \
    --discovery-token-ca-cert-hash sha256:ef3875ec75bb073fe41e480c9c4d18d18de5cec5270f45fc88f3a5dcea8deeaf
```
此时master集群已经初始化完成。

## 配置执行kubectl命令用户
```bash
# kubernetes建议使用非root用户运行kubectl命令访问集群
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

## 配置flannel网络
```bash
$ wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
$ kubectl apply -f kube-flannel.yml
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.extensions/kube-flannel-ds-amd64 created
daemonset.extensions/kube-flannel-ds-arm64 created
daemonset.extensions/kube-flannel-ds-arm created
daemonset.extensions/kube-flannel-ds-ppc64le created
daemonset.extensions/kube-flannel-ds-s390x created
```
如果你的网络不能下载flannel的镜像，请手动修改`kube-flannel.yml`文件的image为`registry.cn-shenzhen.aliyuncs.com/pyker/flannel:v0.11.0-amd64`，下载完成后，请执行：
```bash
$ docker tag registry.cn-shenzhen.aliyuncs.com/pyker/flannel:v0.11.0-amd64 quay.io/coreos/flannel:v0.11.0-amd64
```
然后在运行 `kubectl apply -f kube-flannel.yml`

此时我们的master节点已经准备好了，可以通过`kubectl get nodes`查看
```bash
$ kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   3h11m   v1.14.3
```
# 部署kubernetes node节点
还记得我们上文在master节点上执行`kubeadm init`命令最后生成的kubeadm join命令吗？那就是我们node节点加入kubernetes集群的命令。
<div class="note primary"><p>注： 以下命令在所有node节点上操作。</p></div>

## node加入集群
```bash
$ kubeadm join 192.168.20.210:6443 --token ankpak.4mjrj58zunfd79mo --discovery-token-ca-cert-hash sha256:ef3875ec75bb073fe41e480c9c4d18d18de5cec5270f45fc88f3a5dcea8deeaf
preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```
<div class="note primary"><p>此时node也会拉取gcr.io上的镜像，需要科学上网，或者手动获取如下镜像：</p></div>

```bash
$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy    v1.14.3             004666307c5b        8 days ago          82.1MB
quay.io/coreos/flannel   v0.11.0-amd64       ff281650a721        4 months ago        52.6MB
k8s.gcr.io/pause         3.1                 da86e6ba6ca1        18 months ago       742kB
```
此时node已经全部加入集群。我们可以在master上执行以下命令查看集群信息：
```bash
# 查看当前集群节点状态
$ kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   3h21m   v1.14.3
node1     Ready    <none>   3h9m    v1.14.3
node2     Ready    <none>   3h8m    v1.14.3
node3     Ready    <none>   3h8m    v1.14.3
```

```bash
# 查看集群组件健康状态
kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}
```

```bash
# 查看kube-system名称空间中所有控制器信息，以下显示了pod状态信息、service状态信息、daemonset状态信息、deployment状态信息以及replicaset状态信息。
$ kubectl get all -n kube-system
NAME                                  READY   STATUS    RESTARTS   AGE
pod/coredns-fb8b8dccf-jw556           1/1     Running   0          3h22m
pod/coredns-fb8b8dccf-s69bn           1/1     Running   0          3h22m
pod/etcd-master1                      1/1     Running   0          3h21m
pod/kube-apiserver-master1            1/1     Running   0          3h21m
pod/kube-controller-manager-master1   1/1     Running   0          3h21m
pod/kube-flannel-ds-amd64-9r4ds       1/1     Running   0          3h10m
pod/kube-flannel-ds-amd64-gk5jt       1/1     Running   0          3h9m
pod/kube-flannel-ds-amd64-gpv97       1/1     Running   0          3h9m
pod/kube-flannel-ds-amd64-w4c64       1/1     Running   0          3h13m
pod/kube-proxy-7jxlr                  1/1     Running   0          3h9m
pod/kube-proxy-hwl4v                  1/1     Running   0          3h10m
pod/kube-proxy-m7l8g                  1/1     Running   0          3h9m
pod/kube-proxy-r9n7m                  1/1     Running   0          3h22m
pod/kube-scheduler-master1            1/1     Running   0          3h21m

NAME               TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
service/kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP   3h22m

NAME                                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                     AGE
daemonset.apps/kube-flannel-ds-amd64     4         4         4       4            4           beta.kubernetes.io/arch=amd64     3h13m
daemonset.apps/kube-flannel-ds-arm       0         0         0       0            0           beta.kubernetes.io/arch=arm       3h13m
daemonset.apps/kube-flannel-ds-arm64     0         0         0       0            0           beta.kubernetes.io/arch=arm64     3h13m
daemonset.apps/kube-flannel-ds-ppc64le   0         0         0       0            0           beta.kubernetes.io/arch=ppc64le   3h13m
daemonset.apps/kube-flannel-ds-s390x     0         0         0       0            0           beta.kubernetes.io/arch=s390x     3h13m
daemonset.apps/kube-proxy                4         4         4       4            4           <none>                            3h22m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/coredns   2/2     2            2           3h22m

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/coredns-fb8b8dccf   2         2         2       3h22m
```
# 配置kubernetes使用ipvs
```bash
# 1、使用参数加载ipvs (master上操作)
$ cat >> /etc/sysconfig/kubelet << EOF
KUBE_PROXY_MODE=ipvs
EOF
# 2、通过在线修改kube-proxy配置文件
$ kubectl edit configMap kube-proxy -n kube-system
    ipvs:
      excludeCIDRs: null
      minSyncPeriod: 0s
      scheduler: ""
      strictARP: false
      syncPeriod: 30s
    kind: KubeProxyConfiguration
    metricsBindAddress: 127.0.0.1:10249
    mode: "ipvs"        # mode改为ipvs
    nodePortAddresses: null
    oomScoreAdj: -999
```

# master隔离
默认情况下，出于安全原因，kubernetes群集不会在master服务器上分配pod。如果您希望能够在master服务器上分配pod，例如，对于用于开发的单机Kubernetes集群，请运行：
```bash
$ kubectl taint nodes --all node-role.kubernetes.io/master-
node "test-01" untainted
taint "node-role.kubernetes.io/master:" not found
taint "node-role.kubernetes.io/master:" not found
```
这将从node-role.kubernetes.io/master包含master节点的任何节点中删除，这意味着调度程序将能够在任何地方分配pod。

# 未来node节点加入集群
节点是运行工作负载的位置。要向群集添加新节点，请为每台计算机执行以下操作：
```bash
$ kubeadm join --token <token> <master-ip>:<master-port> --discovery-token-ca-cert-hash sha256:<hash>
````
如果没有令牌，可以通过在主节点上运行以下命令来获取它：
```bash
$ kubeadm token list
TOKEN                     TTL       EXPIRES                     USAGES                   DESCRIPTION                                                EXTRA GROUPS
ankpak.4mjrj58zunfd79mo   20h       2019-06-15T15:29:36+08:00   authentication,signing   The default bootstrap token generated by 'kubeadm init'.   system:bootstrappers:kubeadm:default-node-token
```
默认情况下，令牌在24小时后过期。如果在当前令牌过期后将节点加入群集，则可以通过在master节点上运行以下命令来创建新令牌：
```bash
$ kubeadm token create
5didvk.d09sbcov8ph2amjw
```
如果没有值`--discovery-token-ca-cert-hash`，可以通过在主节点上运行以下命令链来获取它：
```bash
$ openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'
8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78
```
<div class="note primary"><p>注意：要指定IPv6的master地址`<master-ip>:<master-port>`，必须将IPv6地址括在方括号中，例如：[fd00::101]:2073。</p></div>

几秒钟后，您应该可以看到在master服务器上运行`kubectl get nodes`将会输出此节点信息。也可以直接执行以下命令，获取加入集群节点的命令：
```bash
# 这条命令比刚刚通过手动生成效率更高
$ kubeadm token create --print-join-command
```