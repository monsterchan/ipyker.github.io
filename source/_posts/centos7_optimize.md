layout: post
title: Centos7.X系统初始化优化脚本
author: Pyker
categories: Linux
tags:
  - shell
  - Linux
date: 2018-04-30 19:43:00
---

```bash
#!/bin/bash
#################################################
#  --Info
#         Initialization CentOS 7.x script
#################################################
# Check if user is root
#
if [ $(id -u) != "0" ]; then
    echo "Error: You must be root to run this script, please use root to initialization OS."
    exit 1
fi
echo "+------------------------------------------------------------------------+"
echo "|       To initialization the system for security and performance        |"
echo "+------------------------------------------------------------------------+"

# set hostname
edit_hostname() {
  read -p "Please input your hostname: " hs
  echo "${hs}" > /etc/hostname

}
# add user for os
user_add() {
  read -p "To create a system user, enter username created or input 'n' not created: " new_user
  if [ $new_user == "n" ]; then
    echo "Skip create user..."
  else
    id -u $new_user
    if [ $? -ne 0 ];then
      useradd -s /bin/bash -d /home/$new_user -m $new_user && echo password | passwd --stdin $new_user && echo "mongod ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/$new_user
    else
      echo "$new_user user is exist."
    fi
  fi
}
# update system & install pakeage
system_update() {
    echo "*** Starting update system && install tools pakeage... ***"
    mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
    curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    yum -y update
    yum clean all && yum makecache
    yum -y install rsync wget vim openssh-clients iftop htop iotop sysstat lsof telnet traceroute tree lrzsz  net-tools dstat tree ntpdate dos2unix net-tools egrep
    [ $? -eq 0 ] && echo "System upgrade && install pakeages complete."
}
# Set timezone synchronization
timezone_config() {
    echo "Setting timezone..."
    /usr/bin/timedatectl | grep "Asia/Shanghai"
    if [ $? -eq 0 ];then
       echo "System timezone is Asia/Shanghai."
       else
       timedatectl set-local-rtc 0 && timedatectl set-timezone Asia/Shanghai
    fi
    # config chrony
    yum -y install chrony && systemctl start chronyd.service && systemctl enable chronyd.service
    [ $? -eq 0 ] && echo "Setting timezone && Sync network time complete."
}
# disable selinux
selinux_config() {
  sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
  setenforce 0
  echo "Dsiable selinux complete."
}
# ulimit comfig
ulimit_config() {
  echo "Starting config ulimit..."
  cat >> /etc/security/limits.conf << EOF
* soft nproc 65535
* hard nproc 65535
* soft nofile 65535
* hard nofile 65535
EOF
  [ $? -eq 0 ] && echo "Ulimit config complete!"
}
# sshd config
sshd_config() {
    echo "Starting config sshd..."
    sed -i '/^#Port/s/#Port 22/Port 2024/g' /etc/ssh/sshd_config
    #sed -i "$ a\ListenAddress 0.0.0.0:21212\nListenAddress 0.0.0.0:22 " /etc/ssh/sshd_config
    sed -i '/^#UseDNS/s/#UseDNS yes/UseDNS no/g' /etc/ssh/sshd_config
    systemctl restart sshd
    #sed -i 's/#PermitRootLogin yes/PermitRootLogin no/g' /etc/ssh/sshd_config
    #sed -i 's/#PermitEmptyPasswords no/PermitEmptyPasswords no/g' /etc/ssh/sshd_config
    [ $? -eq 0 ] && echo "SSH config complete."
}
# firewalld config
disable_firewalld() {
   echo "clean iptables roles..."
   iptables -F && iptables -X && iptables -t nat -F && iptables -t nat -X
   echo "Starting disable firewalld..."
   rpm -qa | grep firewalld >> /dev/null
   if [ $? -eq 0 ];then
      systemctl stop firewalld  && systemctl disable firewalld
      [ $? -eq 0 ] && echo "Disable firewalld complete."
      else
      echo "Firewalld not install."
   fi
}
# sysctl config
config_sysctl() {
    echo "Staring config sysctl..."
    /usr/bin/cp -f /etc/sysctl.conf /etc/sysctl.conf.bak
    modprobe nf_conntrack_ipv4
    modprobe nf_conntrack
    cat > /etc/sysctl.conf << EOF
fs.file-max=1000000
vm.swappiness = 0
net.ipv4.ip_forward = 1
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.ip_local_port_range = 1024 65000
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_tw_recycle = 0
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_syncookies = 1
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
# set kernel parameters work
    /usr/sbin/sysctl -p
    [ $? -eq 0 ] && echo "Sysctl config complete."
}
# close THP
close_thp() {
  echo "Close THP..."
  if [[ -f /sys/kernel/mm/transparent_hugepage/enabled ]]; then
    echo never > /sys/kernel/mm/transparent_hugepage/enabled
  fi
  if [[ -f /sys/kernel/mm/transparent_hugepage/defrag ]]; then
    echo never > /sys/kernel/mm/transparent_hugepage/defrag
  fi
  echo "Power Boot close THP..."
  cat >> /etc/rc.local << EOF
if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
   echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
   echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
EOF
  chmod +x /etc/rc.d/rc.local
}
# disable NUMA
disable_numa() {
  echo "configure /etc/default/grub numa=off..."
  sed -i "s/rhgb quiet/rhgb quiet numa=off/g" /etc/default/grub
  echo "rebuild grub2.cfg configure file..."
  grub2-mkconfig -o /etc/grub2.cfg
}
#main function
main() {
    edit_hostname
    user_add
    system_update
    timezone_config
    selinux_config
    ulimit_config
    #sshd_config
    disable_firewalld
    config_sysctl
    close_thp
    disable_numa
}
# execute main functions
main
echo "+------------------------------------------------------------------------+"
echo "|            To initialization system all completed !!!                  |"
echo "|   You can now restart the system for the configuration to take effect  |"
echo "+------------------------------------------------------------------------+"
```