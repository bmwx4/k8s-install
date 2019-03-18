---
title: "Docker"
date: 2019-03-16T18:23:58+08:00
draft: false
---

可以仿照我们以前使用yum安装docker的经验来安装docker，也可是自行下载二进制安装,到 https://download.docker.com/linux/static/stable/x86_64/ 页面下载最新发布包：
```
wget https://download.docker.com/linux/static/stable/x86_64/docker-18.06.1-ce.tgz
tar -xvf docker-18.06.1-ce.tgz
```
-----
#### 创建和分发 systemd unit 文件
```
cat > docker.service <<"EOF"
[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.io

[Service]
Environment="PATH=/opt/k8s/bin:/bin:/sbin:/usr/bin:/usr/sbin"
EnvironmentFile=-/run/flannel/docker
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock  $DOCKER_NETWORK_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
Restart=on-failure
RestartSec=5
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
Delegate=yes
KillMode=process

[Install]
WantedBy=multi-user.target
EOF
```
***PS***
```
EOF 前后有双引号，这样 bash 不会替换文档中的变量，如 $DOCKER_NETWORK_OPTIONS；
dockerd 运行时会调用其它 docker 命令，如 docker-proxy，所以需要将 docker 命令所在的目录加到 PATH 环境变量中.但是我们使用
yum安装的不需要这么操作，所以，如果使用yum安装的忽略 Environment 这一行；
flanneld 启动时将网络配置写入 /run/flannel/docker 文件中，dockerd 启动前读取该文件中的环境变量 DOCKER_NETWORK_OPTIONS ，然后设置 docker0 网桥网段；
如果指定了多个 EnvironmentFile 选项，则必须将 /run/flannel/docker 放在最后(确保 docker0 使用 flanneld 生成的 bip 参数)；
docker 需要以 root 用于运行；
docker 从 1.13 版本开始，可能将 iptables FORWARD chain的默认策略设置为DROP，从而导致 ping 其它 Node 上的 Pod IP 失败，遇到这种情况时，需要手动设置策略为 ACCEPT：iptables -P FORWARD ACCEPT
并且把以下命令写入 /etc/rc.local 文件中，防止节点重启iptables FORWARD chain的默认策略又还原为DROP
```
-----

#### 启动 docker 服务
```
systemctl stop firewalld && systemctl disable firewalld
iptables -F && /usr/sbin/iptables -X && /usr/sbin/iptables -F -t nat && /usr/sbin/iptables -X -t nat
iptables -P FORWARD ACCEPT
systemctl daemon-reload && systemctl enable docker && systemctl restart docker
for intf in /sys/devices/virtual/net/docker0/brif/*; do echo 1 > $intf/hairpin_mode; done
sysctl -p /etc/sysctl.d/kubernetes.conf
```
***PS***
```
关闭 firewalld(centos7)/ufw(ubuntu16.04)，否则可能会重复创建 iptables 规则；
清理旧的 iptables rules 和 chains 规则；
开启 docker0 网桥下虚拟网卡的 hairpin 模式,可能失败，可以忽略;
```

----
#### 检查 docker0 网桥
确认各 work 节点的 docker0 网桥和 flannel.1 接口的 IP 处于同一个网段中(如下 172.30.14.0 和 172.30.14.1)：
```
[root@node01 ~]# ip addr show flannel.1
9: flannel.1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UNKNOWN group default
    link/ether 52:df:0e:a6:00:e7 brd ff:ff:ff:ff:ff:ff
    inet 172.30.14.0/32 scope global flannel.1
       valid_lft forever preferred_lft forever
[root@node01 ~]# ip addr show docker0
6: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default
    link/ether 02:42:8e:35:b1:14 brd ff:ff:ff:ff:ff:ff
    inet 172.30.14.1/24 brd 172.30.14.255 scope global docker0
       valid_lft forever preferred_lft forever
```
