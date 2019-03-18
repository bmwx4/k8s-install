---
title: "创建 CA 证书和秘钥"
date: 2019-03-16T11:33:22+08:00
draft: false
---
### **CA**  
CA (Certificate Authority) 是自签名的根证书，用来签名后续创建的其它证书。本文档使用 CloudFlare 的 PKI 工具集 cfssl 创建所有证书。
为确保安全，kubernetes 系统各组件需要使用 x509 证书对通信进行加密和认证。

-----
#### 安装 cfssl 工具集
```
在master01上执行:
mkdir -p /opt/k8s/cert &&  chown -R k8s /opt/k8s && cd /opt/k8s
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
mv cfssl_linux-amd64 /opt/k8s/bin/cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssljson_linux-amd64 /opt/k8s/bin/cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 /opt/k8s/bin/cfssl-certinfo
chmod +x /opt/k8s/bin/*
export PATH=/opt/k8s/bin:$PATH
```
安装完效果如下：
```
[root@master01 bin]# ll /opt/k8s/bin/cfssl*
-rwxr-xr-x. 1 k8s root 10376657 Mar 30  2016 /opt/k8s/bin/cfssl
-rwxr-xr-x. 1 k8s root  6595195 Mar 30  2016 /opt/k8s/bin/cfssl-certinfo
-rwxr-xr-x. 1 k8s root  2277873 Mar 30  2016 /opt/k8s/bin/cfssljson
```
-----
#### 创建根证书 (CA)  
CA 证书是集群所有节点共享的，***只需要创建一个 CA 证书***，后续创建的所有证书都由它签名。  
CA 证书的配置文件，用于配置根证书的使用场景 (profile) 和具体参数 (usage，过期时间、服务端认证、客户端认证、加密等)，后续在签名其它证书时需要指定特定场景。  
``` json
cat > ca-config.json <<EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
EOF
```

***PS:***
```
signing：表示该证书可用于签名其它证书，生成的 ca.pem 证书中 CA=TRUE；  
server auth：表示 client 可以用该该证书对 server 提供的证书进行验证；  
client auth：表示 server 可以用该该证书对 client 提供的证书进行验证；
```
-----
#### 创建证书签名请求文件
````json
cat > ca-csr.json <<EOF
{
  "CN": "kubernetes",
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
````
***PS:***
```
CN：Common Name，kube-apiserver 从证书中提取该字段作为请求的用户名 (User Name)，浏览器使用该字段验证网站是否合法；
O：Organization，kube-apiserver 从证书中提取该字段作为请求用户所属的组 (Group)；
kube-apiserver 将提取的 User、Group 作为 RBAC 授权的用户标识；
```
-----
#### 分发证书文件
将生成的 CA 证书、秘钥文件、配置文件拷贝到所有master节点和node节点的 /etc/kubernetes/cert 目录下,并保证k8s用户有读写 /etc/kubernetes 目录及其子目录文件的权限：
```
# cp ca*.pem ca-config.json /etc/kubernetes/cert/
# chown -R k8s /etc/kubernetes
```
-----
#### 下载和分发 kubectl 二进制文件
  - [v1.12.0-rc2](https://v1-12.docs.kubernetes.io/docs/setup/release/notes/)  

***下载之后分发二进制文件：***
```
# tar xvf k8s.tgz
k8s/
k8s/kubectl
k8s/kubelet
k8s/kube-proxy
k8s/kube-apiserver
k8s/kube-controller-manager
k8s/kube-scheduler
# cp k8s/* /opt/k8s/bin/
```

-----
#### 创建 admin 证书和私钥
kubectl 与 apiserver https 安全端口通信，apiserver 对提供的证书进行认证和授权。  
kubectl 作为集群的管理工具，需要被授予最高权限。这里创建具有最高权限的 admin 证书。  
创建证书签名请求：
```json
cat > admin-csr.json <<EOF
{
  "CN": "admin",
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
      "O": "system:masters",
      "OU": "4Paradigm"
    }
  ]
}
EOF
```
***PS:***
```
O 为 system:masters，kube-apiserver 收到该证书后将请求的 Group 设置为 system:masters；
预定义的 ClusterRoleBinding cluster-admin 将 Group system:masters 与 Role cluster-admin 绑定，该 Role 授予所有 API的权限；
该证书只会被 kubectl 当做 client 证书使用，所以 hosts 字段为空；
```
生成证书和私钥：
```
# cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
  -ca-key=/etc/kubernetes/cert/ca-key.pem \
  -config=/etc/kubernetes/cert/ca-config.json \
  -profile=kubernetes admin-csr.json | cfssljson -bare admin
# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem
```
-----
#### 创建 kubeconfig 文件
kubeconfig 为 kubectl 的配置文件，包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书；  
```
master01运行

# kube-apiserver 的 VIP（HA 组件 keepalived 发布的 IP）
export MASTER_VIP=192.168.10.232
# kube-apiserver VIP 地址（HA 组件 haproxy 监听 6443 端口）
export KUBE_APISERVER="https://${MASTER_VIP}:6443"
IP地址为master01的IP，如果启动vip,就使用VIP，这里我们直连master01
```

***设置集群参数***
```

# kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kubectl.kubeconfig
```
验证结果,比如：
```bash
# cat kubectl.kubeconfig
apiVersion: v1
clusters:
- cluster:
    .....
    server: 192.168.10.232
  name: kubernetes
contexts: []
current-context: ""
kind: Config
preferences: {}
users: []
```

***设置客户端认证参数***
```
# kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem \
  --embed-certs=true \
  --kubeconfig=kubectl.kubeconfig
```
再次验证，看是否users已被设置，比如：
```
users:
- name: admin
........
```

***设置上下文参数***
```
# kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin \
  --kubeconfig=kubectl.kubeconfig

# 设置默认上下文
# kubectl config use-context kubernetes --kubeconfig=kubectl.kubeconfig
```

***PS:***
```
  --certificate-authority：验证 kube-apiserver 证书的根证书；
  --client-certificate、--client-key：刚生成的 admin 证书和私钥，连接 kube-apiserver 时使用；
  --embed-certs=true：将 ca.pem 和 admin.pem 证书内容嵌入到生成的 kubectl.kubeconfig 文件中(不加时，写入的是证书文件路径)；
```
-----
#### 分发 kubeconfig 文件
将生成的 ***kubectl.kubeconfig*** 文件分发到所有使用 kubectl 命令的节点上，比如我们的master01，并改名放置 ~/.kube/config
```
[root@master01 cert]# mkdir -p ~/.kube
[root@master01 cert]# cp kubectl.kubeconfig ~/.kube/config
```
