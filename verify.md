---
title: "验证集群功能."
date: 2019-03-16T23:22:26+08:00
draft: false
---

本文档使用 daemonset 验证 master 和 worker 节点是否工作正常。

#### 检查节点状态
```
[root@master01 ~]# kubectl  get node
NAME             STATUS   ROLES    AGE    VERSION
192.168.10.242   Ready    <none>   114m   v1.12.0-rc.2
```
----
#### 创建测试文件
```
cat > nginx-dp.yml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - name: http
    port: 80
    targetPort: 80
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
  labels:
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: my-nginx
        image: nginx:latest
        imagePullPolicy :IfNotPresent
        ports:
        - containerPort: 80
EOF
```
***执行定义文件***
```
# kubectl  create -f nginx-dp.yml                 
service/nginx created
deployment.extensions/nginx created
```

#### 检查各 Node 上的 Pod IP 连通性
```
# kubectl  get svc          
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.254.0.1      <none>        443/TCP        6h39m
nginx        NodePort    10.254.76.145   <none>        80:37172/TCP   4m45s
# curl node01:37172

# kubectl  get pod -o wide
NAME                    READY   STATUS    RESTARTS   AGE    IP            NODE             NOMINATED NODE
httpd-7db5849b8-9bhxv   1/1     Running   0          109m   172.30.14.2   192.168.10.242   <none>
httpod                  1/1     Running   0          105m   172.30.14.4   192.168.10.242   <none>
nginx-6b5f8bf67-bs8n4   1/1     Running   0          13m    172.30.14.3   192.168.10.242   <none>

# kubectl  run busybox --image=busybox --image-pull-policy=IfNotPresent --restart=Never --command -- ping 172.30.14.3
# kubectl  logs busybox
PING 172.30.14.3 (172.30.14.3): 56 data bytes
64 bytes from 172.30.14.3: seq=0 ttl=64 time=0.192 ms
64 bytes from 172.30.14.3: seq=1 ttl=64 time=0.171 ms
64 bytes from 172.30.14.3: seq=2 ttl=64 time=0.166 ms
64 bytes from 172.30.14.3: seq=3 ttl=64 time=0.169 ms

```
