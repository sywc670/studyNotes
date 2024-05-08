# client go


RESTClient，封装了http.Client，实际请求用了Request这个结构体来做

Request在发送请求时调用**http.NewRequestWithContext**，并添加了trace功能，调用**httptrace**，这里用到了这个标准库的trace功能，在做请求的过程中可以定义hook触发动作，进行trace

DiscoveryClient，拿到对象的列表，这个接口apiserver直接提供了，所以构建下GroupVersion就能拿到，所以就并发去拿到然后聚合在一起就行

总结一下就是，CachedDiscoveryClient 首先从本地磁盘文件中取缓存的资源组信息，如果取不到或者过期了，就去 memCacheClient 的内存中取，内存中也取不到或者被刷新了，就调用 DiscoveryClient 向 kube-apiserver 发起请求获取。