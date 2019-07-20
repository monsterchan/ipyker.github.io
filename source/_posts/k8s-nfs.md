layout: post
title: 使用NFS为kubernetes提供动态StorageClass
author: Pyker
categories: kubernetes
tags:
  - kubernetes
date: 2019-07-15 22:20:00
---

# PV和PVC简介
`PersistentVolume（PV）`是指由集群管理员配置提供的某存储系统上的段存储空间，它是对底层共享存储的抽象，将共享存储作为种可由用户申请使的资源，实现了“存储消费”机制。通过存储插件机制，PV支持使用多种网络存储系统或云端存储等多种后端存储系统。 PV是集群级别的资源，不属于任何名称空间，用户对PV资源的使需要通过`PersistentVolumeClaim（PVC）`提出的使申请（或称为声明）来完成绑定，PVC是PV资源的消费者，它向PV申请特定大小的空间及访问模式（如rw或ro）。 从创建出PVC存储卷请求后再由Pod资源通过PVC持久卷请求来关联PV存储卷使用。

# StorageClass
通俗的讲它就是将存储资源定义为具有显著特性的类（Class）而不是具体的PV，用户通过PVC直接向意向的存储类发出申请，然后由其按需为用户`动态创建PV`（也可以由管理员事先创建的PV），这样做甚至免去了需要先创建PV的过程。 PV对存储系统的支持可通过其插件来实现，目前，Kubernetes支持如下类型的插件。[官方地址](https://kubernetes.io/docs/concepts/storage/storage-classes/)， 而这里我们使用NFS来搭建StorageClass。

# 环境说明

| 系统 | IP | 应用 |
| ---- | -- | --- |
| centos7.5 | 192.168.3.125 | nfs服务器 |
| centos7.5 | 192.168.3.120 | k8s集群master1 |
| centos7.5 | 192.168.3.121 | k8s集群master2 |
| centos7.5 | 192.168.3.122 | k8s集群master3 |

> 我们这里k8s集群没有node节点，master节点的不被调度的污点(Taints) 被删掉了。`kubectl edit node master1`

# 安装NFS
首先我们得有个NFS服务器，不然就不存在使用NFS服务来做存储类这一说法。
* 安装软件包
```bash
$ yum -y install nfs-utils  rpcbind
```
> `nfs-utils`软件包请在所有k8s集群上安装，因为StorageClass需要挂载NFS共享目录。

* 配置共享目录
```bash
# 这里使用/data/nfs作为共享目录
$ cat >> /etc/exports << EOF
/data/nfs  *(rw,no_root_squash)
EOF
```

* 启动NFS服务
```bash
$ systemctl start rpcbind
$ systemctl nfs start
```

* 测试NFS是否可用
```bash
# 在同网络任意一台装了nfs-utils软件包服务器上测试挂载
$ mount -t 192.168.3.125:/data/nfs /mnt
```
如果一切正常，说明NFS服务正常！此时才能进行后面的操作！

# 创建StorageClass RBAC
我这里是使用了nfs的名称空间，也可以使用别的或者删掉，使用默认的名称空间。如果要使用名称空间需要先创建。
```bash
$ kubectl create ns nfs
```
```bash
$ cat rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-provisioner
  namespace: nfs
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get"]
  - apiGroups: ["extensions"]
    resources: ["podsecuritypolicies"]
    resourceNames: ["nfs-provisioner"]
    verbs: ["use"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-provisioner
     # replace with namespace where provisioner is deployed
    namespace: nfs
roleRef:
  kind: ClusterRole
  name: nfs-provisioner-runner
  apiGroup: rbac.authorization.k8s.io
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-provisioner
  namespace: nfs
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: leader-locking-nfs-provisioner
  namespace: nfs
subjects:
  - kind: ServiceAccount
    name: nfs-provisioner
    # replace with namespace where provisioner is deployed
    namespace: nfs
roleRef:
  kind: Role
  name: leader-locking-nfs-provisioner
  apiGroup: rbac.authorization.k8s.io
```

```bash
$ kubectl apply -f rbac.yaml  # 应用rbac.yaml
serviceaccount/nfs-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-provisioner created
```

# 配置deployment文件
```bash
$ cat deployment.yaml 
kind: Deployment
apiVersion: apps/v1
metadata:
  name: nfs-provisioner
  namespace: nfs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs-provisioner
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-provisioner
    spec:
      serviceAccount: nfs-provisioner
      containers:
        - name: nfs-provisioner
          image: registry.cn-shenzhen.aliyuncs.com/pyker/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: example.com/nfs
            - name: NFS_SERVER
              value: 192.168.3.125  # NFS服务器地址
            - name: NFS_PATH 
              value: /data/nfs/sc   # 需要挂载的路径目录
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.3.125   # NFS服务器地址
            path: /data/nfs/sc      # 需要挂载的路径目录
```
```bash
$ kubectl apply -f deployment.yaml # 应用deployment.yaml
deployment.apps/nfs-provisioner created
```

# 创建StorageClass
```bash
$ cat storageclass.yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: nfs
provisioner: example.com/nfs
#mountOptions:
#  - vers=4.1
```
```bash
$ kubectl apply -f storageclass.yaml  # 应用storageclass.yaml
storageclass.storage.k8s.io/nfs created
```
以上创建完成后，可以查看我们刚才新创建的资源信息如下：
```bash
$ kubectl  get deploy  -n nfs
NAME              READY   UP-TO-DATE   AVAILABLE   AGE
nfs-provisioner   1/1     1            1           67m  # READY为1/1正常

$ kubectl  get sc -n nfs
NAME   PROVISIONER       AGE
nfs    example.com/nfs   66m
```
确认服务正常后，我们可以进行来先进行测试一下NFS的动态存储功能是否正常。

# 测试PVC动态创建PV
手动编写一个PVC来测试一下，**注意：storageClassName应确保与上面创建的StorageClass名称一致。**
```bash
$ cat test-claim.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim1
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi
  storageClassName: nfs
```
```bash
$ kubectl apply -f test-claim.yaml 
persistentvolumeclaim/test-claim1 created
```
```bash
$ kubectl get pvc
NAME                     STATUS        VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-claim1              Bound         pvc-d0a0a29b-ab06-11e9-96e4-080027f62143   1Mi        RWX            nfs            9s

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                            STORAGECLASS   REASON   AGE
pvc-d0a0a29b-ab06-11e9-96e4-080027f62143   1Mi        RWX            Delete           Bound         default/test-claim1              nfs                     21s
```
从上面可以看到PVC已经成功的动态的创建了一个PV存储卷并且与之绑定，此时我们到NFS服务器上去看一下，也可以发现已经自动生成了一个存储类的文件夹。
```bash
$ ls /data/nfs/sc/
default-test-claim1-pvc-d0a0a29b-ab06-11e9-96e4-080027f62143
```

# 创建StatefulSet实例演示
```bash
$ cat sts.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-access
  labels:
    app: nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: web
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: nginx-web
spec:
  selector:
    matchLabels:
      app: nginx
  serviceName: "nginx"
  replicas: 3
  template:
    metadata:
      labels:
        app: nginx
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: nginx-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: nginx-data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs"
      resources:
        requests:
          storage: 1Gi
```
```bash
$ kubectl apply -f sts.yaml
service/nginx created
statefulset.apps/nginx-web created
```
查看3个有状态的nginx是否启动, 也可以发现已经启动完成。
```bash
$ kubectl  get pod,sts
NAME          READY   STATUS        RESTARTS   AGE
nginx-web-0   1/1     Running       0          70m
nginx-web-1   1/1     Running       0          70m
nginx-web-2   1/1     Running       0          68m

NAME                         READY   AGE
statefulset.apps/nginx-web   3/3     71m
```
然后我们在查看一下PVC和PV的创建状态
```bash
$ kubectl  get pvc,pv
NAME                                           STATUS        VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nginx-data-nginx-web-0   Bound         pvc-2c8ec903-ab08-11e9-96e4-080027f62143   1Gi        RWO            nfs            71m
persistentvolumeclaim/nginx-data-nginx-web-1   Bound         pvc-34ce5093-ab08-11e9-96e4-080027f62143   1Gi        RWO            nfs            71m
persistentvolumeclaim/nginx-data-nginx-web-2   Bound         pvc-6e4464ea-ab08-11e9-96e4-080027f62143   1Gi        RWO            nfs            70m
persistentvolumeclaim/test-claim1              Bound         pvc-d0a0a29b-ab06-11e9-96e4-080027f62143   1Mi        RWX            nfs            81m

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS        CLAIM                            STORAGECLASS   REASON   AGE
persistentvolume/pvc-2c8ec903-ab08-11e9-96e4-080027f62143   1Gi        RWO            Delete           Bound         default/nginx-data-nginx-web-0   nfs                     71m
persistentvolume/pvc-34ce5093-ab08-11e9-96e4-080027f62143   1Gi        RWO            Delete           Bound         default/nginx-data-nginx-web-1   nfs                     71m
persistentvolume/pvc-6e4464ea-ab08-11e9-96e4-080027f62143   1Gi        RWO            Delete           Bound         default/nginx-data-nginx-web-2   nfs                     70m
persistentvolume/pvc-d0a0a29b-ab06-11e9-96e4-080027f62143   1Mi        RWX            Delete           Bound         default/test-claim1              nfs                     81m
```
由于statefulset是有状态的，所以他会为每个pod独立分配PVC,PV，且该PV只能该POD使用，所以3个副本pod会生成3个PVC和PV。

# 验证数据存储
上面我们已经搭建好了nfs类型的StorageClass了，也成功启动了。现在我们来测试一下数据是否可以存储到NFS。我们进入刚刚创建的nginx-web-0容器进行操作。
```bash
$ kubectl  exec -it nginx-web-0 -- bash
root@nginx-web-0:/# cd /usr/share/nginx/html/
root@nginx-web-0:/# cat >> index.html << EOF
> hello this is my new pages!
EOF
```
然后我们到NFS服务器的/data/nfs/sc目录上查看有关nginx-web-0目录的存储目录内容。 也可以发现我们刚才在容器里新建index.html的操作确实存放在NFS上了。
```bash
$ cat /data/nfs/sc/default-nginx-data-nginx-web-0-pvc-2c8ec903-ab08-11e9-96e4-080027f62143/index.html 
hello this is my new pages!
```