layout: post
title: kubernetes安装zookeeper 3.5.5
author: Pyker
categories: kubernetes
tags:
  - kubernetes
date: 2019-07-20 09:20:00
---

# 准备zookeeper statefulset文件
```bash
$ cat zk-statefulset.yaml
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: zoo
spec:
  serviceName: "zoo"
  replicas: 3
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: zookeeper
          image: registry.cn-shenzhen.aliyuncs.com/pyker/zookeeper:3.5.5
          imagePullPolicy: IfNotPresent
          readinessProbe:
            httpGet:
              path: /commands/ruok
              port: 8080
            initialDelaySeconds: 10
            timeoutSeconds: 5
            periodSeconds: 3
          livenessProbe:
            httpGet:
              path: /commands/ruok
              port: 8080
            initialDelaySeconds: 30
            timeoutSeconds: 5
            periodSeconds: 3
          env:
            - name: ZOO_SERVERS
              value: server.1=zoo-0.zoo:2888:3888;2181 server.2=zoo-1.zoo:2888:3888;2181 server.3=zoo-2.zoo:2888:3888;2181
          ports:
            - containerPort: 2181
              name: client
            - containerPort: 2888
              name: peer
            - containerPort: 3888
              name: leader-election
          volumeMounts:
            - name: datadir
              mountPath: /data
  volumeClaimTemplates:
  - metadata:
      name: datadir
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "nfs"
      resources:
        requests:
          storage: 1Gi
---

apiVersion: v1
kind: Service
metadata:
  name: zookeeper
spec:
  type: NodePort
  ports:
  - port: 2181
    name: client
    targetPort: 2181
  selector:
    app: zookeeper

---

apiVersion: v1
kind: Service
metadata:
  name: zoo
spec:
  ports:
  - port: 2888
    name: peer
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zookeeper
```
```bash
$ kubectl apply -f zk-statefulset.yaml
# 查看pod是否正常运行
$ kubectl get pod | grep zoo
NAME                                      READY   STATUS    RESTARTS   AGE
zoo-0                                     1/1     Running   3          6h
zoo-1                                     1/1     Running   0          3h
zoo-2                                     1/1     Running   0          3h
```

# 为zookeeper准备图形界面
```bash
$ cat zkui.yaml
apiVersion: v1
kind: Service
metadata:
  name: zkui
  labels:
    app: zkui
spec:
  type: NodePort
  ports:
  - port: 9090
    protocol: TCP
    targetPort: 9090
  selector:
    app: zkui
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: zkui
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: zkui
    spec:
      containers:
      - name: zkui
        image: registry.cn-shenzhen.aliyuncs.com/pyker/zkui:latest
        imagePullPolicy: IfNotPresent
        env:
        - name: ZK_SERVER
#          value: "zoo-1.zoo:2181,zoo-2.zoo:2181,zoo-0.zoo:2181"
# 		   此处用选择的是node ip和 nodeport
          value: "192.168.1.236:2181,192.168.1.236:2182,192.168.1.236:2183"
```
```bash
$ kubectl apply -f zkui.yaml
# 查看pod是否正常运行
kubectl get pod | grep zkui
NAME                                      READY   STATUS    RESTARTS   AGE
zkui-6f65d8b8f8-m27rd                     1/1     Running   1          2d23h
# 查看service状态
$ kubectl get svc | grep zkui
zkui            NodePort    10.254.2.58     <none>        9090:30458/TCP      6d1h
```
此时，可以看到zkui Pod的9090映射到宿主机的30458端口了。那么局域网的主机可以通过`http://nodeip:30458`方式访问zkui界面了。 我这里的默认帐号是admin，密码为manager。密码帐号可以在zkui的pod内的`/var/app/config.cfg`文件进行修改。

# 附上脚本和Dockerfile
```bash
$ cat docker-entrypoint.sh
#!/bin/bash

set -e

# Allow the container to be started with `--user`
if [[ "$1" = 'zkServer.sh' && "$(id -u)" = '0' ]]; then
    chown -R zookeeper "$ZOO_DATA_DIR" "$ZOO_DATA_LOG_DIR" "$ZOO_LOG_DIR" "$ZOO_CONF_DIR"
    exec gosu zookeeper "$0" "$@"
fi

# Generate the config only if it doesn't exist
if [[ ! -f "$ZOO_CONF_DIR/zoo.cfg" ]]; then
    CONFIG="$ZOO_CONF_DIR/zoo.cfg"

    echo "dataDir=$ZOO_DATA_DIR" >> "$CONFIG"
    echo "dataLogDir=$ZOO_DATA_LOG_DIR" >> "$CONFIG"
    echo "4lw.commands.whitelist=$ZOO_COMMANDS_WHITELIST" >> "$CONFIG"

    echo "tickTime=$ZOO_TICK_TIME" >> "$CONFIG"
    echo "initLimit=$ZOO_INIT_LIMIT" >> "$CONFIG"
    echo "syncLimit=$ZOO_SYNC_LIMIT" >> "$CONFIG"

    echo "autopurge.snapRetainCount=$ZOO_AUTOPURGE_SNAPRETAINCOUNT" >> "$CONFIG"
    echo "autopurge.purgeInterval=$ZOO_AUTOPURGE_PURGEINTERVAL" >> "$CONFIG"
    echo "maxClientCnxns=$ZOO_MAX_CLIENT_CNXNS" >> "$CONFIG"
    echo "standaloneEnabled=$ZOO_STANDALONE_ENABLED" >> "$CONFIG"
    echo "admin.enableServer=$ZOO_ADMINSERVER_ENABLED" >> "$CONFIG"

    if [[ -z $ZOO_SERVERS ]]; then
      ZOO_SERVERS="server.1=localhost:2888:3888;2181"
    fi

    for server in $ZOO_SERVERS; do
        echo "$server" >> "$CONFIG"
    done
fi

ZOO_MY_ID=$(($(hostname | sed s/.*-//) + 1))
cp $ZOO_CONF_DIR/zoo.cfg $ZOO_CONF_DIR/zoo.cfg.org
sed -i "s/server\.$ZOO_MY_ID\=[a-z0-9.-]*/server.$ZOO_MY_ID=0.0.0.0/" "$ZOO_CONF_DIR/zoo.cfg"

# Write myid only if it doesn't exist
if [ ! -f "$ZOO_DATA_DIR/myid" ]; then
    echo "${ZOO_MY_ID:-1}" > "$ZOO_DATA_DIR/myid"
fi

exec "$@"
```

