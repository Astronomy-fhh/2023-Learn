# 2023-Learn
## golang
### Go系统调度
[操作系统基础](https://xiaolincoding.com/os/#%E5%B0%8F%E7%99%BD%E9%80%82%E5%90%88%E7%9C%8B%E5%90%97)
[操作系统调度](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)
[Go调度](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)
[并发](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part3.html)
[操作系统-内存](https://xiaolincoding.com/os/3_memory/vmem.html#%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98)
[Go并发机制,多线程模式](https://www.alibabacloud.com/forum/read-916)
[Golang 调度器 GMP 原理与调度全分析](https://learnku.com/articles/41728)
[goroutine如何工作](https://medium.com/the-polyglot-programmer/what-are-goroutines-and-how-do-they-actually-work-f2a734f6f991)

### GoGC
[Go内存管理单元mspan](https://mp.weixin.qq.com/s?__biz=MzA5MDEwMDYyOA==&mid=2454620147&idx=1&sn=0cf6b70a3dd47e8288701183d91649e8&chksm=87aae108b0dd681e46c2616958c0a6a8fecd9ebbd2b728ef3a1cd43e9f38e3ba5e27951e0dae&scene=21#wechat_redirect)
[堆和栈](https://cloud.tencent.com/developer/news/731210)
[内存分配](https://juejin.cn/post/6844903795739082760)
[GoGC](https://www.cnblogs.com/luozhiyun/p/14564903.html)
[GoGC](https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/barrier/)
[视频](https://www.youtube.com/playlist?list=PL_GrAPKmuajz6T5EBXGbEgx9LciuuryHD)



---
---
---

### 整理
> goGC
- 三色标记：黑白灰
  - 黑色：该对象已被标记过，不需要回收，且以下的对象也被标记过
  - 灰色：该对象被标记，但以下的对象还没被标记完
  - 白色：该对象没有被标记，需要被回收。
  
- 三色标记存在的问题：
  - 多标-垃圾浮动问题：已标记的灰色或黑色，被前面引用的黑色断开，断开的其实是无用的，但这轮GC不会再被扫描到。
  - 漏标-悬挂指针问题：已经标记为灰色的引用了新的白色。由于黑色不会再被扫描，所以白色会被清理。

-  解决以上问题提出的思路：
    - 强三色不变性：黑色对象只能指向黑色或灰色对象。
    - 弱三色不变性：如果黑色对象指向了白色对象，那么从灰色对象出发，必然能找到一条灰色到白色的路径。

- 内存屏障：
    - 特定顺序的内存操作（排除多线程或指令重排的影响）。
    - Dijkstra写屏障：对象被引用时触发的机制，A对象引用B对象，B被标记为灰色,满足强三色不变性。栈对象也是根对象，因为栈对象频繁的写屏障开销，go在1.7之后将栈对象标记为恒灰，并在标记终止阶段STW对栈对象重新扫描，需要10-100ms。
    - Yuasa删除屏障：对象被删除时触发的机制，被删除的对象如果是灰色或黑色，就被标记为灰色，满足弱三色不变性。一个被删除的对象依旧可以活过这一轮。
 - v1.8三色标记+混合写屏障：
    - GC开始将栈上的对象标记为黑色（之后不再扫描），GC期间，任何在栈上创建的对象都标记为黑色，被删除的和被添加的对象标记为灰色。满足弱三色不变性
--- 
> Go系统调度
- GO启动时，会为每个虚拟核心分配一个逻辑处理器P，例如一个4核处理器，每个物理内核有2个硬件线程，则GO认为有8个虚拟内核可并行执行OS线程。
- 每个P都分配一个操作系统线程M，M代表机器，这个线程由OS管理。
- 每个GO程序初始会有一个Goroutine(G),协程（coroutine）。Go协程可以认为应用程序级线程。
- GO调度器(Sched)有两种运行队列。全局(GRQ)本地(LRQ)，每个P都有一个LRQ。
- GO调度器的实现不是抢占调度，而是协作式调度。
- 协程状态：Waiting,Runnable,Executing。
- 任务切换
  - 对于一个异步任务G1，当G1要进行网络系统调用，调度系统会将G1移至网络轮询器，处理异步网络调用，可执行LRQ的G2。
  - 对于一个同步任务G1，会阻塞当前M1，调度程序识别到阻塞，会将M1从P中分离出来，然后引用M2服务P，G1还是连接着M1。
  - 窃取：当P1的LRQ没有G可执行，P1会从P2的LRQ中窃取一半的G来执行，如果P2没有任何G，则会从GRQ中窃取。
---
> Goroutine相比系统线程的优势
- 线程具有较大的堆栈大小>1MB, 线程上下文切换需要恢复大量寄存器。线程设置拆卸需要调用操作系统来获取资源。
- gorounter的堆栈更小2KB，可调整大小的有界堆栈，上下文切换代价更小，gorounter的运行时调度是协调式调度，不是抢占式调度，例如网络或磁盘io阻塞等切换M以及P上任务的窃取。
---
> go内存
- 内存管理单元span
  - 由N个连续page组成，N值相同的mspan组成链表，一个mspan会被拆分成更小粒度object组成的freeList(没有next属性，内存的前8字节存储下个节点的指针)
  - object的具体大小由sizeClass决定，sizeClass是一个映射链表（8b-32kb 67种）（sizeClass大小-object大小-由几个page组成）。
  - mspan分成两类，需要垃圾回收和不需要垃圾回收scan/noscan, mspan的spanClass属性存储sizeClass类型和scan类型

- 内存分配
  - TCMalloc 多级内存管理。
  - mcache:每个工作线程的独有,动态的从mcentral中申请，当对象大小小于等于32kb时，使用mcache的mspan进行分配。
  - mcentral:所有工作线程共同享有，为mcache提供切分好的mspan资源，每个mcentral保存一种特定大小的mspan,包含已分配出去的和未分配出去的。
  - mheap:代表程序持有的所有堆空间，当mcentral没有空闲mspan时。会想mheap申请，mheap没有时，会向操作系统申请，mheap主要用于大对象的分配，以及管理未切割的mspan。 结构：spans-bigmap-arnea:arnea区mspan的指针-标识arnea区域的信息-存放span。
--- 
> 并行和并发
  - 操作系统中，线程是处理器调度和分配的基本单位，进程是资源所有权的基本单位。
  - 每个进程都包含私有的虚拟地址空间，代码数据和其他系统资源。
  - 每个进程至少有一个主执行线程。多个线程可在同一进程中并发运行。
  - 并发：一个时间段内有很多进程或线程在执行，但在任何时间地点只有一个进程或线程在执行，多个线程争夺时间片交替执行。
  - 并行：在同一世界段内，同一时间有多个线程或进程在执行。
  - 线程分类：用户级线程ULT,内核级线程KLT。
---
> channel
- 参考：[源码分析](https://liangtian.me/post/go-channel/)
- send
  - chan已关，send会 panic: send on closed channel
  - buff已满/无buff,没有recvq !block send会 fatal：all goroutines are asleep
  - buff已满/无buff,没有recvq  block send会block
- recv
  - chan已关，buff空，recv会收到默认值 0/nil这种
  - chan已关，buff非空，recv会收到有效值
  - buff空,没有sendq !block recv 会 fatal：all goroutines are asleep
  - buff空,没有sendq  block recv 会 block
  ---
  
> sync.Pool 
- 参考：[源码分析](https://www.cnblogs.com/qcrao-2018/p/12736031.html)
- 作用：
  - 增加临时对象的利用率，减少GC负担。
- 原理
  - 对于一个新创建的pool, 会创建 make([]poolLocal, GOMAXPROCS)，也就是每个P上都对应一个poolLocal
  - poolLoacl有private, private只存一个值，shared是一个双向链表，每个双向链表项是一个循环链表，初始容量是8，*2扩容，最大是2的30次方。
  - get元素，先拿private,如果没有 自己popHead,如果还没有 就other P popTail,如果还没有，就从victim cache中拿。
  - push元素，private为空，则赋值，否则pushHead。
  - 清理：STW阶段，会将local和victim交换，这样就算获取对象的速度下降了，这些对象也能保留两个GC周期。

- 理念参考
  - 原子操作代替锁，比如链表的pop push操作。
  - 目标隔离，特点的P访问特点的Local。
  - 行为隔离，生产者访问head,消费者访问tail。
- 不适用：
  - 像 socket 长连接或数据库连接池这种固定数量或强状态性的
  ---
  
> sync.Mutex
- 参考：[源码分析](https://www.cnblogs.com/ricklz/p/14535653.html)
- 结构：
  - state int32 分4部分 waiter-29位、strving-1位饥饿标识、Worken-1位唤醒标识、Lock-1位锁状态标识
  - seme uint32 信号量，用于唤醒goroutine
- 原理：
  - 正常模式下，如果锁是锁定状态且非饥饿模式，则会尝试自旋获取锁（受自旋次数限制，单P不适合自旋）。
  - 自旋未获取到锁，则会进入睡眠，被存在一个先进先出队列中，等待依次被唤醒。
  - 如果已经是饥饿状态，则不会自旋，直接加入到队列的尾部。
  - 如果一个协程等待获取锁的时间超过1ms,则被标记为饥饿状态，会追加到队列的头部，优先被唤醒。
- 为什么有饥饿模式
  - 新到达的协程正在运行，比如自旋转中，比刚唤醒协程更容易获得锁的使用权。

---
> sync.Conc
- 参考：[源码分析](https://www.cyhone.com/articles/golang-sync-cond/)
- 用于等待某个条件成立后唤醒一个或一组协程。
- 注意wait的调用一定要放在lock和unlock之间。
- 用法
```go
    c.L.Lock()
    for !condition() {
           c.Wait()
    }
    ... make use of condition ...
    c.L.Unlock()
```
---
> sync.rwmutex
- 参考：[源码分析](https://juejin.cn/post/6968853664718913543)
--- 
> sync.waitgroup
- 参考：[源码分析](https://juejin.cn/post/7005565255736639524)
---
> 原子操作
- 参考：[原子操作汇编原理+CAS+atomic.value.Store/Load](https://www.cnblogs.com/ricklz/p/13648859.html)
---
> 信号量
> 参考：[源码](https://github.com/golang/sync) [源码分析](https://segmentfault.com/a/1190000039710281)
---
> nocopy

- 参考：[nocopy不可复制的实现](https://blog.csdn.net/yzf279533105/article/details/97640423)
- 代码：
```go
type noCopy struct{}
// Lock is a no-op used by -copylocks checker from `go vet`.
func (*noCopy) Lock()   {}
func (*noCopy) Unlock() {}
```
- 阐述：waitGroupl里有一个nocopy属性，想要自己实现，按照上面代码部分实现即可，这样包含nocopy属性的结构体，就不能复制了，编译运行都不会影响，只有go vet才会提示nocopy问题。
---
> go vet工具
- 参考：[go vet发现的问题的示例](https://studygolang.com/articles/9619)
- 静态代码检查，以发现可能的bug或者可疑的构造。
---
> array slice
- 参考：[切面使用示例](https://www.geeksforgeeks.org/how-to-split-a-slice-of-bytes-in-golang/?ref=lbp) [部分源码解释](http://lifegoeson.cn/2022/01/22/golang%20slice%E6%89%A9%E5%AE%B9%E6%9C%BA%E5%88%B6/)
- 数组和slice的区别
  - 数组固定长度，切片slice动态大小
  - 如何区分
    - var arr1 []int，无长度定义，则是Slice而且是nil slice
    - arr2 := make([]int, 5)，make定义一定是slice
    - arr3 := []int{}，无长度定义，则是Slice
    - var arr4 [5]int，有长度定义，则是Array

- 原理:以go v1.20
  - array:unsafe.Pointer、len:int、cap:int 组成
  - 容量增加：源码参考：runtime.slice.go:growslice。
  - 如果期望容量大于翻倍后的容量：则使用期望容量。
  - 如果期望容量小于翻倍后的容量：
    - 如果当前容量小于256，则使用翻倍后的容量。
    - 如果当前容量不小于256，则容量以将近1.25倍的大小增加，直到新容量大于等于期望容量 newcap += (newcap + 3*threshold) / 4。
  - 新容量计算后，需要进行内存对齐处理，_type=1/指针大小/2的倍数、默认 有不同的实现。对齐后，确认最终的新容量。
  
- 其他知识：
   - 内存对齐：内存对齐是为了提高内存读写操作的效率。计算机的内存访问通常是按照对齐单位的整数倍来进行的，如果数据没有按照对齐规则存储，就会导致内存访问时需要进行多次访问才能读取完整的数据，从而影响程序的性能。
   - TrailingZeros64(8)： et.size 是 8，其二进制表示为 00001000，那么该函数将返回值为 3，表示末尾有 3 个零位。

---
> 值传递 引用传递
- 参考：[值传递引用传递 slice map传递的细节](https://zhuanlan.zhihu.com/p/542218435)
---
