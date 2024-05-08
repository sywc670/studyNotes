### RIB FIB ARP FDB 区分

1. RIB与FIB的区别：

RIB：`路由表`

FIB：`转发信息表`

FIB表更多是出现在需要快速转发的路由器上，这种路由器上的路由表条目通常都达成千上万条，如果按照传统的检索路由表进行转发的方式，其转发效率很低，FIB表作为路由表的一种精简形式出现，通常只记录常用的表项。当需要选路时，先检索FIB表，如果找不到再检索路由表。

在大部分路由器中，RIB表现为路由表的形式， FIB则表现为高速缓存的形式，此在内容上是路由表的一个子集，是依靠路由表来生成的。

一般来说，FIB是进行高速查找而组织的数据结构（不是简单的把路由表中的内容复制出来，数据存储和检索方式等都不同于路由表的组成像是）。

RIB 就一个字：全，知道到所有的地方怎么走，但是速度慢。

FIB就一个字：快，只知道常走的路怎么走，速度快。

如果是分布式设备，通常FIB分布在LPU上，由LPU上的CPU实现快速选路，如果在LPU找不到路，才上到MPU处理，这里的RIB保存了最全的路由信息，可以提供不常用的选路结果。

2. ARP表和FDB表的区别：

ARP表：`IP和MAC的对应关系`；

FDB表：`MAC+VLAN和PORT的对应关系`；

两个最大的区别在于ARP是`三层转发`，FDB是用于`二层转发`。也就是说，就算两个设备不在一个网段或者压根没配IP，只要两者之间的链路层是连通的，就可以通过FDB表进行数据的转发！

FDB表的最主要的作用就是在于交换机二层选路，试想，如果仅仅有ARP表，没有FDB表，就好像只知道地名和方位，而不知道经过哪条路才能到达目的地，设备是无法正常工作的。FDB表的作用就在于告诉设备从某个端口出去就可以到某个目的MAC。

那么FDB表是怎么形成的呢？很简单，交换机会在收到数据帧时，提取数据帧中的源MAC、VLAN和接收数据帧的端口等组成FDB表的条目。当下次有到该VLAN中的该MAC的报文就直接从该端口丢出去就OK了。

当然，FDB表和ARP表一样，都有一个老化时间。

原文链接：https://blog.csdn.net/qq_33706673/article/details/84525393

### websocket

websocket协议主要用于服务器向客户端主动发送请求的场景

TCP是全双工的，但是HTTP是`半双工`的，设计时没有考虑双方互相主动发大量信息的情况。

所有websocket连接建立前都要进行http通信，然后切换成websocket。

### socket

socket在linux上是文件形式，有uds，是本机进程交流用的，也有网络交流的。根据fd找到socket文件，并利用linux内核的网络能力来进行通信，socket就是存放网络通信的相关信息的数据结构。


### AD域

