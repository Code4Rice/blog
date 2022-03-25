[TOC]

## K8S网络基础

> 在进入Cilium介绍学习之前，大概回顾一下K8S的网络模型可以帮助我们更好地理解Cilium的作用

---

### K8S网络模型特点

- **每一个Pod都有自己的IP地址**。不需要在pod之间创建连接，也不需要将容器端口映射到主机端口。
- 一个节点上的pod可以与所有节点上的所有pod进行通信，**不需要NAT转换**。
- **节点上的代理（例如：系统守护进程，Kubelet）可以与该节点中的所有pod进行通信**。
- **Pod中的容器共享它们的网络命名空间（IP和MAC地址）**，因此可以通过loopback地址相互通信。

---

### K8S四种网络模式

#### 容器到容器网络

容器到容器的网络是通过Pod网络命名空间实现的。网络命名空间允许我们拥有独立的网络接口和路由表，这些接口和路由表与系统的其他部分隔离，并且可以独立运行。每一个Pod都拥有自己的网络命名空间，而且Pod里的容器都共享使用相同的IP地址和端口。所有这些容器间的通信都通过localhost的方式，因为它们都属于同一个命名空间。（图中的绿线代表了这种通信方式）

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/vibjb3KpB2ay6ErgWiaDIpbjsUorz9KQvVcltib73Z53WZQLI45TuNZN0lXuGMjr9yBQS7q7L8cAicF8QpuUXJXMcQ/0?wx_fmt=jpeg)

---

#### Pod到Pod网络

在K8S中，每个节点都有一个为Pods指定的CIDR范围内的IP数量。这样可以确保每一个Pod都有一个唯一的IP地址，可以被集群中其他的Pod访问，也可以保证当一个新的Pod被创建时，IP地址不会重叠。和容器到容器网络不同，Pod之间通信使用真实IP，无论Pod部署在集群中的同一个节点还是不同节点上。

如图上所示，Pod之间通信时，流量需要在Pod网络命名空间和根网络命名空间之间流动。通过虚拟以太网设备或者一对虚拟设备接口（例如图中的veth0到pod 1命名空间，veth1到pod2命名空间）来连接Pod和根网络命名空间。这些虚拟网卡接口通过虚拟网桥进行连接，通过ARP协议需要流量在它们之间流动。

举个例子，如果数据流从Pod1发送到Pod2，会经过以下几步：

1. Pod1的流量从eth0到达根网络命名空间的虚拟接口veth0。
2. 流量从veth0到达虚拟网桥。
3. 从虚拟网桥到达veth1。
4. 最后，通过veth1流量到达Pod2的eth0，完成通信

---

#### Pod到Service网络

在实际生产环境中，Pod会经常变化，往往需要根据需求进行扩缩容。当程序崩溃或者节点异常，Pod需要被重新创建。这些操作都会导致Pod的IP经常变化，使得网络通信变得复杂起来。

K8S通过Service来解决这个问题：

1. 在前端分配一个静态虚拟IP地址，用于连接与Service关联的任何后端Pod。
2. 将任何指向这个虚拟IP的流量负载均衡到后台的Pods池里
3. 跟踪Pods的IP地址，这样即使Pod的IP地址改变，客户端也不会受影响，因为客户端只和Service本身的静态虚拟IP地址直连。

集群内的负载均衡可以通过两种方式实现：

1. IPTABLES：这种模式下，kube-proxy监视API Server的变化，对于每一个新的Service，安装iptables的规则，这些规则捕捉到Service的集群IP和端口的流量，并将流量重定向到该Service的后端Pod。Pod是随机选择的。这种模式更可靠，系统开销更低，因为流量是通过Linux的Netfilter处理，不需要用户态和内核态之间的切换。
2. IPVS：IPVS是建立在Netfilter基础上实现的传输层的负载均衡。IPVS使用Netfilter的钩子函数，使用哈希表作为底层数据结构，在内核态工作。这意味着与iptables模式相比，IPVS模式下的kube-proxy将以更低延迟、更高吞吐和更好的性能重定向流量。

