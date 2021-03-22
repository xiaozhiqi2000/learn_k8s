[TOC]

# kubernetes 组件
[REFERENCE](https://kubernetes.io/zh/docs/concepts/overview/components/)

![image-20210322163339651](imges/image-20210322163339651.png)

## Master 节点
### kube-apiserver

### etcd
[REFERENCE](https://etcd.io/docs/)
> etcd 是兼具一致性和高可用的键值数据库，可以作为保存 kubernetes 所有集群数据的后台数据库。需要备份

### kube-scheduler
> 负责监视新创建的、未指定运行节点（node）的 Pods，选择节点让 Pod 在上面运行。调度策略考虑的因素包括单个 Pod 和 Pod集合的资源需求、硬件/软件/策略约束、亲和性和反亲和性、数据位置、工作负载的干扰和最后时限

### kube-controller-manager
> 从逻辑上讲，每个控制器都是一个单独的进程，但是为了降低复杂性，它们都被编译到同一个可执行文件，并在一个进程中运行。

这些控制器包括：
- 节点控制器（Node Controller）：负责在节点出现故障时进行通知和响应
- 副本控制器（Replication Controller）：负责为系统中的每个副本控制器对象维护正确数量的 Pod
- 端点控制器（Endpoints Controller）：填充端点（Endpoints）对象（即加入 Service 与 Pod）
- 服务账户和令牌控制器（Service Account & Token Controllers）：为新的命名空间创建默认账户和API 访问令牌

## Node节点
节点组件在每个节点上运行，维护运行的 Pod 并提供 kubernetes 运行环境

### kubelet
> 一个在集群中每个节点（node）上运行的代理。它保证容器（containers）都运行在 Pod 中。kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中秒速的容器处于运行状态且健康。kubelet 不会管理不是由 kubernets 创建的容器。

### kube-proxy
> kube-proxy 是集群中每个节点上运行的网络代理，实现 kubernetes 服务（service）概念的一部分。kube-proxy 维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。

### Container runtime
> 容器运行环境是负责运行容器的软件。kubernetes 支持多个容器运行环境：Docker、containerd、CRI-O 以及任何实现 kubernetes CRI （容器运行环境接口）[容器运行环境接口](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md)

### Addons
> 插件实用 kubernetes 资源（DaemonSet、Deployment等）














