GO “非分代的、非紧缩、写屏障、并发标记清理”

并发清理： 垃圾回收(清理过程)与用户逻辑并发执行 三色并发标记 : 标记与用户逻辑并发执行

## 一般常用垃圾回收方法

- 引用计数

这是最简单的一种垃圾回收算法，和之前提到的智能指针异曲同工。对每个对象维护一个 引用计数 ，当引用该对象的对象被销毁或更新时被引用对象的引用计数自动减一，当被引用对象被创建或被赋值给其他对象时引用计数自动加一。当引用计数为0时则立即回收对象。

#### 优点

是实现简单，并且内存的回收很及时。

#### 缺点

频繁更新引用计数降低了性能 循环引用问题

- 标记-清除

该方法分为两步， 标记 从根变量开始迭代得遍历所有被引用的对象，对能够通过应用遍历访问到的对象都进行标记为“被引用”；标记完成后进行 清除 操作，对没有标记过的内存进行回收（回收同时可能伴有碎片整理操作）。这种方法解决了引用计数的不足，但是也有比较明显的问题：每次启动垃圾回收都会暂停当前所有的正常代码执行，回收是系统响应能力大大降低！当然后续也出现了很多mark&sweep算法的变种（如 三色标记法 ）优化了这个问题。

- 分代收集

经过大量实际观察得知，在面向对象编程语言中，绝大多数对象的生命周期都非常短。分代收集的基本思想是，将堆划分为两个或多个称为 代（generation） 的空间。新创建的对象存放在称为 新生代（young generation） 中（一般来说，新生代的大小会比 老年代 小很多），随着垃圾回收的重复执行，生命周期较长的对象会被 提升（promotion） 到老年代中。因此，新生代垃圾回收和老年代垃圾回收两种不同的垃圾回收方式应运而生，分别用于对各自空间中的对象执行垃圾回收。新生代垃圾回收的速度非常快，比老年代快几个数量级，即使新生代垃圾回收的频率更高，执行效率也仍然比老年代垃圾回收强，这是因为大多数对象的生命周期都很短，根本无需提升到老年代。

## 三色并发标记 (1.5之后使用GC方法)

In a tri-color collector, every object is either white, grey, or black and we view the heap as a graph of connected objects. At the start of a GC cycle all objects are white. The GC visits all roots, which are objects directly accessible by the application such as globals and things on the stack, and colors these grey. The GC then chooses a grey object, blackens it, and then scans it for pointers to other objects. When this scan finds a pointer to a white object, it turns that object grey. This process repeats until there are no more grey objects. At this point, white objects are known to be unreachable and can be reused.

这是让标记与用户代码并发的基本保障， 基本原理： * 起初所有对象都是白色 * 扫描所有可达对象，标记为灰色，放入待处理队列 * 从队列提取灰色对象，将其引用对象标记为灰色放入队列，自身标记为黑色 * 写屏障监控对象内存修改，从新标色或是放入队列

当完成所有的扫描和标记的工作后，剩余不是白色就是黑色，分别代表要回收和活跃对象，清理操作只需要把白色对象回收内存回收就好

