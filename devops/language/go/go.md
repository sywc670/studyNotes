
### 小知识点

#### 接口型函数

为什么需要接口型的函数，接口比函数适用更广泛的情况，接口型函数只是接口实现的一种形式，如果直接用函数作为参数适用性就不够了

其实就是方便直接传入函数作为参数

[ref](https://geektutu.com/post/7days-golang-q1.html)

#### uintptr和unsafe.Pointer的区别

unsafe.Pointer只是单纯的通用指针类型，**用于转换不同类型指针, 不同类型的指针无法直接转换**，它不可以参与指针运算；

而uintptr是用于指针运算的，GC 不把 uintptr 当指针，也就是说 uintptr 无法持有对象， **uintptr 类型的目标会被回收，只是目标会回收**；

unsafe.Pointer 可以和 普通指针 进行相互转换；

unsafe.Pointer 可以和 uintptr 进行相互转换。


#### 多个变量同时赋值

```go
    // 下面的赋值是语法糖，前面的赋值不会影响后面的赋值，因为有临时变量
	// tmp := dp[j+1]
	// dp[j+1] = pre+1
	// pre = tmp
	dp[j+1], pre = pre+1, dp[j+1]
```

#### struct比较

相同struct类型的可以⽐较，但是仅限其字段可以比较的时候

不同struct类型的不可以⽐较,编译都不过，类型不匹配

#### iota使用

```go
func Test10(t *testing.T) {
 const (
 x = iota
 _
 y
 z = "pi"
 k
 p = iota
 q
 )
 fmt.Println(x, y, z, k, p, q)
}
// 0 2 pi pi 5 6
```

#### panic recover

一个协程会发生 panic ，导致程序崩溃，但是只会执行自己所在 Goroutine 的延迟函数，所以正好验证了多个 Goroutine 之间没有太多的关联，一个 Goroutine 在 panic 时也不应该执行其他 Goroutine 的延迟函数。




## 《GO语言精进之路》

### GMP

Go从1.5版本开始将P的默认数量由1改为CPU核的数量(实际上还乘了每个核上硬线程数量)

#### I/O

Go运行时已经实现了**netpoller**，这使得即便G发起网络I/O操作也不会导致M被阻塞（仅阻塞G），因而不会导致大量线程（M）被创建出来。但是对于常规文件的I/O操作一旦阻塞，那么线程（M）将进入挂起状态，等待I/O返回后被唤醒。这种情况下P将与挂起的M分离，再选择一个处于空闲状态（idle）的M。如果此时没有空闲的M，则会新创建一个M（线程），这就是大量文件I/O操作会导致大量线程被创建的原因。

Go开发团队的Ian Lance Taylor在Go 1.9版本中增加了一个针对文件I/O的Poller，它可以像netpoller那样，在G操作那些支持监听的（pollable）文件描述符时，仅阻塞G，而不会阻塞M。**不过该功能依然对常规文件无效，常规文件是不支持监听的**。但对于goroutine调度器而言，这也算是一个不小的进步了。

#### 抢占式调度

栈增长抢占：

原理是在每个函数或方法的入口加上一段额外的代码，让运行时有机会检查是否需要执行抢占调度。

这种协作式抢占调度的解决方案只是局部解决了“饿死”问题，对于没有函数调用而是纯算法循环计算的G，goroutine调度器依然无法抢占

异步抢占：

但Go 1.14版本中加入了**基于系统信号的goroutine抢占式调度机制**，上述问题被解决

---

运行时间过长被调度：

可以看出，如果一个G任务运行超过10ms，sysmon就会认为其运行时间太久而发出抢占式调度的请求。一旦G的抢占标志位被设为true，那么在这个G下一次调用函数或方法时，运行时便可以将G抢占并移出运行状态，放入P的本地运行队列中（如果P的本地运行队列已满，那么将放在全局运行队列中），等待下一次被调度。

channel阻塞或网络I/O情况下的调度:

如果G被阻塞在某个channel操作或网络I/O操作上，那么G会被放置到某个等待队列中，而M会尝试运行P的下一个可运行的G。如果此时P没有可运行的G供M运行，那么M将解绑P，并进入挂起状态。当I/O操作完成或channel操作完成，在等待队列中的G会被唤醒，标记为runnable（可运行），并被放入某个P的队列中，绑定一个M后继续执行。

系统调用阻塞情况下的调度:

如果G被阻塞在某个系统调用上，**那么不仅G会阻塞，执行该G的M也会解绑P（实质是被sysmon抢走了），与G一起进入阻塞状态**。如果此时有空闲的M，则P会与其绑定并继续执行其他G；如果没有空闲的M，但仍然有其他G要执行，那么就会创建一个新M（线程）。当系统调用返回后，阻塞在该系统调用上的G会尝试获取一个可用的P，如果有可用P，之前运行该G的M将绑定P继续运行G；如果没有可用的P，那么G与M之间的关联将解除，同时G会被标记为runnable，放入全局的运行队列中，等待调度器的再次调度

##### sysmon

监控线程sysmon由main goroutine创建，不是工作线程，不受GPM模型管理，不需要P，以g0执行一系列重复任务

sysmon每20us~10ms启动一次，主要完成如下工作:

- 释放闲置超过5 分钟的 span 物理内存； 
- 如果超过2 分钟没有垃圾回收，强制执⾏； 
- 将⻓时间未处理的 netpoll 添加到全局队列； 
- 向⻓时间运⾏的 G 任务发出抢占调度(超过10ms的 g，会进⾏ retake)；
- 收回因 syscall ⻓时间阻塞的 P；

## 《Go系统调用与阻塞处理》

[ref](https://mp.weixin.qq.com/s/u6vC_sUf5ultPRhaJnSRHQ)

在 Go 里面阻塞主要分为以下 4 种场景：

1. 由于原子、互斥量或通道操作调用导致  Goroutine  阻塞，调度器将把当前阻塞的 Goroutine 切换出去，重新调度 LRQ 上的其他 Goroutine；
2. 由于网络请求和 IO 操作导致  Goroutine  阻塞。Go 程序提供了网络轮询器（NetPoller）来处理网络请求和 IO 操作的问题，其后台通过 kqueue（MacOS），epoll（Linux）或  iocp（Windows）来实现 IO 多路复用。通过使用 NetPoller 进行网络系统调用，调度器可以防止  Goroutine  在进行这些系统调用时阻塞 M。这可以让 M 执行 P 的  LRQ  中其他的  Goroutines，而不需要创建新的 M。执行网络系统调用不需要额外的 M，网络轮询器使用系统线程，它时刻处理一个有效的事件循环，有助于减少操作系统上的调度负载。用户层眼中看到的 Goroutine 中的“block socket”，实现了 goroutine-per-connection 简单的网络编程模式。实际上是通过 Go runtime 中的 netpoller 通过 Non-block socket + I/O 多路复用机制“模拟”出来的。
3. 当调用一些系统方法的时候（如文件 I/O），如果系统方法调用的时候发生阻塞，这种情况下，网络轮询器（NetPoller）无法使用，而进行系统调用的  G1  将阻塞当前 M1。调度器引入 其它M 来服务 M1 的P。
4. 如果在 Goroutine 去执行一个 sleep 操作，导致 M 被阻塞了。Go 程序后台有一个监控线程 sysmon，它监控那些长时间运行的 G 任务然后设置可以强占的标识符，别的 Goroutine 就可以抢先进来执行。

>time.Sleep会导致m被阻塞吗？
futex是系统调用，所以会阻塞m，但是m应该只会在一小段时间阻塞
In summary, when we call Sleep() function in Golang,
App calls sleep of Golang -> Golang calls futex of Linux -> futex utilizes hrtimer to be timer.
[ref](https://xwu64.github.io/2019/02/27/Understanding-Golang-sleep-function/)

sysmon负责：

1. 检查死锁runtime.checkdead
2. 运行计时器 — 获取下一个需要被触发的计时器；
3. 定时从 netpoll 中获取 ready 的协程
4. Go 的抢占式调度：当 sysmon 发现 M 已运行同一个 G（Goroutine）10ms 以上时，它会将该 G 的内部参数 preempt 设置为 true。然后，在函数序言中，当 G 进行函数调用时，G 会检查自己的 preempt 标志，如果它为 true，则它将自己与 M 分离并推入“全局队列”。由于它的工作方式（函数调用触发），在 for{} 的情况下并不会发生抢占，如果没有函数调用，即使设置了抢占标志，也不会进行该标志的检查。Go1.14 引入抢占式调度（使用信号的异步抢占机制），sysmon 仍然会检测到运行了 10ms 以上的 G（goroutine）。然后，sysmon 向运行 G 的 P 发送信号（SIGURG）。Go 的信号处理程序会调用P上的一个叫作 gsignal 的 goroutine 来处理该信号，将其映射到 M 而不是 G，并使其检查该信号。gsignal 看到抢占信号，停止正在运行的 G。
5. 在满足条件时触发垃圾收集回收内存；
6. 打印调度信息,归还内存等定时任务.

## 《Go专家编程》

### select

select的case语句**读channel不会阻塞，尽管channel中没有数据**。这是由于case语句编译后调用读channel时会明确传入不阻塞的参数，此时读不到数据时不会将当前goroutine加入到等待队列，而是直接返回。

### slice

```go
s1 := make([]int, 5, 10)
s2 := s1[:6:7] // 第三位也是cap，不能超过s1的cap，但是第二位可以超过切片的len，不能超过cap
s1[8] = 1 // panic 
```

只有数组可以比较，切片不可比较

切片初始化指data会指向一个数组，如果没有初始化就只是零值nil

使用copy()内置函数拷贝两个切片时，会将源切片的数据逐个拷贝到目的切片指向的数组中，拷贝数量取两个切片长度的最小值。

例如长度为10的切片拷贝到长度为5的切片时，将会拷贝5个元素。

也就是说，copy过程中不会发生扩容。

**扩容规则**

Go1.18开始：

先判断旧的容量，如果小于256，2倍扩容。否则计算增长因子(growth factor)扩容，增长因子使得大切片(>256)的扩容更加平滑，随着容量的增加，增长因子会越来越小，直到无限趋近于1.25倍

最后，进行内存对齐计算，和老版本计算规则一样

### map

结构：

```go
type hmap struct {
    count     int // 当前保存的元素个数
    ...
    B         uint8  // 表示bucket的个数，实际个数为2^B
    ...
    buckets    unsafe.Pointer // bucket数组指针
    ...
}

type bmap struct {
    tophash [8]uint8 //存储哈希值的高8位
    data    byte[1]  //key value数据:key/key/key/.../value/value/value...
    overflow *bmap   //溢出bucket的地址
}
```
每个bucket可以存储8个键值对。

- tophash是个长度为8的数组，哈希值相同的键（准确的说是哈希值低位相同的键）存入当前bucket时会将哈希值的高位存储在该数组中，以方便后续匹配。
- data区存放的是key-value数据，存放顺序是key/key/key/...value/value/value，如此存放是为了节省字节对齐带来的空间浪费。
- overflow 指针指向的是下一个bucket，据此将所有冲突的键连接起来。

注意：上述中data和overflow并不是在结构体中显示定义的，而是直接通过指针运算进行访问的。

查找过程：
![](../../../reference/pic/gomap.png)

### string

因为string通常指向字符串字面量，而字符串字面量存储位置是只读段，而不是堆或栈上，所以才有了string不可修改的约定。

### defer

```go
func deferFuncParameter() {
    var aInt = 1

    defer fmt.Println(aInt)

    aInt = 2
    return
}
```

输出1

### array

数组长度不可变，但可以改变取值

### range 

slice:

遍历slice前会先获以slice的长度len_temp作为循环次数，循环体中，每次循环会先获取元素值，如果for-range中接收index和value的话，则会对index和value进行一次赋值。

由于循环开始前循环次数就已经确定了，所以循环过程中新添加的元素是没办法遍历到的。

另外，数组与数组指针的遍历过程与slice基本一致，不再赘述。

map:

遍历map时没有指定循环次数，循环体与遍历slice类似。由于map底层实现与slice不同，map底层使用hash表实现，插入数据位置是随机的，所以遍历过程中新插入的数据不能保证遍历到。

channel:

channel遍历是依次从channel中读取数据,读取前是不知道里面有多少个元素的。如果channel中没有元素，则会阻塞等待，如果channel已被关闭，则会解除阻塞并退出循环。

### RWMutex

```go
type RWMutex struct {
    w           Mutex  //用于控制多个写锁，获得写锁首先要获取该锁，如果有一个写锁在进行，那么再到来的写锁将会阻塞于此
    writerSem   uint32 //写阻塞等待的信号量，最后一个读者释放锁时会释放信号量
    readerSem   uint32 //读阻塞的协程等待的信号量，持有写锁的协程释放锁后会释放信号量
    readerCount int32  //记录读者个数
    readerWait  int32  //记录写阻塞时读者个数
}
```

**写操作是如何阻止写操作的**

读写锁包含一个互斥锁(Mutex)，写锁定必须要先获取该互斥锁，如果互斥锁已被协程A获取（或者协程A在阻塞等待读结束），意味着协程A获取了互斥锁，那么协程B只能阻塞等待该互斥锁。

所以，写操作依赖互斥锁阻止其他的写操作。

**写操作是如何阻止读操作的**

我们知道RWMutex.readerCount是个整型值，用于表示读者数量，不考虑写操作的情况下，每次读锁定将该值+1，每次解除读锁定将该值-1，所以readerCount取值为`[0, N]`，N为读者个数，实际上最大可支持2^30个并发读者。

当写锁定进行时，会先将readerCount减去2^30，从而readerCount变成了负值，此时再有**读锁定到来时检测到readerCount为负值，便知道有写操作在进行，只好阻塞等待。而真实的读操作个数并不会丢失，只需要将readerCount加上2^30即可获得**。

所以，写操作将readerCount变成负值来阻止读操作的。

**读操作是如何阻止写操作的**

读锁定会先将RWMutext.readerCount加1，此时写操作到来时发现读者数量不为0，会阻塞等待所有读操作结束。

所以，读操作通过readerCount来将来阻止写操作的。

**为什么写锁定不会被饿死**

我们知道，写操作要等待读操作结束后才可以获得锁，写操作等待期间可能还有新的读操作持续到来，如果写操作等待所有读操作结束，很可能被饿死。然而，通过RWMutex.readerWait可完美解决这个问题。

写操作到来时，会把RWMutex.readerCount值拷贝到RWMutex.readerWait中，用于标记排在写操作前面的读者个数。

前面的读操作结束后，除了会递减RWMutex.readerCount，还会递减RWMutex.readerWait值，当RWMutex.readerWait值变为0时唤醒写操作。

所以，写操作就相当于把一段连续的读操作划分成两部分，前面的读操作结束后唤醒写操作，写操作结束后唤醒后面的读操作。


### 逃逸分析

1. 局部变量被外部引用必然逃逸，被闭包引用也会逃逸
2. 当栈空间不足以存放当前对象时或无法判断当前切片长度时会将对象分配到堆中
3. interface无法在编译时判断类型，也会逃逸

### 反射 接口

```go
type Myint int

var i int
var j Myint
```

i与j虽然底层类型一样，但是**静态类型不同**，interface就是一种特殊的静态类型
每个变量都有一个静态类型，且在编译时就确定了，静态类型就是声明的类型

```go
var r io.Reader
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty
```

r的类型始终是io.Readerinterface类型，无论其存储什么值。

r保存了一个(value, type)对来表示其所存储值的信息。 value即为r所持有元素的值，type即为所持有元素的**动态类型(不是底层类型)**

```go
	var i interface{}
	type myint int
	var j myint
	i = j
	fmt.Println(reflect.TypeOf(i)) // main.myint
```
注意：反射是针对interface类型变量的，其中TypeOf()和ValueOf()接受的参数都是interface{}类型的，也即值是被转成了interface传入的。

反射三定律：

- 反射第一定律：反射可以将interface类型变量转换成反射对象
- 反射第二定律：反射可以将反射对象还原成interface对象
- 反射第三定律：反射对象可修改，value值必须是可设置的

### go标准库

#### io

- ReadAt WriteAt,WriteTo ReadFrom, ReadAll这些进一步封装，更细化的操作
- io.Seeker

#### os

- os.ReadFile os.ReadDir 
- os.Open os.OpenFile

#### fmt

- Stringer、Formatter、Scanner、GoStringer这几个接口，类似于python的魔术方法

#### bufio

提供了带缓存的一些实现了io库接口的结构体，方便读写操作
Scanner方便每次读写一部分，其中分词默认是分行，可以自定义实现

#### strings

提供Replacer、Reader、Builder三个结构体

#### time

两种定时器：Timer（到达指定时间触发且只触发一次）和 Ticker（间隔特定时间触发）

#### template

[ref](https://www.cnblogs.com/f-ck-need-u/p/10053124.html#%E5%B5%8C%E5%A5%97templatedefine%E5%92%8Ctemplate)

helm模板也有用到go template，可以check下那边

在template中，点"."代表当前作用域的当前对象

有一个特殊变量$，它代表模板的最顶级作用域对象(通俗地理解，是以模板为全局作用域的全局变量)，在Execute()执行的时候进行赋值，且一直不变

{{template "name"}}
{{template "name" pipeline}}

第一种是直接执行名为name的template，点设置为nil。第二种是点"."设置为pipeline的值，并执行名为name的template。

{{range pipeline}} T1 {{end}}
{{range pipeline}} T1 {{else}} T0 {{end}}

对于第一个表达式，当迭代对象的值为0值时，则range直接跳过，就像if一样。对于第二个表达式，则在迭代到0值时执行else语句。

如果range中只赋值给一个变量，则这个变量是当前正在迭代元素的值。如果赋值给两个变量，则第一个变量是索引值(map/slice是数值，map是key)，第二个变量是当前正在迭代元素的值。

{{with pipeline}} T1 {{end}}
{{with pipeline}} T1 {{else}} T0 {{end}}

对于第一种格式，当pipeline不为0值的时候，点"."设置为pipeline运算的值，否则跳过。对于第二种格式，当pipeline为0值时，执行else语句块，否则"."设置为pipeline运算的值，并执行T1。

block和define十分类似，根据官方文档的解释：block等价于define定义一个名为name的模板，并在"有需要"的地方执行这个模板，执行时将"."设置为pipeline的值。

但应该注意，block的第一个动作是执行名为name的模板，如果不存在，则在此处自动定义这个模板，并执行这个临时定义的模板。换句话说，block可以认为是**设置一个默认模板，执行模板**。

{{block "T1" .}} one {{end}}

它首先表示{{template "T1" .}}，也就是说先找到T1模板，如果T1存在，则执行找到的T1，如果没找到T1，则临时定义一个{{define "T1"}} one {{end}}，并执行它。

对于html/template包，有一个很好用的功能：上下文感知。text/template没有该功能。

上下文感知具体指的是根据所处环境css、js、html、url的path、url的query，自动进行不同格式的转义。

{{ $v := or .Site.Language.LanguageCode .Site.Language.Lang }} or用来设置值，如果前一个没有值，就会用后一个设置值，相当于有backup

不要将代码用{{/* */}}注释

##### hugo

template

Use the partial or partialCached function to include one or more partial templates

seq 4 相当于 python的range(1,5)