在下图中，Pod1到Pod3的流量路线由红色的线标注出来。请注意，传输到虚拟网桥的包必须通过默认路由(eth0)来路由，因为在网桥上运行的ARP协议并不能识别Service，然后这些数据包被iptables过滤，iptables将使用kube-proxy在节点中定义的规则。

![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/vibjb3KpB2ay6ErgWiaDIpbjsUorz9KQvVwA70mbg6T7obsELQOmYU1RsPcjdS0CbPyhYrylz4Ywd1KpkA9GGMxA/0?wx_fmt=jpeg)

---

#### 外网到Service网络

至此我们讨论了流量是如何在集群内传输的。如果我们需要把一个应用暴露给外网使用，我们可以有两种方式：

1. **Egress：**当你准备将你的K8S服务的流量路由到外部互联网时。iptables将执行源NAT，以便流量看起来将来自节点而不是pod。

2. **Ingress：**这是当外网流量进入你的K8S服务时。使用一组连接规则，Ingress还允许和阻止与服务的特定通信。通常，有两个入口解决方案在不同的网络堆栈区起作用：服务负载平衡器（Service load balancer）和入口控制器（Ingress controller）。

   ![](https://mmbiz.qpic.cn/sz_mmbiz_jpg/vibjb3KpB2ay6ErgWiaDIpbjsUorz9KQvVlmKHGkr9ibzFd7tcx8N6GZodp0w6wR67wA8HVgBOKeqbSm4vBIfd6aw/0?wx_fmt=jpeg)

---

### 服务发现

K8S有两种方式来进行服务发现：

1. **环境变量：**在Pod运行的节点上运行的kubelet服务负责为每个活跃的服务设置环境变量，设置的格式是{SVCNAME}\_SERVICE_HOST & {SVCNAME}_SERVICE_POST。在Pods创建之前需要创建Service，否则这些Posd不会填充它们的环境变量。
2. **DNS：**DNS服务是一种K8S服务的类型，映射到一个或多个DNS服务pod上，和其他pod一样被正常调度。集群中的pod可以配置使用DNS服务，有一个DNS搜索列表，包含了pod自己的命名空间和集群的默认域名。一个支持集群DNS的服务，例如CoreDNS，监听K8S的API，每当有新服务创建时，为每个服务创建一组DNS记录。如果DNS已经在整个集群被使用，那么这个集群里的所有pod都能够通过服务的DNS名称自动解析服务。K8S的DNS服务器是访问ExternalName服务的唯一方法（ExternalName服务是一种特殊服务，它没有选择器selector，而是使用DNS名字）。

---///////////////

### 服务类型

K8S为我们提供了一种访问一组Pod的方法，一组pod通常可以使用标签选择器来定义。这使得我们可以在集群内的应用之间访问，或者允许应用暴露在外网访问的环境中。K8S的ServiceTypes允许你指定想要的服务类型。

![](https://mmbiz.qpic.cn/sz_mmbiz_png/vibjb3KpB2ay6ErgWiaDIpbjsUorz9KQvVpzoLdM9Iqcnm0Zl2hwtwyNRKP5mu97ndl6zuwxxLsHDZCETS3fUk0Q/0?wx_fmt=png)

有这么几种ServiceTypes：

1. **ClusterIP：**这是默认的服务类型。服务只能在集群内被触达，并且允许集群内的服务互相通信。没有外部访问权限。
2. **LoadBalancer：**这种服务类型将服务暴露给外网环境，通常使用云服务商提供的负载均衡器，例如CLB。来自外部的流量通过LB直接打到后端Pod上。云服务商决定采用什么样的算法进行负载均衡，通常也是可以让用户选择使用。
3. **NodePort：**通过给所有节点开放一个指定端口号的方式，这种服务类型允许外部流量访问服务。所有流量都发送到这个端口，然后转发到相应服务。
4. **ExternalName：**这种类型的服务通过使用externalName字段返回一条CNAME记录，做为服务映射到DNS的名字。不使用任何类型的代理。


