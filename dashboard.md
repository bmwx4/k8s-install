# dashboard


#### 修改配置

修改 dashboard-controller.yaml配置文件，更新dashboard镜像地址，如果网络允许的情况下，可以不修改，默认的是 k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3 ：
```bash
$ pwd
/opt/k8s/kubernetes/cluster/addons/dashboard
$ cp dashboard-controller.yaml{,.orig}
$ diff dashboard-controller.yaml{,.orig}
33c33
<         image: siriuszg/kubernetes-dashboard-amd64:v1.8.3
---
>         image: k8s.gcr.io/kubernetes-dashboard-amd64:v1.8.3
```
------

修改service配置文件，指定端口类型为 NodePort，这样外界可以通过地址 nodeIP:nodePort 访问 dashboard；改完的 service 配置如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  labels:
    k8s-app: kubernetes-dashboard
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
spec:
  selector:
    k8s-app: kubernetes-dashboard
  ports:
  - port: 443
    targetPort: 8443
  type: NodePort
```

#### 安装
进入到 dashboard 配置目录:
```bash
[root@master01 dashboard]# ll *.yaml
-rw-rw-r-- 1 53913 53913  264 Sep 26  2018 dashboard-configmap.yaml
-rw-rw-r-- 1 53913 53913 1821 Sep 26  2018 dashboard-controller.yaml
-rw-rw-r-- 1 53913 53913 1353 Sep 26  2018 dashboard-rbac.yaml
-rw-rw-r-- 1 53913 53913  551 Sep 26  2018 dashboard-secret.yaml
-rw-rw-r-- 1 53913 53913  339 Mar 31 08:22 dashboard-service.yaml
[root@master01 dashboard]# kubectl create -f  .
configmap/kubernetes-dashboard-settings created
serviceaccount/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-key-holder created
service/kubernetes-dashboard created
```

#### 查看分配的 NodePort
```bash
[root@master01 dashboard]# kubectl get deployment kubernetes-dashboard  -n kube-system
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kubernetes-dashboard   1         1         1            0           50s
[root@master01 dashboard]# kubectl --namespace kube-system get pods -o wide
NAME                                    READY   STATUS    RESTARTS   AGE     IP             NODE             NOMINATED NODE
coredns-779ffd89bd-vjlj8                1/1     Running   0          8h      172.30.14.11   192.168.10.242   <none>
heapster-684777c4cb-rpd6b               1/1     Running   0          8h      172.30.14.6    192.168.10.242   <none>
kubernetes-dashboard-659798bd99-nktfk   1/1     Running   0          4m53s   172.30.14.17   192.168.10.242   <none>
[root@master01 dashboard]# kubectl get services kubernetes-dashboard -n kube-system
NAME                   TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   NodePort   10.254.240.63   <none>        443:46953/TCP   5m55s
```

**NodePort 46953 映射到 dashboard pod 443 端口；**

dashboard 的 --authentication-mode 支持 token、basic，默认为 token。如果使用 basic，则 kube-apiserver 必须配置 '--authorization-mode=ABAC' 和 '--basic-auth-file' 参数。

![dashboard-login](/images/dashboard-login.png)

#### 创建登录 Dashboard 的 token 和 kubeconfig 配置文件
**创建登录 token**

```bash
kubectl create sa dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
ADMIN_SECRET=$(kubectl get secrets -n kube-system | grep dashboard-admin | awk '{print $1}')
DASHBOARD_LOGIN_TOKEN=$(kubectl describe secret -n kube-system ${ADMIN_SECRET} | grep -E '^token' | awk '{print $2}')
echo ${DASHBOARD_LOGIN_TOKEN}
```

**创建使用 token 的 KubeConfig 文件**
# 设置集群API地址
```bash
export KUBE_APISERVER="https://192.168.10.232:6443"
```
# 设置集群参数
```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/cert/ca.pem \
  --embed-certs=true \
  --server=${KUBE_APISERVER} \
  --kubeconfig=dashboard.kubeconfig
```
# 设置客户端认证参数，使用上面创建的 Token
```bash
kubectl config set-credentials dashboard_user \
  --token=${DASHBOARD_LOGIN_TOKEN} \
  --kubeconfig=dashboard.kubeconfig
```

# 设置上下文参数
```bash
kubectl config set-context default \
  --cluster=kubernetes \
  --user=dashboard_user \
  --kubeconfig=dashboard.kubeconfig
```
# 设置默认上下文
```bash
kubectl config use-context default --kubeconfig=dashboard.kubeconfig
```

用生成的 dashboard.kubeconfig 登录 Dashboard。
![dashboard](/images/dashboard0.png)

#### 通过 kube-apiserver 访问 dashboard
获取集群服务地址列表：
```bash
[root@master01 cert]# kubectl cluster-info  
Kubernetes master is running at https://192.168.10.232:6443
Heapster is running at https://192.168.10.232:6443/api/v1/namespaces/kube-system/services/heapster/proxy
CoreDNS is running at https://192.168.10.232:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
kubernetes-dashboard is running at https://192.168.10.232:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy
```
![dashboard](/images/apiserver-dashboard-login.png)
访问受限，参考：[A.浏览器访问kube-apiserver安全端口](kube-apiserver-sec-port.md)

---------

#### 常见错误
##### kubernetes dashboard 无法登录提示Not enough data to create auth info structure.
```
通过配置文件登录K8S控制台时报如下错误：
"Not enough data to create auth info structure.""
```
![dashboard-error1](/images/dashboard-error1.png)

解决办法：执行以下命令生成可用令牌：
```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```
将生成的token复制到kubeconfig 文件中，如下图所示：
![dashboard-error1](/images/dashboard-error2.png)
使用新的配置文件即可登录成功。
