layout: post
title: mongodb4.0分片集群安装
author: Pyker
categories: SQL
tags:
  - mongodb
date: 2019-07-7 11:12:00
---

# mongodb集群概念介绍
mongodb支持架构有`单机(stand-alone)`、`主从(master-slave)`、`副本集(replica set)`以及`分片(sharding)`, 而最常用的架构莫过于`副本集 + 分片`。而分片有三大组件，分别为`mongos`、`configsvr`、`sharding server`，他们的功能如下：
* `mongos`: 它是前端路由，应用程序由此接入，且让整个集群看上去像单一数据库，前端应用可以透明使用。它不存储任何数据，只载入configsvr的配置。
* `configsvr`: 它是一个mongod 实例，存储了整个 Cluster Metadata，其中包括 chunk 信息。它是存放着分片和数据的对应关系。
* `sharding server`: 它是一个mongod 实例，用于存储实际的数据块，实际生产环境中一个shard server角色可由几台机器组个一个replica set承担，防止主机单点故障。

# 安装环境
本次试验有三台主机，即三个分片和三个副本集，可以保证高可用，即使一台机器全宕机了，服务仍然能够正常访问, 本次环境的主机分别充当另外一个分片的从，他们的角色功能如下：
> 角色后面的数字为该服务在该主机上占用的端口

主机 | IP | 角色
--- | --- | ---
mongo1 | 192.168.3.125 | mongos1:20000<br>configsvr1:21000<br>shard1:27001<br>shard2:27002<br>shard3:27003
mongo2 | 192.168.3.126 | mongos2:20000<br>configsvr2:21000<br>shard1:27001<br>shard2:27002<br>shard3:27003
mongo3 | 192.168.3.127 | configsvr3:21000<br>shard1:27001<br>shard2:27002<br>shard3:27003

# 试验架构
本次试验环境的架构图如下：
![](/images/pic/mongodb-soa.png)

# 环境准备
> 环境准备的操作在所有主机上执行，除非另有说明。

## 创建mongod用户
```bash
$ useradd mongod
```

## 下载mongodb二进制文件
```bash
$ wget https://fastdl.mongodb.org/linux/mongodb-linux-x86_64-rhel70-4.0.10.tgz
$ tar zxvf mongodb-linux-x86_64-rhel70-4.0.10.tgz
$ cp mongodb-linux-x86_64-rhel70-4.0.10/bin/* /usr/bin
```

## 准备环境目录
```bash
$ mkdir -pv /usr/local/mongodb/conf     # 用于存放集群配置文件
$ mkdir -pv /data/mongos/log            # 用于存放mongos日志文件，它本身不存在数据，只是路由
$ mkdir -pv /data/{config,shard1,shard2,shard3}/{data,log}    # configsvr和3个分片的数据目录、日志目录
```

## 关闭Transparent HugePages
```bash
$ cat > /etc/systemd/system/thpset.service << EOF
[Unit]
Description="thp set"

[Service]
User=root
PermissionsStartOnly=true
Type=oneshot
ExecStart=/bin/bash -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
ExecStart=/bin/bash -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

[Install]
WantedBy=multi-user.target
EOF
$ systemctl start thpset && systemctl enable thpset
```

# 集群认证文件
在分片集群环境中，副本集内成员之间需要用`keyFile`认证，mongos与配置服务器，副本集之间也要keyFile认证，集群所有`mongod`和`mongos`实例使用内容相同的keyFile文件，我们这里只在mongo1上生成，然后复制到其他节点上。
```bash
$ openssl rand -base64 753  > /usr/local/mongodb/keyfile
$ scp /usr/local/mongodb/keyfile mongo2:/usr/local/mongodb/keyfile
$ scp /usr/local/mongodb/keyfile mongo3:/usr/local/mongodb/keyfile
$ chmod 400 /usr/local/mongodb/keyfile
```
该文件在mongodb集群配置中需要使用到，所以我们先生成。

