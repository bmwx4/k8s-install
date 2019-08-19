---
title: "部署 etcd 集群"
date: 2019-03-16T14:35:12+08:00
draft: false
---

etcd 是基于 Raft 的分布式 key-value 存储系统，由 CoreOS 开发，常用于服务发现、共享配置以及并发控制（如 leader 选举、分布式锁等）。kubernetes 使用 etcd 存储所有运行数据。

本文档介绍部署一个单实例 etcd 集群的步骤：  
下载和分发 etcd 二进制文件；
创建 etcd 集群各节点的 x509 证书，用于加密客户端(如 etcdctl) 与 etcd 集群、etcd 集群之间的数据流；  
创建 etcd 的 systemd unit 文件，配置服务参数；  
检查集群工作状态；  

etcd 集群各节点的名称和 IP 如下：
```
master01: 192.168.10.232
```
----
#### 下载和分发 etcd 二进制文件
```
# wget https://github.com/coreos/etcd/releases/download/v3.3.7/etcd-v3.3.7-linux-amd64.tar.gz
# tar -xvf etcd-v3.3.7-linux-amd64.tar.gz
# cp etcd-v3.3.7-linux-amd64/etcd* /opt/k8s/bin/
```

----
#### 创建 etcd 证书和私钥
***创建证书签名请求：***
```
cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "192.168.10.232",
    "192.168.10.242",
    "192.168.10.243"
  ],
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
***PS****
```
hosts 字段指定授权使用该证书的 etcd 节点 IP 或域名列表，这里将 master01,node01,node02 IP 都列在其中,用于以后扩展etcd集群；
```
生成证书和私钥：
```
cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
    -ca-key=/etc/kubernetes/cert/ca-key.pem \
    -config=/etc/kubernetes/cert/ca-config.json \
    -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem
```
分发生成的证书和私钥到各 etcd 节点：
```
如果使用etcd集群，这个动作需要在每个节点都运行，这里只是作为演示，使用单实例的方式进行安装
# mkdir -p /etc/etcd/cert/
# cp etcd*.pem /etc/etcd/cert/
# chown -R k8s /etc/etcd/cert/
```
----
#### 创建 etcd 的 systemd unit 模板文件
```
export ETCD_NODES="master01=https://192.168.10.232:2380"

cat > etcd.service.template <<EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
User=k8s
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/opt/k8s/bin/etcd \\
  --data-dir=/var/lib/etcd \\
  --name=##NODE_NAME## \\
  --cert-file=/etc/etcd/cert/etcd.pem \\
  --key-file=/etc/etcd/cert/etcd-key.pem \\
  --trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-cert-file=/etc/etcd/cert/etcd.pem \\
  --peer-key-file=/etc/etcd/cert/etcd-key.pem \\
  --peer-trusted-ca-file=/etc/kubernetes/cert/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --listen-peer-urls=https://##NODE_IP##:2380 \\
  --initial-advertise-peer-urls=https://##NODE_IP##:2380 \\
  --listen-client-urls=https://##NODE_IP##:2379,http://127.0.0.1:2379 \\
  --advertise-client-urls=https://##NODE_IP##:2379 \\
  --initial-cluster-token=etcd-cluster-0 \\
  --initial-cluster=${ETCD_NODES} \\
  --initial-cluster-state=new
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
替换真是的NODE_NAME 和NODE_IP
# export NODE_NAME=master01
# export NODE_IP=192.168.10.232
# sed -e "s/##NODE_NAME##/${NODE_NAME}/" -e "s/##NODE_IP##/${NODE_IP}/" etcd.service.template > etcd.service
# cp etcd.service /usr/lib/systemd/system/
```

***PS:***
```
User：指定以 k8s 账户运行；
WorkingDirectory、--data-dir：指定工作目录和数据目录为 /var/lib/etcd，需在启动服务前创建这个目录；
--name：指定节点名称，当 --initial-cluster-state 值为 new 时，--name 的参数值必须位于 --initial-cluster 列表中；
--cert-file、--key-file：etcd server 与 client 通信时使用的证书和私钥；
--trusted-ca-file：签名 client 证书的 CA 证书，用于验证 client 证书；
--peer-cert-file、--peer-key-file：etcd 与 peer 通信使用的证书和私钥；
--peer-trusted-ca-file：签名 peer 证书的 CA 证书，用于验证 peer 证书；
```

-----
#### 启动 etcd 服务
```
确保data-dir 是否存在：
# mkdir -p /var/lib/etcd && chown -R k8s /var/lib/etcd
启动etcd:
# systemctl daemon-reload && systemctl enable etcd && systemctl restart etcd
```
#### 验证etcd 服务是否正常
```
export node_ip=192.168.10.232
ETCDCTL_API=3 etcdctl \
    --endpoints=https://${node_ip}:2379 \
    --cacert=/etc/kubernetes/cert/ca.pem \
    --cert=/etc/etcd/cert/etcd.pem \
    --key=/etc/etcd/cert/etcd-key.pem endpoint health

https://192.168.10.232:2379 is healthy: successfully committed proposal: took = 1.230902ms
```
输出均为 healthy 时表示集群服务正常。

#### etcd 线上服务器推荐机型
因为高性能的磁盘是保证etcd性能和稳定性的关键因素。 但是该如何进行硬件配置选择呢？官方给了一份参考:
[Hardware recommendations](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/hardware.md)
