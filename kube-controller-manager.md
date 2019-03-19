---
title: "部署 kube-controller-manager"
date: 2019-03-16T17:11:22+08:00
draft: false
---
本文档介绍部署 kube-controller-manager 集群的步骤。
该集群包含 3 个节点，启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。
为保证通信安全，本文档先生成 x509 证书和私钥，kube-controller-manager 在如下两种情况下使用该证书：  

- 与 kube-apiserver 的安全端口通信时;
- 在安全端口(https，10252) 输出 prometheus 格式的 metrics；

#### 创建 kube-controller-manager 证书和私钥
***创建证书签名请求：***

```
cat > kube-controller-manager-csr.json <<EOF
{
    "CN": "system:kube-controller-manager",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "hosts": [
      "127.0.0.1",
      "192.168.10.232",
      "192.168.10.242",
      "192.168.10.243"
    ],
    "names": [
      {
        "C": "CN",
        "ST": "BeiJing",
        "L": "BeiJing",
        "O": "system:kube-controller-manager",
        "OU": "4Paradigm"
      }
    ]
}
EOF
```
***PS***
```
hosts 列表包含所有 kube-controller-manager 节点 IP；
CN 为 system:kube-controller-manager、O 为 system:kube-controller-manager，kubernetes 内置的 ClusterRoleBindings system:kube-controller-manager 赋予 kube-controller-manager 工作所需的权限。
```
***生成证书和私钥：***
```
# cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
  -ca-key=/etc/kubernetes/cert/ca-key.pem \
  -config=/etc/kubernetes/cert/ca-config.json \
  -profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
# cp kube-controller-manager*.pem /etc/kubernetes/cert/ && chown -R k8s /etc/kubernetes/cert/
```
----
#### 创建和分发 kube-controller-manager的 kubeconfig 文件
```
export KUBE_APISERVER="https://192.168.10.232:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
  --client-certificate=kube-controller-manager.pem \
  --client-key=kube-controller-manager-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config set-context system:kube-controller-manager \
  --cluster=kubernetes \
  --user=system:kube-controller-manager \
  --kubeconfig=kube-controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager --kubeconfig=kube-controller-manager.kubeconfig
cp kube-controller-manager.kubeconfig /etc/kubernetes/ && chown -R k8s /etc/kubernetes/kube-controller-manager.kubeconfig
```

#### 创建和分发 kube-controller-manager systemd unit 文件
```
export SERVICE_CIDR="10.254.0.0/16"
cat > kube-controller-manager.service <<EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/k8s/bin/kube-controller-manager \\
  --port=10252 \\
  --secure-port=0 \\
  --bind-address=127.0.0.1 \\
  --kubeconfig=/etc/kubernetes/kube-controller-manager.kubeconfig \\
  --service-cluster-ip-range=${SERVICE_CIDR} \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/etc/kubernetes/cert/ca.pem \\
  --cluster-signing-key-file=/etc/kubernetes/cert/ca-key.pem \\
  --experimental-cluster-signing-duration=8760h \\
  --root-ca-file=/etc/kubernetes/cert/ca.pem \\
  --service-account-private-key-file=/etc/kubernetes/cert/ca-key.pem \\
  --leader-elect=true \\
  --feature-gates=RotateKubeletServerCertificate=true \\
  --controllers=*,bootstrapsigner,tokencleaner \\
  --horizontal-pod-autoscaler-use-rest-clients=true \\
  --horizontal-pod-autoscaler-sync-period=10s \\
  --tls-cert-file=/etc/kubernetes/cert/kube-controller-manager.pem \\
  --tls-private-key-file=/etc/kubernetes/cert/kube-controller-manager-key.pem \\
  --use-service-account-credentials=true \\
  --alsologtostderr=true \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes \\
  --v=2
Restart=on
Restart=on-failure
RestartSec=5
User=k8s

[Install]
WantedBy=multi-user.target
EOF
cp kube-controller-manager.service /etc/systemd/system/
```
***PS：***
```
--port=0：关闭监听 http /metrics 的请求，同时 --address 参数无效，--bind-address 参数有效；
--secure-port=10252、--bind-address=0.0.0.0: 在所有网络接口监听 10252 端口的 https /metrics 请求；
--kubeconfig：指定 kubeconfig 文件路径，kube-controller-manager 使用它连接和验证 kube-apiserver；
--cluster-signing-*-file：签名 TLS Bootstrap 创建的证书；
--experimental-cluster-signing-duration：指定 TLS Bootstrap 证书的有效期；
--root-ca-file：放置到容器 ServiceAccount 中的 CA 证书，用来对 kube-apiserver 的证书进行校验；
--service-account-private-key-file：签名 ServiceAccount 中 Token 的私钥文件，必须和 kube-apiserver 的 --service-account-key-file 指定的公钥文件配对使用；
--service-cluster-ip-range ：指定 Service Cluster IP 网段，必须和 kube-apiserver 中的同名参数一致；
--leader-elect=true：集群运行模式，启用选举功能；被选为 leader 的节点负责处理工作，其它节点为阻塞状态；
--feature-gates=RotateKubeletServerCertificate=true：开启 kublet server 证书的自动更新特性；
--controllers=*,bootstrapsigner,tokencleaner：启用的控制器列表，tokencleaner 用于自动清理过期的 Bootstrap token；
--horizontal-pod-autoscaler-*：custom metrics 相关参数，支持 autoscaling/v2alpha1；
--tls-cert-file、--tls-private-key-file：使用 https 输出 metrics 时使用的 Server 证书和秘钥；
--use-service-account-credentials=true:
User=k8s：使用 k8s 账户运行；
```
-----
#### 启动 kube-controller-manager 服务
```
# systemctl daemon-reload && systemctl enable kube-controller-manager && systemctl restart kube-controller-manager
```

------
#### 查看当前的controller manager leader
```
# kubectl get endpoints kube-controller-manager --namespace=kube-system  -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master01_998ac0d0-47cf-11e9-b05f-000c29a5444f","leaseDurationSeconds":15,"acquireTime":"2019-03-16T09:40:57Z","renewTime":"2019-03-16T09:41:55Z","leaderTransitions":0}'
  creationTimestamp: 2019-03-16T09:40:57Z
  name: kube-controller-manager
  namespace: kube-system
  resourceVersion: "541"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-controller-manager
  uid: 998c601b-47cf-11e9-b4e9-000c29a5444f
```
