# K8S 从安装到精通

部署k8s集群之前，需要对k8s的架构有个基本的了解，要清楚安装的每个组件是做什么用的；

### k8s 架构
Kubernetes 最初源于谷歌内部的 Borg，提供了面向应用的容器集群部署和管理系统。
![k8s-arch](/images/k8s_arch.png)

**Kubernetes** 主要由以下几个核心组件组成：

**etcd** 保存了整个集群的状态；  
**apiserver** 提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；  
**controller manager** 负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
**scheduler** 负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；  
**kubelet** 负责维护容器的生命周期，同时也负责Volume（CVI）和网络（CNI）的管理；  
**Container runtime** 负责镜像管理以及Pod和容器的真正运行（CRI）；  
**kube-proxy** 负责为Service提供cluster内部的服务发现和负载均衡；  
**cAdvisor** 负责容器资源监控：

### k8s 组件间通信
![k8s-components](/images/k8s_components.png)

除了核心组件， 为了完善 k8s 功能，k8s还有一些推荐的组件:  
**kube-dns**  负责为整个集群提供DNS服务  
**Dashboard** 提供管理界面  
**Ingress Controller** 为服务提供外网入口  
**Heapster/MetricServer** 提供资源监控  
**Federation** 提供跨可用区的集群  
**Fluentd-elasticsearch** 提供集群日志采集、存储与查询  


### 安装步骤

* [Introduction](README.md)
1. [00.组件版本和配置策略](version.md)
1. [01.系统初始化和全局变量](os-init.md)
1. [02.创建CA证书和秘钥](ca.md)
1. [03.部署kubectl命令行工具](kubectl.md)
1. [04.部署etcd集群](etcd.md)
1. [05.部署flannel网络](flannel.md)
1. [06.部署master节点](master.md)
    1. ~~[06-1.ha]~~
    1. [06-2.api-server](kube-apiserver.md)
    1. [06-3.controller-manager集群](kube-controller-manager.md)
    1. [06-4.scheduler集群](kube-scheduler.md)		
1. [07.部署worker节点]()
    1. [07-1.docker](docker.md)
    1. [07-2.kubelet](kubelet.md)
    1. [07-3.kubelet-without-bootstrap](kubelet-without-bootstrap.md)
    1. [07-4.kube-proxy](kube-proxy.md)
1. [08.验证集群功能](verify.md)
1. [09.部署集群插件](deploy-plugins.md)
    1. [09-1.coredns](coredns.md)
    1. [09-2.dashboard](dashboard.md)
    1. [09-3.heapster](heapster.md)

1. [A.浏览器访问kube-apiserver安全端口](kube-apiserver-sec-port.md)
