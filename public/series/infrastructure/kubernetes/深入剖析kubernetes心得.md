### namespace cgroups

namespace和cgroups都是对linux原生进程就存在的机制，可以对每个进程都实施如此的限制，docker容器要限制也可以在docker run的时候用参数限制

mount namespace只是隔离了文件系统，新创建的容器可以直接看到原有的文件系统，只有将rootfs挂载后，再chroot才能够让容器真正隔离

### dockerfile

docker inspect看到的层是rootfs，每一层都是rootfs的一部分，这部分表现在`容器运行时目录`中，就是只读层，目录中还会加上init层和读写层

只读层就算是在`容器镜像目录`中的编号也与在inspect中看到的编号不一样

Dockerfile 中的每个原语执行后，都会生成一个对应的镜像层。即使原语本身并没有明显地修改文件的操作（比如，ENV 原语），它对应的层也会存在。只不过在外界看来，这个层是空的。

>在docker desktop里可以看到确实如此，每个命令都有一层，但是与docker inspect命令看到层数不同，可能是因为inspect只算了其中一些命令的层，ENV命令不算，RUN命令算

docker容器进程可以在宿主机上看到并查看信息，故可以exec

### headless service

**clusterIP与headless服务的区别**

clusterIP 
- 有VIP 访问VIP会随机访问pod 
- 访问域名会解析到VIP随机访问pod 
- 没有pod的域名

headless 
- 无VIP 
- 访问服务域名会解析出选中pod的域名解析并返回列表 
- 可以访问pod的域名

### 版本控制

**StatefulSet 与 DaemonSet 都使用 ControllerRevision 进行pod的版本管理**，但是ControllerRevision并不管理pod，所以与replicaset不同，ControllerRevision的owner是daemonset这些

### volume 两阶段处理

1. attach理解为为虚拟机添加一块硬盘连接上去，没有执行挂载命令mount，只是能看到硬盘
2. mount理解为在格式化之后在宿主机上挂载，但是没有挂载到pod中，该目录为`/var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>`
3. 之后需要kubelet通过cri来挂载，也就是docker来实现挂载

远程文件存储如NFS不需要attach步骤

attach阶段由volume controller负责，是controller manager一部分，具体是volume controller中的`AttachDetachController`控制循环实现

>PV与PVC绑定也由volume controller负责，是其中的`PersistentVolumeController`控制循环

mount阶段由kubelet负责，是`VolumeManagerReconciler`控制循环，是一个独立于主循环的goroutine

### storageclass

只要sc名字匹配，就会进行匹配，而不是实际存在这样的sc，这样使用sc可以将pvc和pv对应起来，即使不使用dynamic provisioning

将一个 PV 与 PVC 进行“`绑定`”，其实就是将这个 PV 对象的名字，填在了 PVC 对象的 `spec.volumeName` 字段上

如果你的集群有一个默认的sc，也就是annotation里面标注为默认，它就会为 PVC 和 PV 自动添加一个默认的 StorageClass
如果没有默认的sc，又没有设置PVC对应的sc，PVC 的 storageClassName 的值就是“”，这也意味着它只能够跟 storageClassName 也是“”的 PV 进行绑定。

sc不仅是可以dynamic provisioning、static provisioning，还可以处理PV和PVC的绑定关系

### local PV

常规PV都是先调度pod到某个节点上，然后再绑定挂载PV，但是local PV不行，因为local PV要用到宿主机上额外的硬盘，只有特定节点才有这样的硬盘，所以需要在调度pod的时候考虑PV分布

local PV其实还是PV类型，需要加上local字段，以及nodeAffinity指向当前节点，如此调度器才能正确调度

pod的PVC不能在创建出来之后就绑定好一个local PV，因为pod的nodeAffinity还没有考虑，所以得推迟这个绑定过程，使用sc的特性volumeBindingMode=WaitForFirstConsumer

通过这个延迟绑定机制，原本实时发生的 PVC 和 PV 的绑定过程，就被延迟到了 Pod 第一次调度的时候在调度器中进行，从而保证了这个绑定结果不会影响 Pod 的正常调度。在具体实现中，调度器实际上维护了一个与 Volume Controller 类似的控制循环，专门负责为那些声明了“延迟绑定”的 PV 和 PVC 进行绑定工作

