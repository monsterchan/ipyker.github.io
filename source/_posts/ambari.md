layout: post
title: Ambari 2.7.3与HDP 3.1.0安装过程
author: Pyker
categories: ambari
tags:
  - ambari
date: 2019-07-30 19:43:00
---

# 概念介绍
## Amabri介绍
* Ambari 跟 Hadoop 等开源软件一样，也是 Apache Software Foundation 中的一个项目，并且是顶级项目。就 Ambari 的作用来说，就是创建、管理、监视 Hadoop 的集群，但是这里的 Hadoop 指的是 Hadoop 整个生态圈（例如 Hive，Hbase，Sqoop，Zookeeper 等）， 而并不仅是特指 Hadoop。用一句话来说，Ambari 就是为了让 Hadoop 以及相关的大数据软件更容易使用的一个工具。
* Ambari 自身也是一个分布式架构的软件，主要由两部分组成：Ambari Server 和 Ambari Agent。简单来说，用户通过 Ambari Server 通知 Ambari Agent 安装对应的软件；Agent 会定时地发送各个机器每个软件模块的状态给 Ambari Server，最终这些状态信息会呈现在 Ambari 的 GUI，方便用户了解到集群的各种状态，并进行相应的维护。

## HDP介绍
* HDP是hortonworks的软件栈，里面包含了hadoop生态系统的所有软件项目，比如HBase,Zookeeper,Hive,Pig等等。

## HDP-UTILS介绍
* HDP-UTILS是工具类库。

# 安装前准备
## 主机列表
本次实验选择6台Cetnos7.6主机，其中2台作为`Ambari Server`主机，6台作为`Ambari Agent`。而2台`Ambari Server`主机使用静态手动配置VIP方式进行切换高可用。或使用keepalive+haproxy方式进行高可用。

| 主机名/解析记录 | IP | 运行组件 |
| --- | ---|  --- |
| bigdata1.jms.com | 192.168.12.201 | ambari-server、ambari-agent、mysql-master |
| bigdata2.jms.com | 192.168.12.202 | ambari-server、ambari-agent、mysql-slave |
| bigdata3.jms.com | 192.168.12.203 | ambari-agent |
| bigdata4.jms.com | 192.168.12.204 | ambari-agent |
| bigdata5.jms.com | 192.168.12.205 | ambari-agent、ambari-yumrepo |
| bigdata6.jms.com | 192.168.12.206 | ambari-agent |
| ambari.jms.com | 192.168.12.200 | ambari-server VIP地址 |

> 注意：
* 2台Ambari Server主机到Ambari Agent主机配置免密登录
* Ambari Server/Agent主机需安装JDK
* 确保主机的hostname -f 满足FQDN格式（在安装集群的第三步Confirm Host需要）
* 关闭防火墙
* 确认主机字符集编码为UTF-8（否则Ambari Server 配置数据库可能报错）
* 开启NTP服务
* selinux配置为disbaled
* 配置描述文件数量65535

## 配置免密钥登陆
```bash
# bigdata1、bigdata2都要操作
$ ssh-keygen -t rsa
$ for i in `seq 1 6`;do ssh-copy-id bigdata${i}.jms.com;done 
```
## 手动配置VIP地址
先在192.168.12.201服务器上配置ambari-server VIP，当192.168.12.201宕机后，手动配置`ifcfg-em1:1`到192.168.12.202上。
```bash
$ cat > /etc/sysconfig/network-scripts/ifcfg-em1:1 << EOF
TYPE=Ethernet
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
NAME=em1:1
DEVICE=em1:1
ONBOOT=yes
IPADDR=192.168.12.200
PREFIX=23
GATEWAY=192.168.13.1
DNS1=192.168.1.211
EOF

$ systemctl restart network
```
> 注意： 切勿在192.168.12.202同时配置，否则由于VIP不稳定，造成ambari集群各种故障。只需在192.168.12.201宕机后配置即可，或者直接使用keepalive+haproxy。

