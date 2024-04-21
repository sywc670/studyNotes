# coreDNS coredns

[ref](https://coredns.io/manual/toc/)

Once CoreDNS has been started and has parsed the configuration, it runs Servers. Each Server is defined by the zones it serves and on what port. Each Server has its own Plugin Chain.

The ordering of the Plugins in the Corefile does not determine the order of the plugin chain. The order in which the plugins are executed is determined by the ordering in `plugin.cfg`.

Note that if you use the `bind` plugin you can have the same zone listening on the same port, provided they are binded to different interfaces or IP addresses. 

```
.:1054 {
    bind lo
    whoami
}

.:1054 {
    bind eth0
    whoami
}
```

Currently CoreDNS accepts four different protocols: DNS, DNS over TLS (DoT), DNS over HTTP/2 (DoH) and DNS over gRPC. You can specify what a server should accept in the server configuration by prefixing a zone name with a scheme.

That line in the `health` plugin’s documentation means that once health is specified, it is global for the entire CoreDNS process, even though you’ve only specified it for one server.

### kubedns 与 coreDNS

[ref](https://zhuanlan.zhihu.com/p/80141656?utm_id=0)

kubedns由dnsmasq实现