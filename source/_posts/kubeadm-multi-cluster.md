layout: post
title: kubeadm安装Kubernetes1.14.3多master集群
author: Pyker
categories: kubernetes
tags:
  - kubernetes
date: 2019-06-15 12:20:00
---

前面我们已经使用[kubeadm安装Kubernetes1.14.3单master集群](https://www.ipyker.com/2019/06/12/kubeadm-cluster.html)，该架构部署的kubernetes集群比较适合开发、测试环境，并不太适用于生产环境，因为master存在单点故障，从而导致整个集群不可用。而鉴于此问题我们现在需要部署一套多master节点的kubernetes集群。此种架构的kubernetes master各个组件是通过选举leader经行高可用的。

# 集群环境
本次构建kubernetes集群是在ESXI主机上创建11个VM虚拟机进行演示的，docker-ce版本为`18.9.7`，kubernetes组件均为`1.14.3`版本，环境信息如表所示：

系统 | 节点角色 | ip | Hostname | 安装组件
--- | --- | --- | --- | ---
centos7.4 | master | 192.168.20.210 | master1 | docker-ce、 kubeadm、 kubelet
centos7.4 | master | 192.168.20.211 | master2 | docker-ce、 kubeadm、 kubelet
centos7.4 | master | 192.168.20.212 | master3 | docker-ce、 kubeadm、 kubelet
centos7.4 | worker | 192.168.20.213 | node1 | docker-ce、 kubeadm、 kubelet
centos7.4 | worker | 192.168.20.214 | node2 | docker-ce、 kubeadm、 kubelet
centos7.4 | worker | 192.168.20.215 | node3 | docker-ce、 kubeadm、 kubelet
centos7.4 | etcd | 192.168.20.216 | etcd1 | etcd
centos7.4 | etcd | 192.168.20.217 | etcd2 | etcd
centos7.4 | etcd | 192.168.20.218 | etcd3 | etcd
centos7.4 | ha | 192.168.20.219 | HA1 | keepalived、haproxy
centos7.4 | ha | 192.168.20.220 | HA2 | keepalived、haproxy
none | SLB vip | 192.168.20.222 | none | none

# 安装前准备
{% note primary %}注： 在所有节点上进行以下操作，除非另有说明。{% endnote %}

## 设置本地解析
```bash
$ cat >> /etc/hosts << EOF
192.168.20.210   master1.ipyker.com  master1
192.168.20.211   master2.ipyker.com  master2
192.168.20.212   master3.ipyker.com  master3
192.168.20.213   node1.ipyker.com  node1
192.168.20.214   node2.ipyker.com  node2
192.168.20.215   node3.ipyker.com  node3
192.168.20.216   etcd1.ipyker.com  etcd1
192.168.20.217   etcd2.ipyker.com  etcd2
192.168.20.218   etcd3.ipyker.com  etcd3
192.168.20.219   ha1.ipyker.com  ha1
192.168.20.220   ha2.ipyker.com  ha2
EOF
```

## 配置SSH免密钥登陆
```bash
# master1上生成密钥对
$ ssh-keygen -t rsa 
$ ssh-copy-id -i ~/.ssh/id_rsa.pub master1
$ ssh-copy-id -i ~/.ssh/id_rsa.pub master2
$ ssh-copy-id -i ~/.ssh/id_rsa.pub master3
$ ssh-copy-id -i ~/.ssh/id_rsa.pub node1
$ ssh-copy-id -i ~/.ssh/id_rsa.pub node2
$ ssh-copy-id -i ~/.ssh/id_rsa.pub node3
$ ssh-copy-id -i ~/.ssh/id_rsa.pub etcd1
$ ssh-copy-id -i ~/.ssh/id_rsa.pub etcd2
$ ssh-copy-id -i ~/.ssh/id_rsa.pub etcd3
$ ssh-copy-id -i ~/.ssh/id_rsa.pub ha1
$ ssh-copy-id -i ~/.ssh/id_rsa.pub ha2
```

## 禁用防火墙
```bash
# 禁用iptables和firewalld开机自启动
$ systemctl disable iptables
$ systemctl disable firewalld
```

## 时间同步
```bash
$ yum install -y ntp ntpdate ntp-doc    # 安装ntp服务
$ ntpdate ntp1.aliyun.com   # 使用阿里云的时间同步服务器
$ clock -w     # 系统时间写入blos时间
$ crontab -e   # 时间同步计划任务
*/5 * * * * /usr/sbin/ntpdate ntp1.aliyun.com
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
> 该参数只在kubernetes的master和node上配置即可。

```bash
$ cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
EOF
```

## 加载ipvs模块
> 该参数只在kubernetes的master和node上配置即可。

```bash
$ vi /etc/sysconfig/modules/ipvs.modules
#!/bin/sh
ipvs_mods_dir="/usr/lib/modules/$(uname -r)/kernel/net/netfilter/ipvs/"
for mod in `ls ${ipvs_mods_dir} | grep -o "^[^.]*"`; do
    /sbin/modprobe $mod
done

$ chmod +x /etc/sysconfig/modules/ipvs.modules && sh /etc/sysconfig/modules/ipvs.modules
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

# 安装负载均衡器
本次环境使用`keepalived`和`haproxy`来负载均衡`kube-apiserver`组件，该组件同一时刻只能启动一个，另外两个将阻塞，直到正在运行的kube-apiserver宕机，此时再通过选举策略选举出另外一个kube-apiserver。
> 只在HA1和HA2主机上操作。

## 安装工具包
```bash
$ yum install -y wget lrzsz vim epel-release
```

## 安装keepalive和haproxy软件包
```bash
$ yum install -y keepalived haproxy psmisc
```

## haproxy配置
```bash
# listen kube-master指定的为master地址和apiserver的6443端口
$ cat > /etc/haproxy/haproxy.cfg << EOF
global
    log 127.0.0.1    local0
    log 127.0.0.1    local1 notice
    chroot /var/lib/haproxy
    stats socket /var/run/haproxy-admin.sock mode 660 level admin
    stats timeout 30s
    user haproxy
    group haproxy
    daemon
    nbproc 1

defaults
    log     global
    timeout connect 5000
    timeout client  10m
    timeout server  10m

listen  admin_stats
    bind 0.0.0.0:10080
    mode http
    log 127.0.0.1 local0 err
    stats refresh 30s
    stats uri /status
    stats realm welcome login\ Haproxy
    stats auth admin:123456
    stats hide-version
    stats admin if TRUE

listen kube-master
    bind 0.0.0.0:6443
    mode tcp
    option tcplog
    balance roundrobin
    server 192.168.20.210 192.168.20.210:6443 check inter 2000 fall 2 rise 2 weight 1
    server 192.168.20.211 192.168.20.211:6443 check inter 2000 fall 2 rise 2 weight 1
    server 192.168.20.212 192.168.20.212:6443 check inter 2000 fall 2 rise 2 weight 1
EOF
```

## keepalived配置
```bash
# HA1配置
$ cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
    router_id lb-master-105
}

vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 3
}

vrrp_instance VI-kube-master {
    state BACKUP
    nopreempt
    priority 120
    dont_track_primary
    interface ens192
    virtual_router_id 68
    advert_int 3
    track_script {
        check-haproxy
    }
    virtual_ipaddress {
        192.168.20.222
    }
}
EOF

# HA2配置
$ cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
    router_id lb-master-105
}

vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 3
}

vrrp_instance VI-kube-master {
    state BACKUP
    priority 100
    dont_track_primary
    interface ens192
    virtual_router_id 68
    advert_int 3
    track_script {
        check-haproxy
    }
    virtual_ipaddress {
        192.168.20.222
    }
}
EOF
```
## 启动HA服务
```bash
$ systemctl enable haproxy
$ systemctl start haproxy

$ systemctl enable keepalived
$ systemctl start keepalived
```

## 查看VIP地址绑定
```bash
# 在HA1和HA2上查看VIP地址，这里VIP绑定在HA1上，如果HA1宕机会漂移到HA2上，结果我们最后集群搭建完成验证。
$ ip addr show | grep 192.168.20.222
    inet 192.168.20.222/32 scope global ens192
```

# 搭建ETCD集群
`ETCD`用于存放kubernetes集群所有配置信息，例如：各个组件的维护着哪些主机、POD、Service以及用户执行kubectl下达的信息都会由apiserver存放在ETCD中，因此搭建kubernetes集群etcd是必不可少的。而etcd在kubernetes中有两种部署方式：
* 使用堆叠部署方式部署在master节点上。这种方法需要较少的基础设备，etcd成员和master节点位于同一位置。
* 使用外部etcd集群。这种方法需要更多的基础设备，master节点和etcd成员是分开的。

我们这里使用外部etcd集群配置。

> 目前版本的kubernetes均采用证书进行验证通信，为了统一因此我们这里也对etcd配置证书。

## 下载CFSSL签证工具
```bash
$ curl -o /usr/local/bin/cfssl https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
$ curl -o /usr/local/bin/cfssljson https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
$ chmod +x /usr/local/bin/cfssl*
```

## 创建CA证书和密钥
```bash
# 创建CA配置文件
$ cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```
```bash
# 创建CA证书签名请求
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "4Paradigm"
    }
  ]
}
EOF
```
```bash
# 创建etcd证书签名请求文件
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.20.216",
    "192.168.20.217",
    "192.168.20.218"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "BeiJing",
      "L": "BeiJing",
      "O": "k8s",
      "OU": "4Paradigm"
    }
  ]
}
EOF
```
## 生成 CA 证书和私钥
```bash
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
$ cfssl gencert -ca=ca.pem \
    -ca-key=ca-key.pem \
    -config=ca-config.json \
    -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```
## 证书分发
```bash
# 这里etcd1的没生成密钥并copy到各个节点，所以要手动输入密码
$ mkdir -pv /etc/kubernetes/pki/etcd/   # etcd、master节点手动创建该目录
$ cp *.pem /etc/kubernetes/pki/etcd/
$ scp *.pem etcd2:/etc/kubernetes/pki/etcd/
$ scp *.pem etcd3:/etc/kubernetes/pki/etcd/
# kubernetes master 也需要etcd的证书
$ scp /etc/kubernetes/pki/etcd/* master1:/etc/kubernetes/pki/etcd/
$ scp /etc/kubernetes/pki/etcd/* master2:/etc/kubernetes/pki/etcd/
$ scp /etc/kubernetes/pki/etcd/* master3:/etc/kubernetes/pki/etcd/
```

## 下载etcd二进制tar包
```bash
$ wget https://github.com/etcd-io/etcd/releases/download/v3.3.10/etcd-v3.3.10-linux-amd64.tar.gz
$ tar zxf etcd-v3.3.10-linux-amd64.tar.gz
$ cp  etcd-v3.3.10-linux-amd64/etcd* /usr/local/bin
$ scp /usr/local/bin/etcd* etcd2:/usr/local/bin/
$ scp /usr/local/bin/etcd* etcd3:/usr/local/bin/
```

## 创建工作目录
> 所有etcd节点都要操作

```bash
$ mkdir -p /var/lib/etcd
```

## 创建systemctl unit文件
```bash
# 复制到对应etcd节点请注意修改 --name名称和节点ip，这里以etcd1和192.168.20.216为例
$ cat > /etc/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
  --name etcd1 \
  --cert-file=/etc/kubernetes/pki/etcd/etcd.pem \
  --key-file=/etc/kubernetes/pki/etcd/etcd-key.pem \
  --peer-cert-file=/etc/kubernetes/pki/etcd/etcd.pem \
  --peer-key-file=/etc/kubernetes/pki/etcd/etcd-key.pem \
  --trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
  --peer-trusted-ca-file=/etc/kubernetes/pki/etcd/ca.pem \
  --initial-advertise-peer-urls https://192.168.20.216:2380 \
  --listen-peer-urls https://192.168.20.216:2380 \
  --listen-client-urls https://192.168.20.216:2379,http://127.0.0.1:2379 \
  --advertise-client-urls https://192.168.20.216:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster etcd1=https://192.168.20.216:2380,etcd2=https://192.168.20.217:2380,etcd3=https://192.168.20.218:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```
## 启动etcd服务
```bash
$ systemctl daemon-reload
$ systemctl enable etcd
$ systemctl start etcd
```

## 验证etcd集群
```bash
$ etcdctl --ca-file=/etc/kubernetes/pki/etcd/ca.pem --cert-file=/etc/kubernetes/pki/etcd/etcd.pem --key-file=/etc/kubernetes/pki/etcd/etcd-key.pem cluster-health
member 8d6cbe5fbadf12e8 is healthy: got healthy result from https://192.168.20.216:2379
member 99ca23ca7acda4bf is healthy: got healthy result from https://192.168.20.217:2379
member 9c6200830b8143c8 is healthy: got healthy result from https://192.168.20.218:2379
cluster is healthy

# 也可以查看当前leader
$ etcdctl --ca-file=/etc/kubernetes/pki/etcd/ca.pem --cert-file=/etc/kubernetes/pki/etcd/etcd.pem --key-file=/etc/kubernetes/pki/etcd/etcd-key.pem member list
8d6cbe5fbadf12e8: name=etcd1 peerURLs=https://192.168.20.216:2380 clientURLs=https://192.168.20.216:2379 isLeader=false
99ca23ca7acda4bf: name=etcd2 peerURLs=https://192.168.20.217:2380 clientURLs=https://192.168.20.217:2379 isLeader=true
9c6200830b8143c8: name=etcd3 peerURLs=https://192.168.20.218:2380 clientURLs=https://192.168.20.218:2379 isLeader=false
```
显示cluster is healthy表示etcd集群搭建完成。至此，我们已经为kubernetes部署准备好了环境，下面我们来配置我们的主角kubernetes。

# 安装docker-ce
<div class="note primary"><p>注：只在kubernetes的master和node上安装docker-ce即可。</p></div>

## 配置docker-ce的yum源
```bash
$ wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
```

# 安装docker-ce
```bash
# 这里默认为18.9.7最新版本
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
<div class="note primary"><p>注： 只在master、node节点上进行以下操作。</p></div>

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
$ yum install -y kubelet-1.14.3 kubeadm-1.14.3 kubectl-1.14.3
```
```bash
# 设置开机自启动kubelet
$ systemctl enable kubelet
```
`kubelet`负责与其他节点集群通信，并进行本节点Pod和容器生命周期的管理。`kubeadm`是Kubernetes的自动化部署工具，降低了部署难度，提高效率。`kubectl`是Kubernetes集群客户端管理工具。

# 部署kubernetes master
以下内容是根据`kubeadm config print init-defaults`指令打印出来的，并根据自己需求手动修改后的结果。
```bash
$ cat > /root/kubeadm-init.yaml << EOF
apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
kubernetesVersion: v1.14.3
#imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: 192.168.20.222:6443
apiServer:
  certSANs:
    - master1
    - master2
    - master3
    - 192.168.20.210
    - 192.168.20.211
    - 192.168.20.212
    - 192.168.20.222
    - 127.0.0.1
etcd:
  external:
    endpoints:
    - "https://192.168.20.216:2379"
    - "https://192.168.20.217:2379"
    - "https://192.168.20.218:2379"
    caFile: /etc/kubernetes/pki/etcd/ca.pem
    certFile: /etc/kubernetes/pki/etcd/etcd.pem
    keyFile: /etc/kubernetes/pki/etcd/etcd-key.pem
networking:
  podSubnet: 10.244.0.0/16

---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
EOF
```
其中`apiServer.certSANS`中配置的是所有要和apiserver交互的地址，包括VIP。 `etcd.external.endpoints` 配置的是外部etcd集群，其中也指定了etcd证书路径，这就是为什么etcd的证书要复制到kubernetes所有master节点的原因。

## 获取kubernetes组件镜像
这里手动拉取master所需要的镜像是为了等会`kubeadm init`时能够快些，也可以不手动拉取。在kubeadm init时也会自动拉取。
```bash
$ kubeadm config images pull --config kubeadm-init.yaml 
```
如果你能科学上网，执行`kubeadm config images pull`后将自动拉取kubeadm对应版本的master节点需要的镜像，拉取完需要的镜像如下：
```bash
$ docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.3             004666307c5b        8 days ago          82.1MB
k8s.gcr.io/kube-controller-manager   v1.14.3             ac2ce44462bc        8 days ago          158MB
k8s.gcr.io/kube-apiserver            v1.14.3             9946f563237c        8 days ago          210MB
k8s.gcr.io/kube-scheduler            v1.14.3             953364a3ae7a        8 days ago          81.6MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        5 months ago        40.3MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        18 months ago       742kB
```
当你不能科学上网时，可以手动从别的镜像仓库中下载，这样也能达到我们的要求，如下操作：
```bash
$ vi /root/kubernetes-images.sh
#!/bin/sh
PACKAGE=(kube-apiserver:v1.14.3 kube-controller-manager:v1.14.3 kube-scheduler:v1.14.3 kube-proxy:v1.14.3 pause:3.1)
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

## 初始化集群
```bash
$ kubeadm init --config kubeadm-init.yaml
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

You can now join any number of control-plane nodes by copying certificate authorities 
and service account keys on each node and then running the following as root:

  kubeadm join 192.168.20.222:6443 --token wvufw9.uzdq9vfcwgfq3lve \
    --discovery-token-ca-cert-hash sha256:2cd1440d49aaf1f8b9321f71ff14f977f5fb37e80f865b136cdda372f94fabe7 \
    --experimental-control-plane 	  

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.20.222:6443 --token wvufw9.uzdq9vfcwgfq3lve \
    --discovery-token-ca-cert-hash sha256:2cd1440d49aaf1f8b9321f71ff14f977f5fb37e80f865b136cdda372f94fabe7
```
执行完后结果会告诉我们如何配置kubectl用户权限以及其它master节点和node节点加入集群的命令。后面会用到这两条命令。

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
```
`kube-flannel.yml`文件下载后修改以下内容：
```
  net-conf.json: |
    {
      "Network": "10.244.0.0/16",
      "Backend": {
        "Type": "vxlan",
        "Directrouting": true
      }
    }
```
`"Directrouting": true`的作用使同网段的pod通信直接走本node网卡，以静态路由表方式配置。而`vxlan`将虚拟网络的数据帧添加到VxLAN首部，封装在物理网络的UDP报文中，进行隧道传输到达目标主机后去掉物理网络报文的头部信息以及VxLAN首部，并交付给目的终端。该配置会优先使用host-gw，不在同一网段的主机在选择vxlan模式。
```bash
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

# master2和master3加入集群

## kubernetes证书分发
复制master1生成的证书文件到master2和master3，因为kubernetes使用证书配置集群，所以其它节点要使用同一套证书。
```bash
# 在master1上操作
$ cd /etc/kubernetes/pki/
$ scp *.crt *.key sa.pub master2:/etc/kubernetes/pki/
$ scp *.crt *.key sa.pub master3:/etc/kubernetes/pki/
#这里还有etcd证书，因为在配置etcd的时候我们已经复制过去了，所以这里不用在复制了。
```
## 手动载入组件镜像
为了节约安装时间，我们这里手动将master1上的组件镜像载入到master2和master3上。如果不手动载入也可以，时间比较久而已。当然如果是k8s.gcr.io的仓库地址的话记得要科学上网哦。
```bash
$ docker save k8s.gcr.io/kube-proxy:v1.14.3 k8s.gcr.io/kube-controller-manager:v1.14.3 k8s.gcr.io/kube-apiserver:v1.14.3 k8s.gcr.io/kube-scheduler:v1.14.3 k8s.gcr.io/coredns:1.3.1 k8s.gcr.io/pause:3.1 quay.io/coreos/flannel:v0.11.0-amd64 -o k8s-masterimages.tar
$ scp k8s-masterimages.tar master2:/root
$ scp k8s-masterimages.tar master3:/root
```
```bash
# 在master2和master3上手动加载镜像
$ docker load -i k8s-masterimages.tar
$ docker images
REPOSITORY                           TAG                 IMAGE ID            CREATED             SIZE
k8s.gcr.io/kube-proxy                v1.14.3             004666307c5b        3 weeks ago         82.1MB
k8s.gcr.io/kube-controller-manager   v1.14.3             ac2ce44462bc        3 weeks ago         158MB
k8s.gcr.io/kube-apiserver            v1.14.3             9946f563237c        3 weeks ago         210MB
k8s.gcr.io/kube-scheduler            v1.14.3             953364a3ae7a        3 weeks ago         81.6MB
quay.io/coreos/flannel               v0.11.0-amd64       ff281650a721        5 months ago        52.6MB
k8s.gcr.io/coredns                   1.3.1               eb516548c180        5 months ago        40.3MB
k8s.gcr.io/pause                     3.1                 da86e6ba6ca1        18 months ago       742kB
```
## 加入集群
使用kubeadm安装的集群要让一个新节点加入集群那是相当简单了，如下：
```bash
# 在master2 和 master3执行
$ kubeadm join 192.168.20.222:6443 --token wvufw9.uzdq9vfcwgfq3lve \
    --discovery-token-ca-cert-hash sha256:2cd1440d49aaf1f8b9321f71ff14f977f5fb37e80f865b136cdda372f94fabe7 \
    --experimental-control-plane 
[preflight] Running pre-flight checks
	[WARNING IsDockerSystemdCheck]: detected "cgroupfs" as the Docker cgroup driver. The recommended driver is "systemd". Please follow the guide at https://kubernetes.io/docs/setup/cri/
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[preflight] Running pre-flight checks before initializing the new control plane instance
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Using the existing "front-proxy-client" certificate and key
[certs] Using the existing "apiserver" certificate and key
[certs] Using the existing "apiserver-kubelet-client" certificate and key
[certs] Valid certificates and keys now exist in "/etc/kubernetes/pki"
[certs] Using the existing "sa" key
[kubeconfig] Generating kubeconfig files
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[check-etcd] Skipping etcd check in external mode
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...
[control-plane-join] using external etcd - no local stacked instance added
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[mark-control-plane] Marking the node master2 as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node master2 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.


To start administering your cluster from this node, you need to run the following as a regular user:

	mkdir -p $HOME/.kube
	sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
	sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster. 
```
> 至此master2和master3以master身份加入集群成功，而`--experimental-control-plane`参数是主要参数。

## 配置执行kubectl命令用户
```bash
# kubernetes建议使用非root用户运行kubectl命令访问集群
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## 查看集群节点
```bash
# 三台master节点都可以执行kubectl命令，因为都复制了集群的/etc/kubernetes/admin.conf认证文件到用户/.kube/config中。
$ kubectl get nodes
NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   4h2m    v1.14.3
master2   Ready    master   3h48m   v1.14.3
master3   Ready    master   3h47m   v1.14.3
```
现在我们三台master都搭建好了，现在我们为集群部署node。

# 部署kubernetes node节点
## 手动载入组件镜像
同样为了节约时间，我们将master1上的部分node需要的组件镜像打包，然后发送到node节点进行载入。
```bash
$ docker save k8s.gcr.io/kube-proxy:v1.14.3 k8s.gcr.io/pause:3.1 quay.io/coreos/flannel:v0.11.0-amd64 -o k8s-nodeimages.tar
$ scp k8s-nodeimages.tar node1:/root
$ scp k8s-nodeimages.tar node2:/root
$ scp k8s-nodeimages.tar node3:/root
```
## 载入node组件镜像
> 以下操作3台node都要进行

```bash
$ docker load -i k8s-nodeimages.tar
```

## node加入集群
```bash
$ kubeadm join 192.168.20.222:6443 --token wvufw9.uzdq9vfcwgfq3lve --discovery-token-ca-cert-hash sha256:2cd1440d49aaf1f8b9321f71ff14f977f5fb37e80f865b136cdda372f94fabe7
```
node节点加入集群就这么简单。集群部署完后，查看集群当前节点，3master和3node都已经为Ready了，集群可用。

## 查看节点数及状态
```bash
$ kubectl get node
NAME      STATUS   ROLES    AGE     VERSION
master1   Ready    master   4h14m   v1.14.3
master2   Ready    master   4h      v1.14.3
master3   Ready    master   4h      v1.14.3
node1     Ready    <none>   3h30m   v1.14.3
node2     Ready    <none>   3h31m   v1.14.3
node3     Ready    <none>   3h31m   v1.14.3
```

## 查看集群组件状态
```bash
$ kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"} 
```

# 验证集群
通过上面的步骤，我们已经完成了集群的配置安装，现在我们要验证高可用集群的特性，看看是否能高可用，要验证的有：
* 停掉当前已选举的master来验证组件是否会重新选举。
* 停掉某个etcd来验证etcd的集群是否可用。
* 停掉vip地址所在的主机服务，验证vip是否会偏移到另外一台HA上并且集群可用。

## 验证master选举高可用
要验证masrer集群是否高可用，我们先来查看当前各组件选举在哪台master节点上。
```bash
$ kubectl get endpoints kube-controller-manager -n kube-system -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master1_05043914-9bb3-11e9-bb13-000c29b438b3","leaseDurationSeconds":15,"acquireTime":"2019-07-01T03:48:05Z","renewTime":"2019-07-01T08:13:12Z","leaderTransitions":0}'
  creationTimestamp: "2019-07-01T03:48:05Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "31206"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 0859c6e9-9bb3-11e9-a8cd-000c29b438b3

$ kubectl get endpoints kube-scheduler -n kube-system -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master1_04bc7a76-9bb3-11e9-9dd4-000c29b438b3","leaseDurationSeconds":15,"acquireTime":"2019-07-01T03:48:04Z","renewTime":"2019-07-01T08:13:31Z","leaderTransitions":0}'
  creationTimestamp: "2019-07-01T03:48:04Z"
  name: kube-scheduler
  namespace: kube-system
  resourceVersion: "31243"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-scheduler
  uid: 0805a77e-9bb3-11e9-a8cd-000c29b438b3
```
通过`holderIdentity":"master1`我们可以看到当前2组件都运行在master1上。所以我们现在把master1关机或者停掉kubernetes服务来验证。
> 为什么没kube-apiserver？ 因为kube-apiserver是阻塞的。只能存在一个在运行，而其他两个节点可以运行，但是没被调用而已。

```bash
# 在master1上杀掉kube所有进程,或者直接关机
$ pkill kube
$ ps -ef | grep kube
root      37678   1531  0 16:17 pts/0    00:00:00 grep --color=auto kube
```
当前显示kube相关的进程已经没有了，那么我们在执行命令查看当前组件选举的哪个节点
```bash
$ kubectl get endpoints kube-scheduler -n kube-system -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master3_11cc6547-9bb5-11e9-a0ed-000c29870b1d","leaseDurationSeconds":15,"acquireTime":"2019-07-01T08:17:17Z","renewTime":"2019-07-01T08:18:23Z","leaderTransitions":1}'
  creationTimestamp: "2019-07-01T03:48:04Z"
  name: kube-scheduler
  namespace: kube-system
  resourceVersion: "31875"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-scheduler
  uid: 0805a77e-9bb3-11e9-a8cd-000c29b438b3

$ kubectl get endpoints kube-controller-manager -n kube-system -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master3_120c93e2-9bb5-11e9-9fca-000c29870b1d","leaseDurationSeconds":15,"acquireTime":"2019-07-01T08:17:20Z","renewTime":"2019-07-01T08:18:32Z","leaderTransitions":1}'
  creationTimestamp: "2019-07-01T03:48:05Z"
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "31893"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 0859c6e9-9bb3-11e9-a8cd-000c29b438b3
```
通过`holderIdentity":"master3`我们可以看到当前2组件通过选举已经到master3上了。因此kubernetes master高可用验证完成。

## 验证etcd集群高可用
我们已经知道，kubernetes所有操作的配置以及组件维护的状态都要存储在etcd中，当etcd不能用时，整个kubernetes集群也不能正常工作了。那么我们关掉etcd1节点来测试。
查看当前组件监控状态：
```bash
kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-1               Healthy   {"health":"true"}  # 当前该etcd为健康状态
```
在etcd1节点上以下操作：
```bash
$ systemctl stop etcd
$ ps -ef | grep etcd
root      12919   2305  0 16:26 pts/0    00:00:00 grep --color=auto etcd
```
显示etcd1的etcd服务已经都被关掉了，然后我们在其他etcd节点上执行以下命令查看etcd集群状态：
```bash
$ etcdctl --ca-file=/etc/kubernetes/pki/etcd/ca.pem --cert-file=/etc/kubernetes/pki/etcd/etcd.pem --key-file=/etc/kubernetes/pki/etcd/etcd-key.pem cluster-health
failed to check the health of member 8d6cbe5fbadf12e8 on https://192.168.20.216:2379: Get https://192.168.20.216:2379/health: dial tcp 192.168.20.216:2379: connect: connection refused
member 8d6cbe5fbadf12e8 is unreachable: [https://192.168.20.216:2379] are all unreachable
member 99ca23ca7acda4bf is healthy: got healthy result from https://192.168.20.217:2379
member 9c6200830b8143c8 is healthy: got healthy result from https://192.168.20.218:2379
cluster is degraded
```
已经很明显的看到了etcd1这台服务已经不可达了，而且整个集群也变成degraded，被降级了。
回到kubernetes master1节点上，我们再次查看组件健康状态
```bash
kubectl get cs
NAME                 STATUS      MESSAGE                                                                                             ERROR
etcd-1               Unhealthy   Get https://192.168.20.216:2379/health: dial tcp 192.168.20.216:2379: connect: connection refused   
scheduler            Healthy     ok                                                                                                  
controller-manager   Healthy     ok                                                                                                  
etcd-2               Healthy     {"health":"true"}                                                                                   
etcd-0               Healthy     {"health":"true"}
```
我们也可以通过kubernetes查看到etcd1已经连接失败了。那么我们kubernetes会受影响吗？，我们操作一下即可！
```bash
$ kubectl create namespace prod
namespace/prod created
[root@master1 ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   4h43m
kube-node-lease   Active   4h43m
kube-public       Active   4h43m
kube-system       Active   4h43m
prod              Active   6s
```
可以看到etcd1节点服务宕机后，我们的kubernetes依然可以用，因此etcd集群也是高可用的。

## 验证vip地址漂移
我们都知道我们的集群apiserver是通过haproxy的vip地址做反向代理的。可以通过以下命令查看
```bash
$ kubectl cluster-info
Kubernetes master is running at https://192.168.20.222:6443
KubeDNS is running at https://192.168.20.222:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
当前显示的master是`https://192.168.20.222:6443`,也就是我们的VIP地址代理。那么我们手动停掉haproxy主机，结果会怎么样呢？我们还是来验证一下。
通过开始的HA配置，我们的VIP地址在HA1主机上，现在我们关掉该服务器。
```bash
[root@HA1 ~]# shutdown -h now
PolicyKit daemon disconnected from the bus.
We are no longer a registered authentication agent.
Connection closing...Socket close.

Connection closed by foreign host.

Disconnected from remote host(HA1-192.168.20.219) at 16:36:36.

Type `help' to learn how to use Xshell prompt.
[C:\~]$ 
```
那么我们现在到HA2主机上查看VIP是否漂移过来了。
```bash
[root@HA2 ~]# ip addr show  | grep 192.168.20.222
    inet 192.168.20.222/32 scope global ens192
```
可以看到VIP地址已经成功漂移到HA2节点上了，那么我们的kubernetes服务正常吗？我们依然来验证一下：
```bash
$ kubectl get pod -n kube-system
NAME                              READY   STATUS    RESTARTS   AGE
coredns-fb8b8dccf-95hs5           1/1     Running   2          4h52m
coredns-fb8b8dccf-ct7qr           1/1     Running   2          4h52m
kube-apiserver-master1            1/1     Running   1          4h52m
kube-apiserver-master2            1/1     Running   0          4h37m
kube-apiserver-master3            1/1     Running   0          4h37m
kube-controller-manager-master1   1/1     Running   1          4h51m
kube-controller-manager-master2   1/1     Running   0          4h37m
kube-controller-manager-master3   1/1     Running   0          4h37m
kube-flannel-ds-amd64-4s5th       1/1     Running   0          4h38m
kube-flannel-ds-amd64-4xcpp       1/1     Running   0          4h9m
kube-flannel-ds-amd64-jn8sv       1/1     Running   0          4h45m
kube-flannel-ds-amd64-p8qmf       1/1     Running   0          4h9m
kube-flannel-ds-amd64-wltd6       1/1     Running   0          4h38m
kube-flannel-ds-amd64-x5p77       1/1     Running   0          4h9m
kube-proxy-c8lmp                  1/1     Running   0          4h9m
kube-proxy-drgnv                  1/1     Running   0          4h9m
kube-proxy-f8s5k                  1/1     Running   1          4h52m
kube-proxy-n6mmr                  1/1     Running   0          4h9m
kube-proxy-qnlsq                  1/1     Running   0          4h38m
kube-proxy-r4jjd                  1/1     Running   0          4h38m
kube-scheduler-master1            1/1     Running   1          4h51m
kube-scheduler-master2            1/1     Running   0          4h37m
kube-scheduler-master3            1/1     Running   0          4h37m
```
显示我们的kubectl客户端依然可以通过VIP访问kubernetes集群。因此HA高可用验证成功。