# 配置ConfigSvr
分别在mongo1、mongo2、mongo3上配置，要修改的地方只有bindIp为本机IP即可：
```bash
$ cat > /usr/local/mongodb/conf/config.conf << EOF
## content
systemLog:
  destination: file
  logAppend: true
  path: /data/config/log/config.log

# Where and how to store data.
storage:
  dbPath: /data/config/data
  journal:
    enabled: true
# how the process runs
processManagement:
  fork: true
  pidFilePath: /data/config/log/configsrv.pid

# network interfaces
net:
  port: 21000
  bindIp: 192.168.3.125

#operationProfiling:
replication:
    replSetName: config

sharding:
    clusterRole: configsvr

# configure security
security:
  authorization: enabled
  keyFile: /usr/local/mongodb/keyfile
EOF
```
分别在mongo1、mongo2、mongo3上配置systemctl文件：
```bash
$ cat > /etc/systemd/system/mongo-config.service << EOF
[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongod
Type=forking
ExecStart=/usr/bin/mongod --config /usr/local/mongodb/conf/config.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/usr/bin/mongod --shutdown --config /usr/local/mongodb/conf/config.conf

[Install]
WantedBy=multi-user.target
EOF
```
分别启动三台服务器的config server
```bash
$ chown -R mongod /data/ && chown -R mongod /usr/local/mongodb/
$ systemctl start mongo-config && systemctl enable mongo-config
```
登录任意一台配置服务器，初始化配置副本集
```bash
$ mongo 192.168.3.125:21000
> rs.initiate( {
    _id : "config",
     members : [
         {_id : 0, host : "192.168.3.125:21000" },
         {_id : 1, host : "192.168.3.126:21000" },
         {_id : 2, host : "192.168.3.127:21000" }
     ]
   }
)
> rs.status();    # 查看当前configsvr副本集状态
```
> 其中，`"_id"` : `"configs"`应与配置文件中配置的 replicaction.replSetName 一致，"members" 中的 "host" 为三个节点的ip和port。

# 配置分片、副本集
配置mongo1上的3个分片副本集：
```bash
$ vi /usr/local/mongodb/conf/shard1.conf
# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /data/shard1/log/shard1.log

# Where and how to store data.
storage:
  dbPath: /data/shard1/data
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
       cacheSizeGB: 20

# how the process runs
processManagement:
  fork: true
  pidFilePath: /data/shard1/log/shard1.pid

# network interfaces
net:
  port: 27001
  bindIp: 192.168.3.125

#operationProfiling:
replication:
    replSetName: shard1
sharding:
    clusterRole: shardsvr
# configure security
security:
  authorization: enabled
  keyFile: /usr/local/mongodb/keyfile
```
```bash
$ vi /usr/local/mongodb/conf/shard2.conf
# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /data/shard2/log/shard2.log

# Where and how to store data.
storage:
  dbPath: /data/shard2/data
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
       cacheSizeGB: 20

# how the process runs
processManagement:
  fork: true
  pidFilePath: /data/shard2/log/shard2.pid

# network interfaces
net:
  port: 27002
  bindIp: 192.168.3.125

#operationProfiling:
replication:
    replSetName: shard2
sharding:
    clusterRole: shardsvr
# configure security
security:
  authorization: enabled
  keyFile: /usr/local/mongodb/keyfile
```
```bash
$ vi /usr/local/mongodb/conf/shard3.conf
# where to write logging data.
systemLog:
  destination: file
  logAppend: true
  path: /data/shard3/log/shard3.log

# Where and how to store data.
storage:
  dbPath: /data/shard3/data
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
       cacheSizeGB: 20
# how the process runs
processManagement:
  fork: true
  pidFilePath: /data/shard3/log/shard3.pid

# network interfaces
net:
  port: 27003
  bindIp: 192.168.3.125


#operationProfiling:
replication:
    replSetName: shard3
sharding:
    clusterRole: shardsvr

# configure security
security:
  authorization: enabled
  keyFile: /usr/local/mongodb/keyfile
```
复制mongo1节点上的shard1.conf、shard2.conf、shard3.conf到mongo2和mongo3主机上。要改动的地方只有bindIp，修改为本机IP即可。