### 存储插件

csi的driver字段与storageclass的provisioner一致，为csi插件的名字，遵守`反向 DNS`格式

flexvolume和csi都是与pv的实现有关的，在定义PV时才会有存储插件的字段

#### flexvolume

无论是 FlexVolume，还是 Kubernetes 内置的其他存储插件，它们实际上担任的角色，仅仅是 Volume 管理中的“Attach 阶段”和“Mount 阶段”的具体执行者。而像 Dynamic Provisioning 这样的功能，就不是存储插件的责任，而是 Kubernetes 本身存储管理功能的一部分。

attach阶段未知

mount阶段的工作原理：

kubelet 的 VolumeManagerReconcile 控制循环调用flexvolume的一个函数，再调用宿主机上的插件目录中的插件例如`/usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs`，这个插件只要接受传来的参数，即flexvolume类型的PV的一些参数，进行对应的mount操作，然后返回json格式的消息即可，该插件可以用任意形式的语言实现

总之，flexvolume简单但局限性很大，每次调用都是独立的

![](/reference/pic/vol.webp)

#### csi

相比之下，CSI 插件体系的设计思想，就是把这个 **Provision 阶段，以及 Kubernetes 里的一部分存储管理功能**，从主干代码里剥离出来，做成了几个单独的组件。这些组件会通过 Watch API 监听 Kubernetes 里与存储相关的事件变化，比如 PVC 的创建，来执行具体的存储管理动作。

Provision 等价于“创建磁盘”，Attach 等价于“挂载磁盘到虚拟机”，Mount 等价于“将该磁盘格式化后，挂载在 Volume 的宿主机目录上”。

![](/reference/pic/csi.webp)

external components:

- driver registrar负责注册csi信息到kubelet，调用csi identity获取信息
- external provisioner负责provision，监听apiserver的pvc对象，调用csi controller创建对应的csi volume
- external attacher负责attach，监听apiserver的*volumeattachment*对象，创建该对象会触发调用csi controller进行attach操作

kubelet的VolumeManagerReconciler控制循环负责mount，通过pkg/volume/csi 包，直接调用 CSI Node 服务完成 Volume 的“Mount 阶段”

VolumeAttachment 对象是 Kubernetes 确认一个 Volume 可以进入“Attach 阶段”的重要标志，包含PV、宿主机、存储插件的名字

在有了csi插件后，是在原有流程上分出了支流：

- 当 master 节点的 controller 的 AttachDetachController 控制循环需要进行“Attach”操作时（“Attach 阶段”），它实际上会执行到 pkg/volume/csi 目录中，创建一个 VolumeAttachment 对象，从而触发 External Attacher 调用 CSI Controller 服务的 ControllerPublishVolume 方法。
- 当 kubelet 中的 VolumeManagerReconciler控制循环 需要进行“Mount”操作时（“Mount 阶段”），它实际上也会执行到 pkg/volume/csi 目录中，直接向 CSI Node 服务发起调用 NodePublishVolume 方法的请求。

custom components (grpc协议):

- csi identity提供名字、是否实现attach、mount的能力、健康状况等信息
- csi controller负责provision和attach两个阶段的实现
- csi node负责mount阶段的实现，先格式化挂载到staging目录，然后再挂载到宿主机对应目录

>下面的csi插件指custom components

rook会把driver registrar和csi插件放在demonset中，把provisioner和attacher还有resizer等容器和csi插件放在deployment或statefulset里

部署csi插件的常用原则：

- demonset包含driver registrar和csi插件，起到注册和实现mount阶段作用，这两个功能都与kubelet相关，所以每个节点都需要有
- statefulset包含external provisioner、external attacher和csi插件，使得集群中只存在一个，为 External Components 提供 CSI Controller 服务

>注意csi node是以容器形式部署的，所以执行的挂载操作只能在容器的mount namespace，如何挂载到宿主机上呢？需要将宿主机的`/var/lib/kubelet` 以 Volume 的方式挂载进 CSI 插件容器的同名目录下，然后设置这个 Volume 的 `mountPropagation=Bidirectional`，即开启双向挂载传播，从而将容器在这个目录下进行的挂载操作“传播”给宿主机

