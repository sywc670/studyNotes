### 运行模式

[ref](https://cloud.tencent.com/developer/article/1444664)

- CLIENT
CLIENT表示consul的client模式，就是客户端模式。是consul节点的一种模式，这种模式下，所有注册到当前节点的服务会被转发到SERVER，本身是不持久化这些信息。

- SERVER
SERVER表示consul的server模式，表明这个consul是个server，这种模式下，功能和CLIENT都一样，唯一不同的是，它会把所有的信息持久化的本地，这样遇到故障，信息是可以被保留的。

- SERVER-LEADER
中间那个SERVER下面有LEADER的字眼，表明这个SERVER是它们的老大，它和其它SERVER不一样的一点是，它需要负责同步注册的信息给其它的SERVER，同时也要负责各个节点的健康监测。

### 服务注册原理

consul启动时加入config路径目录参数，该目录下文件是json格式，写有各个服务信息，可以直接访问consul http接口来查询

### kv存储

可以通过访问http接口来获取kv值，相当于一个集中的键值配置中心