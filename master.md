---
title: "部署 master 节点"
date: 2019-03-16T16:15:48+08:00
draft: false
---
kubernetes master 节点运行如下组件：  
***kube-apiserver***  
***kube-scheduler***  
***kube-controller-manager***  
***kube-scheduler*** 和 ***kube-controller-manager*** 可以以集群模式运行，通过 leader 选举产生一个工作进程，其它进程处于阻塞模式。
对于 ***kube-apiserver***，可以运行多个实例，但对其它组件需要提供统一的访问地址，该地址需要高可用。本文档使用 keepalived 和 haproxy 实现 kube-apiserver VIP 高可用和负载均衡。本次我们先使用1个实例运行(master01)，以后再扩展成3个实例；  

将我们下载好的二进制放入 /opt/k8s/bin/ 目录下
```
cp kube-apiserver kube-controller-manager kube-scheduler /opt/k8s/bin/
```
