# helm

## material

![](../../../../reference/pic/helm-struct.webp)

### helm安装

```shell
wget https://get.helm.sh/helm-v3.6.3-linux-amd64.tar.gz
tar -xvzf helm-v3.6.3-linux-amd64.tar.gz
mv linux-amd64/helm /bin/
chmod +x /bin/helm
```

### helm自动补全

```shell
helm completion bash > $HOME/.helm-completion.bash
echo 'source $HOME/.helm-completion.bash' >> ~/.bashrc
bash
```

### helm搜索

两种方式：
1. helm search repo (keyword) 本地repo搜索
2. helm search hub (keyword) artifact hub网络搜索

### helm命令

```shell
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install bitnami/mysql --generate-name # 给生成的release自动生成名字
helm status (release)
helm show values (release)
helm get values mysql-1629528555
helm list
helm rollback (release) (revision)
helm history (release)
helm upgrade mysql-1629528555 bitnami/mysql --set auth.rootPassword='iam59!z$'
helm dependency update
```

### helm配置数据

-f, --values：使用 YAML 文件覆盖配置。可以指定多次，优先使用最右边的文件。

--set：通过命令行的方式对指定配置项进行覆盖。

如果同时使用两种方式，则 --set 中的值会被合并到 --values 中，但是 --set 中的值优先级更高。在--set中覆盖的内容会被保存在 ConfigMap 中。你可以通过 `helm get values release-name` 来查看指定 Release 中 --set 设置的值，也可以通过运行 `helm upgrade` 并指定 --reset-values 字段，来清除 --set中设置的值。

### chart依赖关系

[ref](https://helm.sh/zh/docs/topics/charts/)

导入chart，写入依赖的chart，`helm dependency update`下载
可以通过tag和condition来控制是否使用依赖的chart，values.yaml中配置是否开启

### crd管理

CRD 文件 无法模板化，必须是普通的YAML文档

Helm会尝试加载CRD目录中 所有的 文件到Kubernetes。

不像大部分的Kubernetes对象，CRD是全局安装的。因此Helm管理CRD时会采取非常谨慎的方式。 CRD受到以下限制：

- CRD从不重新安装。 如果Helm确定crds/目录中的CRD已经存在（忽略版本），Helm不会安装或升级。
- CRD从不会在升级或回滚时安装。Helm只会在安装时创建CRD。
- CRD从不会被删除。自动删除CRD会删除集群中所有命名空间中的所有CRD内容。因此Helm不会删除CRD。

## takeaway

### helm解决的问题

一个应用包含多个服务，每个服务都有各自的配置，当在不同的环境下需要不同的配置时，传统方案很难优雅解决，只能繁琐的为每个服务编写文件并各自维护。

helm由模板+配置组成，可以灵活的根据环境进行配置