![img](https://img1.tuicool.com/INBzE3J.gif)

#### 增量

三色标记的目的，主要是用于做增量的垃圾回收。注意到，如果只有黑色和白色两种颜色，那么回收过程将不能中断，必须一次性完成，期间用户程序是不能运行的。

而使用三色标记，即使在标记过程中对象的引用关系发生了改变，例如分配内存并修改对象属性域的值，只要满足黑色对象不引用白色对象的约束条件，垃圾回收器就可以继续正常工作。于是每次并不需要将回收过程全部执行完，只是处理一部分后停下来，后续会慢慢再次触发的回收过程，实现增量回收。相当于是把垃圾回收过程打散，减少停顿时间。

#### 写屏障 (write barrier)

如果是STW的，三色标记没有什么问题。但是如果允许用户代码跟垃圾回收同时运行，需要维护一条约束条件：

```
黑色对象绝对不能引用白色对象
```

为什么不能让黑色引用白色？因为黑色对象是活跃对象，它引用的对象是也应该属于活跃的，不应该被清理。但是，由于在三色标记算法中，黑色对象已经处理完毕，它不会被重复扫描。那么，这个对象引用的白色对象将没有机会被着色，最终会被误当作垃圾清理。 STW中，一个对象，只有它引用的对象全标记后才会标记为黑色。所以黑色对象要么引用的黑色对象，要么引用的灰色对象。不会出现黑色引用白色对象。 对于垃圾回收和用户代码并行的场景，用户代码可能会修改已经标记为黑色的对象，让它引用白色对象。看一个例子来说明这个问题：

```
stack -> A.ref -> B
```

A是从栈对象直接可达，将它标记为灰色。此时B是白色对象。假设这个时候用户代码执行：

```
localRef = A.ref
A.ref = NULL
```

localRef是栈上面的一个黑色对象，前一行赋值语句使得它引用到B对象。后一行A.ref被置为空之后，A将不再引用到B。A是灰色但是不再引用到B了，B不会着色。localRef是黑色，处理完毕的对象，引用了B但是不会被再次处理。于是B将永远不再有机会被标记，它会被误当作垃圾清理掉！

如果实现满足这种约束条件呢？write barrier! 来自wiki的对这个术语的解释：”A write barrier in a garbage collector is a fragment of code emitted by the compiler immediately before every store operation to ensure that (e.g.) generational invariants are maintained.” 即是说，在每一处内存写操作的前面，编译器会生成的一小段代码段，来确保不要打破一些约束条件。 增量和分代，都需要维护一个write barrier。 先看分代的垃圾回收，跨越不同分代之间的引用，需要特别注意。通常情况下，大多数的交叉引用应该是由新生代对象引用老生代对象。当我们回收新生代的时候，这没有什么问题。但是当我们回收老生代的时候，如果只扫描老生代不扫描新生代，则老生代中的一些对象可能被误当作不可达对象回收掉！为了处理这种情况，可以做一个约定–如果回收老生代，那么比它年轻的新生代都要一起回收一遍。另外一种交叉引用是老生代对象引用到新生代对象，这时就需要write barrier了，所有的这种类型引用都应该记录下来，放到一个集合中，标记的时候要处理这个集合。

再看三色标记中，黑色对象不能引用白色对象。这就是一个约束条件，write barrier就是要维护这条约束。

## go1.5 GC 实现过程

![img](https://img0.tuicool.com/qme2Mru.png!web)

#### Go1.5垃圾回收的实现被划分为五个阶段：

- GCoff 垃圾回收关闭状态
- GCscan 扫描阶段
- GCmark 标记阶段，write barrier生效
- GCmarktermination 标记结束阶段，STW，分配黑色对象
- GCsweep 清扫阶段

![img](https://img0.tuicool.com/fqI7fie.png!web)

#### 控制器

全程参与并发回收任务， 记录相关状态数据， 动态调整运行策略，影响并发标记工作单元的工作模式和数量， 平衡CPU资源占用。当回收结束时，参与next_gc 回收阀值设置，调整垃圾回收触发频率

#### 过程

- 初始化

设置 `gcprecent(GOGC)` 和 `next_gc` 阀值

- 启动

在为对象分配堆内存后， `mallocgo` 函数会检查垃圾回收触发条件，并依照相关状态启动或参与辅助回收 垃圾回收默认以全并发，但可用环境变量或事参数禁用并发标记和并发清理，gc goroutine 一直循环，直到符合触发条件时被唤醒

- 标记

分俩步骤 > 扫描 ：遍历相关内存区域，依照指针标记找出灰色可达对象，加入队列 。扫描函数 (gcscan_m) 启动时，用户代码和标记函数 (MarkWorker) 都在运行 > 标记 ： 将灰色对象从队列中取出，将其应用对象标记为灰色，自身标记为黑色。 并发标记由多个MarkWorker goroutine 共同完成，它们在回收任务完成前绑定到 P ， 然后进入休眠状态，知道被调度器唤醒

- 清理

清理未被标记的白色对象 ，将其内存回收

并发清理本质上是一个死循环，被唤醒后开始执行清理任务。 通过遍历所有span 对象，触发内存回收器的回收操作。任务完成后，再次休眠，等待下次任务

- 监控

模拟情景：服务重启，海量服务重新接入，瞬间分配大量对象，将垃圾回收触发阀值next_gc推到一个很大的值。而当服务正常后，因活跃对象远小于该阀值，造成垃圾回收迟迟无法触发，大量白色对象无法回收，造成隐形内存泄漏。同样情景也有可能由于某个算法在短期内大量使用临时变量造成 。 这个时候只有forcegc介入，才能将next_gc恢复正常， 监控服务sysmon每隔两分钟检查一次垃圾回收状态，如果超过两分钟未曾触发，就会强制执行gc

- gc 过程中几种辅助结构

parfor 并行任务框架 ： 关注的是任务的分配和调度，自身不具备执行能力。它将多个任务分组交给多个执行线程。然后在执行过程中重新平衡线程的任务分配，确保整个任务在最短的时间内完成 缓存队列： workbuf 无锁栈节点，本身是一个缓存容器

### 问题

- go程序内存占用大的问题

我们模拟大量的用户请求访问后台服务，这时各服务模块能观察到明显的内存占用上升。但是当停止压测时，内存占用并未发生明显的下降。花了很长时间定位问题，使用gprof等各种方法，依然没有发现原因。最后发现原来这时正常的…主要的原因有两个，

一是go的垃圾回收有个触发阈值，这个阈值会随着每次内存使用变大而逐渐增大（如初始阈值是10MB则下一次就是20MB，再下一次就成为了40MB…），如果长时间没有触发gc go会主动触发一次（2min）。高峰时内存使用量上去后，除非持续申请内存，靠阈值触发gc已经基本不可能，而是要等最多2min主动gc开始才能触发gc。

第二个原因是go语言在向系统交还内存时只是告诉系统这些内存不需要使用了，可以回收；同时操作系统会采取“拖延症”策略，并不是立即回收，而是等到系统内存紧张时才会开始回收这样该程序又重新申请内存时就可以获得极快的分配速度。

- gc时间长的问题

对于对用户响应事件有要求的后端程序，golang gc时的stop the world兼职是噩梦。根据上文的介绍，1.5版本的go再完成上述改进后应该gc性能会提升不少，但是所有的垃圾回收型语言都难免在gc时面临性能下降，对此我们对于应该尽量避免频繁创建临时堆对象（如&abc{}, new, make等）以减少垃圾收集时的扫描时间，对于需要频繁使用的临时对象考虑直接通过数组缓存进行重用；很多人采用cgo的方法自己管理内存而绕开垃圾收集，这种方法除非迫不得已个人是不推荐的（容易造成不可预知的问题），当然迫不得已的情况下还是可以考虑的，这招带来的效果还是很明显的~

- goroutine泄露的问题

我们的一个服务需要处理很多长连接请求，实现时，对于每个长连接请求各开了一个读取和写入协程，全部采用endless for loop不停地处理收发数据。当连接被远端关闭后，如果不对这两个协程做处理，他们依然会一直运行，并且占用的channel也不会被释放…这里就必须十分注意，在不使用协程后一定要把他依赖的channel close并通过再协程中判断channel是否关闭以保证其退出。

## 如何测量GC

```
$ go build -gcflags "-l" -o test test.go
$ GODEBUG="gctrace=1" ./test

gctrace: setting gctrace=1 causes the garbage collector to emit a single line to standard
error at each collection, summarizing the amount of memory collected and the
length of the pause. Setting gctrace=2 emits the same summary but also
repeats each collection.
```

之前说了那么多，那如何测量gc的之星效率，判断它到底是否对程序的运行造成了影响呢？ 第一种方式是设置godebug的环境变量，比如运行GODEBUG=gctrace=1 ./myserver，如果要想对于输出结果了解，还需要对于gc的原理进行更进一步的深入分析，这篇文章的好处在于，清晰的之处了golang的gc时间是由哪些因素决定的，因此也可以针对性的采取不同的方式提升gc的时间：

根据之前的分析也可以知道，golang中的gc是使用标记清楚法，所以gc的总时间为：

Tgc = Tseq + Tmark + Tsweep(T表示time)

Tseq表示是停止用户的 goroutine 和做一些准备活动（通常很小）需要的时间 Tmark 是堆标记时间，标记发生在所有用户 goroutine 停止时，因此可以显著地影响处理的延迟 Tsweep 是堆清除时间，清除通常与正常的程序运行同时发生，所以对延迟来说是不太关键的 之后粒度进一步细分，具体的概念还是有些不太懂：

与Tmark相关的：1 垃圾回收过程中，堆中活动对象的数量，2 带有指针的活动对象占据的内存总量 3 活动对象中的指针数量。 与Tsweep相关的：1 堆内存的总量 2 堆中的垃圾总量

## 如何进行gc调优（gopher大会 Danny）

硬性参数

涉及算法的问题，总是会有些参数。GOGC参数主要控制的是下一次gc开始的时候的内存使用量。

比如当前的程序使用了4M的对内存（这里说的是堆内存），即是说程序当前reachable的内存为4m，当程序占用的内存达到reachable*(1+GOGC/100)=8M的时候，gc就会被触发，开始进行相关的gc操作。

如何对GOGC的参数进行设置，要根据生产情况中的实际场景来定，比如GOGC参数提升，来减少GC的频率。

### 参考

1. [go15gc](https://blog.golang.org/go15gc)
2. https://talks.golang.org/2015/go-gc.pdf
3. http://dave.cheney.net/tag/godebug
4. [1.5源码分析](https://github.com/qyuhen/book)
5. [golang gc 探究](http://www.open-open.com/lib/view/open1435846881544.html)
6. [golang gc 基本知识](http://wangzhezhe.github.io/blog/2016/04/30/golang-gc/)
7. [go 性能调试问题](http://www.oschina.net/translate/debugging-performance-issues-in-go-programs)