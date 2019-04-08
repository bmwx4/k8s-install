---
title: "系统初始化和全局变量 "
date: 2019-03-16T09:38:20+08:00
draft: false
---

#### 准备系统环境  
os version: centos 7

----  

#### 部署拓扑：

| Hostname  | IP | Roles |
| :------------- | :------------- | :------------- |
| master01  | 192.168.10.232  | kube-apiserver kube-controllermanager kube-scheduler etcd |
| node01  | 192.168.10.242  | kubelet |
| node02  | 192.168.10.243  | kubelet |

修改三个节点的主机名:
```bash
[root@master01 ~]# hostnamectl --static set-hostname master01
[root@master01 ~]# echo "master01" > /etc/hostname

[root@node1 ~]# hostnamectl --static set-hostname node1
[root@node1 ~]# echo "node1" > /etc/hostname

[root@node2 ~]# hostnamectl --static set-hostname node2
[root@node2 ~]# echo "node2" > /etc/hostname
```

-----

#### 添加主机名解析
```bash
# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.10.242 node01
192.168.10.243 node02
192.168.10.232 maste01
```
----
#### 安装系统依赖
```
# yum install -y epel-release
# yum install -y conntrack ipvsadm ipset jq sysstat curl iptables libseccomp
```
-----
#### 关闭防火墙  
```
#  systemctl stop firewalld
#  systemctl disable firewalld
#  iptables -F &&  iptables -X &&  iptables -F -t nat &&  iptables -X -t nat
#  iptables -P FORWARD ACCEPT
```
***验证防火墙是否关闭***
```bash
# firewall-cmd --state
not running
```

----
#### 关闭 swap 分区
```
# swapoff -a
```
为了防止开机自动挂载 swap 分区，可以注释 /etc/fstab 中相应的条目：
````
# sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
````
***PS:***
>不禁用swap的话，在启动kubelet 或者 docker 都会有问题；

-----
#### 关闭 SELinux，否则后续 K8S 挂载目录时可能报错 Permission denied：
```
# setenforce 0
# grep SELINUX /etc/selinux/config
SELINUX=disabled
```
-----
#### 关闭 dnsmasq (可选)
linux 系统开启了 dnsmasq 后(如 GUI 环境)，将系统 DNS Server 设置为 127.0.0.1，这会导致 docker 容器无法解析域名，需要关闭它：
```
# service dnsmasq stop
# systemctl disable dnsmasq
```
-----
#### 安装网络模块
```
# modprobe br_netfilter
# modprobe ip_vs
```
***PS:***
> flannel 网络方案会用到

-----
#### 设置系统参数
```
# cat > kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF
# cp kubernetes.conf  /etc/sysctl.d/kubernetes.conf
# sysctl -p /etc/sysctl.d/kubernetes.conf
# mount -t cgroup -o cpu,cpuacct none /sys/fs/cgroup/cpu,cpuacct
```
**PS:**  
> tcp_tw_recycle 和 Kubernetes 的 NAT 冲突，必须关闭 ，否则会导致服务不通；  
> [被抛弃的tcp_recycles](https://juejin.im/post/5c0642e65188251a82662912)  
> [一个NAT问题引起的思考](http://perthcharles.github.io/2015/08/27/timestamp-NAT/)  
> 关闭不使用的 IPV6 协议栈，防止触发 docker BUG；

------
#### 设置系统时区
```
调整系统 TimeZone
# timedatectl set-timezone Asia/Shanghai

更新系统时间
# ntpdate cn.pool.ntp.org

将当前的 UTC 时间写入硬件时钟
# timedatectl set-local-rtc 0

重启依赖于系统时间的服务
# systemctl restart rsyslog
# systemctl restart crond
```
-----
#### 添加 k8s 账户
```bash
useradd -m k8s
sh -c 'echo 123456 | passwd k8s --stdin' # 为 k8s 账户设置密码
```
因为我们k8s组件etcd和docker组件是用k8s用户启动的，但是在生产环境中，为了提高安全，可是使用如下命令设置用户没有登录shell的能力；
```bash
# chsh k8s -s /sbin/nologin
```

-----

#### 创建工作目录
```
# mkdir -p /opt/k8s/bin
# chown -R k8s /opt/k8s

# mkdir -p /etc/kubernetes/cert
# chown -R k8s /etc/kubernetes

master only:
# mkdir -p /etc/etcd/cert
# chown -R k8s /etc/etcd/cert
# mkdir -p /var/lib/etcd && chown -R k8s /etc/etcd/cert
```
***PS:***
> 在下一步启动组件时，很可能会因为权限不对，导致服务启动失败，要注意给相关目录设置属主为 k8s

-----
#### 将可执行文件路径 /opt/k8s/bin 添加到 PATH 变量中
```
# sh -c "echo 'export PATH=/opt/k8s/bin:$PATH' >> ~/.bashrc"
# source ~/.bashrc
```
