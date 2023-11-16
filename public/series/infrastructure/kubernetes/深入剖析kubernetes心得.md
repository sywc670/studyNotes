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

