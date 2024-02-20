# docker

### docker destop proxy

先试下可以自动配置clash的代理行不行，不用设置下面的配置
```
{
  "builder": {
    "gc": {
      "defaultKeepStorage": "20GB",
      "enabled": true
    }
  },
  "experimental": false,
  "proxies": {
    "http-proxy": "http://host.docker.internal:3130",
    "https-proxy": "http://host.docker.internal:3130",
    "no-proxy": "*.test.example.com,.example.org, localhost, 127.0.0.1,host.docker.internal"
  }
}
```


## docker 导入导出

### docker 打包镜像

docker save alpine | gzip > alpine-latest.tar.gz
docker load -i alpine-latest.tar.gz

一条命令形式
docker save <镜像名> | bzip2 | pv | ssh <用户名>@<主机名> 'cat | docker load'

### docker 容器导入导出

导出容器
如果要导出本地某个容器，可以使用 docker export 命令。
docker export 7691a814370e > ubuntu.tar

可以使用 docker import 从容器快照文件中再导入为镜像
cat ubuntu.tar | docker import - test/ubuntu:v1.0

>用户既可以使用 docker load 来导入镜像存储文件到本地镜像库，也可以使用 docker import 来导入一个容器快照到本地镜像库。这两者的区别在于容器快照文件将丢弃所有的历史记录和元数据信息（即仅保存容器当时的快照状态），而镜像存储文件将保存完整记录，体积也要大。此外，从容器快照文件导入时可以重新指定标签等元数据信息。

### 两者对比

export/import 操作对象：容器 导出对象：tar文件 导入对象：镜像 镜像层数：一层
save/load 操作对象：镜像 导出对象：tar文件 导入对象：镜像 镜像层数：多层

说明：
你需要把 A 机器上的 甲 容器迁移到 B 机器, 且 甲 容器中有重要的数据需要随之一起迁移的, 就可以使用 export 和 import 参数来导入和导出

【导入的镜像层数】
最大的区别就在这里, 通过export 和 import导出的容器形成镜像时, 该镜像只有一层。通过save 和 load 导出的镜像保留了原镜像所有的层次结构, 导出时原镜像有几层, 导入的时候就还是有几层

想导出容器, 但是还想保留层次结构怎么办?

这个时候就需要引入一个新的参数 commit, 用来保存容器现有的状态为一个新的镜像。
比如在 A 机器上运行的 甲 容器是基于 甲方乙方 这个镜像跑起来的, 那么我就可以通过 commit 参数, 将 甲 容器的所有内容保存为一个新的镜像, 名字叫 私人订制 (内含一梗哦) 最后我再通过镜像导出工具 save 就可以完整的将 私人订制镜像(也就是 甲容器 )导出为一个 tar 包了
而且包含了 X+1 层镜像, X 层是原镜像 甲方乙方 的所有镜像层数, 1是容器 甲 多的那一层可写层的镜像

原文链接：https://blog.csdn.net/u014078109/article/details/126243823

我的总结：
export/import 会保存容器层数据，但最终镜像只有一层
save/load 没有容器层数据
commit 会保存容器层数据，生成的镜像整体结构与原来的容器一致，但不属于导入导出操作，需要结合save来导出