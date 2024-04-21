
# ceph 安装

## ceph 安装教程

https://blog.csdn.net/dandanfengyun/article/details/106235667

https://www.cnblogs.com/cloudhere/p/10519647.html

## ceph与openstack集成教程

https://www.cnblogs.com/sammyliu/p/4804037.html
https://docs.ceph.com/en/latest/rbd/rbd-openstack/

## ceph-deploy 不完善的安装流程

注意主机名必须匹配、时间同步
```
vim /etc/yum.repos.d/ceph.repo
[ceph]
name=Ceph packages for $basearch
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/$basearch
enabled=1
priority=1
gpgcheck=0
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-noarch]
name=Ceph noarch packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/noarch
enabled=1
priority=1
gpgcheck=0
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

[ceph-source]
name=Ceph source packages
baseurl=http://mirrors.aliyun.com/ceph/rpm-nautilus/el7/SRPMS
enabled=1
priority=1
gpgcheck=0
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc

yum install yum-plugin-priorities
yum -y install ceph-deploy
mkdir /ceph
cd /ceph
ceph-deploy install --release nautilus --no-adjust-repos ceph-1 ceph-2 
ceph-deploy new ceph-1
ceph-deploy mon create-initial
ceph-deploy mgr create ceph-1
ceph-deploy admin ceph-1

ceph-deploy disk zap ceph-1 /dev/sdb
ceph-deploy disk zap ceph-2 /dev/sdb

ceph-deploy osd create --data /dev/sdb ceph-1
ceph-deploy osd create --data /dev/sdb ceph-2


```
https://blog.csdn.net/dandanfengyun/article/details/112917279?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-3-112917279-blog-120721817.pc_relevant_aa2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-3-112917279-blog-120721817.pc_relevant_aa2&utm_relevant_index=6

# ceph 信息

## ceph结构