>用statefulset而不是deployment，是因为 StatefulSet 需要确保应用拓扑状态的稳定性，所以它对 Pod 的更新，是严格保证顺序的，即：只有在前一个 Pod 停止并删除之后，它才会创建并启动下一个 Pod。而deployment只负责有多少个pod存在。

### 网络

linux Network Namespace 中存在的网络栈包含：网卡eth0、回环设备lo、iptables、路由表

虚拟网卡一旦插在网桥上，就失去了处理数据包的资格，由网桥进行处理

网桥是二层设备，根据 MAC learning table 或者 forwarding database (**FDB**)查找端口和 MAC 地址的对应关系

docker0网桥有自己的ip地址，可以ping，原因主要是为了方便管理，如果没有ip地址，无法进行远程管理

#### flannel

UDP、vxlan、host-gw

##### UDP模式

![](/reference/pic/flannel1.webp)

TUN(tunnel设备)：在操作系统内核和用户应用程序之间传递 IP 包，是三层设备

flannel使用etcd维护宿主机IP地址与容器子网地址的映射关系，每个宿主机上运行flanneld进程，监听固定udp端口

docker0 网桥的地址范围必须是 Flannel 为宿主机分配的子网 `dockerd --bip=$FLANNEL_SUBNET`，如果是kubenetes则`kubeadm init --pod-network-cidr=10.244.0.0/16`

性能问题：多次切换内核态、用户态

##### VXLAN模式

![](/reference/pic/flannel2.webp)

而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。

vtep的地址是容器子网网络地址例如10.1.15.0，但是并不表示网段，它既有 IP 地址，也有 MAC 地址

vtep和udp模式的flanneld类似，但是解封装对象是二层数据帧，且全程在内核态进行

首先明确一点：mac地址只用于同子网下的通信，如果不是直接网桥相通，不能获取对方的mac地址，而是填入网关mac地址。一种情况例外，就是隧道封装vxlan，vtep向另一个vtep通信时会直接填入目标vtep mac地址

容器跨主机通信流程重点：

1. 先要寻址目标IP地址，这是容器ip地址，目标mac地址填入docker0网桥，先过docker0网桥，到达宿主机，宿主机路由表匹配目标ip，从而发往对应的vtep，这里已经进行寻址操作了，这里目标mac地址是该vtep的mac地址
2. 找到了对应的源vtep设备，需要找到目标vtep设备mac地址，该地址已经被flanneld写入arp表中
3. 下一步完成mac层封装，加入vni头，这是VTEP设备识别某个数据帧是不是应该归自己处理的重要标识
4. 最后作为内容封装到一个外部的udp包里，此时需要找到目标主机的ip地址，通过vtep上的fdb查找对应目标vtep mac的条目

>对于每一个容器子网ip维护对应的vtep设备，比如，当 Node 2 启动并加入 Flannel 网络之后，在 Node 1（以及所有其他节点）上，flanneld 就会添加一条对应的路由规则

>寻找目标vtep的mac地址要用到的 ARP 记录，也是 flanneld 进程在 Node 2 节点启动时，自动添加在 Node 1 上的。并不依赖 L3 MISS 事件和 ARP 学习，而会在每台节点启动时把它的 VTEP 设备对应的 ARP 记录，直接下放到其他每台宿主机。

>vtep扮演一个网桥的角色，进行udp转发，根据其fdb的条目，绑定了主机ip地址与目的vtep mac地址，这个条目也是flanneld维护

总结：vxlan需要flanneld维护容器子网ip地址、vtep mac地址、宿主机ip地址的对应关系

