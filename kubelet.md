---
title: "部署 kubelet 组件"
date: 2019-03-16T18:42:34+08:00
draft: false
---

kublet 运行在每个 worker 节点上，接收 kube-apiserver 发送的请求，管理 Pod 容器，执行交互式命令，如 exec、run、logs 等。
kublet 启动时自动向 kube-apiserver 注册节点信息,（内置的 cadvisor 统计和监控节点的资源使用情况）。
为确保安全，本文档只开启接收 https 请求的安全端口，对请求进行认证和授权，拒绝未授权的访问(如 apiserver、heapster)。

#### 下载和分发 kubelet 二进制文件  
参考 部署master节点.md

#### 创建 kubelet bootstrap kubeconfig 文件  
省略
```
export node_name="node01"
# 创建 token
export BOOTSTRAP_TOKEN=$(kubeadm token create \
      --description kubelet-bootstrap-token \
      --groups system:bootstrappers:${node_name} \
      --kubeconfig ~/.kube/config)
# 设置集群参数
    kubectl config set-cluster kubernetes \
      --certificate-authority=/etc/kubernetes/cert/ca.pem \
      --embed-certs=true \
      --server=${KUBE_APISERVER} \
      --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

# 设置客户端认证参数
    kubectl config set-credentials kubelet-bootstrap \
      --token=${BOOTSTRAP_TOKEN} \
      --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

# 设置上下文参数
    kubectl config set-context default \
      --cluster=kubernetes \
      --user=kubelet-bootstrap \
      --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig

# kubectl config use-context default --kubeconfig=kubelet-bootstrap-${node_name}.kubeconfig
```

证书中写入 Token 而非证书，证书后续由 controller-manager 创建。查看 kubeadm 为各节点创建的 token：
```
#  kubeadm token list --kubeconfig ~/.kube/config
```
***PS:***
```
创建的 token 有效期为 1 天，超期后将不能再被使用，且会被 kube-controller-manager 的 tokencleaner 清理(如果启用该 controller 的话)；
kube-apiserver 接收 kubelet 的 bootstrap token 后，将请求的 user 设置为 system:bootstrap:，group 设置为 system:bootstrappers；
```

----
#### 分发 bootstrap kubeconfig 文件到所有 worker 节点
```
scp kubelet-bootstrap-${node_name}.kubeconfig k8s@${node_name}:/etc/kubernetes/kubelet-bootstrap.kubeconfig
```

#### 创建和分发 kubelet config json 文件
```
export CLUSTER_DNS_DOMAIN="cluster.local."
export CLUSTER_DNS_SVC_IP="10.254.0.2"
cat > kubelet.config.json.template <<EOF
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "authentication": {
    "x509": {
      "clientCAFile": "/etc/kubernetes/cert/ca.pem"
    },
    "webhook": {
      "enabled": true,
      "cacheTTL": "2m0s"
    },
    "anonymous": {
      "enabled": false
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "address": "192.168.10.242",
  "port": 10250,
  "readOnlyPort": 0,
  "cgroupDriver": "cgroupfs",
  "hairpinMode": "promiscuous-bridge",
  "serializeImagePulls": false,
  "featureGates": {
    "RotateKubeletClientCertificate": true,
    "RotateKubeletServerCertificate": true
  },
  "clusterDomain": "${CLUSTER_DNS_DOMAIN}",
  "clusterDNS": ["${CLUSTER_DNS_SVC_IP}"]
}
EOF
# cp kubelet.config.json.template /etc/kubernetes/kubelet.config.json
```

#### 创建和分发 kubelet systemd unit 文件
***创建 kubelet systemd unit 文件模板：***
```
cat > kubelet.service.template <<EOF
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/opt/k8s/bin/kubelet \\
  --cert-dir=/etc/kubernetes/cert \\
  --bootstrap-kubeconfig=/etc/kubernetes/kubelet-bootstrap.kubeconfig \\
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \\
  --config=/etc/kubernetes/kubelet.config.json \\
  --hostname-override=192.168.10.242 \\
  --pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest \\
  --allow-privileged=true \\
  --alsologtostderr=true \\
  --logtostderr=false \\
  --log-dir=/var/log/kubernetes \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
cp kubelet.service.template /etc/systemd/system/kubelet.service
```

***PS***
```
如果设置了 --hostname-override 选项，则 kube-proxy 也需要设置该选项，否则会出现找不到 Node 的情况；
--bootstrap-kubeconfig：指向 bootstrap kubeconfig 文件，kubelet 使用该文件中的用户名和 token 向 kube-apiserver 发送 TLS Bootstrapping 请求；

K8S approve kubelet 的 csr 请求后，在 --cert-dir 目录创建证书和私钥文件，然后写入 --kubeconfig 文件；
```
----

#### Bootstrap Token Auth 和授予权限
kublet 启动时查找配置的 --kubeletconfig 文件是否存在，如果不存在则使用 --bootstrap-kubeconfig 向 kube-apiserver 发送证书签名请求 (CSR)。