分别在mongo1、mongo2、mongo3上配置分片副本的systemctl文件：
```bash
$ cat > /etc/systemd/system/mongo-27001.service << EOF
[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongod
Type=forking
ExecStart=/usr/bin/mongod --config /usr/local/mongodb/conf/shard1.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/usr/bin/mongod --shutdown --config /usr/local/mongodb/conf/shard1.conf

[Install]
WantedBy=multi-user.target
EOF
```
```bash
$ cat > /etc/systemd/system/mongo-27002.service << EOF
[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongod
Type=forking
ExecStart=/usr/bin/mongod --config /usr/local/mongodb/conf/shard2.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/usr/bin/mongod --shutdown --config /usr/local/mongodb/conf/shard2.conf

[Install]
WantedBy=multi-user.target
EOF
```
```bash
$ cat > /etc/systemd/system/mongo-27003.service << EOF
[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongod
Type=forking
ExecStart=/usr/bin/mongod --config /usr/local/mongodb/conf/shard3.conf
ExecReload=/bin/kill -s HUP $MAINPID
ExecStop=/usr/bin/mongod --shutdown --config /usr/local/mongodb/conf/shard3.conf

[Install]
WantedBy=multi-user.target
EOF
```
分别启动三台服务器的分片副本
```bash
$ chown -R mongod /data/ && chown -R mongod /usr/local/mongodb/
$ systemctl start mongo-27001 && systemctl enable mongo-27001
$ systemctl start mongo-27002 && systemctl enable mongo-27002
$ systemctl start mongo-27003 && systemctl enable mongo-27003
```
登录任意一台配置服务器，初始化各个配置副本集
```bash
$ mongo 192.168.3.125:27001
> rs.initiate( {
    _id : "shard1",
     members : [
         {_id : 0, host : "192.168.3.125:21000" },
         {_id : 1, host : "192.168.3.126:21000" },
         {_id : 2, host : "192.168.3.127:21000" }
     ]
   }
)
> rs.status();    # 查看当前shard1副本集状态
```
```bash
$ mongo 192.168.3.125:27002
> rs.initiate( {
    _id : "shard2",
     members : [
         {_id : 0, host : "192.168.3.125:21000" },
         {_id : 1, host : "192.168.3.126:21000" },
         {_id : 2, host : "192.168.3.127:21000" }
     ]
   }
)

> rs.status();    # 查看当前shard2副本集状态
```
```bash
$ mongo 192.168.3.125:27003
> rs.initiate( {
    _id : "shard3",
     members : [
         {_id : 0, host : "192.168.3.125:21000" },
         {_id : 1, host : "192.168.3.126:21000" },
         {_id : 2, host : "192.168.3.127:21000" }
     ]
   }
)
> rs.status();    # 查看当前shard3副本集状态
```

# 配置mongos
mongos这里只配置在mongo1和mongo2上，所以只要在这两台主机上配置即可。
> 注意：先启动配置服务器和分片服务器,后启动路由实例