![结构](https://docs.ceph.com/en/quincy/_images/stack.png)

## ceph日志

/var/log/ceph/ceph.log 集群日志位置

# ceph 操作

## ceph 创建 pool

ceph osd pool create images 32
ceph osd pool create vms 32
ceph osd pool create volumes 32
ceph osd pool create backups 32

https://blog.csdn.net/dandanfengyun/article/details/112917279?spm=1001.2101.3001.6650.3&utm_medium=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-3-112917279-blog-120721817.pc_relevant_aa2&depth_1-utm_source=distribute.pc_relevant.none-task-blog-2%7Edefault%7ECTRLIST%7ERate-3-112917279-blog-120721817.pc_relevant_aa2&utm_relevant_index=6

## ceph 删除 pool 

```
打开mon节点的配置文件：
[root@node1 ceph]# vi /etc/ceph/ceph.conf
在配置文件中添加如下内容：
[mon]
mon_allow_pool_delete = true
重启ceph-mon服务：
[root@node1 ceph]# systemctl restart ceph-mon.target
执行删除pool命令：
[root@node3 ~]# ceph osd pool delete ecpool ecpool –yes-i-really-really-mean-it
pool ‘ecpool’ removed
```
————————————————
https://blog.csdn.net/qq_32595453/article/details/82558226

# 报错

## ceph故障指南
https://blog.csdn.net/qq_33218245/article/details/108604504

## ceph application not enabled on 2 pool(s)
ceph osd pool application enable images rbd
ceph osd pool application enable vms rbd
ceph osd pool application enable volumes rbd
ceph osd pool application enable backups rbd
原因
Pools need to be associated with an application before use. Pools that will be used with CephFS or pools that are automatically created by RGW are automatically associated. Pools that are intended for use with RBD should be initialized using the rbd tool (see Block Device Commands for more information).

For other cases, you can manually associate a free-form application name to a pool.:

ceph osd pool application enable {pool-name} {application-name}

CephFS uses the application name cephfs, RBD uses the application name rbd, and RGW uses the application name rgw.

https://docs.ceph.com/en/latest/rados/operations/pools/#id1

## ceph安装 monitor is not yet in quorum
在安装节点上设置对应hostname，并pgrep ceph 查看有的话pkill ceph

# ceph 操作命令
ceph 常用命令

https://www.cnblogs.com/cloudhere/p/10664965.html
## 情况
```
rados df
rados lspools
ceph -s
ceph df
rbd ls [osd pool] # To list block devices in a particular pool
rbd info {pool-name}/{image-name}
ceph osd dump
ceph mon dump
ceph mon stat
ceph pg dump
ceph osd tree # You can also check view OSDs according to their position in the CRUSH map 
ceph daemon /var/run/ceph/ceph-mon.$(hostname -s).asok config show # 查看ceph集群配置信息
```
## 日志

```
ceph log last [ n ]
ceph -w
```

## 认证
ceph auth list

## 镜像
rbd rm -p images imagename
rbd import /root/test2.raw -p volumes --image test-import 
```shell
# 批量删除,有快照等情况删除不完全
POOL=volumes
VOLUMES=$(rbd ls -p $POOL)
for volume in $VOLUMES:
do
  echo "removing $POOL/$volume"
  rbd rm -p $POOL $volume
done 
```

# ceph 概念

RADOS Cluster
A proper subset of the Ceph Cluster consisting of OSDs, Ceph Monitors, and Ceph Managers.

cephfs
The Ceph File System, or CephFS, is a POSIX-compliant file system built on top of Ceph’s distributed object store, RADOS

RBD
The block storage component of Ceph. Also called “RADOS Block Device” or Ceph Block Device.
rbd is a utility for manipulating rados block device (RBD) images, used by the Linux rbd driver and the rbd storage driver for QEMU/KVM. RBD images are simple block devices that are striped over objects and stored in a RADOS object store. 

RGW
RADOS Gate Way.
The component of Ceph that provides a gateway to both the Amazon S3 RESTful API and the OpenStack Swift API. Also called “RADOS Gateway” and “Ceph Object Gateway”.

# ceph rook示例

```yaml
apiVersion: ceph.rook.io/v1beta1
kind: Pool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: ceph.rook.io/block
parameters:
  pool: replicapool
  clusterNamespace: rook-ceph
```
在这里，我用到了 StorageClass 来完成这个操作。它的作用，是自动地为集群里存在的每一个 PVC，调用存储插件（Rook）创建对应的 PV，从而省去了我们手动创建 PV 的机械劳动。我在后续讲解容器存储的时候，会再详细介绍这个机制。

备注：在使用 Rook 的情况下，mysql-statefulset.yaml 里的 volumeClaimTemplates 字段需要加上声明 storageClassName=rook-ceph-block，才能使用到这个 Rook 提供的持久化存储。

https://time.geekbang.org/column/article/41217

# rook

## rook 部署

```
git clone --single-branch --branch v1.11.4 https://github.com/rook/rook.git

cd rook/deploy/examples
kubectl create -f crds.yaml -f common.yaml -f operator.yaml
kubectl create -f cluster.yaml

# 必须要三个节点才可以部署cluster.yaml
否则改用cluster-test.yaml
```

每个节点至少需要一块裸盘

## rook 使用

设置sc为默认
`kubectl patch storageclass rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'`

```yml
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: replicapool
  namespace: rook-ceph
spec:
  failureDomain: host
  replicated:
    size: 3
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: replicapool
  imageFormat: "2"
  imageFeatures: layering

  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph

  csi.storage.k8s.io/fstype: ext4
reclaimPolicy: Delete
```

toolbox
kubectl create -f deploy/examples/toolbox.yaml
kubectl -n rook-ceph rollout status deploy/rook-ceph-tools
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash


# cephadm 部署

注意点：部署可以设置ceph.conf为可以删除pool

curl --silent --remote-name --location https://github.com/ceph/ceph/raw/quincy/src/cephadm/cephadm
chmod +x cephadm
./cephadm add-repo --release quincy
./cephadm install
cephadm bootstrap --mon-ip mon-ip
ssh-copy-id -f -i /etc/ceph/ceph.pub root@host2
ceph orch daemon add osd ceph1:/dev/nvme0n2

## 报错

```
GPT headers found, they must be removed on: /dev/sda

apt install gdisk -y
sgdisk --zap-all /dev/sdX
```


## 使用命令

daemon是host上的mon\mgr等
service是同类型daemon在各host的集合

ceph orch host ls
ceph orch host ls --detail
ceph orch ls
ceph orch restart mon
ceph orch restart mgr
ceph orch daemon restart mon.host

ceph config set mon.{id} mon_allow_pool_delete true
ceph config set client.{id} parameter value
ceph config set global parameter value

ceph.conf配置
https://lists.ceph.io/hyperkitty/list/ceph-users@ceph.io/thread/E6RRHADNJHQ7MN2AJFHMRFI3USQRRUOH/