## 配置本地解析
```bash
$ cat >> /etc/hosts << EOF
192.168.12.200  ambari.jms.com
192.168.12.201  bigdata1.jms.com bigdata1
192.168.12.202  bigdata2.jms.com bigdata2
192.168.12.203  bigdata3.jms.com bigdata3
192.168.12.204  bigdata4.jms.com bigdata4
192.168.12.205  bigdata5.jms.com bigdata5
192.168.12.206  bigdata6.jms.com bigdata6
EOF
```
## 配置sysctl文件
```bash
$ cat > /etc/sysctl.conf << EOF
fs.file-max=65535
vm.swappiness=1
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0

net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_tw_recycle = 0

net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_syncookies = 1

kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296

net.ipv4.tcp_max_syn_backlog = 262144
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 262144
net.ipv4.tcp_max_orphans = 262144

net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30

net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096 87380 4194304
net.ipv4.tcp_wmem = 4096 16384 4194304
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.nf_conntrack_max = 6553500
net.netfilter.nf_conntrack_max = 6553500
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_established = 3600
EOF
```
## 安装基础软件包
```bash
$ yum install -y lrzsz ntpdate supervisor sysstat net-tools vim wget ntp lsof
```
## 关闭transparent_hugepage
```bash
$ cat >> /etc/rc.local << EOF
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
   echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if -f /sys/kernel/mm/transparent_hugepage/defrag; then
   echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
EOF
chmod +x /etc/rc.local
```

# 安装mysql主从



# 配置ambari yum私仓
在这里ambari yum私仓安装在192.168.12.205服务器上。
## 下载私仓包
```bash
$ mkdir -p /data/wwwroot/default/{ambari,hdp/{HDP-UTILS-1.1.0.22}}
$ wget http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.3.0/ambari-2.7.3.0-centos7.tar.gz
$ wget http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.1.0.0/HDP-3.1.0.0-centos7-rpm.tar.gz
$ wget http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz

$ tar -zxvf ambari-2.7.3.0-centos7.tar.gz -C /data/wwwroot/default/ambari
$ tar -zxvf HDP-3.1.0.0-centos7-rpm.tar.gz -C /data/wwwroot/default/hdp/
$ tar -zxvf HDP-UTILS-1.1.0.22-centos7.tar.gz -C /data/wwwroot/default/hdp/HDP-UTILS-1.1.0.22/
```

## 配置nginx文件服务
### 安装nginx
```bash
# yum安装或者源代码安装nginx，我是源代码安装nginx的
$ yum install nginx -y
```
### 修改配置文件
```bash
$ vi /usr/local/nginx/conf/nginx.conf
...略
  server {
    listen 80;
    server_name bigdata5.jms.com;
    access_log /data/wwwlogs/access_nginx.log combined;
    index index.html index.htm index.php;
    autoindex on;
    autoindex_exact_size on;
    autoindex_localtime on;

    location /ambari {
      root /data/wwwroot/default;
    }
    location /hdp {
      root /data/wwwroot/default;
    }
  }
```
浏览器中访问http://bigdata5.jms.com/ambari 和 http://bigdata5.jms.com/hdp ,访问正常即ambari文件服务配置正常。

## 配置私仓repo
```bash
$ cat > /etc/yum.repos.d/ambari.repo << EOF
#VERSION_NUMBER=2.7.3.0-139
[ambari-2.7.3.0]
#json.url = http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json
name=ambari Version - ambari-2.7.3.0
#baseurl=http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.3.0
baseurl=http://bigdata5.jms.com/ambari/ambari/centos7/2.7.3.0-139
gpgcheck=1
gpgkey=http://bigdata5.jms.com/ambari/ambari/centos7/2.7.3.0-139/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
EOF

$ cat > /etc/yum.repos.d/HDP.repo << EOF
#VERSION_NUMBER=3.1.0.0-78
[HDP-3.1.0.0]
name=HDP Version - HDP-3.1.0.0
baseurl=http://bigdata5.jms.com/hdp/HDP/centos7/
gpgcheck=1
gpgkey=http://bigdata5.jms.com/hdp/HDP/centos7/3.1.0.0-78/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1

[HDP-UTILS-1.1.0.22]
name=HDP-UTILS Version - HDP-UTILS-1.1.0.22
baseurl=http://bigdata5.jms.com/hdp/HDP-UTILS-1.1.0.22/
gpgcheck=1
gpgkey=http://bigdata5.jms.com/hdp/HDP-UTILS-1.1.0.22/HDP-UTILS/centos7/1.1.0.22/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
EOF
```
将Ambari.repo和HDP.repo分发到bigdata1，2，3，4，6节点/etc/yum.repos.d下。

