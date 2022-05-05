
# 2022-05-05-Kubernetes网络三部曲之一～Pod网络
* * *
## 前言

K8s 是一个强大的平台，但它的网络比较复杂，涉及很多概念，例如 Pod 网络，Service 网络，Cluster IPs，NodePort，LoadBalancer 和 Ingress 等等，这么多概念足以让新手望而生畏。但是，只有深入理解 K8s 网络，才能为理解和用好 K8s 打下坚实基础。为了帮助大家理解，模仿 TCP/IP 协议栈，我把 K8s 的网络分解为四个抽象层，从 0 到 3，除了第 0 层，每一层都是构建于前一层之上，如下图所示：

![](https://img-blog.csdnimg.cn/20190921112602532.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmc3NTEwOA==,size_16,color_FFFFFF,t_70)

第 0 层[Node](https://so.csdn.net/so/search?q=Node&spm=1001.2101.3001.7020)节点网络比较好理解，也就是保证 K8s 节点 (物理或虚拟机) 之间能够正常 IP 寻址和互通的网络，这个一般由底层 (公有云或数据中心) 网络基础设施支持。第 0 层我们假定已经存在，所以不展开。第 1 到 3 层网络，我将分三篇文章，分别进行剖析，本文是第一篇 Pod 网络。

注意，本文旨在帮助大家建立 K8s 网络的概念模型，而不是对底层技术的精确描述。实际我们学技术以应用为主，重要的是快速建立起直观易懂的概念模型，能够指导我们正常应用即可，当然理解一定的技术细节也是有帮助的。另外，本文假定读者对基本的网络技术，ip 地址空间和容器技术等有一定的了解。

## Pod 网络概念模型

Pod 相当于是 K8s 云平台中的虚拟机，它是 K8s 的基本调度单位。所谓 Pod 网络，就是能够保证 K8s 集群中的所有 Pods(包括同一节点上的，也包括不同节点上的 Pods)，逻辑上看起来都在同一个平面网络内，能够相互做 IP 寻址和通信的网络，下图是 Pod 网络的简化概念模型：

![](https://img-blog.csdnimg.cn/20190921112649439.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmc3NTEwOA==,size_16,color_FFFFFF,t_70)

Pod 网络构建于 Node 节点网络之上，它又是上层 Service 网络的基础。为了进一步理解 Pod 网络，我将对同一节点上的 Pod 之间的网络，以及不同节点上的 Pod 之间网络，分别进行剖析。

## 同一节点上的 Pod 网络

前面提到，Pod 相当于是 K8s 云平台中的虚拟机，实际一个 Pod 中可以住一个或者多个 (大多数场景住一个) 应用容器，这些容器共享 Pod 的网络栈和其它资源如 Volume。那么什么是共享网络栈？同一节点上的 Pod 之间如何寻址和互通？我以下图样例来解释：

![](https://img-blog.csdnimg.cn/20190921112728407.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmc3NTEwOA==,size_16,color_FFFFFF,t_70)

上图节点上展示了 Pod 网络所依赖的 3 个网络设备，eth0 是节点主机上的网卡，这个是支持该节点流量出入的设备，也是支持集群节点间 IP 寻址和互通的设备。docker0 是一个虚拟网桥，可以简单理解为一个虚拟交换机，它是支持该节点上的 Pod 之间进行 IP 寻址和互通的设备。veth0 则是 Pod1 的虚拟网卡，是支持该 Pod 内容器互通和对外访问的虚拟设备。docker0 网桥和 veth0 网卡，都是 linux 支持和创建的虚拟网络设备。

上图 Pod1 内部住了 3 个容器，它们都共享一个虚拟网卡 veth0。内部的这些容器可以通过 localhost 相互访问，但是它们不能在同一端口上同时开启服务，否则会有端口冲突，这就是共享网络栈的意思。Pod1 中还有一个比较特殊的叫**pause**的容器，这个容器运行的唯一目的是为 Pod 建立共享的 veth0 网络接口。如果你 SSH 到 K8s 集群中一个有 Pod 运行的节点上去，然后运行**docker ps**，可以看到通过**pause**命令运行的容器。

Pod 的 IP 是由 docker0 网桥分配的，例如上图 docker0 网桥的 IP 是 172.17.0.1，它给第一个 Pod1 分配 IP 为 172.17.0.2。如果该节点上再启一个 Pod2，那么相应的分配 IP 为 172.17.0.3，如果再启动 Pod 可依次类推。因为这些 Pods 都连在同一个网桥上，在同一个网段内，它们可以进行 IP 寻址和互通，如下图所示：

![](https://img-blog.csdnimg.cn/20190921112825755.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmc3NTEwOA==,size_16,color_FFFFFF,t_70)

从上图我们可以看到，节点内 Pod 网络在 172.17.0.0/24 这个地址空间内，而节点主机在 10.100.0.0/24 这个地址空间内，也就是说 Pod 网络和节点网络不在同一个网络内，那么不同节点间的 Pod 该如何 IP 寻址和互通呢？下一节我们来分析这个问题。

## 不同节点间的 Pod 网络

现在假设我们有两个节点主机，host1(10.100.0.2) 和 host2(10.100.0.3)，它们在 10.100.0.0/24 这个地址空间内。host1 上有一个 PodX(172.17.0.2)，host2 上有一个 PodY(172.17.1.3)，Pod 网络在 172.17.0.0/16 这个地址空间内。注意，Pod 网络的地址，是由 K8s 统一管理和分配的，保证集群内 Pod 的 IP 地址唯一。我们发现节点网络和 Pod 网络不在同一个网络地址空间内，那么 host1 上的 PodX 该如何与 host2 上的 PodY 进行互通？

![](https://img-blog.csdnimg.cn/20190921113006546.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmc3NTEwOA==,size_16,color_FFFFFF,t_70)

实际上不同节点间的 Pod 网络互通，有很多技术实现方案，底层的技术细节也很复杂。为了简化描述，我把这些方案大体分为两类，一类是路由方案，另外一类是覆盖 (Overlay) 网络方案。

如果底层的网络是你可以控制的，比如说企业内部自建的数据中心，并且你和运维团队的关系比较好，可以采用路由方案，如下图所示：

![](https://img-blog.csdnimg.cn/20190921113037220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmc3NTEwOA==,size_16,color_FFFFFF,t_70)

这个方案简单理解，就是通过路由设备为 K8s 集群的 Pod 网络单独划分网段，并配置路由器支持 Pod 网络的转发。例如上图中，对于目标为 172.17.1.0/24 这个范围内的包，转发到 10.100.0.3 这个主机上，同样，对于目标为 172.17.0.0/24 这个范围内的包，转发到 10.100.0.2 这个主机上。当主机的 eth0 接口接收到来自 Pod 网络的包，就会向内部网桥转发，这样不同节点间的 Pod 就可以相互 IP 寻址和通信。这种方案依赖于底层的网络设备，但是不引入额外性能开销。

如果底层的网络是你无法控制的，比如说公有云网络，或者企业的运维团队不支持路由方案，可以采用覆盖 (Overlay) 网络方案，如下图所示：

![](https://img-blog.csdnimg.cn/20190921113100343.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmc3NTEwOA==,size_16,color_FFFFFF,t_70)

所谓覆盖网络，就是在现有网络之上再建立一个虚拟网络，实现技术有很多，例如 flannel/weavenet 等等，这些方案大都采用隧道封包技术。简单理解，Pod 网络的数据包，在出节点之前，会先被封装成节点网络的数据包，当数据包到达目标节点，包内的 Pod 网络数据包会被解封出来，再转发给节点内部的 Pod 网络。这种方案对底层网络没有特别依赖，但是封包解包会引入额外性能开销。

## CNI 简介

考虑到 Pod 网络实现技术众多，为了简化集成，K8s 支持 CNI(Container Network Interface) 标准，不同的 Pod 网络技术可以通过 CNI 插件形式和 K8s 进行集成。节点上的 Kubelet 通过 CNI 标准接口操作 Pod 网路，例如添加或删除网络接口等，它不需要关心 Pod 网络的具体实现细节。

![](https://img-blog.csdnimg.cn/20190921113136617.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3lhbmc3NTEwOA==,size_16,color_FFFFFF,t_70)

## 总结

1.  K8s 的网络可以抽象成四层网络，第 0 层节点网络，第 1 层 Pod 网络，第 2 层 Service 网络，第 3 层外部接入网络。除了第 0 层，每一层都构建于上一层之上。
2.  一个节点内的 Pod 网络依赖于虚拟网桥和虚拟网卡等 linux 虚拟设备，保证同一节点上的 Pod 之间可以正常 IP 寻址和互通。一个 Pod 内容器共享该 Pod 的网络栈，这个网络栈由 pause 容器创建。
3.  不同节点间的 Pod 网络，可以采用路由方案实现，也可以采用覆盖网络方案。路由方案依赖于底层网络设备，但没有额外性能开销，覆盖网络方案不依赖于底层网络，但有额外封包解包开销。
4.  CNI 是一个 Pod 网络集成标准，简化 K8s 和不同 Pod 网络实现技术的集成。

有了 Pod 网络，K8s 集群内的所有 Pods 在逻辑上都可以看作在一个平面网络内，可以正常 IP 寻址和互通。但是 Pod 仅仅是 K8s 云平台中的虚拟机抽象，最终，我们需要在 K8s 集群中运行的是应用或者说服务 (Service)，而一个 Service 背后一般由多个 Pods 组成集群，这时候就引入了服务发现(Service Discovery) 和负载均衡 (Load Balancing) 等问题，这是波波的下一篇《[Kubernetes](https://so.csdn.net/so/search?q=Kubernetes&spm=1001.2101.3001.7020)网络三部曲～Service 网络》要剖析的内容，敬请期待。 
 [https://blog.csdn.net/yang75108/article/details/101101384](https://blog.csdn.net/yang75108/article/details/101101384)
