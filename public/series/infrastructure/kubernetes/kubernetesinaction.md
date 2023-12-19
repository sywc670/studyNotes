# kubernetes in action

## takeaway

### 获取集群组件状态

`kubectl get componentstatuses`被废弃了，可以改用`curl -k https://localhost:6443/livez?verbose`，或者`kubectl get --raw='/readyz?verbose'`

healthz接口被废弃，readyz和livez接口可以互换使用

### kubernetes组件高可用

etcd和apiserver可以多个同时运行，但是scheduler、controller manager无法多个同时运行，同一时间只能一个运行，其余处于standby模式

etcd使用raft来保证多数的节点可以有决定权

```shell
# 获取pod并以特定格式显示
kubectl get po -o custom-columns=POD:metadata.name,NODE:spec.nodeName --sort-by spec.nodeName -n kube-system
```

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



### quotas

## material