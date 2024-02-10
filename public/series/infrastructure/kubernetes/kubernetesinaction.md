# kubernetes in action

### 获取集群组件状态

`kubectl get componentstatuses`被废弃了，可以改用`curl -k https://localhost:6443/livez?verbose`，或者`kubectl get --raw='/readyz?verbose'`

healthz接口被废弃，readyz和livez接口可以互换使用

### apiserver做了什么

通过apiserver的请求，需要经过认证、鉴权、准入插件，最后检查object并存放进etcd

>当请求只是读取的时候，不会经过admission webhook

apiserver除此之外不做任何事情

#### admission webhook 

实际应用有：
1. 设置默认的serviceaccount
2. resourceQuota，确保只能使用一定量的资源
3. override一些字段，强制使用某些规则

#### 乐观锁机制

当想要更新一个数据时，会先读取该数据，同时读取到version number，然后在提交数据前检查version number是否改变，如果改变了就重新开始，没有就提交

All Kubernetes resources include a `metadata.resourceVersion` field, which clients need to pass back to the API server when updating an object. If the version doesn’t
match the one stored in etcd, the API server rejects the update.

这个机制用在apiserver更新etcd中存储对象的字段时

#### watch机制

controller manager和scheduler会使用watch机制，与apiserver建立长连接，当watch的对象变动时由apiserver通知，不需要一直轮询，但是controller还是会周期性的re-list防止漏掉信息

### scheduler

可以替换默认的scheduler，或者为pod指定自定义的scheduler，甚至不使用scheduler手动调度

scheduler watch没有设置nodeName的pod，然后进行调度，一般先predicate，然后priorities

### kubelet

kubelet一般从apiserver读取到yaml然后部署pod，但也可以从本地读取(static pod)

### kube-dns

容器设置/etc/resolv.conf的nameserver为kube-dns的service VIP，该service背后的pod就是dns服务器，且使用watch机制监听apiserver，更新dns条目，**但该更新不会瞬间完成，更新未完成时不会生效**

### kube-proxy

容器的IP地址段在一个集群中必须是唯一的，不能重复

kube-proxy会watch service和endpoint对象，调整iptables

### 高可用

#### 应用高可用

使用deployment来管理pod，但是pod fail之后还是会有downtime

可以不在代码中进行选主的操作，而使用sidecar的方式，将leader-election放在外部。

leader-election可以是leader负责所有工作，worker待命；也可以是leader负责写，worker只能读。

#### 控制平面高可用

etcd和apiserver可以多个同时运行，但是scheduler、controller manager无法多个同时运行，同一时间只能一个运行，其余处于standby模式，这个机制是组件自己已经实现的，选举选项是默认打开的

etcd使用raft来保证多数的节点可以有决定权

apiserver是stateless的，一种部署方案就是apiserver只和本地的etcd进行通信，为了确保apiserver高可用，可以在前面加入loadbalancer

```shell
# 获取pod并以特定格式显示
kubectl get po -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system
```

scheduler、controller manager的leader election：

使用一个configmap标识，control-plane.alpha.kubernetes.io/leader的annotation会标识谁是leader


### serviceaccount

只能在创建pod的时候赋值，不能修改

### hostport

不会使用整个宿主机的网络栈，而是相当于一个portforward的作用，在宿主机上会生成iptables转发，到达该port的请求转发到容器中

>It’s important to understand that if a pod is using a specific host port, only one instance of the pod can be scheduled to each node, because two processes can’t bind to the same host port. 

>Initially, people also used it to ensure two replicas of the same pod were never scheduled to the same node, but now you have a better way of achieving this

### securityContext

To get full access to the node’s kernel, the pod’s container runs in privileged mode.

如果runasuser和容器镜像内设定的不一致，不会允许该pod调度

为了让两个以不同用户运行的pod可以共享volume，可以定义fsGroup或者supplementalGroups，创建文件的用户组就会是共同定义的组

PodSecurityPolicy被废弃，替代品为PodSecurityAdmission

>`PodSecurityPolicy` is a cluster-level (non-namespaced) resource, which defines what security-related features users can or can’t use in their pods. The job of upholding the policies configured in PodSecurityPolicy resources is performed by the `PodSecurityPolicy admission control plugin` running in the API server