分别在mongo1、mongo2上配置，要修改的地方只有bindIp为本机IP即可：
```bash
$ vi /usr/local/mongodb/conf/mongos.conf
systemLog:
  destination: file
  logAppend: true
  path: /data/mongos/log/mongos.log
processManagement:
  fork: true
  pidFilePath: /data/mongos/log/mongos.pid

# network interfaces
net:
  port: 20000
  bindIp: 192.168.3.125
#监听的配置服务器,只能有1个或者3个 configs为配置服务器的副本集名字
sharding:
   configDB: config/192.168.3.125:21000,192.168.3.126:21000,192.168.3.127:21000

# configure security
security:
  keyFile: /usr/local/mongodb/keyfile
```
分别在mongo1、mongo2上配置systemctl文件：
```bash
$ cat > /etc/systemd/system/mongos.service << EOF
[Unit]
Description=High-performance, schema-free document-oriented database
After=network.target

[Service]
User=mongod
Type=forking
ExecStart=/usr/bin/mongos --config /usr/local/mongodb/conf/mongos.conf
ExecReload=/bin/kill -s HUP $MAINPID

[Install]
WantedBy=multi-user.target
EOF
```
分别启动mongo1和mongo2服务器上的mongos
```bash
$ chown -R mongod /data/ && chown -R mongod /usr/local/mongodb/
$ systemctl start mongos && systemctl enable mongos
$ systemctl start mongos && systemctl enable mongos
```
至此已经搭建了mongodb配置服务器、路由服务器，各个分片服务器，不过应用程序连接到mongos路由服务器并不能使用分片机制，还需要在程序里设置分片配置，让分片生效。
登录任意一台mongos，设置分片配置。
```bash
$ mongo 192.168.3.125:20000
> sh.addShard("shard1/192.168.3.125:27001,192.168.3.126:27001,192.168.3.127:27001")
> sh.addShard("shard2/192.168.3.125:27002,192.168.3.126:27002,192.168.3.127:27002")
> sh.addShard("shard3/192.168.3.125:27003,192.168.3.126:27003,192.168.3.127:27003")
> sh.status()    # 查看集群状态
```
# 创建管理员帐号
登陆任意一台mongos，建立管理员账号，赋所有权限（admin和config数据库）
```bash
$ mongo 192.168.3.125:20000
> use admin
> db.createUser({user: "admin",pwd: "123456",roles: [ { role: "root", db: "admin" } ]}) # root所有权限
> use config
> db.createUser({user: "admin",pwd: "123456",roles: [ { role: "root", db: "admin" } ]}) # root所有权限
> db.auth("admin","123456")  # 登陆认证，返回1为验证成功
```
```bash
$ netstat -ptln | grep mongo
tcp        0      0 192.168.3.125:21000     0.0.0.0:*               LISTEN      3277/mongod         
tcp        0      0 192.168.3.125:27001     0.0.0.0:*               LISTEN      3974/mongod         
tcp        0      0 192.168.3.125:27002     0.0.0.0:*               LISTEN      3281/mongod         
tcp        0      0 192.168.3.125:27003     0.0.0.0:*               LISTEN      3280/mongod         
tcp        0      0 192.168.3.125:20000     0.0.0.0:*               LISTEN      3251/mongos
```
# 查看集群状态
登陆任意一台mongos查看集群状态
```bash
$ mongo 192.168.3.125:20000
mongos> use admin
mongos> db.auth("admin","123456")   # 这里要认证了。不然看不到集群信息
mongos> sh.status();
--- Sharding Status --- 
  sharding version: {
  	"_id" : 1,
  	"minCompatibleVersion" : 5,
  	"currentVersion" : 6,
  	"clusterId" : ObjectId("5d208c95fdd1bee8ab631a40")
  }
  shards:
        {  "_id" : "shard1",  "host" : "shard1/192.168.3.125:27001,192.168.3.126:27001,192.168.3.127:27001",  "state" : 1 }
        {  "_id" : "shard2",  "host" : "shard2/192.168.3.125:27002,192.168.3.126:27002,192.168.3.127:27002",  "state" : 1 }
        {  "_id" : "shard3",  "host" : "shard3/192.168.3.125:27003,192.168.3.126:27003,192.168.3.127:27003",  "state" : 1 }
  active mongoses:
        "4.0.10" : 2
  autosplit:
        Currently enabled: yes
  balancer:
        Currently enabled:  yes
        Currently running:  no
        Failed balancer rounds in last 5 attempts:  0
        Migration Results for the last 24 hours: 
                No recent migrations
```
这样mongodb如架构图所示的集群搭建就已经完成了。