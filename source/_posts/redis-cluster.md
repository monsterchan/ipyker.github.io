layout: post
title: Kubernetes安装redis集群
author: Pyker
categories: kubernetes
tags:
  - kubernetes
date: 2019-07-17 10:20:00
---

# 部署方式
* Statefulset
* Service & depolyment

对于redis,mysql这种有状态的服务,我们使用statefulset方式为首选.我们这边主要就是介绍statefulset这种方式。
> statefulset 的设计原理模型:
　　拓扑状态：应用的多个实例之间不是完全对等的关系,这个应用实例的启动必须按照某些顺序启动,比如应用的
　　　　主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个Pod删除掉,他们再次被创建出来是也必须严格按
　　　　照这个顺序才行,并且,新创建出来的Pod,必须和原来的Pod的网络标识一样,这样原先的访问者才能使用同样
　　　　的方法,访问到这个新的Pod
　　存储状态：应用的多个实例分别绑定了不同的存储数据.对于这些应用实例来说,Pod A第一次读取到的数据,和
　　　　隔了十分钟之后再次读取到的数据,应该是同一份,哪怕在此期间Pod A被重新创建过.一个数据库应用的多个
　　　　存储实例。

# 存储卷
了解statefulset状态后，你应该知道要为数据准备一个存储卷了，创建方式有`静态方式`和`动态方式`，静态方式就是手动创建PV、PVC，然后POD进行进行调用即可。我们这里使用动态nfs作为挂载卷，可以参考上文：[部署NFS动态StorageClass](https://www.ipyker.com/2019/07/15/k8s-nfs.html)

# 准备名称空间
```bash
$ kubectl create ns sts-app
```
> 我这里使用了sts-app名称空间来部署redis，所以必须先创建该名称空间，你也可以不创建使用默认的default名称空间。

# 准备redis配置文件
redis配置文件使用configmap方式进行挂载，如果将配置封装到docker image中的话，俺么每次修改配置就需要重新docker build。个人觉得比较麻烦，所以使用`configmap`方式挂载配置。
```bash
$ cat redis-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-conf-cluster
  namespace: sts-app
data:
  fix-ip.sh: |
    #!/bin/sh
    CLUSTER_CONFIG="/data/nodes.conf"
    if [ -f ${CLUSTER_CONFIG} ]; then
      if [ -z "${POD_IP}" ]; then
        echo "Unable to determine Pod IP address!"
        exit 1
      fi
      echo "Updating my IP to ${POD_IP} in ${CLUSTER_CONFIG}"
      sed -i.bak -e '/myself/ s/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/'${POD_IP}'/' ${CLUSTER_CONFIG}
    fi
    exec "$@"
  redis.conf: |
    cluster-enabled yes
    cluster-config-file /data/nodes.conf
    cluster-node-timeout 10000
    protected-mode no
    daemonize no
    pidfile /var/run/redis.pid
    port 6379
    tcp-backlog 511
    bind 0.0.0.0
    timeout 3600
    tcp-keepalive 1
    loglevel verbose
    logfile /data/redis.log
    databases 16
    save 900 1
    save 300 10
    save 60 10000
    stop-writes-on-bgsave-error yes
    rdbcompression yes
    rdbchecksum yes
    dbfilename dump.rdb
    dir /data
    #requirepass yl123456
    appendonly yes
    appendfilename "appendonly.aof"
    appendfsync everysec
    no-appendfsync-on-rewrite no
    auto-aof-rewrite-percentage 100
    auto-aof-rewrite-min-size 64mb
    lua-time-limit 20000
    slowlog-log-slower-than 10000
    slowlog-max-len 128
    #rename-command FLUSHALL  ""
    latency-monitor-threshold 0
    notify-keyspace-events ""
    hash-max-ziplist-entries 512
    hash-max-ziplist-value 64
    list-max-ziplist-entries 512
    list-max-ziplist-value 64
    set-max-intset-entries 512
    zset-max-ziplist-entries 128
    zset-max-ziplist-value 64
    hll-sparse-max-bytes 3000
    activerehashing yes
    client-output-buffer-limit normal 0 0 0
    client-output-buffer-limit slave 256mb 64mb 60
    client-output-buffer-limit pubsub 32mb 8mb 60
    hz 10
    aof-rewrite-incremental-fsync yes
```
> `fix-ip.sh` 脚本的作用用于当redis集群某pod重建后Pod IP发生变化，在/data/nodes.conf中将新的Pod IP替换原Pod IP。不然集群会出问题。

#  准备redis statsfulset文件
```bash
$ cat redis-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-headless
  namespace: sts-app
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    port: 6379
  clusterIP: None
  selector:
    app: redis
---
apiVersion: v1
kind: Service
metadata:
  name: redis-client
  namespace: sts-app
  labels:
    app: redis
spec:
  ports:
  - name: redis-port
    port: 6379
    targetPort: 6379
  type: NodePort
  selector:
    app: redis
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: redis
  namespace: sts-app
spec:
  serviceName: "redis-headless"
  replicas: 6
  template:
    metadata:
      labels:
        app: redis
    spec:
      terminationGracePeriodSeconds: 20
#      affinity:
#        podAntiAffinity:
#          preferredDuringSchedulingIgnoredDuringExecution:
#          - weight: 100
#            podAffinityTerm:
#              labelSelector:
#                matchExpressions:
#                - key: app
#                  operator: In
#                  values:
#                  - redis
#              topologyKey: kubernetes.io/hostname
      containers:
      - name: redis
        image: registry.cn-shenzhen.aliyuncs.com/pyker/redis:4.0.11
        command: ["/etc/redis/fix-ip.sh", "redis-server", "/etc/redis/redis.conf"]
        readinessProbe:
#          exec:
#            command: ["/bin/sh", "-c", "redis-cli -h $(hostname) ping"]
          tcpSocket:
            port: 6379
          timeoutSeconds: 5
          periodSeconds: 3
          initialDelaySeconds: 2
        livenessProbe:
          tcpSocket:
            port: 6379
          timeoutSeconds: 5
          periodSeconds: 3
          initialDelaySeconds: 2
        env:
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        resources:
          requests:
            memory: "1Gi"
        ports:
            - name: redis
              containerPort: 6379
              protocol: TCP
            - name: cluster
              containerPort: 16379
              protocol: TCP
        volumeMounts:
          - name: redis-conf
            mountPath: /etc/redis/
            readOnly: false
          - name: redis-data
            mountPath: /data
            readOnly: false
      volumes:
      - name: redis-conf
        configMap:
          name: redis-conf-cluster
          items:
            - key: redis.conf
              path: redis.conf
            - key: fix-ip.sh
              path: fix-ip.sh
          defaultMode: 0755
  volumeClaimTemplates:
  - metadata:
      name: redis-data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: nfs
      resources:
        requests:
          storage: 2Gi
```

# 应用配置
```bash
$ kubectl apply -f redis-configmap.yaml
$ kubectl apply -f redis-statefulset.yaml
```
稍等片刻查看redis是否已经运行。
```bash
kubectl  get pod -n sts-app
NAME      READY   STATUS    RESTARTS   AGE
redis-0   1/1     Running   0          1m
redis-1   1/1     Running   0          1m
redis-2   1/1     Running   0          1m
redis-3   1/1     Running   0          1m
redis-4   1/1     Running   0          1m
redis-5   1/1     Running   0          1m
```

# 初始化redis集群
如上redis 实例已经全部处于Running状态了。因此我们可以来初始化集群了，如下步骤：
```bash
# 随便选一台redis pod实例进去初始化集群。这里选redis-0
$ $ kubectl exec -it redis-0 -n sts-app -- bash
#初始化集群
[root@redis-0]# redis-trib.rb create --replicas 1 \
`dig +short redis-0.redis-headless.sts-app.svc.cluster.local`:6379 \
`dig +short redis-1.redis-headless.sts-app.svc.cluster.local`:6379 \
`dig +short redis-2.redis-headless.sts-app.svc.cluster.local`:6379 \
`dig +short redis-3.redis-headless.sts-app.svc.cluster.local`:6379 \
`dig +short redis-4.redis-headless.sts-app.svc.cluster.local`:6379 \
`dig +short redis-5.redis-headless.sts-app.svc.cluster.local`:6379
```
> `redis-trib.rb`必须使用ip进行初始化redis集群，使用域名会报如下错误！因此这里使用`dig +short`获取pod IP。
```
/var/lib/gems/2.3.0/gems/redis-4.1.2/lib/redis/client.rb:126:in `call’: ERR Invalid node address specified: redis-0.redis-headless.sts-app.svc.cluster.local:6379 (Redis::CommandError)
```

# 附上Dockerfile
```bash
$ cat Dockerfile
FROM redis:4.0.11
RUN apt-get update -y
RUN apt-get install -y  ruby \
rubygems
RUN apt-get clean all
RUN gem install redis
RUN apt-get install dnsutils -y
COPY redis-trib.rb /usr/local/bin/
```
```bash
$ chmod +x redis-trib.rb
```
`redis-trib.rb`工具可以去redis源码中拷贝一个到当前目录,然后构建镜像。