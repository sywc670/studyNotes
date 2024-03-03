### python语法

#### open模式

w在打开之后会删除文件所有内容，所以w+和r+的区别是w+会删除内容，而r+只会覆盖写的部分，a+是从末尾开始

#### 除法对比

/在python3中可以产生浮点数，python2不行
//在python3才是整除

go语言中整数相除是整除，浮点数除整数是浮点数，仅限字面量可以

#### *, **用法

*可以传list，但是要传list得是`*list`,**可以传dict，但是`**dict`，且key不能与其他参数重复
```
def f1(a, b, c=0, *args, **kw):
    print('a =', a, 'b =', b, 'c =', c, 'args =', args, 'kw =', kw)

```

*args是可变参数，args接收的是一个tuple；

**kw是关键字参数，kw接收的是一个dict。

可变参数既可以直接传入：func(1, 2, 3)，又可以先组装list或tuple，再通过*args传入：func(*(1, 2, 3))；

关键字参数既可以直接传入：func(a=1, b=2)，又可以先组装dict，再通过**kw传入：func(**{'a': 1, 'b': 2})。

### flask

#### WSGI

[ref](https://segmentfault.com/a/1190000003069785)

WSGI相当于是Web服务器和Python应用程序之间的桥梁。那么这个桥梁是如何工作的呢？首先，我们明确桥梁的作用，WSGI存在的目的有两个：

让Web服务器知道如何调用Python应用程序，并且把用户的请求告诉应用程序。

让Python应用程序知道用户的具体请求是什么，以及如何返回结果给Web服务器。

web服务器和应用通信就通过WSGI协议，至于为什么需要这个协议？是一个功能的拆分，web框架不用实现这部分功能。

>这里的web服务器是python中的一部分，不是nginx

### python库

#### pandas

[官方文档](https://pandas.pydata.org/pandas-docs/stable/index.html)

### venv 虚拟环境

#### 方法一

mkdir venv
cd venv
python3 -m venv .
cd bin
source activate
deactivate
https://www.liaoxuefeng.com/wiki/1016959663602400/1019273143120480

#### 方法二

sudo yum install python-devel libffi-devel gcc openssl-devel libselinux-python
sudo yum install python-virtualenv
mkdir /opt/virtualenv
virtualenv /opt/virtualenv
source /opt/virtualenv/bin/activate

### pip使用国内镜像源

#### 方法一

我们可以直接在 pip 命令中使用 -i 参数来指定镜像地址，例如：
pip3 install numpy -i https://pypi.tuna.tsinghua.edu.cn/simple

pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple

https://www.runoob.com/w3cnote/pip-cn-mirror.html
https://mirrors.tuna.tsinghua.edu.cn/help/pypi/

#### 方法二

配置国内的pip源
```
mkdir ~/.pip
cat << EOF > ~/.pip/pip.conf
[global]
index-url = https://mirrors.aliyun.com/pypi/simple/

[install]
trusted-host=mirrors.aliyun.com
EOF
```

原文链接：https://blog.csdn.net/qq_45382565/article/details/126657144

#### There was a problem confirming the ssl certificate

临时
pip install numpy -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn
新建pip.ini
```ini
[global]
index-url = http://mirrors.aliyun.com/pypi/simple/
[install]
trusted-host = mirrors.aliyun.com
```
https://www.cnblogs.com/yinhaiping/p/13375375.html

#### pip install -U pip 升级到22版本报错
pip install --upgrade pip==20.3.4
或者找到对应版本的get-pip文件 手动升级。
wget https://bootstrap.pypa.io/2.7/get-pip.py
python get-pip.py 