kubectl可以创建用户的context：kubectl config set-credentials alice --username=alice --password=password

### limit request

内存限制在容器内的应用看来是不存在的，只能看到整个node的内存量，cpu限制同理，应用可以看到所有cpu核，只是分配的cpu时间片变化了

You may want to use the `Downward API` to pass the CPU limit to the container and use it instead of relying on the number of CPUs your app can see on the system. You can also tap into the cgroups system directly to get the configured CPU limit by reading the following files:

- /sys/fs/cgroup/cpu/cpu.cfs_quota_us
- /sys/fs/cgroup/cpu/cpu.cfs_period_us

#### limitrange resourcequota

limitrange可以规定默认的limit和request，没有规定的pod会自动携带上，通过admission plugin实现，也可以规定范围检查

resourcequota针对整个命名空间的limit和request总量，而limitrange只规定单个pod

### kubeconfig

有cluster、user和context、当前context名称四大部分，其中context是包含了cluster、user、namespace对应关系的配置

其中user包含了用户名和登录credentials

### custom object

custom object，crd，可以没有对应的controller来控制，像configmap一样，可以被其他资源查找使用不需要controller

### Cluster Autoscaler

The Cluster Autoscaler takes care of automatically provisioning additional nodes when it notices a pod that can’t be scheduled to existing nodes because of a lack of resources on those nodes. It also de-provisions nodes when they’re underutilized for longer periods of time.

### PodDisruptionBudget

为了防止进行节点缩容时影响服务，定义该资源，规定pod必须有一定数量，如果缩容会影响，则不能进行该操作

### Vertical Pod Autoscaler (VPA)

Autoscaling is configured with a Custom Resource Definition object called VerticalPodAutoscaler. It allows to specify which pods should be vertically autoscale as well as if/how the resource recommendations are applied.

[ref](https://foxutech.medium.com/vertical-pod-autoscaler-vpa-know-everything-about-it-6a2d7a383268)

### 调度

taint、affinity(node, pod, anti)

#### taint

Each taint has an effect associated with it. Three possible effects exist: 

- NoSchedule, which means pods won’t be scheduled to the node if they don’t tolerate the taint.
- PreferNoSchedule is a soft version of NoSchedule, meaning the scheduler will try to avoid scheduling the pod to the node, but will schedule it to the node if it can’t schedule it somewhere else.
- NoExecute, unlike NoSchedule and PreferNoSchedule that only affect scheduling, also affects pods already running on the node. If you add a NoExecute taint to a node, pods that are already running on that node and don’t tolerate the NoExecute taint will be evicted from the node.


#### node affinity

Node selectors will eventually be deprecated, so it’s important you understand the new node affinity rules.

node selector是node有某个label，pod有这个label的selector就可以调度上去，功能比较局限

`preferredDuringSchedulingIgnoredDuringExecution`这个规则使得node affinity可以在调度时有优先级，先考虑某些节点，如果不行再考虑其他节点，多个preference可以有多个优先级，但是**这个优先级只是调度器参考的一部分**，如果还有其他影响调度的因素如：调度器默认会将pod分散到不同的节点，这个优先级不一定完全生效

>The reason is that besides the node affinity prioritization function, the Scheduler also uses other prioritization functions to decide where to schedule a pod. One of those is the SelectorSpreadPriority function, which makes sure pods belonging to the same ReplicaSet or Service are spread around different nodes so a node failure won’t bring the whole ser-vice down. 

#### pod affinity

>What’s interesting is that if you now delete the backend pod, the Scheduler will schedule the pod to node2 even though it doesn’t define any pod affinity rules itself (the rules are only on the frontend pods). This makes sense, because otherwise if the backend pod were to be deleted by accident and rescheduled to a different node, the fron-tend pods’ affinity rules would be broken.

pod affinity会使得双方pod都会调度到一个节点上，注意是调度器会考虑双方

topologyKey不仅可以让双方pod在同一节点：topologyKey: kubernetes.io/hostname

还可以在同一区域failure-domain.beta.kubernetes.io/zone，同一地域failure-domain.beta.kubernetes.io/region

这个topologyKey生效的原理是：由于pod affinity也是将pod调度到node上，所以会先用topologyKey选出有该key的node，这些node如果有满足affinity的pod，就会调度在这些node上

pod affinity也可以设置preference而不是hard requirement

