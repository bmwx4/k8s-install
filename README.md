# kubernetes 1.12 集群部署

### 安装步骤

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
    1. [07-3.kube-proxy](kube-proxy.md)			
1. [08.验证集群功能](verify.md)
1. [09.部署集群插件](deploy-plugins.md)
    1. [09-1.coredns](coredns.md)
    1. [09-2.dashboard](dashboard.md)
    1. [09-3.heapster](heapster.md)

1. [A.浏览器访问kube-apiserver安全端口](kube-apiserver-sec-port.md)
