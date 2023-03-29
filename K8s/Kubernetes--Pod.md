### Pod 
#### Pod 介绍
**Pod 结构**

![pod结构](https://gitee.com/yooome/golang/raw/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B-%E8%B0%83%E6%95%B4%E7%89%88/Kubenetes.assets/image-20200407121501907-1626781151898.png)
每个 Pod 都一个或多个容器，容器有两类：
- 用户程序所在的容器，数量可以多个
- Pause 容器，每个 Pod 的根容器，作用为：
	- 协调 Pod 中的其他容器，他会创建 Pod 的网络命名空间和共享存储卷，在容器之间共享这些资源
	- 确保 Pod 的生命周期与容器的生命周期保持一致
 > 具体来说，pause 容器会在 Pod 创建时自动启动，并且创建一个网络命名空间和一个共享存储卷。然后 Pod 中的其他容器可以加入这个网络命名空间和共享存储卷，从而实现容器间的通信和资源共享。Pause 容器自身不会执行其他任何实际的任务，只是一个协调器。
 > pause 还有一个重要的作用就是确保 Pod 的生命周期和容器的生命周期保持一致，当 Pod 中有一个容器退出时，pause 容器也会退出，从而触发 pod 的删除。这样可以确保 Pod 中的所有容器都可以同时启动和停止，从而实现容器间的协同工作。
#### Pod 生命周期
**pod 的生命周期出现的 5 种相位：**
- 悬停 (Pending)：apiserver 已经创建好了 pod 的资源，但是还没有被调度完成或者在拉取镜像的阶段
- 运行中 (Running)：pod 已经被调度到节点上，并且所有容器都已经被 kubelet 创建完成
- 成功 (Succeeded)：所有容器都已经成功终止，并且不需要再重启
- 失败 (Failed)：所有容器都已经被终止，但至少有一个容器终止失败
- 未知 (Unknown)：apiserver 无法获取 pod 的状态信息
> 1. 当一个 Pod 被删除时，这个 Pod 状态会显示 Terminating 状态，这个不是 Pod 的阶段之一。Pod 被赋予一个可以体面终止的期限，默认为 30s，可以使用 `--force` 强制终止 Pod
> 2. 如果某节点死掉或者与集群中的其他节点失联，k8s 会实施一种策略，将失去的节点上运行的所有 Pod 设置为 Failed
#### Container 特性
**容器的生命周期回调**

有两个回调暴露给容器：
- `PostStart` 这个回调在容器被创建之后立即执行。如果失败会重启容器
- `PreStop` 容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作
这些回调函数可以通过在 Pod 文件中定义 `lifecycle` 字段来添加。例如，下面的 YAML 文件定义了一个包含 `PostStart` 和 `PreStop` 两个回调函数的容器：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: my-image
    lifecycle:
      postStart:
        exec:
          command: ["/bin/sh", "-c", "echo Container started"]
      preStop:
        exec:
          command: ["/bin/sh", "-c", "echo Container stopping"]
```

**容器的重启策略**

Pod 定义文件中的 `spec` 中包含一个 `resartPolicy` 字段，可能的取值为 `Always(总是重启)、OnFailure(容器异常退出状态码非 0,重启) 和 Never`。默认值是 `Always`。
1. Always：无论容器退出的原因是什么，Kubernetes 都会自动重启容器。
2. OnFailure：表述仅当容器由于错误或异常情况退出时，Kubernetes 才会重启容器。
3. Never：不自动重启。

**容器探针**

probe 是由 kubelet对容器执行的定期诊断。 要执行诊断，kubelet 既可以在容器内执行代码，也可以发出一个网络请求。

探针类型：
-   `livenessProbe` **指示容器是否正在运行**。如果存活态探测失败，则 kubelet 会杀死容器， 并且容器将根据其重启策略决定未来。如果容器不提供存活探针， 则默认状态为 `Success`。
-   `readinessProbe`**指示容器是否准备好为请求提供服**。如果就绪态探测失败， 端点控制器将从与 Pod 匹配的所有服务的端点列表中删除该 Pod 的 IP 地址。 初始延迟之前的就绪态的状态值默认为 `Failure`。 如果容器不提供就绪态探针，则默认状态为 `Success`。
-   `startupProbe 1.7+`**指示容器中的应用是否已经启动**。如果提供了启动探针，则所有其他探针都会被 禁用，直到此探针成功为止。如果启动探测失败，`kubelet` 将杀死容器， 而容器依其重启策略进行重启。 如果容器没有提供启动探测，则默认状态为 `Success`。

探针机制：
- `exec`
    在容器内执行指定命令。如果命令退出时返回码为 0 则认为诊断成功。
-   `grpc`
    使用 gRPC 执行一个远程过程调用。 目标应该实现 gRPC健康检查。 如果响应的状态是 "SERVING"，则认为诊断成功。 gRPC 探针是一个 Alpha 特性，只有在你启用了 "GRPCContainerProbe" 特性门控时才能使用。
-   `httpGet`
    对容器的 IP 地址上指定端口和路径执行 HTTP `GET` 请求。如果响应的状态码大于等于 200 且小于 400，则诊断被认为是成功的。
-   `tcpSocket`
    对容器的 IP 地址上的指定端口执行 TCP 检查。如果端口打开，则诊断被认为是成功的。 如果远程系统（容器）在打开连接后立即将其关闭，这算作是健康的。

**Init 容器**

Pod 中的 init 容器是在容器启动之前运行的一种特殊类型的容器。init 容器与普通容器一样，都可以运行一个镜像，但是他们的目的是在主容器启动之前做一些初始化工作任务，例如初始化数据库、加载配置文件等。
init 容器与普通容器非常像，除了以下几点：
- init 容器总是运行到完成。如果 Pod 中的 init 容器失败，kubelet 会不断的重启这个 init 容器，直到成功为止，如果 Pod 对应的 `restartPolicy` 值为“Never”，并且 init 容器启动失败，则 kubernetes 会将整个 Pod 状态设置为失败。
- 会按照 Pod 文件的顺序启动，上一个启动成功之后下一个才启动。
- Init 容器不支持 生命周期回调 和 探针机制 因为他必须在 Pod 就绪之前运行完成
使用 Init 容器：
在 Pod 的规约中与用来描述应用容器的 `containers` 数组平行的位置指定 Init 容器。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh', '-c', 'echo The app is running! && sleep 3600']
  initContainers:
  - name: init-myservice
    image: busybox:1.28
    command: ['sh', '-c', 'echo init-myservice is running! && sleep 5']
  - name: init-mydb
    image: busybox:1.28
    command: ['sh', '-c', 'echo init-mydb is running! && sleep 10']
```

#### Pod 调度
**定向调度**

定向调度，指的是利用在 pod 上声明 nodeName 或者 nodeSelector，以此将Pod调度到期望的node节点上。
nodeName：用于强制约束将Pod调度到指定的Name的Node节点上。这种方式，其实是直接跳过Scheduler的调度逻辑，直接将Pod调度到指定名称的节点。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeName: node1 # 指定调度到node1节点上
```
NodeSelector：NodeSelector用于将pod调度到添加了指定标签的node节点上。它是通过kubernetes的label-selector机制实现的，也就是说，在pod创建之前，会由scheduler使用MatchNodeSelector调度策略进行label匹配，找出目标node，然后将pod调度到目标节点，该匹配规则是强制约束。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector
  namespace: dev
spec:
  containers:
  - name: nginx
    image: nginx:1.17.1
  nodeSelector: 
    nodeenv: pro # 指定调度到具有nodeenv=pro标签的节点上
```
**亲和性调度**

定向调度方式中，如果没有满足条件的 Node，那么 Pod 将不会被运行，处于 Pending 状态，亲和性调度（Affinity），**在 NodeSelector 上进行了扩展，可以通过配置的形式实现优先优先满足使用条件的 Node 进行调度，如果没有也可以调度到不满足条件的节点上。**
Affinity 主要分为三类：
- nodeAffinity（node 亲和性）：以 node 为目标解决 pod 可以调度到哪些 node 的问题。
```yaml
# node 亲和性配置项
pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution  Node节点必须满足指定的所有规则才可以，相当于硬限制
    nodeSelectorTerms  节点选择列表
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operat or 关系符 支持Exists, DoesNotExist, In, NotIn, Gt, Lt
  preferredDuringSchedulingIgnoredDuringExecution 优先调度到满足指定的规则的Node，相当于软限制 (倾向)
    preference   一个节点选择器项，与相应的权重相关联
      matchFields   按节点字段列出的节点选择器要求列表
      matchExpressions   按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist, Gt, Lt
	weight 倾向权重，在范围1-100。
```
> 注意事项：
> 	1. 如果同时定义了 nodeSelector 和 nodeAffinity，那么必须两个条件都满足，Pod才能部署到指定的 Node 上
> 	2. 如果 Pod 所在的 Node 在 Pod 运行期间改变了 label 不再符合亲和性，那么系统将会忽略此变化
- podAffinity（pod 亲和性）：以 Pod 为目标，解决 Pod 可以和哪些已经存在的 Pod 部署在同一个拓扑域的问题
```yaml
# pod 亲和性配置项
pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution  硬限制
    namespaces       指定参照pod的namespace
    topologyKey      指定调度作用域
    labelSelector    标签选择器
      matchExpressions  按节点标签列出的节点选择器要求列表(推荐)
        key    键
        values 值
        operator 关系符 支持In, NotIn, Exists, DoesNotExist.
      matchLabels    指多个matchExpressions映射的内容
  preferredDuringSchedulingIgnoredDuringExecution 软限制
    podAffinityTerm  选项
      namespaces      
      topologyKey
      labelSelector
        matchExpressions  
          key    键
          values 值
          operator
        matchLabels 
    weight 倾向权重，在范围1-100
```

#### 污点和容忍
**污点**

前面的调度方式都是站在 Pod 的角度，在 Pod 上配置条件，来决定 Pod 是否需要调度到某个节点上，其实也可以站在 Node 的角度，通过对 Node 添加 污点 属性，来决定是否允许 Pod 调度过来。
Node 被设置上污点之后，就可以和 Pod 之间形成一种相斥的关系，进而拒绝 Pod 调度过来，甚至可以将已存在的 Pod 驱逐出去。
污点的格式为：`key=value:effect`，key 和 value 是污点的标签，effect 是描述污点的作用，支持三个选项：
- PreferNoSchedule：kubernetes 将尽量避免把 Pod 调度到具有该污点的 Node 上，除非没有其他节点可以调度。
- NoSchedule：kubernetes 将不会把 Pod 调度到具有该污点的 Node 上，但不会影响当前 Node 上已存在的 Pod；
- NoExecute ：kubernetes 将不会把 Pod 调度到具有该污点的 Node 上，同时也会将 Node 上已经存在的 Pod 进行驱逐。
使用 kubelet 设置和去除污点的命令：
```
# 设置污点
kubectl taint nodes node1 key=value:effect

# 去除污点
kubectl taint nodes node1 key:effect-

# 去除所有污点
kubectl taint nodes node1 key-
```

**容忍**

污点可以拒绝一个 Pod 调度到这个 Node 上，那么如果我们就是想调度到一个有污点的节点上，这个时候就需要容忍。
![](https://gitee.com/yooome/golang/raw/main/k8s%E8%AF%A6%E7%BB%86%E6%95%99%E7%A8%8B-%E8%B0%83%E6%95%B4%E7%89%88/Kubenetes.assets/image-20200514095913741.png)
> 污点就是拒绝，容忍就是忽略，Node 通过污点拒绝 Pod 调度上去，Pod 通过容忍忽略拒绝