## 生成本地源
使用createrepo命令，创建yum本地源（软件仓库），即为存放本地特定位置的众多rpm包建立索引，描述各包所需依赖信息，并形成元数据。
```bash
$ yum install createrepo -y
$ createrepo /data/wwwroot/default/hdp/HDP/centos7/
$ createrepo /data/wwwroot/default/hdp/HDP-UTILS-1.1.0.22/
```

# 安装ambari-server
在192.168.12.201和192.168.12.202上安装ambari-server。安装前请配置mysql和ambari连接器，如下：
```bash
$ ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
```
```bash
# 安装ambari-server
$ yum install -y ambari-server
# 配置ambari-server
$ ambari-server setup
Using python  /usr/bin/python
Setup ambari-server
Checking SELinux...
SELinux status is 'disabled'
Customize user account for ambari-server daemon [y/n] (n)? y
Enter user account for ambari-server daemon (root):root
Adjusting ambari-server permissions and ownership...
Checking firewall status...
Checking JDK...
Do you want to change Oracle JDK [y/n] (n)? y
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
==============================================================================
Enter choice (1): 2
WARNING: JDK must be installed on all hosts and JAVA_HOME must be valid on all hosts.
WARNING: JCE Policy files are required for configuring Kerberos security. If you plan to use Kerberos,please make sure JCE Unlimited Strength Jurisdiction Policy Files are valid on all hosts.
Path to JAVA_HOME: /usr/lib/jvm/java-openjdk
Validating JDK on Ambari Server...done.
Checking GPL software agreement...
Completing setup...
Configuring database...
Enter advanced database configuration [y/n] (n)? Y
Configuring database...
==============================================================================
Choose one of the following options:
[1] - PostgreSQL (Embedded)
[2] - Oracle
[3] - MySQL / MariaDB
[4] - PostgreSQL
[5] - Microsoft SQL Server (Tech Preview)
[6] - SQL Anywhere
[7] - BDB
==============================================================================
Enter choice (3): 3         ####如果主机字符集编码未设置正确，可能会启动报错，具体可以查看日志/var/log/ambari-server/ambari-server.log
Hostname (localhost): ambari.jms.com
Port (3306): 		# 默认
Database name (ambari): 
Username (ambari): 
Enter Database Password (bigdata): 		# 密码不显示
Re-enter password: 
Configuring ambari database...
Should ambari use existing default jdbc /usr/share/java/mysql-connector-java.jar [y/n] (y)? y
Configuring remote database connection properties...
WARNING: Before starting Ambari Server, you must run the following DDL directly from the database shell to create the schema: /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql		# 此处需注意，启动ambari之前需要执行此句
Proceed with configuring remote database connection properties [y/n] (y)? y
Extracting system views...
ambari-admin-2.7.3.0.139.jar
....
Ambari repo file contains latest json url http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json, updating stacks repoinfos with it...
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.		# 安装成功
```
登陆主mysql开始导入ambari的数据库
```bash
mysql> use ambari;
mysql> source /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql;
```
无报错即导入正常。之后即可启动ambari-server
```bash
$ ambari-server start
Using python  /usr/bin/python
Starting ambari-server
Ambari Server running with administrator privileges.
Organizing resource files at /var/lib/ambari-server/resources...
Ambari database consistency check started...
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start......................................................
Server started listening on 8080

DB configs consistency check: no errors and warnings were found.
Ambari Server 'start' completed successfully.
```


# 安装ambari-agent
在所有节点上安装ambari-agent，执行以下命令安装：
```bash
$ yum install -y ambari-agent
```
由于ambari-server有2台主机，ambari-server使用的是VIP地址，所以我们这里得手动修改所有节点ambari-agent的配置文件。如下：
```bash
$ vi /etc/ambari-agent/conf/ambari-agent.ini
[server]
hostname=ambari.jms.com # 修改此地址即可
url_port=8440
secured_url_port=8441
connect_retry_delay=10
max_reconnect_retry_delay=30
...
```

# 访问ambari-server web页面
默认端口8080，Username：admin；Password：admin；http://ambari.jms.com:8080
![](/images/ambari/1.jpg)

