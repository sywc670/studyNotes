### python博客

[*, **](https://blog.csdn.net/m0_72557783/article/details/128121520)

### flask

#### WSGI

[ref](https://segmentfault.com/a/1190000003069785)

WSGI相当于是Web服务器和Python应用程序之间的桥梁。那么这个桥梁是如何工作的呢？首先，我们明确桥梁的作用，WSGI存在的目的有两个：

让Web服务器知道如何调用Python应用程序，并且把用户的请求告诉应用程序。

让Python应用程序知道用户的具体请求是什么，以及如何返回结果给Web服务器。

web服务器和应用通信就通过WSGI协议，至于为什么需要这个协议？是一个功能的拆分，web框架不用实现这部分功能。

>这里的web服务器是python中的一部分，不是nginx