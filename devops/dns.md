### 原理

[ref](https://juejin.cn/post/6844903497494855687)

zone文件配置其实就是各种dns记录，而bind配置中会写明zone配置文件的路径和其对应属性。

递归解析服务器 是负责解析域名的，例如谷歌的8.8.8.8 

权威域名服务器是负责存储域名记录的，例如baidu.com这个域的记录存储服务器

NS记录nameserver：指名一个域的权威域名解析服务器的IP地址，该服务器存放这个域的所有A、MX类型的记录

SOA record: SOA stands for "start of authority." It's an important DNS record type that stores admin information about a domain. This information includes the email address of the admin and when the domain was last updated.

SRV record: Using this DNS record type, it's possible to store the IP address and port for specific services.

`_sip._tcp.example.com. 3600	IN	SRV	10 20 5000 sip-server.example.com.`
5000是端口号 _sip是服务名，_tcp是协议 3600是TTL 10是优先级 20是权重

### unbound

zone.conf包含各类DNS解析条目，unbound.conf包含对该服务的设置，forward.conf包含stub-zone权威服务器设置和forward-zone转发服务器设置


### dig host nslookup

都不会用到/etc/hosts里面的内容