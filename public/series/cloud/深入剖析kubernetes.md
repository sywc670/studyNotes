- [docker 篇](#docker-篇)
  - [namespace cgroups](#namespace-cgroups)
  - [mount namespace rootfs](#mount-namespace-rootfs)
  - [dockerfile](#dockerfile)
  - [docker exec如何进入容器](#docker-exec如何进入容器)
  - [volume](#volume)
  - [linux 容器](#linux-容器)
- [kuberntes篇](#kuberntes篇)
  - [k8s 架构](#k8s-架构)
  - [k8s 部署 kubeadm](#k8s-部署-kubeadm)
  - [pod](#pod)
  - [volume projected 类型](#volume-projected-类型)
  - [probe](#probe)
  - [podpreset 废弃](#podpreset-废弃)
  - [deployment](#deployment)
  - [statefulset](#statefulset)
    - [例子 mysql集群容器化](#例子-mysql集群容器化)
    - [滚动更新](#滚动更新)
  - [daemonset](#daemonset)
  - [job cronjob](#job-cronjob)
  - [声明式API Initializer](#声明式api-initializer)
    - [crd](#crd)
    - [控制器](#控制器)
  - [RBAC](#rbac)
  - [operator](#operator)
  - [PV PVC](#pv-pvc)
    - [volume被pod使用的处理流程](#volume被pod使用的处理流程)
  - [storageclass](#storageclass)
  - [Local PV](#local-pv)
  - [存储插件](#存储插件)
    - [flexvolume](#flexvolume)
    - [CSI](#csi)
  - [网络](#网络)
  - [flannel](#flannel)
    - [UDP模式](#udp模式)
    - [VXLAN模式](#vxlan模式)
  - [cni](#cni)
    - [网桥方案](#网桥方案)
    - [三层方案](#三层方案)
      - [flannel host-gw](#flannel-host-gw)
      - [calico网络](#calico网络)
      - [calico](#calico)
        - [node to node mesh](#node-to-node-mesh)
        - [route reflector](#route-reflector)
        - [三层模式限制](#三层模式限制)
        - [ipip](#ipip)
        - [设置网关BGP](#设置网关bgp)
        - [三层和隧道的异同](#三层和隧道的异同)
        - [networkpolicy](#networkpolicy)
  - [service](#service)
    - [NodePort](#nodeport)
    - [LoadBalancer](#loadbalancer)
    - [externalName](#externalname)
    - [externalIP](#externalip)
  - [ingress](#ingress)
  - [资源调度](#资源调度)
  - [调度器](#调度器)
    - [总结](#总结)
    - [gpu 支持](#gpu-支持)
    - [gpu 总结](#gpu-总结)
  - [kubelet cri](#kubelet-cri)
  - [kata container and gvisor](#kata-container-and-gvisor)
  - [prometheus](#prometheus)
    - [自定义监控指标](#自定义监控指标)
    - [hpa 总结](#hpa-总结)
  - [日志](#日志)
    - [总结](#总结-1)


# docker 篇

备注：之所以要强调 Linux 容器，是因为比如 Docker on Mac，以及 Windows Docker（Hyper-V 实现），实际上是基于**虚拟化技术**实现的，跟我们这个专栏着重介绍的 Linux 容器完全不同。

一组联合挂载在 `/var/lib/docker/aufs/mnt` 上的 **rootfs**，这一部分我们称为`“容器镜像”`（Container Image），是容器的静态视图；一个由 Namespace+Cgroups 构成的隔离环境，这一部分我们称为`“容器运行时”`（Container Runtime），是容器的动态视图。

## namespace cgroups

首先，既然容器只是运行在宿主机上的一种特殊的进程，那么多个容器之间使用的就还是`同一个宿主机的操作系统内核`。尽管你可以在容器里通过 Mount Namespace 单独挂载其他不同版本的操作系统文件，比如 CentOS 或者 Ubuntu，但这并不能改变共享宿主机内核的事实。这意味着，如果你要在 Windows 宿主机上运行 Linux 容器，或者在低版本的 Linux 宿主机上运行高版本的 Linux 容器，都是行不通的。

其次，在 Linux 内核中，有很多资源和对象是不能被 Namespace 化的，最典型的例子就是：**时间**。这就意味着，如果你的容器中的程序使用 `settimeofday(2)` 系统调用修改了时间，整个宿主机的时间都会被随之修改，这显然不符合用户的预期。

更为棘手的是，尽管在实践中我们确实可以使用 `Seccomp` 等技术，对容器内部发起的所有系统调用进行过滤和甄别来进行安全加固，但这种方法因为多了一层对系统调用的过滤，必然会拖累容器的性能。何况，默认情况下，谁也不知道到底该开启哪些系统调用，禁止哪些系统调用。所以，在生产环境中，没有人敢把运行在物理机上的 Linux 容器直接暴露到公网上。当然，我后续会讲到的基于虚拟化或者独立内核技术的容器实现，则可以比较好地在隔离与性能之间做出平衡。

一个进程，可以选择加入到某个进程已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的，这正是 docker exec 的实现原理。

Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等。此外，Cgroups 还能够对进程进行优先级设置、审计，以及将进程挂起和恢复等操作。

在 Linux 中，Cgroups 给用户暴露出来的操作接口是文件系统，即它以文件和目录的方式组织在操作系统的 `/sys/fs/cgroup` 路径下
可以看到，在 /sys/fs/cgroup 下面有很多诸如 cpuset、cpu、 memory 这样的子目录，也叫子系统。这些都是我这台机器当前可以被 Cgroups 进行限制的资源种类。

如果熟悉 Linux CPU 管理的话，你就会在它的输出里注意到 cfs_period 和 cfs_quota 这样的关键词。这两个参数需要组合使用，可以用来限制进程在长度为 cfs_period 的一段时间内，只能被分配到总量为 cfs_quota 的 CPU 时间。

除 CPU 子系统外，Cgroups 的每一个子系统都有其独有的资源限制能力，比如：blkio，为​​​块​​​设​​​备​​​设​​​定​​​I/O 限​​​制，一般用于磁盘等设备；cpuset，为进程分配单独的 CPU 核和对应的内存节点；memory，为进程设定内存使用的限制。

Linux Cgroups 的设计还是比较易用的，简单粗暴地理解呢，它就是一个子系统目录加上一组资源限制文件的组合。而对于 Docker 等 Linux 容器项目来说，它们只需要在每个子系统下面，为每个容器创建一个`控制组`（即创建一个新目录），然后在启动容器进程之后，把这个进程的 PID 填写到对应控制组的 `tasks` 文件中就可以了。

由于一个容器的本质就是一个进程，用户的应用进程实际上就是容器里 `PID=1` 的进程，也是其他后续创建的所有进程的`父进程`。这就意味着，在一个容器中，你没办法同时运行两个不同的应用，除非你能事先找到一个公共的 PID=1 的程序来充当两个不同应用的父进程，这也是为什么很多人都会用 systemd 或者 supervisord 这样的软件来代替应用本身作为容器的启动进程。

>容器是『单进程』模型，只有 PID=1 的进程才会被 Dockerd 控制，即 pid=1 的进程挂了 Dockerd 能够感知到，但是其它的进程却不受 `Dockerd` 的管理，当出现孤儿进程的时候，管理和调度是个问题

另外，跟 Namespace 的情况类似，Cgroups 对资源的限制能力也有很多不完善的地方，被提及最多的自然是  `/proc` 文件系统的问题。

众所周知，Linux 下的 /proc 目录存储的是记录当前内核运行状态的一系列特殊文件，用户可以通过访问这些文件，查看系统以及当前正在运行的进程的信息，比如 CPU 使用情况、内存占用率等，这些文件也是 top 指令查看系统信息的主要数据来源。但是，你如果在容器里执行 top 指令，就会发现，它显示的信息居然是宿主机的 CPU 和内存数据，而不是当前容器的数据。

造成这个问题的原因就是，/proc 文件系统并不知道用户通过 Cgroups 给这个容器做了什么样的资源限制，即：**/proc 文件系统不了解 Cgroups 限制的存在**。

`lxcfs`就是来实现这个功能的，做法是把宿主机的 `/var/lib/lxcfs/proc/memoinfo` 文件挂载到Docker容器的`/proc/meminfo`位置后。容器中进程读取相应文件内容时，LXCFS的FUSE实现会从容器对应的Cgroup中读取正确的内存限制。从而使得应用获得正确的资源约束设定。kubernetes环境下，也能用，以`ds` 方式运行 lxcfs ，自动给容器注入争取的 proc 信息。

>原理是不挂载宿主机的/proc目录，让lxcfs挂载cgroup正确限制
>注意是/proc不知道cgroup，但是挂载mount namespace是可以覆盖/proc的

你在查看 Docker 容器的 Namespace 时，是否注意到有一个叫 cgroup 的 Namespace？它是 Linux 4.6 之后新增加的一个 Namespace，你知道它的作用吗？

cgroupns作用
(1)可以限制容器的cgroup filesytem视图，使得在容器中也可以安全的使用cgroup；
(2)此外，会使容器迁移更加容易；在迁移时，/proc/self/cgroup需要复制到目标机器，这要求容器的cgroup路径是唯一的，否则可能会与目标机器冲突。有了cgroupns，每个容器都有自己的cgroup filesystem视图，不用担心这种冲突。

## mount namespace rootfs

mount namespace只隔离`增量`，不隔离`存量`: 如果不对容器（初始概念的容器）挂载额外的文件系统，那么即使使用了mount namespace也能看到宿主机的`文件系统`

例子：
`C语言创建子进程时开启 mount namespace` 编译：`gcc -o ns ns.c`
```c

#define _GNU_SOURCE
#include <sys/mount.h> 
#include <sys/types.h>
#include <sys/wait.h>
#include <stdio.h>
#include <sched.h>
#include <signal.h>
#include <unistd.h>
#define STACK_SIZE (1024 * 1024)
static char container_stack[STACK_SIZE];
char* const container_args[] = {
  "/bin/bash",
  NULL
};

int container_main(void* arg)
{  
  printf("Container - inside the container!\n");
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}

int main()
{
  printf("Parent - start a container!\n");
  int container_pid = clone(container_main, container_stack+STACK_SIZE, CLONE_NEWNS | SIGCHLD , NULL);
  waitpid(container_pid, NULL, 0);
  printf("Parent - container stopped!\n");
  return 0;
}
```
需要添加挂载操作才能不看到宿主机文件系统

```c

int container_main(void* arg)
{
  printf("Container - inside the container!\n");
  // 如果你的机器的根目录的挂载类型是shared，那必须先重新挂载根目录
  // mount("", "/", NULL, MS_PRIVATE, "");
  mount("none", "/tmp", "tmpfs", 0, "");
  // 进入容器执行自定义语句
  execv(container_args[0], container_args);
  printf("Something's wrong!\n");
  return 1;
}
```

这就是 Mount Namespace 跟其他 Namespace 的使用略有不同的地方：它对容器进程视图的改变，一定是伴随着挂载操作（`mount`）才能生效。


现在，你应该可以理解，对 Docker 项目来说，它**最核心的原理**实际上就是为待创建的用户进程：
```
启用 Linux Namespace 配置；
设置指定的 Cgroups 参数；
切换进程的根目录（Change Root）。
```

需要明确的是，rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。

实际上，同一台机器上的所有容器，都共享宿主机操作系统的内核。

这就意味着，如果你的应用程序需要配置内核参数、加载额外的内核模块，以及跟内核进行直接的交互，你就需要注意了：这些操作和依赖的对象，都是宿主机操作系统的内核，它对于该机器上的所有容器来说是一个“全局变量”，牵一发而动全身。

`mount -t aufs -o dirs=./A:./B none ./C`

Docker 会把这些增量联合挂载在一个统一的`挂载点`上（等价于前面例子里的“/C”目录）。这个挂载点就是 `/var/lib/docker/aufs/mnt/` 这部分内容可能过时了

镜像的层都放置在 `/var/lib/docker/aufs/diff` 目录下，然后被联合挂载在 /var/lib/docker/aufs/mnt 里面。

只读层都以增量的形式存放操作系统的一部分

>新版本的docker 镜像层是在：`/var/lib/docker/overlay2/id/diff` 运行容器时,这些镜像层是挂载在`/var/lib/docker/overlay2/id/merged`下的

>删除只读层的文件原理：实际上是在可读写层创建一个名为.wh.xxx的文件。这样，当这两个层被联合挂载之后，该文件就会被.wh.xxx遮挡起来，消失了。这个功能就是“ro+wh”的挂载方式，即只读+whiteout的含义。

![aufs](../../reference/pic/aufs.webp)

`init层`是一个以“-init”结尾的层，夹在只读层和读写层之间。Init 层是 Docker 项目单独生成的一个内部层，专门用来存放 /etc/hosts、/etc/resolv.conf 等信息。

需要这样一层的原因是，这些文件本来属于只读的 Ubuntu 镜像的一部分，但是用户往往需要在启动容器时写入一些指定的值比如 hostname，所以就需要在可读写层对它们进行修改。

可是，这些修改往往只对当前的容器有效，我们并不希望执行 `docker commit` 时，把这些信息连同可读写层一起提交掉。所以，Docker 做法是，在修改了这些文件之后，以一个单独的层挂载了出来。而用户执行 docker commit 只会提交可读写层，所以是不包含这些内容的。

>1，在使用 Linux 宿主机时，推荐使用 `OverlayFS`。这是因为 OverlayFS 是 Linux `内核`的一部分，并且在大多数发行版中已经默认启用，因此它具有广泛的支持和较好的稳定性。 
2，如果宿主机使用了 Btrfs 文件系统，则建议使用 Btrfs 子卷（subvolumes）来存储容器镜像和数据，以获得更好的性能和可管理性。 
3，如果使用较旧的 Linux 内核版本，则可能需要使用 AuFS。但是，需要注意的是，AuFS 的稳定性不如 OverlayFS，并且在某些发行版中可能需要手动安装。 
4，对于 Windows 或 macOS 宿主机，建议使用 Docker Desktop 自带的 Hyper-V 或者 HFS+ 文件系统的实现。这些实现经过了充分测试，并且提供了最佳的性能和兼容性。

>1. 上面的读写层通常也称为容器层，下面的只读层称为镜像层，所有的增删查改操作都只会作用在容器层，相同的文件上层会覆盖掉下层。知道这一点，就不难理解镜像文件的修改，比如修改一个文件的时候，首先会从上到下查找有没有这个文件，找到，就复制到容器层中，修改，修改的结果就会作用到下层的文件，这种方式也被称为`copy-on-write`。 
>2. 查了一下，包括但不限于以下这几种：aufs, device mapper, btrfs, overlayfs, vfs, zfs。aufs是ubuntu 常用的，device mapper 是 centos，btrfs 是 SUSE，overlayfs ubuntu 和 centos 都会使用，现在最新的 docker 版本中默认两个系统都是使用的 overlayfs，vfs 和 zfs 常用在 solaris 系统。

## dockerfile

**Dockerfile 中的每个原语执行后，都会生成一个对应的镜像层。即使原语本身并没有明显地修改文件的操作（比如，ENV 原语），它对应的层也会存在。只不过在外界看来，这个层是空的。**

这里的层可能并不对应docker inspect看到的层

另外，在使用 Dockerfile 时，你可能还会看到一个叫作 ENTRYPOINT 的原语。实际上，它和 CMD 都是 Docker 容器进程启动所必需的参数，完整执行格式是：“ENTRYPOINT CMD”。

但是，默认情况下，Docker 会为你提供一个**隐含的 ENTRYPOINT**，即：/bin/sh -c。所以，在不指定 ENTRYPOINT 时，比如在我们这个例子里，实际上运行在容器里的完整进程是：/bin/sh -c "python app.py"，即 CMD 的内容就是 ENTRYPOINT 的参数。备注：基于以上原因，我们后面会统一称 Docker 容器的启动进程为 ENTRYPOINT，而不是 CMD。

## docker exec如何进入容器

可以查看到容器在宿主机的进程号
```
$ docker inspect --format '{{ .State.Pid }}'  4ddf4638572d
25686
```

然后找到ns
```
$ ls -l  /proc/25686/ns
total 0
lrwxrwxrwx 1 root root 0 Aug 13 14:05 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 ipc -> ipc:[4026532278]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 mnt -> mnt:[4026532276]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 net -> net:[4026532281]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid -> pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid_for_children -> pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 uts -> uts:[4026532277]
```
可以看到，一个进程的每种 Linux Namespace，都在它对应的 `/proc/[进程号]/ns` 下有一个对应的虚拟文件，并且链接到一个`真实的` Namespace 文件上。

有了这样一个可以“hold 住”所有 Linux Namespace 的文件，我们就可以对 Namespace 做一些很有意义事情了，比如：加入到一个已经存在的 Namespace 当中。

这也就意味着：一个进程，可以选择加入到某个进程已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的，这正是 `docker exec` 的实现原理。

而这个操作所依赖的，乃是一个名叫 `setns()` 的 Linux 系统调用。它的调用方法，我可以用如下一段小程序为你说明：

```c
#define _GNU_SOURCE
#include <fcntl.h>
#include <sched.h>
#include <unistd.h>
#include <stdlib.h>
#include <stdio.h>

#define errExit(msg) do { perror(msg); exit(EXIT_FAILURE);} while (0)

int main(int argc, char *argv[]) {
    int fd;
    
    fd = open(argv[1], O_RDONLY);
    if (setns(fd, 0) == -1) {
        errExit("setns");
    }
    execvp(argv[2], &argv[2]); 
    errExit("execvp");
}
```
```
gcc -o set_ns set_ns.c
./set_ns /proc/25686/ns/net /bin/bash 
```
这段代码功能非常简单：它一共接收两个参数，第一个参数是 `argv[1]`，即当前进程要加入的 Namespace 文件的路径，比如 /proc/25686/ns/net；而第二个参数，则是你要在这个 Namespace 里运行的进程，比如 /bin/bash。

这段代码的`核心操作`，则是通过 open() 系统调用打开了指定的 Namespace 文件，并把这个文件的描述符 fd 交给 setns() 使用。在 setns() 执行后，当前进程就加入了这个文件对应的 Linux Namespace 当中了。

docker exec的进程将`共享`容器的namespace，即namespace的链接文件指向同一个namespace文件上


## volume

那么，Docker 又是如何做到把一个宿主机上的目录或者文件，`挂载`到容器里面去呢？
难道又是 Mount Namespace 的黑科技吗？实际上，并不需要这么麻烦。

我已经介绍过，当容器进程被创建之后，尽管开启了 Mount Namespace，但是在它执行 `chroot`（或者 `pivot_root`）之前，容器进程一直可以看到宿主机上的整个文件系统。

而宿主机上的文件系统，也自然包括了我们要使用的容器镜像。这个镜像的各个层，保存在 /var/lib/docker/aufs/diff 目录下，在容器进程启动后，它们会被联合挂载在 /var/lib/docker/aufs/mnt/ 目录中，这样容器所需的 rootfs 就准备好了。

所以，我们只需要在 rootfs 准备好之后，在执行 chroot 之前，把 Volume 指定的宿主机目录（比如 /home 目录），挂载到指定的容器目录（比如 /test 目录）在宿主机上对应的目录`（即 /var/lib/docker/aufs/mnt/[可读写层 ID]/test）`上，这个 Volume 的挂载工作就完成了。

更重要的是，由于执行这个挂载操作时，“`容器进程`”已经创建了，也就意味着此时 `Mount Namespace 已经开启了`。所以，这个挂载事件`只在这个容器里可见`。你在宿主机上，是看不见容器内部的这个挂载点的。这就保证了容器的隔离性不会被 Volume 打破。

>注意：这里提到的"容器进程"，是 Docker 创建的一个容器初始化进程 (`dockerinit`)，而不是应用进程 (`ENTRYPOINT + CMD`)。dockerinit 会负责完成根目录的准备、挂载设备和目录、配置 hostname 等一系列需要在容器内进行的初始化操作。最后，它通过 execv() 系统调用，让应用进程取代自己，成为容器里的 PID=1 的进程。

而这里要使用到的挂载技术，就是 Linux 的绑定挂载（`bind mount`）机制。它的主要作用就是，允许你将一个目录或者文件，而不是整个设备，挂载到一个指定的目录上。并且，这时你在该挂载点上进行的任何操作，只是发生在被挂载的目录或者文件上，而原挂载点的内容则会被隐藏起来且不受影响。

其实，如果你了解 Linux 内核的话，就会明白，绑定挂载实际上是一个 `inode 替换`的过程。在 Linux 操作系统中，inode 可以理解为存放文件内容的“对象”，而 `dentry`，也叫目录项，就是访问这个 inode 所使用的“指针”。

![](../../reference/pic/inode.webp)
正如上图所示，`mount --bind /home /test`，会将 /home 挂载到 /test 上。其实相当于将 /test 的 dentry，`重定向`到了 /home 的 inode。这样当我们修改 /test 目录时，实际修改的是 /home 目录的 inode。这也就是为何，一旦执行 umount 命令，/test 目录原先的内容就会恢复：因为修改真正发生在的，是 /home 目录里。

这样，进程在容器里对这个 /test 目录进行的所有操作，都实际发生在宿主机的对应目录（比如，/home，或者 `/var/lib/docker/volumes/[VOLUME_ID]/_data`）里，而不会影响容器镜像的内容。

那么，这个 /test 目录里的内容，既然挂载在容器 rootfs 的可读写层，它会不会被 `docker commit` 提交掉呢？也不会。

这个原因其实我们前面已经提到过。容器的镜像操作，比如 docker commit，都是发生在`宿主机空间`的。而由于 Mount Namespace 的隔离作用，宿主机并不知道这个绑定挂载的存在。

所以，在宿主机看来，容器中`可读写层`的 /test 目录（`/var/lib/docker/aufs/mnt/[可读写层 ID]/test`），始终是空的。不过，由于 Docker 一开始还是要创建 /test 这个目录作为挂载点，所以执行了 docker commit 之后，你会发现新产生的镜像里，会多出来一个`空的` /test 目录。毕竟，新建目录操作，又不是挂载操作，Mount Namespace 对它可起不到“障眼法”的作用。

## linux 容器

一个正在运行的 Linux 容器，其实可以被“一分为二”地看待：

一组联合挂载在 /var/lib/docker/aufs/mnt 上的 rootfs，这一部分我们称为“`容器镜像`”（Container Image），是容器的静态视图；

一个由 Namespace+Cgroups 构成的隔离环境，这一部分我们称为“`容器运行时`”（Container Runtime），是容器的动态视图。

# kuberntes篇

## k8s 架构

![](https://static001.geekbang.org/resource/image/8e/67/8ee9f2fa987eccb490cfaa91c6484f67.png?wh=1920*1080)

我们可以看到，Kubernetes 项目的架构，跟它的原型项目 Borg 非常类似，都由 Master 和 Node 两种节点组成，而这两种角色分别对应着控制节点和计算节点。

其中，`控制节点`，即 Master 节点，由三个紧密协作的独立组件组合而成，它们分别是负责 API 服务的 `kube-apiserver`、负责调度的 `kube-scheduler`，以及负责容器编排的 `kube-controller-manager`。整个集群的持久化数据，则由 kube-apiserver 处理后保存在 Etcd 中。

而`计算节点`上最核心的部分，则是一个叫作 `kubelet` 的组件。

在 Kubernetes 项目中，kubelet 主要负责同`容器运行时`（比如 Docker 项目）打交道。而这个交互所依赖的，是一个称作 `CRI`（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。

这也是为何，Kubernetes 项目并不关心你部署的是什么容器运行时、使用的什么技术实现，只要你的这个容器运行时能够运行标准的容器镜像，它就可以通过实现 CRI 接入到 Kubernetes 项目当中。

此外，kubelet 还通过 gRPC 协议同一个叫作**Device Plugin**的插件进行交互。这个插件，是 Kubernetes 项目用来管理 GPU 等宿主机物理设备的主要组件，也是基于 Kubernetes 项目进行机器学习训练、高性能作业支持等工作必须关注的功能。

而 kubelet 的另一个重要功能，则是调用网络插件和存储插件为容器配置网络和持久化存储。这两个插件与 kubelet 进行交互的接口，分别是 `CNI`（Container Networking Interface）和 `CSI`（Container Storage Interface）。

可以看到，Kubernetes 项目并没有像其他项目那样，为每一个管理功能创建一个指令，然后在项目中实现其中的逻辑。这种做法，的确可以解决当前的问题，但是在更多的问题来临之后，往往会力不从心。

相比之下，在 Kubernetes 项目中，我们所推崇的使用方法是：首先，通过一个“编排对象”，比如 Pod、Job、CronJob 等，来描述你试图管理的应用；然后，再为它定义一些“服务对象”，比如 Service、Secret、Horizontal Pod Autoscaler（自动水平扩展器）等。这些对象，会负责具体的平台级功能。

这种使用方法，就是所谓的“`声明式 API`”。这种 API 对应的“编排对象”和“服务对象”，都是 Kubernetes 项目中的 API 对象（API Object）。

实际上，过去很多的集群管理项目（比如 Yarn、Mesos，以及 Swarm）所擅长的，都是把一个容器，按照某种规则，放置在某个最佳节点上运行起来。这种功能，我们称为“`调度`”。

而 Kubernetes 项目所擅长的，是按照用户的意愿和整个系统的规则，完全自动化地处理好容器之间的各种关系。这种功能，就是我们经常听到的一个概念：`编排`。所以说，Kubernetes 项目的本质，是为用户提供一个具有普遍意义的容器编排工具。

>不能迁移“有状态”的容器，是因为迁移的是容器的rootfs，但是一些动态视图是没有办法伴随迁移一同进行迁移的。
就算是现在的k8s也无法迁移，而是重建

通过前面几篇文章的内容，我其实阐述了这样一个思想：要真正发挥容器技术的实力，你就不能仅仅局限于对 Linux 容器本身的钻研和使用。

这些知识更适合作为你的技术储备，以便在需要的时候可以帮你更快地定位问题，并解决问题。

而更深入地学习容器技术的关键在于，如何使用这些技术来“容器化”你的应用。比如，我们的应用既可能是 Java Web 和 MySQL 这样的组合，也可能是 Cassandra 这样的分布式系统。而要使用容器把后者运行起来，你单单通过 Docker 把一个 Cassandra 镜像跑起来是没用的。

要把 Cassandra 应用容器化的关键，在于如何处理好这些 Cassandra 容器之间的`编排关系`。比如，哪些 Cassandra 容器是主，哪些是从？主从容器如何区分？它们之间又如何进行自动发现和通信？Cassandra 容器的持久化数据又如何保持，等等。

## k8s 部署 kubeadm

将k8s部署在docker上会带来一个很麻烦的问题，即：如何容器化 `kubelet`。

除了跟容器运行时打交道外，kubelet 在配置容器网络、管理容器数据卷时，都需要直接操作`宿主机`。而如果现在 kubelet 本身就运行在一个容器里，那么直接操作宿主机就会变得很麻烦。

对于网络配置来说还好，kubelet 容器可以通过不开启 Network Namespace（即 Docker 的 host network 模式）的方式，直接共享宿主机的网络栈。可是，要让 kubelet 隔着容器的 Mount Namespace 和文件系统，操作宿主机的文件系统，就有点儿困难了。

在容器里运行 kubelet，依然没有很好的`解决办法`，我也不推荐你用容器去部署 Kubernetes 项目。正因为如此，kubeadm 选择了一种妥协方案：把 kubelet 直接运行在宿主机上，然后使用容器部署其他的 Kubernetes 组件。

`$ apt-get install kubeadm`

当你执行 `kubeadm init` 指令后，kubeadm 首先要做的，是一系列的检查工作，以确定这台机器可以用来部署 Kubernetes。这一步检查，我们称为“`Preflight Checks`”，它可以为你省掉很多后续的麻烦。

在通过了 **Preflight Checks** 之后，kubeadm 要为你做的，是生成 Kubernetes 对外提供服务所需的各种**证书**和对应的目录。

Kubernetes 对外提供服务时，除非专门开启“不安全模式”，否则都要通过 HTTPS 才能访问 kube-apiserver。这就需要为 Kubernetes 集群配置好证书文件。

kubeadm 为 Kubernetes 项目生成的证书文件都放在 Master 节点的 `**/etc/kubernetes/pki**` 目录下。在这个目录下，最主要的证书文件是 ca.crt 和对应的私钥 ca.key。

证书生成后，kubeadm 接下来会为其他组件生成**访问 kube-apiserver 所需的配置文件**。这些文件的路径是：/etc/kubernetes/xxx.conf

这些文件里面记录的是，当前这个 Master 节点的服务器地址、监听端口、证书目录等信息。这样，对应的客户端（比如 scheduler，kubelet 等），可以直接加载相应的文件，使用里面的信息与 kube-apiserver 建立安全连接。

> xxx.conf里私钥和公钥来进行认证

在 Kubernetes 中，有一种特殊的容器启动方法叫做**Static Pod**。它允许你把要部署的 Pod 的 YAML 文件放在一个指定的目录里。这样，当这台机器上的 kubelet 启动时，它会自动检查这个目录，加载所有的 Pod YAML 文件，然后在这台机器上启动它们。

从这一点也可以看出，kubelet 在 Kubernetes 项目中的地位非常高，在设计上它就是一个`完全独立`的组件，而其他 Master 组件，则更像是辅助性的系统容器。

在 kubeadm 中，Master 组件的 YAML 文件会被生成在 `/etc/kubernetes/manifests` 路径下。

Master 容器启动后，kubeadm 会通过检查 `localhost:6443/healthz` 这个 Master 组件的健康检查 URL，等待 Master 组件完全运行起来。

然后，kubeadm 就会为集群生成一个 **bootstrap token**。在后面，只要持有这个 token，任何一个安装了 kubelet 和 kubadm 的节点，都可以通过 `kubeadm join` 加入到这个集群当中。这个 token 的值和使用方法，会在 kubeadm init 结束后被打印出来。

在 token 生成之后，kubeadm 会将 ca.crt 等 Master 节点的重要信息，通过 `ConfigMap` 的方式保存在 Etcd 当中，供后续部署 Node 节点使用。这个 ConfigMap 的名字是 **cluster-info**。

kubeadm init 的最后一步，就是安装默认插件。Kubernetes 默认 `kube-proxy` 和 `DNS` 这两个插件是必须安装的。它们分别用来提供整个集群的服务发现和 DNS 功能。其实，这两个插件也只是两个容器镜像而已，所以 kubeadm 只要用 Kubernetes 客户端创建两个 Pod 就可以了。

可是，为什么执行 kubeadm join 需要这样一个 **token** 呢？

因为，任何一台机器想要成为 Kubernetes 集群中的一个节点，就必须在集群的 kube-apiserver 上注册。可是，要想跟 apiserver 打交道，这台机器就必须要获取到相应的证书文件（**CA 文件**）。可是，为了能够一键安装，我们就不能让用户去 Master 节点上手动拷贝这些文件。所以，kubeadm 至少需要发起一次“不安全模式”的访问到 kube-apiserver，从而拿到保存在 ConfigMap 中的 cluster-info（它保存了 APIServer 的授权信息）。而 bootstrap token，扮演的就是这个过程中的安全验证的角色。

只要有了 **cluster-info 里的 kube-apiserver 的地址、端口、证书**，kubelet 就可以以“安全模式”连接到 apiserver 上，这样一个新的节点就部署完成了。

我要指定 kube-apiserver 的启动参数，该怎么办？

在这里，我强烈推荐你在使用 kubeadm init 部署 Master 节点时，使用下面这条指令：` kubeadm init --config kubeadm.yaml`

然后，kubeadm 就会使用上面这些信息替换 `/etc/kubernetes/manifests/kube-apiserver.yaml` 里的 command 字段里的参数了。
而这个 YAML 文件提供的可配置项远不止这些。比如，你还可以修改 kubelet 和 kube-proxy 的配置，修改 Kubernetes 使用的`基础镜像的 URL`（默认的k8s.gcr.io/xxx镜像 URL 在国内访问是有困难的），指定自己的证书文件，指定特殊的容器运行时等等。这些配置项，就留给你在后续实践中探索了。

## pod

另外，在 Metadata 中，还有一个与 Labels 格式、层级完全相同的字段叫 `Annotations`，它专门用来携带 key-value 格式的`内部信息`。所谓内部信息，指的是对这些信息感兴趣的，是 Kubernetes 组件本身，而不是用户。所以大多数 Annotations，都是在 Kubernetes 运行过程中，被`自动`加在这个 API 对象上。

再次强调一下：容器的“单进程模型”，并不是指容器里只能运行“一个”进程，而是指容器没有管理多个进程的能力。这是因为容器里 PID=1 的进程就是应用本身，其他的进程都是这个 PID=1 进程的子进程。可是，用户编写的应用，并不能够像正常操作系统里的 init 进程或者 systemd 那样拥有进程管理的功能。比如，你的应用是一个 Java Web 程序（PID=1），然后你执行 docker exec 在后台启动了一个 Nginx 进程（PID=3）。可是，当这个 Nginx 进程异常退出的时候，你该怎么知道呢？这个进程退出后的垃圾收集工作，又应该由谁去做呢？

在 Pod 中，所有 **Init Container** 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。

`empty dir`类型

它其实就等同于我们之前讲过的 Docker 的**隐式 Volume 参数**，即：不显式声明宿主机目录的 Volume。所以，Kubernetes 也会在宿主机上创建一个临时目录，这个目录将来就会被绑定挂载到容器所声明的 Volume 目录上。

备注：不难看到，Kubernetes 的 emptyDir 类型，只是把 Kubernetes 创建的临时目录作为 Volume 的宿主机目录，交给了 Docker。这么做的原因，是 Kubernetes 不想依赖 Docker 自己创建的那个 _data 目录。

像这样容器间的紧密协作，我们可以称为“`超亲密关系`”。这些具有“超亲密关系”容器的典型特征包括但不限于：互相之间会发生直接的文件交换、使用 localhost 或者 Socket 文件进行本地通信、会发生非常频繁的远程调用、需要共享某些 Linux Namespace（比如，一个容器要加入另一个容器的 Network Namespace）等等。

这也就意味着，并不是所有有“关系”的容器都属于同一个 Pod。比如，PHP 应用容器和 MySQL 虽然会发生访问关系，但并没有必要、也不应该部署在同一台机器上，它们更适合做成两个 Pod。

首先，关于 Pod 最重要的一个事实是：它只是一个`逻辑概念`。也就是说，Kubernetes 真正处理的，还是宿主机操作系统上 Linux 容器的 Namespace 和 Cgroups，而并不存在一个所谓的 Pod 的边界或者隔离环境。

那么，Pod 又是怎么被“创建”出来的呢？答案是：Pod，其实是一组共享了某些资源的容器。具体的说：Pod 里的所有容器，共享的是`同一个 Network Namespace`，并且可以声明共享同一个 `Volume`。

所以，在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 **Infra** 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。

在 Kubernetes 项目里，Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：**k8s.gcr.io/pause**。这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有 100~200 KB 左右。

而在 Infra 容器“Hold 住”Network Namespace 后，用户容器就可以加入到 Infra 容器的 Network Namespace 当中了。所以，如果你查看这些容器在宿主机上的 Namespace 文件（这个 Namespace 文件的路径，我已经在前面的内容中介绍过），它们指向的值一定是完全一样的。

这也就意味着，对于 Pod 里的容器 A 和容器 B 来说：
- 它们可以直接使用 localhost 进行通信；
- 它们看到的网络设备跟 Infra 容器看到的完全一样；
- 一个 Pod 只有一个 IP 地址，也就是这个 Pod 的 Network Namespace 对应的 IP 地址；
- 当然，其他的所有网络资源，都是一个 Pod 一份，并且被该 Pod 中的所有容器共享；
- **Pod 的生命周期只跟 Infra 容器一致，而与容器 A 和 B 无关**。

而对于同一个 Pod 里面的所有用户容器来说，它们的进出流量，也可以认为都是通过 Infra 容器完成的。这一点很重要，因为将来如果你要为 Kubernetes 开发一个`网络插件`时，应该重点考虑的是如何配置这个 Pod 的 Network Namespace，而不是每一个用户容器如何使用你的网络配置，这是没有意义的。

这就意味着，如果你的网络插件需要在容器里`安装`某些包或者配置才能完成的话，是不可取的：Infra 容器镜像的 `rootfs` 里几乎什么都没有，没有你随意发挥的空间。当然，这同时也意味着你的网络插件完全不必关心用户容器的启动与否，而只需要关注如何配置 Pod，也就是 Infra 容器的 Network Namespace 即可。

有了这个设计之后，`共享 Volume` 就简单多了：Kubernetes 项目只要把所有 Volume 的定义都设计在 Pod 层级即可。

这样，一个 Volume 对应的宿主机目录对于 Pod 来说就只有一个，Pod 里的容器只要声明挂载这个 Volume，就一定可以共享这个 Volume 对应的宿主机目录。

第一个最典型的`例子`是：WAR 包与 Web 服务器。

我们现在有一个 Java Web 应用的 WAR 包，它需要被放在 Tomcat 的 webapps 目录下运行起来。

假如，你现在只能用 Docker 来做这件事情，那该如何处理这个组合关系呢？

一种方法是，把 WAR 包直接放在 Tomcat 镜像的 webapps 目录下，做成一个新的镜像运行起来。可是，这时候，如果你要更新 WAR 包的内容，或者要升级 Tomcat 镜像，就要重新制作一个新的发布镜像，非常麻烦。

另一种方法是，你压根儿不管 WAR 包，永远只发布一个 Tomcat 容器。不过，这个容器的 webapps 目录，就必须声明一个 hostPath 类型的 Volume，从而把宿主机上的 WAR 包挂载进 Tomcat 容器当中运行起来。不过，这样你就必须要解决一个问题，即：如何让每一台宿主机，都预先准备好这个存储有 WAR 包的目录呢？这样来看，你只能独立维护一套分布式存储系统了。

实际上，有了 Pod 之后，这样的问题就很容易解决了。我们可以把 `WAR 包和 Tomcat 分别做成镜像`，然后把它们作为一个 Pod 里的两个容器“组合”在一起。这个 Pod 的配置文件如下所示：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: javaweb-2
spec:
  initContainers:
  - image: geektime/sample:v2
    name: war
    command: ["cp", "/sample.war", "/app"]
    volumeMounts:
    - mountPath: /app
      name: app-volume
  containers:
  - image: geektime/tomcat:7.0
    name: tomcat
    command: ["sh","-c","/root/apache-tomcat-7.0.42-v2/bin/start.sh"]
    volumeMounts:
    - mountPath: /root/apache-tomcat-7.0.42-v2/webapps
      name: app-volume
    ports:
    - containerPort: 8080
      hostPort: 8001 
  volumes:
  - name: app-volume
    emptyDir: {}
```

在 Pod 中，所有 `Init Container` 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都`启动并且退出`了，用户容器才会启动。

像这样，我们就用一种“组合”方式，解决了 WAR 包与 Tomcat 容器之间耦合关系的问题。实际上，这个所谓的“组合”操作，正是容器设计模式里最常用的一种模式，它的名字叫：**sidecar**。

顾名思义，sidecar 指的就是我们可以在一个 Pod 中，启动一个辅助容器，来完成一些独立于主进程（主容器）之外的工作。比如，在我们的这个应用 Pod 中，Tomcat 容器是我们要使用的主容器，而 WAR 包容器的存在，只是为了给它提供一个 WAR 包而已。所以，我们用 Init Container 的方式优先运行 WAR 包容器，扮演了一个 sidecar 的角色。

**HostAliases**：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容

需要指出的是，在 Kubernetes 项目中，如果要设置 hosts 文件里的内容，一定要通过这种方法。否则，如果直接修改了 hosts 文件的话，在 Pod 被删除重建之后，kubelet 会自动覆盖掉被修改的内容。

我在前面介绍容器基础时，曾经讲解过什么是 **tty** 和 **stdin**。而在 Pod 的 YAML 文件里声明开启它们俩，其实等同于设置了 docker run 里的 -it（-i 即 stdin，-t 即 tty）参数。如果你还是不太理解它们俩的作用的话，可以直接认为 tty 就是 Linux 给用户提供的一个常驻小程序，用于接收用户的标准输入，返回操作系统的标准输出。当然，为了能够在 tty 中输入信息，你还需要同时开启 stdin（标准输入流）。

先说 `postStart` 吧。它指的是，在容器启动后，立刻执行一个指定的操作。需要明确的是，postStart 定义的操作，虽然是在 Docker 容器 ENTRYPOINT 执行之后，但它并`不严格`保证顺序。也就是说，在 postStart 启动时，ENTRYPOINT 有可能还没有结束。当然，如果 postStart 执行超时或者错误，Kubernetes 会在该 Pod 的 Events 中报出该容器启动失败的错误信息，导致 Pod 也处于失败的状态。

而类似地，`preStop` 发生的时机，则是容器被杀死之前（比如，收到了 SIGKILL 信号）。而需要明确的是，preStop 操作的执行，是`同步`的。所以，它会阻塞当前的容器杀死流程，直到这个 Hook 定义操作完成之后，才允许容器被杀死，这跟 postStart 不一样。

凡是 Pod 中的容器要共享宿主机的 Namespace，也一定是 Pod 级别的定义
hostNetwork: true  
hostIPC: true  
hostPID: true

## volume projected 类型

在 Kubernetes 中，有几种特殊的 Volume，它们存在的意义不是为了存放容器里的数据，也不是用来进行容器和宿主机之间的数据交换。这些特殊 Volume 的作用，是为容器提供`预先定义好的数据`。所以，从容器的角度来看，这些 Volume 里的信息就是仿佛是被 Kubernetes“投射”（Project）进入容器当中的。

支持的类型有
Secret；
ConfigMap；
Downward API；
ServiceAccountToken。

这里需要注意的是，像这样创建的 Secret 对象里面的内容仅仅是经过了转码，而并没有被加密。在真正的生产环境中，你需要在 Kubernetes 中开启 **Secret 的加密插件**，增强数据的安全性。关于开启 Secret 加密插件的内容，我会在后续专门讲解 Secret 的时候，再做进一步说明。

更重要的是，像这样通过挂载方式进入到容器里的 Secret，**一旦其对应的 Etcd 里的数据被更新，这些 Volume 里的文件内容，同样也会被更新**。其实，这是 kubelet 组件在定时维护这些 Volume。需要注意的是，这个更新可能会有一定的`延时`。所以在编写应用程序时，在发起数据库连接的代码处写好重试和超时的逻辑，绝对是个好习惯。

不过，需要注意的是，**Downward API 能够获取到的信息，一定是 Pod 里的容器进程启动之前就能够确定下来的信息**。而如果你想要获取 Pod 容器运行后才会出现的信息，比如，容器进程的 PID，那就肯定不能使用 Downward API 了，而应该考虑在 Pod 里定义一个 `sidecar` 容器。

其实，Secret、ConfigMap，以及 Downward API 这三种 Projected Volume 定义的信息，大多还可以通过`环境变量`的方式出现在容器里。但是，**通过环境变量获取这些信息的方式，不具备自动更新的能力**。所以，一般情况下，我都建议你使用 Volume 文件的方式获取这些信息。

`Service Account` 对象的作用，就是 Kubernetes 系统内置的一种“`服务账户`”，它是 Kubernetes 进行权限分配的对象。比如，Service Account A，可以只被允许对 Kubernetes API 进行 GET 操作，而 Service Account B，则可以有 Kubernetes API 的所有操作权限。

像这样的 Service Account 的授权信息和文件，实际上保存在它所绑定的一个特殊的 `Secret` 对象里的。这个特殊的 Secret 对象，就叫作 `ServiceAccountToken`。任何运行在 Kubernetes 集群上的应用，都必须使用这个 ServiceAccountToken 里保存的授权信息，也就是 Token，才可以合法地访问 API Server。

所以说，Kubernetes 项目的 Projected Volume 其实只有三种，因为第四种 **ServiceAccountToken，只是一种特殊的 Secret 而已**。

>这ServiceAccountToken无法被查看到，我猜测是和serviceacount绑定的，相当于是sa`内置`的属性

为了方便使用，Kubernetes **已经为你提供了一个默认“服务账户”**（default Service Account）。并且，任何一个运行在 Kubernetes 里的 Pod，都可以直接使用这个默认的 Service Account，而无需显示地声明挂载它。

所以说，Kubernetes 其实在每个 Pod 创建的时候，自动在它的 spec.volumes 部分添加上了默认 ServiceAccountToken 的定义，然后自动给每个容器加上了对应的 volumeMounts 字段。这个过程对于用户来说是完全透明的。这样，一旦 Pod 创建完成，容器里的应用就可以直接从这个默认 ServiceAccountToken 的挂载目录里访问到授权信息和文件。这个容器内的路径在 Kubernetes 里是固定的，即：`/var/run/secrets/kubernetes.io/serviceaccount` 

这种把 Kubernetes 客户端以容器的方式运行在集群里，然后使用 default Service Account 自动授权的方式，被称作**InClusterConfig**，也是我最推荐的进行 Kubernetes API 编程的授权方式。

当然，考虑到自动挂载默认 ServiceAccountToken 的潜在风险，Kubernetes 允许你设置默认不为 Pod 里的容器自动挂载这个 Volume。

## probe

需要注意的是：Kubernetes 中并没有 Docker 的 Stop 语义。所以虽然是 Restart（重启），但实际却是重新创建了容器。

这个功能就是 Kubernetes 里的 Pod 恢复机制，也叫 `restartPolicy`。它是 Pod 的 Spec 部分的一个标准字段（pod.spec.restartPolicy），默认值是 Always，即：任何时候这个容器发生了异常，它一定会被重新创建。

但一定要强调的是，**Pod 的恢复过程，永远都是发生在当前节点上，而不会跑到别的节点上去**。事实上，一旦一个 Pod 与一个节点（Node）绑定，除非这个绑定发生了变化（pod.spec.node 字段被修改），否则它永远都不会离开这个节点。这也就意味着，如果这个宿主机宕机了，这个 Pod 也不会主动迁移到其他节点上去。

只要 Pod 的 restartPolicy 指定的策略允许重启异常的容器（比如：Always），那么这个 Pod 就会保持 `Running` 状态，并进行容器重启。否则，Pod 就会进入 `Failed` 状态 。

**对于包含多个容器的 Pod，只有它里面所有的容器都进入异常状态后，Pod 才会进入 Failed 状态**。在此之前，Pod 都是 Running 状态。此时，Pod 的 READY 字段会显示正常容器的个数

而如果你要关心这个容器退出后的上下文环境，比如容器退出后的日志、文件和目录，就需要将 restartPolicy 设置为 `Never`。因为一旦容器被自动重新创建，这些内容就有可能丢失掉了（被`垃圾回收`了）。

`readinessProbe` 检查结果的成功与否，决定的这个 Pod 是不是能被通过 Service 的方式访问到，而并不影响 Pod 的生命周期

## podpreset 废弃

运维人员就可以定义一个 PodPreset 对象。在这个对象中，凡是他想在开发人员编写的 Pod 里追加的字段，都可以预先定义好

需要说明的是，PodPreset 里定义的内容，只会在 Pod API 对象被创建之前追加在这个对象本身上，而不会影响任何 Pod 的控制器的定义。比如，我们现在提交的是一个 nginx-deployment，那么这个 Deployment 对象本身是永远不会被 PodPreset 改变的，被修改的只是这个 Deployment 创建出来的所有 Pod。这一点请务必区分清楚。

这里有一个问题：如果你定义了同时作用于一个 Pod 对象的多个 PodPreset，会发生什么呢？实际上，Kubernetes 项目会帮你合并（Merge）这两个 PodPreset 要做的修改。而如果它们要做的修改有冲突的话，这些冲突字段就不会被修改。

## deployment

我在前面介绍 Kubernetes 架构的时候，曾经提到过一个叫作 `kube-controller-manager` 的组件。实际上，这个组件，就是一系列控制器的集合。

实际上，这些控制器之所以被统一放在 `pkg/controller` 目录下，就是因为它们都遵循 Kubernetes 项目中的一个通用编排模式，即：`控制循环`（control loop）。

```go

for {
  实际状态 := 获取集群中对象X的实际状态（Actual State）
  期望状态 := 获取集群中对象X的期望状态（Desired State）
  if 实际状态 == 期望状态{
    什么都不做
  } else {
    执行编排动作，将实际状态调整为期望状态
  }
}
```

在具体实现中，**实际状态往往来自于 Kubernetes 集群本身**。

比如，kubelet 通过心跳汇报的容器状态和节点状态，或者监控系统中保存的应用监控数据，或者控制器主动收集的它自己感兴趣的信息，这些都是常见的实际状态的来源。

而**期望状态，一般来自于用户提交的 YAML 文件**。

接下来，以 `Deployment` 为例，我和你简单描述一下它对控制器模型的实现：

Deployment 控制器从 Etcd 中获取到所有携带了“app: nginx”标签的 Pod，然后统计它们的数量，这就是`实际状态`；
Deployment 对象的 Replicas 字段的值就是`期望状态`；
Deployment 控制器将两个状态做比较，然后根据比较结果，确定是创建 Pod，还是删除已有的 Pod（具体如何操作 Pod 对象，我会在下一篇文章详细介绍）。

可以看到，一个 Kubernetes 对象的主要编排逻辑，实际上是在第三步的“对比”阶段完成的。
这个操作，通常被叫作`调谐`（Reconcile）。这个调谐的过程，则被称作“Reconcile Loop”（调谐循环）或者“Sync Loop”（同步循环）。

而**调谐的最终结果，往往都是对被控制对象的某种写操作**。比如，增加 Pod，删除已有的 Pod，或者更新 Pod 的某个字段。这也是 Kubernetes 项目“面向 API 对象编程”的一个直观体现。

可以看到，Deployment 这个 `template` 字段里的内容，跟一个标准的 Pod 对象的 API 定义，丝毫不差。而所有被这个 Deployment 管理的 Pod 实例，其实都是根据这个 template 字段的内容创建出来的。像 Deployment 定义的 template 字段，在 Kubernetes 项目中有一个专有的名字，叫作 `PodTemplate`（Pod 模板）。

这个概念非常重要，因为后面我要讲解到的`大多数控制器`，都会使用 PodTemplate 来统一定义它所要管理的 Pod。更有意思的是，我们还会看到其他类型的对象模板，比如 `Volume 的模板`。

类似 Deployment 这样的一个控制器，实际上都是由上半部分的控制器定义（包括期望状态），加上下半部分的被控制对象的模板组成的。这就是为什么，在所有 API 对象的 Metadata 里，都有一个字段叫作 `ownerReference`，用于保存当前这个 API 对象的拥有者（Owner）的信息。

这也正是 Deployment `只允许`容器的 `restartPolicy=Always` 的主要原因：只有在容器能保证自己始终是 Running 状态的前提下，ReplicaSet 调整 Pod 的个数才有意义。

ReplicaSet 的 DESIRED、CURRENT 和 READY 字段的含义，和 Deployment 中是一致的。所以，相比之下，Deployment 只是在 ReplicaSet 的基础上，添加了 `UP-TO-DATE` 这个跟版本有关的状态字段。

如果需要滚动更新保存历史记录，就要在创建deployment的时候加上`--record`

你可能已经想到了一个问题：我们对 Deployment 进行的每一次更新操作，都会生成一个新的 ReplicaSet 对象，是不是有些多余，甚至浪费资源呢？

而在这个 `kubectl rollout resume` 指令执行之前，在 `kubectl rollout pause` 指令之后的这段时间里，我们对 Deployment 进行的所有修改，最后只会触发一次“滚动更新”。

不过，即使你像上面这样小心翼翼地控制了 `ReplicaSet` 的生成数量，随着应用版本的不断增加，Kubernetes 中还是会为同一个 Deployment 保存很多很多不同的 ReplicaSet。

那么，我们又该**如何控制这些“历史”ReplicaSet 的数量**呢？很简单，Deployment 对象有一个字段，叫作 `spec.revisionHistoryLimit`，就是 Kubernetes 为 Deployment 保留的“历史版本”个数。所以，如果把它设置为 0，你就再也不能做回滚操作了。

maxSurge 指定的是除了 DESIRED 数量之外，在**一次“滚动”中**，Deployment 控制器还可以创建多少个新 Pod；而 maxUnavailable 指的是，在一次“滚动”中，Deployment 控制器可以删除多少个旧 Pod。

## statefulset

StatefulSet 的设计其实非常容易理解。它把真实世界里的应用状态，抽象为了两种情况：

`拓扑状态`。这种情况意味着，应用的多个实例之间不是完全对等的关系。这些应用实例，必须按照某些顺序启动，比如应用的主节点 A 要先于从节点 B 启动。而如果你把 A 和 B 两个 Pod 删除掉，它们再次被创建出来时也必须严格按照这个顺序才行。并且，新创建出来的 Pod，必须和原来 Pod 的网络标识一样，这样原先的访问者才能使用同样的方法，访问到这个新 Pod。

`存储状态`。这种情况意味着，应用的多个实例分别绑定了不同的存储数据。对于这些应用实例来说，Pod A 第一次读取到的数据，和隔了十分钟之后再次读取到的数据，应该是同一份，哪怕在此期间 Pod A 被重新创建过。这种情况最典型的例子，就是一个数据库应用的多个存储实例。

尽管 web-0.nginx 这条记录本身不会变，但它解析到的 Pod 的 IP 地址，并不是固定的。这就意味着，**对于“有状态应用”实例的访问，你必须使用 DNS 记录或者 hostname 的方式，而绝不应该直接访问这些 Pod 的 IP 地址**。

那么，这个 Service 又是如何被访问的呢？

第一种方式，是以 **Service 的 VIP（Virtual IP，即：虚拟 IP）方式**。比如：当我访问 10.0.23.1 这个 Service 的 IP 地址时，10.0.23.1 其实就是一个 VIP，它会把请求转发到该 Service 所代理的某一个 Pod 上。这里的具体原理，我会在后续的 Service 章节中进行详细介绍。

第二种方式，**就是以 Service 的 DNS 方式**。比如：这时候，只要我访问“my-svc.my-namespace.svc.cluster.local”这条 DNS 记录，就可以访问到名叫 my-svc 的 Service 所代理的某一个 Pod。

而在第二种 Service DNS 的方式下，具体还可以分为两种处理方法：

第一种处理方法，是 `Normal Service`。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，正是 my-svc 这个 Service 的 VIP，后面的流程就跟 VIP 方式一致了。

而第二种处理方法，正是 `Headless Service`。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是 my-svc 代理的某一个 Pod 的 IP 地址。

可以看到，这里的区别在于，**Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址**。

**clusterIP与headless服务的区别**

clusterIP 有VIP 访问VIP会随机访问pod 访问域名会解析到VIP随机访问pod 没有pod的域名
headless 无VIP 访问服务域名会解析出选中pod的域名解析 可以访问pod的域名

可以看到，所谓的 Headless Service，其实仍是一个标准 Service 的 YAML 文件。只不过，它的 `clusterIP` 字段的值是：None

StatefulSet 管理的“有状态应用”的多个实例，也都是通过`同一份 Pod 模板`创建出来的，使用的是同一个 Docker 镜像。这也就意味着：如果你的应用要求不同节点的镜像不一样，那就不能再使用 StatefulSet 了。对于这种情况，应该考虑我后面会讲解到的 `Operator`。

**定义statefulset前必须先定义`headless`的service并且在`statefulset.spec.serviceName`制定，statefulset既有pod定义也要定义PVC模板**

当你按照这样的方式创建了一个 Headless Service 之后，它所代理的所有 Pod 的 IP 地址，都会被绑定一个这样格式的 DNS 记录，如下所示：

`<pod-name>.<svc-name>.<namespace>.svc.cluster.local`这个 DNS 记录，正是 Kubernetes 项目为 Pod 分配的唯一的“可解析身份”（Resolvable Identity）。有了这个“可解析身份”，只要你知道了一个 Pod 的名字，以及它对应的 Service 的名字，你就可以非常确定地通过这条 DNS 记录访问到 Pod 的 IP 地址。

```yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.1
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
```
创建出来的PVC都以`<PVC 名字 >-<StatefulSet 名字 >-< 编号 >`的方式命名

首先，StatefulSet 的控制器直接管理的是 `Pod`。这是因为，StatefulSet 里的不同 Pod 实例，不再像 ReplicaSet 中那样都是完全一样的，而是有了细微区别的。比如，每个 Pod 的 hostname、名字等都是不同的、携带了编号的。而 StatefulSet 区分这些实例的方式，就是通过在 Pod 的名字里加上事先约定好的编号。

其次，Kubernetes 通过 `Headless Service`，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录。只要 StatefulSet 能够保证这些 Pod 名字里的编号不变，那么 Service 里类似于 web-0.nginx.default.svc.cluster.local 这样的 DNS 记录也就不会变，而这条记录解析出来的 Pod 的 IP 地址，则会随着后端 Pod 的删除和再创建而自动更新。这当然是 Service 机制本身的能力，不需要 StatefulSet 操心。

最后，StatefulSet 还为每一个 Pod 分配并创建一个同样编号的 `PVC`。这样，Kubernetes 就可以通过 Persistent Volume 机制为这个 PVC 绑定上对应的 PV，从而保证了每一个 Pod 都拥有一个独立的 Volume。在这种情况下，即使 Pod 被删除，它所对应的 PVC 和 PV 依然会保留下来。所以当这个 Pod 被重新创建出来之后，Kubernetes 会为它找到同样编号的 PVC，挂载这个 PVC 对应的 Volume，从而获取到以前保存在 Volume 里的数据。

### 例子 mysql集群容器化

首先，用自然语言来描述一下我们想要部署的“有状态应用”。是一个“主从复制”（Maser-Slave Replication）的 MySQL 集群；有 1 个主节点（Master）；有多个从节点（Slave）；从节点需要能水平扩展；所有的写操作，只能在主节点上执行；读操作可以在所有节点上执行。

通过上面的叙述，我们不难看到，将部署 MySQL 集群的流程迁移到 Kubernetes 项目上，需要能够“容器化”地解决下面的“`三座大山`”：

1. Master 节点和 Slave 节点需要有`不同的配置文件`（即：不同的 my.cnf）；
2. Master 节点和 Slave 节点需要能够`传输备份信息文件`；
3. 在 Slave 节点第一次启动之前，需要执行一些`初始化 SQL` 操作；

而由于 MySQL 本身同时拥有拓扑状态（主从节点的区别）和存储状态（MySQL 保存在本地的数据），我们自然要通过 `StatefulSet` 来解决这“三座大山”的问题。

“第一座大山：Master 节点和 Slave 节点需要有不同的配置文件”，很容易处理：我们只需要给主从节点分别准备两份不同的 MySQL 配置文件，然后根据 Pod 的序号（Index）挂载进去即可。
正如我在前面文章中介绍过的，这样的配置文件信息，应该保存在 `ConfigMap` 里供 Pod 使用。

```yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: mysql
  labels:
    app: mysql
data:
  master.cnf: |
    # 主节点MySQL的配置文件
    [mysqld]
    log-bin
  slave.cnf: |
    # 从节点MySQL的配置文件
    [mysqld]
    super-read-only
```

在这里，我们定义了 master.cnf 和 slave.cnf 两个 MySQL 的配置文件。
master.cnf 开启了 log-bin，即：使用二进制日志文件的方式进行主从复制，这是一个标准的设置。
slave.cnf 的开启了 super-read-only，代表的是从节点会拒绝除了主节点的数据同步操作之外的所有写操作，即：它对用户是只读的。

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  clusterIP: None
  selector:
    app: mysql
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-read
  labels:
    app: mysql
spec:
  ports:
  - name: mysql
    port: 3306
  selector:
    app: mysql
```
可以看到，这两个 Service 都代理了所有携带 app=mysql 标签的 Pod，也就是所有的 MySQL Pod。端口映射都是用 Service 的 3306 端口对应 Pod 的 3306 端口。

不同的是，第一个名叫“mysql”的 Service 是一个 Headless Service（即：clusterIP= None）。所以它的作用，是通过为 Pod 分配 DNS 记录来固定它的`拓扑状态`，比如“mysql-0.mysql”和“mysql-1.mysql”这样的 DNS 名字。其中，编号为 0 的节点就是我们的主节点。

而第二个名叫“mysql-read”的 Service，则是一个常规的 Service。并且我们规定，所有用户的`读请求`，都必须访问第二个 Service 被自动分配的 DNS 记录，即：“mysql-read”（当然，也可以访问这个 Service 的 VIP）。这样，读请求就可以被转发到任意一个 MySQL 的主节点或者从节点上。

而所有用户的`写请求`，则必须直接以 DNS 记录的方式访问到 MySQL 的主节点，也就是：“mysql-0.mysql“这条 DNS 记录。


![](../../reference/pic/k8s-mysql.webp)

```yml
# 注意mysql5.7的镜像可能存在hostname没有的问题
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec: 
  selector:
    matchLabels:
      app: mysql 
  serviceName: mysql 
  replicas: 3
  template:
    metadata: 
      labels: 
        app: mysql 
    spec:
      # template.spec
      # 配置文件
      initContainers:
      - name: init-mysql
        image: mysql:5.7
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 从Pod的序号，生成server-id
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          echo [mysqld] > /mnt/conf.d/server-id.cnf
          # 由于server-id=0有特殊含义，我们给ID加一个100来避开它
          echo server-id=$((100 + $ordinal)) >> /mnt/conf.d/server-id.cnf
          # 如果Pod序号是0，说明它是Master节点，从ConfigMap里把Master的配置文件拷贝到/mnt/conf.d/目录；
          # 否则，拷贝Slave的配置文件
          if [[ $ordinal -eq 0 ]]; then
            cp /mnt/config-map/master.cnf /mnt/conf.d/
          else
            cp /mnt/config-map/slave.cnf /mnt/conf.d/
          fi
        volumeMounts:
        - name: conf
          mountPath: /mnt/conf.d
        - name: config-map
          mountPath: /mnt/config-map
      # template.spec.initContainers
      # 拷贝数据库
      - name: clone-mysql
        image: yizhiyong/xtrabackup
        command:
        - bash
        - "-c"
        - |
          set -ex
          # 拷贝操作只需要在第一次启动时进行，所以如果数据已经存在，跳过
          [[ -d /var/lib/mysql/mysql ]] && exit 0
          # Master节点(序号为0)不需要做这个操作
          [[ `hostname` =~ -([0-9]+)$ ]] || exit 1
          ordinal=${BASH_REMATCH[1]}
          [[ $ordinal -eq 0 ]] && exit 0
          # 使用ncat指令，远程地从前一个节点拷贝数据到本地
          ncat --recv-only mysql-$(($ordinal-1)).mysql 3307 | xbstream -x -C /var/lib/mysql
          # 执行--prepare，这样拷贝来的数据就可以用作恢复了
          xtrabackup --prepare --target-dir=/var/lib/mysql
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d

      # template.spec
      containers:
      - name: mysql
        image: mysql:5.7
        env:
        - name: MYSQL_ALLOW_EMPTY_PASSWORD
          value: "1"
        ports:
        - name: mysql
          containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d
        resources:
          requests:
            cpu: 500m
            memory: 1Gi
        livenessProbe:
          exec:
            command: ["mysqladmin", "ping"]
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          exec:
            # 通过TCP连接的方式进行健康检查
            command: ["mysql", "-h", "127.0.0.1", "-e", "SELECT 1"]
          initialDelaySeconds: 5
          periodSeconds: 2
          timeoutSeconds: 1                 
      # template.spec.containers
      # 初始化
      - name: xtrabackup
        image: yizhiyong/xtrabackup
        ports:
        - name: xtrabackup
          containerPort: 3307
        command:
        - bash
        - "-c"
        - |
          set -ex
          cd /var/lib/mysql
          
          # 从备份信息文件里读取MASTER_LOG_FILEM和MASTER_LOG_POS这两个字段的值，用来拼装集群初始化SQL
          if [[ -f xtrabackup_slave_info ]]; then
            # 如果xtrabackup_slave_info文件存在，说明这个备份数据来自于另一个Slave节点。这种情况下，XtraBackup工具在备份的时候，就已经在这个文件里自动生成了"CHANGE MASTER TO" SQL语句。所以，我们只需要把这个文件重命名为change_master_to.sql.in，后面直接使用即可
            mv xtrabackup_slave_info change_master_to.sql.in
            # 所以，也就用不着xtrabackup_binlog_info了
            rm -f xtrabackup_binlog_info
          elif [[ -f xtrabackup_binlog_info ]]; then
            # 如果只存在xtrabackup_binlog_inf文件，那说明备份来自于Master节点，我们就需要解析这个备份信息文件，读取所需的两个字段的值
            [[ `cat xtrabackup_binlog_info` =~ ^(.*?)[[:space:]]+(.*?)$ ]] || exit 1
            rm xtrabackup_binlog_info
            # 把两个字段的值拼装成SQL，写入change_master_to.sql.in文件
            echo "CHANGE MASTER TO MASTER_LOG_FILE='${BASH_REMATCH[1]}',\
                  MASTER_LOG_POS=${BASH_REMATCH[2]}" > change_master_to.sql.in
          fi
          
          # 如果change_master_to.sql.in，就意味着需要做集群初始化工作
          if [[ -f change_master_to.sql.in ]]; then
            # 但一定要先等MySQL容器启动之后才能进行下一步连接MySQL的操作
            echo "Waiting for mysqld to be ready (accepting connections)"
            until mysql -h 127.0.0.1 -e "SELECT 1"; do sleep 1; done
            
            echo "Initializing replication from clone position"
            # 将文件change_master_to.sql.in改个名字，防止这个Container重启的时候，因为又找到了change_master_to.sql.in，从而重复执行一遍这个初始化流程
            mv change_master_to.sql.in change_master_to.sql.orig
            # 使用change_master_to.sql.orig的内容，也是就是前面拼装的SQL，组成一个完整的初始化和启动Slave的SQL语句
            mysql -h 127.0.0.1 <<EOF
          $(<change_master_to.sql.orig),
            MASTER_HOST='mysql-0.mysql',
            MASTER_USER='root',
            MASTER_PASSWORD='',
            MASTER_CONNECT_RETRY=10;
          START SLAVE;
          EOF
          fi
          
          # 使用ncat监听3307端口。它的作用是，在收到传输请求的时候，直接执行"xtrabackup --backup"命令，备份MySQL的数据并发送给请求者
          exec ncat --listen --keep-open --send-only --max-conns=1 3307 -c \
            "xtrabackup --backup --slave-info --stream=xbstream --host=127.0.0.1 --user=root"
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
          subPath: mysql
        - name: conf
          mountPath: /etc/mysql/conf.d  
      volumes: 
      - name: conf 
        emptyDir: {}
      - name: config-map 
        configMap:
          name: mysql 
  volumeClaimTemplates:
  - metadata: 
      name: data
    spec: 
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi


```
在这个名叫 xtrabackup 的 `sidecar` 容器的启动命令里，其实实现了两部分工作。第一部分工作，当然是 MySQL 节点的初始化工作。这个初始化需要使用的 SQL，是 sidecar 容器拼装出来、保存在一个名为 change_master_to.sql.in 的文件里的，具体过程如下所示：sidecar 容器首先会判断当前 Pod 的 /var/lib/mysql 目录下，是否有 xtrabackup_slave_info 这个备份信息文件。如果有，则说明这个目录下的备份数据是由一个 Slave 节点生成的。这种情况下，XtraBackup 工具在备份的时候，就已经在这个文件里自动生成了"CHANGE MASTER TO" SQL 语句。所以，我们只需要把这个文件重命名为 change_master_to.sql.in，后面直接使用即可。如果没有 xtrabackup_slave_info 文件、但是存在 xtrabackup_binlog_info 文件，那就说明备份数据来自于 Master 节点。这种情况下，sidecar 容器就需要解析这个备份信息文件，读取 MASTER_LOG_FILE 和 MASTER_LOG_POS 这两个字段的值，用它们拼装出初始化 SQL 语句，然后把这句 SQL 写入到 change_master_to.sql.in 文件中。接下来，sidecar 容器就可以执行初始化了。从上面的叙述中可以看到，只要这个 change_master_to.sql.in 文件存在，那就说明接下来需要进行集群初始化操作。所以，这时候，sidecar 容器只需要读取并执行 change_master_to.sql.in 里面的“CHANGE MASTER TO”指令，再执行一句 START SLAVE 命令，一个 Slave 节点就被成功启动了。
需要注意的是：Pod 里的容器并没有先后顺序，所以在执行初始化 SQL 之前，必须先执行一句 SQL（select 1）来检查一下 MySQL 服务是否已经可用。当然，上述这些初始化操作完成后，我们还要删除掉前面用到的这些备份信息文件。否则，下次这个容器重启时，就会发现这些文件存在，所以又会重新执行一次数据恢复和集群初始化的操作，这是不对的。同理，change_master_to.sql.in 在使用后也要被重命名，以免容器重启时因为发现这个文件存在又执行一遍初始化。

在完成 MySQL 节点的初始化后，这个 sidecar 容器的第二个工作，则是启动一个数据传输服务。具体做法是：sidecar 容器会使用 ncat 命令启动一个工作在 3307 端口上的网络发送服务。一旦收到数据传输请求时，sidecar 容器就会调用 xtrabackup --backup 指令备份当前 MySQL 的数据，然后把这些备份数据返回给请求者。这就是为什么我们在 InitContainer 里定义数据拷贝的时候，访问的是“上一个 MySQL 节点”的 3307 端口。值得一提的是，由于 sidecar 容器和 MySQL 容器同处于一个 Pod 里，所以它是直接通过 Localhost 来访问和备份 MySQL 容器里的数据的，非常方便。同样地，我在这里举例用的只是一种备份方法而已，你完全可以选择其他自己喜欢的方案。比如，你可以使用 innobackupex 命令做数据备份和准备，它的使用方法几乎与本文的备份方法一样。

### 滚动更新

比如，如何对 StatefulSet 进行“`滚动更新`”（rolling update）？很简单。你只要修改 StatefulSet 的 Pod 模板，就会自动触发“滚动更新”:
```shell
kubectl patch statefulset mysql --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/containers/0/image", "value":"mysql:5.7.23"}]'
statefulset.apps/mysql patched
```
这样，StatefulSet Controller 就会按照与 Pod `编号相反`的顺序，从最后一个 Pod 开始，逐一更新这个 StatefulSet 管理的每个 Pod。而如果更新发生了错误，这次“滚动更新”就会停止。此外，StatefulSet 的“滚动更新”还允许我们进行更精细的控制，比如`金丝雀发布`（Canary Deploy）或者`灰度发布`，这意味着应用的多个实例中被指定的一部分不会被更新到最新的版本。这个字段，正是 StatefulSet 的 `spec.updateStrategy.rollingUpdate` 的 partition 字段。


## daemonset

这个 DaemonSet，管理的是一个 `fluentd-elasticsearch` 镜像的 Pod。这个镜像的功能非常实用：通过 fluentd 将 Docker 容器里的日志转发到 ElasticSearch 中。

```yml

apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: registry.k8s.io/fluentd-elasticsearch:1.20
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
```

需要注意的是，Docker 容器里应用的日志，默认会保存在宿主机的 `/var/lib/docker/containers/{{. 容器 ID}}/{{. 容器 ID}}-json.log` 文件里，所以这个目录正是 fluentd 的搜集目标。

那么，DaemonSet 又是如何保证每个 Node 上有且只有一个被管理的 Pod 呢？显然，这是一个典型的“`控制器模型`”能够处理的问题。DaemonSet Controller，首先从 Etcd 里获取所有的 Node 列表，然后遍历所有的 Node。这时，它就可以很容易地去检查，当前这个 Node 上是不是有一个携带了 name=fluentd-elasticsearch 标签的 Pod 在运行。

假如当前 DaemonSet 管理的，是一个网络插件的 Agent Pod，那么你就必须在这个 DaemonSet 的 YAML 文件里，给它的 Pod 模板**加上一个能够“容忍”node.kubernetes.io/network-unavailable“污点”的 Toleration**。

而我在前面的文章中讲解 Deployment 对象的时候，曾经提到过，Deployment 管理这些`版本`，靠的是“一个版本对应一个 ReplicaSet 对象”。可是，DaemonSet 控制器操作的直接就是 Pod，不可能有 ReplicaSet 这样的对象参与其中。那么，它的这些版本又是如何维护的呢？

至此，通过上面这些内容，你应该能够明白，DaemonSet 其实是一个非常简单的控制器。在它的控制循环中，只需要遍历所有节点，然后根据节点上是否有被管理 Pod 的情况，来决定是否要创建或者删除一个 Pod。

只不过，**在创建每个 Pod 的时候，DaemonSet 会自动给这个 Pod 加上一个 nodeAffinity**，从而保证这个 Pod 只会在指定节点上启动。同时，它还会自动给这个 Pod 加上一个 Toleration，从而忽略节点的 unschedulable“污点”。

所谓，一切皆对象！在 Kubernetes 项目中，任何你觉得需要记录下来的状态，都可以被用 API 对象的方式实现。当然，“版本”也不例外。Kubernetes v1.7 之后添加了一个 API 对象，名叫 **ControllerRevision**，专门用来记录某种 Controller 对象的版本。

这个 ControllerRevision 对象，**实际上是在 `Data` 字段保存了该版本对应的完整的 DaemonSet 的 API 对象。并且，在 `Annotation` 字段保存了创建这个对象所使用的 kubectl 命令**。

这个 `kubectl rollout undo` 操作，实际上相当于读取到了 Revision=1 的 ControllerRevision 对象保存的 Data 字段。而这个 Data 字段里保存的信息，就是 Revision=1 时这个 DaemonSet 的完整 API 对象。

与此同时，DaemonSet 使用 ControllerRevision，来保存和管理自己对应的“版本”。这种“面向 API 对象”的设计思路，大大简化了控制器本身的逻辑，也正是 Kubernetes 项目“声明式 API”的优势所在。


**StatefulSet 与 DaemonSet 都使用 ControllerRevision 进行pod的版本管理**，但是ControllerRevision并不管理pod，所以与replicaset不同，ControllerRevision的owner是daemonset这些

## job cronjob

```yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  template:
    spec:
      containers:
      - name: pi
        image: resouer/ubuntu-bc 
        command: ["sh", "-c", "echo 'scale=10000; 4*a(1)' | bc -l "]
      restartPolicy: Never
  backoffLimit: 4
```

Job 对象在创建后，**它的 Pod 模板，被自动加上了一个 `controller-uid=< 一个随机字符串 >` 这样的 Label**。而这个 Job 对象本身，则被自动加上了这个 Label 对应的 `Selector`，从而 保证了 Job 与它所管理的 Pod 之间的匹配关系。

而 Job Controller 之所以要使用这种携带了 UID 的 Label，就是**为了避免不同 Job 对象所管理的 Pod 发生重合**。需要注意的是，这种自动生成的 Label 对用户来说并不友好，所以不太适合推广到 Deployment 等长作业编排对象上。

>事实上，`restartPolicy` 在 Job 对象里只允许被设置为 Never 和 OnFailure；而在 Deployment 对象里，restartPolicy 则只允许被设置为 Always。

`restartPolicy=Never`，那么离线作业失败后 Job Controller 就会不断地尝试创建一个新 Pod

我们就在 Job 对象的 `spec.backoffLimit` 字段里定义了重试次数为 4（即，backoffLimit=4），而这个字段的`默认值`是 6。需要注意的是，Job Controller 重新创建 Pod 的间隔是呈`指数增加`的，即下一次重新创建 Pod 的动作会分别发生在 10 s、20 s、40 s …后。

而如果你定义的 `restartPolicy=OnFailure`，那么离线作业失败后，Job Controller 就不会去尝试创建新的 Pod。但是，它会不断地尝试重启 Pod 里的容器。

在 Job 对象中，负责并行控制的参数有两个：`spec.parallelism`，它定义的是一个 Job 在任意时间最多可以启动多少个 Pod 同时运行；`spec.completions`，它定义的是 Job 至少要完成的 Pod 数目，即 Job 的最小完成

Job Controller 实际上控制了，作业执行的并行度，以及总共需要完成的任务数这两个重要参数。而在实际使用时，你需要根据作业的特性，来决定并行度（parallelism）和任务数（completions）的合理取值。

我再和你分享**三种常用的、使用 Job 对象的方法**。

第一种用法，也是最简单粗暴的用法：`外部管理器 +Job 模板`。这种模式的特定用法是：把 Job 的 YAML 文件定义为一个“模板”，然后用一个外部工具控制这些“模板”来生成 Job。

```yaml

apiVersion: batch/v1
kind: Job
metadata:
  name: process-item-$ITEM
  labels:
    jobgroup: jobexample
spec:
  template:
    metadata:
      name: jobexample
      labels:
        jobgroup: jobexample
    spec:
      containers:
      - name: c
        image: busybox
        command: ["sh", "-c", "echo Processing item $ITEM && sleep 5"]
      restartPolicy: Never
```
```shell

mkdir ./jobs
for i in apple banana cherry
do
  cat job-tmpl.yaml | sed "s/\$ITEM/$i/" > ./jobs/job-$i.yaml
done
```

第二种用法：拥有`固定任务数目`的并行 Job。

这种模式下，我只关心最后是否有指定数目（spec.completions）个任务成功退出。至于执行时的并行度是多少，我并不关心。

第三种用法，也是很常用的一个用法：指定`并行度`（parallelism），但不设置`固定`的 completions 的值。

此时，你就必须自己想办法，来决定什么时候启动新 Pod，什么时候 Job 才算执行完成。在这种情况下，任务的总数是未知的，所以你不仅需要一个工作队列来负责任务分发，还需要能够判断工作队列已经为空（即：所有的工作已经结束了）。

```yaml

apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes cluster
          restartPolicy: OnFailure
```
没错，CronJob 与 Job 的关系，正如同 Deployment 与 ReplicaSet 的关系一样。CronJob 是一个专门用来管理 Job 对象的`控制器`。

需要注意的是，由于定时任务的特殊性，很可能某个 Job 还没有执行完，另外一个新 Job 就产生了。这时候，你可以通过 `spec.concurrencyPolicy` 字段来定义具体的处理策略。比如：concurrencyPolicy=Allow，这也是默认情况，这意味着这些 Job 可以同时存在；concurrencyPolicy=Forbid，这意味着不会创建新的 Pod，该创建周期被跳过；concurrencyPolicy=Replace，这意味着新产生的 Job 会替换旧的、没有执行完的 Job。

## 声明式API Initializer

实际上，你可以简单地理解为，`kubectl replace` 的执行过程，是使用新的 YAML 文件中的 API 对象，替换原有的 API 对象；而 `kubectl apply`，则是执行了一个对原有 API 对象的 PATCH 操作。

>我能想到的就是replace是对原来对象的替换，而apply是对原有对象的字段更新

类似地，`kubectl set image` 和 `kubectl edit` 也是对已有 API 对象的修改。更进一步地，这意味着 `kube-apiserver` 在响应命令式请求（比如，kubectl replace）的时候，一次只能处理一个写请求，否则会有产生冲突的可能。而对于声明式请求（比如，kubectl apply），**一次能处理多个写操作，并且具备 Merge 能力**。

实际上，Istio 项目使用的，是 Kubernetes 中的一个非常重要的功能，叫作 **Dynamic Admission Control**。在 Kubernetes 项目中，当一个 Pod 或者任何一个 API 对象被提交给 APIServer 之后，总有一些“初始化”性质的工作需要在它们被 Kubernetes 项目正式处理之前进行。比如，自动为所有 Pod 加上某些标签（Labels）。

而这个“初始化”操作的实现，借助的是一个叫作 Admission 的功能。它其实是 Kubernetes 项目里一组被称为 `Admission Controller` 的代码，可以选择性地被编译进 APIServer 中，在 API 对象创建之后会被立刻调用到。但这就意味着，如果你现在想要添加一些自己的规则到 Admission Controller，就会比较困难。因为，这要求重新编译并重启 APIServer。

显然，这种使用方法对 Istio 来说，影响太大了。所以，Kubernetes 项目为我们额外提供了一种“热插拔”式的 Admission 机制，它就是 Dynamic Admission Control，也叫作：**Initializer**。

>现在的istio可能换成了webhook

Istio 要做的，就是编写一个用来为 Pod“`自动注入`”Envoy 容器的 Initializer。首先，Istio 会将这个 Envoy 容器本身的定义，以 `ConfigMap` 的方式保存在 Kubernetes 当中。

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-initializer
data:
  config: |
    containers:
      - name: envoy
        image: lyft/envoy:845747db88f102c0fd262ab234308e9e22f693a1
        command: ["/usr/local/bin/envoy"]
        args:
          - "--concurrency 4"
          - "--config-path /etc/envoy/envoy.json"
          - "--mode serve"
        ports:
          - containerPort: 80
            protocol: TCP
        resources:
          limits:
            cpu: "1000m"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "64Mi"
        volumeMounts:
          - name: envoy-conf
            mountPath: /etc/envoy
    volumes:
      - name: envoy-conf
        configMap:
          name: envoy
```

不难想到，Initializer 要做的工作，就是把这部分 Envoy 相关的字段，`自动添加`到用户提交的 Pod 的 API 对象里。可是，用户提交的 Pod 里本来就有 containers 字段和 volumes 字段，所以 Kubernetes 在处理这样的更新请求时，就必须使用类似于 `git merge` 这样的操作，才能将这两部分内容合并在一起。

接下来，**Istio 将一个编写好的 Initializer，作为一个 Pod 部署在 Kubernetes 中**。
```yaml

apiVersion: v1
kind: Pod
metadata:
  labels:
    app: envoy-initializer
  name: envoy-initializer
spec:
  containers:
    - name: envoy-initializer
      image: envoy-initializer:0.0.1
      imagePullPolicy: Always
```
envoy-initializer 使用的 envoy-initializer:0.0.1 镜像，就是一个事先编写好的“`自定义控制器`”（Custom Controller）

一个 Kubernetes 的控制器，实际上就是一个“死循环”：它不断地获取“实际状态”，然后与“期望状态”作对比，并以此为依据决定下一步的操作。

**而 Initializer 的控制器，不断获取到的`实际状态`，就是用户新创建的 Pod。而它的`期望状态`，则是：这个 Pod 里被添加了 Envoy 容器的定义。**

### crd

>原文的crd定义不再合法，见新定义

```
将原始代码clone下来后, 执行 go mod init -> go mod tidy -> go build -o samplecrd-controller .

运行:
./samplecrd-controller -kubeconfig=$HOME/.kube/config -alsologtostderr=true
```

crd 的yaml格式需要修改为:
```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
  annotations:
    "api-approved.kubernetes.io": "https://github.com/kubernetes/kubernetes/pull/78458"
spec:
  group: samplecrd.k8s.io
  names:
    kind: Network
    plural: networks
    singular: network
    shortNames:
    - nw
  scope: Namespaced
  versions:
  - served: true
    storage: true
    name: v1
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              cidr:
                type: string
              gateway:
                type: string
```

![](../../reference/pic/apis.webp)

需要明确的是，对于 Kubernetes 里的核心 API 对象，比如：Pod、Node 等，是不需要 `Group` 的（即：它们的 Group 是“”）。所以，对于这些 API 对象来说，Kubernetes 会直接在 /api 这个层级进行下一步的匹配过程。

而对于 CronJob 等非核心 API 对象来说，Kubernetes 就必须在 /apis 这个层级里查找它对应的 Group，进而根据“batch”这个 Group 的名字，找到 /apis/batch。

首先，当我们发起了创建 CronJob 的 POST 请求之后，我们编写的 YAML 的信息就被提交给了 APIServer。而 APIServer 的第一个功能，就是**过滤**这个请求，并完成一些前置性的工作，比如授权、超时处理、审计等。

然后，请求会进入 **MUX** 和 **Routes** 流程。如果你编写过 Web Server 的话就会知道，MUX 和 Routes 是 APIServer 完成 URL 和 Handler 绑定的场所。而 APIServer 的 Handler 要做的事情，就是按照我刚刚介绍的匹配过程，找到对应的 CronJob 类型定义。

接着，APIServer 最重要的职责就来了：根据这个 CronJob 类型定义，使用用户提交的 YAML 文件里的字段，创建一个 CronJob `对象`。而在这个过程中，APIServer 会进行一个 Convert 工作，即：**把用户提交的 YAML 文件，转换成一个叫作 Super Version 的对象**，它正是该 API 资源类型所有版本的字段全集。这样用户提交的不同版本的 YAML 文件，就都可以用这个 Super Version 对象来进行处理了。

接下来，APIServer 会先后进行 **Admission() 和 Validation()** 操作。比如，我在上一篇文章中提到的 Admission Controller 和 Initializer，就都属于 Admission 的内容。

而 Validation，则负责验证这个对象里的各个字段是否合法。这个被验证过的 API 对象，都保存在了 APIServer 里一个叫作 **Registry** 的数据结构中。也就是说，只要一个 API 对象的定义能在 Registry 里查到，它就是一个有效的 Kubernetes API 对象。

最后，APIServer 会把验证过的 API 对象转换成用户最初提交的版本，进行**序列化**操作，并调用 Etcd 的 API 把它保存起来。

CRD 的全称是 `Custom Resource Definition`。顾名思义，它指的就是，允许用户在 Kubernetes 中添加一个跟 Pod、Node 类似的、新的 API 资源类型，即：自定义 API 资源。

```yaml

apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: "192.168.0.0/16"
  gateway: "192.168.0.1"
```

可以看到，我想要描述“网络”的 API 资源类型是 Network；API 组是samplecrd.k8s.io；API 版本是 v1。

其实，上面的这个 YAML 文件，就是一个具体的“自定义 API 资源”实例，也叫 `CR`（Custom Resource）。而为了能够让 Kubernetes 认识这个 CR，你就需要让 Kubernetes 明白这个 CR 的宏观定义是什么，也就是 `CRD`（Custom Resource Definition）。

所以，接下来，我就先编写一个 CRD 的 YAML 文件，它的名字叫作 network.yaml，内容如下所示：
```yaml

apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
spec:
  group: samplecrd.k8s.io
  version: v1
  names:
    kind: Network
    plural: networks
  scope: Namespaced
```

这就是一个 Network API 资源类型的 API 部分的`宏观定义`了。这就等同于告诉了计算机：“兔子是哺乳动物”。所以这时候，Kubernetes 就能够认识和处理所有声明了 API 类型是“samplecrd.k8s.io/v1/network”的 YAML 文件了。

接下来，我还需要让 Kubernetes“认识”这种 YAML 文件里描述的“网络”部分，比如“cidr”（网段），“gateway”（网关）这些字段的`含义`。这就相当于我要告诉计算机：“兔子有长耳朵和三瓣嘴”。这时候呢，我就需要稍微做些`代码工作`了。

代码仓库：https://github.com/resouer/k8s-controller-custom-resource/

types.go
```go

package v1
...
// +genclient
// +genclient:noStatus
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Network describes a Network resource
type Network struct {
 // TypeMeta is the metadata for the resource, like kind and apiversion
 metav1.TypeMeta `json:",inline"`
 // ObjectMeta contains the metadata for the particular object, including
 // things like...
 //  - name
 //  - namespace
 //  - self link
 //  - labels
 //  - ... etc ...
 metav1.ObjectMeta `json:"metadata,omitempty"`
 
 Spec networkspec `json:"spec"`
}
// networkspec is the spec for a Network resource
type networkspec struct {
 Cidr    string `json:"cidr"`
 Gateway string `json:"gateway"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// NetworkList is a list of Network resources
type NetworkList struct {
 metav1.TypeMeta `json:",inline"`
 metav1.ListMeta `json:"metadata"`
 
 Items []Network `json:"items"`
}
```

你会看到 `+<tag_name>[=value]`格式的注释，这就是 Kubernetes 进行代码生成要用的 Annotation 风格的注释。其中，`+k8s:deepcopy-gen=package` 意思是，请为整个 v1 包里的所有类型定义自动生成 DeepCopy 方法；而`+groupName=samplecrd.k8s.io`，则定义了这个包对应的 API 组的名字。可以看到，这些定义在 doc.go 文件的注释，起到的是全局的代码生成控制的作用，所以也被称为 `Global Tags`。

此外，除了定义 Network 类型，你还需要定义一个 NetworkList 类型，用来描述一组 Network 对象应该包括哪些字段。之所以需要这样一个类型，是因为在 Kubernetes 中，获取所有 X 对象的 List() 方法，返回值都是`List 类型`，而不是 X 类型的数组。

同样地，在 Network 和 NetworkList 类型上，也有代码生成注释。其中，`+genclient` 的意思是：请为下面这个 API 资源类型生成对应的 Client 代码（这个 Client，我马上会讲到）。而 `+genclient:noStatus` 的意思是：这个 API 资源类型定义里，没有 Status 字段。否则，生成的 Client 就会自动带上 UpdateStatus 方法。如果你的类型定义包括了 Status 字段的话，就不需要这句 +genclient:noStatus 注释了。

需要注意的是，+genclient 只需要写在 Network 类型上，而不用写在 NetworkList 上。因为 NetworkList 只是一个`返回值类型`，Network 才是“主类型”。

而由于我在 Global Tags 里已经定义了为所有类型生成 DeepCopy 方法，所以这里就不需要再显式地加上 +k8s:deepcopy-gen=true 了。当然，这也就意味着你可以用 +k8s:deepcopy-gen=false 来阻止为某些类型生成 DeepCopy。

你可能已经注意到，在这两个类型上面还有一句`+k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object`的注释。它的意思是，请在生成 DeepCopy 的时候，实现 Kubernetes 提供的 `runtime.Object` 接口。否则，在某些版本的 Kubernetes 里，你的这个类型定义会出现编译错误。这是一个固定的操作，记住即可。

在前面对 APIServer 工作原理的讲解中，我已经提到，“`registry`”的作用就是注册一个类型（Type）给 APIServer。其中，Network 资源类型在服务器端注册的工作，APIServer 会自动帮我们完成。但与之对应的，我们还需要让`客户端`也能“知道”Network 资源类型的定义。这就需要我们在项目里添加一个 register.go 文件。

有了这个方法，Kubernetes 就能够在后面生成客户端的时候，“知道”Network 以及 NetworkList 类型的定义了。像上面这种 register.go 文件里的内容其实是非常`固定`的，你以后可以直接使用我提供的这部分代码做模板，然后把其中的资源类型、GroupName 和 Version 替换成你自己的定义即可。

接下来，我就要使用 Kubernetes 提供的`代码生成工具`，为上面定义的 Network 资源类型自动生成 `clientset`、`informer` 和 `lister`。其中，clientset 就是操作 Network 对象所需要使用的`客户端`，而 informer 和 lister 这两个包的主要功能，我会在下一篇文章中重点讲解。

这个代码生成工具名叫`k8s.io/code-generator`，使用方法如下所示：
```shell

# 代码生成的工作目录，也就是我们的项目路径
ROOT_PACKAGE="github.com/resouer/k8s-controller-custom-resource"
# API Group
CUSTOM_RESOURCE_NAME="samplecrd"
# API Version
CUSTOM_RESOURCE_VERSION="v1"

# 安装k8s.io/code-generator
go get -u k8s.io/code-generator/...
cd $GOPATH/src/k8s.io/code-generator

# 执行代码自动生成，其中pkg/client是生成目标目录，pkg/apis是类型定义目录
./generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION"
```

其中，pkg/apis/samplecrd/v1 下面的 zz_generated.deepcopy.go 文件，就是自动生成的 DeepCopy 代码文件。而整个 client 目录，以及下面的三个包（clientset、informers、 listers），都是 Kubernetes 为 Network 类型生成的客户端库，这些库会在后面编写自定义控制器的时候用到。

>可以用 `kubebuild` 自动生成项目框架，添加自己的 CRD 并实现 controller 即可

不过，创建出这样一个自定义 API 对象，我们只是完成了 Kubernetes 声明式 API 的一半工作。接下来的另一半工作是：为这个 API 对象编写一个`自定义控制器`（Custom Controller）。这样， Kubernetes 才能根据 Network API 对象的“增、删、改”操作，在真实环境中做出相应的响应。比如，“创建、删除、修改”真正的 Neutron 网络。而这，正是 Network 这个 API 对象所关注的“业务逻辑”。

“声明式 API”并不像“命令式 API”那样有着明显的执行逻辑。这就使得基于声明式 API 的业务功能实现，往往需要通过控制器模式来“监视”API 对象的变化（比如，创建或者删除 Network），然后以此来决定实际要执行的具体工作。

### 控制器

```go
func main() {
  ...
  
  cfg, err := clientcmd.BuildConfigFromFlags(masterURL, kubeconfig)
  ...
  kubeClient, err := kubernetes.NewForConfig(cfg)
  ...
  networkClient, err := clientset.NewForConfig(cfg)
  ...
  
  networkInformerFactory := informers.NewSharedInformerFactory(networkClient, ...)
  
  controller := NewController(kubeClient, networkClient,
  networkInformerFactory.Samplecrd().V1().Networks())
  
  go networkInformerFactory.Start(stopCh)
 
  if err = controller.Run(2, stopCh); err != nil {
    glog.Fatalf("Error running controller: %s", err.Error())
  }
}
```

可以看到，这个 main 函数主要通过三步完成了初始化并启动一个`自定义控制器`的工作。

第一步：main 函数根据我提供的 `Master 配置`（APIServer 的地址端口和 kubeconfig 的路径），创建一个 Kubernetes 的 client（kubeClient）和 Network 对象的 client（networkClient）。

但是，如果我没有提供 Master 配置呢？这时，main 函数会直接使用一种名叫 `InClusterConfig` 的方式来创建这个 client。这个方式，会假设你的自定义控制器是以 Pod 的方式运行在 Kubernetes 集群里的。

而我在第 15 篇文章《深入解析 Pod 对象（二）：使用进阶》中曾经提到过，Kubernetes 里所有的 Pod 都会以 Volume 的方式自动挂载 Kubernetes 的默认 ServiceAccount。所以，这个控制器就会直接使用默认 `ServiceAccount` 数据卷里的授权信息，来访问 APIServer。

第二步：main 函数为 Network 对象创建一个叫作 `InformerFactory`（即：networkInformerFactory）的工厂，并使用它生成一个 Network 对象的 Informer，传递给控制器。

第三步：main 函数启动上述的 Informer，然后执行 controller.Run，启动自定义控制器。至此，main 函数就结束了。

![](../../reference/pic/controller.webp)

这个控制器要做的第一件事，是从 Kubernetes 的 APIServer 里获取它所关心的对象，也就是我定义的 Network 对象。这个操作，依靠的是一个叫作 `Informer`（可以翻译为：通知器）的代码库完成的。Informer 与 API 对象是一一对应的，所以我传递给自定义控制器的，正是一个 Network 对象的 Informer（Network Informer）。

不知你是否已经注意到，我在创建这个 Informer 工厂的时候，需要给它传递一个 networkClient。事实上，Network Informer 正是使用这个 networkClient，跟 APIServer 建立了连接。

不过，真正负责维护这个连接的，则是 Informer 所使用的 `Reflector` 包。更具体地说，Reflector 使用的是一种叫作 `ListAndWatch` 的方法，来“获取”并“监听”这些 Network 对象实例的变化。

在 ListAndWatch 机制下，一旦 APIServer 端有新的 Network 实例被创建、删除或者更新，Reflector 都会收到“事件通知”。这时，该事件及它对应的 API 对象这个组合，就被称为`增量`（Delta），它会被放进一个 `Delta FIFO Queue`（即：增量先进先出队列）中。而另一方面，Informe 会不断地从这个 Delta FIFO Queue 里读取（Pop）增量。每拿到一个增量，Informer 就会判断这个增量里的事件类型，然后创建或者更新本地对象的缓存。这个缓存，在 Kubernetes 里一般被叫作 `Store`。

比如，如果事件类型是 Added（添加对象），那么 Informer 就会通过一个叫作 `Indexer` 的库把这个增量里的 API 对象保存在本地缓存中，并为它创建索引。相反，如果增量的事件类型是 Deleted（删除对象），那么 Informer 就会从本地缓存中删除这个对象。这个`同步本地缓存`的工作，是 Informer 的第一个职责，也是它最重要的职责。

而 Informer 的第二个职责，则是根据这些事件的类型，触发事先注册好的 `ResourceEventHandler`。这些 Handler，需要在创建控制器的时候注册给它对应的 Informer。

```go

func NewController(
  kubeclientset kubernetes.Interface,
  networkclientset clientset.Interface,
  networkInformer informers.NetworkInformer) *Controller {
  ...
  controller := &Controller{
    kubeclientset:    kubeclientset,
    networkclientset: networkclientset,
    networksLister:   networkInformer.Lister(),
    networksSynced:   networkInformer.Informer().HasSynced,
    workqueue:        workqueue.NewNamedRateLimitingQueue(...,  "Networks"),
    ...
  }
    networkInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
    AddFunc: controller.enqueueNetwork,
    UpdateFunc: func(old, new interface{}) {
      oldNetwork := old.(*samplecrdv1.Network)
      newNetwork := new.(*samplecrdv1.Network)
      if oldNetwork.ResourceVersion == newNetwork.ResourceVersion {
        return
      }
      controller.enqueueNetwork(new)
    },
    DeleteFunc: controller.enqueueNetworkForDelete,
 return controller
}
```

我前面在 main 函数里创建了两个 client（kubeclientset 和 networkclientset），然后在这段代码里，使用这两个 client 和前面创建的 Informer，初始化了自定义控制器。

值得注意的是，在这个自定义控制器里，我还设置了一个`工作队列`（work queue），它正是处于示意图中间位置的 WorkQueue。这个工作队列的作用是，负责同步 Informer 和控制循环之间的数据。实际上，Kubernetes 项目为我们提供了很多个工作队列的实现，你可以根据需要选择合适的库直接使用。

然后，我为 networkInformer 注册了三个 `Handler`（AddFunc、UpdateFunc 和 DeleteFunc），分别对应 API 对象的“添加”“更新”和“删除”事件。而具体的处理操作，都是将该事件对应的 API 对象加入到工作队列中。

需要注意的是，实际入队的并不是 API 对象本身，而是它们的 `Key`，即：该 API 对象的`<namespace>/<name>`。

而我们后面即将编写的`控制循环`，则会不断地从这个工作队列里拿到这些 Key，然后开始执行真正的控制逻辑。

综合上面的讲述，你现在应该就能明白，所谓 `Informer`，其实就是一个**带有本地缓存和索引机制的、可以注册 EventHandler 的 client**。它是自定义控制器跟 APIServer 进行数据同步的重要组件。

更具体地说，Informer 通过一种叫作 `ListAndWatch` 的方法，把 APIServer 中的 API 对象缓存在了本地，并负责更新和维护这个缓存。其中，ListAndWatch 方法的含义是：首先，通过 APIServer 的 LIST API“获取”所有最新版本的 API 对象；然后，再通过 WATCH API 来“监听”所有这些 API 对象的变化。

而通过监听到的事件变化，Informer 就可以实时地更新本地缓存，并且调用这些事件对应的 EventHandler 了。

此外，在这个过程中，每经过 `resyncPeriod` 指定的时间，Informer 维护的本地缓存，都会使用最近一次 LIST 返回的结果`强制更新`一次，从而保证缓存的有效性。在 Kubernetes 中，这个缓存强制更新的操作就叫作：`resync`。

需要注意的是，这个定时 resync 操作，也会触发 Informer 注册的“更新”事件。但此时，这个“更新”事件对应的 Network 对象实际上并没有发生变化，即：新、旧两个 Network 对象的 ResourceVersion 是一样的。在这种情况下，Informer 就不需要对这个更新事件再做进一步的处理了。这也是为什么我在上面的 UpdateFunc 方法里，先判断了一下新、旧两个 Network 对象的版本（ResourceVersion）是否发生了变化，然后才开始进行的入队操作。

接下来，我们就来到了示意图中最后面的`控制循环`（Control Loop）部分，也正是我在 main 函数最后调用 controller.Run() 启动的“控制循环”。它的主要内容如下所示：
```go

func (c *Controller) Run(threadiness int, stopCh <-chan struct{}) error {
 ...
  if ok := cache.WaitForCacheSync(stopCh, c.networksSynced); !ok {
    return fmt.Errorf("failed to wait for caches to sync")
  }
  
  ...
  for i := 0; i < threadiness; i++ {
    go wait.Until(c.runWorker, time.Second, stopCh)
  }
  
  ...
  return nil
}
```

可以看到，启动控制循环的逻辑非常简单：
首先，等待 Informer 完成一次本地缓存的数据同步操作；然后，直接通过 goroutine 启动一个（或者并发启动多个）“无限循环”的任务。而这个“无限循环”任务的每一个循环周期，执行的正是我们真正关心的业务逻辑。

```go
func (c *Controller) runWorker() {
  for c.processNextWorkItem() {
  }
}

func (c *Controller) processNextWorkItem() bool {
  obj, shutdown := c.workqueue.Get()
  
  ...
  
  err := func(obj interface{}) error {
    ...
    if err := c.syncHandler(key); err != nil {
     return fmt.Errorf("error syncing '%s': %s", key, err.Error())
    }
    
    c.workqueue.Forget(obj)
    ...
    return nil
  }(obj)
  
  ...
  
  return true
}

func (c *Controller) syncHandler(key string) error {

  namespace, name, err := cache.SplitMetaNamespaceKey(key)
  ...
  
  network, err := c.networksLister.Networks(namespace).Get(name)
  if err != nil {
    if errors.IsNotFound(err) {
      glog.Warningf("Network does not exist in local cache: %s/%s, will delete it from Neutron ...",
      namespace, name)
      
      glog.Warningf("Network: %s/%s does not exist in local cache, will delete it from Neutron ...",
    namespace, name)
    
     // FIX ME: call Neutron API to delete this network by name.
     //
     // neutron.Delete(namespace, name)
     
     return nil
  }
    ...
    
    return err
  }
  
  glog.Infof("[Neutron] Try to process network: %#v ...", network)
  
  // FIX ME: Do diff().
  //
  // actualNetwork, exists := neutron.Get(namespace, name)
  //
  // if !exists {
  //   neutron.Create(namespace, name)
  // } else if !reflect.DeepEqual(actualNetwork, network) {
  //   neutron.Update(namespace, name)
  // }
  
  return nil
}
```

可以看到，在这个执行周期里（processNextWorkItem），我们首先从工作队列里出队（workqueue.Get）了一个成员，也就是一个 Key（Network 对象的：namespace/name）。然后，在 `syncHandler` 方法中，我使用这个 `Key`，尝试从 Informer 维护的缓存中拿到了它所对应的 Network `对象`。

可以看到，在这里，我使用了 `networksLister` 来尝试获取这个 Key 对应的 Network 对象。这个操作，其实就是在访问本地缓存的索引。

实际上，在 Kubernetes 的源码中，你会经常看到控制器从各种 `Lister` 里获取对象，比如：podLister、nodeLister 等等，它们使用的都是 Informer 和缓存机制。

而如果控制循环从缓存中拿不到这个对象（即：networkLister 返回了 IsNotFound 错误），那就意味着这个 Network 对象的 Key 是通过前面的“删除”事件添加进工作队列的。所以，尽管队列里有这个 Key，但是对应的 Network 对象已经被删除了。这时候，我就需要调用 Neutron 的 API，把这个 Key 对应的 Neutron 网络从真实的集群里删除掉。

而如果能够获取到对应的 Network 对象，我就可以执行控制器模式里的对比“期望状态”和“实际状态”的逻辑了。其中，自定义控制器“千辛万苦”拿到的这个 Network 对象，正是 APIServer 里保存的“`期望状态`”，即：用户通过 YAML 文件提交到 APIServer 里的信息。

当然，在我们的例子里，它已经被 Informer 缓存在了本地。

那么，“`实际状态`”又从哪里来呢？当然是来自于实际的集群了。所以，我们的控制循环需要通过 `Neutron API` 来查询实际的网络情况。

比如，我可以先通过 Neutron 来查询这个 Network 对象对应的真实网络是否存在。如果不存在，这就是一个典型的“期望状态”与“实际状态”不一致的情形。这时，我就需要使用这个 Network 对象里的信息（比如：CIDR 和 Gateway），调用 Neutron API 来创建真实的网络。如果存在，那么，我就要读取这个真实网络的信息，判断它是否跟 Network 对象里的信息一致，从而决定我是否要通过 Neutron 来更新这个已经存在的真实网络。这样，我就通过对比“期望状态”和“实际状态”的差异，完成了一次调协（`Reconcile`）的过程。至此，一个完整的自定义 API 对象和它所对应的自定义控制器，就编写完毕了。

实际上，这套流程不仅可以用在自定义 API 资源上，也完全可以用在 Kubernetes 原生的`默认 API 对象`上。比如，我们在 main 函数里，除了创建一个 Network Informer 外，还可以初始化一个 Kubernetes 默认 API 对象的 Informer 工厂，比如 `Deployment` 对象的 Informer。

```go

func main() {
  ...
  
  kubeInformerFactory := kubeinformers.NewSharedInformerFactory(kubeClient, time.Second*30)
  
  controller := NewController(kubeClient, exampleClient,
  kubeInformerFactory.Apps().V1().Deployments(),
  networkInformerFactory.Samplecrd().V1().Networks())
  
  go kubeInformerFactory.Start(stopCh)
  ...
}
```

在这段代码中，我们首先使用 Kubernetes 的 client（kubeClient）创建了一个工厂；然后，我用跟 Network 类似的处理方法，生成了一个 Deployment Informer；接着，我把 Deployment Informer 传递给了自定义控制器；当然，我也要调用 Start 方法来启动这个 Deployment Informer。而有了这个 Deployment Informer 后，这个控制器也就持有了所有 Deployment 对象的信息。接下来，它既可以通过 deploymentInformer.Lister() 来获取 Etcd 里的所有 Deployment 对象，也可以为这个 Deployment Informer 注册具体的 Handler 来。更重要的是，这就使得在这个自定义控制器里面，我可以通过对自定义 API 对象和默认 API 对象进行协同，从而实现更加复杂的编排功能。比如：用户每创建一个新的 Deployment，这个自定义控制器，就可以为它创建一个对应的 Network 供它使用。

所谓的 Informer，就是一个自带缓存和索引机制，可以触发 Handler 的`客户端库`。这个本地缓存在 Kubernetes 中一般被称为 Store，索引一般被称为 Index。

Informer 使用了 Reflector 包，它是一个可以通过 ListAndWatch 机制获取并监视 API 对象变化的客户端封装。Reflector 和 Informer 之间，用到了一个“增量先进先出队列”进行协同。而 Informer 与你要编写的控制循环之间，则使用了一个工作队列来进行协同。

在实际应用中，除了控制循环之外的所有代码，实际上都是 Kubernetes 为你`自动生成`的，即：pkg/client/{informers, listers, clientset}里的内容。而这些自动生成的代码，就为我们提供了一个可靠而高效地获取 API 对象“期望状态”的编程库。所以，接下来，作为开发者，你就只需要关注如何拿到“实际状态”，然后如何拿它去跟“期望状态”做对比，从而决定接下来要做的业务逻辑即可。


## RBAC

而在 Kubernetes 项目里，我们可以基于插件机制来完成这些工作，而完全不需要修改任何一行代码。不过，你要通过一个`外部插件`，在 Kubernetes 里新增和操作 API 对象，那么就必须先了解一个非常重要的知识：RBAC。

我们知道，Kubernetes 中所有的 API 对象，都保存在 Etcd 里。可是，对这些 API 对象的操作，却一定都是通过访问 `kube-apiserver `实现的。其中一个非常重要的原因，就是你需要 APIServer 来帮助你做授权工作。而在 Kubernetes 项目中，负责完成授权（Authorization）工作的机制，就是 `RBAC`：基于角色的访问控制（Role-Based Access Control）。

而在这里，我只希望你能明确三个最基本的概念。
Role：角色，它其实是一组规则，定义了一组对 Kubernetes API 对象的操作权限。
Subject：被作用者，既可以是“人”，也可以是“机器”，也可以是你在 Kubernetes 里定义的“用户”。
`RoleBinding`：定义了“被作用者”和“角色”的绑定关系。

```yml

kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: mynamespace
  name: example-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]

---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

实际上，Kubernetes 里的“`User`”，也就是“用户”，只是一个授权系统里的`逻辑概念`。它需要通过外部认证服务，比如 Keystone，来提供。或者，你也可以直接给 APIServer 指定一个用户名、密码文件。那么 Kubernetes 的授权系统，就能够从这个文件里找到对应的“用户”了。

```yml

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrole
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-clusterrolebinding
subjects:
- kind: User
  name: example-user
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: example-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

而正如我前面介绍过的，在大多数时候，我们其实都不太使用“用户”这个功能，而是直接使用 Kubernetes 里的“内置用户”。这个由 Kubernetes 负责管理的“内置用户”，正是我们前面曾经提到过的：`ServiceAccount`。

```yml

apiVersion: v1
kind: ServiceAccount
metadata:
  namespace: mynamespace
  name: example-sa
---

kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-rolebinding
  namespace: mynamespace
subjects:
- kind: ServiceAccount
  name: example-sa
  namespace: mynamespace
roleRef:
  kind: Role
  name: example-role
  apiGroup: rbac.authorization.k8s.io
```

>已经不会自动创建secret，但会有serviceAccountToken挂载在pod上
可以看到，Kubernetes 会为一个 ServiceAccount `自动创建`并分配一个 Secret 对象，即：上述 ServiceAcount 定义里最下面的 secrets 字段。这个 Secret，就是这个 ServiceAccount 对应的、用来跟 APIServer 进行交互的授权文件，我们一般称它为：`Token`。Token 文件的内容一般是证书或者密码，它以一个 Secret 对象的方式保存在 Etcd 当中。

这时候，用户的 Pod，就可以声明使用这个 ServiceAccount 了，比如下面这个例子：
```yaml

apiVersion: v1
kind: Pod
metadata:
  namespace: mynamespace
  name: sa-token-test
spec:
  containers:
  - name: nginx
    image: nginx:1.7.9
  serviceAccountName: example-sa
```

如果一个 Pod `没有声明 serviceAccountName`，Kubernetes 会自动在它的 Namespace 下创建一个名叫 default 的默认 ServiceAccount，然后分配给这个 Pod。但在这种情况下，这个默认 ServiceAccount 并没有关联任何 Role。也就是说，此时它有访问 APIServer 的`绝大多数权限`。当然，这个访问所需要的 Token，还是默认 ServiceAccount 对应的 Secret 对象为它提供的

所以，在`生产环境`中，我强烈建议你为所有 Namespace 下的**默认 ServiceAccount，绑定一个只读权限的 Role**。这个具体怎么做，就当作思考题留给你了。

除了前面使用的“用户”（User），Kubernetes 还拥有“用户组”（`Group`）的概念，也就是一组“用户”的意思。如果你为 Kubernetes 配置了`外部认证服务`的话，这个“用户组”的概念就会由外部认证服务提供。

而对于 Kubernetes 的内置“用户”`ServiceAccount` 来说，上述“用户组”的概念也同样适用。

实际上，一个 ServiceAccount，在 Kubernetes 里对应的“用户”的名字是：
`system:serviceaccount:<Namespace名字>:<ServiceAccount名字>`
而它对应的内置“用户组”的名字，就是：
`system:serviceaccounts:<Namespace名字>`

比如，现在我们可以在 RoleBinding 里定义如下的 subjects：
```yml
subjects:
- kind: Group
  name: system:serviceaccounts:mynamespace
  apiGroup: rbac.authorization.k8s.io
```
**这就意味着这个 Role 的权限规则，作用于 mynamespace 里的所有 ServiceAccount**。这就用到了“用户组”的概念。

```yml
subjects:
- kind: Group
  name: system:serviceaccounts
  apiGroup: rbac.authorization.k8s.io
```
**就意味着这个 Role 的权限规则，作用于整个系统里的所有 ServiceAccount**。

最后，值得一提的是，在 Kubernetes 中已经内置了很多个`为系统保留的 ClusterRole`，它们的名字都以 `system:` 开头。你可以通过 kubectl get clusterroles 查看到它们。一般来说，这些系统 ClusterRole，是绑定给 Kubernetes 系统组件对应的 ServiceAccount 使用的。

除此之外，Kubernetes 还提供了四个`预先定义`好的 ClusterRole 来供用户直接使用：`cluster-admin；admin；edit；view`。通过它们的名字，你应该能大致猜出它们都定义了哪些权限。比如，这个名叫 view 的 ClusterRole，就规定了被作用者只有 Kubernetes API 的只读权限。

而我还要提醒你的是，上面这个 cluster-admin 角色，对应的是整个 Kubernetes 项目中的`最高权限`（verbs=*）

## operator

接下来，我就以 `Etcd Operator` 为例，来为你讲解一下 Operator 的工作原理和编写方法。
>例子过期了

Etcd Operator 的使用方法非常简单，只需要两步即可完成：

第一步，将这个 Operator 的代码 Clone 到本地：
`git clone https://github.com/coreos/etcd-operator`

第二步，将这个 Etcd Operator 部署在 Kubernetes 集群里。

不过，在部署 Etcd Operator 的 Pod 之前，你需要先执行这样一个脚本：`example/rbac/create_role.sh`

不用我多说你也能够明白：这个脚本的作用，就是为 Etcd Operator 创建 RBAC 规则。这是因为，Etcd Operator 需要访问 Kubernetes 的 APIServer 来创建对象。

更具体地说，上述脚本为 Etcd Operator 定义了如下所示的`权限`：
对 Pod、Service、PVC、Deployment、Secret 等 API 对象，有所有权限；
对 CRD 对象，有所有权限；
对属于 etcd.database.coreos.com 这个 API Group 的 CR（Custom Resource）对象，有所有权限。
而 Etcd Operator 本身，其实就是一个 Deployment，它的 YAML 文件如下所示：
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: etcd-operator
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: etcd-operator 
  template:
    metadata:
      labels:
        app: etcd-operator
    spec:
      containers:
      - name: etcd-operator
        image: quay.io/coreos/etcd-operator:v0.9.4
        command:
        - etcd-operator
        # Uncomment to act for resources in all namespaces. More information in doc/user/clusterwide.md
        #- -cluster-wide
        env:
        - name: MY_POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: MY_POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

而一旦 Etcd Operator 的 Pod 进入了 Running 状态，你就会发现，有一个 `CRD` 被自动创建了出来

上面的deployment创建pod，pod里是一个自定义控制器，还会注册crd

可以看到，这个 CRD 相当于告诉了 Kubernetes：接下来，如果有 API 组（Group）是etcd.database.coreos.com、API 资源类型（Kind）是“EtcdCluster”的 YAML 文件被提交上来，你可一定要认识啊。

所以说，通过上述两步操作，你实际上是在 Kubernetes 里添加了一个名叫 `EtcdCluster 的自定义资源类型`。而 **Etcd Operator 本身，就是这个自定义资源类型对应的自定义控制器**。

而当 Etcd Operator 部署好之后，接下来在这个 Kubernetes 里创建一个 Etcd 集群的工作就非常简单了。你只需要编写一个 EtcdCluster 的 YAML 文件，然后把它提交给 Kubernetes 即可，如下所示：`kubectl apply -f example/example-etcd-cluster.yaml`

不难想到，这个文件里定义的，正是 EtcdCluster 这个 CRD 的一个具体实例，也就是一个` Custom Resource`（CR）。而它的内容非常简单，如下所示：
```yml
apiVersion: "etcd.database.coreos.com/v1beta2"
kind: "EtcdCluster"
metadata:
  name: "example-etcd-cluster"
spec:
  size: 3
  version: "3.2.13"
```
Operator 的`工作原理`，实际上是利用了 Kubernetes 的自定义 API 资源（CRD），来描述我们想要部署的“有状态应用”；然后在自定义控制器里，根据自定义 API 对象的变化，来完成具体的部署和运维工作。

所以，编写一个 Etcd Operator，与我们前面`编写一个自定义控制器的过程，没什么不同`。

总结：
部署operator，首先需要一个自定义控制器，由deployment管理的pod实现，并由pod提交crd，这个流程需要rbac认证，最后创建cr

Etcd Operator 部署 Etcd 集群，采用的是`静态集群`（Static）的方式。静态集群的好处是，它不必依赖于一个额外的`服务发现机制`来组建集群，非常适合本地容器化部署。而它的难点，则在于你必须在部署的时候，就规划好这个集群的`拓扑结构`，并且能够知道这些节点`固定的 IP 地址`。如：

```shell
etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
  
etcd --name infra1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
  --listen-peer-urls http://10.0.1.11:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
  
etcd --name infra2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
  --listen-peer-urls http://10.0.1.12:2380 \
...
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
```
而我们要编写的 Etcd Operator，就是要把上述过程自动化。这其实等同于：`用代码来生成每个 Etcd 节点 Pod 的启动命令，然后把它们启动起来`。

省略具体实现代码。。。

可以看到，Etcd Operator 启动要做的第一件事（ c.initResource），是`创建 EtcdCluster 对象所需要的 CRD`，即：前面提到的etcdclusters.etcd.database.coreos.com。这样 Kubernetes 就能够“认识”EtcdCluster 这个自定义 API 资源了。

而接下来，Etcd Operator 会定义一个 EtcdCluster 对象的` Informer`。

具体来讲，我们在`控制循环里执行的业务逻辑`，往往是比较耗时间的。比如，创建一个真实的 Etcd 集群。而 Informer 的 WATCH 机制对 API 对象变化的响应，则非常迅速。所以，控制器里的业务逻辑就很可能会拖慢 Informer 的执行周期，甚至可能 Block 它。而要协调这样两个快、慢任务的一个典型解决方法，就是引入一个`工作队列`。

由于 Etcd Operator 里`没有工作队列`，那么在它的 EventHandler 部分，就不会有什么入队操作，而直接就是每种事件对应的具体的业务逻辑了。不过，Etcd Operator 在业务逻辑的实现方式上，`与常规的自定义控制器略有不同`。我把在这一部分的工作原理，提炼成了一个详细的流程图，如下所示：

![](https://static001.geekbang.org/resource/image/e7/36/e7f2905ae46e0ccd24db47c915382536.jpg?wh=1920*1080)

可以看到，Etcd Operator 的特殊之处在于，它`为每一个 EtcdCluster 对象，都启动了一个控制循环`，“并发”地响应这些对象的变化。显然，这种做法不仅可以简化 Etcd Operator 的代码实现，还有助于提高它的响应速度。

当这个 YAML 文件第一次被提交到 Kubernetes 之后，Etcd Operator 的 Informer，就会立刻“感知”到一个新的 EtcdCluster 对象被创建了出来。所以，EventHandler 里的“添加”事件会被触发。而这个 Handler 要做的操作也很简单，即：在 Etcd Operator 内部创建一个对应的 Cluster 对象（`cluster.New`），比如流程图里的 Cluster1。这个 Cluster 对象，就是一个 Etcd 集群在 Operator 内部的描述，所以它与真实的 Etcd 集群的生命周期是一致的。

而一个 Cluster 对象需要具体负责的，其实有两个工作。

其中，第一个工作只在该 Cluster 对象第一次被创建的时候才会执行。

这个工作，就是我们前面提到过的 `Bootstrap`，即：创建一个单节点的种子集群。由于种子集群只有一个节点，所以这一步直接就会生成一个 Etcd 的 Pod 对象。这个 Pod 里有一个 InitContainer，负责检查 Pod 的 `DNS` 记录是否正常。如果检查通过，用户容器也就是 Etcd 容器就会启动起来。

而这个 Etcd 容器最重要的部分，当然就是它的`启动命令`了。

可以看到，在这些启动参数（比如：initial-cluster）里，Etcd Operator 只会使用 Pod 的 DNS 记录，而不是它的 IP 地址。这当然是因为，在 Operator 生成上述启动命令的时候，Etcd 的 Pod 还没有被创建出来，它的 IP 地址自然也无从谈起。这也就意味着，每个 Cluster 对象，都会事先创建一个与该 EtcdCluster 同名的 Headless Service。这样，Etcd Operator 在接下来的所有创建 Pod 的步骤里，就都可以使用 Pod 的 `DNS 记录来代替它的 IP 地址`了。

Cluster 对象的第二个工作，则是启动该集群所对应的`控制循环`。这个控制循环每隔一定时间，就会执行一次下面的 Diff 流程。首先，控制循环要获取到所有正在运行的、属于这个 Cluster 的 Pod 数量，也就是该 Etcd 集群的“实际状态”。而这个 Etcd 集群的“期望状态”，正是用户在 EtcdCluster 对象里定义的 size。所以接下来，控制循环会对比这两个状态的差异。

如果实际的 Pod 数量不够，那么控制循环就会执行一个添加成员节点的操作（即：上述流程图中的 addOneMember 方法）；反之，就执行删除成员节点的操作（即：上述流程图中的 removeOneMember 方法）。

以 `addOneMember` 方法为例，它执行的流程如下所示：
生成一个新节点的 Pod 的名字，比如：example-etcd-cluster-v6v6s6stxd；
调用 Etcd Client，执行前面提到过的 etcdctl member add example-etcd-cluster-v6v6s6stxd 命令；
使用这个 Pod 名字，和已经存在的所有节点列表，组合成一个新的 initial-cluster 字段的值；
使用这个 initial-cluster 的值，生成这个 Pod 里 Etcd 容器的启动命令。

这样，当这个容器启动之后，一个新的 Etcd 成员节点就被加入到了集群当中。控制循环会重复这个过程，直到正在运行的 Pod 数量与 EtcdCluster 指定的 size 一致。在有了这样一个与 EtcdCluster 对象一一对应的控制循环之后，你后续对这个 EtcdCluster 的任何修改，比如：修改 size 或者 Etcd 的 version，它们对应的更新事件都会由这个 Cluster 对象的控制循环进行处理。

以上，就是一个 Etcd Operator 的工作原理了。

如果对比一下 Etcd Operator 与我在第 20 篇文章《深入理解 StatefulSet（三）：有状态应用实践》中讲解过的 MySQL StatefulSet 的话，你可能会有`两个问题`。

第一个问题是，在 StatefulSet 里，它为 Pod 创建的名字是带编号的，这样就把整个集群的拓扑状态固定了下来（比如：一个三节点的集群一定是由名叫 web-0、web-1 和 web-2 的三个 Pod 组成）。可是，在 Etcd Operator 里，为什么我们使用随机名字就可以了呢？

这是因为，Etcd Operator 在每次添加 Etcd 节点的时候，都会先执行 `etcdctl member add <Pod 名字 >`；每次删除节点的时候，则会执行 `etcdctl member remove <Pod 名字 >`。这些操作，其实就会更新 Etcd 内部维护的拓扑信息，所以 Etcd Operator `无需在集群外部通过编号来固定这个拓扑关系`。

第二个问题是，为什么我没有在 EtcdCluster 对象里声明 Persistent Volume？难道，我们不担心节点宕机之后 Etcd 的数据会丢失吗？

我们知道，Etcd 是一个基于 `Raft` 协议实现的高可用 Key-Value 存储。根据 Raft 协议的设计原则，当 Etcd 集群里只有半数以下（在我们的例子里，小于等于一个）的节点失效时，当前集群依然可以正常工作。此时，Etcd Operator 只需要通过控制循环创建出新的 Pod，然后将它们加入到现有集群里，就完成了“期望状态”与“实际状态”的调谐工作。这个集群，是一直可用的 。

但是，当这个 Etcd 集群里有半数以上（在我们的例子里，大于等于两个）的节点失效的时候，这个集群就会丧失数据写入的能力，从而进入“不可用”状态。此时，即使 Etcd Operator 创建出新的 Pod 出来，Etcd 集群本身也无法自动恢复起来。

这个时候，我们就必须使用 Etcd 本身的备份数据来对集群进行恢复操作。在有了 Operator 机制之后，上述 Etcd 的备份操作，是由一个单独的 `Etcd Backup Operator` 负责完成的。

需要注意的是，每当你创建一个 EtcdBackup 对象（backup_cr.yaml），就相当于为它所指定的 Etcd 集群做了一次备份。EtcdBackup 对象的 etcdEndpoints 字段，会指定它要备份的 Etcd 集群的访问地址。

所以，在实际的环境里，我建议你把最后这个备份操作，编写成一个 Kubernetes 的 CronJob 以便定时运行。而当 Etcd 集群发生了故障之后，你就可以通过创建一个 `EtcdRestore` 对象来完成恢复操作。当然，这就意味着你也需要事先启动 `Etcd Restore Operator`。


## PV PVC

```yml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    server: 10.244.1.4
    path: "/"
---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: manual
  resources:
    requests:
      storage: 1Gi
```

而用户创建的 PVC 要真正被容器使用起来，就必须先和某个符合条件的 PV 进行绑定。

这里要检查的条件，包括两部分：
第一个条件，当然是 PV 和 PVC 的 spec 字段。比如，PV 的存储（storage）大小，就必须满足 PVC 的要求。
而第二个条件，则是 PV 和 PVC 的 `storageClassName` 字段必须一样。这个机制我会在本篇文章的最后一部分专门介绍。

在 Kubernetes 中，实际上存在着一个专门处理持久化存储的控制器，叫作 `Volume Controller`。这个 Volume Controller 维护着多个控制循环，其中有一个循环，扮演的就是撮合 PV 和 PVC 的“红娘”的角色。它的名字叫作 **PersistentVolumeController**。

PersistentVolumeController 会不断地查看当前每一个 PVC，是不是已经处于 Bound（已绑定）状态。如果不是，那它就会遍历所有的、可用的 PV，并尝试将其与这个“单身”的 PVC 进行绑定。这样，Kubernetes 就可以保证用户提交的每一个 PVC，只要有合适的 PV 出现，它就能够很快进入绑定状态，从而结束“单身”之旅。
而所谓将一个 PV 与 PVC 进行“`绑定`”，其实就是将这个 PV 对象的名字，填在了 PVC 对象的 `spec.volumeName` 字段上。

### volume被pod使用的处理流程

**当一个 Pod 调度到一个节点上之后，kubelet 就要负责为这个 Pod 创建它的 Volume 目录**。默认情况下，kubelet 为 Volume 创建的目录是如下所示的一个宿主机上的路径：
`/var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>`

如果你的 Volume 类型是`远程块存储`，比如 Google Cloud 的 Persistent Disk（GCE 提供的远程磁盘服务），那么 kubelet 就需要先调用 Goolge Cloud 的 API，将它所提供的 Persistent Disk 挂载到 Pod 所在的宿主机上。

这一步为虚拟机挂载远程磁盘的操作，对应的正是“两阶段处理”的第一阶段。在 Kubernetes 中，我们把这个阶段称为 **Attach**。

>attach理解为为虚拟机添加一块硬盘连接上去，没有执行挂载，只是能看到硬盘
mount理解为在格式化之后在宿主机上挂载，但是没有挂载到pod中，需要kubelet通过cri来挂载，也就是docker来实现挂载

Attach 阶段完成后，为了能够使用这个远程磁盘，kubelet 还要进行第二个操作，即：**格式化这个磁盘设备，然后将它挂载到宿主机指定的挂载点上**。不难理解，这个挂载点，正是我在前面反复提到的 Volume 的宿主机目录。所以，这一步相当于执行：
```shell
# 通过lsblk命令获取磁盘设备ID
$ sudo lsblk
# 格式化成ext4格式
$ sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/<磁盘设备ID>
# 挂载到挂载点
$ sudo mkdir -p /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>
```

而如果你的 Volume 类型是**远程文件存储（比如 NFS）**的话，kubelet 的处理过程就会更简单一些。因为在这种情况下，kubelet 可以跳过“第一阶段”（Attach）的操作，这是因为一般来说，远程文件存储并没有一个“存储设备”需要挂载在宿主机上。所以，kubelet 会直接从“第二阶段”（Mount）开始准备宿主机上的 Volume 目录。
相当于执行：
```shell
$ mount -t nfs <NFS服务器地址>:/ /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字> 
```

而经过了“两阶段处理”，我们就得到了一个“持久化”的 Volume 宿主机目录。所以，接下来，kubelet 只要把这个 Volume 目录通过 `CRI` 里的 Mounts 参数，传递给 Docker，然后就可以为 Pod 里的容器挂载这个“持久化”的 Volume 了。其实，这一步相当于执行了如下所示的命令：
```shell
$ docker run -v /var/lib/kubelet/pods/<Pod的ID>/volumes/kubernetes.io~<Volume类型>/<Volume名字>:/<容器内的目标目录> 我的镜像 ...
```

备注：对应地，在删除一个 PV 的时候，Kubernetes 也需要 Unmount 和 Dettach 两个阶段来处理。这个过程我就不再详细介绍了，执行“反向操作”即可。

在 Kubernetes 中，上述关于 PV 的“两阶段处理”流程，是靠**独立于 kubelet 主控制循环（Kubelet Sync Loop）之外的两个控制循环来实现的**。

其中，“第一阶段”的 Attach（以及 Dettach）操作，是由 **Volume Controller** 负责维护的，这个控制循环的名字叫作：**AttachDetachController**。而**它的作用，就是不断地检查每一个 Pod 对应的 PV，和这个 Pod 所在宿主机之间挂载情况。从而决定，是否需要对这个 PV 进行 Attach（或者 Dettach）操作**。

需要注意，**作为一个 Kubernetes 内置的控制器，Volume Controller 自然是 `kube-controller-manager` 的一部分**。所以，AttachDetachController 也一定是运行在 Master 节点上的。当然，Attach 操作只需要调用公有云或者具体存储项目的 API，并不需要在具体的宿主机上执行操作，所以这个设计没有任何问题。

而“第二阶段”的 Mount（以及 Unmount）操作，必须发生在 Pod 对应的宿主机上，所以它必须是 kubelet 组件的一部分。这个控制循环的名字，叫作：**VolumeManagerReconciler**，它运行起来之后，是一个独立于 kubelet 主循环的 Goroutine。通过这样将 Volume 的处理同 kubelet 的主循环解耦，Kubernetes 就避免了这些耗时的远程挂载操作拖慢 kubelet 的主控制循环，进而导致 Pod 的创建效率大幅下降的问题。

实际上，kubelet 的一个`主要设计原则`，就是它的主控制循环绝对不可以被 block。这个思想，我在后续的讲述容器运行时的时候还会提到。

## storageclass

具体地说，StorageClass 对象会定义如下两个部分内容：
第一，PV 的属性。比如，存储类型、Volume 的大小等等。
第二，创建这种 PV 需要用到的存储插件。比如，Ceph 等等。

有了这样两个信息之后，Kubernetes 就能够根据用户提交的 PVC，找到一个对应的 StorageClass 了。然后，Kubernetes 就会调用该 StorageClass 声明的存储插件，创建出需要的 PV。

```yml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: block-service
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

有了 `Dynamic Provisioning` 机制，运维人员只需要在 Kubernetes 集群里创建出数量有限的 StorageClass 对象就可以了。这就好比，运维人员在 Kubernetes 集群里创建出了各种各样的 PV 模板。这时候，当开发人员提交了包含 StorageClass 字段的 PVC 之后，Kubernetes 就会根据这个 StorageClass 创建出对应的 PV。

需要注意的是，**StorageClass 并不是专门为了 Dynamic Provisioning 而设计的**。比如，在本篇一开始的例子里，我在 PV 和 PVC 里都声明了 storageClassName=manual。而我的集群里，`实际上并没有一个名叫 manual 的 StorageClass 对象`。这完全没有问题，这个时候 Kubernetes 进行的是 `Static Provisioning`，但在做绑定决策的时候，它依然会考虑 PV 和 PVC 的 StorageClass 定义。

>只要sc名字匹配，就会进行匹配，而不是实际存在这样的sc
使用sc可以将pvc和pv对应起来，即使不使用dynamic provisioning

实际上，如果你的集群已经开启了名叫 **DefaultStorageClass 的 Admission Plugin**，它就会为 PVC 和 PV 自动添加一个默认的 StorageClass；否则，PVC 的 storageClassName 的值就是“”，这也意味着它只能够跟 storageClassName 也是“”的 PV 进行绑定。


总结：

容器持久化存储涉及的概念比较多，试着总结一下整体流程。

用户提交请求创建pod，Kubernetes发现这个pod声明使用了PVC，那就靠PersistentVolumeController帮它找一个PV配对。

没有现成的PV，就去找对应的StorageClass，帮它新创建一个PV，然后和PVC完成绑定。

新创建的PV，还只是一个API 对象，需要经过“两阶段处理”变成宿主机上的“持久化 Volume”才真正有用：
第一阶段由运行在master上的AttachDetachController负责，为这个PV完成 Attach 操作，为宿主机挂载远程磁盘；
第二阶段是运行在每个节点上kubelet组件的内部，把第一步attach的远程磁盘 mount 到宿主机目录。这个控制循环叫VolumeManagerReconciler，运行在独立的Goroutine，不会阻塞kubelet主循环。

完成这两步，PV对应的“持久化 Volume”就准备好了，POD可以正常启动，将“持久化 Volume”挂载在容器内指定的路径。

## Local PV

用户希望 Kubernetes 能够`直接使用宿主机上的本地磁盘目录`，而不依赖于远程存储服务，来提供“持久化”的容器 Volume。

不难想象，Local Persistent Volume 的设计，主要面临两个难点。

第一个难点在于：如何把本地磁盘抽象成 PV。可能你会说，`Local Persistent Volume，不就等同于 hostPath 加 NodeAffinity 吗`？

比如，一个 Pod 可以声明使用类型为 Local 的 PV，而这个 PV 其实就是一个 hostPath 类型的 Volume。如果这个 hostPath 对应的目录，已经在节点 A 上被事先创建好了。那么，我只需要再给这个 Pod 加上一个 nodeAffinity=nodeA，不就可以使用这个 Volume 了吗？

事实上，你**绝不应该把一个宿主机上的目录当作 PV 使用**。这是因为，这种本地目录的存储行为完全不可控，它所在的磁盘随时都可能被应用写满，甚至造成整个宿主机宕机。而且，不同的本地目录之间也缺乏哪怕最基础的 `I/O 隔离机制`。

所以，一个 Local Persistent Volume 对应的存储介质，一定是一块**额外挂载**在宿主机的磁盘或者块设备（“额外”的意思是，它不应该是宿主机根目录所使用的主硬盘）。这个原则，我们可以称为“一个 PV 一块盘”。

第二个难点在于：调度器如何保证 Pod 始终能被正确地调度到它所请求的 Local Persistent Volume 所在的节点上呢？

造成这个问题的原因在于，对于**常规的 PV 来说，Kubernetes 都是先调度 Pod 到某个节点上，然后，再通过“两阶段处理”来“持久化”这台机器上的 Volume 目录，进而完成 Volume 目录与容器的绑定挂载**。

可是，对于 Local PV 来说，节点上可供使用的磁盘（或者块设备），必须是运维人员提前准备好的。它们在不同节点上的挂载情况可以完全不同，甚至有的节点可以没这种磁盘。所以，这时候，调度器就必须能够知道所有节点与 Local Persistent Volume 对应的磁盘的关联关系，然后根据这个信息来调度 Pod。这个原则，我们可以称为**在调度的时候考虑 Volume 分布**。

在 Kubernetes 的调度器里，有一个叫作 `VolumeBindingChecker` 的过滤条件专门负责这个事情。在 Kubernetes v1.11 中，这个过滤条件已经默认开启了。基于上述讲述，在开始使用 Local Persistent Volume 之前，你首先需要在集群里配置好磁盘或者块设备。在公有云上，这个操作等同于给虚拟机额外挂载一个磁盘，比如 GCE 的 Local SSD 类型的磁盘就是一个典型例子。

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: example-pv
spec:
  capacity:
    storage: 5Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-storage
  local:
    path: /mnt/disks/vol1
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - node-1
```

可以看到，这个 PV 的定义里：`local 字段`，指定了它是一个 Local Persistent Volume；而 path 字段，指定的正是这个 PV 对应的本地磁盘的路径，即：/mnt/disks/vol1。
>可以挂载内存盘来测试

当然了，这也就意味着如果 Pod 要想使用这个 PV，那它就必须运行在 node-1 上。所以，在这个 PV 的定义里，需要有一个 `nodeAffinity` 字段指定 node-1 这个节点的名字。这样，调度器在调度 Pod 的时候，就能够知道一个 PV 与节点的对应关系，从而做出正确的选择。这正是 Kubernetes 实现“`在调度的时候就考虑 Volume 分布`”的主要方法。
```yml

kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
这个 StorageClass 的名字，叫作 local-storage。需要注意的是，在它的 provisioner 字段，我们指定的是 **no-provisioner**。这是因为 Local Persistent Volume 目前尚不支持 Dynamic Provisioning，所以它没办法在用户创建 PVC 的时候，就自动创建出对应的 PV。也就是说，我们前面创建 PV 的操作，是不可以省略的。

与此同时，这个 StorageClass 还定义了一个 volumeBindingMode=WaitForFirstConsumer 的属性。它是 **Local Persistent Volume 里一个非常重要的特性，即：延迟绑定**。

>storageclass的作用更像是一个pv和pvc关系的处理机制，而不是单纯的动态供应

我们知道，当你提交了 PV 和 PVC 的 YAML 文件之后，Kubernetes 就会根据它们俩的属性，以及它们指定的 StorageClass 来进行绑定。只有绑定成功后，Pod 才能通过声明这个 PVC 来使用对应的 PV。可是，如果你使用的是 Local Persistent Volume 的话，就会发现，这个流程根本行不通。

比如，现在你有一个 Pod，它声明使用的 PVC 叫作 pvc-1。并且，我们规定，这个 Pod 只能运行在 node-2 上。而在 Kubernetes 集群中，有两个属性（比如：大小、读写权限）相同的 Local 类型的 PV。其中，第一个 PV 的名字叫作 pv-1，它对应的磁盘所在的节点是 node-1。而第二个 PV 的名字叫作 pv-2，它对应的磁盘所在的节点是 node-2。假设现在，Kubernetes 的 Volume 控制循环里，首先检查到了 pvc-1 和 pv-1 的属性是匹配的，于是就将它们俩绑定在一起。然后，你用 kubectl create 创建了这个 Pod。这时候，问题就出现了。调度器看到，这个 Pod 所声明的 pvc-1 已经绑定了 pv-1，而 pv-1 所在的节点是 node-1，根据“调度器必须在调度的时候考虑 Volume 分布”的原则，这个 Pod 自然会被调度到 node-1 上。可是，我们前面已经规定过，这个 Pod 根本不允许运行在 node-1 上。所以。最后的结果就是，这个 Pod 的调度必然会失败。

这就是为什么，在使用 Local Persistent Volume 的时候，我们必须想办法推迟这个“绑定”操作。那么，具体推迟到什么时候呢？答案是：推迟到调度的时候。

所以说，StorageClass 里的 volumeBindingMode=WaitForFirstConsumer 的含义，就是告诉 Kubernetes 里的 Volume 控制循环（“红娘”）：虽然你已经发现这个 StorageClass 关联的 PVC 与 PV 可以绑定在一起，但请不要现在就执行绑定操作（即：设置 PVC 的 VolumeName 字段）。而要等到第一个声明使用该 PVC 的 Pod 出现在调度器之后，调度器再综合考虑所有的调度规则，当然也包括每个 PV 所在的节点位置，来统一决定，这个 Pod 声明的 PVC，到底应该跟哪个 PV 进行绑定。这样，在上面的例子里，由于这个 Pod 不允许运行在 pv-1 所在的节点 node-1，所以它的 PVC 最后会跟 pv-2 绑定，并且 Pod 也会被调度到 node-2 上。所以，**通过这个延迟绑定机制，原本实时发生的 PVC 和 PV 的绑定过程，就被延迟到了 Pod 第一次调度的时候在调度器中进行，从而保证了这个绑定结果不会影响 Pod 的正常调度。**当然，在具体实现中，调度器实际上维护了一个与 Volume Controller 类似的控制循环，专门负责为那些声明了“延迟绑定”的 PV 和 PVC 进行绑定工作。

需要注意的是，我们上面手动创建 PV 的方式，即 **Static 的 PV 管理方式，在删除 PV 时需要按如下流程执行操作**：
删除使用这个 PV 的 Pod；
从宿主机移除本地磁盘（比如，umount 它）；
删除 PVC；
删除 PV。

如果不按照这个流程的话，这个 PV 的删除就会失败。

当然，**由于上面这些创建 PV 和删除 PV 的操作比较繁琐，Kubernetes 其实提供了一个 `Static Provisioner` 来帮助你管理这些 PV**。

那么，当 Static Provisioner 启动后，它就会通过 `DaemonSet`，自动检查每个宿主机的 /mnt/disks 目录。然后，调用 Kubernetes API，为这些目录下面的每一个挂载，创建一个对应的 PV 对象出来。

这个 PV 里的各种定义，比如 StorageClass 的名字、本地磁盘挂载点的位置，都可以通过 provisioner 的`配置文件`指定。当然，provisioner 也会负责前面提到的 PV 的删除工作。而这个 provisioner 本身，其实也是一个我们前面提到过的`External Provisioner`，它的部署方法，在对应的文档里有详细描述。

## 存储插件

在 Kubernetes 中，存储插件的开发有两种方式：FlexVolume 和 CSI。

>csi的driver字段与storageclass的provisioner一致

### flexvolume

接下来，我就先为你剖析一下Flexvolume 的原理和使用方法。举个例子，现在我们要编写的是一个使用 NFS 实现的 FlexVolume 插件。对于一个 FlexVolume 类型的 PV 来说，它的 YAML 文件如下所示：
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-flex-nfs
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  flexVolume:
    driver: "k8s/nfs"
    fsType: "nfs"
    options:
      server: "10.10.0.25" # 改成你自己的NFS服务器地址
      share: "export"
```
可以看到，这个 PV 定义的 Volume 类型是 flexVolume。并且，我们指定了这个 Volume 的 driver 叫作 `k8s/nfs`。这个名字很重要，我后面马上会为你解释它的含义。

而 Volume 的 options 字段，则是一个自定义字段。也就是说，它的类型，其实是 `map[string]string`。所以，你可以在这一部分自由地加上你想要定义的参数。在我们这个例子里，options 字段指定了 NFS 服务器的地址（server: “10.10.0.25”），以及 NFS 共享目录的名字（share: “export”）。当然，你这里定义的所有参数，后面都会被 FlexVolume 拿到。

像这样的一个 PV 被创建后，`一旦和某个 PVC 绑定起来`，这个 FlexVolume 类型的 Volume 就会进入到我们前面讲解过的 `Volume 处理流程`。你应该还记得，这个流程的名字叫作“两阶段处理”，即“Attach 阶段”和“Mount 阶段”。它们的主要作用，是在 Pod 所绑定的宿主机上，完成这个 Volume 目录的持久化过程，比如为虚拟机挂载磁盘（Attach），或者挂载一个 NFS 的共享目录（Mount）。

而在具体的控制循环中，**这两个操作`实际上调用的`，正是 Kubernetes 的 pkg/volume 目录下的`存储插件`（Volume Plugin）**。在我们这个例子里，就是 pkg/volume/flexvolume 这个目录里的代码。

SetUpAt() 的方法，正是 FlexVolume 插件对“`Mount 阶段`”的实现位置。而 SetUpAt() 实际上只做了一件事，那就是封装出了一行命令（即：NewDriverCall），由 `kubelet` 在“Mount 阶段”去执行。在我们这个例子中，kubelet 要通过插件在宿主机上执行的命令，如下所示：
`/usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs mount <mount dir> <json param>`

其中，`/usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs` 就是插件的可执行文件的路径。这个名叫 nfs 的文件，`正是你要编写的插件的实现`。它可以是一个二进制文件，也可以是一个脚本。总之，只要能在宿主机上被执行起来即可。而且这个路径里的 k8s~nfs 部分，正是这个插件在 Kubernetes 里的名字。它是从 driver="k8s/nfs"字段解析出来的。这个 driver 字段的格式是：`vendor/driver`。比如，一家存储插件的提供商（vendor）的名字叫作 k8s，提供的存储驱动（driver）是 nfs，那么 Kubernetes 就会使用 k8s~nfs 来作为插件名。

所以说，当你编写完了 FlexVolume 的实现之后，一定要把它的可执行文件放在`每个节点的插件目录`下。

而紧跟在可执行文件后面的“mount”参数，定义的就是当前的操作。在 FlexVolume 里，这些操作参数的名字是固定的，比如 init、mount、unmount、attach，以及 dettach 等等，分别对应不同的 Volume 处理操作。

而跟在 mount 参数后面的两个字段：`<mount dir>和<json params>`，则是 FlexVolume 必须提供给这条命令的两个执行参数。

其中第一个执行参数`<mount dir>`，正是 kubelet 调用 SetUpAt() 方法传递来的 dir 的值。它代表的是当前正在处理的 Volume 在宿主机上的目录。在我们的例子里，这个路径如下所示：
`/var/lib/kubelet/pods/<Pod ID>/volumes/k8s~nfs/test`

其中，test 正是我们前面定义的 PV 的名字；而 k8s~nfs，则是插件的名字。可以看到，插件的名字正是从你声明的 driver="k8s/nfs"字段里解析出来的。

而第二个执行参数`<json params>`，则是一个 JSON Map 格式的参数列表。我们在前面 PV 里定义的 `options` 字段的值，都会被追加在这个参数里。此外，在 SetUpAt() 方法里可以看到，这个参数列表里还包括了 Pod 的名字、Namespace 等元数据（Metadata）。

在明白了存储插件的调用方式和参数列表之后，这个插件的`可执行文件的实现部分`就非常容易理解了。在这个例子中，我直接编写了一个简单的 `shell 脚本`来作为插件的实现，它对“Mount 阶段”的处理过程，如下所示：
```shell
domount() {
 MNTPATH=$1
 
 NFS_SERVER=$(echo $2 | jq -r '.server')
 SHARE=$(echo $2 | jq -r '.share')
 
 ...
 
 mkdir -p ${MNTPATH} &> /dev/null
 
 mount -t nfs ${NFS_SERVER}:/${SHARE} ${MNTPATH} &> /dev/null
 if [ $? -ne 0 ]; then
  err "{ \"status\": \"Failure\", \"message\": \"Failed to mount ${NFS_SERVER}:${SHARE} at ${MNTPATH}\"}"
  exit 1
 fi
 log '{"status": "Success"}'
 exit 0
}
```

可以看到，当 kubelet 在宿主机上执行“`nfs mount <mount dir> <json params>`”的时候，这个名叫 nfs 的脚本，就可以直接从`<mount dir>`参数里拿到 Volume 在宿主机上的目录，即：MNTPATH=$1。而你在 PV 的 options 字段里定义的 NFS 的服务器地址（options.server）和共享目录名字（options.share），则可以从第二个`<json params>`参数里解析出来。这里，我们使用了 jq 命令，来进行解析工作。

有了这三个参数之后，这个脚本最关键的一步，当然就是执行：`mount -t nfs ${NFS_SERVER}:/${SHARE} ${MNTPATH}` 。这样，一个 NFS 的数据卷就被挂载到了 MNTPATH，也就是 Volume 所在的宿主机目录上，一个持久化的 Volume 目录就处理完了。

需要注意的是，当这个 mount -t nfs 操作完成后，你必须把一个 JOSN 格式的字符串，比如：`{“status”: “Success”}`，返回给调用者，也就是 kubelet。这是 kubelet 判断这次`调用是否成功的唯一依据`。

综上所述，在“Mount 阶段”，kubelet 的 `VolumeManagerReconcile 控制循环`里的一次“调谐”操作的执行流程，如下所示：

`kubelet --> pkg/volume/flexvolume.SetUpAt() --> /usr/libexec/kubernetes/kubelet-plugins/volume/exec/k8s~nfs/nfs mount <mount dir> <json param>`

不过，像这样的 FlexVolume 实现方式，虽然简单，但`局限性`却很大。

比如，跟 Kubernetes 内置的 NFS 插件类似，这个 NFS FlexVolume 插件，也不能支持 Dynamic Provisioning（即：为每个 PVC 自动创建 PV 和对应的 Volume）。除非你再为它编写一个专门的 External Provisioner。再比如，我的插件在执行 mount 操作的时候，可能会生成一些挂载信息。这些信息，在后面执行 unmount 操作的时候会被用到。可是，在上述 FlexVolume 的实现里，你没办法把这些信息保存在一个变量里，等到 unmount 的时候直接使用。这个原因也很容易理解：FlexVolume 每一次对插件可执行文件的调用，都是一次完全独立的操作。所以，我们只能把这些信息写在一个宿主机上的临时文件里，等到 unmount 的时候再去读取。

![](../../reference/pic/vol.webp)
可以看到，在上述体系下，无论是 FlexVolume，还是 Kubernetes 内置的其他存储插件，它们实际上担任的`角色`，仅仅是 Volume 管理中的“Attach 阶段”和“Mount 阶段”的具体执行者。而像 `Dynamic Provisioning` 这样的功能，就不是存储插件的责任，而是 Kubernetes 本身存储管理功能的一部分。

### CSI

相比之下，CSI 插件体系的设计思想，就是把这个 Provision 阶段，以及 Kubernetes 里的一部分存储管理功能，从主干代码里`剥离`出来，做成了几个单独的组件。这些组件会通过 Watch API 监听 Kubernetes 里与存储相关的事件变化，比如 PVC 的创建，来执行具体的存储管理动作。

而这些管理动作，比如“Attach 阶段”和“Mount 阶段”的具体操作，实际上就是通过调用 CSI 插件来完成的。

![](../../reference/pic/csi.webp)

**可以看到，这套存储插件体系多了三个独立的外部组件（`External Components`），即：Driver Registrar、External Provisioner 和 External Attacher，对应的正是从 Kubernetes 项目里面剥离出来的那部分`存储管理功能`**。需要注意的是，External Components 虽然是外部组件，但依然由 Kubernetes 社区来开发和维护。

而图中最右侧的部分，就是需要我们编写代码来实现的 CSI 插件。**一个 CSI 插件只有一个`二进制文件`，但它会以` gRPC` 的方式对外提供三个服务（gRPC Service），分别叫作：CSI Identity、CSI Controller 和 CSI Node**。

我先来为你讲解一下这三个 External Components。

其中，**`Driver Registrar` 组件，负责将插件注册到 kubelet 里面**（这可以类比为，将可执行文件放在插件目录下）。而在具体实现上，Driver Registrar 需要请求 CSI 插件的 `Identity` 服务来获取插件信息。

**而 `External Provisioner` 组件，负责的正是 Provision 阶段**。在具体实现上，External Provisioner 监听（Watch）了 APIServer 里的 PVC 对象。当一个 PVC 被创建时，它就会调用 `CSI Controller` 的 `CreateVolume` 方法，为你创建对应 PV。此外，如果你使用的存储是公有云提供的磁盘（或者块设备）的话，这一步就需要调用公有云（或者块设备服务）的 API 来创建这个 PV 所描述的磁盘（或者块设备）了。

不过，由于 CSI 插件是独立于 Kubernetes 之外的，所以在 CSI 的 API 里不会直接使用 Kubernetes 定义的 PV 类型，而是会自己定义一个`单独的 Volume 类型`。为了方便叙述，在本专栏里，我会把 Kubernetes 里的持久化卷类型叫作 PV，把 CSI 里的持久化卷类型叫作 `CSI Volume`，请你务必区分清楚。

>我的理解是csi volume是pv的实现者，类似于pvc与pv

**最后一个 `External Attacher` 组件，负责的正是Attach 阶段**。在具体实现上，**它监听了 APIServer 里 VolumeAttachment 对象的变化**。VolumeAttachment 对象是 Kubernetes 确认一个 Volume 可以进入“Attach 阶段”的重要标志，我会在下一篇文章里为你详细讲解。

一旦出现了 VolumeAttachment 对象，External Attacher 就会调用 `CSI Controller` 服务的 `ControllerPublish` 方法，完成它所对应的 Volume 的 Attach 阶段。

**而 Volume 的“`Mount 阶段`”，并不属于 External Components 的职责**。当 kubelet 的 VolumeManagerReconciler 控制循环检查到它需要执行 Mount 操作的时候，会通过 pkg/volume/csi 包，直接调用 `CSI Node` 服务完成 Volume 的“Mount 阶段”。

**在实际使用 CSI 插件的时候，我们会将这三个 External Components 作为 `sidecar` 容器和 `CSI 插件`放置在`同一个 Pod` 中**。由于 External Components 对 CSI 插件的调用非常频繁，所以这种 sidecar 的部署方式非常高效。

>rook会把driver registrar和csi插件放在demonset中，把provisioner和attacher还有resizer等容器和csi插件放在deployment里

接下来，我再为你讲解一下 CSI 插件的里三个服务：CSI Identity、CSI Controller 和 CSI Node。

其中，CSI 插件的 CSI Identity 服务，负责对外暴露这个插件本身的信息，如下所示：
```protobuf
service Identity {
  // return the version and name of the plugin
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}
  // reports whether the plugin has the ability of serving the Controller interface
  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}
  // called by the CO just to check whether the plugin is running or not
  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}
```
而 **CSI Controller 服务，定义的则是对 CSI Volume（对应 Kubernetes 里的 PV）的`管理接口`，比如：创建和删除 CSI Volume、对 CSI Volume 进行 Attach/Dettach（在 CSI 里，这个操作被叫作 Publish/Unpublish），以及对 CSI Volume 进行 Snapshot 等**，它们的接口定义如下所示：
```protobuf
service Controller {
  // provisions a volume
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}
    
  // deletes a previously provisioned volume
  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}
    
  // make a volume available on some required node
  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}
    
  // make a volume un-available on some required node
  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}
    
  ...
  
  // make a snapshot
  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}
    
  // Delete a given snapshot
  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}
    
  ...
}
```
需要注意的是，正如我在前面提到的那样，CSI Controller 服务的实际调用者，并不是 Kubernetes（即：通过 pkg/volume/csi 发起 CSI 请求），而是 External Provisioner 和 External Attacher。这两个 External Components，分别通过监听 PVC 和 VolumeAttachement 对象，来跟 Kubernetes 进行协作。

**而 CSI Volume 需要在宿主机上执行的操作，都定义在了 `CSI Node` 服务里面**
```protobuf
service Node {
  // temporarily mount the volume to a staging path
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}
    
  // unmount the volume from staging path
  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}
    
  // mount the volume from staging to target path
  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}
    
  // unmount the volume from staging path
  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}
    
  // stats for the volume
  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}
    
  ...
  
  // Similar to NodeGetId
  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
```

需要注意的是，“Mount 阶段”在 CSI Node 里的接口，是由 `NodeStageVolume` 和 `NodePublishVolume` 两个接口共同实现的。

可以看到，相比于 FlexVolume，CSI 的设计思想，把插件的职责从“两阶段处理”，`扩展成了 Provision、Attach 和 Mount 三个阶段`。其中，**Provision 等价于“创建磁盘”，Attach 等价于“挂载磁盘到虚拟机”，Mount 等价于“将该磁盘格式化后，挂载在 Volume 的宿主机目录上”**。

>备注：CSI 要求插件的名字遵守`反向 DNS`格式。

在有了 CSI 插件之后，Kubernetes 本身依然按照我在第 28 篇文章《PV、PVC、StorageClass，这些到底在说啥？》中所讲述的方式工作，`唯一区别`在于：

当 AttachDetachController 需要进行“Attach”操作时（“Attach 阶段”），它实际上会执行到 pkg/volume/csi 目录中，创建一个 VolumeAttachment 对象，从而触发 External Attacher 调用 CSI Controller 服务的 ControllerPublishVolume 方法。

当 VolumeManagerReconciler 需要进行“Mount”操作时（“Mount 阶段”），它实际上也会执行到 pkg/volume/csi 目录中，直接向 CSI Node 服务发起调用 NodePublishVolume 方法的请求。


```yml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: do-block-storage
  namespace: kube-system
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: com.digitalocean.csi.dobs
```
有了这个 StorageClass，External Provisoner 就会为集群中新出现的 PVC 自动创建出 PV，然后调用 CSI 插件创建出这个 PV 对应的 Volume，这正是 CSI 体系中 Dynamic Provisioning 的实现方式。备注：storageclass.kubernetes.io/is-default-class: "true"的意思，是使用这个 StorageClass 作为默认的持久化存储提供者。不难看到，这个 StorageClass 里唯一引人注意的，是 provisioner=com.digitalocean.csi.dobs 这个字段。显然，这个字段告诉了 Kubernetes，请使用名叫 com.digitalocean.csi.dobs 的 CSI 插件来为我处理这个 StorageClass 相关的所有操作。

举个例子，对于一个部署了 `CSI 存储插件`的 Kubernetes 集群来说：
当用户创建了一个 PVC 之后，你前面部署的 StatefulSet 里的 `External Provisioner` 容器，就会监听到这个 PVC 的诞生，然后调用同一个 Pod 里的 CSI 插件的 CSI Controller 服务的 CreateVolume 方法，为你创建出对应的 PV。

这时候，运行在 Kubernetes Master 节点上的 `Volume Controller`，就会通过 `PersistentVolumeController` 控制循环，发现这对新创建出来的 PV 和 PVC，并且看到它们声明的是同一个 StorageClass。所以，它会把这一对 PV 和 PVC 绑定起来，使 PVC 进入 Bound 状态。

然后，用户创建了一个声明使用上述 PVC 的 Pod，并且这个 Pod 被调度器调度到了宿主机 A 上。这时候，Volume Controller 的 `AttachDetachController` 控制循环就会发现，上述 PVC 对应的 Volume，需要被 Attach 到宿主机 A 上。

所以，AttachDetachController 会创建一个 VolumeAttachment 对象，这个对象携带了宿主机 A 和待处理的 Volume 的名字。这样，StatefulSet 里的 External Attacher 容器，就会监听到这个 VolumeAttachment 对象的诞生。于是，它就会使用这个对象里的宿主机和 Volume 名字，调用同一个 Pod 里的 CSI 插件的 CSI Controller 服务的 ControllerPublishVolume 方法，完成“`Attach` 阶段”。

上述过程完成后，运行在宿主机 A 上的 kubelet，就会通过 VolumeManagerReconciler 控制循环，发现当前宿主机上有一个 Volume 对应的存储设备（比如磁盘）已经被 Attach 到了某个设备目录下。于是 kubelet 就会调用同一台宿主机上的 CSI 插件的 CSI Node 服务的 NodeStageVolume 和 NodePublishVolume 方法，完成这个 Volume 的“`Mount` 阶段”。

我们部署 CSI 插件的**常用原则**是：

第一，通过 `DaemonSet` 在每个节点上都启动一个 CSI 插件，来为 kubelet 提供 CSI Node 服务。这是因为，CSI Node 服务需要被 kubelet 直接调用，所以它要和 kubelet“一对一”地部署起来。此外，在上述 DaemonSet 的定义里面，除了 CSI 插件，我们还以 sidecar 的方式运行着 driver-registrar 这个外部组件。它的作用，是向 kubelet 注册这个 CSI 插件。这个注册过程使用的插件信息，则通过访问同一个 Pod 里的 CSI 插件容器的 Identity 服务获取到。需要注意的是，由于 CSI 插件运行在一个容器里，那么 CSI Node 服务在“Mount 阶段”执行的挂载操作，实际上是发生在这个容器的 Mount Namespace 里的。可是，我们真正希望执行挂载操作的对象，都是宿主机 /var/lib/kubelet 目录下的文件和目录。所以，在定义 DaemonSet Pod 的时候，我们需要把宿主机的 /var/lib/kubelet 以 Volume 的方式挂载进 CSI 插件容器的同名目录下，然后设置这个 Volume 的 mountPropagation=Bidirectional，即开启双向挂载传播，从而将容器在这个目录下进行的挂载操作“传播”给宿主机，反之亦然。

第二，通过 `StatefulSet` 在任意一个节点上再启动一个 CSI 插件，为 External Components 提供 CSI Controller 服务。所以，作为 CSI Controller 服务的调用者，External Provisioner 和 External Attacher 这两个外部组件，就需要以 sidecar 的方式和这次部署的 CSI 插件定义在同一个 Pod 里。你可能会好奇，为什么我们会用 StatefulSet 而不是 Deployment 来运行这个 CSI 插件呢。这是因为，由于 StatefulSet 需要确保应用拓扑状态的稳定性，所以它对 Pod 的更新，是严格保证顺序的，即：只有在前一个 Pod 停止并删除之后，它才会创建并启动下一个 Pod。而像我们上面这样将 StatefulSet 的 replicas 设置为 1 的话，StatefulSet 就会确保 Pod 被删除重建的时候，永远有且只有一个 CSI 插件的 Pod 运行在集群中。这对 CSI 插件的正确性来说，至关重要。

## 网络

在前面讲解容器基础时，我曾经提到过一个 Linux 容器能看见的`网络栈`，实际上是被隔离在它自己的 Network Namespace 当中的。而所谓“网络栈”，就包括了：网卡（Network Interface）、回环设备（Loopback Device）、路由表（Routing Table）和 iptables 规则。对于一个进程来说，这些要素，其实就构成了它发起和响应网络请求的基本环境。

我们前面提到过，这个 eth0 网卡，是一个 `Veth Pair`，它的一端在这个 nginx-1 容器的 Network Namespace 里，而另一端则位于宿主机上（Host Namespace），并且被“插”在了宿主机的 docker0 网桥上。

一旦一张虚拟网卡被“插”在网桥上，它就会变成该网桥的“从设备”。`从设备会被“剥夺”调用网络协议栈处理数据包的资格`，从而“降级”成为网桥上的一个端口。

而这个端口唯一的作用，就是接收流入的数据包，然后把这些数据包的“生杀大权”（比如转发或者丢弃），全部交给对应的网桥。所以，在收到这些 ARP 请求之后，docker0 网桥就会扮演二层交换机的角色，把 ARP 广播转发到其他被“插”在 docker0 上的虚拟网卡上。这样，同样连接在 docker0 上的 nginx-2 容器的网络协议栈就会收到这个 ARP 请求，从而将 172.17.0.3 所对应的 MAC 地址回复给 nginx-1 容器。

docker0 处理转发的过程，则继续扮演二层交换机的角色。此时，docker0 网桥根据数据包的目的 MAC 地址（也就是 nginx-2 容器的 MAC 地址），在它的 **CAM 表（即交换机通过 MAC 地址学习维护的端口和 MAC 地址的对应表）**里查到对应的端口（Port）为：vethb4963f3，然后把数据包发往这个端口。

>此处 MAC 和端口对应表的概念是所有交换机都适用的逻辑概念，英文名称为 MAC learning table 或者 forwarding database (**FDB**)。CAM (content addressable memory) 我的理解是`特殊的硬件设备`，适用 CAM 存储 FDB 可以加速 mac 到端口的查询，但在普通服务器的 Linux bridge 上应该不涉及 CAM。

需要注意的是，在实际的数据传递时，上述数据的传递过程在网络协议栈的不同层次，都有 Linux 内核 `Netfilter 参与其中`。所以，如果感兴趣的话，你可以通过打开 iptables 的 `TRACE 功能`查看到数据包的传输过程，具体方法如下所示：
```
# 在宿主机上执行
$ iptables -t raw -A OUTPUT -p icmp -j TRACE
$ iptables -t raw -A PREROUTING -p icmp -j TRACE
```

通过上述设置，你就可以在 /var/log/syslog 里看到数据包传输的日志了。

**定位方式之一**。 还有一种方式是使用`iptables -nvL <chain> -t <table> --line`来查看对应的chain的报文数目（可以先清空计数：`iptables -Z <chain> -t <table>`）

当你遇到容器连不通“外网”的时候，你都应该先试试 `docker0 网桥能不能 ping 通`，然后查看一下跟 docker0 和 Veth Pair 设备相关的 `iptables 规则是不是有异常`，往往就能够找到问题的答案了。

在上一篇文章中，我为你详细讲解了在单机环境下，Linux 容器网络的实现原理（网桥模式）。并且提到了，在 Docker 的默认配置下，不同宿主机上的容器通过 IP 地址进行互相访问是根本做不到的。而正是为了解决这个容器“`跨主通信`”的问题，社区里才出现了那么多的容器网络方案。



## flannel

Flannel 项目是 CoreOS 公司主推的容器网络方案。事实上，Flannel 项目本身只是一个`框架`，真正为我们提供容器网络功能的，是 Flannel 的`后端实现`。目前，Flannel 支持三种后端实现，分别是：VXLAN；host-gw；UDP。

### UDP模式

而 UDP 模式，是 Flannel 项目最早支持的一种方式，却也是`性能最差`的一种方式。所以，这个模式目前已经被弃用。不过，Flannel 之所以最先选择 UDP 模式，就是因为这种模式是最直接、也是最容易理解的容器跨主网络实现。

而这个 `flannel0` 设备的类型就比较有意思了：它是一个 **TUN** 设备（Tunnel 设备）。在 Linux 中，TUN 设备是一种工作在`三层`（Network Layer）的虚拟网络设备。
TUN 设备的功能非常简单，即：**在操作系统内核和用户应用程序之间传递 IP 包**。以 flannel0 设备为例：像上面提到的情况，当操作系统将一个 IP 包发送给 flannel0 设备之后，flannel0 就会把这个 IP 包，交给创建这个设备的应用程序，也就是 `Flannel 进程`。这是一个从内核态（Linux 操作系统）向用户态（Flannel 进程）的流动方向。
反之，如果 Flannel 进程向 flannel0 设备发送了一个 IP 包，那么这个 IP 包就会出现在宿主机网络栈中，然后根据宿主机的路由表进行下一步处理。这是一个从用户态向内核态的流动方向。

事实上，在由 Flannel 管理的容器网络里，一台宿主机上的所有容器，都属于该宿主机被分配的一个**子网**。
在我们的例子中，Node 1 的子网是 100.96.1.0/24，container-1 的 IP 地址是 100.96.1.2。Node 2 的子网是 100.96.2.0/24，container-2 的 IP 地址是 100.96.2.3。
而这些**子网与宿主机的对应关系，正是保存在 Etcd 当中**，如下所示：
etcdctl ls /coreos.com/network/subnets
/coreos.com/network/subnets/100.96.1.0-24
/coreos.com/network/subnets/100.96.2.0-24
/coreos.com/network/subnets/100.96.3.0-24

所以，flanneld 进程在处理由 flannel0 传入的 IP 包时，就可以根据目的 IP 的地址（比如 100.96.2.3），匹配到对应的子网（比如 100.96.2.0/24），从 Etcd 中找到这个子网对应的宿主机的 IP 地址是 10.168.0.3，如下所示：
etcdctl get /coreos.com/network/subnets/100.96.2.0-24
{"PublicIP":"10.168.0.3"}

所以说，flanneld 在收到 container-1 发给 container-2 的 IP 包之后，就会把这个 IP 包直接封装在一个 `UDP 包`里，然后发送给 Node 2。不难理解，这个 UDP 包的源地址，就是 flanneld 所在的 Node 1 的地址，而目的地址，则是 container-2 所在的宿主机 Node 2 的地址。
当然，这个请求得以完成的原因是，每台宿主机上的 flanneld，都监听着一个 `8285 端口`，所以 flanneld 只要把 UDP 包发往 Node 2 的 8285 端口即可。通过这样一个普通的、宿主机之间的 UDP 通信，一个 UDP 包就从 Node 1 到达了 Node 2。而 Node 2 上监听 8285 端口的进程也是 flanneld，所以这时候，flanneld 就可以从这个 UDP 包里解析出封装在里面的、container-1 发来的原 IP 包。
![](../../reference/pic/flannel1.webp)
需要注意的是，上述流程要正确工作还有一个重要的前提，那就是 `docker0 网桥的地址范围必须是 Flannel 为宿主机分配的子网`。这个很容易实现，以 Node 1 为例，你只需要给它上面的 Docker Daemon 启动时配置如下所示的 bip 参数即可：
FLANNEL_SUBNET=100.96.1.1/24
dockerd --bip=$FLANNEL_SUBNET ...

UDP模式的`缺点`在于使用TUN设备，出现多次内核态和用户态的切换，损失性能

### VXLAN模式

VXLAN 的覆盖网络的设计思想是：**在现有的三层网络之上，“覆盖”一层虚拟的、由内核 VXLAN 模块负责维护的二层网络**，使得连接在这个 VXLAN 二层网络上的“主机”（虚拟机或者容器都可以）之间，可以像在同一个局域网（LAN）里那样自由通信。

当然，实际上，这些“主机”可能分布在不同的宿主机上，甚至是分布在不同的物理机房里。而为了能够在二层网络上打通“隧道”，VXLAN 会在宿主机上设置一个特殊的网络设备作为“隧道”的两端。这个设备就叫作 VTEP，即：VXLAN Tunnel End Point（虚拟隧道端点）。

vtep的地址是**容器子网网络地址**

而 VTEP 设备的作用，其实跟前面的 flanneld 进程非常相似。只不过，它进行封装和解封装的对象，是`二层数据帧`（Ethernet frame）；而且这个工作的执行流程，全部是在`内核`里完成的（因为 VXLAN 本身就是 Linux 内核中的一个模块）。
![](../../reference/pic/flannel2.webp)

可以看到，图中每台宿主机上名叫 flannel.1 的设备，就是 VXLAN 所需的 VTEP 设备，`它既有 IP 地址，也有 MAC 地址`。

源 VTEP 设备”收到“原始 IP 包”后，就要想办法把“原始 IP 包”加上一个目的 MAC 地址，封装成一个二层数据帧，然后发送给“目的 VTEP 设备”（当然，这么做还是因为这个 IP 包的目的地址不是本机）。
这里需要解决的问题就是：“目的 VTEP 设备”的 MAC 地址是什么？

此时，根据前面的路由记录，我们已经知道了“目的 VTEP 设备”的 IP 地址。而要根据三层 IP 地址查询对应的二层 MAC 地址，这正是 `ARP`（Address Resolution Protocol ）表的功能。
而这里要用到的 ARP 记录，也是 flanneld 进程在 Node 2 节点启动时，自动添加在 Node 1 上的。

**最新版本的 Flannel 并不依赖 L3 MISS 事件和 ARP 学习，而会在每台节点启动时把它的 VTEP 设备对应的 ARP 记录，直接下放到其他每台宿主机**。

但是，上面提到的这些 VTEP 设备的 MAC 地址，对于宿主机网络来说并没有什么实际意义。所以上面封装出来的这个数据帧，并不能在我们的宿主机二层网络里传输。为了方便叙述，我们把它称为“`内部数据帧`”（Inner Ethernet Frame）。

所以接下来，Linux 内核还需要再把“内部数据帧”进一步封装成为宿主机网络里的一个普通的数据帧，好让它“载着”“内部数据帧”，通过宿主机的 eth0 网卡进行传输。我们把这次要封装出来的、宿主机对应的数据帧称为“`外部数据帧`”（Outer Ethernet Frame）。

为了实现这个“搭便车”的机制，Linux 内核会在“内部数据帧”前面，加上一个特殊的 VXLAN 头，用来表示这个“乘客”实际上是一个 VXLAN 要使用的数据帧。

而这个 VXLAN 头里有一个重要的标志叫作 `VNI`，它是 VTEP 设备识别某个数据帧是不是应该归自己处理的重要标识。而在 Flannel 中，VNI 的默认值是 1，这也是为何，宿主机上的 VTEP 设备都叫作 flannel.1 的原因，这里的“1”，其实就是 VNI 的值。
然后，Linux 内核会把这个数据帧封装进一个 UDP 包里发出去。

总结：**VXLAN的`VTEP地址`是放在overlay包的内部的，外部的MAC和IP地址都是eth0网卡的，只有到达目的节点后才会解包给目的VTEP，与UDP模式相比将flanneld进程换成了内核里的VTEP**，只是UDP模式中flanneld进程直接管理信息变成维护vtep的信息

不过，不要忘了，一个 flannel.1 设备只知道另一端的 flannel.1 设备的 MAC 地址，却`不知道对应的宿主机地址`是什么。也就是说，这个 UDP 包该发给哪台宿主机呢？

>这个和一般的网络传输不同，一般的网络传输会直接请求ip地址，根据ip地址，发送ARP请求来获取下一跳设备的mac地址，获取mac地址之后会缓存下来，这是一般路由器或者网桥做的事情。但**在容器网络里我们只知道目标容器的ip地址，并不知道宿主机的ip地址，需要将容器的ip地址和宿主机的ip地址关联起来。这个关系由路由规则, fdb存储的数据和ARP记录下来**，这些都是在创建node节点的时候，由各个节点的flanneld创建出来。通过路由规则知道了该将数据包发送到flanneld.1这个vtep设备处理并且知道了对端vtep的ip地址，通过vtep IP地址找到了对端的vtep mac地址，通过对端vtep mac地址查询 bridge fdb list 查到了这个mac地址对应的宿主机地址。

注释：**为了找到宿主机地址，每个节点加入时，为其他节点分发本节点的vtep的arp条目，vtep的mac地址与宿主机ip对应条目**

在这种场景下，flannel.1 设备实际上要扮演一个“网桥”的角色，在二层网络进行 UDP 包的转发。而在 Linux 内核里面，“网桥”设备进行转发的依据，来自于一个叫作 `FDB`（Forwarding Database）的转发数据库。

不难想到，这个 flannel.1“网桥”对应的 FDB 信息，也是 flanneld 进程负责维护的。它的内容可以通过 bridge fdb 命令查看到，如下所示：
```shell
# 在Node 1上，使用“目的VTEP设备”的MAC地址进行查询
bridge fdb show flannel.1 | grep 5e:f8:4f:00:e3:37
5e:f8:4f:00:e3:37 dev flannel.1 dst 10.168.0.3 self permanent
```
可以看到，在上面这条 FDB 记录里，指定了这样一条规则，即：发往我们前面提到的“目的 VTEP 设备”（MAC 地址是 5e:f8:4f:00:e3:37）的二层数据帧，应该通过 flannel.1 设备，发往 IP 地址为 10.168.0.3 的主机。显然，这台主机正是 Node 2，`UDP 包要发往的目的地就找到了`。

![帧结构](https://static001.geekbang.org/resource/image/8c/85/8cede8f74a57617494027ba137383f85.jpg?wh=1864*192)

接下来，Node 1 上的 flannel.1 设备就可以把这个数据帧从 Node 1 的 eth0 网卡发出去。显然，这个帧会经过宿主机网络来到 Node 2 的 eth0 网卡。这时候，Node 2 的内核网络栈会发现这个数据帧里有 VXLAN Header，并且 VNI=1。所以 Linux 内核会对它进行拆包，拿到里面的内部数据帧，然后根据 VNI 的值，把它交给 Node 2 上的 flannel.1 设备。而 flannel.1 设备则会进一步拆包，取出“原始 IP 包”。

接下来就回到了我在上一篇文章中分享的单机容器网络的处理流程。最终，IP 包就进入到了 container-2 容器的 Network Namespace 里。

>备注：如果你想要在我们前面部署的集群中实践 Flannel 的话，可以在 Master 节点上执行如下命令来替换网络插件。
第一步，执行
rm -rf /etc/cni/net.d/*；
第二步，执行
kubectl delete -f "https://cloud.weave.works/k8s/net?k8s-version=1.11"；
第三步，在/etc/kubernetes/manifests/kube-controller-manager.yaml里，为容器启动命令添加如下两个参数：--allocate-node-cidrs=true--cluster-cidr=10.244.0.0/16
第四步， 重启所有 kubelet；
第五步， 执行
kubectl create -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml。

思考题：可以看到，Flannel 通过上述的“隧道”机制，实现了容器之间`三层网络（IP 地址）的连通性`。但是，根据这个机制的工作原理，你认为 Flannel 能负责保证`二层网络`（MAC 地址）的连通性吗？为什么呢？

**二层网络连通是指发送mac层数据包可以寻址到对方，而不依赖ip协议**

**只要所有宿主机属于同一个网段，比如192.168.1.*/24。那么它们就是两层互通的。**

**二层互通是查mac表，三层互通是查路由表**

## cni 

在上一篇文章中，我以 Flannel 项目为例，为你详细讲解了容器跨主机网络的两种实现方法：UDP 和 VXLAN。不难看到，这些例子有一个共性，那就是用户的容器都连接在 docker0 网桥上。而网络插件则在宿主机上创建了一个特殊的设备（UDP 模式创建的是 TUN 设备，VXLAN 模式创建的则是 VTEP 设备），docker0 与这个设备之间，通过 IP 转发（路由表）进行协作。然后，网络插件真正要做的事情，则是通过某种方法，把不同宿主机上的特殊设备连通，从而达到容器跨主机通信的目的。实际上，上面这个流程，也正是 Kubernetes 对容器网络的主要处理方法。只不过，Kubernetes 是通过一个叫作 `CNI` 的接口，维护了一个单独的网桥来代替 `docker0`。这个网桥的名字就叫作：CNI 网桥，它在宿主机上的设备名称默认是：`cni0`。

### 网桥方案

Kubernetes 之所以要设置这样一个与 docker0 网桥功能几乎一样的 `CNI 网桥`，主要原因包括两个方面：一方面，Kubernetes 项目并没有使用 `Docker 的网络模型（CNM）`，所以它并不希望、也不具备配置 docker0 网桥的能力；另一方面，这还与 Kubernetes 如何配置 Pod，也就是 `Infra 容器的 Network Namespace` 密切相关。

CNI 的设计思想，就是：Kubernetes 在启动 Infra 容器之后，就可以直接调用 CNI 网络插件，为这个 Infra 容器的 Network Namespace，配置符合预期的网络栈。

我们在部署 Kubernetes 的时候，有一个步骤是安装 `kubernetes-cni` 包，它的目的就是在宿主机上安装 `CNI 插件所需的基础可执行文件`。在安装完成后，你可以在宿主机的 **/opt/cni/bin** 目录下看到它们

第一类，叫作 **Main 插件**，它是用来`创建具体网络设备`的二进制文件。比如，bridge（网桥设备）、ipvlan、loopback（lo 设备）、macvlan、ptp（Veth Pair 设备），以及 vlan。我在前面提到过的 Flannel、Weave 等项目，都属于“网桥”类型的 CNI 插件。所以在具体的实现中，它们往往会调用 bridge 这个二进制文件。这个流程，我马上就会详细介绍到。

第二类，叫作 **IPAM（IP Address Management）插件**，它是负责`分配 IP 地址`的二进制文件。比如，dhcp，这个文件会向 DHCP 服务器发起请求；host-local，则会使用预先配置的 IP 地址段来进行分配。

第三类，是由 CNI 社区维护的**内置 CNI 插件**。比如：flannel，就是专门为 Flannel 项目提供的 CNI 插件；tuning，是一个通过 sysctl 调整网络设备参数的二进制文件；portmap，是一个通过 iptables 配置端口映射的二进制文件；bandwidth，是一个使用 Token Bucket Filter (TBF) 来进行限流的二进制文件。

首先，实现这个**网络方案本身**。这一部分需要编写的，其实就是 **flanneld 进程里的主要逻辑**。比如，创建和配置 flannel.1 设备、配置宿主机路由、配置 ARP 和 FDB 表里的信息等等。

然后，实现该**网络方案对应的 CNI 插件**。这一部分主要需要做的，就是**配置 Infra 容器里面的网络栈，并把它连接在 CNI 网桥上**。由于 Flannel 项目对应的 CNI 插件已经被内置了，所以它无需再单独安装。

而对于 Weave、Calico 等其他项目来说，我们就必须在安装插件的时候，把对应的 CNI 插件的可执行文件放在 `/opt/cni/bin/` 目录下。
实际上，对于 Weave、Calico 这样的网络方案来说，它们的 DaemonSet 只需要`挂载`宿主机的 /opt/cni/bin/，就可以实现插件可执行文件的安装了。

接下来，你就需要在宿主机上安装 flanneld（**网络方案本身**）。而在这个过程中，**flanneld 启动后会在每台宿主机上生成它对应的 `CNI 配置文件`（它其实是一个 `ConfigMap`），从而告诉 Kubernetes，这个集群要使用 Flannel 作为容器网络方案**。

```shell
cat /etc/cni/net.d/10-flannel.conflist 
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

需要注意的是，**在 Kubernetes 中，`处理容器网络相关的逻辑`并不会在 kubelet 主干代码里执行，而是会在具体的 `CRI`（Container Runtime Interface，容器运行时接口）实现里完成。对于 Docker 项目来说，它的 CRI 实现叫作 dockershim，你可以在 kubelet 的代码里找到它。所以，接下来 dockershim 会加载上述的 CNI 配置文件**。

需要注意，Kubernetes 目前不支持多个 CNI 插件混用。如果你在 CNI 配置目录（/etc/cni/net.d）里放置了多个 CNI 配置文件的话，dockershim 只会加载按字母顺序排序的第一个插件。
但另一方面，CNI 允许你在一个 CNI 配置文件里，通过 `plugins` 字段，定义多个插件进行协作。

这时候，dockershim 会把这个 CNI 配置文件加载起来，并且把列表里的第一个插件、也就是 flannel 插件，设置为默认插件。而在后面的执行过程中，flannel 和 portmap 插件会**按照定义顺序被调用**，从而依次完成“配置容器网络”和“配置端口映射”这两步操作。

当 kubelet 组件需要创建 Pod 的时候，它第一个创建的一定是 Infra 容器。所以在这一步，dockershim 就会先调用 Docker API 创建并启动 Infra 容器，紧接着执行一个叫作 **SetUpPod** 的方法。这个方法的作用就是：**为 CNI 插件准备参数，然后调用 CNI 插件为 Infra 容器配置网络**。

这里要调用的 CNI 插件，就是 `/opt/cni/bin/flannel`；而`调用它所需要的参数`，分为两部分。**第一部分，是由 dockershim 设置的一组 CNI 环境变量。**其中，最重要的环境变量参数叫作：**CNI_COMMAND**。它的取值只有两种：ADD 和 DEL。这个 ADD 和 DEL 操作，就是 CNI 插件唯一需要实现的两个方法。其中 ADD 操作的含义是：把容器添加到 CNI 网络里；DEL 操作的含义则是：把容器从 CNI 网络里移除掉。

而对于`网桥类型`的 CNI 插件来说，这两个操作意味着把容器以 `Veth Pair` 的方式“插”到 CNI 网桥上，或者从网桥上“拔”掉。

CNI 的 ADD 操作需要的`参数`包括：容器里网卡的名字 eth0（CNI_IFNAME）、Pod 的 Network Namespace 文件的路径（CNI_NETNS）、容器的 ID（CNI_CONTAINERID）等。这些参数都属于上述环境变量里的内容。其中，Pod（Infra 容器）的 Network Namespace 文件的路径，我在前面讲解容器基础的时候提到过，即：/proc/< 容器进程的 PID>/ns/net。

除此之外，在 CNI 环境变量里，还有一个叫作 CNI_ARGS 的参数。通过这个参数，CRI 实现（比如 dockershim）就可以以 Key-Value 的格式，传递自定义信息给网络插件。这是用户将来自定义 CNI 协议的一个重要方法。

**第二部分，则是 dockershim 从 `CNI 配置文件`里加载到的、默认插件的配置信息。**

这个配置信息在 CNI 中被叫作 Network Configuration，它的完整定义你可以参考这个文档。dockershim 会把 Network Configuration 以 JSON 数据的格式，通过标准输入（stdin）的方式传递给 Flannel CNI 插件。而有了这两部分参数，Flannel CNI 插件实现 ADD 操作的过程就非常简单了。

不过，需要注意的是，Flannel 的 CNI 配置文件（ /etc/cni/net.d/10-flannel.conflist）里有这么一个字段，叫作 **delegate**：
```
...
     "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
```
Delegate 字段的意思是，这个 CNI 插件并不会自己做事儿，而是会调用 Delegate 指定的某种 CNI 内置插件来完成。对于 Flannel 来说，它调用的 Delegate 插件，就是前面介绍到的 CNI bridge 插件。

所以说，dockershim 对 Flannel CNI 插件的调用，其实就是走了个过场。**Flannel CNI 插件唯一需要做的，就是对 dockershim 传来的 Network Configuration 进行补充**。比如，将 Delegate 的 Type 字段设置为 bridge，将 Delegate 的 IPAM 字段设置为 host-local 等。
经过 Flannel CNI 插件补充后的、完整的 Delegate 字段如下所示：

```
{
    "hairpinMode":true,
    "ipMasq":false,
    "ipam":{
        "routes":[
            {
                "dst":"10.244.0.0/16"
            }
        ],
        "subnet":"10.244.1.0/24",
        "type":"host-local"
    },
    "isDefaultGateway":true,
    "isGateway":true,
    "mtu":1410,
    "name":"cbr0",
    "type":"bridge"
}
```
其中，ipam 字段里的信息，比如 10.244.1.0/24，读取自 Flannel 在宿主机上生成的 Flannel 配置文件，即：宿主机上的 /run/flannel/subnet.env 文件。接下来，Flannel CNI 插件就会调用 CNI bridge 插件，也就是执行：/opt/cni/bin/bridge 二进制文件。这一次，调用 CNI bridge 插件需要的两部分参数的第一部分、也就是 CNI 环境变量，并没有变化。所以，它里面的 CNI_COMMAND 参数的值还是“ADD”。而第二部分 Network Configration，正是上面补充好的 Delegate 字段。Flannel CNI 插件会把 Delegate 字段的内容以标准输入（stdin）的方式传递给 CNI bridge 插件。

有了这两部分参数，接下来 `CNI bridge` 插件就可以“代表”Flannel，进行“将容器加入到 CNI 网络里”这一步操作了。而这一部分内容，与容器 Network Namespace 密切相关，所以我要为你详细讲解一下。

首先，CNI bridge 插件会在宿主机上检查 CNI 网桥是否存在。如果没有的话，那就创建它。

flannel插件将delegate给cni bridge插件来操作容器加入网络
```shell
#在容器里
#创建一对Veth Pair设备。其中一个叫作eth0，另一个叫作vethb4963f3
$ ip link add eth0 type veth peer name vethb4963f3

# 启动eth0设备
$ ip link set eth0 up 

#将Veth Pair设备的另一端（也就是vethb4963f3设备）放到宿主机（也就是Host Namespace）里
$ ip link set vethb4963f3 netns $HOST_NS

#通过Host Namespace，启动宿主机上的vethb4963f3设备
$ ip netns exec $HOST_NS ip link set vethb4963f3 up 

# 在宿主机上
$ ip link set vethb4963f3 master cni0
```
在将 vethb4963f3 设备连接在 CNI 网桥之后，CNI bridge 插件还会为它设置 **Hairpin Mode**（发夹模式）。这是因为，在默认情况下，网桥设备是不允许一个数据包从一个端口进来后，再从这个端口发出去的。但是，它允许你为这个端口开启 Hairpin Mode，从而取消这个限制。
发夹模式举例来说，容器访问自己的映射的服务，可以通过veth设备发送请求并接受该请求

举个例子，比如我们执行 docker run -p 8080:80，就是在宿主机上通过 iptables 设置了一条DNAT（目的地址转换）转发规则。这条规则的作用是，当宿主机上的进程访问“< 宿主机的 IP 地址 >:8080”时，iptables 会把该请求直接转发到“< 容器的 IP 地址 >:80”上。也就是说，这个请求最终会经过 docker0 网桥进入容器里面。但如果你是在容器里面访问宿主机的 8080 端口，那么这个容器里发出的 IP 包会经过 vethb4963f3 设备（端口）和 docker0 网桥，来到宿主机上。此时，根据上述 DNAT 规则，这个 IP 包又需要回到 docker0 网桥，并且还是通过 vethb4963f3 端口进入到容器里。所以，这种情况下，我们就需要开启 vethb4963f3 端口的 Hairpin Mode 了。所以说，Flannel 插件要在 CNI 配置文件里声明 hairpinMode=true。这样，将来这个集群里的 Pod 才可以通过它自己的 Service 访问到自己。

接下来，CNI bridge 插件会调用 **CNI ipam** 插件，从 ipam.subnet 字段规定的网段里为容器分配一个可用的 IP 地址。然后，CNI bridge 插件就会把这个 IP 地址添加在容器的 eth0 网卡上，同时为容器设置默认路由

最后，CNI bridge 插件会为 CNI 网桥添加 IP 地址。

在执行完上述操作之后，CNI 插件会把容器的 IP 地址等信息返回给 dockershim，然后被 kubelet 添加到 Pod 的 Status 字段。至此，CNI 插件的 `ADD 方法就宣告结束`了。

需要注意的是，对于`非网桥类型`的 CNI 插件，上述“将容器添加到 CNI 网络”的操作流程，以及网络方案本身的工作原理，就都不太一样了。

在本篇文章中，我为你详细讲解了 Kubernetes 中 CNI 网络的实现原理。根据这个原理，你其实就很容易理解所谓的“Kubernetes 网络模型”了：
**所有容器都可以直接使用 IP 地址与其他容器通信，而无需使用 NAT。**
**所有宿主机都可以直接使用 IP 地址与所有容器通信，而无需使用 NAT。反之亦然。**
**容器自己“看到”的自己的 IP 地址，和别人（宿主机或者容器）看到的地址是完全一样的。**

**总结**：
其实本章难点在于实现网络方案对应的CNI插件，即配置Infra容器的网络栈，并连到网桥上。整体流程是：kubelet创建Pod->创建Infra容器->调用SetUpPod（）方法，该方法需要为CNI准备参数，然后调用CNI插件（flannel）为Infra配置网络；其中参数来源于1、dockershim设置的一组CNI环境变量；2、dockershim从CNI配置文件里（有flanneld启动后生成，类型为configmap）加载到的、默认插件的配置信息（network configuration），这里对CNI插件的调用，实际是network configuration进行补充。参数准备好后，调用Flannel CNI->调用CNI bridge（所需参数即为上面：设置的CNI环境变量和补充的network configuation）来执行具体的操作流程。

### 三层方案

在上一篇文章中，我以网桥类型的 Flannel 插件为例，为你讲解了 Kubernetes 里容器网络和 CNI 插件的主要工作原理。不过，除了这种模式之外，还有一种纯三层（Pure Layer 3）网络方案非常值得你注意。

#### flannel host-gw

其中的典型例子，莫过于 Flannel 的 host-gw 模式和 Calico 项目了。我们先来看一下 Flannel 的 host-gw 模式。

![](../../reference/pic/flannel3.webp)

可以看到，host-gw 模式的工作原理，其实就是将每个 Flannel 子网（Flannel Subnet，比如：10.244.1.0/24）的“`下一跳`”，设置成了该子网对应的宿主机的 IP 地址。
也就是说，这台“主机”（Host）会充当这条容器通信路径里的“`网关`”（Gateway）。这也正是“host-gw”的含义。

当然，Flannel 子网和主机的信息，都是保存在 `Etcd` 当中的。flanneld 只需要 WACTH 这些数据的变化，然后实时更新路由表即可。

而在这种模式下，容器通信的过程就免除了额外的封包和解包带来的`性能损耗`。根据实际的测试，host-gw 的性能损失大约在 10% 左右，而其他所有基于 VXLAN“隧道”机制的网络方案，性能损失都在 20%~30% 左右。

一旦配置了下一跳地址，那么接下来，当 IP 包从网络层进入链路层封装成帧的时候，eth0 设备就会使用下一跳地址对应的 MAC 地址，作为该数据帧的`目的 MAC 地址`。显然，这个 MAC 地址，正是 `Node 2` 的 MAC 地址。这样，这个数据帧就会从 Node 1 通过宿主机的二层网络顺利到达 Node 2 上。

>没有封包，故只有一个mac和ip地址

而 Node 2 的内核网络栈从二层数据帧里拿到 IP 包后，会“看到”这个 IP 包的目的 IP 地址是 10.244.1.3，即 `Infra-container-2` 的 IP 地址。这时候，根据 Node 2 上的路由表，该目的地址会匹配到第二条路由规则（也就是 10.244.1.0 对应的路由规则），从而进入 cni0 网桥，进而进入到 Infra-container-2 当中。

当然，通过上面的叙述，你也应该看到，**host-gw 模式能够正常工作的核心，就在于 IP 包在封装成帧发送出去的时候，会使用路由表里的“下一跳”来设置目的 MAC 地址**。这样，它就会经过二层网络到达目的宿主机。

所以说，Flannel host-gw 模式必须要求**集群宿主机之间是二层连通的**。

需要注意的是，宿主机之间二层不连通的情况也是广泛存在的。比如，宿主机分布在了不同的子网（`VLAN`）里，处于不同网段。但是，在一个 Kubernetes 集群里，宿主机之间必须可以通过 IP 地址进行通信，也就是说至少是三层可达的。否则的话，你的集群将不满足上一篇文章中提到的宿主机之间 IP 互通的假设（Kubernetes 网络模型）。当然，“三层可达”也可以通过为几个子网设置`三层转发`来实现。

#### calico网络

![](https://mmbiz.qpic.cn/mmbiz_png/ibD9iaaPDn99iaJZiaVbsmZRUGgEVwxN9vGXnyIciar1B1ZhKqW7FX1dRdcb3VuGfgOHAa8Oa6r2Q1EO3ftdU3aibibwg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

[来源](https://mp.weixin.qq.com/s?__biz=MzA4Nzg5Nzc5OA==&mid=2651726665&idx=1&sn=f4e183df452f3545b7cf5158284fc9f8&chksm=8bc8e0e0bcbf69f6736fe5dd1313236a7554b7526bca3658bc096787ea688b31d76d85ec0adc&mpshare=1&scene=1&srcid=1209WhfipAtf2NRuMZrjaDoR&sharer_sharetime=1670558969690&sharer_shareid=1817186607f682c2c8dc4a6ca1d644dc&version=4.1.0.6007&platform=win#rd)

#### calico

而在容器生态中，要说到像 Flannel host-gw 这样的三层网络方案，我们就不得不提到这个领域里的“龙头老大”Calico 项目了。
实际上，Calico 项目提供的网络解决方案，与 Flannel 的 host-gw 模式，`几乎是完全一样的`。也就是说，Calico 也会在每台宿主机上，添加一个格式如下所示的路由规则：`<目的容器IP地址段> via <网关的IP地址> dev eth0`

不过，**不同于 Flannel 通过 Etcd 和宿主机上的 flanneld 来维护路由信息的做法，Calico 项目使用了一个“重型武器”来自动地在整个集群中分发路由信息。这个“重型武器”，就是 BGP**。BGP 的全称是 Border Gateway Protocol，即：边界网关协议。它是一个 Linux 内核原生就支持的、专门用在大规模数据中心里维护不同的“自治系统”之间路由信息的、无中心的路由协议。

![](../../reference/pic/bgp.webp)

在使用了 BGP 之后，你可以认为，在每个`边界网关`上都会运行着一个小程序，它们会将各自的路由表信息，通过 TCP 传输给其他的边界网关。而其他边界网关上的这个小程序，则会对收到的这些数据进行分析，然后将需要的信息添加到自己的路由表里。这样，图 2 中 Router 2 的路由表里，就会自动出现 10.10.0.2 和 10.10.0.3 对应的路由规则了。

所以说，所谓 BGP，就是在大规模网络中实现节点路由信息共享的一种协议。而 BGP 的这个能力，正好可以取代 Flannel 维护主机上路由表的功能。而且，BGP 这种原生就是为大规模网络环境而实现的协议，其可靠性和可扩展性，远非 Flannel 自己的方案可比。

>需要注意的是，BGP 协议实际上是最复杂的一种路由协议。我在这里的讲述和所举的例子，仅是为了能够帮助你建立对 BGP 的感性认识，并不代表 BGP 真正的实现方式。

在了解了 BGP 之后，**Calico 项目的架构就非常容易理解了。它由三个部分组成**：
Calico 的 `CNI` 插件。这是 Calico 与 Kubernetes 对接的部分。我已经在上一篇文章中，和你详细分享了 CNI 插件的工作原理，这里就不再赘述了。
`Felix`。它是一个 DaemonSet，负责在宿主机上插入路由规则（即：写入 Linux 内核的 `FIB 转发信息库`），以及维护 Calico 所需的网络设备等工作。
`BIRD`。它就是 BGP 的客户端，专门负责在集群里`分发路由规则信息`。

除了对路由信息的维护方式之外，Calico 项目与 Flannel 的 host-gw 模式的另一个不同之处，就是**它不会在宿主机上创建任何网桥设备**(flannel会有cni网桥)。

![](../../reference/pic/calico.webp)

可以看到，Calico 的 CNI 插件会为每个容器设置一个 `Veth Pair` 设备，然后把其中的一端放置在宿主机上（它的名字以 `cali` 前缀开头）。
此外，**由于 Calico 没有使用 CNI 的网桥模式，Calico 的 CNI 插件还需要在宿主机上为每个容器的 Veth Pair 设备配置一条路由规则，用于接收传入的 IP 包。**

比如，宿主机 Node 2 上的 Container 4 对应的路由规则，如下所示：`10.233.2.3 dev cali5863f3 scope link`

基于上述原因，Calico 项目在宿主机上设置的`路由规则`，肯定要比 Flannel 项目多得多。不过，Flannel host-gw 模式使用 CNI 网桥的主要原因，其实是为了跟 VXLAN 模式保持一致。否则的话，Flannel 就需要维护两套 CNI 插件了。

有了这样的 Veth Pair 设备之后，容器发出的 IP 包就会经过 Veth Pair 设备出现在宿主机上。然后，宿主机网络栈就会根据路由规则的下一跳 IP 地址，把它们转发给正确的网关。接下来的流程就跟 Flannel host-gw 模式完全一致了。

其中，**这里最核心的“`下一跳`”路由规则，就是由 Calico 的 Felix 进程负责维护的。这些`路由规则信息`，则是通过 BGP Client 也就是 BIRD 组件，使用 BGP 协议传输而来的。**

而这些通过 BGP 协议传输的消息，你可以简单地理解为如下格式：
```
[BGP消息]
我是宿主机192.168.1.3
10.233.2.0/24网段的容器都在我这里
这些容器的下一跳地址是我
```

不难发现，**Calico 项目实际上将集群里的所有节点，都当作是边界路由器来处理**，它们一起组成了一个全连通的网络，互相之间通过 BGP 协议交换路由规则。这些节点，我们称为 BGP Peer。

##### node to node mesh

需要注意的是，Calico 维护的网络在默认配置下，是一个被称为“`Node-to-Node Mesh`”的模式。这时候，每台宿主机上的 BGP Client 都需要跟其他所有节点的 BGP Client 进行通信以便交换路由信息。但是，随着节点数量 N 的增加，这些连接的数量就会以 N²的规模快速增长，从而给集群本身的网络带来巨大的压力。

所以，Node-to-Node Mesh 模式一般推荐用在少于 100 个节点的集群里。而在更大规模的集群中，你需要用到的是一个叫作 Route Reflector 的模式。

##### route reflector

在这种模式下，Calico 会指定一个或者几个专门的节点，来负责跟所有节点建立 BGP 连接从而学习到全局的路由规则。而其他节点，只需要跟这几个专门的节点交换路由信息，就可以获得整个集群的路由规则信息了。

这些专门的节点，就是所谓的 `Route Reflector` 节点，它们实际上扮演了“中间代理”的角色，从而把 BGP 连接的规模控制在 N 的数量级上。

##### 三层模式限制

此外，我在前面提到过，Flannel host-gw 模式最主要的限制，就是**要求集群宿主机之间是二层连通的**。而这个限制对于 Calico 来说，也同样存在。

举个例子，假如我们有两台处于不同子网的宿主机 Node 1 和 Node 2，对应的 IP 地址分别是 192.168.1.2 和 192.168.2.2。
需要注意的是，这两台机器通过路由器实现了三层转发，所以这两个 IP 地址之间是可以相互通信的。而我们现在的需求，还是 Container 1 要访问 Container 4。按照我们前面的讲述，Calico 会尝试在 Node 1 上添加如下所示的一条路由规则：
`10.233.2.0/16 via 192.168.2.2 eth0`
但是，这时候问题就来了。上面这条规则里的下一跳地址是 192.168.2.2，可是它对应的 Node 2 跟 Node 1 却根本不在一个子网里，没办法通过二层网络把 IP 包发送到下一跳地址。

>没法直接通过MAC寻址；不在一个子网；需要经过一个路由器-在这指节点当作路由器。 但是经过路由器的话，与iptables表中的“下一跳”冲突。“下一跳”的意义是通过目的物理机直接找到目的虚拟机，如果中间经过路由，出了路由之后就不知道去找哪个物理机了，因为**目的ip是目的虚拟机的ip，目的mac应当是下一跳IP对应的mac，但是不在同一子网所以无法找到mac地址**。


在这种情况下，你就需要为 Calico 打开 IPIP 模式。

##### ipip

![](../../reference/pic/calico_ipip.webp)

在 Calico 的 IPIP 模式下，Felix 进程在 Node 1 上添加的路由规则，会稍微不同，如下所示：
`10.233.2.0/24 via 192.168.2.2 tunl0`
可以看到，尽管这条规则的下一跳地址仍然是 Node 2 的 IP 地址，但这一次，**要负责将 IP 包发出去的设备，变成了 tunl0**。注意，是 T-U-N-L-0，而不是 Flannel UDP 模式使用的 T-U-N-0（tun0），这两种设备的功能是完全不一样的。

Calico 使用的这个 tunl0 设备，是一个 `IP 隧道`（IP tunnel）设备。

在上面的例子中，IP 包进入 IP 隧道设备之后，就会被 Linux 内核的 IPIP 驱动接管。**IPIP 驱动会将这个 IP 包直接封装在一个宿主机网络的 IP 包中**

其中，经过封装后的新的 IP 包的目的地址（图 5 中的 Outer IP Header 部分），正是原 IP 包的下一跳地址，即 Node 2 的 IP 地址：192.168.2.2。
而原 IP 包本身，则会被直接封装成新 IP 包的 Payload。
这样，原先从`容器到 Node 2` 的 IP 包，就被伪装成了一个从 `Node 1 到 Node 2` 的 IP 包。

不难看到，当 Calico 使用 IPIP 模式的时候，集群的`网络性能`会因为额外的封包和解包工作而下降。在实际测试中，Calico IPIP 模式与 Flannel VXLAN 模式的性能大致相当。

##### 设置网关BGP

不过，通过上面对 Calico 工作原理的讲述，你应该能发现这样一个事实：如果 Calico 项目能够让宿主机之间的`路由设备`（也就是网关），也通过 BGP 协议“学习”到 Calico 网络里的路由规则，那么从容器发出的 IP 包，不就可以通过这些设备路由到目的宿主机了么？

比如，只要在上面“IPIP 示意图”中的 Node 1 上，添加如下所示的一条路由规则：10.233.2.0/24 via 192.168.1.1 eth0
然后，在 Router 1 上（192.168.1.1），添加如下所示的一条路由规则：10.233.2.0/24 via 192.168.2.1 eth0
那么 Container 1 发出的 IP 包，就可以通过两次“下一跳”，到达 Router 2（192.168.2.1）了。以此类推，我们可以继续在 Router 2 上添加“下一条”路由，最终把 IP 包转发到 Node 2 上。
遗憾的是，上述流程虽然简单明了，但是在 Kubernetes 被广泛使用的公有云场景里，却完全不可行。
这里的原因在于：`公有云环境`下，宿主机之间的网关，肯定不会允许用户进行干预和设置。

不过，在`私有部署`的环境下，**宿主机属于不同子网（VLAN）**反而是更加常见的部署状态。这时候，想办法将宿主机网关也加入到 BGP Mesh 里从而避免使用 IPIP，就成了一个非常迫切的需求。

而在 Calico 项目中，它已经为你提供了两种将宿主机网关设置成 BGP Peer 的`解决方案`。

第一种方案，就是所有宿主机都跟宿主机网关建立 BGP Peer 关系。

这种方案下，Node 1 和 Node 2 就需要主动跟宿主机网关 Router 1 和 Router 2 建立 BGP 连接。从而将类似于 10.233.2.0/24 这样的路由信息同步到网关上去。需要注意的是，这种方式下，Calico **要求宿主机网关必须支持一种叫作 Dynamic Neighbors 的 BGP 配置方式**。这是因为，在常规的路由器 BGP 配置里，运维人员必须明确给出所有 BGP Peer 的 IP 地址。考虑到 Kubernetes 集群可能会有成百上千个宿主机，而且还会动态地添加和删除节点，这时候再手动管理路由器的 BGP 配置就非常麻烦了。而 Dynamic Neighbors 则允许你给路由器配置一个网段，然后路由器就会自动跟该网段里的主机建立起 BGP Peer 关系。

不过，相比之下，我更愿意推荐第二种方案。这种方案，是使用一个或多个独立组件负责搜集整个集群里的所有路由信息，然后通过 BGP 协议同步给网关。

而我们前面提到，在大规模集群中，Calico 本身就推荐使用 `Route Reflector` 节点的方式进行组网。所以，这里负责跟宿主机网关进行沟通的独立组件，直接由 Route Reflector 兼任即可。更重要的是，这种情况下网关的 BGP Peer 个数是有限并且固定的。
所以我们就可以直接把这些独立组件配置成路由器的 BGP Peer，而无需 Dynamic Neighbors 的支持。当然，这些独立组件的工作原理也很简单：它们只需要 WATCH Etcd 里的宿主机和对应网段的变化信息，然后把这些信息通过 BGP 协议分发给网关即可。

需要注意的是，在大规模集群里，三层网络方案在宿主机上的`路由规则`可能会非常多，这会导致错误排查变得困难。此外，在系统故障的时候，路由规则出现重叠冲突的概率也会变大。

基于上述原因，如果是在公有云上，由于宿主机网络本身比较“直白”，我一般会推荐更加简单的 Flannel host-gw 模式。但不难看到，在私有部署环境里，Calico 项目才能够覆盖更多的场景，并为你提供更加可靠的组网方案和架构思路。

##### 三层和隧道的异同

相同之处是都实现了跨主机容器的三层互通，而且都是通过对目的 MAC 地址的操作来实现的；
不同之处是三层通过配置下一条主机的路由规则来实现互通，隧道则是通过通过在 IP 包外再封装一层 MAC 包头来实现。
三层的优点：少了封包和解包的过程，性能肯定是更高的。
三层的缺点：需要自己想办法维护路由规则。
隧道的优点：简单，原因是大部分工作都是由 Linux 内核的模块实现了，应用层面工作量较少。
隧道的缺点：主要的问题就是性能低。

##### networkpolicy

在 Kubernetes 里，`网络隔离能力`的定义，是依靠一种专门的 API 对象来描述的，即：NetworkPolicy。

```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - ipBlock:
        cidr: 172.17.0.0/16
        except:
        - 172.17.1.0/24
    - namespaceSelector:
        matchLabels:
          project: myproject
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 6379
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/24
    ports:
    - protocol: TCP
      port: 5978
```
综上所述，这个 NetworkPolicy 对象，指定的隔离规则如下所示：

该隔离规则只对 default Namespace 下的，携带了 role=db 标签的 Pod 有效。限制的请求类型包括 ingress（流入）和 egress（流出）。Kubernetes 会拒绝任何访问被隔离 Pod 的请求，除非这个请求来自于以下“白名单”里的对象，并且访问的是被隔离 Pod 的 6379 端口。

这些“白名单”对象包括：
a. default Namespace 里的，携带了 role=fronted 标签的 Pod；
b. 携带了 project=myproject 标签的 Namespace 里的任何 Pod；
c. 任何源地址属于 172.17.0.0/16 网段，且不属于 172.17.1.0/24 网段的请求。

Kubernetes 会拒绝被隔离 Pod 对外发起任何请求，除非请求的目的地址属于 10.0.0.0/24 网段，并且访问的是该网段地址的 5978 端口。

**Kubernetes 里的 Pod 默认都是“允许所有”（Accept All）的**，即：Pod 可以接收来自任何发送方的请求；或者，向任何接收方发送请求。而如果你要对这个情况作出限制，就必须通过 NetworkPolicy 对象来指定。

而在上面这个例子里，你首先会看到 podSelector 字段。它的作用，就是定义这个 NetworkPolicy 的限制范围，比如：当前 Namespace 里携带了 role=db 标签的 Pod。
而如果你把 podSelector 字段留空：
spec: podSelector: {}
那么这个 NetworkPolicy 就会作用于当前 Namespace 下的所有 Pod。

而一旦 Pod 被 NetworkPolicy 选中，那么这个 Pod **就会进入“拒绝所有”（Deny All）的状态**，即：这个 Pod 既不允许被外界访问，也不允许对外界发起访问。

而 NetworkPolicy 定义的规则，其实就是 **“白名单”**。

需要注意的是，定义一个 NetworkPolicy 对象的过程，容易犯错的是“白名单”部分（from 和 to 字段）。
**选择器两者都是-的时候是或的关系，而两者只有开始有-后者无-的时候是和的关系**

```  ...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
    - podSelector:
        matchLabels:
          role: client
  ...
...
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          user: alice
      podSelector:
        matchLabels:
          role: client
  ...
```

此外，如果要使上面定义的 NetworkPolicy 在 Kubernetes 集群里真正产生作用，你的 CNI 网络插件就必须是支持 Kubernetes 的 NetworkPolicy 的。
**在具体实现上，凡是支持 NetworkPolicy 的 `CNI 网络插件`，都维护着一个 `NetworkPolicy Controller`，通过控制循环的方式对 NetworkPolicy 对象的增删改查做出响应，然后在宿主机上完成 iptables 规则的配置工作。**

在 Kubernetes 生态里，目前已经实现了 NetworkPolicy 的网络插件包括 Calico、Weave 和 kube-router 等多个项目，但是并不包括 `Flannel` 项目。
所以说，如果想要在使用 Flannel 的同时还使用 NetworkPolicy 的话，你就需要再额外安装一个网络插件，比如 Calico 项目，来负责执行 NetworkPolicy。
安装 Flannel + Calico 的流程非常简单，你直接参考这个文档[一键安装](https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/flannel)即可。

**Kubernetes 网络插件对 Pod 进行隔离，其实是靠在宿主机上生成 NetworkPolicy 对应的 iptable 规则来实现的**。

此外，在设置好上述“隔离”规则之后，网络插件还需要想办法，将所有对被隔离 Pod 的访问请求，都转发到上述 KUBE-NWPLCY-CHAIN 规则上去进行匹配。并且，如果匹配不通过，这个请求应该被“拒绝”。

>kube-router 其实是一个简化版的 Calico，它也使用 BGP 来维护路由信息，但是使用 CNI bridge 插件负责跟 Kubernetes 进行交互。

需要注意的是，在有网桥参与的情况下，上述 Netfilter 设置“检查点”的流程，实际上也会出现在链路层（二层），并且会跟我在上面讲述的网络层（三层）的流程有交互。这些链路层的“检查点”对应的操作界面叫作 **ebtables**。所以，准确地说，数据包在 Linux Netfilter 子系统里完整的流动过程，其实应该如下所示：
![](../../reference/pic/netfilter.webp)

在本篇文章中，我主要和你分享了 Kubernetes 对 Pod 进行“隔离”的手段，即：NetworkPolicy。

可以看到，NetworkPolicy 实际上只是宿主机上的一系列 **iptables 规则**。
这跟传统 IaaS 里面的安全组（Security Group）其实是非常类似的。

而基于上述讲述，你就会发现这样一个事实：**Kubernetes 的网络模型以及大多数容器网络实现，其实既不会保证容器之间二层网络的互通，也不会实现容器之间的二层网络隔离**。

这跟 IaaS 项目管理虚拟机的方式，是完全不同的。

所以说，Kubernetes 从底层的设计和实现上，更倾向于假设你已经有了一套完整的物理基础设施。然后，Kubernetes 负责在此基础上提供一种“弱多租户”（soft multi-tenancy）的能力。

并且，基于上述思路，Kubernetes 将来也不大可能把 Namespace 变成一个具有实质意义的隔离机制，或者把它映射成为“子网”或者“租户”。毕竟你可以看到，NetworkPolicy 对象的描述能力，要比基于 Namespace 的划分丰富得多。

这也是为什么，到目前为止，Kubernetes 项目在云计算生态里的定位，其实是基础设施与 PaaS 之间的中间层。这是非常符合“容器”这个本质上就是进程的抽象粒度的。


## service

**实际上，Service 是由 `kube-proxy` 组件，加上 `iptables` 来共同实现的**。

举个例子，对于我们前面创建的名叫 hostnames 的 Service 来说，一旦它被提交给 Kubernetes，那么 **kube-proxy 就可以通过 Service 的 `Informer` 感知到这样一个 Service 对象的添加。而作为对这个事件的响应，它就会在宿主机上创建这样一条 iptables 规则**

而我们前面已经看到，10.0.1.175 正是这个 Service 的 VIP。所以这一条规则，就为这个 Service 设置了一个固定的入口地址。并且，由于 10.0.1.175 只是一条 iptables 规则上的配置，**并没有真正的网络设备**，所以你 ping 这个地址，是不会有任何响应的。

**service的iptables实现**：
```
#iptables-save

# 访问service地址的包控制
-A KUBE-SERVICES -d 10.0.1.175/32 -p tcp -m comment --comment "default/hostnames: cluster IP" -m tcp --dport 80 -j KUBE-SVC-NWV5X2332I4OT4T3

# 可以看到，这条 iptables 规则的含义是：凡是目的地址是 10.0.1.175、目的端口是 80 的 IP 包，都应该跳转到另外一条名叫 KUBE-SVC-NWV5X2332I4OT4T3 的 iptables 链进行处理。

# 负载均衡
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-WNBA2IHDGP2BOBGZ
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-X3P2623AGDH6CDF3
-A KUBE-SVC-NWV5X2332I4OT4T3 -m comment --comment "default/hostnames:" -j KUBE-SEP-57KPRZ3JQVENLNBR

# 而这三条链指向的最终目的地，其实就是这个 Service 代理的三个 Pod。所以这一组规则，就是 Service 实现负载均衡的位置。需要注意的是，iptables 规则的匹配是从上到下逐条进行的，所以为了保证上述三条规则每条被选中的概率都相同，我们应该将它们的 probability 字段的值分别设置为 1/3（0.333…）、1/2 和 1。

# 三条链的DNAT
-A KUBE-SEP-57KPRZ3JQVENLNBR -s 10.244.3.6/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-57KPRZ3JQVENLNBR -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.3.6:9376

-A KUBE-SEP-WNBA2IHDGP2BOBGZ -s 10.244.1.7/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-WNBA2IHDGP2BOBGZ -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.1.7:9376

-A KUBE-SEP-X3P2623AGDH6CDF3 -s 10.244.2.3/32 -m comment --comment "default/hostnames:" -j MARK --set-xmark 0x00004000/0x00004000
-A KUBE-SEP-X3P2623AGDH6CDF3 -p tcp -m comment --comment "default/hostnames:" -m tcp -j DNAT --to-destination 10.244.2.3:9376
```
`DNAT` 规则的作用，就是在 PREROUTING 检查点之前，也就是在路由之前，将流入 IP 包的目的地址和端口，改成–to-destination 所指定的新的目的地址和端口。可以看到，这个目的地址和端口，正是被代理 Pod 的 IP 地址和端口。

这样，访问 Service VIP 的 IP 包经过上述 iptables 处理之后，就已经变成了访问具体某一个后端 Pod 的 IP 包了。不难理解，**这些 Endpoints 对应的 iptables 规则，正是 kube-proxy 通过监听 Pod 的变化事件，在宿主机上生成并维护的**。

此外，你可能已经听说过，Kubernetes 的 kube-proxy 还支持一种叫作 **IPVS** 的模式。这又是怎么一回事儿呢？

其实，通过上面的讲解，你可以看到，kube-proxy 通过 iptables 处理 Service 的过程，其实需要在宿主机上设置相当多的 iptables 规则。
而且，**kube-proxy 还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的**。不难想到，当你的宿主机上有大量 Pod 的时候，成百上千条 iptables 规则不断地被刷新，会大量占用该宿主机的 CPU 资源，甚至会让宿主机“卡”在这个过程中。

所以说，一直以来，基于 iptables 的 Service 实现，都是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。

IPVS 模式的工作原理，其实跟 iptables 模式类似。**当我们创建了前面的 Service 之后，kube-proxy 首先会在宿主机上创建一个`虚拟网卡`（叫作：kube-ipvs0），并为它分配 Service VIP 作为 IP 地址**，如下所示：
```
# ip addr
  ...
  73：kube-ipvs0：<BROADCAST,NOARP>  mtu 1500 qdisc noop state DOWN qlen 1000
  link/ether  1a:ce:f5:5f:c1:4d brd ff:ff:ff:ff:ff:ff
  inet 10.0.1.175/32  scope global kube-ipvs0
  valid_lft forever  preferred_lft forever

```

而接下来，**kube-proxy 就会通过 Linux 的 IPVS 模块，为这个 IP 地址设置三个 IPVS 虚拟主机，并设置这三个虚拟主机之间使用轮询模式 (rr) 来作为负载均衡策略**。
我们可以通过 ipvsadm 查看到这个设置，如下所示：
```
# ipvsadm -ln
 IP Virtual Server version 1.2.1 (size=4096)
  Prot LocalAddress:Port Scheduler Flags
    ->  RemoteAddress:Port           Forward  Weight ActiveConn InActConn     
  TCP  10.102.128.4:80 rr
    ->  10.244.3.6:9376    Masq    1       0          0         
    ->  10.244.1.7:9376    Masq    1       0          0
    ->  10.244.2.3:9376    Masq    1       0          0
```
可以看到，这三个 IPVS 虚拟主机的 IP 地址和端口，对应的正是三个被代理的 Pod。
这时候，任何发往 10.102.128.4:80 的请求，就都会被 IPVS 模块转发到某一个后端 Pod 上了。

而相比于 iptables，**IPVS 在内核中的实现其实也是基于 Netfilter 的 NAT 模式**，所以在转发这一层上，理论上 IPVS 并没有显著的性能提升。但是，IPVS 并不需要在宿主机上为每个 Pod 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价。
这也正印证了我在前面提到过的，“`将重要操作放入内核态`”是提高性能的重要手段。

>IPVS在Kubernetes1.11中升级为GA稳定版。IPVS则专门用于高性能负载均衡，并使用更高效的数据结构（Hash表），允许几乎无限的规模扩张，因此被kube-proxy采纳为最新模式。 在IPVS模式下，使用iptables的扩展**ipset**，而不是直接调用iptables来生成规则链。iptables规则链是一个线性的数据结构，ipset则引入了带索引的数据结构，因此当规则很多时，也可以很高效地查找和匹配。 可以将ipset简单理解为一个IP（段）的集合，这个集合的内容可以是IP地址、IP网段、端口等，iptables可以直接添加规则对这个“可变的集合”进行操作，这样做的好处在于可以大大减少iptables规则的数量，从而减少性能损耗。 https://kubernetes.io/zh/docs/concepts/services-networking/service/ 
IPVS代理模式基于类似于 iptables 模式的 netfilter 挂钩函数， 但是使用哈希表作为基础数据结构，并且在内核空间中工作。 这意味着，与 iptables 模式下的 kube-proxy 相比，IPVS 模式下的 kube-proxy 重定向通信的延迟要短，并且在同步代理规则时具有更好的性能。 与其他代理模式相比，IPVS 模式还支持更高的网络流量吞吐量。

**不过需要注意的是，IPVS 模块只负责上述的负载均衡和代理功能。而一个完整的 Service 流程正常工作所需要的包过滤、SNAT 等操作，还是要靠 iptables 来实现。**
只不过，这些辅助性的 iptables 规则数量有限，也不会随着 Pod 数量的增加而增加。
所以，在大规模集群里，我非常建议你为 kube-proxy 设置`–proxy-mode=ipvs` 来开启这个功能。它为 Kubernetes 集群规模带来的提升，还是非常巨大的。

在 Kubernetes 中，**Service 和 Pod 都会被分配对应的 DNS A 记录**（从域名解析 IP 的记录）。

对于 ClusterIP 模式的 Service 来说（比如我们上面的例子），它的 A 记录的格式是：..svc.cluster.local。当你访问这条 A 记录的时候，它解析到的就是该 Service 的 VIP 地址。

而对于指定了 clusterIP=None 的 Headless Service 来说，它的 A 记录的格式也是：..svc.cluster.local。但是，当你访问这条 A 记录的时候，它返回的是所有被代理的 Pod 的 IP 地址的集合。当然，如果你的客户端没办法解析这个集合的话，它可能会只会拿到第一个 Pod 的 IP 地址。

此外，对于 ClusterIP 模式的 Service 来说，它代理的 Pod 被自动分配的 A 记录的格式是：..pod.cluster.local。这条记录指向 Pod 的 IP 地址。**注意格式为podIP地址(横杠连接).namespace名字.pod.cluster.local**

而对 Headless Service 来说，它代理的 Pod 被自动分配的 A 记录的格式是：...svc.cluster.local。这条记录也指向 Pod 的 IP 地址。

**但如果你为 Pod 指定了 Headless Service，并且 Pod 本身声明了 hostname 和 subdomain 字段，那么这时候 Pod 的 A 记录就会变成：(pod 的 hostname)...svc.cluster.local**

需要注意的是，**在 Kubernetes 里，/etc/hosts 文件是单独挂载的**，这也是为什么 kubelet 能够对 hostname 进行修改并且 Pod 重建后依然有效的原因。这跟 Docker 的 Init 层是一个原理。

ClusterIP 模式的 Service 为你提供的，就是一个 Pod 的**稳定的 IP 地址**，即 VIP。并且，这里 Pod 和 Service 的关系是可以通过 Label 确定的。

而 Headless Service 为你提供的，则是一个 Pod 的**稳定的 DNS 名字**，并且，这个名字是可以通过 Pod 名字和 Service 名字拼接出来的。

### NodePort

实现原理与service类似，都是iptables

-A KUBE-NODEPORTS -p tcp -m comment --comment "default/my-nginx: nodePort" -m tcp --dport 8080 -j KUBE-SVC-67RL4FN6JRUPOJYM

需要注意的是，**在 NodePort 方式下，Kubernetes 会在 IP 包离开宿主机发往目的 Pod 时，对这个 IP 包做一次 SNAT 操作**，如下所示：

-A KUBE-POSTROUTING -m comment --comment "kubernetes service traffic requiring SNAT" -m mark --mark 0x4000/0x4000 -j MASQUERADE

可以看到，这条规则设置在 POSTROUTING 检查点，也就是说，它给即将离开这台主机的 IP 包，进行了一次 SNAT 操作，将这个 IP 包的源地址替换成了这台宿主机上的 CNI 网桥地址，或者宿主机本身的 IP 地址（如果 CNI 网桥不存在的话）。
```
           client
             \ ^
              \ \
               v \
   node 1 <--- node 2
    | ^   SNAT
    | |   --->
    v |
 endpoint
```
当然，这个 SNAT 操作只需要对 Service 转发出来的 IP 包进行（否则普通的 IP 包就被影响了）。而 **iptables 做这个判断的依据，就是查看该 IP 包是否有一个“0x4000”的“`标志`”**。你应该还记得，这个标志正是在 IP 包被执行 DNAT 操作之前被打上去的。

当一个外部的 client 通过 node 2 的地址访问一个 Service 的时候，node 2 上的负载均衡规则，就可能把这个 IP 包转发给一个在 node 1 上的 Pod。这里没有任何问题。

而当 node 1 上的这个 Pod 处理完请求之后，它就会按照这个 IP 包的源地址发出回复。可是，如果没有做 SNAT 操作的话，这时候，被转发来的 IP 包的源地址就是 client 的 IP 地址。所以此时，Pod 就会直接将回复发给client。对于 client 来说，它的请求明明发给了 node 2，收到的回复却来自 node 1，这个 client 很可能会报错。

所以，在上图中，当 IP 包离开 node 2 之后，它的源 IP 地址就会被 SNAT 改成 node 2 的 CNI 网桥地址或者 node 2 自己的地址。这样，Pod 在处理完成之后就会先回复给 node 2（而不是 client），然后再由 node 2 发送给 client。

当然，这也就意味着这个 Pod 只知道该 IP 包来自于 node 2，而不是外部的 client。对于 `Pod 需要明确知道所有请求来源的场景`来说，这是不可以的。

所以这时候，你**就可以将 Service 的 `spec.externalTrafficPolicy` 字段设置为 local，这就保证了所有 Pod 通过 Service 收到请求之后，一定可以看到真正的、外部 client 的源地址**。

而这个机制的实现原理也非常简单：**这时候，一台宿主机上的 iptables 规则，会设置为只将 IP 包转发给运行在这台宿主机上的 Pod。所以这时候，Pod 就可以直接使用源地址将回复包发出，不需要事先进行 SNAT 了**。这个流程，如下所示：
```
       client
       ^ /   \
      / /     \
     / v       X
   node 1     node 2
    ^ |
    | |
    | v
 endpoint
```

当然，这也就意味着如果在一台宿主机上，没有任何一个被代理的 Pod 存在，比如上图中的 node 2，那么你使用 node 2 的 IP 地址访问这个 Service，就是无效的。此时，你的请求会直接被 DROP 掉。

### LoadBalancer

在公有云提供的 Kubernetes 服务里，都使用了一个叫作 CloudProvider 的转接层，来跟公有云本身的 API 进行对接。

所以，在上述 LoadBalancer 类型的 Service 被提交后，Kubernetes 就会调用 CloudProvider 在公有云上为你创建一个负载均衡服务，并且把被代理的 Pod 的 IP 地址配置给负载均衡服务做后端。

### externalName

```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  type: ExternalName
  externalName: my.database.example.com
```
在上述 Service 的 YAML 文件中，我指定了一个 `externalName=my.database.example.com` 的字段。而且你应该会注意到，这个 YAML 文件里不需要指定 selector。这时候，当你通过 Service 的 DNS 名字访问它的时候，比如访问：my-service.default.svc.cluster.local。那么，Kubernetes 为你返回的就是my.database.example.com。

所以说，**ExternalName 类型的 Service，其实是在 `kube-dns` 里为你添加了一条 `CNAME` 记录**。这时，访问 my-service.default.svc.cluster.local 就和访问 my.database.example.com 这个域名是一个效果了。


ExternalName用处在于`迁移外部服务到k8s`:
1. 如 有app01、pythonSvr 两个服务; 预计将两者都迁移到 k8s中;
2. app01 先迁移, 预计pythonSvr 不久之后也迁移:
我们先修改app01 的配置，让其访问pythonSvr 不再通过 pythonSvr.example.com; 而是通过 pythonSvr.default.svc.cluster.local;
3. 我们此时创建一个 ExternalName类型的 Service,将 pythonSvr.default.svc.cluster.local 指向外部的 pythonSvr.example.com;
4. pythonSvr 迁移完成后，我们修改c中创建的Service，修改其配置类型为 ClusterIP;
5. app01 的配置不需要动，也无需重启，因为他一开始访问的域名 pythonSvr.default.svc.cluster.local 就是对的;

### externalIP

此外，Kubernetes 的 Service 还允许你为 Service 分配`公有 IP 地址`，比如下面这个例子：
```
kind: Service
apiVersion: v1
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 9376
  externalIPs:
  - 80.11.12.10
```
在上述 Service 中，我为它指定的 externalIPs=80.11.12.10，那么此时，你就可以通过访问 80.11.12.10:80 访问到被代理的 Pod 了。不过，在这里 Kubernetes 要求 externalIPs 必须是`至少能够路由到一个 Kubernetes 的节点`。

>为什么 Kubernetes 要求 externalIPs 必须是至少能够路由到一个 Kubernetes 的节点？
因为k8s只是在集群中的每个节点上创建了一个 `externalIPs 与kube-ipvs0网卡`的绑定关系.
当externalIP设置为k8s节点时，会使得其他节点访问该节点的流量被转移到本节点的kube-ipvs0网卡上
若流量都无法路由到任意的一个k8s节点,那自然无法将流量转给具体的service.

一个 iptables 模式的 Service 对应的规则，我在上一篇以及这一篇文章里已经全部介绍到了，它们包括：
KUBE-SERVICES 或者 KUBE-NODEPORTS 规则对应的 Service 的入口链，这个规则应该与 VIP 和 Service 端口一一对应；
KUBE-SEP-(hash) 规则对应的 DNAT 链，这些规则应该与 Endpoints 一一对应；
KUBE-SVC-(hash) 规则对应的负载均衡链，这些规则的数目应该与 Endpoints 数目一致；
如果是 NodePort 模式的话，还有 POSTROUTING 处的 SNAT 链。

还有一种典型问题，就是 `Pod 没办法通过 Service 访问到自己`。这往往就是因为 kubelet 的 hairpin-mode 没有被正确设置。关于 Hairpin 的原理我在前面已经介绍过，这里就不再赘述了。你只需要确保将 kubelet 的 hairpin-mode 设置为 `hairpin-veth 或者 promiscuous-bridge` 即可。

在 hairpin-veth 模式下，你应该能看到 CNI 网桥对应的各个 VETH 设备，都将 Hairpin 模式设置为了 1，如下所示：
```
for d in /sys/devices/virtual/net/cni0/brif/veth*/hairpin_mode; do echo "$d = $(cat $d)"; done
/sys/devices/virtual/net/cni0/brif/veth4bfbfe74/hairpin_mode = 1
/sys/devices/virtual/net/cni0/brif/vethfc2a18c5/hairpin_mode = 1
```
而如果是 promiscuous-bridge 模式的话，你应该看到 CNI 网桥的混杂模式（PROMISC）被开启，如下所示：
```
ifconfig cni0 |grep PROMISC
UP BROADCAST RUNNING PROMISC MULTICAST  MTU:1460  Metric:1
```

通过上述讲解不难看出，所谓 Service，其实就是 Kubernetes 为 Pod 分配的、固定的、基于 iptables（或者 IPVS）的访问入口。而这些访问入口代理的 Pod 信息，则来自于 Etcd，由 kube-proxy 通过控制循环来维护。

## ingress 

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cafe-ingress
spec:
  tls:
  - hosts:
    - cafe.example.com
    secretName: cafe-secret
  rules:
  - host: cafe.example.com
    http:
      paths:
      - path: /tea
        backend:
          serviceName: tea-svc
          servicePort: 80
      - path: /coffee
        backend:
          serviceName: coffee-svc
          servicePort: 80
```

所谓 Ingress 对象，其实就是 Kubernetes 项目对“反向代理”的一种抽象。

一个 Ingress 对象的主要内容，实际上就是一个“反向代理”服务（比如：Nginx）的配置文件的描述。而这个代理服务对应的转发规则，就是 IngressRule。

部署一个nginx ingress controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
```

而这个 **Pod 本身，就是一个监听 Ingress 对象以及它所代理的后端 Service 变化的`控制器`**。
**当一个新的 Ingress 对象由用户创建后，nginx-ingress-controller 就会根据 Ingress 对象里定义的内容，生成一份对应的 Nginx 配置文件（/etc/nginx/nginx.conf），并使用这个配置文件启动一个 `Nginx 服务`**。

而一旦 Ingress 对象被更新，nginx-ingress-controller 就会更新这个配置文件。**需要注意的是，如果这里只是被代理的 Service 对象被更新，nginx-ingress-controller 所管理的 Nginx 服务是不需要重新加载（reload）的。这当然是因为 nginx-ingress-controller 通过`Nginx Lua`方案实现了 Nginx Upstream 的动态配置**。

此外，**nginx-ingress-controller 还允许你通过 Kubernetes 的 `ConfigMap` 对象来对上述 Nginx 配置文件进行定制**。这个 ConfigMap 的名字，需要以参数的方式传递给 nginx-ingress-controller。而你在这个 ConfigMap 里添加的字段，将会被合并到最后生成的 Nginx 配置文件当中。

可以看到，一个 Nginx Ingress Controller 为你提供的服务，其实是一个可以根据 Ingress 对象和被代理后端 Service 的变化，来自动进行更新的 Nginx 负载均衡器。

当然，为了让用户能够用到这个 Nginx，我们就需要创建一个 Service 来把 Nginx Ingress Controller 管理的 Nginx 服务暴露出去，如下所示：
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml
```
由于我们使用的是 Bare-metal 环境，所以 service-nodeport.yaml 文件里的内容，就是一个 NodePort 类型的 Service，如下所示：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
```
可以看到，这个 Service 的唯一工作，就是将所有携带 ingress-nginx 标签的 Pod 的 80 和 433 端口暴露出去。

不过，你可能会有一个疑问，如果我的请求没有匹配到任何一条 IngressRule，那么会发生什么呢？

首先，既然 Nginx Ingress Controller 是用 Nginx 实现的，那么它当然会为你返回一个 Nginx 的 404 页面。

不过，Ingress Controller 也允许你通过 Pod 启动命令里的–default-backend-service 参数，设置一条`默认规则`，比如：–default-backend-service=nginx-default-backend。这样，任何匹配失败的请求，就都会被转发到这个名叫 nginx-default-backend 的 Service。所以，你就可以通过部署一个专门的 Pod，来为用户返回自定义的 404 页面了。

目前，Ingress 只能工作在七层，而 Service 只能工作在四层。所以当你想要在 Kubernetes 里为应用进行**TLS 配置等 HTTP 相关的操作时，都必须通过 Ingress 来进行**。

## 资源调度

Kubernetes 里 Pod 的 CPU 和内存资源，实际上还要分为 limits 和 requests 两种情况

这两者的区别其实非常简单：**在调度的时候，kube-scheduler 只会按照 requests 的值进行计算。而在真正设置 Cgroups 限制的时候，kubelet 则会按照 limits 的值来进行设置**。

在 Kubernetes 中，不同的 requests 和 limits 的设置方式，其实会将这个 Pod 划分到不同的 **QoS** 级别当中。

**当 Pod 里的每一个 Container 都同时设置了 requests 和 limits，并且 requests 和 limits 值相等的时候，这个 Pod 就属于 `Guaranteed` 类别**

**需要注意的是，当 Pod 仅设置了 limits 没有设置 requests 的时候，Kubernetes 会自动为它设置与 limits 相同的 requests 值，所以，这也属于 `Guaranteed` 情况。**

**而当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 `Burstable` 类别。**

**而如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 `BestEffort`。**

实际上，QoS 划分的主要应用场景，是当宿主机资源紧张的时候，kubelet 对 Pod 进行 `Eviction`（即资源回收）时需要用到的。

具体地说，当 Kubernetes 所管理的宿主机上不可压缩资源短缺时，就有可能触发 Eviction。比如，可用内存（memory.available）、可用的宿主机磁盘空间（nodefs.available），以及容器运行时镜像存储空间（imagefs.available）等等。

当然，上述各个触发条件在 kubelet 里都是可配置的。

`kubelet --eviction-hard=imagefs.available<10%,memory.available<500Mi,nodefs.available<5%,nodefs.inodesFree<5% --eviction-soft=imagefs.available<30%,nodefs.available<10% --eviction-soft-grace-period=imagefs.available=2m,nodefs.available=2m --eviction-max-pod-grace-period=600`

在这个配置中，你可以看到**Eviction 在 Kubernetes 里其实分为 Soft 和 Hard 两种模式**。

其中，`Soft Eviction` 允许你为 Eviction 过程设置一段“优雅时间”，比如上面例子里的 imagefs.available=2m，就意味着当 imagefs 不足的阈值达到 2 分钟之后，kubelet 才会开始 Eviction 的过程。

而 `Hard Eviction` 模式下，Eviction 过程就会在阈值达到之后立刻开始。

Kubernetes 计算 Eviction 阈值的`数据来源`，主要依赖于从 Cgroups 读取到的值，以及使用 cAdvisor 监控到的数据。

当宿主机的 Eviction 阈值达到后，就会进入 `MemoryPressure` 或者 `DiskPressure` 状态，从而避免新的 Pod 被调度到这台宿主机上。

而当 Eviction 发生的时候，kubelet 具体会**挑选哪些 Pod 进行删除操作**，就需要参考这些 Pod 的 QoS 类别了。

首当其冲的，自然是 BestEffort 类别的 Pod。
其次，是属于 Burstable 类别、并且发生“饥饿”的资源使用量已经超出了 requests 的 Pod。
最后，才是 Guaranteed 类别。并且，Kubernetes 会保证只有当 Guaranteed 类别的 Pod 的资源使用量超过了其 limits 的限制，或者宿主机本身正处于 Memory Pressure 状态时，Guaranteed 的 Pod 才可能被选中进行 Eviction 操作。

在理解了 Kubernetes 里的 QoS 类别的设计之后，我再来为你讲解一下Kubernetes 里一个非常有用的特性：**cpuset** 的设置。

我们知道，在使用容器的时候，你可以通过**设置 cpuset 把容器绑定到某个 CPU 的核上，而不是像 cpushare 那样共享 CPU 的计算能力**。这种情况下，由于操作系统在 CPU 之间进行上下文切换的次数大大减少，容器里应用的性能会得到大幅提升。事实上，cpuset 方式，是生产环境里部署在线应用类型的 Pod 时，非常常用的一种方式。

可是，这样的需求在 Kubernetes 里又该如何实现呢？其实非常简单。**首先，你的 Pod 必须是 Guaranteed 的 QoS 类型；然后，你只需要将 Pod 的 CPU 资源的 requests 和 limits 设置为同一个相等的`整数值`即可**。

在namespace **limitRange** 里面设置了default request 和 default limit之后，创建出来的pod即使不显式指定limit和request，也是guaranteed

## 调度器

默认调度器会首先调用一组叫作 **Predicate** 的调度算法，来检查每个 Node。然后，再调用一组叫作 **Priority** 的调度算法，来给上一步得到的结果里的每个 Node 打分。最终的调度结果，就是得分最高的那个 Node。而我在前面的文章中曾经介绍过，调度器对一个 Pod 调度成功，实际上就是将它的 `spec.nodeName` 字段填上调度结果的节点名字。

其中，第一个控制循环，我们可以称之为 `Informer Path`。

它的主要目的，是启动一系列 Informer，用来监听（Watch）Etcd 中 Pod、Node、Service 等与调度相关的 API 对象的变化。比如，当一个待调度 Pod（即：它的 nodeName 字段是空的）被创建出来之后，调度器就会通过 Pod Informer 的 Handler，将这个待调度 Pod 添加进调度队列。在默认情况下，Kubernetes 的调度队列是一个 PriorityQueue（优先级队列），并且当某些集群信息发生变化的时候，调度器还会对调度队列里的内容进行一些特殊操作。这里的设计，主要是出于调度优先级和抢占的考虑，我会在后面的文章中再详细介绍这部分内容。此外，Kubernetes 的默认调度器还要负责对调度器缓存（即：scheduler cache）进行更新。事实上，Kubernetes 调度部分进行性能优化的一个最根本原则，就是尽最大可能将集群信息 Cache 化，以便从根本上提高 Predicate 和 Priority 调度算法的执行效率。

而第二个控制循环，是调度器负责 Pod 调度的主循环，我们可以称之为 `Scheduling Path`。

Scheduling Path 的主要逻辑，就是不断地从调度队列里出队一个 Pod。然后，调用 Predicates 算法进行“过滤”。这一步“过滤”得到的一组 Node，就是所有可以运行这个 Pod 的宿主机列表。当然，Predicates 算法需要的 Node 信息，都是从 Scheduler Cache 里直接拿到的，这是调度器保证算法执行效率的主要手段之一。接下来，调度器就会再调用 Priorities 算法为上述列表里的 Node 打分，分数从 0 到 10。得分最高的 Node，就会作为这次调度的结果。

调度算法执行完成后，调度器就需要将 Pod 对象的 nodeName 字段的值，修改为上述 Node 的名字。这个步骤在 Kubernetes 里面被称作 `Bind`。

但是，为了不在关键调度路径里远程访问 APIServer，Kubernetes 的默认调度器在 Bind 阶段，只会更新 Scheduler Cache 里的 Pod 和 Node 的信息。这种基于“乐观”假设的 API 对象更新方式，在 Kubernetes 里被称作 `Assume`。

Assume 之后，调度器才会创建一个 Goroutine 来异步地向 APIServer 发起更新 Pod 的请求，来真正完成 Bind 操作。如果这次异步的 Bind 过程失败了，其实也没有太大关系，等 Scheduler Cache 同步之后一切就会恢复正常。

当然，正是由于上述 Kubernetes 调度器的“乐观”绑定的设计，当一个新的 Pod 完成调度需要在某个节点上运行起来之前，该节点上的 kubelet 还会通过一个叫作 `Admit` 的操作来再次验证该 Pod 是否确实能够运行在该节点上。这一步 Admit 操作，实际上就是把一组叫作 GeneralPredicates 的、最基本的调度算法，比如：“资源是否可用”“端口是否冲突”等再执行一遍，作为 kubelet 端的二次确认。

除了上述的“Cache 化”和“乐观绑定”，Kubernetes 默认调度器还有一个重要的设计，那就是“`无锁化`”。在 Scheduling Path 上，调度器会启动多个 Goroutine 以节点为粒度并发执行 Predicates 算法，从而提高这一阶段的执行效率。而与之类似的，Priorities 算法也会以 MapReduce 的方式并行计算然后再进行汇总。而在这些所有需要并发的路径上，调度器会避免设置任何全局的竞争资源，从而免去了使用锁进行同步带来的巨大的性能损耗。


**Predicates** 在调度过程中的作用，可以理解为 Filter，即：它按照调度策略，从当前集群的所有节点中，“过滤”出一系列符合条件的节点。这些节点，都是可以运行待调度 Pod 的宿主机。而在 Kubernetes 中，`默认的调度策略`有如下四种。第一种类型，叫作 **GeneralPredicate**s。

顾名思义，这一组过滤规则，负责的是最基础的调度策略。比如，`PodFitsResources` 计算的就是宿主机的 CPU 和内存资源等是否够用。当然，我在前面已经提到过，PodFitsResources 检查的只是 Pod 的 requests 字段。需要注意的是，Kubernetes 的调度器并没有为 GPU 等硬件资源定义具体的资源类型，而是统一用一种名叫**Extended Resource** 的、Key-Value 格式的扩展字段来描述的。比如下面这个例子：
```yml
apiVersion: v1
kind: Pod
metadata:
  name: extended-resource-demo
spec:
  containers:
  - name: extended-resource-demo-ctr
    image: nginx
    resources:
      requests:
        alpha.kubernetes.io/nvidia-gpu: 2
      limits:
        alpha.kubernetes.io/nvidia-gpu: 2
```
可以看到，我们这个 Pod 通过alpha.kubernetes.io/nvidia-gpu=2这样的定义方式，声明使用了两个 NVIDIA 类型的 GPU。而在 PodFitsResources 里面，调度器其实并不知道这个字段 Key 的含义是 GPU，而是直接使用后面的 Value 进行计算。当然，在 Node 的 `Capacity` 字段里，你也得相应地加上这台宿主机上 GPU 的总数，比如：alpha.kubernetes.io/nvidia-gpu=4。这些流程，我在后面讲解 Device Plugin 的时候会详细介绍。

而 `PodFitsHost` 检查的是，宿主机的名字是否跟 Pod 的 spec.nodeName 一致。PodFitsHostPorts 检查的是，Pod 申请的宿主机端口（spec.nodePort）是不是跟已经被使用的端口有冲突。PodMatchNodeSelector 检查的是，Pod 的 nodeSelector 或者 nodeAffinity 指定的节点，是否与待考察节点匹配，等等。可以看到，像上面这样一组 GeneralPredicates，正是 Kubernetes 考察一个 Pod 能不能运行在一个 Node 上最基本的过滤条件。所以，GeneralPredicates 也会被其他组件（比如 kubelet）直接调用。

我在上一篇文章中已经提到过，kubelet 在启动 Pod 前，会执行一个 Admit 操作来进行二次确认。这里二次确认的规则，就是执行一遍 GeneralPredicates。

第二种类型，**是与 Volume 相关的过滤规则**。

这一组过滤规则，负责的是跟容器持久化 Volume 相关的调度策略。其中，NoDiskConflict 检查的条件，是多个 Pod 声明挂载的持久化 Volume 是否有冲突。比如，AWS EBS 类型的 Volume，是不允许被两个 Pod 同时使用的。所以，当一个名叫 A 的 EBS Volume 已经被挂载在了某个节点上时，另一个同样声明使用这个 A Volume 的 Pod，就不能被调度到这个节点上了。而 MaxPDVolumeCountPredicate 检查的条件，则是一个节点上某种类型的持久化 Volume 是不是已经超过了一定数目，如果是的话，那么声明使用该类型持久化 Volume 的 Pod 就不能再调度到这个节点了。而 VolumeZonePredicate，则是检查持久化 Volume 的 Zone（高可用域）标签，是否与待考察节点的 Zone 标签相匹配。此外，这里还有一个叫作 VolumeBindingPredicate 的规则。它负责检查的，是该 Pod 对应的 PV 的 nodeAffinity 字段，是否跟某个节点的标签相匹配。

第三种类型，**是宿主机相关的过滤规则**。

这一组规则，主要考察待调度 Pod 是否满足 Node 本身的某些条件。比如，PodToleratesNodeTaints，负责检查的就是我们前面经常用到的 Node 的“污点”机制。只有当 Pod 的 Toleration 字段与 Node 的 Taint 字段能够匹配的时候，这个 Pod 才能被调度到该节点上。而 NodeMemoryPressurePredicate，检查的是当前节点的内存是不是已经不够充足，如果是的话，那么待调度 Pod 就不能被调度到该节点上。

第四种类型，**是 Pod 相关的过滤规则**。

这一组规则，跟 GeneralPredicates 大多数是重合的。而比较特殊的，是 `PodAffinityPredicate`。这个规则的作用，**是检查待调度 Pod 与 Node 上的已有 Pod 之间的亲密（affinity）和反亲密（anti-affinity）关系**。
```yml
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-antiaffinity
spec:
  affinity:
    podAntiAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution: 
      - weight: 100  
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: security 
              operator: In 
              values:
              - S2
          topologyKey: kubernetes.io/hostname
  containers:
  - name: with-pod-affinity
    image: docker.io/ocpqe/hello-pod
---
apiVersion: v1
kind: Pod
metadata:
  name: with-pod-affinity
spec:
  affinity:
    podAffinity: 
      requiredDuringSchedulingIgnoredDuringExecution: 
      - labelSelector:
          matchExpressions:
          - key: security 
            operator: In 
            values:
            - S1 
        topologyKey: failure-domain.beta.kubernetes.io/zone
  containers:
  - name: with-pod-affinity
    image: docker.io/ocpqe/hello-pod
```
上面这四种类型的 Predicates，就构成了调度器确定一个 Node 可以运行待调度 Pod 的基本策略。

在具体执行的时候， 当开始调度一个 Pod 时，Kubernetes 调度器会同时启动 16 个 Goroutine，来`并发`地为集群里的所有 Node 计算 Predicates，最后返回可以运行这个 Pod 的宿主机列表。需要注意的是，在为每个 Node 执行 Predicates 时，调度器会按照`固定的顺序`来进行检查。这个顺序，是按照 Predicates 本身的含义来确定的。比如，宿主机相关的 Predicates 会被放在相对靠前的位置进行检查。要不然的话，在一台资源已经严重不足的宿主机上，上来就开始计算 PodAffinityPredicate，是没有实际意义的。

接下来，我们再来看一下 `Priorities`。

在 Predicates 阶段完成了节点的“过滤”之后，Priorities 阶段的工作就是为这些节点打分。这里打分的范围是 0-10 分，得分最高的节点就是最后被 Pod 绑定的最佳节点。

Priorities 里最常用到的一个打分规则，是 `LeastRequestedPriority`。它的计算方法，可以简单地总结为如下所示的公式：
score = (cpu((capacity-sum(requested))10/capacity) + memory((capacity-sum(requested))10/capacity))/2

可以看到，这个算法实际上就是在选择空闲资源（CPU 和 Memory）最多的宿主机。

而与 LeastRequestedPriority 一起发挥作用的，还有 `BalancedResourceAllocation`。它的计算公式如下所示：
score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10

其中，每种资源的 Fraction 的定义是 ：Pod 请求的资源 / 节点上的可用资源。而 variance 算法的作用，则是计算每两种资源 Fraction 之间的“距离”。而最后选择的，则是资源 Fraction 差距最小的节点。所以说，BalancedResourceAllocation 选择的，其实是调度完成后，所有节点里各种资源分配最均衡的那个节点，从而避免一个节点上 CPU 被大量分配、而 Memory 大量剩余的情况。

此外，还有 NodeAffinityPriority、TaintTolerationPriority 和 InterPodAffinityPriority 这三种 Priority。顾名思义，它们与前面的 PodMatchNodeSelector、PodToleratesNodeTaints 和 PodAffinityPredicate 这三个 Predicate 的含义和计算方法是类似的。但是作为 Priority，一个 Node 满足上述规则的字段数目越多，它的得分就会越高。在默认 Priorities 里，还有一个叫作 ImageLocalityPriority 的策略。它是在 Kubernetes v1.12 里新开启的调度规则，即：如果待调度 Pod 需要使用的镜像很大，并且已经存在于某些 Node 上，那么这些 Node 的得分就会比较高。当然，为了避免这个算法引发调度堆叠，调度器在计算得分的时候还会根据镜像的分布进行优化，即：如果大镜像分布的节点数目很少，那么这些节点的权重就会被调低，从而“对冲”掉引起调度堆叠的风险。以上，就是 Kubernetes 调度器的 Predicates 和 Priorities 里默认调度规则的主要工作原理了。

在实际的执行过程中，调度器里关于集群和 Pod 的信息都已经缓存化，所以这些算法的执行过程还是比较快的。

首先需要明确的是，`优先级和抢占机制`，解决的是 Pod 调度失败时该怎么办的问题。正常情况下，当一个 Pod 调度失败后，它就会被暂时“搁置”起来，直到 Pod 被更新，或者集群状态发生变化，调度器才会对这个 Pod 进行重新调度。但在有时候，我们希望的是这样一个场景。当一个高优先级的 Pod 调度失败后，该 Pod 并不会被“搁置”，而是会“挤走”某个 Node 上的一些低优先级的 Pod 。这样就可以保证这个高优先级 Pod 的调度成功。这个特性，其实也是一直以来就存在于 Borg 以及 Mesos 等项目里的一个基本功能。

而在 Kubernetes 里，优先级和抢占机制是在 1.10 版本后才逐步可用的。要使用这个机制，你首先需要在 Kubernetes 里提交一个 `PriorityClass` 的定义，如下所示：
```yml
apiVersion: scheduling.k8s.io/v1beta1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for high priority service pods only."
```
上面这个 YAML 文件，定义的是一个名叫 high-priority 的 PriorityClass，其中 value 的值是 1000000 （一百万）。Kubernetes 规定，优先级是一个 32 bit 的整数，最大值不超过 1000000000（10 亿，1 billion），并且值越大代表优先级越高。而超出 10 亿的值，其实是被 Kubernetes 保留下来分配给`系统 Pod` 使用的。显然，这样做的目的，就是保证系统 Pod 不会被用户抢占掉。

而一旦上述 YAML 文件里的 `globalDefault` 被设置为 true 的话，那就意味着这个 PriorityClass 的值会成为系统的默认值。而如果这个值是 false，就表示我们只希望声明使用该 PriorityClass 的 Pod 拥有值为 1000000 的优先级，而对于没有声明 PriorityClass 的 Pod 来说，它们的优先级就是 0。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

可以看到，这个 Pod 通过 priorityClassName 字段，声明了要使用名叫 high-priority 的 PriorityClass。当这个 Pod 被提交给 Kubernetes 之后，Kubernetes 的 PriorityAdmissionController 就会自动将这个 Pod 的 spec.priority 字段设置为 1000000。

而我在前面的文章中曾为你介绍过，**调度器里维护着一个调度队列。所以，当 Pod 拥有了优先级之后，高优先级的 Pod 就可能会比低优先级的 Pod 提前出队，从而尽早完成调度过程**。这个过程，就是“优先级”这个概念在 Kubernetes 里的主要体现。

而当一个高优先级的 Pod 调度失败的时候，调度器的抢占能力就会被触发。这时，调度器就会试图从当前集群里寻找一个节点，使得当这个节点上的一个或者多个低优先级 Pod 被删除后，待调度的高优先级 Pod 就可以被调度到这个节点上。这个过程，就是“`抢占`”这个概念在 Kubernetes 里的主要体现。

为了方便叙述，我接下来会把待调度的高优先级 Pod 称为“抢占者”（Preemptor）。当上述抢占过程发生时，抢占者并不会立刻被调度到被抢占的 Node 上。事实上，调度器只会将抢占者的 spec.nominatedNodeName 字段，设置为被抢占的 Node 的名字。然后，抢占者会重新进入下一个调度周期，然后在新的调度周期里来决定是不是要运行在被抢占的节点上。这当然也就意味着，即使在下一个调度周期，调度器也不会保证抢占者一定会运行在被抢占的节点上。这样设计的一个重要原因是，调度器只会通过标准的 DELETE API 来删除被抢占的 Pod，所以，这些 Pod 必然是有一定的“优雅退出”时间（默认是 30s）的。而在这段时间里，其他的节点也是有可能变成可调度的，或者直接有新的节点被添加到这个集群中来。所以，鉴于优雅退出期间，集群的可调度性可能会发生的变化，把抢占者交给下一个调度周期再处理，是一个非常合理的选择。而在抢占者等待被调度的过程中，如果有其他更高优先级的 Pod 也要抢占同一个节点，那么调度器就会清空原抢占者的 spec.nominatedNodeName 字段，从而允许更高优先级的抢占者执行抢占，并且，这也就使得原抢占者本身，也有机会去重新抢占其他节点。这些，都是设置 `nominatedNodeName` 字段的主要目的。

那么，Kubernetes 调度器里的`抢占机制`，又是如何设计的呢？接下来，我就为你详细讲述一下这其中的原理。我在前面已经提到过，抢占发生的原因，一定是一个高优先级的 Pod 调度失败。这一次，我们还是称这个 Pod 为“抢占者”，称被抢占的 Pod 为“牺牲者”（victims）。而 Kubernetes 调度器实现抢占算法的一个最重要的设计，就是在调度队列的实现里，使用了两个不同的队列。

第一个队列，叫作 `activeQ`。凡是在 activeQ 里的 Pod，都是下一个调度周期需要调度的对象。所以，当你在 Kubernetes 集群里新创建一个 Pod 的时候，调度器会将这个 Pod 入队到 activeQ 里面。而我在前面提到过的、调度器不断从队列里出队（Pop）一个 Pod 进行调度，实际上都是从 activeQ 里出队的。

第二个队列，叫作 `unschedulableQ`，专门用来存放调度失败的 Pod。而这里的一个关键点就在于，当一个 unschedulableQ 里的 Pod 被更新之后，调度器会自动把这个 Pod 移动到 activeQ 里，从而给这些调度失败的 Pod “重新做人”的机会。

现在，回到我们的抢占者调度失败这个时间点上来。调度失败之后，抢占者就会被放进 unschedulableQ 里面。然后，这次失败事件就会触发调度器为抢占者寻找牺牲者的流程。

第一步，调度器会检查这次失败事件的原因，来确认抢占是不是可以帮助抢占者找到一个新节点。这是因为有很多 Predicates 的失败是不能通过抢占来解决的。比如，PodFitsHost 算法（负责的是，检查 Pod 的 nodeSelector 与 Node 的名字是否匹配），这种情况下，除非 Node 的名字发生变化，否则你即使删除再多的 Pod，抢占者也不可能调度成功。第二步，如果确定抢占可以发生，那么调度器就会把自己缓存的所有节点信息复制一份，然后使用这个副本来模拟抢占过程。

第二步，如果确定抢占可以发生，那么调度器就会把自己缓存的所有节点信息复制一份，然后使用这个副本来模拟抢占过程。这里的抢占过程很容易理解。调度器会检查缓存副本里的每一个节点，然后从该节点上最低优先级的 Pod 开始，逐一“删除”这些 Pod。而每删除一个低优先级 Pod，调度器都会检查一下抢占者是否能够运行在该 Node 上。一旦可以运行，调度器就记录下这个 Node 的名字和被删除 Pod 的列表，这就是一次抢占过程的结果了。当遍历完所有的节点之后，调度器会在上述模拟产生的所有抢占结果里做一个选择，找出最佳结果。而这一步的判断原则，就是尽量减少抢占对整个系统的影响。比如，需要抢占的 Pod 越少越好，需要抢占的 Pod 的优先级越低越好，等等。

第一步，调度器会检查牺牲者列表，清理这些 Pod 所携带的 nominatedNodeName 字段。第二步，调度器会把抢占者的 nominatedNodeName，设置为被抢占的 Node 的名字。第三步，调度器会开启一个 Goroutine，同步地删除牺牲者。而第二步对抢占者 Pod 的更新操作，就会触发到我前面提到的“重新做人”的流程，从而让抢占者在下一个调度周期重新进入调度流程。所以接下来，调度器就会通过正常的调度流程把抢占者调度成功。这也是为什么，我前面会说调度器并不保证抢占的结果：在这个正常的调度流程里，是一切皆有可能的。不过，对于任意一个待调度 Pod 来说，因为有上述抢占者的存在，它的调度过程，其实是有一些特殊情况需要特殊处理的。具体来说，在为某一对 Pod 和 Node 执行 Predicates 算法的时候，如果待检查的 Node 是一个即将被抢占的节点，即：调度队列里有 nominatedNodeName 字段值是该 Node 名字的 Pod 存在（可以称之为：“潜在的抢占者”）。那么，调度器就会对这个 Node ，将同样的 Predicates 算法运行两遍。第一遍， 调度器会假设上述“潜在的抢占者”已经运行在这个节点上，然后执行 Predicates 算法；第二遍， 调度器会正常执行 Predicates 算法，即：不考虑任何“潜在的抢占者”。而只有这两遍 Predicates 算法都能通过时，这个 Pod 和 Node 才会被认为是可以绑定（bind）的。不难想到，这里需要执行第一遍 Predicates 算法的原因，是由于 InterPodAntiAffinity 规则的存在。由于 InterPodAntiAffinity 规则关心待考察节点上所有 Pod 之间的互斥关系，所以我们在执行调度算法时必须考虑，如果抢占者已经存在于待考察 Node 上时，待调度 Pod 还能不能调度成功。当然，这也就意味着，我们在这一步只需要考虑那些优先级等于或者大于待调度 Pod 的抢占者。毕竟对于其他较低优先级 Pod 来说，待调度 Pod 总是可以通过抢占运行在待考察 Node 上。而我们需要执行第二遍 Predicates 算法的原因，则是因为“潜在的抢占者”最后不一定会运行在待考察的 Node 上。关于这一点，我在前面已经讲解过了：Kubernet

### 总结

调度器的作用就是为Pod寻找一个合适的Node。

调度过程：待调度Pod被提交到apiServer -> 更新到etcd -> 调度器Watch etcd感知到有需要调度的pod（Informer） -> 取出待调度Pod的信息 ->Predicates： 挑选出可以运行该Pod的所有Node  ->  Priority：给所有Node打分 -> 将Pod绑定到得分最高的Node上 -> 将Pod信息更新回Etcd -> node的kubelet感知到etcd中有自己node需要拉起的pod -> 取出该Pod信息，做基本的二次检测（端口，资源等） -> 在node 上拉起该pod 。

Predicates阶段会有很多过滤规则：比如volume相关，node相关，pod相关
Priorities阶段会为Node打分，Pod调度到得分最高的Node上，打分规则比如： 空余资源、实际物理剩余、镜像大小、Pod亲和性等

Kuberentes中可以为Pod设置优先级，高优先级的Pod可以： 1、在调度队列中先出队进行调度 2、调度失败时，触发抢占，调度器为其抢占低优先级Pod的资源。

Kuberentes默认调度器有两个调度队列：
activeQ：凡事在该队列里的Pod，都是下一个调度周期需要调度的
unschedulableQ:  存放调度失败的Pod，当里面的Pod更新后就会重新回到activeQ，进行“重新调度”

默认调度器的抢占过程： 确定要发生抢占 -> 调度器将所有节点信息复制一份，开始模拟抢占 ->  检查副本里的每一个节点，然后从该节点上逐个删除低优先级Pod，直到满足抢占者能运行 -> 找到一个能运行抢占者Pod的node -> 记录下这个Node名字和被删除Pod的列表 -> 模拟抢占结束 -> 开始真正抢占 -> 删除被抢占者的Pod，将抢占者调度到Node上 

### gpu 支持

以 NVIDIA 的 GPU 设备为例，上面的需求就意味着当用户的容器被创建之后，这个容器里必须出现如下两部分设备和目录：GPU 设备，比如 /dev/nvidia0；GPU 驱动目录，比如 /usr/local/nvidia/*。其中，GPU 设备路径，正是该容器启动时的 Devices 参数；而驱动目录，则是该容器启动时的 Volume 参数。所以，在 Kubernetes 的 GPU 支持的实现里，kubelet 实际上就是将上述两部分内容，设置在了创建该容器的 CRI （Container Runtime Interface）参数里面。这样，等到该容器启动之后，对应的容器里就会出现 GPU 设备和驱动的路径了。不过，Kubernetes 在 Pod 的 API 对象里，并没有为 GPU 专门设置一个资源类型字段，而是使用了一种叫作 `Extended Resource`（ER）的特殊字段来负责传递 GPU 的信息。比如下面这个例子：
```yml
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vector-add
spec:
  restartPolicy: OnFailure
  containers:
    - name: cuda-vector-add
      image: "k8s.gcr.io/cuda-vector-add:v0.1"
      resources:
        limits:
          nvidia.com/gpu: 1
```
可以看到，在上述 Pod 的 limits 字段里，这个资源的名称是nvidia.com/gpu，它的值是 1。也就是说，这个 Pod 声明了自己要使用一个 NVIDIA 类型的 GPU。而在 kube-scheduler 里面，它其实并不关心这个字段的具体含义，只会在计算的时候，一律将调度器里保存的该类型资源的可用量，直接减去 Pod 声明的数值即可。所以说，Extended Resource，其实是 Kubernetes 为用户设置的一种对自定义资源的支持。

当然，为了能够让调度器知道这个自定义类型的资源在每台宿主机上的可用量，宿主机节点本身，就必须能够向 API Server 汇报该类型资源的可用数量。在 Kubernetes 里，各种类型的资源可用量，其实是` Node` 对象 Status 字段的内容，比如下面这个例子：
```yml
apiVersion: v1
kind: Node
metadata:
  name: node-1
...
Status:
  Capacity:
   cpu:  2
   memory:  2049008Ki
```
而为了能够在上述 Status 字段里添加自定义资源的数据，你就必须使用 `PATCH API` 来对该 Node 对象进行更新，加上你的自定义资源的数量。这个 PATCH 操作，可以简单地使用 curl 命令来发起，如下所示：
```shell
# 启动 Kubernetes 的客户端 proxy，这样你就可以直接使用 curl 来跟 Kubernetes  的API Server 进行交互了
$ kubectl proxy

# 执行 PACTH 操作
$ curl --header "Content-Type: application/json-patch+json" \
--request PATCH \
--data '[{"op": "add", "path": "/status/capacity/nvidia.com/gpu", "value": "1"}]' \
http://localhost:8001/api/v1/nodes/<your-node-name>/status
```

当然，在 Kubernetes 的 GPU 支持方案里，你并不需要真正去做上述关于 Extended Resource 的这些操作。在 Kubernetes 中，对所有硬件加速设备进行管理的功能，都是由一种叫作 `Device Plugin` 的插件来负责的。这其中，当然也就包括了对该硬件的 Extended Resource 进行汇报的逻辑。Kubernetes 的 Device Plugin 机制，我可以用如下所示的一幅示意图来和你解释清楚。

![](../../reference/pic/deviceplugin.webp)
我们先从这幅示意图的右侧开始看起。

首先，对于每一种硬件设备，都需要有它所对应的 Device Plugin 进行管理，这些 Device Plugin，都通过 gRPC 的方式，同 kubelet 连接起来。以 NVIDIA GPU 为例，它对应的插件叫作`NVIDIA GPU device plugin`。这个 Device Plugin 会通过一个叫作 `ListAndWatch` 的 API，定期向 kubelet 汇报该 Node 上 GPU 的列表。比如，在我们的例子里，一共有三个 GPU（GPU0、GPU1 和 GPU2）。这样，kubelet 在拿到这个列表之后，就可以直接在它向 APIServer 发送的心跳里，以 `Extended Resource` 的方式，加上这些 GPU 的数量，比如nvidia.com/gpu=3。所以说，用户在这里是不需要关心 GPU 信息向上的汇报流程的。

需要注意的是，ListAndWatch 向上汇报的信息，只有本机上 GPU 的 ID 列表，而不会有任何关于 GPU 设备本身的信息。而且 kubelet 在向 API Server 汇报的时候，只会汇报该 GPU 对应的 Extended Resource 的数量。当然，kubelet 本身，会将这个 GPU 的 ID 列表保存在自己的内存里，并通过 ListAndWatch API 定时更新。

而当一个 Pod 想要使用一个 GPU 的时候，它只需要像我在本文一开始给出的例子一样，在 Pod 的 limits 字段声明nvidia.com/gpu: 1。那么接下来，Kubernetes 的调度器就会从它的缓存里，寻找 GPU 数量满足条件的 Node，然后将缓存里的 GPU 数量减 1，完成 Pod 与 Node 的绑定。

这个调度成功后的 Pod 信息，自然就会被对应的 kubelet 拿来进行容器操作。而当 kubelet 发现这个 Pod 的容器请求一个 GPU 的时候，kubelet 就会从自己持有的 GPU 列表里，为这个容器分配一个 GPU。此时，kubelet 就会向本机的 Device Plugin 发起一个 Allocate() 请求。这个请求携带的参数，正是即将分配给该容器的设备 ID 列表。

当 Device Plugin 收到 Allocate 请求之后，它就会根据 kubelet 传递过来的设备 ID，从 Device Plugin 里找到这些设备对应的设备路径和驱动目录。当然，这些信息，正是 Device Plugin 周期性的从本机查询到的。比如，在 NVIDIA Device Plugin 的实现里，它会定期访问 nvidia-docker 插件，从而获取到本机的 GPU 信息。而被分配 GPU 对应的设备路径和驱动目录信息被返回给 kubelet 之后，kubelet 就完成了为一个容器分配 GPU 的操作。接下来，kubelet 会把这些信息追加在创建该容器所对应的 CRI 请求当中。这样，当这个 CRI 请求发给 Docker 之后，Docker 为你创建出来的容器里，就会出现这个 GPU 设备，并把它所需要的驱动目录挂载进去。

至此，Kubernetes 为一个 Pod 分配一个 GPU 的流程就完成了。

### gpu 总结

Kuberentes通过Extended Resource来支持自定义资源，比如GPU。为了让调度器知道这种自定义资源在各Node上的数量，需要的Node里添加自定义资源的数量。实际上，这些信息并不需要人工去维护，所有的硬件加速设备的管理都通过Device Plugin插件来支持，也包括对该硬件的Extended Resource进行汇报的逻辑。

Device Plugin 、kubelet、调度器如何协同工作：

汇报资源： Device Plugin通过gRPC与本机kubelet连接 ->  Device Plugin定期向kubelet汇报设备信息，比如GPU的数量 -> kubelet 向APIServer发送的心跳中，以Extended Reousrce的方式加上这些设备信息，比如GPU的数量 

调度： Pod申明需要一个GPU -> 调度器找到GPU数量满足条件的node -> Pod绑定到对应的Node上 -> kubelet发现需要拉起一个Pod，且该Pod需要GPU -> kubelet向 Device Plugin 发起 Allocate()请求 -> Device Plugin根据kubelet传递过来的需求，找到这些设备对应的设备路径和驱动目录，并返回给kubelet -> kubelet将这些信息追加在创建Pod所对应的CRI请求中 -> 容器创建完成之后，就会出现这个GPU设备（设备路径+驱动目录）-> 调度完成

## kubelet cri

![](../../reference/pic/kubelet.webp)

可以看到，kubelet 的工作核心，就是一个控制循环，即：SyncLoop（图中的大圆圈）。而驱动这个控制循环运行的事件，包括四种：Pod 更新事件；Pod 生命周期变化；kubelet 本身设置的执行周期；定时的清理事件。

所以，跟其他控制器类似，**kubelet 启动的时候，要做的第一件事情，就是设置 Listers，也就是注册它所关心的各种事件的 Informer**。这些 Informer，就是 SyncLoop 需要处理的数据的来源。

此外，kubelet 还负责维护着很多很多其他的`子控制循环`（也就是图中的小圆圈）。这些控制循环的名字，一般被称作某某 Manager，比如 Volume Manager、Image Manager、Node Status Manager 等等。

不难想到，这些控制循环的责任，就是通过控制器模式，完成 kubelet 的某项具体职责。比如 Node Status Manager，就负责响应 Node 的状态变化，然后将 Node 的状态收集起来，并通过 Heartbeat 的方式上报给 APIServer。再比如 CPU Manager，就负责维护该 Node 的 CPU 核的信息，以便在 Pod 通过 cpuset 的方式请求 CPU 核的时候，能够正确地管理 CPU 核的使用量和可用量。

那么这个 SyncLoop，又是如何根据 Pod 对象的变化，来进行容器操作的呢？**实际上，kubelet 也是通过 `Watch` 机制，监听了与自己相关的 Pod 对象的变化。当然，这个 Watch 的过滤条件是该 Pod 的 `nodeName` 字段与自己相同。kubelet 会把这些 Pod 的信息缓存在自己的内存里。**

而当一个 Pod 完成调度、与一个 Node 绑定起来之后， 这个 Pod 的变化就会触发 kubelet 在控制循环里注册的 Handler，也就是上图中的 `HandlePods` 部分。此时，通过检查该 Pod 在 kubelet 内存里的状态，kubelet 就能够判断出这是一个新调度过来的 Pod，从而触发 Handler 里 ADD 事件对应的处理逻辑。在具体的处理过程当中，kubelet 会启动一个名叫 `Pod Update Worker` 的、单独的 Goroutine 来完成对 Pod 的处理工作。

在这里需要注意的是，kubelet 调用下层容器运行时的执行过程，并不会直接调用 Docker 的 API，而是通过一组叫作 `CRI`（Container Runtime Interface，容器运行时接口）的 gRPC 接口来间接执行的。

![](../../reference/pic/cri.webp)

可以看到，当 Kubernetes 通过编排能力创建了一个 Pod 之后，调度器会为这个 Pod 选择一个具体的节点来运行。这时候，kubelet 当然就会通过前面讲解过的 SyncLoop 来判断需要执行的具体操作，比如创建一个 Pod。那么此时，kubelet 实际上就会调用一个叫作 GenericRuntime 的通用组件来发起创建 Pod 的 CRI 请求。

那么，这个 CRI 请求，又该由谁来响应呢？

如果你使用的容器项目是 Docker 的话，那么负责响应这个请求的就是一个叫作 **dockershim** 的组件。它会把 CRI 请求里的内容拿出来，然后组装成 Docker API 请求发给 Docker Daemon。

需要注意的是，在 Kubernetes 目前的实现里，dockershim 依然是 kubelet 代码的一部分。当然，在将来，dockershim 肯定会被从 kubelet 里移出来，甚至直接被废弃掉。

而更普遍的场景，就是你需要在每台宿主机上单独安装一个负责响应 CRI 的组件，这个组件，一般被称作 CRI shim。顾名思义，CRI shim 的工作，就是扮演 kubelet 与容器项目之间的“垫片”（shim）。所以它的作用非常单一，那就是实现 CRI 规定的每个接口，然后把具体的 CRI 请求“翻译”成对后端容器项目的请求或者操作。

所以说，这里的 CRI shim，就是容器项目的维护者们自由发挥的“场地”了。而除了 dockershim 之外，其他容器运行时的 CRI shim，都是需要额外部署在宿主机上的。举个例子。CNCF 里的 **containerd** 项目，就可以提供一个典型的 `CRI shim` 的能力，即：将 Kubernetes 发出的 CRI 请求，转换成对 containerd 的调用，然后创建出 **runC** 容器。

而 runC 项目，才是负责执行我们前面讲解过的设置容器 Namespace、Cgroups 和 chroot 等基础操作的组件。所以，这几层的组合关系，可以用如下所示的示意图来描述。
![](../../reference/pic/containerd.webp)

具体地说，我们可以把 **CRI 分为两组**：
第一组，是 `RuntimeService`。它提供的接口，主要是跟容器相关的操作。比如，创建和启动容器、删除容器、执行 exec 命令等等。
而第二组，则是 `ImageService`。它提供的接口，主要是容器镜像相关的操作，比如拉取镜像、删除镜像等等。

比如，当我们执行 kubectl run 创建了一个名叫 foo 的、包括了 A、B 两个容器的 Pod 之后。这个 Pod 的信息最后来到 kubelet，kubelet 就会按照图中所示的顺序来调用 CRI 接口。
![](../../reference/pic/cripod.webp)

在这一部分，**CRI 设计的一个重要原则，就是确保这个接口本身，只关注容器，不关注 Pod**。

所以，在 CRI 的设计里，并没有一个直接创建 Pod 或者启动 Pod 的接口。

不过，相信你也已经注意到了，CRI 里还是有一组叫作 `RunPodSandbox` 的接口的。这个 PodSandbox，对应的并不是 Kubernetes 里的 Pod API 对象，而只是抽取了 Pod 里的一部分与容器运行时相关的字段，比如 HostName、DnsConfig、CgroupParent 等。所以说，PodSandbox 这个接口描述的，其实是 Kubernetes 将 Pod 这个概念映射到容器运行时层面所需要的字段，或者说是一个 Pod 对象子集。

**在具体的 CRI shim 中，这些接口的实现是可以完全不同的**。

比如，如果是 Docker 项目，dockershim 就会创建出一个名叫 foo 的 Infra 容器（pause 容器），用来“hold”住整个 Pod 的 Network Namespace。

而如果是基于虚拟化技术的容器，比如 Kata Containers 项目，它的 CRI 实现就会直接创建出一个轻量级虚拟机来充当 Pod。

此外，需要注意的是，**在 RunPodSandbox 这个接口的实现中，你还需要调用 networkPlugin.SetUpPod(…) 来为这个 Sandbox 设置网络**。这个 SetUpPod(…) 方法，实际上就在执行 CNI 插件里的 add(…) 方法，也就是我在前面为你讲解过的 `CNI` 插件为 Pod 创建网络，并且把 Infra 容器加入到网络中的操作。

接下来，kubelet 继续调用 CreateContainer 和 StartContainer 接口来创建和启动容器 A、B。对应到 dockershim 里，就是直接启动 A，B 两个 Docker 容器。所以最后，宿主机上会出现三个 Docker 容器组成这一个 Pod。

而如果是 Kata Containers 的话，CreateContainer 和 StartContainer 接口的实现，就只会在前面创建的轻量级虚拟机里创建两个 A、B 容器对应的 Mount Namespace。所以，最后在宿主机上，只会有一个叫作 foo 的轻量级虚拟机在运行。

除了上述对容器生命周期的实现之外，CRI shim 还有一个重要的工作，就是如何实现 exec、logs 等接口。这些接口跟前面的操作有一个很大的不同，就是这些 gRPC 接口调用期间，**kubelet 需要跟容器项目`维护一个长连接来传输数据`。这种 API，我们就称之为 `Streaming API`**。

CRI shim 里对 Streaming API 的实现，依赖于一套独立的 `Streaming Server 机制`。这一部分原理，可以用如下所示的示意图来为你描述。
![](../../reference/pic/streamingapi.webp)
可以看到，当我们对一个容器执行 kubectl exec 命令的时候，这个请求首先交给 API Server，然后 API Server 就会调用 kubelet 的 Exec API。

这时，kubelet 就会调用 CRI 的 Exec 接口，而负责响应这个接口的，自然就是具体的 CRI shim。但在这一步，CRI shim 并不会直接去调用后端的容器项目（比如 Docker ）来进行处理，而只会返回一个 URL 给 kubelet。

这个 URL，就是该 CRI shim 对应的 Streaming Server 的地址和端口。而 kubelet 在拿到这个 URL 之后，就会把它以 Redirect 的方式返回给 API Server。

所以这时候，API Server 就会通过重定向来向 `Streaming Server` 发起真正的 /exec 请求，与它建立长连接。

## kata container and gvisor

不难看到，无论是 Kata Containers，还是 gVisor，它们实现安全容器的方法其实是殊途同归的。**这两种容器实现的本质，都是给进程分配了一个`独立的操作系统内核`，从而避免了让容器共享宿主机的内核**。这样，容器进程能够看到的攻击面，就从整个宿主机内核变成了一个极小的、独立的、以容器为单位的内核，从而有效解决了容器进程发生“逃逸”或者夺取整个宿主机的控制权的问题。

而它们的区别在于，Kata Containers 使用的是`传统的虚拟化技术`，通过虚拟硬件模拟出了一台“小虚拟机”，然后在这个小虚拟机里安装了一个裁剪后的 Linux 内核来实现强隔离。而 gVisor 的做法则更加激进，Google 的工程师直接用 Go 语言“`模拟`”出了一个运行在用户态的操作系统内核，然后通过这个模拟的内核来代替容器进程向宿主机发起有限的、可控的系统调用。


## prometheus

可以看到，Prometheus 项目工作的核心，是使用 Pull （抓取）的方式去搜集被监控对象的 Metrics 数据（监控指标数据），然后，再把这些数据保存在一个 TSDB （时间序列数据库，比如 OpenTSDB、InfluxDB 等）当中，以便后续可以按照时间进行检索。

有了这套核心监控机制， Prometheus 剩下的组件就是用来配合这套机制的运行。比如 Pushgateway，可以允许被监控对象以 `Push` 的方式向 Prometheus 推送 Metrics 数据。而 Alertmanager，则可以根据 Metrics 信息灵活地设置报警。当然， Prometheus 最受用户欢迎的功能，还是通过 Grafana 对外暴露出的、可以灵活配置的监控数据可视化界面。

**第一种 Metrics，是宿主机的监控数据。这部分数据的提供，需要借助一个由 Prometheus 维护的Node Exporter 工具。**

**第二种 Metrics，是来自于 Kubernetes 的 API Server、kubelet 等组件的 /metrics API。**除了常规的 CPU、内存的信息外，这部分信息还主要包括了各个组件的核心监控指标。比如，对于 API Server 来说，它就会在 /metrics API 里，暴露出各个 Controller 的工作队列（Work Queue）的长度、请求的 QPS 和延迟数据等等。这些信息，是检查 Kubernetes 本身工作情况的主要依据。

**第三种 Metrics，是 Kubernetes 相关的监控数据。**这部分数据，一般叫作 Kubernetes 核心监控数据（core metrics）。这其中包括了 Pod、Node、容器、Service 等主要 Kubernetes 核心概念的 Metrics。

其中，容器相关的 Metrics 主要来自于 **kubelet 内置的 cAdvisor 服务**。在 kubelet 启动后，cAdvisor 服务也随之启动，而它能够提供的信息，可以细化到每一个容器的 CPU 、文件系统、内存、网络等资源的使用情况。

需要注意的是，这里提到的 Kubernetes 核心监控数据，其实使用的是 Kubernetes 的一个非常重要的扩展能力，叫作 **Metrics Server**。

Metrics Server 在 Kubernetes 社区的定位，其实是用来取代 Heapster 这个项目的。在 Kubernetes 项目发展的初期，Heapster 是用户获取 Kubernetes 监控数据（比如 Pod 和 Node 的资源使用情况） 的主要渠道。而后面提出来的 Metrics Server，则把这些信息，通过标准的 Kubernetes API 暴露了出来。这样，Metrics 信息就跟 Heapster 完成了解耦，允许 Heapster 项目慢慢退出舞台。

而有了 Metrics Server 之后，用户就可以通过标准的 Kubernetes API 来访问到这些监控数据了。比如，下面这个 URL：
`http://127.0.0.1:8001/apis/metrics.k8s.io/v1beta1/namespaces/<namespace-name>/pods/<pod-name>`

当你访问这个 Metrics API 时，它就会为你返回一个 Pod 的监控数据，而这些数据，其实是从 kubelet 的 Summary API （即 `<kubelet_ip>:<kubelet_port>/stats/summary`）采集而来的。Summary API 返回的信息，既包括了 cAdVisor 的监控数据，也包括了 kubelet 本身汇总的信息。

**要指出的是， Metrics Server 并不是 kube-apiserver 的一部分，而是通过 `Aggregator` 这种插件机制，在独立部署的情况下同 kube-apiserver 一起统一对外服务的。**

可以看到，当 Kubernetes 的 API Server 开启了 Aggregator 模式之后，你再访问 apis/metrics.k8s.io/v1beta1 的时候，实际上访问到的是一个叫作 `kube-aggregator` 的代理。而 kube-apiserver，正是这个代理的一个后端；而 Metrics Server，则是另一个后端。

而且，在这个机制下，你还可以添加更多的后端给这个 kube-aggregator。所以 kube-aggregator 其实就是一个根据 URL 选择具体的 API 后端的代理服务器。通过这种方式，我们就可以很方便地扩展 Kubernetes 的 API 了。

### 自定义监控指标

实际上，借助上述监控体系，Kubernetes 就可以为你提供一种非常有用的能力，那就是 **Custom Metrics**，自定义监控指标。

在过去的很多 PaaS 项目中，其实都有一种叫作 **Auto Scaling**，即自动水平扩展的功能。只不过，这个功能往往只能依据某种指定的资源类型执行水平扩展，比如 CPU 或者 Memory 的使用值。

而在真实的场景中，用户需要进行 Auto Scaling 的依据往往是自定义的监控指标。比如，某个应用的等待队列的长度，或者某种应用相关资源的使用情况。这些复杂多变的需求，在传统 PaaS 项目和其他容器编排项目里，几乎是不可能轻松支持的。

而凭借强大的 API 扩展机制，Custom Metrics 已经成为了 Kubernetes 的一项标准能力。并且，Kubernetes 的自动扩展器组件 **Horizontal Pod Autoscaler** （HPA）， 也可以直接使用 Custom Metrics 来执行用户指定的扩展策略，这里的整个过程都是非常灵活和可定制的。

不难想到，Kubernetes 里的 Custom Metrics 机制，也是借助 `Aggregator APIServer` 扩展机制来实现的。

这里的具体原理是，当你把 **Custom Metrics APIServer** 启动之后，Kubernetes 里就会出现一个叫作custom.metrics.k8s.io的 API。而当你访问这个 URL 时，Aggregator 就会把你的请求转发给 Custom Metrics APIServer 。而 Custom Metrics APIServer 的实现，**其实就是一个 Prometheus 项目的 Adaptor**。

比如，现在我们要实现一个根据指定 Pod 收到的 HTTP 请求数量来进行 Auto Scaling 的 Custom Metrics，这个 Metrics 就可以通过访问如下所示的自定义监控 URL 获取到：
`https://<apiserver_ip>/apis/custom-metrics.metrics.k8s.io/v1beta1/namespaces/default/pods/sample-metrics-app/http_requests `

这里的工作原理是，当你访问这个 URL 的时候，Custom Metrics APIServer 就会去 Prometheus 里查询名叫 sample-metrics-app 这个 Pod 的 http_requests 指标的值，然后按照固定的格式返回给访问者。

```yaml
kind: HorizontalPodAutoscaler
apiVersion: autoscaling/v2beta1
metadata:
  name: sample-metrics-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sample-metrics-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Object
    object:
      target:
        kind: Service
        name: sample-metrics-app
      metricName: http_requests
      targetValue: 100
```

可以看到，HPA 的配置，就是你设置 Auto Scaling 规则的地方。比如，scaleTargetRef 字段，就指定了被监控的对象是名叫 sample-metrics-app 的 `Deployment`，也就是我们上面部署的被监控应用。并且，它最小的实例数目是 2，最大是 10。在 metrics 字段，我们指定了这个 HPA 进行 Scale 的依据，是名叫 http_requests 的 Metrics。而获取这个 Metrics 的途径，则是访问名叫 sample-metrics-app 的 `Service`。有了这些字段里的定义， HPA 就可以向如下所示的 URL 发起请求来获取 Custom Metrics 的值了：
`https://<apiserver_ip>/apis/custom-metrics.metrics.k8s.io/v1beta1/namespaces/default/services/sample-metrics-app/http_requests`

需要注意的是，上述这个 URL 对应的被监控对象，是我们的应用对应的 Service。这跟本文一开始举例用到的 Pod 对应的 Custom Metrics URL 是不一样的。当然，**对于一个多实例应用来说，通过 Service 来采集 Pod 的 Custom Metrics 其实才是合理的做法**。

在本篇文章中，我为你详细讲解了 Kubernetes 里对自定义监控指标，即 Custom Metrics 的设计与实现机制。这套机制的可扩展性非常强，也终于使得 Auto Scaling 在 Kubernetes 里面不再是一个“食之无味”的鸡肋功能了。另外可以看到，Kubernetes 的 Aggregator APIServer，是一个非常行之有效的 API 扩展机制。

而且，Kubernetes 社区已经为你提供了一套叫作 `KubeBuilder` 的工具库，帮助你生成一个 API Server 的完整代码框架，你只需要在里面添加自定义 API，以及对应的业务逻辑即可。

### hpa 总结

HPA 通过 HorizontalPodAutoscaler 配置要访问的 Custom Metrics, 来决定如何scale。

Custom Metric APIServer 的实现其实是一个Prometheus 的Adaptor，会去Prometheus中读取某个Pod/Servicce的具体指标值。比如，http request的请求率。

Prometheus 通过 ServiceMonitor object 配置需要监控的pod和endpoints，来确定监控哪些pod的metrics。

应用需要实现/metrics， 来响应Prometheus的数据采集请求。

## 日志

在前面的文章中，我为你详细讲解了 Kubernetes 的核心监控体系和自定义监控体系的设计与实现思路。而在本篇文章里，我就来为你详细介绍一下 Kubernetes 里关于容器日志的处理方式。

首先需要明确的是，Kubernetes 里面对容器日志的处理方式，都叫作 `cluster-level-logging`，即：这个日志处理系统，与容器、Pod 以及 Node 的生命周期都是完全无关的。这种设计当然是为了保证，无论是容器挂了、Pod 被删除，甚至节点宕机的时候，应用的日志依然可以被正常获取到。

而对于一个容器来说，当应用把日志输出到 stdout 和 stderr 之后，容器项目在默认情况下就会把这些日志输出到宿主机上的一个 JSON 文件里。这样，你通过 kubectl logs 命令就可以看到这些容器的日志了。

上述机制，就是我们今天要讲解的容器日志收集的基础假设。而如果你的应用是把文件输出到其他地方，比如直接输出到了容器里的某个文件里，或者输出到了远程存储里，那就属于特殊情况了。当然，我在文章里也会对这些特殊情况的处理方法进行讲述。

而 Kubernetes 本身，实际上是不会为你做容器日志收集工作的，所以为了实现上述 cluster-level-logging，你需要在部署集群的时候，提前对具体的日志方案进行规划。而 Kubernetes 项目本身，主要为你推荐了**三种日志方案**。

第一种，在 Node 上部署 `logging agent`，将日志文件转发到后端存储里保存起来。这个方案的架构图如下所示。
![](https://static001.geekbang.org/resource/image/b5/43/b5515aed076aa6af63ace55b62d36243.jpg?wh=2284*1610)

不难看到，这里的核心就在于 logging agent ，它一般都会以 DaemonSet 的方式运行在节点上，然后将宿主机上的容器日志目录挂载进去，最后由 logging-agent 把日志转发出去。

举个例子，我们可以通过 Fluentd 项目作为宿主机上的 logging-agent，然后把日志转发到远端的 ElasticSearch 里保存起来供将来进行检索。具体的操作过程，你可以通过阅读[这篇文档](https://kubernetes.io/docs/user-guide/logging/elasticsearch)来了解。另外，在很多 Kubernetes 的部署里，会自动为你启用 logrotate，在日志文件超过 10MB 的时候自动对日志文件进行 rotate 操作。

可以看到，在 Node 上部署 logging agent 最大的优点，在于一个节点只需要部署一个 agent，并且不会对应用和 Pod 有任何侵入性。所以，这个方案，在社区里是`最常用`的一种。

但是也不难看到，这种方案的不足之处就在于，它要求应用输出的日志，都必须是直接输出到容器的 `stdout 和 stderr` 里。

所以，Kubernetes 容器日志方案的第二种，就是对这种特殊情况的一个处理，即：当容器的日志只能输出到某些文件里的时候，我们可以通过一个 `sidecar` 容器把这些日志文件重新输出到 sidecar 的 stdout 和 stderr 上，这样就能够继续使用第一种方案了。
![](https://static001.geekbang.org/resource/image/48/20/4863e3d7d1ef02a5a44e431369ac4120.jpg?wh=2284*1610)

由于 sidecar 跟主容器之间是共享 Volume 的，所以这里的 sidecar 方案的额外性能损耗并不高，也就是多占用一点 CPU 和内存罢了。

但需要注意的是，这时候，宿主机上`实际上会存在两份相同的日志文件`：一份是应用自己写入的；另一份则是 sidecar 的 stdout 和 stderr 对应的 JSON 文件。这对磁盘是很大的浪费。

所以说，除非万不得已或者应用容器完全不可能被修改，否则我还是建议你直接使用方案一，或者直接使用下面的第三种方案。

第三种方案，就是通过一个 sidecar 容器，直接把应用的日志文件发送到远程存储里面去。也就是相当于把方案一里的 logging agent，放在了应用 Pod 里。这种方案的架构如下所示：
![](https://static001.geekbang.org/resource/image/d4/c7/d464401baec60c11f96dfeea3ae3a9c7.jpg?wh=2284*998)

在这种方案里，你的应用还可以直接把日志输出到固定的文件里而不是 stdout，你的 logging-agent 还可以使用 fluentd，后端存储还可以是 ElasticSearch。只不过， fluentd 的输入源，变成了应用的日志文件。一般来说，我们会把 fluentd 的输入源配置保存在一个 ConfigMap 里，如下所示：
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluentd.conf: |
    <source>
      type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>
    
    <source>
      type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>
    
    <match **>
      type google_cloud
    </match>
```
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    args:
    - /bin/sh
    - -c
    - >
      i=0;
      while true;
      do
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-agent
    image: k8s.gcr.io/fluentd-gcp:1.30
    env:
    - name: FLUENTD_ARGS
      value: -c /etc/fluentd-config/fluentd.conf
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /etc/fluentd-config
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
```
需要注意的是，这种方案虽然部署简单，并且对宿主机非常友好，但是这个 sidecar 容器很可能会消耗较多的资源，甚至拖垮应用容器。并且，由于日志还是没有输出到 stdout 上，所以你通过 kubectl logs 是看不到任何日志输出的。

除此之外，还有一种方式就是在**编写应用的时候，就直接指定好日志的存储后端**，如下所示：
![](https://static001.geekbang.org/resource/image/13/99/13e8439d9945fea58c9672fc4ca30799.jpg?wh=2284*705)
在这种方案下，Kubernetes 就完全不必操心容器日志的收集了，这对于本身已经有完善的日志处理系统的公司来说，是一个非常好的选择。

### 总结

Kuberentes提供了三种日志收集方案：
（1）logging agent:  pod会默认日志通过stdout/stderr输出到宿主机的一个目录，宿主机上以DaemonSet启动一个logging-agent，这个logging-agent定期将日志转存到后端。
优势： 1对Pod无侵入 2一个node只需要一个agent 3）可以通过kubectl logs查看日志
劣势： 必须将日志输出到stdout/stderr
（2 sidecar模式： pod将日志输出到一个容器目录，同pod内启动一个sidecar读取这些日志输出到stdout/stderr，后续就跟方案1）一样了。
优势：1）sidecar跟主容器是共享Volume的，所以cpu和内存损耗不大。2）可以通过kubectl logs查看日志
劣势：机器上实际存了两份日志，浪费磁盘空间，因此不建议使用
（3）sidercar模式2：pod将日志输出到一个容器文件，同pod内启动一个sidecar读取这个文件并直接转存到后端存储。
优势：部署简单，宿主机友好
劣势：1） 这个sidecar容器很可能会消耗比较多的资源，甚至拖垮应用容器。2）通过kubectl logs是看不到任何日志输出的。