[AD域控系列博客](https://www.cnblogs.com/itzgr/p/15554185.html)

[ref](https://zhuanlan.zhihu.com/p/45553448)

[阿里云创建域控加入](https://help.aliyun.com/zh/ecs/use-cases/ecs-instance-building-windows-active-directory-domain)

[官方文档](https://learn.microsoft.com/zh-cn/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)

AD认证和ldap都可以做到一个账户登录多个机器，AD使用到了ldap

#### 域的含义

域(Domain)是Windows网络中独立运行的单位，域之间相互访问则需要建立**信任关系**。信任关系是连接在域与域之间的桥梁。当一个域与其他域建立了信任关系后，2个域之间不但可以按需要相互进行管理，还可以跨网分配文件和打印机等设备资源，使不同的域之间实现网络资源的共享与管理，以及相互通信和数据传输。

域既是 Windows 网络操作系统的逻辑组织单元，也是Internet的逻辑组织单元，在 Windows 网络操作系统中，域是安全边界。域管理员只能管理域的内部，除非其他的域显式地赋予他管理权限，他才能够访问或者管理其他的域，每个域都有自己的安全策略，以及它与其他域的安全信任关系。

当企业网络中计算机和用户数量较多时，为了实现高效管理，就需要windows域。

#### 域的原理

在工作组上你一切的设置在本机上进行包括各种策略，用户登录也是登录在本机的，密码是放在本机的数据库来验证的。而如果你的计算机加入域的话，各种策略是域控制器统一设定，用户名和密码也是放到域控制器去验证，也就是说你的账号密码可以在同一域的任何一台计算机登录。

在“域”模式下，至少有一台服务器负责每一台联入网络的电脑和用户的验证工作，相当于一个单位的门卫一样，称为“**域控制器**（Domain Controller，简写为DC）”。

域控制器中包含了由这个域的账户、密码、属于这个域的计算机等信息构成的**数据库**。当电脑联入网络时，域控制器首先要鉴别这台电脑是否是属于这个域的，用户使用的登录账号是否存在、密码是否正确。如果以上信息有一样不正确，那么域控制器就会拒绝这个用户从这台电脑登录。不能登录，用户就不能访问服务器上有权限保护的资源，他只能以对等网用户的方式访问Windows共享出来的资源，这样就在一定程度上保护了网络上的资源。

要把一台电脑加入域，仅仅使它和服务器在网上邻居中能够相互“看”到是远远不够的，必须要由网络管理员进行相应的设置，把这台电脑加入到域中。这样才能实现文件的共享，集中统一，便于管理。

域是一个有安全边界的计算机集合，在同一个域中的计算机彼此之间已经建立了信任关系，**在域内访问其他机器，不再需要被访问机器的许可了**。为什么是这样的呢？因为在加入域的时候，管理员为每个计算机在域中（可和用户不在同一域中）建立了一个计算机帐户，这个帐户和用户帐户一样，也有密码（登录凭证，由2000的DC（域控制器）上的KDC服务来颁发和维护）保护的。

AD(active directory)**活动目录**,动态的建立整个域模式网络中的对象的数据库或索引,**使用了LDAP协议**，安装了AD的服务器称为DC域控制器,存储整个域的对象的信息并周期性更新。

一个域内可以有多台域控制器，每台域控制器的地位几乎是平等的，它们各自存储着一份几乎完全相同的活动目录（ Active Directory）；当在任何一台域控制器内添加了一个用户账户后，此账户默认被创建在此域控制器的活动目录，之后会自动被复制到其他域控制器的活动目录，以便让所有域控制器内的活动数据都能够同步。

#### 活动目录(AD)与域名系统(DNS)的关系

在TCP/IP网络中，域名系统(DNS)是用来解决计算机名字和IP地址的映射关系的，活动目录和DNS是紧密不可分的，活动目录使用DNS服务器来登记域控制器的IP、各种资源的定位等，在一个域林中至少要有一个DNS服务器存在，所以安装活动目录时需要同时安装DNS。此外，域的命名也是采用DNS的格式来命名的。

#### 林

林上可以有多个域，当建立第一个域时就需要新建林，林的作用是可以管理多个域



#### 域与组

在工作组模式下，网络中的用户、资源和权限这些内容都是由每台计算机的使用者自行管理的。域不同。

#### DN

标识名称（distinguished Name，DN）：它是对象在活动目录内的完整路径，DN 有三个属性，分别是 CN，OU，DC。OU和DC可以有多个。

DC (Domain Component)：域名组件，表示 DNS 域名中的组件（如：coffeeMilk.com中的`[coffee]`和`[com]`）；

OU (Organizational Unit)：组织单位；OU就是分组。

CN (Common Name)：通用名称，一般为用户名或计算机名；

### LDAP

LDAP（Light Directory Access Portocol），它是基于X.500标准的轻量级目录访问协议。

目录是一个为查询、浏览和搜索而优化的**数据库**，它成树状结构组织数据，类似文件目录一样。

目录数据库和关系数据库不同，它**有优异的读性能，但写性能差**，并且没有事务处理、回滚等复杂功能，不适于存储修改频繁的数据。所以目录天生是用来查询的，就好象它的名字一样。

LDAP目录服务是由目录数据库和一套访问协议组成的系统。

[LDAP服务端客户端搭建-未记录](https://blog.csdn.net/Eng_ingLi/article/details/131574784)

### /etc/nsswitch.conf

/etc/nsswitch.conf文件的配置语法为“database: source”，其中database表示要检索的数据库类型，source表示检索顺序。源可以是“files”（从文件中检索）、“dns”（从DNS服务器中检索）、“nis”（从NIS服务器中检索）等。

### NIS

NIS是一种用于跨网络进行身份验证和用户信息管理的服务，最初由Sun Microsystems开发。它基于客户端-服务器架构，使用了关系数据库来存储用户信息。NIS使用了传统的基于文本文件的配置，包括/etc/passwd、/etc/group和/etc/hosts等。

NIS主要用于UNIX/Linux系统，特别是传统的基于文本文件的配置。它在网络中传播用户和组信息，并提供了集中管理的用户身份验证。它适用于小型、简单的网络环境，但随着网络规模的增长，性能和安全性可能会成为问题。

### CDN

CDN技术原理是在各地建立缓存服务器，分担源服务器的流量，加速本地访问速度。

CDN的访问过程依赖于**DNS的重定向技术**，即将用户定向至地理位置上距离其最近的边缘CDN节点服务器上。用户首先向根DNS服务器发送域名解析请求，根DNS服务器向授权DNS服务器发送域名解析请求，请求中包含了根服务器的IP地址，当域名解析服务器/根DNS服务器接受到一个CNAME类的DNS记录，域名解析服务器会重定向到CDN节点网络层中的**智能CDN域名服务器**上，CDN域名服务器将进行一系列的智能解析操作，根据**本地DNS域名解析服务器的IP地址**，分析各个网络线路的拥堵情况和负载情况，将最适合的CDN节点服务器IP地址返还给根DNS服务器，用户接受到CDN节点的IP地址后，直接向CDN节点服务器发送请求获取网站内容。

#### 回源跟随

CDN回源跟随（Follow Redirect）是一种内容分发网络（CDN）的功能，它允许CDN节点在处理源站（即原始服务器）返回的301或302状态码（HTTP重定向响应）时，自动跟随重定向到新的URL地址去获取资源。这样做的目的是为了确保用户能够获取到最新的资源，同时保持资源的一致性和可用性。

在没有开启回源跟随功能的情况下，当CDN节点收到源站的301或302响应时，它会将这个响应直接传递给用户，由用户的浏览器或客户端去处理重定向，这可能会导致用户体验上的延迟，因为用户需要额外的请求来获取资源。

开启回源跟随功能后，CDN节点会代替用户处理这些重定向，直接从新的URL地址获取资源，并将获取到的资源缓存起来。这样，当其他用户请求相同的资源时，CDN可以直接从缓存中提供资源，从而提高了资源的加载速度和用户体验。

这个功能在源站地址发生变化，但希望用户仍然能够通过原来的URL访问资源时特别有用。例如，如果源站的某个文件被移动到了新的路径，通过配置回源跟随，CDN可以确保用户访问旧的URL时仍然能够获取到最新的资源。

### 跨域CORS

[ref](https://web.dev/same-site-same-origin/)

[ref](https://www.ruanyifeng.com/blog/2016/04/cors.html)

CORS需要浏览器和服务器同时支持，允许浏览器向跨源服务器，发出XMLHttpRequest请求，从而克服了AJAX只能同源使用的限制。

整个CORS通信过程，都是浏览器自动完成，不需要用户参与。对于开发者来说，CORS通信与同源的AJAX通信没有差别，代码完全一样。浏览器一旦发现AJAX请求跨源，就会自动添加一些附加的头信息，有时还会多出一次附加的请求，但用户不会有感觉。

因此，实现CORS通信的关键是服务器。只要服务器实现了CORS接口，就可以跨源通信。

浏览器将CORS请求分成两类：**简单请求（simple request）和非简单请求（not-so-simple request）**。

（1） 请求方法是以下三种方法之一：

HEAD
GET
POST
（2）HTTP的头信息不超出以下几种字段：

Accept
Accept-Language
Content-Language
Last-Event-ID
Content-Type：只限于三个值application/x-www-form-urlencoded、multipart/form-data、text/plain

这是为了兼容表单（form），因为历史上**表单一直可以发出跨域请求**。AJAX 的跨域设计就是，只要表单可以发，AJAX 就可以直接发。

凡是不同时满足上面两个条件，就属于非简单请求。

浏览器对这两种请求的处理，是不一样的。

#### 简单请求

对于简单请求，浏览器直接发出CORS请求。具体来说，就是在头信息之中，增加一个Origin字段。

上面的头信息中，Origin字段用来说明，本次请求来自哪个源（协议 + 域名 + 端口）。服务器根据这个值，决定是否同意这次请求。

如果Origin指定的源，不在许可范围内，服务器会返回一个正常的HTTP回应。浏览器发现，这个回应的头信息没有包含Access-Control-Allow-Origin字段（详见下文），就知道出错了，从而抛出一个错误，被XMLHttpRequest的onerror回调函数捕获。注意，这种错误无法通过状态码识别，因为HTTP回应的状态码有可能是200。

如果Origin指定的域名在许可范围内，服务器返回的响应，会多出几个头信息字段。

```
Access-Control-Allow-Origin: http://api.bob.com
Access-Control-Allow-Credentials: true
Access-Control-Expose-Headers: FooBar
Content-Type: text/html; charset=utf-8
```

（1）Access-Control-Allow-Origin

该字段是必须的。它的值要么是请求时Origin字段的值，要么是一个*，表示接受任意域名的请求。

（2）Access-Control-Allow-Credentials

该字段可选。它的值是一个布尔值，表示是否允许发送Cookie。默认情况下，Cookie不包括在CORS请求之中。设为true，即表示服务器明确许可，Cookie可以包含在请求中，一起发给服务器。这个值也只能设为true，如果服务器不要浏览器发送Cookie，删除该字段即可。

（3）Access-Control-Expose-Headers

该字段可选。CORS请求时，XMLHttpRequest对象的getResponseHeader()方法只能拿到6个基本字段：Cache-Control、Content-Language、Content-Type、Expires、Last-Modified、Pragma。如果想拿到其他字段，就必须在Access-Control-Expose-Headers里面指定。上面的例子指定，getResponseHeader('FooBar')可以返回FooBar字段的值。

#### 非简单请求

非简单请求是那种对服务器有特殊要求的请求，比如请求方法是PUT或DELETE，或者Content-Type字段的类型是application/json。

非简单请求的CORS请求，**会在正式通信之前，增加一次HTTP查询请求，称为"预检"请求**（preflight）。

浏览器发现，这是一个非简单请求，就**自动发出一个"预检"请求**，要求服务器确认可以这样请求。

（1）Access-Control-Request-Method

该字段是必须的，用来列出浏览器的CORS请求会用到哪些HTTP方法，上例是PUT。

（2）Access-Control-Request-Headers

该字段是一个逗号分隔的字符串，指定浏览器CORS请求会额外发送的头信息字段，上例是X-Custom-Header。

一旦服务器通过了"预检"请求，以后每次浏览器正常的CORS请求，就都跟简单请求一样，会有一个Origin头信息字段。

#### 同源与同网站

“Origin”是架构（也称为协议，例如 HTTP 或 HTTPS）、主机名和端口（如果指定）的组合。例如，如果网址为 https://www.example.com:443/foo，则“origin”为 https://www.example.com:443。

![](https://web.dev/static/articles/same-site-same-origin/image/site-tld1-ae5ebbc587fbe_856.png?hl=zh-cn)

根区数据库中列出了顶级域名 (TLD)（例如 .com 和 .org）。在上面的示例中，“site”是架构、TLD 和域名前面部分（我们称之为 TLD+1）的组合。例如，如果网址为 https://www.example.com:443/foo，则“网站”为 https://example.com。

### DNS在代理环境

[ref](https://tachyondevel.medium.com/%E6%BC%AB%E8%B0%88%E5%90%84%E7%A7%8D%E9%BB%91%E7%A7%91%E6%8A%80%E5%BC%8F-dns-%E6%8A%80%E6%9C%AF%E5%9C%A8%E4%BB%A3%E7%90%86%E7%8E%AF%E5%A2%83%E4%B8%AD%E7%9A%84%E5%BA%94%E7%94%A8-62c50e58cbd0)

没有任何设置时，浏览器请求dns服务器

socks5代理设置后，浏览器不需要进行DNS请求，直接将域名写入socks请求发给代理服务器，可以是v2ray

如果v2ray中outbound设置为freedom，默认还是会用主机设置dns服务器请求，但是整个流程就多出了v2ray这一层

如果再给v2ray中outbound加上domainStrategy：UseIP，就会用到v2ray内置的dns，也就是配置的dns服务器

>在目标地址为域名时，Freedom 可以直接向此域名发出连接（"AsIs"），或者将域名解析为 IP 之后再建立连接（"UseIP"、"UseIPv4"、"UseIPv6"）。解析 IP 的步骤会使用 V2Ray 内建的 DNS。默认值为"AsIs"。

一般这种 DNS 请求都是纯文本（我们也只讨论这种 DNS），它们所流经的各种设备，各种程序都可以看到，可以修改里面的内容，发出的请求可以修改，返回的结果也可以修改，甚至修改过后毫无痕迹，就是说，**对于发出的请求，DNS 服务器没办法知道它是否在中间链路被修改过，对于返回的结果，应用程序也没办法知道它是否在中间链路被修改过。**

```json
{
    "dns": {
        "servers": [
            "8.8.8.8",
            "localhost"
        ]
    },
    "outbounds": [
        {
            "protocol": "vmess",
            "settings": {
                "vnext": [
                    {
                        "users": [
                            {
                                "id": "xxx-x-x-x-xx-x-x-x-x"
                            }
                        ],
                        "address": "1.2.3.4",
                        "port": 10086
                    }
                ]
            },
            "streamSettings": {
                "network": "tcp"
            },
            "tag": "proxy"
        },
        {
            "tag": "direct",
            "protocol": "freedom",
            "settings": {}
        }
    ],
    "inbounds": [
        {
            "domainOverride": [
                "http",
                "tls"
            ],
            "port": 1086,
            "listen": "127.0.0.1",
            "protocol": "socks",
            "settings": {
                "auth": "noauth",
                "udp": true,
                "ip": "127.0.0.1"
            }
        }
    ],
    "routing": {
        "domainStrategy": "IPIfNonMatch",
        "rules": [
            {
                "type": "field",
                "ip": [
                    "8.8.8.8"
                ],
                "outboundTag": "proxy"
            },
            {
                "type": "field",
                "domain": [
                    "geosite:cn"
                ],
                "outboundTag": "direct"
            }
        ]
    }
}
```
内置 DNS 用 8.8.8.8 做首选服务器，localhost 作备用，路由中首先来一条规则让 8.8.8.8 的流量一定走 proxy，匹配了 geosite:cn 中的域名的请求走 direct，如果没匹配任何规则，则走主 outbound，也即 outbounds 中的第一个，也即 proxy。

S4
1. 假设浏览器请求 https://www.bilibili.com
2. 浏览器发 SOCKS 请求到 V2Ray
3. 请求来到 V2Ray 的 inbound，再到路由过程
4. 很明显 www.bilibili.com 这个域名包括在 geosite:cn 中，走 direct
5. Freedom outbound (direct) 对 www.bilibili.com 发起 TCP 连接
6. Freedom outbound 解析域名，因为这次没有用 UseIP，用的是系统 DNS
7. 直接发 DNS 请求到 114.114.114.114
8. 得到结果后可以跟 Bilibili 服务器建立连接，准备代理浏览器发过来的 HTTPS 流量

S5
1. 再假设浏览器请求 https://www.google.com
2. 浏览器发 SOCKS 请求到 V2Ray
3. 请求来到 V2Ray 的 inbound，再到路由过程
4. www.google.com 不在 gesoite:cn，也没匹配任何规则，本来应该直接走主 outbound: proxy，但因为我们用了 IPIfNonMatch 策略，V2Ray 会去尝试使用内置的 DNS 把 www.google.com 的 IP 解析出来
5. V2Ray 使用内置 DNS 向 8.8.8.8 发起针对 www.google.com 的 DNS 请求，这个请求的流量将会是 UDP 流量
6. 内置 DNS 发出的 DNS 请求会按路由规则走，因为 8.8.8.8 匹配了路由中的第一条规则，这个 DNS 请求的流量会走 proxy
7. proxy 向远端代理服务器发起 TCP 代理连接（因为 "network": "tcp"）
8. 建立起 TCP 连接后，proxy 向远端代理服务器发出 udp:8.8.8.8:53 这样的代理请求
9. 远端服务器表示接受这个代理请求后，proxy 用建立好的 TCP 连接向远端服务器发送承载了 DNS 请求的 UDP 流量（所以 V2Ray/VMess 目前是 UDP over TCP）
10. 远端代理服务器接收到这些承载 DNS 请求的 UDP 流量后，发送给最终目标 udp:8.8.8.8:53
11. 8.8.8.8 返回给远端代理服务器 DNS 结果后，远端代理服务器原路返回至本地 V2Ray 的内置 DNS，至此，从步骤 5 ~ 11，整个 DNS 解析过程完成。
12. 接上面步骤 4，V2Ray 得到 www.google.com 的 IP，再进行一次规则匹配，很明显路由规则中没有相关的 IP 规则，所以还是没匹配到任何规则，最终还是走了主 outbound: proxy
13. proxy 向远端代理服务器发起 TCP 代理连接（因为 "network": "tcp"）
14. 连接建立后，因为 proxy 中所用的 VMess 协议可以像 SOCKS 那样把域名交给代理服务器处理，所以本地的 V2Ray 不需要自己解析 www.google.com，把域名放进 VMess 协议的参数中一并交给代理服务器来处理
15. 远端的 V2Ray 代理服务器收到这个代理请求后，它可能自己做域名解析，也可能继续交给下一级代理处理，只要后续代理都支持类 SOCKS 的域名处理方式，这个 DNS 请求就可以一推再推，推给最后一个代理服务器来处理，这个超出本文范围不作讨论，反正这个域名不需要我们本地去解析
16. 远端代理服务器最后会发出针对 www.google.com 的 DNS 请求（至于究竟是如何发，发到哪个 DNS 服务器，我们不一定能知道，也不关心这个）
17. 远端代理服务器得到 DNS 结果后，可以真正地向 Google 的服务器建立 TCP 连接18. 远端的 V2Ray 做好准备后告诉本地 V2Ray 连接建立好了，可以传数据了
19. 本地 V2Ray 就告诉浏览器连接好了，可以传数据了，浏览器就可以把 HTTPS 流量顺着这个代理链发送至 Google 的服务器

#### DNS分流

根据域名来选择使用哪个域名服务器来解析该域名，这个功能的用途在于直接用国外的dns服务器解析会被墙，而走代理解析又可能会返回国外cdn的结果，所以需要直接用国内的dns服务器来解析

关于内置 DNS，最后再说一说 localhost ，上面的描述不适用于 localhost，简单说，内置 DNS 在向 servers 列表中的 localhost 发 DNS 请求时，不会用任何 outbound 来发（甚至不用 Freedom outbound），而是直接从本机发出，就像任何其它程序做 DNS 请求那样，直接调用系统的 DNS API，用系统 DNS 中配置的 DNS 服务器。

### cookie session token JWT

cookie用作身份识别，一段时间不用再登录，cookie为了防止被冒名，可以进行数字签名

session其实就是一种cookie，目的是服务器不希望存储太多数据在浏览器cookie jar中，可以在服务器上绑定一个key和登录信息的关系，这个key发给浏览器设置为cookie，就是session id，服务器就可以根据session id查找到登录信息

除了浏览器的其他客户端没有cookie机制，所以就出现了token，token就相当于一种session

JWT是一种token，反而类似于cookie，原因与session出现的理由相反，不希望服务器存放太多数据，希望接口保持无状态，因为有些服务器可能是分布式，如果需要存储这个数据，会相对麻烦。所以让客户端多携带信息，然后进行数字签名，JWT就是一种。

总结：cookie、session用于浏览器，token、JWT用于客户端。cookie、JWT携带信息，session、token让服务器存储信息，只携带对应的key

### ssh tunnel

可以将远程主机监听localhost的端口暴露到本地主机上，也可以访问只有堡垒机才能访问的端口

[ref](https://iximiuz.com/en/posts/ssh-tunnels/)

### 查询外网出口ip

[ref](https://www.cnblogs.com/nbnode/p/12780875.html)

curl icanhazip.com
curl ifconfig.me
curl curlmyip.com
curl ip.appspot.com
curl ipinfo.io/ip
curl ipecho.net/plain
curl www.trackip.net/i