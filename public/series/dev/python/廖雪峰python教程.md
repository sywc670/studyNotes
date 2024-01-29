# python

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