# 2023-Learn
## golang
### Go系统调度
[操作系统基础](https://xiaolincoding.com/os/#%E5%B0%8F%E7%99%BD%E9%80%82%E5%90%88%E7%9C%8B%E5%90%97)

[操作系统调度](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part1.html)

[Go调度](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part2.html)

[并发](https://www.ardanlabs.com/blog/2018/08/scheduling-in-go-part3.html)

### GoGC
[9张图轻松吃透Go内存管理单元](https://mp.weixin.qq.com/s?__biz=MzA5MDEwMDYyOA==&mid=2454620147&idx=1&sn=0cf6b70a3dd47e8288701183d91649e8&chksm=87aae108b0dd681e46c2616958c0a6a8fecd9ebbd2b728ef3a1cd43e9f38e3ba5e27951e0dae&scene=21#wechat_redirect)

[堆和栈](https://cloud.tencent.com/developer/news/731210)

[内存分配](https://juejin.cn/post/6844903795739082760)

[GoGC](https://www.cnblogs.com/luozhiyun/p/14564903.html)

[GoGC](https://golang.design/under-the-hood/zh-cn/part2runtime/ch08gc/barrier/)


### 整理
#### goGC
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
    - Dijkstra写屏障：当一个指针被写入时，指向的对象标记为灰色。在下一次扫描中加入扫描队列。因为所有的根对象都会被标记为活动对象，栈对象也是根对象，因为栈对象频繁的写屏障开销，go在1.7之后将栈对象标记为恒灰，并在标记终止阶段STW对栈对象重新扫描。
    - Yuasa写屏障：一种删屏障技术，在删除对象时，会检测被删除对象是否包含是否包含指向其他对象的引用，如果包含，则将其他对象标记为灰色，同时会将引用关系记录到一个数据结构中。以便垃圾回收器在扫描时能正确的处理。


