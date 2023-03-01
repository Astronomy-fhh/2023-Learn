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
> chan
- 参考：[chan源码分析](https://liangtian.me/post/go-channel/)
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
