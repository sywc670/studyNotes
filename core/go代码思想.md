
### ferry

ferry工单系统，最重要的其实是工单流程的逻辑，还有工单任务的执行调度，这部分交给了machinery库。

- 路由分组注册
- casbin权限管理
- JWT

其他见ferry.md

### gee

### geecache

### geeorm

geeorm用到了database/sql，将对数据库的操作封装起来，实际是提取变量组成sql语句进行执行，首先就是包装了sql.DB的执行和查询等函数

然后实现了数据库的表与go语言结构体之间的转换，进一步封装对表的操作

接着一些复杂的语句需要先分解成多个从句，每个从句进行组装后才能执行，所以引入clause包

### geerpc

### strings.Builder

其实原理是将string转换成[]byte，进行拼接，最后再以string输出，string如果直接用+，又不string不可变会导致重复分配内存

### atomic

原子操作由底层硬件支持，而锁则由操作系统的调度器实现。**锁应当用来保护一段逻辑，对于一个变量更新的保护，原子操作通常会更有效率**，并且更能利用计算机多核的优势，如果要更新的是一个复合对象，则应当使用`atomic.Value`封装好的实现。

底层实现：Mutex由**操作系统的调度器**实现，而atomic包中的原子操作则由**底层硬件指令直接提供支持**，这些指令在执行的过程中是不允许中断的，因此原子操作可以在lock-free的情况下保证并发安全，并且它的性能也能做到随CPU个数的增多而线性扩展。

其实Mutex的底层实现也是依赖原子操作中的CAS实现的，原子操作的atomic包相当于是sync包里的那些同步原语的实现依赖

Go语言通过内置包sync/atomic提供了对原子操作的支持，其提供的原子操作有以下几大类：
- 增减，操作的方法名方式为AddXXXType，保证对操作数进行原子的增减，支持的类型为int32、int64、uint32、uint64、uintptr，使用时以实际类型替换前面我说的XXXType就是对应的操作方法。
- 载入，保证了读取到操作数前没有其他任务对它进行变更，操作方法的命名方式为LoadXXXType，支持的类型除了基础类型外还支持Pointer，也就是支持载入任何类型的指针。
- 存储，有载入了就必然有存储操作，这类操作的方法名以Store开头，支持的类型跟载入操作支持的那些一样。
- 比较并交换，也就是CAS （Compare And Swap），像Go的很多并发原语实现就是依赖的CAS操作，同样是支持上面列的那些类型。
- 交换，这个简单粗暴一些，不比较直接交换，这个操作很少会用。

### sync.Cond

```go
var done = false

func read(name string, c *sync.Cond) {
	c.L.Lock()
	for !done {
		c.Wait()
	}
	fmt.Println(name, "starts reading")
	c.L.Unlock()
}

func write(name string, c *sync.Cond) {
	fmt.Println(name, "starts writing")
	time.Sleep(time.Second)
	done = true
	fmt.Println(name, "wakes all")
	c.Broadcast()
}

func main() {
	cond := sync.NewCond(&sync.Mutex{})

	go read("reader1", cond)
	go read("reader2", cond)
	go read("reader3", cond)
	write("writer", cond)

	time.Sleep(time.Second * 3)
}
```

sync.Cond 不能被复制的原因，并不是因为其内部嵌套了 Locker。NewCond 时传入的 Mutex/RWMutex 指针，对于 Mutex 指针复制是没有问题的。

主要原因是 sync.Cond 内部是维护着一个 Goroutine 通知队列 notifyList。如果这个队列被复制的话，那么在并发场景下导致不同 Goroutine 之间操作的 notifyList.wait、notifyList.notify 并不是同一个，这会导致出现有些 Goroutine 会一直阻塞。

实际上 sync.Cond 与 Channel 是有区别的，channel 定位于通信，用于一发一收的场景，sync.Cond 定位于同步，用于一发多收的场景。虽然 channel 可以通过 close 操作来达到一发多收的效果，但是 closed 的 channel 已无法继续使用，而 sync.Cond 依旧可以继续使用。这可能就是“全能”与“专精”的区别。

### 值拷贝

go中值类型只需要赋值操作就可以进行复制，引用类型会复制指针，所以复制一个结构体就是:=就行

### 切片内存陷阱

如果一直做切片操作，可能导致实际用到的内存很少，但是底层数组大量空间无法释放的问题，可以通过copy来解决，这样会创建新的底层数组，原来的被GC

### 内存对齐

[ref](https://geektutu.com/post/hpg-struct-alignment.html)

CPU 始终以字长访问内存，如果不进行内存对齐，很可能增加 CPU 访问内存的次数

在对内存特别敏感的结构体的设计上，我们可以通过调整字段的顺序，减少内存的占用

### sync.Pool

[ref](https://www.cnblogs.com/qcrao-2018/p/12736031.html)

理解：sync.Pool的目的是在多个goroutine每个都需要一个对象，但是如果每个goroutine都创建对象就浪费的情况，解决这个问题，使得goroutine直接可以从Pool里拿到对象就行，如果没有就会自动创建。

需要注意的点：Get之后Pool里的对象就没了，需要再Put进去，且最好将对象清空之后放进去，因为这样其他goroutine才能安全使用，这里的对象就类似于一组方法集

对于很多需要重复分配、回收内存的地⽅，sync.Pool 是⼀个很好的选择。频繁地分配、回收内存会给 GC 带来⼀定的负担，严重的时候会引起 CPU 的⽑刺，⽽ sync.Pool 可以将暂时不⽤的对象缓存起来，待下次需要的时候直接使⽤，不⽤再次经过内存分配，复⽤对象的内存，减轻 GC 的压⼒，提升系统的性能。

相当于一个临时存储，sync.Pool 的大小是可伸缩的，高负载时会动态扩容，存放在池中的对象如果不活跃了会被自动清理。

### 大端 小端

[ref](https://www.ruanyifeng.com/blog/2022/06/endianness-analysis.html)

大端序的最高位在左边，最低位在右边，符合阅读习惯。所以，对于这些国家的人来说，从左到右的大端序的可读性更好。

但是现实中，从右到左的小端序虽然可读性差，但应用更广泛，x86 和 ARM 这两种 CPU 架构都采用小端序，这是为什么？

或者换一种问法，两种不同的字节序为什么会并存，统一规定只使用一种，难道不是更方便吗？

原因是它们有各自的适用场景，某些场景大端序有优势，另一些场景小端序有优势，下面就逐一分析。

**如果需要逐位运算，或者需要到从个位数开始运算，都是小端序占优势。反之，如果运算只涉及到高位，或者数据的可读性比较重要，则是大端序占优势。**

```go
// 判断是否大端序
var data int32 = 1
pointer := unsafe.Pointer(&data)
bp := (*byte)(pointer)
if *bp == 1 {
	fmt.Println(true)
}
	```