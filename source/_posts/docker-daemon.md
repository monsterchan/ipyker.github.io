layout: post
title: Docker守护进程配置文件daemon.json
author: Pyker
categories: docker
tags:
  - docker
date: 2018-03-22 12:02:00
---

# daemon.json文件说明

docker安装后默认没有`/etc/docker/daemon.json` 文件，需要进行手动创建。`--config-file`命令参数可用于指定非默认位置。但请注意：配置文件中设置的选项不得与启动参数设置的选项冲突。 如果文件和启动参数之间的选项重复，则docker守护程序无法启动，无论其值如何。 例如，如果在daemon.json配置文件中设置了守护程序标签并且还通过--label命令参数设置守护程序标签，则守护程序无法启动。 守护程序启动时将忽略文件中不存在的选项。

这是Linux上允许的配置选项的完整示例：
```json
{
    "authorization-plugins": [],//访问授权插件
    "data-root": "",//docker数据持久化存储的根目录,默认为/var/lib/docker
    "dns": [],//DNS服务器
    "dns-opts": [],//DNS配置选项，如端口等
    "dns-search": [],//DNS搜索域名
    "exec-opts": [],//执行选项
    "exec-root": "",//执行状态的文件的根目录
    "experimental": false,//是否开启试验性特性
    "features": {},//启用或禁用特定功能。如：{"buildkit": true}使buildkit成为默认的docker镜像构建器。
    "storage-driver": "",//存储驱动器类型
    "storage-opts": [],//存储选项
    "labels": [],//键值对式标记docker元数据
    "live-restore": true,//dockerd挂掉是否保活容器（避免了docker服务异常而造成容器退出）
    "log-driver": "json-file",//容器日志的驱动器
    "log-opts": {
        "max-size": "10m",
        "max-file":"5",
        "labels": "somelabel",
        "env": "os,customer"
    },//容器日志的选项
    "mtu": 0,//设置容器网络MTU（最大传输单元）
    "pidfile": "",//daemon PID文件的位置
    "cluster-store": "",//集群存储系统的URL
    "cluster-store-opts": {},//配置集群存储
    "cluster-advertise": "",//对外的地址名称
    "max-concurrent-downloads": 3,//设置每个pull进程的最大并发
    "max-concurrent-uploads": 5,//设置每个push进程的最大并发
    "default-shm-size": "64M",//设置默认共享内存的大小
    "shutdown-timeout": 15,//设置关闭的超时时限
    "debug": true,//开启调试模式
    "hosts": [],//dockerd守护进程的监听地址
    "log-level": "",//日志级别
    "tls": true,//开启传输层安全协议TLS
    "tlsverify": true,//开启输层安全协议并验证远程地址
    "tlscacert": "",//CA签名文件路径
    "tlscert": "",//TLS证书文件路径
    "tlskey": "",//TLS密钥文件路径
    "swarm-default-advertise-addr": "",//swarm对外地址
    "api-cors-header": "",//设置CORS（跨域资源共享-Cross-origin resource sharing）头
    "selinux-enabled": false,//开启selinux(用户、进程、应用、文件的强制访问控制)
    "userns-remap": "",//给用户命名空间设置 用户/组
    "group": "",//docker所在组
    "cgroup-parent": "",//设置所有容器的cgroup的父类
    "default-ulimits": {
        "nofile": {
            "Name": "nofile",
            "Hard": 64000,
            "Soft": 64000
        }
    },//设置所有容器的ulimit
    "init": false,//容器执行初始化，来转发信号或控制(reap)进程
    "init-path": "/usr/libexec/docker-init",//docker-init文件的路径
    "ipv6": false,//支持IPV6网络
    "iptables": false,//开启防火墙规则
    "ip-forward": false,//开启net.ipv4.ip_forward
    "ip-masq": false,//开启ip掩蔽(IP封包通过路由器或防火墙时重写源IP地址或目的IP地址的技术)
    "userland-proxy": false,//用户空间代理
    "userland-proxy-path": "/usr/libexec/docker-proxy",//用户空间代理路径
    "ip": "0.0.0.0",//默认IP
    "bridge": "",//将容器依附(attach)到桥接网络上的桥标识
    "bip": "",//指定桥接IP
    "fixed-cidr": "",//(ipv4)子网划分，即限制ip地址分配范围，用以控制容器所属网段实现容器间(同一主机或不同主机间)的网络访问
    "fixed-cidr-v6": "",//（ipv6）子网划分
    "default-gateway": "",//默认网关
    "default-gateway-v6": "",//默认ipv6网关
    "icc": false,//容器间通信
    "raw-logs": false,//原始日志(无颜色、全时间戳)
    "allow-nondistributable-artifacts": [],//不对外分发的产品提交的registry仓库
    "registry-mirrors": [],//registry仓库镜像加速地址
    "seccomp-profile": "",//seccomp配置文件
    "insecure-registries": [],//配置非https的registry地址
    "no-new-privileges": false,//禁止新优先级
    "default-runtime": "runc",//OCI联盟(The Open Container Initiative)默认运行时环境
    "oom-score-adjust": -500,//内存溢出被杀死的优先级(-1000~1000)
    "node-generic-resources": ["NVIDIA-GPU=UUID1", "NVIDIA-GPU=UUID2"],//对外公布的资源节点
    "runtimes": {
        "cc-runtime": {
            "path": "/usr/bin/cc-runtime"
        },
        "custom": {
            "path": "/usr/local/bin/my-runc-replacement",
            "runtimeArgs": [
                "--debug"
            ]
        }
    },//运行时
    "default-address-pools":[{"base":"172.80.0.0/16","size":24},//默认的dhcp分配地址
    {"base":"172.90.0.0/16","size":24}]
}
```
>您无法在`daemon.json`文件中使用已经在dockerd守护程序启动时设置的选项命令参数。 例如：在使用systemd启动Docker守护程序的系统上，-H已设置，因此您无法使用daemon.json中的hosts键添加侦听地址。 有关如何使用systemd drop-in文件完成此任务，请参阅https://docs.docker.com/engine/admin/systemd/#custom-docker-daemon-options。

