# ---
title: "二进制快速部署"
date: 2019-03-16T18:42:34+08:00
draft: false
---

#### 准备根证书

```bash
cat > kubelet-csr.json <<EOF
{
    "CN": "kubernetes",
    "hosts": [""],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "CN",
            "ST": "BJ",
            "L": "BJ",
            "O": "system:masters",
            "OU": "apiserver client"
        }
    ]
}
EOF
```

生成kubelet 证书
```bash
cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
  -ca-key=/etc/kubernetes/cert/ca-key.pem  \
  -config=/etc/kubernetes/cert/ca-config.json  \
  -profile=kubernetes  kubelet-csr.json | cfssljson -bare kubelet
```

#### 创建 kubelet kubeconfig 文件  
省略
```
export KUBE_APISERVER="https://192.168.76.233:6443"
# 设置集群参数
kubectl config set-cluster kubernetes \
  --certificate-authority=root/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=client.kubeconfig

# 设置客户端认证参数
kubectl config set-credentials client \
  --client-certificate=kubelet.pem  \
  --client-key=kubelet-key.pem \
  --embed-certs=true \
  --kubeconfig=client.kubeconfig

# 设置上下文参数
kubectl config set-context default \
  --cluster=kubernetes \
  --user=client \
  --kubeconfig=client.kubeconfig

# 设置默认上下文
kubectl config use-context default --kubeconfig=kubelet.kubeconfig

cp client.kubeconfig kubelet.kubeconfig
```

这样一个通用 kubelet 的 kubeconfig文件就生成了；