Dockerfile
```bash
$ cat Dockerfile
FROM openjdk:8-jre-slim

ENV ZOO_CONF_DIR=/conf \
    ZOO_DATA_DIR=/data \
    ZOO_DATA_LOG_DIR=/datalog \
    ZOO_LOG_DIR=/logs \
    ZOO_TICK_TIME=2000 \
    ZOO_INIT_LIMIT=5 \
    ZOO_SYNC_LIMIT=2 \
    ZOO_AUTOPURGE_PURGEINTERVAL=0 \
    ZOO_AUTOPURGE_SNAPRETAINCOUNT=3 \
    ZOO_MAX_CLIENT_CNXNS=60 \
    ZOO_STANDALONE_ENABLED=true \
    ZOO_ADMINSERVER_ENABLED=true \
    ZOO_COMMANDS_WHITELIST=*

# Add a user with an explicit UID/GID and create necessary directories
RUN set -eux; \
    groupadd -r zookeeper --gid=1000; \
    useradd -r -g zookeeper --uid=1000 zookeeper; \
    mkdir -p "$ZOO_DATA_LOG_DIR" "$ZOO_DATA_DIR" "$ZOO_CONF_DIR" "$ZOO_LOG_DIR"; \
    chown zookeeper:zookeeper "$ZOO_DATA_LOG_DIR" "$ZOO_DATA_DIR" "$ZOO_CONF_DIR" "$ZOO_LOG_DIR"

# Install required packges
RUN set -eux; \
    apt-get update; \
    DEBIAN_FRONTEND=noninteractive \
    apt-get install -y --no-install-recommends \
        ca-certificates \
        dirmngr \
        gosu \
        gnupg \
        netcat \
        wget; \
    rm -rf /var/lib/apt/lists/*; \
# Verify that gosu binary works
    gosu nobody true

ARG GPG_KEY=3F7A1D16FA4217B1DC75E1C9FFE35B7F15DFA1BA
ARG SHORT_DISTRO_NAME=zookeeper-3.5.5
ARG DISTRO_NAME=apache-zookeeper-3.5.5-bin

# Download Apache Zookeeper, verify its PGP signature, untar and clean up
COPY apache-zookeeper-3.5.5-bin.tar.gz ./
RUN set -eux; \
#    wget -q "https://www.apache.org/dist/zookeeper/$SHORT_DISTRO_NAME/$DISTRO_NAME.tar.gz"; \
    wget -q "https://www.apache.org/dist/zookeeper/$SHORT_DISTRO_NAME/$DISTRO_NAME.tar.gz.asc"; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --keyserver ha.pool.sks-keyservers.net --recv-key "$GPG_KEY" || \
    gpg --keyserver pgp.mit.edu --recv-keys "$GPG_KEY" || \
    gpg --keyserver keyserver.pgp.com --recv-keys "$GPG_KEY"; \
    gpg --batch --verify "$DISTRO_NAME.tar.gz.asc" "$DISTRO_NAME.tar.gz"; \
    tar -zxf "$DISTRO_NAME.tar.gz"; \
    mv "$DISTRO_NAME/conf/"* "$ZOO_CONF_DIR"; \
    rm -rf "$GNUPGHOME" "$DISTRO_NAME.tar.gz" "$DISTRO_NAME.tar.gz.asc"; \
    chown -R zookeeper:zookeeper "/$DISTRO_NAME"

WORKDIR $DISTRO_NAME
VOLUME ["$ZOO_DATA_DIR", "$ZOO_DATA_LOG_DIR", "$ZOO_LOG_DIR"]

EXPOSE 2181 2888 3888 8080

ENV PATH=$PATH:/$DISTRO_NAME/bin \
    ZOOCFGDIR=$ZOO_CONF_DIR

COPY docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]
CMD ["zkServer.sh", "start-foreground"]
```