![帧结构](https://static001.geekbang.org/resource/image/8c/85/8cede8f74a57617494027ba137383f85.jpg?wh=1864*192)

##### host-gw

![](/reference/pic/flannel3.webp)

保留docker0或者cni0网桥，在宿主机上创建路由规则，根据下一跳地址来决定发送给谁，也就是目标mac地址填写目标主机，但是目标ip地址还是容器地址

这样必须保证二层网络连通才行，因为寻址要靠mac地址

flannel只需要在etcd中维护容器子网和宿主机地址条目，及时更新路由规则

性能比隧道机制好

Flannel host-gw 模式使用 CNI 网桥的主要原因，其实是为了跟 VXLAN 模式保持一致。否则的话，Flannel 就需要维护两套 CNI 插件了。

#### cni

二层网络连通是指发送mac层数据包可以寻址到对方，而不依赖ip协议

只要所有宿主机属于同一个网段，比如192.168.1.*/24。那么它们就是两层互通的。

二层互通是查mac表，三层互通是查路由表

cni插件与网络方案不同：cni插件是为 Infra 容器的 Network Namespace，配置符合预期的网络栈，而网络方案是利用cni插件的网络栈实现跨主通信的

在部署 Kubernetes 的时候，有一个步骤是安装 kubernetes-cni 包，它的目的就是在宿主机上安装 CNI 插件所需的基础可执行文件。

/opt/cni/bin 目录的基础可执行文件：

第一类，叫作 Main 插件，它是用来创建具体网络设备的二进制文件。

第二类，叫作 IPAM（IP Address Management）插件，它是负责分配 IP 地址的二进制文件。

第三类，是由 CNI 社区维护的内置 CNI 插件。

flannel的cni插件被内置了，其余网络方案通过damonset挂载/opt/cni/bin目录可以加载自己的插件

flanneld启动后会在每台宿主机上生成它对应的` CNI 配置文件`（它其实是一个 ConfigMap），从而告诉 Kubernetes，这个集群要使用 Flannel 作为容器网络方案

处理容器网络相关的逻辑并不会在 kubelet 主干代码里执行，而是会在具体的 CRI实现里完成。对于 Docker 项目来说，它的 CRI 实现叫作 dockershim

当 kubelet 组件需要创建 Pod 的时候，它第一个创建的一定是 Infra 容器。dockershim 就会先调用 Docker API 创建并启动 Infra 容器，为 CNI 插件准备参数，然后调用 CNI 插件为 Infra 容器配置网络。

这里要调用的 CNI 插件，就是 /opt/cni/bin/flannel；而调用它所需要的参数，分为两部分。

第一部分，是**由 dockershim 设置的一组 CNI 环境变量**。最重要的环境变量参数叫作：CNI_COMMAND。它的取值只有两种：ADD 和 DEL。这个 ADD 和 DEL 操作，就是 CNI 插件唯一需要实现的两个方法。

其中 ADD 操作的含义是：把容器添加到 CNI 网络里；DEL 操作的含义则是：把容器从 CNI 网络里移除掉。而对于网桥类型的 CNI 插件来说，这两个操作意味着把容器以 Veth Pair 的方式“插”到 CNI 网桥上，或者从网桥上“拔”掉。

第二部分，则是**dockershim 从 CNI 配置文件里加载到的、默认插件的配置信息**。

flannel的cni插件实际上只会对第二部分做补充，然后将两部分参数传给/opt/cni/bin/bridge，bridge会根据参数检查并创建网桥，创建veth pair并进行相关配置，调用ipam插件进行容器地址分配，为 CNI 网桥添加 IP 地址

#### calico

![](/reference/pic/calico.webp)

正常模式和flannel的host-gw类似，根据下一跳寻址，但不需要cni0网桥，使用bgp

在了解了 BGP 之后，**Calico 项目的架构就非常容易理解了。它由三个部分组成**：
Calico 的 `CNI` 插件。这是 Calico 与 Kubernetes 对接的部分。我已经在上一篇文章中，和你详细分享了 CNI 插件的工作原理，这里就不再赘述了。
`Felix`。它是一个 DaemonSet，负责在宿主机上插入路由规则（即：写入 Linux 内核的 `FIB 转发信息库`），以及维护 Calico 所需的网络设备等工作。
`BIRD`。它就是 BGP 的客户端，专门负责在集群里`分发路由规则信息`。

可以看到，Calico 的 CNI 插件会为每个容器设置一个 `Veth Pair` 设备，然后把其中的一端放置在宿主机上（它的名字以 `cali` 前缀开头）。
此外，**由于 Calico 没有使用 CNI 的网桥模式，Calico 的 CNI 插件还需要在宿主机上为每个容器的 Veth Pair 设备配置一条路由规则，用于接收传入的 IP 包。**

其中，**这里最核心的“`下一跳`”路由规则，就是由 Calico 的 Felix 进程负责维护的。这些`路由规则信息`，则是通过 BGP Client 也就是 BIRD 组件，使用 BGP 协议传输而来的。**