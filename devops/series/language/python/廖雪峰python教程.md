### 生成器

yield相当于return，只是每次调用完毕后，下次调用会从yield之后开始执行，直到下一个yield

凡是可作用于for循环的对象都是Iterable类型；

凡是可作用于next()函数的对象都是Iterator类型，它们表示一个惰性计算的序列；

集合数据类型如list、dict、str等是Iterable但不是Iterator，不过可以通过iter()函数获得一个Iterator对象。

### map reduce filter

map是将函数作用在所有数上，reduce是将函数两两作用，变成一个数

```python
# map
>>> def f(x):
...     return x * x
...
>>> r = map(f, [1, 2, 3, 4, 5, 6, 7, 8, 9])
>>> list(r)

# reduce
>>> from functools import reduce
>>> def fn(x, y):
...     return x * 10 + y
...
>>> reduce(fn, [1, 3, 5, 7, 9])
```

filter根据返回值来筛选

### 闭包

使用闭包时，对外层变量赋值前，需要先使用nonlocal声明该变量不是当前函数的局部变量。

### 模块

在Python中，一个.py文件就称之为一个模块（Module）

为了避免模块名冲突，Python又引入了按目录来组织模块的方法，称为包（Package）

每一个包目录下面都会有一个__init__.py的文件，这个文件是必须存在的，否则，Python就把这个目录当成普通目录，而不是一个包。`__init__.py`可以是空文件，也可以有Python代码，因为__init__.py本身就是一个模块，而它的模块名就是所在目录名。

>go里模块比包的概念要大

python会在`sys.path`中搜索模块，可以临时append路径，也可以用环境变量`PYTHONPATH`

### __slots__

默认情况下类和实例可以绑定任何属性和方法
```py
class Student(object):
    __slots__ = ('name', 'age') # 用tuple定义允许绑定的属性名称
```
使用__slots__要注意，__slots__定义的属性仅对当前类实例起作用，对继承的子类是不起作用的

### mixin

由于Python允许使用多重继承，因此，MixIn就是一种常见的设计。

只允许单一继承的语言（如Java）不能使用MixIn的设计。

### 定制类

`__str__()`返回用户看到的字符串，而__repr__()返回程序开发者看到的字符串，也就是说，`__repr__()`是为调试服务的

如果一个类想被用于for ... in循环，类似list或tuple那样，就必须实现一个`__iter__()`方法，该方法返回一个迭代对象，然后，Python的for循环就会不断调用该迭代对象的`__next__()`方法拿到循环的下一个值，直到遇到StopIteration错误时退出循环

要表现得像list那样按照下标取出元素，需要实现`__getitem__()`方法，该方法如果要实现切片、步长等功能比较复杂。

`__getattr__()`方法，动态返回一个属性，可以是一个函数，在没有找到该属性时会调用

`__call__()`可以让一个实例可以被以函数的方式调用，如`s = Student(); s()`

#### 链式调用

利用完全动态的__getattr__，我们可以写出一个链式调用：
```py
class Chain(object):

    def __init__(self, path=''):
        self._path = path

    def __getattr__(self, path):
        return Chain('%s/%s' % (self._path, path))

    def __str__(self):
        return self._path

    __repr__ = __str__
#试试：

>>> Chain().status.user.timeline.list
'/status/user/timeline/list'

#实现Chain().users('will').list
#可以特殊处理users这个属性，返回一个匿名函数，接受一个参数
```

### type()

动态语言和静态语言最大的不同，就是函数和类的定义，不是编译时定义的，而是运行时动态创建的。

要创建一个class对象，type()函数依次传入3个参数：

class的名称；
继承的父类集合，注意Python支持多重继承，如果只有一个父类，别忘了tuple的单元素写法；
class的方法名称与函数绑定，这里我们把函数fn绑定到方法名hello上。

### 元类

metaclass，直译为元类，简单的解释就是：

当我们定义了类以后，就可以根据这个类创建出实例，所以：先定义类，然后创建实例。

但是如果我们想创建出类呢？那就必须根据metaclass创建出类，所以：先定义metaclass，然后创建类。

连接起来就是：先定义metaclass，就可以创建类，最后创建实例。

### 错误

因为错误是class，捕获一个错误就是捕获到该class的一个实例。因此，错误并不是凭空产生的，而是有意创建并抛出的。Python的内置函数会抛出很多类型的错误，我们自己编写的函数也可以抛出错误。

### I/O

当我们写文件时，操作系统往往不会立刻把数据写入磁盘，而是放到内存缓存起来，空闲的时候再慢慢写入。只有调用close()方法时，操作系统才保证把没有写入的数据全部写入磁盘。忘记调用close()的后果是数据可能只写了一部分到磁盘，剩下的丢失了

### 正则

正则匹配默认是贪婪匹配，也就是匹配尽可能多的字符

### 异步I/O

多线程和多进程的模型虽然解决了并发问题，但是系统不能无上限地增加线程。由于系统切换线程的开销也很大，所以，一旦线程数量过多，CPU的时间就花在线程切换上了，真正运行代码的时间就少了，结果导致性能严重下降。

由于我们要解决的问题是CPU高速执行能力和IO设备的龟速严重不匹配，多线程和多进程只是解决这一问题的一种方法。

另一种解决IO问题的方法是异步IO。当代码需要执行一个耗时的IO操作时，**它只发出IO指令，并不等待IO结果，然后就去执行其他代码了**。一段时间后，当IO返回结果时，再通知CPU进行处理。

异步的好处是可以快速响应请求，节省IO时间

>go的解决方法就是goroutine，同步的方式