- [kvm](#kvm)
  - [kvm 知识](#kvm-知识)
    - [KVM QEMU 关系](#kvm-qemu-关系)
    - [KVM 内存和CPU虚拟化原理](#kvm-内存和cpu虚拟化原理)
    - [KVM 存储虚拟化](#kvm-存储虚拟化)
    - [KVM 网络虚拟化](#kvm-网络虚拟化)
    - [KVM 概念](#kvm-概念)
- [KVM 安装](#kvm-安装)
  - [查看CPU是否支持KVM虚拟化](#查看cpu是否支持kvm虚拟化)
- [kvm 使用](#kvm-使用)
  - [远程管理其他主机虚拟机配置](#远程管理其他主机虚拟机配置)
  - [kvm 操作命令](#kvm-操作命令)
    - [virt-install 启动例子](#virt-install-启动例子)
    - [virt-install](#virt-install)
      - [参数说明](#参数说明)
        - [一般选项](#一般选项)
        - [安装方法](#安装方法)
        - [存储配置](#存储配置)
        - [网络配置](#网络配置)
        - [其它常用的选项](#其它常用的选项)
        - [图形配置](#图形配置)
        - [设备选项](#设备选项)
        - [虚拟化平台](#虚拟化平台)
        - [其它](#其它)
- [hyperv](#hyperv)
  - [hyper-v 虚拟交换机网络类型](#hyper-v-虚拟交换机网络类型)
- [虚拟机迁移方式 概念](#虚拟机迁移方式-概念)
- [系统镜像](#系统镜像)
  - [镜像格式](#镜像格式)
    - [raw](#raw)
    - [qcow2](#qcow2)
    - [raw与qcow2](#raw与qcow2)
      - [truncate](#truncate)
      - [fallocate命令](#fallocate命令)
      - [文件空洞](#文件空洞)
      - [稀疏文件](#稀疏文件)
      - [比较示例](#比较示例)
  - [虚拟磁盘镜像操作](#虚拟磁盘镜像操作)
    - [查看和转换虚拟磁盘格式](#查看和转换虚拟磁盘格式)
  - [制作windows openstack镜像](#制作windows-openstack镜像)
    - [转换一个windows虚拟机注意事项](#转换一个windows虚拟机注意事项)
  - [windows镜像 Migration guide for Windows VMs from Hyper-V](#windows镜像-migration-guide-for-windows-vms-from-hyper-v)
    - [Convert the disk](#convert-the-disk)
    - [Inject the proper driver in the Virtual Machine](#inject-the-proper-driver-in-the-virtual-machine)
    - [Create the Virtual Machine in OpenStack](#create-the-virtual-machine-in-openstack)
    - [Annex: Patch Nova to accept UEFI settings on the instance](#annex-patch-nova-to-accept-uefi-settings-on-the-instance)
  - [cloudinit 定制](#cloudinit-定制)
  - [制作定制镜像工具](#制作定制镜像工具)
- [virtio 驱动](#virtio-驱动)
  - [balloon](#balloon)
- [Union文件系统](#union文件系统)
  - [aufs](#aufs)

# kvm

## kvm 知识

### KVM QEMU 关系

首先KVM（Kernel Virtual Machine）是Linux的一个内核驱动模块，它能够让Linux主机成为一个Hypervisor（虚拟机监控器）。在支持VMX（Virtual Machine Extension）功能的x86处理器中，Linux在原有的用户模式和内核模式中新增加了客户模式，并且客户模式也拥有自己的内核模式和用户模式，虚拟机就是运行在客户模式中。KVM模块的职责就是打开并初始化VMX功能，提供相应的接口以支持虚拟机的运行。

QEMU(quick emulator)本身并不包含或依赖KVM模块，而是一套由Fabrice Bellard编写的模拟计算机的自由软件。QEMU虚拟机是一个纯软件的实现，可以在没有KVM模块的情况下独立运行，但是性能比较低。QEMU有整套的虚拟机实现，包括处理器虚拟化、内存虚拟化以及I/O设备的虚拟化。QEMU是一个用户空间的进程，需要通过特定的接口才能调用到KVM模块提供的功能。从QEMU角度来看，虚拟机运行期间，QEMU通过KVM模块提供的系统调用接口进行内核设置，由KVM模块负责将虚拟机置于处理器的特殊模式运行。QEMU使用了KVM模块的虚拟化功能，为自己的虚拟机提供硬件虚拟化加速以提高虚拟机的性能。

KVM只模拟CPU和内存，因此一个客户机操作系统可以在宿主机上跑起来，但是你看不到它，无法和它沟通。于是，有人修改了QEMU代码，把他模拟CPU、内存的代码换成KVM，而网卡、显示器等留着，因此QEMU+KVM就成了一个完整的虚拟化平台。

KVM只是内核模块，用户并没法直接跟内核模块交互，需要借助用户空间的管理工具，而这个工具就是QEMU。KVM和QEMU相辅相成，QEMU通过KVM达到了硬件虚拟化的速度，而KVM则通过QEMU来模拟设备。对于KVM来说，libvirt virsh啥的都是调用qemu工作的

所以简单直接的理解就是：QEMU是个计算机模拟器，而KVM为计算机的模拟提供加速功能。

### KVM 内存和CPU虚拟化原理

一个 KVM 虚机在宿主机中其实是一个 qemu-kvm 进程，与其他 Linux 进程一样被调度 
https://www.xjimmy.com/openstack-5min-6.html

### KVM 存储虚拟化

LV 由于没有磁盘的 MBR 引导记录，不能作为虚拟机的启动盘，只能作为数据盘使用
https://www.xjimmy.com/openstack-5min-7.html

### KVM 网络虚拟化

https://www.xjimmy.com/openstack-5min-10.html

### KVM 概念

KVM 全称是 Kernel-Based Virtual Machine。也就是说 KVM 是基于 Linux 内核实现的。KVM有一个内核模块叫 kvm.ko，只用于管理虚拟 CPU 和内存。
那 IO 的虚拟化，比如存储和网络设备由谁实现呢？这个就交给 Linux 内核和Qemu来实现。
Libvirt 包含 3 个东西：后台 daemon 程序 libvirtd、API 库和命令行工具 virsh
libvirtd是服务程序，接收和处理 API 请求；
API 库使得其他人可以开发基于 Libvirt 的高级工具，比如 virt-manager，这是个图形化的 KVM 管理工具，后面我们也会介绍；
virsh 是我们经常要用的 KVM 命令行工具，后面会有使用的示例。

https://www.xjimmy.com/openstack-5min-2.html

# KVM 安装

其一：
qemu-kvm 和 qemu-system 是 KVM 和 QEMU 的核心包，提供 CPU、内存和 IO 虚拟化功能
libvirt-bin 就是 libvirt，用于管理 KVM 等 Hypervisor
virt-manager 是 KVM 图形化管理工具
bridge-utils 和 vlan，主要是网络虚拟化需要，KVM 网络虚拟化的实现是基于 linux-bridge 和 VLAN，后面我们会讨论。

https://www.xjimmy.com/openstack-5min-3.html

其二：
```
[root@Init ~]# yum -y install libvirt-daemon-kvm qemu-kvm virt-manager libvirt
#安装相应管理的软件
[root@Init ~]# modprobe kvm 
#加载KVM模块儿
[root@Init ~]# systemctl start libvirtd.service
#启动守护进程
```

## 查看CPU是否支持KVM虚拟化

egrep -o '(vmx|svm)'  /proc/cpuinfo

lsmod | grep kvm

# kvm 使用

## 远程管理其他主机虚拟机配置

https://www.xjimmy.com/openstack-5min-5.html

## kvm 操作命令

```
virsh list --all
virsh edit [name]
virsh console [name]
virsh undefine [name] # 该命令只删除配置文件，并不删除磁盘文件
virsh destroy [name] # 强制关机
virsh net-list
virsh net-edit [network-name]
virsh vncdisplay [name] # 默认端口5900 
virsh dumpxml centos > centos.xml # 导出
源文件地址：/etc/libvirt/qemu/centos7.xml
virsh define centos.xml # 导入
退出ctrl+]

virsh attach-disk centos7_mini15 /home/disk_data/centos7_mini15.img vdb 
virsh detach-disk centos7_mini15 /home/disk_data/centos7_mini15.img vdb 

virt-clone -o centos7_mini -n centos7_mini15 --auto-clone #克隆mini，新克隆的为mini15
-o #原始机名字，必须为关闭或暂停状态
-n #新客户机的名称
--auto-clone #从原始客户机配置中自动生成克隆名称和存储路径
--replace #不检查命名冲突，覆盖任何使用相同名称的客户机
-f #可以指定克隆后的主机镜像放在指定目录下
 
virsh autostart xxx #让子机随宿主机开机自动启动
virsh autostart --disable xxx #解除自动启动
```

[创建kvm不同方式](https://blog.csdn.net/taoxicun/article/details/129423200?utm_medium=distribute.pc_relevant.none-task-blog-2~default~baidujs_baidulandingword~default-5-129423200-blog-127815896.235^v27^pc_relevant_landingrelevant&spm=1001.2101.3001.4242.4&utm_relevant_index=8)

[virsh常用命令](https://www.shuzhiduo.com/A/ke5j4y7azr/)

### virt-install 启动例子

启动centos7
virt-install --connect qemu:///system --name centos7 --ram 2048 --vcpus 2 --disk path=centos7-pw.qcow2,format=qcow2 --graphics vnc,listen=0.0.0.0 --noautoconsole --import --virt-type kvm --os-variant rhel7

### virt-install

virt-install # 创建虚拟机，也可导入也可安装
如果是已经安装好了系统的镜像，使用--import
否则必须要指定启动的安装文件地址

#### 参数说明

##### 一般选项

指定虚拟机的名称、内存大小、VCPU个数及特性等；
```
​ -n NAME, --name=NAME：虚拟机名称，需全局惟一；

​ -r MEMORY, --ram=MEMORY：虚拟机内在大小，单位为MB；

​ --vcpus=VCPUS[,maxvcpus=MAX][,sockets=#][,cores=#][,threads=#]：VCPU个数及相关配置；

​ -cpu=CPU：CPU模式及特性，如coreduo等；可以使用qemu-kvm -cpu ?来获取支持的CPU模式；
```

##### 安装方法

```
指定安装方法、GuestOS类型等；

​ -c CDROM, --cdrom=CDROM：光盘安装介质；

​ -l LOCATION, --location=LOCATION：安装源URL，支持FTP、HTTP及NFS等，如ftp://172.16.0.1/pub；

​ --pxe：基于PXE完成安装；

​ --livecd: 把光盘当作LiveCD；

​ --os-type=DISTRO_TYPE：操作系统类型，如linux、unix或windows等；

​ --os-variant=DISTRO_VARIANT：某类型操作系统的变体，如rhel5、fedora8、debian10等；

osinfo-query os # 查看支持的os

​ -x EXTRA, --extra-args=EXTRA：根据–location指定的方式安装GuestOS时，用于传递给内核的额外选项，例如指定kickstart文件的位置，–extra-args “ks=http://172.16.0.1/class.cfg”

​ --boot=BOOTOPTS：指定安装过程完成后的配置选项，如指定引导设备次序、使用指定的而非安装的kernel/initrd来引导系统启动等
```
​ 例如：

​ --boot cdrom,hd,network：指定引导次序；

​ --boot kernel=KERNEL,initrd=INITRD,kernel_args=”console=/dev/ttyS0”：指定启动系统的内核及initrd文件；

##### 存储配置

存储配置包含指定存储类型、位置及属性等

–disk=DISKOPTS 指定存储设备及其属性；

格式: --disk /some/storage/path,opt1=val1，opt2=val2等；

常用的选项有：

​ device：设备类型，如cdrom、disk或floppy等，默认为disk；

​ bus：磁盘总结类型，其值可以为ide、scsi、usb、virtio或xen；

​ perms：访问权限，如rw、ro或sh（共享的可读写），默认为rw；

​ size：新建磁盘映像的大小，单位为GB；

​ cache：缓存模型，其值有none、writethrouth（缓存读）及writeback（缓存读写）；

​ format：磁盘映像格式，如raw、qcow2、vmdk等；

​ sparse：磁盘映像使用稀疏格式，即不立即分配指定大小的空间；

​ –nodisks：不使用本地磁盘，在LiveCD模式中常用；

##### 网络配置

指定网络接口的网络类型及接口属性如MAC地址、驱动模式等；

-w NETWORK，–network=NETWORK,opt1=val1,opt2=val2：将虚拟机连入宿主机的网络中，其中NETWORK可以为：

​ bridge=BRIDGE：连接至名为“BRIDEG”的桥设备；

​ network=NAME：连接至名为“NAME”的网络；

##### 其它常用的选项

​ model：GuestOS中看到的网络设备型号，如e1000、rtl8139或virtio等；

​ mac：固定的MAC地址；省略此选项时将使用随机地址，但无论何种方式，对于KVM来说，其前三段必须为52:54:00；

​ --nonetworks：虚拟机不使用网络功能；

##### 图形配置

定义虚拟机显示功能相关的配置，如VNC相关配置；

​ --graphics TYPE,opt1=val1,opt2=val2：指定图形显示相关的配置，此选项不会配置任何显示硬件（如显卡），而是仅指定虚拟机启动后对其进行访问的接口；

​ -TYPE：指定显示类型，可以为vnc、sdl、spice或none等，默认为vnc；

​ port：TYPE为vnc或spice时其监听的端口；

​ listen：TYPE为vnc或spice时所监听的IP地址，默认为127.0.0.1，可以通过修改/etc/libvirt/qemu.conf定义新的默认值；

​ password：TYPE为vnc或spice时，为远程访问监听的服务进指定认证密码；

​ --noautoconsole：禁止自动连接至虚拟机的控制台；

##### 设备选项

指定文本控制台、声音设备、串行接口、并行接口、显示接口等；

​ --serial=CHAROPTS：附加一个串行设备至当前虚拟机，根据设备类型的不同，可以使用不同的选项，格式为“–serial type,opt1=val1,opt2=val2,…”，例如：–serial pty：创建伪终端； --serial

dev,path=HOSTPATH：附加主机设备至此虚拟机；

​ --video=VIDEO：指定显卡设备模型，可用取值为cirrus、vga、qxl或vmvga；

##### 虚拟化平台

虚拟化模型（hvm或paravirt）、模拟的CPU平台类型、模拟的主机类型、hypervisor类型（如kvm、xen或qemu等）以及当前虚拟机的UUID等；

##### 其它

​ --autostart：指定虚拟机是否在物理启动后自动启动；

​ --print-xml：如果虚拟机不需要安装过程(–import、–boot)，则显示生成的XML而不是创建此虚拟机；默认情况下，此选项仍会创建磁盘映像；

​ --force：禁止命令进入交互式模式，如果有需要回答yes或no选项，则自动回答为yes；

​ --dry-run：执行创建虚拟机的整个过程，但不真正创建虚拟机、改变主机上的设备配置信息及将其创建的需求通知给libvirt；

​ -d, --debug：显示debug信息；

尽管virt-install命令有着类似上述的众多选项，但实际使用中，其必须提供的选项仅包括–name、–ram、–disk（也可是–nodisks）及安装过程相关的选项。此外，有时还需要使用括–connect=CONNCT选项来指定连接至一个非默认的hypervisor

原文链接：https://blog.csdn.net/taoxicun/article/details/129423200

# hyperv

## hyper-v 虚拟交换机网络类型

Hyper-V专用网络，如其名一样，虚拟机专用，数据的通信只能在虚拟机之间进行，虚拟机与宿主机以及物理网络之间是不能通信的。
Hyper-V的内部网络类型，只允许虚拟机与虚拟机之间，虚拟机与运行Hyper-v的主机之间进行网络通信，而不允许虚拟机与物理网络进行通信。
Hyper-V的外部网络类型，允许虚拟机与虚拟机之间，虚拟机与运行Hyper-v的主机之间，虚机与物理网络，他们互相网络通信。

https://blog.csdn.net/qq_35363270/article/details/103164460?spm=1001.2101.3001.6650.6&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-6-103164460-blog-9282755.pc_relevant_3mothn_strategy_recovery&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7EBlogCommendFromBaidu%7ERate-6-103164460-blog-9282755.pc_relevant_3mothn_strategy_recovery&utm_relevant_index=7


# 虚拟机迁移方式 概念

虚拟机迁移主要有三种方式：P2V、V2V和V2P。P2V指迁移物理服务器上的操作系统及其上的应用软件和数据到VMM（Virtual Machine Monitor）管理的虚拟服务器中。V2V迁移是在虚拟机之间移动操作系统和数据。V2P 指把一个操作系统、应用程序和数据从一个虚拟机中迁移到物理机的主硬盘上，是 P2V 的逆操作。

# 系统镜像

## 镜像格式

### raw

linux中，raw是磁盘映像的纯二进制映像格式。
dd命令可以产生

dd if=/dev/zero of=c.raw bs=1024k count=4096 //创建4G的扩容空间
cat linux.img c.raw > new-linux.img //追加到原有的镜像之后，再次查看已扩容成功

由于支持文件空洞，文件大小很大，在scp传输时，可能会消耗过多流量，可以先转换为qcow2再传输

### qcow2

Qcow2（QEMU copy-on-write）
qcow2 镜像格式是 QEMU 模拟器支持的一种磁盘镜像。它也可以用一个文件的形式来表示一块固定大小的块设备磁盘。Qcow2是目前比较主流的一种虚拟化镜像格式，目前qcow2的性能上接近raw裸格式的性能，与普通的 raw 格式的镜像相比，它还有以下特性：

更小的空间占用，即使文件系统不支持空洞(holes)；qcow2 格式的镜像比 Raw 格式文件更小，只有在虚拟机实际占用了磁盘空间时，其文件才会增长，能方便的减少迁移花费的流量，更适用于云计算系统
支持写时拷贝（COW, copy-on-write），镜像文件只反映底层磁盘的变化；
支持快照（snapshot），镜像文件能够包含多个快照的历史；
可选择基于 zlib 的压缩方式，它允许每个簇（cluster）单独使用 zlib 压缩。
可以选择 AES 加密，支持使用 128 位的 AES 密钥进行加密。

qemu-img create -f qcow2 test.qcow2 10G 
创建的10G只是上限，实际刚创建的镜像大小只有1M不到

可以通过raw来创建qcow2，backing_file是以raw为基础的，raw不能移动，如果需要转化直接用qemu-img convert
当raw格式文件中安装的系统是windows系列的时候，创建qcow2格式最后文件大小需要比raw格式大2-3倍
qemu-img create -f qcow2 -o cluster_size=2M,backing_file=win7.raw windows-7.qcow2 60G

qcow2是kvm支持的磁盘镜像格式，我们创建一个100G的qcow2磁盘之后，无论用ls来看，还是du来看，都是很小的。这说明了，qcow2本身会记录一些内部块分配的信息的。

[qemu-img命令使用等](./virtualization.md#查看和转换虚拟磁盘格式)


原文链接：https://blog.csdn.net/ximenjianxue/article/details/108061844

### raw与qcow2

https://blog.csdn.net/sssssuuuuu666/article/details/106999198/

ls看到的是文件大小
du看到的是文件占有磁盘大小，由于每个块block大小为4KB，所以会多占用空间
stat -f 能看到文件占用磁盘大小
fdisk -l 能看到每个扇区大小，通常为512B

空洞文件看起来很大，但实际占据硬盘空间不一定

#### truncate

将文件缩减或扩展至指定大小。如果指定文件超出指定大小则超出的数据将丢失。
如果指定文件小于指定大小则用0 补足。（同样形成文件空洞，不会真实文件对应得改变磁盘大小）
-s， --size=大小 。size后的数字若不指定单位则默认为字节数，也可指定KB/MB...等单位。
指定大小也可使用以下前缀修饰：
“+” 增加，"-" 减少，"<" 至多，">" 至少，
“/” 小于等于原尺寸数字的指定数字的最小倍数，"%" 大于等于原尺寸数字的指定数字的最大倍数。

用truncate扩展后，ls -l会变大，du不会变化
```
使用seek跳过输出文件的一定块=乘上bs，被ls包含进文件大小
[root@suhw ~]# dd if=/dev/zero of=dd_seek count=1 bs=4M seek=1000
1+0 records in
1+0 records out
4194304 bytes (4.2 MB) copied, 0.0115562 s, 363 MB/s
[root@suhw ~]# ls -lh dd_seek
-rw-r--r-- 1 root root 4.0G Jun 28 11:09 dd_seek
[root@suhw ~]# du -h dd_seek
4.0M    dd_seek
```

#### fallocate命令

可以为文件预分配物理空间。-l后接空间大小，默认单位为字节。也可后跟k、m、g、t、p、e来指定单位，分别代表KB、MB、GB、TB、PB、EB。
```
[root@suhw ~]# fallocate -l 1G test_file

[root@csmp-standalone ~]# ll
-rw-r--r--  1 root root 1073741824 Jun 24 21:41 test_file

# 查看文件占用磁盘大小
[root@csmp-standalone ~]# du -h test_file
1.1G    test_file
```

#### 文件空洞

dd if=/dev/zero of=test bs=1 count=0 seek=4M # 创建空洞文件

文件空洞作用
迅雷下载文件时，在未下载完成时就已经占据了全部文件大小的空间，这时候就是空洞文件。下载的时候如果没有空洞文件，多线程下载时文件就都只能从一个地方写入，此时就会出问题。如果有了空洞文件，可以从不同的地址写入，就完成了多线程的优势任务。
在创建虚拟机的时候，我们会使用img工具生成一个例如50GB大小的镜像文件，但是其实在安装完系统之后，镜像的大小可能只有4GB，也就是说img并不会马上就占用掉物理存储空间的50GB，而是在未来使用过程中不断增加的。

#### 稀疏文件

稀疏文件，这是UNIX类和NTFS等文件系统的一个特性。稀疏文件就是在文件中留有很多空余空间，留备将来插入数据使用。如果这些空余空间被ASCII码的NULL字符占据，并且这些空间相当大，那么，这个文件就被称为稀疏文件，而且，这些多余得空间并不分配对应的磁盘块。
![稀疏文件示意图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9naXRlZS5jb20vc3VoYW93L25vdGUvcmF3L21hc3Rlci9pbWcvMjAyMDA2MjQxNTI0MDcucG5n?x-oss-process=image/format,png)

#### 比较示例

```
[root@suhw ~]# qemu-img create -f qcow2 qcow2-image.qcow2 2G
Formatting 'qcow2-image.qcow2', fmt=qcow2 size=2147483648 cluster_size=65536 lazy_refcounts=off refcount_bits=16

[root@suhw ~]# qemu-img create -f raw raw-image.raw 2G
Formatting 'qcow2-image.raw', fmt=raw size=2147483648

[root@suhw ~]# ls -lh *-image*
-rw-r--r-- 1 root root 193K Jun 28 02:14 qcow2-image.qcow2
-rw-r--r-- 1 root root 2.0G Jun 28 02:14 raw-image.raw

[root@suhw ~]# du -h *-image*
196K    qcow2-image.qcow2
0       raw-image.raw


[root@suhw ~]# stat qcow2-image.qcow2
  File: 'qcow2-image.qcow2'
  Size: 196640          Blocks: 392        IO Block: 4096   regular file
...

[root@suhw ~]# stat raw-image.raw
  File: 'raw-image.raw'
  Size: 2147483648      Blocks: 0          IO Block: 4096   regular file
...
```

## 虚拟磁盘镜像操作

使用前确认镜像已经关闭
yum install libguestfs-tools

truncate 扩大缩小文件
virt-format 格式化、改变文件系统等
 truncate -s 10G target.img
 virt-format -a target.img --partition=mbr --lvm --filesystem=ext3
guestfish shell交互
guestmount 将虚拟磁盘挂载到主机上再调整

virt-edit for editing a file inside of an image.
virt-df for displaying free space inside of an image.
virt-resize for resizing an image.
virt-sysprep for preparing an image for distribution (for example, delete SSH host keys, remove MAC address info, or remove user accounts).
virt-sparsify for making an image sparse.
virt-p2v for converting a physical machine to an image that runs on KVM.
virt-v2v for converting Xen and VMware images to KVM images.

virt-customize -a image_name.qcow2 --root-password password:yourpassword # 修改镜像账号密码
报错尝试export LIBGUESTFS_BACKEND=direct

https://docs.openstack.org/image-guide/modify-images.html
https://libguestfs.org/guestfs-recipes.1.html
https://libguestfs.org/

### 查看和转换虚拟磁盘格式

```
qemu-img 
qemu-img allows you to create, convert and modify images offline. It can handle all image formats supported by QEMU.

qemu-img info test.qcow2 # 查看镜像信息，包括原始大小和实际大小

qemu-img check -f 磁盘格式 磁盘镜像文件的名称 //对磁盘镜像文件进行一致性检查

qemu-img commit [-f fmt] 虚拟磁盘；用来提交虚拟磁盘文件中的更改到后端支持镜像文件（创建时通过backing_file指定的）中去

qemu-img snapshot [-l | -a snapshot | -c snapshot | -d snapshot] filename
其中，“-l” 选项是查询并列出镜像文件中的所有快照，“-a snapshot”是让镜像文件使用某个快照，“-c snapshot”是创建一个快照，“-d”是删除一个快照。

qemu-img resize filename [+ | -]size //使其不同于创建之时的大小
qemu-img resize test.qcow2 +2G # 只更改上限，不更改实际大小
```

https://docs.openstack.org/image-guide/convert-images.html



## 制作windows openstack镜像
准备windows iso， 下载virtio工具，通过tigervnc连接

下载virtio工具地址：https://github.com/virtio-win/virtio-win-pkg-scripts/blob/master/README.md
注意版本可能不支持windows系统版本

```
yum groupinstall Virtualization "Virtualization Client"
yum install libvirt
yum install libguestfs-tools
systemctl enable libvirtd
systemctl start libvirtd
systemctl status libvirtd

qemu-img create -f qcow2 ws2022.qcow2 15G
virt-install --connect qemu:///system --name ws2022 --ram 4048 --vcpus 3 --network network=default,model=virtio --disk path=ws2022.qcow2,format=qcow2,device=disk,bus=virtio --disk zh-cn_windows_server_2022_updated_aug_2022_x64_dvd_41a50abf.iso,device=cdrom,bus=ide --disk path=virtio-win-0.1.229.iso,device=cdrom --os-type windows --graphics vnc,listen=0.0.0.0 --noautoconsole
#virt-install --connect qemu:///system --name ws10 --ram 4048 --vcpus 3 --network network=default,model=virtio --disk path=ws10.qcow2,format=qcow2,device=disk,bus=virtio --disk Win10_21H2_x64.iso,device=cdrom,bus=ide --disk path=virtio-win-0.1.126.iso,device=cdrom --os-type windows --os-variant win10 --graphics vnc,listen=0.0.0.0 --noautoconsole
添加驱动，可以使用pnputil命令行添加
ss -nlutp | grep qemu-kvm
virt-sparsify --compress ws2022.qcow2 ws2022-final.qcow2 # 使用之后可以在virt-install中标注--disk path=rhel5.8.img,size=120,bus=virtio,sparse
openstack image create --file /root/Windows-Server-2012r2.qcow2 --disk-format qcow2 --container-format bare ws2012
```

https://docs.openstack.org/image-guide/windows-image.html
https://blog.csdn.net/qq_32262243/article/details/127731721
https://www.shuzhiduo.com/A/ZOJPN9Bldv/

### 转换一个windows虚拟机注意事项

When converting an image file with Windows, ensure the virtio driver is installed. Otherwise, you will get a blue screen when launching the image due to lack of the virtio driver. Another option is to set the image properties as below when you update the image in the Image service to avoid this issue, but it will reduce virtual machine performance significantly.

openstack image set --property hw_disk_bus='ide' image_name_or_id

## windows镜像 Migration guide for Windows VMs from Hyper-V

First get the virtual disk (vhdx) from the Hyper-V platform.

Use a Linux machine with libvirt installed.

Do not forget to install the UEFI firmware:
```console
sudo apt install ovmf
```

### Convert the disk

We need to convert the disk to the native format used by OpenStack:
```console
qemu-img check -r all windows-disk.vhdx
qemu-img convert -O qcow2 windows-disk.vhdx windows-disk.qcow2
```

### Inject the proper driver in the Virtual Machine

OpenStack (and KVM) use para-virtualized (virtio) drivers for the disk and the network. Although most modern Linux distributions have these drivers pre-loaded, Windows does not.

The best way to inject the drivers will be to do the following:

Get the virtio drivers for Windows:
```console
curl -LO https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso
```

Create a dummy disk that will make Windows detect the virtio disk driver:
```console
qemu-img create -f qcow2 dummy.qcow2 5G
```

Create a temporary Virtual Machine:
```console
virt-install --connect qemu:///system \
  --ram 2048 \
  --vcpus 2 --cpu host \
  --network network=default,model=virtio \
  --disk path=windows-disk.qcow2,device=disk,format=qcow2,bus=ide \
  --disk path=dummy.qcow2,format=qcow2,device=disk,bus=virtio \
  --disk path=./iso/virtio-win.iso,device=cdrom,perms=ro \
  --graphics vnc,listen=0.0.0.0,password=1234 --noautoconsole \
  --boot uefi \
  --os-type windows --os-variant win2k12 \
  --import \
  --name myvm
```

**Note:** if the virtual disk is not an UEFI image, the `--boot uefi` should not be used.

Once the Virtual Machine has booted, log in and install the drivers:
```console
d:
cd \balloon\2k12R2\amd64
pnputil -i -a *.inf
cd \NetKVM\2k12R2\amd64
pnputil -i -a *.inf
cd \viostor\2k12R2\amd64
pnputil -i -a *.inf
```

Then, stop the virtual machine from inside (Powershell: `Stop-Computer`).  
And undefine it in KVM:
```console
virsh undefine --nvram myvm
```

Finally, start the Virtual Machine again with the virtio bus:
```console
virt-install --connect qemu:///system \
  --ram 2048 \
  --vcpus 2 --cpu host \
  --network network=default,model=virtio \
  --disk path=windows-disk.qcow2,device=disk,format=qcow2,bus=virtio \
  --graphics vnc,listen=0.0.0.0,password=1234 --noautoconsole \
  --boot uefi \
  --os-type windows --os-variant win2k12 \
  --import \
  --name myvm
```

**Note:** if the virtual disk is not an UEFI image, the `--boot uefi` should not be used.

Check the Virtual Machine to see if it is working fine.

### Create the Virtual Machine in OpenStack

First Import the virtual disk into an Openstack Image:
```console
openstack image create \
  --container-format bare \
  --disk-format qcow2  \
  --file windows-disk.qcow2 \
  myvm-disk
```

And, create a volume from that image:
```console
openstack volume create \
  --image myvm-disk \
  --size 60G \
  myvm-disk
```

Check the progress with the Horizon web server or with: `openstack volume list`. Once the volume is ready, you can start the instance:
```console
openstack server create
  --volume myvm-disk \
  --flavor w1.small \
  --security-group xxx \
  --property hw_firmware_type=uefi \
  myvm
```

**Notes:**
- if the virtual disk is not an UEFI image, do not add the propery `hw_firmware_type=uefi`.
- Don't forget to choose the appropriate security group!

### Annex: Patch Nova to accept UEFI settings on the instance

Unfortunately, nova reads the UEFI settings only from the image of the instance. And, in our case, there is no base image as the instance is directly attached to a volume.

There might be ways to read the image from the volume, but I have not found it in nova's source code.

The best was to patch the source code to read the firmware type from the instance metadata.

Here is the patch to apply in `/usr/lib/python2.7/dist-packages/nova/virt/libvirt`:

```diff
--- driver.py   2020-10-08 19:45:10.847239335 +0900
+++ driver.py.new       2020-10-08 19:45:18.935288598 +0900
@@ -4895,6 +4895,8 @@
                 guest.sysinfo = self._get_guest_config_sysinfo(instance)
                 guest.os_smbios = vconfig.LibvirtConfigGuestSMBIOS()
             hw_firmware_type = image_meta.properties.get('hw_firmware_type')
+            if not hw_firmware_type:
+                hw_firmware_type = instance.get('metadata').get('hw_firmware_type')
             if caps.host.cpu.arch == fields.Architecture.AARCH64:
                 if not hw_firmware_type:
                     hw_firmware_type = fields.FirmwareType.UEFI
```

You apply it like this:
```console
cd /usr/lib/python2.7/dist-packages/nova/virt/libvirt
sudo patch driver.py < driver.py.patch
sudo python -m compileall driver.py
sudo systemctl restart nova-compute.service
```

## cloudinit 定制

instance 每次启动 cloud-init 都会执行初始化工作，如果希望改变所有 instance 的初始化行为，则修改镜像的/etc/cloud/cloud.cfg 文件；如果只想改变某个 instance 的初始化行为，直接修改 instance 的 /etc/cloud/cloud.cfg。

## 制作定制镜像工具

https://docs.openstack.org/image-guide/create-images-automatically.html

# virtio 驱动

## balloon

balloon driver是一种驱动程序，可以从客户机汲取内存或追添内存给予客户机。从理论上，如果你的客户机需要更多的内存，你可以使用balloonDriver给客户机提供更多内存;如果主机需要从客户机汲取内存，balloonDriver也可以做到。无论是给客户机提供更多内存还是主机从客户机汲取内存都不需要暂停或者重启客户机，完全可以动态实现，相当于实现客户机内存的热插拔(hot-plugin)。

https://www.cnblogs.com/oyym/p/3485337.html

# Union文件系统

## aufs

https://blog.csdn.net/weixin_42645653/article/details/119982032