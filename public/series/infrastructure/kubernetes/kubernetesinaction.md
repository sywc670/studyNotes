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

### 乐观锁机制

当想要更新一个数据时，会先读取该数据，同时读取到version number，然后在提交数据前检查version number是否改变，如果改变了就重新开始，没有就提交

All Kubernetes resources include a `metadata.resourceVersion` field, which clients need to pass back to the API server when updating an object. If the version doesn’t
match the one stored in etcd, the API server rejects the update.

这个机制用在apiserver更新etcd中存储对象的字段时

### quotas

## material