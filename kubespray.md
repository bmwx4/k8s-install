## KubeSpray 安装 kubernetes

#### 安装系统环境
| 主机名 | ip | 操作系统 | 角色 |
| ------ | ------ | ------ | ------ |
| node1 | 10.178.27.119 | CentOS Linux 7 (Core) | master,etcd,node |
| node2 | 10.178.24.199 | CentOS Linux 7 (Core) | node |
| ansible | 172.24.24.7 | CentOS Linux 7 (Core) | ansible client|
ansible client 是我的一台虚拟机，因为环境怎么倒腾都不怕，呵呵;

#### 下载 KubeSpray
拉取github上的代码,并切换到最新的分支,不建议直接使用master分支:
```
# git clone https://github.com/kubernetes-incubator/kubespray.git
# git checkout v2.8.3
或者
# git checkout v2.11.0
```

#### 安装 KubeSpray 所依赖的包,都包括哪些呢:
```bash
# cat requirements.txt
ansible>=2.5.0,!=2.7.0,<2.8.0
jinja2>=2.9.6
netaddr
pbr>=1.6
hvac
```
在我做实验的时候，我先装了 ansible，而且是 2.8.3 版本的，在执行play-book的时候遇到了问题:[Running ansible-playbook errors #15](https://github.com/zonca/zonca.github.io-source/issues/15), 最后，按照社区推荐的方式，修改了requirements.txt文件中ansible的版本依赖问题，加了一个 <2.8.0版本限制，然后重新运行下面的命令；所以你不需要事先单独安装 ansible 的，把pip 安装好即可，然后按照下面的方式进行依赖安装:
```bash
# cd kubespray
# pip install -r requirements.txt
```
如果使用 v2.11.0 的版本则不需要修改，直接安装依赖即可.

#### 主机配置
生成自己的配置，然后修改:
```bash
# cp -ar inventory/sample inventory/k8s/
```
修改inventory/k8s/ 目录下的 host.ini文件,v2.11.0 的版本叫 inventory.ini :
```
# cat inventory/k8s/
group_vars/ hosts.ini
[root@ks-allinone kubespray]# cat inventory/k8s/hosts.ini
# ## Configure 'ip' variable to bind kubernetes services on a
# ## different ip than the default iface
# ## We should set etcd_member_name for etcd cluster. The node that is not a etcd member do not need to set the value, or can set the empty string value.
[all]
node1 ansible_host=10.178.27.119 ip=10.178.27.119 ansible_user=root ansible_ssh_port=8000 ansible_python_interpreter=/usr/bin/python2.7
node2 ansible_host=10.178.24.199 ip=10.178.24.199 ansible_user=root ansible_ssh_port=8000 ansible_python_interpreter=/usr/bin/python2.7
# node1 ansible_host=95.54.0.12  # ip=10.3.0.1 etcd_member_name=etcd1
# node2 ansible_host=95.54.0.13  # ip=10.3.0.2 etcd_member_name=etcd2
# node3 ansible_host=95.54.0.14  # ip=10.3.0.3 etcd_member_name=etcd3
# node4 ansible_host=95.54.0.15  # ip=10.3.0.4 etcd_member_name=etcd4
# node5 ansible_host=95.54.0.16  # ip=10.3.0.5 etcd_member_name=etcd5
# node6 ansible_host=95.54.0.17  # ip=10.3.0.6 etcd_member_name=etcd6

# ## configure a bastion host if your nodes are not directly reachable
# bastion ansible_host=x.x.x.x ansible_user=some_user

[kube-master]
node1

[etcd]
node1

[kube-node]
node2

[k8s-cluster:children]
kube-master
kube-node

[vault]
node1
```
确定主机信息之后，下一步要做ssh信任机制， 把 ansible client 上的公钥加入到各个主机上的 ~/.ssh/authorized_keys 文件中；
```bash
# ssh-keygen
# ssh-copy-id -i ~/.ssh/id_rsa.pub root@{$target-host}
```

#### 替换镜像地址
这一步没啥可说的， 我直接使用微软云提供的的镜像仓库 proxy 了，速度上可以接受，所以，我们替换一下镜像地址,主要需要修改一下两个文件:
```bash
./inventory/k8s/group_vars/k8s-cluster/k8s-cluster.yml
./roles/download/defaults/main.yml
```
修改后的效果, 也可以按照文后的国内镜像源进行相应的替换:
```
# grep -r 'azk8s.cn' ./
./inventory/k8s/group_vars/k8s-cluster/k8s-cluster.yml:kube_image_repo: "gcr.azk8s.cn/google-containers"
< kube_image_repo: "gcr.io/google-containers"
---
> kube_image_repo: "gcr.azk8s.cn/google-containers"
217c217
< etcd_image_repo: "quay.io/coreos/etcd"
---
> etcd_image_repo: "quay.azk8s.cn/coreos/etcd"
219c219
< flannel_image_repo: "quay.io/coreos/flannel"
---
> flannel_image_repo: "quay.azk8s.cn/coreos/flannel"
221c221
< flannel_cni_image_repo: "quay.io/coreos/flannel-cni"
---
> flannel_cni_image_repo: "quay.azk8s.cn/coreos/flannel-cni"
223c223
< calico_node_image_repo: "docker.io/calico/node"
---
> calico_node_image_repo: "dockerhub.azk8s.cn/calico/node"
225c225
< calico_cni_image_repo: "docker.io/calico/cni"
---
> calico_cni_image_repo: "dockerhub.azk8s.cn/calico/cni"
227c227
< calico_policy_image_repo: "docker.io/calico/kube-controllers"
---
> calico_policy_image_repo: "dockerhub.azk8s.cn/calico/kube-controllers"
229c229
< calico_rr_image_repo: "docker.io/calico/routereflector"
---
> calico_rr_image_repo: "dockerhub.azk8s.cn/calico/routereflector"
231c231
< calico_typha_image_repo: "docker.io/calico/typha"
---
> calico_typha_image_repo: "dockerhub.azk8s.cn/calico/typha"
233c233
< pod_infra_image_repo: "gcr.io/google_containers/pause-{{ image_arch }}"
---
> pod_infra_image_repo: "gcr.azk8s.cn/google_containers/pause-{{ image_arch }}"
235c235
< install_socat_image_repo: "docker.io/xueshanf/install-socat"
---
> install_socat_image_repo: "dockerhub.azk8s.cn/xueshanf/install-socat"
238c238
< netcheck_agent_image_repo: "quay.io/l23network/k8s-netchecker-agent"
---
> netcheck_agent_image_repo: "quay.azk8s.cn/l23network/k8s-netchecker-agent"
240c240
< netcheck_server_image_repo: "quay.io/l23network/k8s-netchecker-server"
---
> netcheck_server_image_repo: "quay.azk8s.cn/l23network/k8s-netchecker-server"
242c242
< weave_kube_image_repo: "docker.io/weaveworks/weave-kube"
---
> weave_kube_image_repo: "dockerhub.azk8s.cn/weaveworks/weave-kube"
244c244
< weave_npc_image_repo: "docker.io/weaveworks/weave-npc"
---
> weave_npc_image_repo: "dockerhub.azk8s.cn/weaveworks/weave-npc"
246c246
< contiv_image_repo: "docker.io/contiv/netplugin"
---
> contiv_image_repo: "dockerhub.azk8s.cn/contiv/netplugin"
248c248
< contiv_init_image_repo: "docker.io/contiv/netplugin-init"
---
> contiv_init_image_repo: "dockerhub.azk8s.cn/contiv/netplugin-init"
250c250
< contiv_auth_proxy_image_repo: "docker.io/contiv/auth_proxy"
---
> contiv_auth_proxy_image_repo: "dockerhub.azk8s.cn/contiv/auth_proxy"
252c252
< contiv_etcd_init_image_repo: "docker.io/ferest/etcd-initer"
---
> contiv_etcd_init_image_repo: "dockerhub.azk8s.cn/ferest/etcd-initer"
254c254
< contiv_ovs_image_repo: "docker.io/contiv/ovs"
---
> contiv_ovs_image_repo: "dockerhub.azk8s.cn/contiv/ovs"
256c256
< cilium_image_repo: "docker.io/cilium/cilium"
---
> cilium_image_repo: "dockerhub.azk8s.cn/cilium/cilium"
258c258
< cilium_init_image_repo: "docker.io/cilium/cilium-init"
---
> cilium_init_image_repo: "dockerhub.azk8s.cn/cilium/cilium-init"
260c260
< cilium_operator_image_repo: "docker.io/cilium/operator"
---
> cilium_operator_image_repo: "dockerhub.azk8s.cn/cilium/operator"
262,264c262,264
< kube_ovn_db_image_repo: "docker.io/kubeovn/kube-ovn-db"
< kube_ovn_node_image_repo: "docker.io/kubeovn/kube-ovn-node"
< kube_ovn_cni_image_repo: "docker.io/kubeovn/kube-ovn-cni"
---
> kube_ovn_db_image_repo: "dockerhub.azk8s.cn/kubeovn/kube-ovn-db"
> kube_ovn_node_image_repo: "dockerhub.azk8s.cn/kubeovn/kube-ovn-node"
> kube_ovn_cni_image_repo: "dockerhub.azk8s.cn/kubeovn/kube-ovn-cni"
270c270
< kube_router_image_repo: "docker.io/cloudnativelabs/kube-router"
---
> kube_router_image_repo: "dockerhub.azk8s.cn/cloudnativelabs/kube-router"
272c272
< multus_image_repo: "docker.io/nfvpe/multus"
---
> multus_image_repo: "dockerhub.azk8s.cn/nfvpe/multus"
274c274
< nginx_image_repo: docker.io/nginx
---
> nginx_image_repo: dockerhub.azk8s.cn/library/nginx
277c277
< haproxy_image_repo: docker.io/haproxy
---
> haproxy_image_repo: dockerhub.azk8s.cn/haproxy
281c281
< coredns_image_repo: "docker.io/coredns/coredns"
---
> coredns_image_repo: "dockerhub.azk8s.cn/coredns/coredns"
285c285
< nodelocaldns_image_repo: "k8s.gcr.io/k8s-dns-node-cache"
---
> nodelocaldns_image_repo: "gcr.azk8s.cn/google_containers/k8s-dns-node-cache"
289c289
< dnsautoscaler_image_repo: "k8s.gcr.io/cluster-proportional-autoscaler-{{ image_arch }}"
---
> dnsautoscaler_image_repo: "gcr.azk8s.cn/google_containers/cluster-proportional-autoscaler-{{ image_arch }}"
291c291
< test_image_repo: docker.io/busybox
---
> test_image_repo: dockerhub.azk8s.cn/busybox
293c293
< busybox_image_repo: docker.io/busybox
---
> busybox_image_repo: dockerhub.azk8s.cn/busybox
296c296
< helm_image_repo: "docker.io/lachlanevenson/k8s-helm"
---
> helm_image_repo: "dockerhub.azk8s.cn/lachlanevenson/k8s-helm"
298c298
< tiller_image_repo: "gcr.io/kubernetes-helm/tiller"
---
> tiller_image_repo: "gcr.azk8s.cn/kubernetes-helm/tiller"
301c301
< registry_image_repo: "docker.io/registry"
---
> registry_image_repo: "dockerhub.azk8s.cn/registry"
303c303
< registry_proxy_image_repo: "gcr.io/google_containers/kube-registry-proxy"
---
> registry_proxy_image_repo: "gcr.azk8s.cn/google_containers/kube-registry-proxy"
306c306
< metrics_server_image_repo: "gcr.io/google_containers/metrics-server-amd64"
---
> metrics_server_image_repo: "gcr.azk8s.cn/google_containers/metrics-server-amd64"
308c308
< local_volume_provisioner_image_repo: "quay.io/external_storage/local-volume-provisioner"
---
> local_volume_provisioner_image_repo: "quay.azk8s.cn/external_storage/local-volume-provisioner"
310c310
< cephfs_provisioner_image_repo: "quay.io/external_storage/cephfs-provisioner"
---
> cephfs_provisioner_image_repo: "quay.azk8s.cn/external_storage/cephfs-provisioner"
312c312
< rbd_provisioner_image_repo: "quay.io/external_storage/rbd-provisioner"
---
> rbd_provisioner_image_repo: "quay.azk8s.cn/external_storage/rbd-provisioner"
314c314
< local_path_provisioner_image_repo: "docker.io/rancher/local-path-provisioner"
---
> local_path_provisioner_image_repo: "dockerhub.azk8s.cn/rancher/local-path-provisioner"
316c316
< ingress_nginx_controller_image_repo: "quay.io/kubernetes-ingress-controller/nginx-ingress-controller"
---
> ingress_nginx_controller_image_repo: "quay.azk8s.cn/kubernetes-ingress-controller/nginx-ingress-controller"
319c319
< cert_manager_controller_image_repo: "quay.io/jetstack/cert-manager-controller"
---
> cert_manager_controller_image_repo: "quay.azk8s.cn/jetstack/cert-manager-controller"
322c322
< addon_resizer_image_repo: "k8s.gcr.io/addon-resizer"
---
> addon_resizer_image_repo: "gcr.azk8s.cn/google_containers/addon-resizer"
325c325
< dashboard_image_repo: "gcr.io/google_containers/kubernetes-dashboard-{{ image_arch }}"
---
> dashboard_image_repo: "gcr.azk8s.cn/google_containers/kubernetes-dashboard-{{ image_arch }}"
```
#### 集群创建
```
ansible-playbook -i inventory/k8s/inventory.ini cluster.yml -b -vvv
```

#### 增加节点
添加新节点信息到 inventory/k8s/hosts.ini 文件中，
```bash
# cd kubespray
# ansible-playbook -i inventory/k8s/inventory.ini scale.yml -b -v --private-key=~/.ssh/id_rsa --limit node2
```

#### 遇到问题卸载
安装过程中，遇到了 [Can't join a new node using kubeadm #4117](https://github.com/kubernetes-sigs/kubespray/issues/4117), 发现node2 无法加入到集群，详细查看确实没有对应的configmap，但我觉得没有必要手动去创建 configmap ， 就选择了卸载重新安装了:
```
# ansible-playbook -i inventory/mycluster/hosts.ini reset.yml
```

#### 集群验证
顺利的情况下， 10分钟左右会把集群安装好，让我们验证一下:
```bash
root@node1:~$ kubectl  get node
NAME    STATUS   ROLES    AGE     VERSION
node1   Ready    master   6h27m   v1.12.5
node2   Ready    node     6h25m   v1.12.5

root@node1:~$ kubectl  run nginx --image=nginx --replicas=2
deployment.apps/nginx created

root@node1:~$ kubectl  get pod -o wide                   
NAME                    READY   STATUS    RESTARTS   AGE   IP            NODE    NOMINATED NODE
nginx-dbddb74b8-vbmb8   1/1     Running   0          39s   10.233.96.4   node2   <none>
nginx-dbddb74b8-z4rjt   1/1     Running   0          39s   10.233.96.3   node2   <none>
```
#### dashboard 访问
我们首先要改一下 kubernetes-dashboard 的service类型，默认是clusterIP类型， 改成NodePort的，然后再访问，改完之后的效果:
```bash
root@node1:~$ kubectl  get svc -n kube-system                        
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
coredns                ClusterIP   10.233.0.3     <none>        53/UDP,53/TCP,9153/TCP   6h26m
kubernetes-dashboard   NodePort    10.233.17.39   <none>        443:32638/TCP            6h26m
```

![kube-sprary-login](/images/kube-sprary-login.png)
通过 dashboard的 token 访问:
```bash
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | awk '{print $1}'| grep kubernetes-dashboard-token)            
Name:         kubernetes-dashboard-token-2s6xg
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard
             kubernetes.io/service-account.uid: c864305a-cf7f-11e9-9766-005056fbe45f

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      xxxxxxx
```

#### 国内镜像源
Azure China提供了目前用过的质量最好的镜像源，涵盖了docker.io，gcr.io，quay.io。无论是速度还是覆盖范围，体验都极佳。而且都支持匿名拉取，也就是不需要登录。这点特别友好。azk8s.cn支持的镜像代理转换如下列表。

| global |	proxy in China |	format |	example |
| ------ | ------ | ------ | ------ |
|dockerhub (docker.io)|	dockerhub.azk8s.cn |	dockerhub.azk8s.cn/<repo-name\>/<image-name\>:<version\> |	dockerhub.azk8s.cn/microsoft/azure-cli:2.0.61 dockerhub.azk8s.cn/library/nginx:1.15|
|gcr.io	|gcr.azk8s.cn |	gcr.azk8s.cn/<repo-name\>/<image-name>:<version\>	| gcr.azk8s.cn/google_containers/hyperkube-amd64:v1.13.5|
|quay.io	|quay.azk8s.cn | quay.azk8s.cn/<repo-name\>/<image-name\>:<version\> |quay.azk8s.cn/deis/go-dev:v1.10.0|

**参考:**   
 https://github.com/Azure/container-service-for-azure-china/tree/master/aks#22-container-registry-proxy
