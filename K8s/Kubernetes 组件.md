### 组件
一个 Kubernetes 集群主要由 控制节点 和 工作节点 构成，每个节点都会安装不同的组件。

**master**：集群的控制平面，负责集群的决策和管理
>ApiServer: 资源操作的唯一入口，接收用户输入的命令，提供认证、授权、API 注册和发现等级制
>Scheduler：负责集群的资源调度，按照预定的策略将 Pod 调度到相应的 Node 上。
>ControllerManager：负责维护集群的状态，比如程序部署安排、故障检测、自动扩展，滚动更新等。
>Etcd：负责存储集群中个资源对象的信息。

**node：** 集群的数据平面，负责为容器提供运行环境
>Kubelet：负责维护容器的生命周期，通过控制 docker 来创建、更新、销毁容器
>KubeProxy：负责提供集群内部的服务发现和负载均衡
>容器运行时：Kubernetes 支持许多容器运行环境，例如 containerd、 CRI-0、Docker 以及 Kubernetes CRI 的其他任何实现。

![](https://gitee.com/yooome/golang/raw/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B-%E8%B0%83%E6%95%B4%E7%89%88/Kubenetes.assets/image-20200406184656917.png)
