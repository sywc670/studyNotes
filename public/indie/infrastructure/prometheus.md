### prometheus operator

prometheus operator会根据prometheus、alertmanager、servicemonitor这些cr来新建对应的实例

如何给operator配置拿到相关指标？通过servicemonitor对象

#### 方案

- 利用cadvisor拿到容器指标
- 通过kubelet暴露接口拿到kubelet指标
- 通过node-exporter拿到主机指标
- 通过blackbox-exporter拿到网络指标
- 通过kube-state-metrics拿到k8s资源对象和组件指标
- 通过在软件中使用prometheus库暴露接口拿到软件自定义指标