# dockerd守护进程选项
```bash
dockerd --help

Usage:  dockerd [OPTIONS]

A self-sufficient runtime for containers.

Options:
      --add-runtime runtime                     Register an additional OCI compatible runtime (default [])
      --allow-nondistributable-artifacts list   Allow push of nondistributable artifacts to registry
      --api-cors-header string                  Set CORS headers in the Engine API
      --authorization-plugin list               Authorization plugins to load
      --bip string                              Specify network bridge IP
  -b, --bridge string                           Attach containers to a network bridge
      --cgroup-parent string                    Set parent cgroup for all containers
      --cluster-advertise string                Address or interface name to advertise
      --cluster-store string                    URL of the distributed storage backend
      --cluster-store-opt map                   Set cluster store options (default map[])
      --config-file string                      Daemon configuration file (default "/etc/docker/daemon.json")
      --containerd string                       containerd grpc address
      --cpu-rt-period int                       Limit the CPU real-time period in microseconds
      --cpu-rt-runtime int                      Limit the CPU real-time runtime in microseconds
      --data-root string                        Root directory of persistent Docker state (default "/var/lib/docker")
  -D, --debug                                   Enable debug mode
      --default-address-pool pool-options       Default address pools for node specific local networks
      --default-gateway ip                      Container default gateway IPv4 address
      --default-gateway-v6 ip                   Container default gateway IPv6 address
      --default-ipc-mode string                 Default mode for containers ipc ("shareable" | "private") (default "shareable")
      --default-runtime string                  Default OCI runtime for containers (default "runc")
      --default-shm-size bytes                  Default shm size for containers (default 64MiB)
      --default-ulimit ulimit                   Default ulimits for containers (default [])
      --dns list                                DNS server to use
      --dns-opt list                            DNS options to use
      --dns-search list                         DNS search domains to use
      --exec-opt list                           Runtime execution options
      --exec-root string                        Root directory for execution state files (default "/var/run/docker")
      --experimental                            Enable experimental features
      --fixed-cidr string                       IPv4 subnet for fixed IPs
      --fixed-cidr-v6 string                    IPv6 subnet for fixed IPs
  -G, --group string                            Group for the unix socket (default "docker")
      --help                                    Print usage
  -H, --host list                               Daemon socket(s) to connect to
      --icc                                     Enable inter-container communication (default true)
      --init                                    Run an init in the container to forward signals and reap processes
      --init-path string                        Path to the docker-init binary
      --insecure-registry list                  Enable insecure registry communication
      --ip ip                                   Default IP when binding container ports (default 0.0.0.0)
      --ip-forward                              Enable net.ipv4.ip_forward (default true)
      --ip-masq                                 Enable IP masquerading (default true)
      --iptables                                Enable addition of iptables rules (default true)
      --ipv6                                    Enable IPv6 networking
      --label list                              Set key=value labels to the daemon
      --live-restore                            Enable live restore of docker when containers are still running
      --log-driver string                       Default driver for container logs (default "json-file")
  -l, --log-level string                        Set the logging level ("debug"|"info"|"warn"|"error"|"fatal") (default "info")
      --log-opt map                             Default log driver options for containers (default map[])
      --max-concurrent-downloads int            Set the max concurrent downloads for each pull (default 3)
      --max-concurrent-uploads int              Set the max concurrent uploads for each push (default 5)
      --metrics-addr string                     Set default address and port to serve the metrics api on
      --mtu int                                 Set the containers network MTU
      --network-control-plane-mtu int           Network Control plane MTU (default 1500)
      --no-new-privileges                       Set no-new-privileges by default for new containers
      --node-generic-resource list              Advertise user-defined resource
      --oom-score-adjust int                    Set the oom_score_adj for the daemon (default -500)
  -p, --pidfile string                          Path to use for daemon PID file (default "/var/run/docker.pid")
      --raw-logs                                Full timestamps without ANSI coloring
      --registry-mirror list                    Preferred Docker registry mirror
      --seccomp-profile string                  Path to seccomp profile
      --selinux-enabled                         Enable selinux support
      --shutdown-timeout int                    Set the default shutdown timeout (default 15)
  -s, --storage-driver string                   Storage driver to use
      --storage-opt list                        Storage driver options
      --swarm-default-advertise-addr string     Set default address or interface for swarm advertised address
      --tls                                     Use TLS; implied by --tlsverify
      --tlscacert string                        Trust certs signed only by this CA (default "/root/.docker/ca.pem")
      --tlscert string                          Path to TLS certificate file (default "/root/.docker/cert.pem")
      --tlskey string                           Path to TLS key file (default "/root/.docker/key.pem")
      --tlsverify                               Use TLS and verify the remote
      --userland-proxy                          Use userland proxy for loopback traffic (default true)
      --userland-proxy-path string              Path to the userland proxy binary
      --userns-remap string                     User/Group setting for user namespaces
  -v, --version                                 Print version information and quit
```