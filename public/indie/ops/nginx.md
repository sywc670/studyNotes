# nginx

[一个nginx系列博客](https://blog.csdn.net/zhangyue0503/article/details/131346099?spm=1001.2014.3001.5502)

### nginx重定向和反向代理是有区别的

重定向是服务器告诉客户端在其他地方寻找资源，客户端必须能访问到新地址，反向代理是服务器将请求转发到其他地方，客户端不需要能访问到
重定向：return rewrite等
反向代理：proxy_pass

### request_time等字段解析

[ref](https://blog.csdn.net/zzhongcy/article/details/105819628)

![](../../../reference/pic/nginxrequesttime.png)

### 限流设置

[ref](https://blog.csdn.net/goGoing_/article/details/130634946)

limit_req_zone：只定义速度限制的相关参数，一般用于http块中，使其可以在多个相关server中使用

limit_req：启用定义的限速参数

burst ：表示在超过设定的处理速率后能额外处理的请求数

"nodelay"：表示遇到超过速率限制的请求时不进行延迟处理。如果不设置这个选项，超出速率限制的请求将会被延时处理，直到能够在允许的速率范围内进行处理。

```conf
    location /brand {
        # 同一个地址只允许连接2次
        limit_conn zoneName 2;
        proxy_pass http://192.168.211.1:18081;
    }
```    
### 实现服务端获取客户端真实ip

[ref](https://zhuanlan.zhihu.com/p/391215425)

nginx会将认为是真实客户端IP赋值给remote_addr

#### X-Real-IP

realIp 模块用于配置 nginx 自己如何获取客户端真实 IP，nginx 获取的客户端 IP 会放到 $remote_addr 变量中

**nginx 首先会从 TCP 连接层面拿到 IP 赋值给变量 $remote_addr ，并认为该 IP 就是客户端真实 IP，如果把从 TCP 连接拿到的 IP 通过 set_real_ip_from 指令配置为授信 IP，则 nginx 才会通过 realIp 模块的相关指令配置拿到 IP 赋值给变量 $remote_addr，并认为该IP是客户端真实的 IP**

nginx需要向后端传参数，`proxy_set_header X-real-ip $remote_addr;`，其中这个X-real-ip是一个自定义的变量名，名字可以随意取，这样做完之后，用户的真实ip就被放在X-real-ip这个变量里了，然后，在web端可以这样获取：`request.getAttribute(“X-real-ip”)`

#### X-Forwarded-For

`proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;`，这里是另一种方式，变量$proxy_add_x_forwarded_for取值是nginx自行决定赋值的

set_real_ip_from ：设置反向代理服务器，即信任服务器IP

real_ip_header X-Forwarded-For ：用户真实IP存在 X-Forwarded-For 请求头中，默认是X-Real-IP，但是现在主流用X-Forwarded-For

real_ip_recursive ：
- off ：会将 real_ip_header 指定的HTTP头中的最后一个IP作为真实IP
- on ：会将 real_ip_header 指定的HTTP头中的最后一个不是信任服务器的IP当成真实IP

### gzip

[ref](https://blog.csdn.net/zhangyue0503/article/details/132750308)

### 反向代理缓存

[ref](https://blog.csdn.net/shark_chili3007/article/details/104009742)

### nginx重载注意事项

nginx在加载配置文件启动后，重载需要`nginx -s reload -c nginx.conf`