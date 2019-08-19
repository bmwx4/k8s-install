---
title: "部署 flannel 网络"
date: 2019-03-16T15:22:00+08:00
draft: true
---

kubernetes 要求集群内各节点(包括 master 节点)能通过 Pod 网段互联互通。flannel 使用 vxlan 技术为各节点创建一个可以互通的 Pod 网络，使用的端口为 UDP 8472，需要开放该端口（如公有云 AWS 等）。

flannel 第一次启动时，从 etcd 获取 Pod 网段信息，为本节点分配一个未使用的 /24 段地址，然后创建 flannel.1（也可能是其它名称，如 flannel1 等） 接口。

flannel 将分配的 Pod 网段信息写入 /run/flannel/docker 文件，docker 后续使用这个文件中的环境变量设置 docker0 网桥。

#### 下载和分发 flanneld 二进制文件,在node上执行
到 https://github.com/coreos/flannel/releases 页面下载最新版本的发布包：
```
# wget https://github.com/coreos/flannel/releases/download/v0.11.0/flannel-v0.11.0-linux-amd64.tar.gz
# tar xvf flannel-v0.11.0-linux-amd64.tar.gz
# cp flanneld /opt/k8s/bin/
# cp mk-docker-opts.sh  /opt/k8s/bin/
```

#### 创建 flannel 证书和私钥
flannel 从 etcd 集群存取网段分配信息，而 etcd 集群启用了双向 x509 证书认证，所以需要为 flanneld 生成证书和私钥,也可以复用etcd的客户端证书和key.
***创建证书签名请求：***
```
cat > flanneld-csr.json <<EOF
{
  "CN": "flanneld",
  "hosts": [],
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
***PS:***
```
该证书只会被 flannel 当做 client 证书使用，所以 hosts 字段为空；
```
***生成证书和私钥：***
```
cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
  -ca-key=/etc/kubernetes/cert/ca-key.pem \
  -config=/etc/kubernetes/cert/ca-config.json \
  -profile=kubernetes flanneld-csr.json | cfssljson -bare flanneld
ls flanneld*pem
```

将生成的证书和私钥分发到所有节点（master01,node01 和 node02）：
```bash
mkdir -p  /etc/flanneld/cert && chown -R k8s /etc/flanneld/cert
scp flanneld*.pem /etc/flanneld/cert
scp flanneld*.pem k8s@192.168.10.242:/etc/flanneld/cert
scp flanneld*.pem k8s@192.168.10.243:/etc/flanneld/cert
```
-----

#### 向 etcd 写入集群 Pod 网段信息
````
export ETCD_ENDPOINTS="https://192.168.10.232:2379"
# Pod 网段，建议 /16 段地址，部署前路由不可达，部署后集群内路由可达(flanneld 保证)
export CLUSTER_CIDR="172.30.0.0/16"
# flanneld 网络配置前缀
export FLANNEL_ETCD_PREFIX="/kubernetes/network"

etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/cert/ca.pem \
  --cert-file=/etc/flanneld/cert/flanneld.pem \
  --key-file=/etc/flanneld/cert/flanneld-key.pem \
  set ${FLANNEL_ETCD_PREFIX}/config '{"Network":"'${CLUSTER_CIDR}'", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}'

{"Network":"172.30.0.0/16", "SubnetLen": 24, "Backend": {"Type": "vxlan"}}
````
***PS***
```
flanneld 当前版本 (v0.11.0) 不支持 etcd v3，故使用 etcd v2 API 写入配置 key 和网段数据；
写入的 Pod 网段 ${CLUSTER_CIDR} 必须是 /16 段地址，必须与 kube-controller-manager 的 --cluster-cidr 参数值一致；
```
#### 验证写入 Pod 网段是否正确
```
etcdctl \
  --endpoints=${ETCD_ENDPOINTS} \
  --ca-file=/etc/kubernetes/cert/ca.pem \
  --cert-file=/etc/flanneld/cert/flanneld.pem \
  --key-file=/etc/flanneld/cert/flanneld-key.pem \
  get ${FLANNEL_ETCD_PREFIX}/config
```
----

#### 创建 flanneld 的 systemd unit 文件
在node01,node02上添加
```
export ETCD_ENDPOINTS="https://192.168.10.232:2379"
export IFACE="ens33"
export FLANNEL_ETCD_PREFIX="/kubernetes/network"
cat > flanneld.service << EOF
[Unit]
Description=Flanneld overlay address etcd agent
After=network.target
After=network-online.target
Wants=network-online.target
After=etcd.service
Before=docker.service

[Service]
Type=notify
ExecStart=/opt/k8s/bin/flanneld \\
  -etcd-cafile=/etc/kubernetes/cert/ca.pem \\
  -etcd-certfile=/etc/flanneld/cert/flanneld.pem \\
  -etcd-keyfile=/etc/flanneld/cert/flanneld-key.pem \\
  -etcd-endpoints=${ETCD_ENDPOINTS} \\
  -etcd-prefix=${FLANNEL_ETCD_PREFIX} \\
  -iface=${IFACE}
ExecStartPost=/opt/k8s/bin/mk-docker-opts.sh -k DOCKER_NETWORK_OPTIONS -d /run/flannel/docker
Restart=on-failure

[Install]
WantedBy=multi-user.target
RequiredBy=docker.service
EOF
```
***PS:***
```
mk-docker-opts.sh 脚本将分配给 flanneld 的 Pod 子网网段信息写入 /run/flannel/docker 文件，后续 docker 启动时使用这个文件中的环境变量配置 docker0 网桥；
flanneld 使用系统缺省路由所在的接口与其它节点通信，对于有多个网络接口（如内网和公网）的节点，可以用 -iface 参数指定通信接口，如上面的 eth0 接口;
flanneld 运行时需要 root 权限；
```

#### 分发 flanneld systemd unit 文件到所有node节点
将 ***flanneld.service*** 文件放置到/etc/systemd/system/ 目录下
```
# cp flanneld.service /etc/systemd/system/
```

#### 启动 flanneld 服务
```bash
# systemctl daemon-reload && systemctl enable flanneld && systemctl restart flanneld
```

#### 验证 flanneld 服务启动
```
# systemctl status flanneld
```

#### 验证各节点能通过 Pod 网段互通
```
# ip addr show flannel.1|grep -w inet
输出：
inet 172.30.5.0/32 scope global flannel.1
inet 172.30.14.0/32 scope global flannel.1
# ping -c 1 172.30.5.0
# ping -c 1 172.30.14.0
如果能通，说明一切OK
```
