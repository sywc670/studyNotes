### affinity

pod affinity: 

在pod配置文件中，用于管理pod与其它pod的关系，有required和preferred两种，另有pod antiaffinity

node affinity:

在pod配置文件中，用于管理pod与node的关系，有required和preferred两种

### etcd

etcd 的高可用方案有下面这 3 种思路：

1. 使用独立的 etcd 集群，独立的 etcd 集群自带高可用能力。
2. 在每个 Master 节点上，使用 Static Pod 来部署 etcd，多个节点上的 etcd 实例数据相互同步。每个 kube-apiserver 只与本 Master 节点的 etcd 通信。
3. 使用 CoreOS 提出的 self-hosted 方案，将 etcd 集群部署在 kubernetes 集群中，通过 kubernetes 集群自身的容灾能力来实现 etcd 的高可用。