kube-apiserver 收到 CSR 请求后，对其中的 Token 进行认证（事先使用 kubeadm 创建的 token），认证通过后将请求的 user 设置为 system:bootstrap:，group 设置为 system:bootstrappers，这一过程称为 Bootstrap Token Auth。
默认情况下，这个 user 和 group 没有创建 CSR 的权限，kubelet 启动失败，错误日志如下：

```
Mar 16 21:26:39 node01 kubelet[69360]: F0316 21:26:39.078403   69360 server.go:262] failed to run Kubelet: cannot create certificate signing request: certificatesigningrequests.certificates.k8s.io is forbidden: User "system:bootstrap:br9gy4" cannot create resource "certificatesigningrequests" in API group "certificates.k8s.io" at the cluster scope
Mar 16 21:26:39 node01 kubelet[69360]: goroutine 1 [running]:
Mar 16 21:26:39 node01 kubelet[69360]: k8s.io/kubernetes/vendor/github.com/golang/glog.stacks(0xc420988f00, 0xc420a9c000, 0x137, 0x233)
```
解决办法是：创建一个 clusterrolebinding，将 group system:bootstrappers 和 clusterrole system:node-bootstrapper 绑定：
```
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --group=system:bootstrappers
```

重启kubelet, kubelet 启动后使用 --bootstrap-kubeconfig 向 kube-apiserver 发送 CSR 请求，当这个 CSR 被 approve 后，kube-controller-manager 为 kubelet 创建 TLS 客户端证书、私钥和 --kubeletconfig 文件。

***PS：***  
kube-controller-manager 需要配置 --cluster-signing-cert-file 和 --cluster-signing-key-file 参数，才会为 TLS Bootstrap 创建证书和私钥。
```
[root@master01 cert]# kubectl get csr
NAME                                                   AGE     REQUESTOR                 CONDITION
node-csr-ofWszodlIzBa0s8KYBnVrKD6p47Y3aA00uOf75vIZMg   3m33s   system:bootstrap:br9gy4   Pending
[root@master01 cert]# kubectl  get node
No resources found.
```
----
#### approve kubelet CSR 请求
可以手动或自动 approve CSR 请求。推荐使用自动的方式，因为从 v1.8 版本开始，可以自动轮转approve csr 后生成的证书。

**手动 approve CSR 请求**
```
# kubectl  certificate  approve node-csr-ofWszodlIzBa0s8KYBnVrKD6p47Y3aA00uOf75vIZMg
# kubectl describe  csr node-csr-ofWszodlIzBa0s8KYBnVrKD6p47Y3aA00uOf75vIZMg
```
----
#### 自动 approve CSR 请求
创建三个 ClusterRoleBinding，分别用于自动 approve client、renew client、renew server 证书：
```
cat > csr-crb.yaml <<EOF
 # Approve all CSRs for the group "system:bootstrappers"
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: auto-approve-csrs-for-group
 subjects:
 - kind: Group
   name: system:bootstrappers
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
   apiGroup: rbac.authorization.k8s.io
---
 # To let a node of the group "system:nodes" renew its own credentials
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: node-client-cert-renewal
 subjects:
 - kind: Group
   name: system:nodes
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
   apiGroup: rbac.authorization.k8s.io
---
# A ClusterRole which instructs the CSR approver to approve a node requesting a
# serving cert matching its client cert.
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: approve-node-server-renewal-csr
rules:
- apiGroups: ["certificates.k8s.io"]
  resources: ["certificatesigningrequests/selfnodeserver"]
  verbs: ["create"]
---
 # To let a node of the group "system:nodes" renew its own server credentials
 kind: ClusterRoleBinding
 apiVersion: rbac.authorization.k8s.io/v1
 metadata:
   name: node-server-cert-renewal
 subjects:
 - kind: Group
   name: system:nodes
   apiGroup: rbac.authorization.k8s.io
 roleRef:
   kind: ClusterRole
   name: approve-node-server-renewal-csr
   apiGroup: rbac.authorization.k8s.io
EOF

# kubectl apply -f csr-crb.yaml
```
授权之后，kube-controller-manager 为各 node 生成了 kubeconfig 文件和公私钥；

----
#### kubelet 提供的 API 接口
```
# netstat -lnpt|grep kubelet
4194: cadvisor http 服务,但是自1.10之后，cadvisor从kubelet中移除来了，需要独立安装，或使用daemonset；
10248: healthz http 服务；
10250: https API 服务；注意：未开启只读端口 10255；
```
安装完之后，验证集群是否可以创建POD和Deployment
```
kubectl  run httpd --image=httpd --replicas=1
kubectl  run httpod --image=httpd --restart=Never --image-pull-policy=IfNotPresent
[root@master01 ~]# kubectl  get pod
NAME                    READY   STATUS    RESTARTS   AGE
httpd-7db5849b8-9bhxv   1/1     Running   0          6m42s
httpod                  1/1     Running   0          115s
```
