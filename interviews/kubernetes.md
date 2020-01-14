### Kubernetes 面试题总结

**1、Kubernetes有什么特点**

* 自动调度
* 自愈能力
* 水平扩容
* 负载均衡

**2、什么是Heapster**

Heapster是容器集群监控和性能分析工具，天然的支持Kubernetes和CoreOS。

Kubernetes有个出名的监控agent—cAdvisor。在每个kubernetes Node上都会运行cAdvisor，它会收集本机以及容器的监控数据(cpu,memory,filesystem,network,uptime)。在较新的版本中，K8S已经将cAdvisor功能集成到kubelet组件中。每个Node节点可以直接进行web访问。

Heapster是一个收集者，Heapster可以收集Node节点上的cAdvisor数据，将每个Node上的cAdvisor的数据进行汇总，还可以按照kubernetes的资源类型来集合资源，比如Pod、Namespace，可以分别获取它们的CPU、内存、网络和磁盘的metric。默认的metric数据聚合时间间隔是1分钟。还可以把数据导入到第三方工具(如InfluxDB)。

Kubernetes原生dashboard的监控图表信息来自heapster。在Horizontal Pod Autoscaling中也用到了Heapster，HPA将Heapster作为Resource Metrics API，向其获取metric。

**3、kubernetes 包含几个组件。各个组件的功能是什么。组件之间是如何交互的。**

- etcd保存了整个集群的状态；
- apiserver提供了资源操作的唯一入口，并提供认证、授权、访问控制、API注册和发现等机制；
- controller manager负责维护集群的状态，比如故障检测、自动扩展、滚动更新等；
- scheduler负责资源的调度，按照预定的调度策略将Pod调度到相应的机器上；
- kubelet负责维护容器的生命周期，同时也负责Volume（CSI）和网络（CNI）的管理；
- Container runtime负责镜像管理以及Pod和容器的真正运行（CRI）；
- kube-proxy负责为Service提供cluster内部的服务发现和负载均衡；

**4、Kubernetes的Pause容器有什么用，是否可以去掉？**

不能，共享命名空间，决定pod状态

- PID命名空间：Pod中的不同应用程序可以看到其他应用程序的进程ID；
- 网络命名空间：Pod中的多个容器能够访问同一个IP和端口范围；
- IPC命名空间：Pod中的多个容器能够使用SystemV IPC或POSIX消息队列进行通信；
- UTS命名空间：Pod中的多个容器共享一个主机名；Volumes（共享存储卷）：
- Pod中的各个容器可以访问在Pod级别定义的Volumes；

**5、一个经典Pod的完整生命周期。**

- 挂起（Pending）：Pod 已被 Kubernetes 系统接受，但有一个或者多个容器镜像尚未创建。等待时间包括调度 Pod 的时间和通过网络下载镜像的时间，这可能需要花点时间。
- 运行中（Running）：该 Pod 已经绑定到了一个节点上，Pod 中所有的容器都已被创建。至少有一个容器正在运行，或者正处于启动或重启状态。
- 成功（Successed）：Pod 中的所有容器都被成功终止，并且不会再重启。
- 失败（Failed）：Pod 中的所有容器都已终止了，并且至少有一个容器是因为失败终止。也就是说，容器以非0状态退出或者被系统终止。
- 未知（Unkonwn）：因为某些原因无法取得 Pod 的状态，通常是因为与 Pod 所在主机通信失败。

**6、详述kube-proxy的工作原理，一个请求是如何经过层层转发落到某个Pod上的整个过程。请求可能来自Pod也可能来自外部。**

* 每个Node节点上都会运行一个kube-proxy服务进程。

* 对每一个TCP类型的Kubernetes Service,kube-proxy都会在本地Node节点上建立一个SocketServer来负责接收请求，然后均匀发送到后端某个Pod的端口上。这个过程默认采用Round Robin负载均衡算法。

* kube-proxy在运行过程中动态创建与Service相关的Iptables规则，这些规则实现了ClusterIp及NodePort的请求流量重定向到kube-proxy进程上对应服务的代理端口功能。

* kube-proxy通过查询和监听API Server 中Service与Endpoints的变化，为每个Service都建立一个“服务代理对象”，并自动同步。服务代理对象是kube-proxy程序内部的一种数据结构，它包括一个用于监听此服务请求的SockerServer,SocketServer的端口是随机选择一个本地空闲端口。此外，kube-proxy内部创建了一个负载均衡器-LoadBalancer.

* 针对发生变化的Service列表，kube-proxy会逐个处理：
  a. 如果没有设置集群IP，则不做任何处理，否则，取该Service的所有端口定义列表。
  b.为Service端口分配服务代理对象并为该Service创建相关的Iptables规则。
  c.更新负载均衡器组件中对应Service的转发地址列表

* kube-proxy在启动时和监听到Service或Endpoint的变化后，会在本机Iptables的NAT表中添加4条规则链。
  a.KUBE-PORTALS-CONTAINER: 从容器中通过Cluster IP 和端口号访问service.
  b.KUBE-PORTALS-HOST: 从主机中通过Cluster IP 和端口号访问service.
  c.KUBE-NODEPORT-CONTAINER:从容器中通过NODE IP 和端口号访问service.
  d. KUBE-NODEPORT-HOST:从主机中通过Node IP 和端口号访问service.

**7、设想一个一千台物理机，上万规模的容器的kubernetes集群，请详述使用kubernetes时需要注意哪些问题？应该怎样解决？（提示可以从高可用，高性能等方向，覆盖到从镜像中心到kubernetes各个组件等）**

[官方资料](https://kubernetes.io/zh/docs/setup/best-practices/cluster-large/)

**8、设想Kubernetes集群管理从一千台节点到五千台节点，可能会遇到什么样的瓶颈，应该如何解决？**

[资料](https://juejin.im/entry/5b6d55d2e51d4519475f91c8)

**9、Kubernetes的运营中有哪些注意的要点。集群发生雪崩的条件，以及预防手段。**

[资料](https://dbaplus.cn/news-141-2139-1.html)

**10、Sidecar的设计模式如何在Kubernetes中进行应用，有什么意义？**

[资料](https://www.servicemesher.com/blog/sidecar-design-pattern-in-microservices-ecosystem/)

**11、灰度发布是什么，如何使用Kubernetes现有的资源实现灰度发布？**

[资料1](https://zhaohuabing.com/2017/11/08/istio-canary-release/)

[资料2](https://blog.csdn.net/wangyinghong_2013/article/details/78650290)



