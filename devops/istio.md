
# istio部署

## istio安装脱坑

//用istioctl安装
istioctl install --set profile=default -y
//用kubectl安装
istioctl manifest generate --set profile=default > ./deploy.yaml
kubectl apply -f deploy.yaml
//第三种方式
istioctl manifest apply --set profile=default

https://blog.csdn.net/weixin_43669903/article/details/111709545

## 之前无法安装istio可能和网络插件有关系

安装calico插件
```
#kubectl apply -f https://docs.projectcalico.org/v3.8/manifests/calico.yaml
#kubectl apply -f https://docs.projectcalico.org/v3.14/manifests/calico.yaml
curl https://docs.projectcalico.org/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```

## istio卸载

istioctl manifest generate --set profile=default | kubectl delete -f -

# istio命令

kubectl -n istio-system get deploy #检查安装了什么
istioctl dashboard --address "192.163.117" kiali # 打开kiali仪表盘

# istio学习

[ref](https://mp.weixin.qq.com/s?__biz=MzkzODUxMTY1Mg==&mid=2247485738&idx=1&sn=cb35390e3d65cd92f7ea6b4d0684b824)

为什么需要Istio，因为它提高了系统的弹性、安全性，并有助于应用程序的可观察性。Istio有两个关键概念：VirtualService和DestinationRule。VirtualService告诉Istio如何定向流量，而DestinationRule指导了在Service中的具体路径。

## VirtualService

```yml
#1. 将 95% 的流量发送到旧子集。
#2. 新子集获得 5% 的流量。
kind: VirtualService
metadata:
  name: service-b-vs
spec:
  hosts:
    - Service-B # 这里的hosts是域名或者IP等可寻址地址
  http:
    - route:
        - destination:
            host: Service-B
            subset: old
          weight: 95
        - destination:
            host: Service-B # 这里的host可以是DestinationRule声明的host，istio也会自动认别原生的service
            subset: new
          weight: 5
```

## DestinationRule

```yml
# old表示应用程序 1.0
# new表示应用程序 2.0
kind: DestinationRule
metadata:
  name: service-b-dr
spec:
  host: Service-B
  subsets:
    - name: old
      labels:
        App: "1.0"
    - name: new
      labels:
        App: "2.0"
```
通过这个配置，服务 "Service-B" 被划分为两个子集，分别是 "old" 和 "new"。每个子集都有一组标签，用于标识服务的版本或其他属性。这些标签可以在 Istio 的流量管理规则中使用，例如在 VirtualService 中进行流量路由。这种标签的使用有助于实现服务的细粒度流量控制和版本管理。

## Gateway

Gateway的配置是管理边界上独立的envoy proxy的，不管理sidecar envoy。Gateway只负责4-6层，7层转发由VirtualService负责，在VirtualService中绑定Gateway实现。

Gateway可以负责进出两种链路的流量。Gateway的selector是用来找到独立的envoy proxy的。

Gateway和Ingress资源类似，创建了之后如果有可用的LB就会分配IP

## Service Entry

用于注册网格外部的服务到istio，这样就可以配置路由到外部服务这样。

## Sidecar

控制sidecar envoy可以访问pod的哪些端口或者可以访问哪个服务的。

# 蓝绿发布 滚动发布 灰度发布 金丝雀发布 平滑更新

蓝绿发布：

蓝色测试，绿色当前使用，在蓝色测试完毕后切换，绿色销毁，再循环，为得是没有服务中断时间

滚动发布：

相比蓝绿，更节约资源，但是如果遇到意外，回滚会比较困难，并且滚动更新无法确定新版本是否可用，出现问题难以定位

灰度发布就是金丝雀发布：

相比滚动发布，灰度发布将部分流量按权重导到新版本，逐渐增大权重，如果在这时对两个版本做数据对比，就是A/B测试

**我认为灰度发布像是蓝绿发布和滚动发布的一种结合，蓝绿发布直接切换流量，而灰度是逐渐按照权重切换，滚动发布根据两个版本的数量作流量权重，而灰度发布是人为调整流量权重，且灰度可以直接知道哪些流量到达了新版本，以此进行测试，相当于多了一个流量控制器**。

原生deployment滚动发布无法做到发布新版本的时候不让流量走到新版本，不能做到只用一个版本来支撑流量的平滑发布，会存在更新的时候同时有两个版本提供服务，istio可以解决

平滑更新：

有A和B都在提供服务，先将流量都引向B，更新A，A更新了再将流量引向A，更新B，实现无宕机更新

灰度发布等都是为了确保新版本稳定可用，平滑更新不在乎这一点