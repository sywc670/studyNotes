# windows

### 彻底解决docker在windows上的端口绑定问题

>Error invoking remote method ‘docker-start-container’: Error: (HTTP code 500) server error - Ports are not available: listen tcp 0.0.0.0:xxxx: bind: An attempt was made to access a socket in a way forbidden by access permissions.

```shell
netsh int ipv4 set dynamic tcp start=49152 num=16384
netsh int ipv6 set dynamic tcp start=49152 num=16384
#重启电脑
```

Windows 中个东西叫做“TCP 动态端口范围”，这个范围内的端口有时候会被一些服务占用。在 Windows Vista（或 Windows Server 2008）之前，动态端口范围是 1025 到 5000；在其之后的版本中，新的默认起始端口为 49152，新的默认结束端口为 65535。
如果安装了 Hyper-V，则 Hyper-V 会保留一些随机端口号供 Windows 容器主机网络服务使用。
一般情况（正常情况下）Hyper-V 会在“TCP 动态端口范围”中预留一些随机的端口号，但是预留的端口号一般都很大，所以即使预留了成百上千个端口，也影响不大。但是 Windows 自动更新有时会出错（万恶的自动更新），把“TCP 动态端口范围”起始端口被重置为 1024，导致 Hyper-V 在预留端口的时候占用了常用端口号，使得一些常用端口因为被预留而无法使用。

你可以使用命令 `netsh int ipv4 show dynamicport tcp` 查看当前的“TCP 动态端口范围”，还可使用 `netsh int ipv4 show excludeportrange protocol=tcp` 查看当前已经被征用的端口。

### hosts

C:\Windows\System32\drivers\etc