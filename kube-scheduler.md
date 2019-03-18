---
title: "部署 kube-scheduler"
date: 2019-03-16T16:24:30+08:00
draft: false
---
该集群包含 3 个节点，启动后将通过竞争选举机制产生一个 leader 节点，其它节点为阻塞状态。当 leader 节点不可用后，剩余节点将再次进行选举产生新的 leader 节点，从而保证服务的可用性。
为保证通信安全，本文档先生成 x509 证书和私钥，kube-scheduler 在如下两种情况下使用该证书：

- 与 kube-apiserver 的安全端口通信;
- 在安全端口(https，10251) 输出 prometheus 格式的 metrics；
------
#### 创建 kube-scheduler 证书和私钥
***创建证书签名请求：***
```
cat > kube-scheduler-csr.json <<EOF
{
    "CN": "system:kube-scheduler",
    "hosts": [
      "127.0.0.1",
      "192.168.10.232",
      "192.168.10.243",
      "192.168.10.242"
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
        "O": "system:kube-scheduler",
        "OU": "4Paradigm"
      }
    ]
}
EOF
```
***PS***
```
hosts 列表包含所有 kube-scheduler 节点 IP；
CN 为 system:kube-scheduler、O 为 system:kube-scheduler，kubernetes 内置的 ClusterRoleBindings system:kube-scheduler 将赋予 kube-scheduler 工作所需的权限。
```
***生成证书和私钥：***
```
cfssl gencert -ca=/etc/kubernetes/cert/ca.pem \
  -ca-key=/etc/kubernetes/cert/ca-key.pem \
  -config=/etc/kubernetes/cert/ca-config.json \
  -profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```
----
#### 创建和分发 kubeconfig 文件
kubeconfig 文件包含访问 apiserver 的所有信息，如 apiserver 地址、CA 证书和自身使用的证书；
```
export KUBE_APISERVER="https://192.168.10.232:6443"

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
  --client-certificate=kube-scheduler.pem \
  --client-key=kube-scheduler-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config set-context system:kube-scheduler \
  --cluster=kubernetes \
  --user=system:kube-scheduler \
  --kubeconfig=kube-scheduler.kubeconfig

kubectl config use-context system:kube-scheduler --kubeconfig=kube-scheduler.kubeconfig
cp kube-scheduler.kubeconfig /etc/kubernetes/
chown -R k8s /etc/kubernetes/
```
-----
#### 创建和分发 kube-scheduler systemd unit 文件
```
cat > kube-scheduler.service <<EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/opt/k8s/bin/kube-scheduler \\
  --address=127.0.0.1 \\
  --kubeconfig=/etc/kubernetes/kube-scheduler.kubeconfig \\
  --leader-elect=true \\
  --alsologtostderr=true \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes \\
  --v=2
Restart=on-failure
RestartSec=5
User=k8s

[Install]
WantedBy=multi-user.target
EOF
# cp kube-scheduler.service /etc/systemd/system/
```
***PS***
```
--address：在 127.0.0.1:10251 端口接收 http /metrics 请求；kube-scheduler 目前还不支持接收 https 请求；
--kubeconfig：指定 kubeconfig 文件路径，kube-scheduler 使用它连接和验证 kube-apiserver；
--leader-elect=true：集群运行模式，启用选举功能；被选为 leader 的节点负责处理工作，其它节点为阻塞状态；
User=k8s：使用 k8s 账户运行；
```
----
#### 启动 kube-scheduler 服务
```
# systemctl daemon-reload && systemctl enable kube-scheduler && systemctl restart kube-scheduler
# systemctl status kube-scheduler|grep Active
```

#### 查看当前的 leader
```
# kubectl get endpoints kube-scheduler --namespace=kube-system  -o yaml
apiVersion: v1
kind: Endpoints
metadata:
  annotations:
    control-plane.alpha.kubernetes.io/leader: '{"holderIdentity":"master01_7da08f64-47d4-11e9-b9be-000c29a5444f","leaseDurationSeconds":15,"acquireTime":"2019-03-16T10:15:58Z","renewTime":"2019-03-16T10:18:11Z","leaderTransitions":0}'
  creationTimestamp: 2019-03-16T10:15:58Z
  name: kube-scheduler
  namespace: kube-system
  resourceVersion: "1895"
  selfLink: /api/v1/namespaces/kube-system/endpoints/kube-scheduler
  uid: 7e3d1bad-47d4-11e9-b4e9-000c29a5444f
```

#### 查看当前的各master组件状态
```
# kubectl  get componentstatuses     
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"}
```
