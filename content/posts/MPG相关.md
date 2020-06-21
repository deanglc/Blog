---
title: "MPG.md"
description: "> https://github.com/golang/go/blob/master/src/runtime/HACKING.md 待翻译"
tags: [ "golang", ]
categories: [ "golang", ]
keywords: [ "golang", "goroutine" ]
isCJKLanguage: true

date: 2020-06-21T11:00:53+08:00
draft: false
---


MPG 链接：https://www.zhihu.com/question/20862617/answer/36191625

要理解这个事儿首先得了解操作系统是怎么玩线程的。一个线程就是一个栈加一堆资源。操作系统一会让cpu跑线程A，一会让cpu跑线程B，靠A和B的栈来保存A和B的执行状态。每个线程都有他自己的栈。
但是线程又老贵了，花不起那个钱，所以go发明了goroutine。大致就是说给每个goroutine弄一个分配在heap里面的栈来模拟线程栈。比方说有3个goroutine，A,B,C，就在heap上弄三个栈出来。然后Go让一个单线程的scheduler开始跑他们仨。相当于 { A(); B(); C() }，连续的，串行的跑。
和操作系统不太一样的是，操作系统可以随时随地把你线程停掉，切换到另一个线程。这个单线程的scheduler没那个能力啊，他就是user space的一段朴素的代码，他跑着A的时候控制权是在A的代码里面的。A自己不退出谁也没办法。
所以A跑一小段后需要主动说，老大（scheduler），我不想跑了，帮我把我的所有的状态保存在我自己的栈上面，让我歇一会吧。这时候你可以看做A返回了。A返回了B就可以跑了，然后B跑一小段说，跑够了，保存状态，返回，然后C再跑。C跑一段也返回了。
这样跑完{A(); B(); C()}之后，我们发现，好像他们都只跑了一小段啊。所以外面要包一个循环，大致是： 

```python
goroutine_list = [A, B, C]
while(goroutine):
  for goroutine in goroutine_list:
    r = goroutine()
    if r.finished():
      goroutine_list.remove(r)
```

比如跑完一圈A，B，C之后谁也没执行完，那么就在回到A执行一次。由于我们把A的栈保存在了HEAP里，这时候可以把A的栈复制粘贴会系统栈里（我很确定真实情况不是这么玩的，会意就行），然后再调用A，这时候由于A是跑到一半自己说跳出来的，所以会从刚刚跳出来的地方继续执行。比如A的内部大致上是这样

```text
def A:
  上次跑到的地方 = 找到上次跑哪儿了
  读取所有临时变量
  goto 上次跑到的地方
  a = 1
  print("do something")
  go.scheduler.保存程序指针 // 设置"这次跑哪儿了"
  go.scheduler.保存临时变量们
  go.scheduler.跑够了_换人 //相当于return
  print("do something again")
  print(a)
```

第一次跑A，由于这是第一次，会打印do something，然后保存临时变量a，并保存跑到的地方，然后返回。再跑一次A，他会找到上次返回的地方的下一句，然后恢复临时变量a，然后接着跑，会打印“do something again"和1

所以你看出来了，这个关键就在于每个goroutine跑一跑就要让一让。一般支持这种玩意（叫做coroutine）的语言都是让每个coroutine自己说，我跑够了，换人。goroutine比较文艺的地方就在于，他可以来帮你判断啥时候“跑够了”。

其中有一大半就是靠的你说的“异步并发”。go把每一个能异步并发的操作，像你说的文件访问啦，网络访问啦之类的都包包好，包成一个看似朴素的而且是同步的“方法”，比如string readFile（我瞎举得例子）。但是神奇的地方在于，这个方法里其实会调用“异步并发”的操作，比如某操作系统提供的asyncReadFile。你也知道，这种异步方法都是很快返回的。
所以你自己在某个goroutine里写了

```text
string s = go.file.readFile("/root")
```


其实go偷偷在里面执行了某操作系统的API asyncReadFIle。跑起来之后呢，这个方法就会说，我当前所在的goroutine跑够啦，把刚刚跑的那个异步操作的结果保存下下，换人：

```text
// 实际上
handler h = someOS.asyncReadFile("/root") //很快返回一个handler
while (!h.finishedAsyncReadFile()): //很快返回Y/N
  go.scheduler.保存现状()
  go.scheduler.跑够了_换人() // 相当于return，不过下次会从这里的下一句开始执行
string s = h.getResultFromAsyncRead()
```



然后scheduler就换下一个goroutine跑了。等下次再跑回刚才那个goroutine的时候，他就看看，说那个asyncReadFile到底执行完没有啊，如果没有，就再换个人吧。如果执行完了，那就把结果拿出来，该干嘛干嘛。所以你看似写了个同步的操作，已经被go替换成异步操作了。

还有另外一种情况是，某个goroutine执行了某个不能异步调用的会blocking的系统调用，这个时候goroutine就没法玩那种异步调用的把戏了。他会把你挪到一个真正的线程里让你在那个县城里等着，他接茬去跑别的goroutine。比如A这么定义

```python
def A:
  print("do something")
  go.os.InvokeSomeReallyHeavyAndBlockingSystemCall()
  print("do something 2")
```

go会帮你转成

```python
def 真实的A:
  print("do something")
  Thread t = new Thread( () => {
    SomeReallyHeavyAndBlockingSystemCall();
  })
  t.start()
  while !t.finished():
    go.scheduler.保存现状
    go.scheduler.跑够了_换人
  print("finished")
```

所以真实的A还是不会blocking，还是可以跟别的小伙伴(goroutine)愉快地玩耍（轮流往复的被执行），但他其实已经占了一个真是的系统线程了。

当然会有一种情况就是A完全没有调用任何可能的“异步并发”的操作，也没有调用任何的同步的系统调用，而是一个劲的用CPU做运算（比如用个死循环调用a++）。在早期的go里，这个A就把整个程序block住了。后面新版本的go好像会有一些处理办法，比如如果你A里面call了任意一个别的函数的话，就有一定几率被踢下去换人。好像也可以自己主动说我要换人的，可以去查查新的go的spec
另外，请不要在意语言细节，技术细节。会意即可



---

###Golang 的 goroutine 是如何实现的？

链接：https://www.zhihu.com/question/20862617/answer/27964865


[The Go scheduler](https://link.zhihu.com/?target=http%3A//morsmachine.dk/go-scheduler) 纯翻译如下：

Go runtime的调度器：
在了解Go的运行时的scheduler之前，需要先了解为什么需要它，因为我们可能会想，OS内核不是已经有一个线程scheduler了嘛？
熟悉POSIX API的人都知道，POSIX的方案在很大程度上是对Unix process进场模型的一个逻辑描述和扩展，两者有很多相似的地方。 Thread有自己的信号掩码，CPU affinity等。但是很多特征对于Go程序来说都是累赘。 尤其是context上下文切换的耗时。另一个原因是Go的垃圾回收需要所有的goroutine停止，使得内存在一个一致的状态。垃圾回收的时间点是不确定的，如果依靠OS自身的scheduler来调度，那么会有大量的线程需要停止工作。 

单独的开发一个GO得调度器，可以是其知道在什么时候内存状态是一致的，也就是说，当开始垃圾回收时，运行时只需要为当时正在CPU核上运行的那个线程等待即可，而不是等待所有的线程。

用户空间线程和内核空间线程之间的映射关系有：N:1,1:1和M:N
N:1是说，多个（N）用户线程始终在一个内核线程上跑，context上下文切换确实很快，但是无法真正的利用多核。
1：1是说，一个用户线程就只在一个内核线程上跑，这时可以利用多核，但是上下文switch很慢。
M:N是说， 多个goroutine在多个内核线程上跑，这个看似可以集齐上面两者的优势，但是无疑增加了调度的难度。

![img](https://pic1.zhimg.com/2f5c6ef32827fb4fc63c60f4f5314610_b.jpg)![img](https://pic1.zhimg.com/80/2f5c6ef32827fb4fc63c60f4f5314610_1440w.jpg)


Go的调度器内部有三个重要的结构：M，P，S
M:代表真正的内核OS线程，和POSIX里的thread差不多，真正干活的人
G:代表一个goroutine，它有自己的栈，instruction pointer和其他信息（正在等待的channel等等），用于调度。
P:代表调度的上下文，可以把它看做一个局部的调度器，使go代码在一个线程上跑，它是实现从N:1到N:M映射的关键。

![img](https://pic1.zhimg.com/67f09d490f69eec14c1824d939938e14_b.jpg)![img](https://pic1.zhimg.com/80/67f09d490f69eec14c1824d939938e14_1440w.jpg)


图中看，有2个物理线程M，每一个M都拥有一个context（P），每一个也都有一个正在运行的goroutine。
P的数量可以通过GOMAXPROCS()来设置，它其实也就代表了真正的并发度，即有多少个goroutine可以同时运行。
图中灰色的那些goroutine并没有运行，而是出于ready的就绪态，正在等待被调度。P维护着这个队列（称之为runqueue），
Go语言里，启动一个goroutine很容易：go function 就行，所以每有一个go语句被执行，runqueue队列就在其末尾加入一个
goroutine，在下一个调度点，就从runqueue中取出（如何决定取哪个goroutine？）一个goroutine执行。



为何要维护多个上下文P？因为当一个OS线程被阻塞时，P可以转而投奔另一个OS线程！
图中看到，当一个OS线程M0陷入阻塞时，P转而在OS线程M1上运行。调度器保证有足够的线程来运行所以的context P。

![img](https://pic3.zhimg.com/f1125f3027ebb2bd5183cf8c9ce4b3f2_b.jpg)![img](https://pic3.zhimg.com/80/f1125f3027ebb2bd5183cf8c9ce4b3f2_1440w.jpg)


图中的M1可能是被创建，或者从线程缓存中取出。



当MO返回时，它必须尝试取得一个context P来运行goroutine，一般情况下，它会从其他的OS线程那里steal偷一个context过来，
如果没有偷到的话，它就把goroutine放在一个global runqueue里，然后自己就去睡大觉了（放入线程缓存里）。Contexts们也会周期性的检查global runqueue，否则global runqueue上的goroutine永远无法执行。

![img](https://pic2.zhimg.com/31f04bb69d72b72777568063742741cd_b.jpg)![img](https://pic2.zhimg.com/80/31f04bb69d72b72777568063742741cd_1440w.jpg)


另一种情况是P所分配的任务G很快就执行完了（分配不均），这就导致了一个上下文P闲着没事儿干而系统却任然忙碌。但是如果global runqueue没有任务G了，那么P就不得不从其他的上下文P那里拿一些G来执行。一般来说，如果上下文P从其他的上下文P那里要偷一个任务的话，一般就‘偷’run queue的一半，这就确保了每个OS线程都能充分的使用。



---

### 为什么协程切换的代价比线程切换低? https://www.zhihu.com/question/308641794

核心在于，线程切换需要借助内核完成，意味着一次用户态到内核态的切换，以及一次内核态到用户态的切换。而协程的切换只在用户态就可以完成，无需借助内核，也就不需要进入内核态。

用户态和内核态的切换才是最主要的开销。

---

[http://interview.wzcu.com/Golang/goroutine.html#goroutine-%E5%92%8C-thread-%E7%9A%84%E5%8C%BA%E5%88%AB](http://interview.wzcu.com/Golang/goroutine.html#goroutine-和-thread-的区别)

---

#GO夜读:https://www.youtube.com/watch?v=98pIzaOeD2k

**调度的机制用一句话描述：**
runtime准备好G,P,M，然后M绑定一个P，最开始创建g0，然后调度g0，通过g0创建G，M从各种队列中获取G，在汇编代码层面上切换到G的执行栈上并执行G上的任务函数，执行完成后，调用goexit()（事前被放入了G的pc计数器，所以return后进入）做清理工作并回到M，M重新在队列中寻找G，如此反复。
运行函数 schedule() --找G--> execute(g) --执行G，gogo(g)在汇编代码层面上真正执行G-->goexit() --清理工作，重新将g0加入P的空闲队列-->schedule()

# 基本概念

------

#### M（machine）

- M代表着真正的执行计算资源，可以认为它就是os thread（系统线程）。每个M绑定一个 kernel。
- M是真正调度系统的执行者，总是从各种队列（全局队列，本局队列等）中找到可运行的G，而且这样M的可以同时存在多个。
- M在绑定有效的P后，进入调度循环，而且M并不保留G状态，这是G可以跨M调度的基础。

#### P（processor）

- P表示逻辑processor，是线程M的执行的上下文。
- P的最大作用是其拥有的各种G对象队列、链表、cache和状态。

#### G（goroutine）

- 调度系统的最基本单位goroutine，存储了goroutine的执行stack信息、goroutine状态以及goroutine的任务函数等。
- 在G的眼中只有P，P就是运行G的“CPU”。
- 相当于两级线程

M 的状态很少，G最多。一开始的Go是只有M和G的，但是存在很多的全局锁，导致性能很慢，后来加了P，有了本地队列，减少了锁。一个P底下最多有256个G(本地队列的长度)

## 线程实现模型



```ruby
来自Go并发编程实战
                    +-------+       +-------+      
                    |  KSE  |       |  KSE  |          
                    +-------+       +-------+      
                        |               |                       内核空间
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -        
                        |               |                       用户空间
                    +-------+       +-------+
                    |   M   |       |   M   |
                    +-------+       +-------+
                  |          |         |          |
              +------+   +------+   +------+   +------+            
              |   P  |   |   P  |   |   P  |   |   P  |
              +------+   +------+   +------+   +------+   
           |     |     |     |     |     |     |     |     | 
         +---+ +---+ +---+ +---+ +---+ +---+ +---+ +---+ +---+ 
         | G | | G | | G | | G | | G | | G | | G | | G | | G | 
         +---+ +---+ +---+ +---+ +---+ +---+ +---+ +---+ +---+ 
```

KSE（Kernel Scheduling Entity）是内核调度实体
M与P，P与G之前的关联都是动态的，可以变的

## 关系示意图



```ruby
来自golang源码剖析
                            +-------------------- sysmon ---------------//------+ 
                            |                                                   |
                            |                                                   |
               +---+      +---+-------+                   +--------+          +---+---+
go func() ---> | G | ---> | P | local | <=== balance ===> | global | <--//--- | P | M |
               +---+      +---+-------+                   +--------+          +---+---+
                            |                                 |                 | 
                            |      +---+                      |                 |
                            +----> | M | <--- findrunnable ---+--- steal <--//--+
                                   +---+ 
                                     |
                                   mstart
                                     |
              +--- execute <----- schedule 
              |                      |   
              |                      |
              +--> G.fn --> goexit --+ 


              1. go func() 语气创建G。
              2. 将G放入P的本地队列（或者平衡到全局全局队列）。
              3. 唤醒或新建M来执行任务。
              4. 进入调度循环
              5. 尽力获取可执行的G，并执行
              6. 清理现场并且重新进入调度循环
```

上图的 schedule 循环，是调度循环，是不会停止的，通过环内的函数不断进行互相调用，而一直执行下去。

# 必须了解的思想

------

#### Worker thread parking/unparking

- 涉及到 m 的 spinning 和 unspinning 状态
- 涉及到 gorotine ready 时候的操作



```go
    我们需要在`保持足够的 running（运行中） 工作线程以利用可用的硬件并行性` 与`停放过多的运行中工作线程以节省CPU资源和功耗`之间进行权衡。
    这并不简单，原因有二：（1）调度状态（scheduler state）是有意分配的（特别是针对每个P的工作队列），因此无法在 fast paths 上计算全局预测； 
                        （2）为了实现最佳的线程管理，我们需要知道未来的情况（当一个 goroutine 不久将被 readied 时，则不需要停靠工作线程）。

    三种被弃用的效果很差的方法：
        1.集中所有调度状态（scheduler state）
            将抑制可伸缩性）。
        2.直接切换goroutine。 
            也就是说，当我们准备好一个新的goroutine并且当前有一个备用P时，释放一个线程，并将该线程和goroutine转交给P。
            这将导致线程状态冲突（thread state thrashing），因为准备好goroutine的线程可能下一刻就停止了工作，我们需要 park 该线程。
            另外，由于我们要在同一线程上保留依赖的goroutine，它将破坏计算的局部性。 并引入额外的延迟。
        3.每当我们准备好goroutine并且有一个空闲的P时，unpark 一个附加线程，但不进行切换。 
            这将导致过多的线程 parking/unparking，因为附加线程没有发现要执行的工作将立即 park。

    当前方法：准备好一个goroutine时，如果当前有一个空闲 P 且没有“spinning”工作线程（即处于 spinning 状态的 M），则我们 unpark 一个附加线程。
            （“spinning”指一个工作线程 M 完成了本地工作，并且在全局 run queue / netpoller 中均未找到工作）
             the spinning state 用 m.spinning 和 sched.nmspinning 来表示，前者表示 M 是否在 spinning状态，后者表示在 spinning 状态的 M 个数。

             通过上述方式 unpark 的线程也被认为是 spinning 状态的。此时我们不执行goroutine切换，因此此类线程最初是没有工作的。
             “spinning”线程在 park 之前，会在 P 的运行队列中寻找工作。如果 spinning 线程找到工作，它将退出 spinning state 并继续执行工作。如果找不到工作，它将退出 spinning state，然后 park。
    如果至少有一个“spinning”线程（sched.nmspinning> 1），则在准备goroutine时我们不会 unpark 新线程。为了弥补这一点，如果最后一个“spinning”线程找到了工作并停止“spinning”，则必须 unpark 一个新的“spinning”线程。
    这种方法可以消除线程 unparking 中的不合理的峰值，但同时可以保证最终的最大CPU并行利用率。

    实现的主要复杂之处在于，我们在线程从 spinning->non-spinning 过渡时，需要非常小心。 这种过渡可能会与新goroutine的提交相互竞争，同时一部分或另一部分需要 unpark 另一个工作线程。如果它们俩都失败了，那么我们可能会导致半永久性的CPU利用率不足。
    goroutine准备的一般模式是：将goroutine提交到本地工作队列，＃StoreLoad-style memory barrier，检查sched.nmspinning。
    spinning->non-spinning 过渡的一般模式是：递减nmspinning，＃StoreLoad-style memory barrier，检查所有 P 的本地工作队列中是否有新工作。
    请注意，所有这些复杂性都不适用于全局运行队列，因为在提交到全局队列时，我们对线程的 unparking 并不草率。 另请参见有关nmspinning操作的注释。
```

# GPM的来由

------

g0和m0是在proc.go文件中的两个全局变量
m0：进程启动后的初始线程
g0：代表着初始线程的stack
asm_amd64.go --> runtime·rt0_go(SB)



```cpp
    // 程序刚启动的时候必定有一个线程启动（主线程）
    // 将当前的栈和资源保存在g0
    // 将该线程保存在m0
    // tls: Thread Local Storage
    // set the per-goroutine and per-mach "registers"
    get_tls(BX)
    LEAQ    runtime·g0(SB), CX
    MOVQ    CX, g(BX)
    LEAQ    runtime·m0(SB), AX

    // save m->g0 = g0
    MOVQ    CX, m_g0(AX)
    // save m0 to g0->m
    MOVQ    AX, g_m(CX)
```

## M的一生

#### M的创建

```
proc.go
```



```swift
// Create a new m. It will start off with a call to fn, or else the scheduler.
// fn needs to be static and not a heap allocated closure.
// May run with m.p==nil, so write barriers are not allowed.
//go:nowritebarrierrec
// 创建一个新的m，它将从fn或者调度程序开始
func newm(fn func(), _p_ *p) {
    // 根据fn和p和绑定一个m对象
    mp := allocm(_p_, fn)
    // 设置当前m的下一个p为_p_
    mp.nextp.set(_p_)
    mp.sigmask = initSigmask
    ...
    // 真正的分配os thread
    newm1(mp)
}
```



```go
func newm1(mp *m) {
    // 对cgo的处理
    ...
    execLock.rlock() // Prevent process clone.
    // 创建一个系统线程
    newosproc(mp, unsafe.Pointer(mp.g0.stack.hi))
    execLock.runlock()
}
```

#### M的状态

sched.nmspinning 保存 spinning 的 m 个数

| m.spinning | value | 含义                                              |
| :--------: | ----- | ------------------------------------------------- |
|  spinning  | true  | m is out of work and is actively looking for work |
| unspinning | false | m is working                                      |



```ruby
       mstart
          |
          v            找不到可执行任务,gc STW,
      +----------+     任务执行时间过长,系统阻塞等   +----------+
      | spinning | ------------------------------> |unspinning| 
      +----------+            mstop                +----------+
          ^                                          ^
          |                                          |
      notewakeup <-----------------------------  notesleep
```

#### M的问题

[M的问题](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fgolang%2Fgo%2Fissues%2F14592)
线程不会被释放，即便不用

## P的一生

#### P的创建

```
proc.go
```



```go
// Change number of processors. The world is stopped, sched is locked.
// gcworkbufs are not being modified by either the GC or
// the write barrier code.
// Returns list of Ps with local work, they need to be scheduled by the caller.
// 所有的P都在这个函数分配，不管是最开始的初始化分配，还是后期调整
func procresize(nprocs int32) *p {
    // 默认传入的 nprocs 就是 CPU 个数，不能为 0

    old := gomaxprocs
    // 如果 gomaxprocs <=0 抛出异常
    if old < 0 || nprocs <= 0 {
        throw("procresize: invalid arg")
    }
  ...
    // Grow allp if necessary. allp 是全局数组
    if nprocs > int32(len(allp)) {
        // Synchronize with retake, which could be running
        // concurrently since it doesn't run on a P.
        lock(&allpLock)
        if nprocs <= int32(cap(allp)) {
            allp = allp[:nprocs]
        } else {
            // 分配nprocs个*p
            nallp := make([]*p, nprocs)
            // Copy everything up to allp's cap so we
            // never lose old allocated Ps.
            copy(nallp, allp[:cap(allp)])
            allp = nallp
        }
        unlock(&allpLock)
    }

    // initialize new P's
    for i := int32(0); i < nprocs; i++ {
        pp := allp[i]
        if pp == nil {
            pp = new(p)
            pp.id = i
            pp.status = _Pgcstop            // 更改状态
            pp.sudogcache = pp.sudogbuf[:0] //将sudogcache指向sudogbuf的起始地址
            for i := range pp.deferpool {
                pp.deferpool[i] = pp.deferpoolbuf[i][:0]
            }
            pp.wbBuf.reset()
            // 将pp保存到allp数组里, 下面这行代码等价于 allp[i] = pp
            atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
        }
        ...
    }
  ...

    _g_ := getg()
    // 如果当前的M已经绑定P，继续使用，否则将当前的M绑定一个P
    if _g_.m.p != 0 && _g_.m.p.ptr().id < nprocs {
        // continue to use the current P
        _g_.m.p.ptr().status = _Prunning
    } else {
        // release the current P and acquire allp[0]
        // 获取allp[0]
        if _g_.m.p != 0 {
            _g_.m.p.ptr().m = 0
        }
        _g_.m.p = 0
        _g_.m.mcache = nil
        p := allp[0]
        p.m = 0
        p.status = _Pidle
        // 将当前的m和p绑定
        acquirep(p)
        if trace.enabled {
            traceGoStart()
        }
    }
    var runnablePs *p
    for i := nprocs - 1; i >= 0; i-- {
        p := allp[i]
        if _g_.m.p.ptr() == p {
            continue
        }
        p.status = _Pidle
        // 判断当前的 p 是不是被绑定
        if runqempty(p) { // 将空闲p放入空闲链表
            pidleput(p)
        } else {
            p.m.set(mget())
            p.link.set(runnablePs)
            runnablePs = p
        }
    }
    stealOrder.reset(uint32(nprocs))
    var int32p *int32 = &gomaxprocs // make compiler check that gomaxprocs is an int32
    atomic.Store((*uint32)(unsafe.Pointer(int32p)), uint32(nprocs))
    return runnablePs
}
```

所有的P在程序启动的时候就设置好了，并用一个allp slice维护，可以调用runtime.GOMAXPROCS调整P的个数，虽然代价很大（会停止世界 stopTheWorld，里面会 stop go）

#### P的状态

`runtime2.go`中的全局变量 *allp* 存储所有可以拿到的 p
*sched.pidle* 存储所有的空闲的 P，是 P 的空闲队列（链表，sched.pidle存储一个p指针，p.link存储下一个p指针）

**P被放入空闲队列(`pidleput(p)`)的情况：**

1. 执行完成当前g，在调度过程中，窃取不到其他的g，则会被加入空闲队列（`schedule()`函数中查找本地队列无可用g，调用`findrunnable()`函数，仍找不到g，则调用 `pidleput(p)`）
   `findrunnable()`：本地队列获取g→全局队列获取g→从netpoll获取→其他p处偷g（一般从队列偷一半，如果偷不到，则尝试偷从其他p的 p.runnext 偷取）
2. m退出，m和p解绑，并将p加入空闲队列（`handoff(release(p))`函数中调用 `pidleput(p)`）
3. 修改了*GOMAXPROCS*后，世界停止，调度停止。（`procresize(nprocs)`函数中调用 `pidleput(p)`）
   如果*GOMAXPROCS*减小，则多余的p进入 _Pdead；
   如果*GOMAXPROCS*增大，则创建缺少的 p；
   对于所有即将使用的 p （修改后的*GOMAXPROCS*个p），本地队列没有 go 任务的 p ，加入空闲队列

见`runtime2.go`

| p.status  | value | 执行用户代码 | 位于空闲队列 | 分配了M | 含义                                                         |
| :-------: | :---: | :----------: | :----------: | :-----: | :----------------------------------------------------------- |
|  _Pidle   |   0   |   **`×`**    |   **`√`**    | **`○`** | P没有被用来执行用户代码，也没有被调度，而是被放在 the idle P list，可以被调度器获取。也可能只是在状态转换的中间过程中 |
| _Prunning |   1   |   **`√`**    |   **`×`**    | **`√`** | P被一个M所拥有，用来运行用户代码或者调度器，只有拥有该P的M能够从这个状态修改为其他状态（比如转化为：_Pidle-没有工作需要做；_Psyscall-进入一个系统调用；_Pgcstop-停下来执行gc）。M也可以将P交给其他的M（比如调度一个加锁的G） |
| _Psyscall |   2   |   **`×`**    |   **`×`**    | **`√`** | P没有运行用户代码。 它与系统调用中的M绑定，但不属于它，并且可能被另一个M窃取。这类似于_Pidle，但使用轻量级的状态转换，同时与M绑定。离开_Psyscall 必须与CAS（atomic.Cas 原子操作的状态转换函数）一起调用，以窃取或重新获得P。请注意，这存在ABA危险：即使M在syscall后成功将其原始P返回_Prunning状态，它也必须了解P可能在此期间已被另一个M使用 。 |
| _Pgcstop  |   3   |   **`×`**    |   **`×`**    | **`○`** | 执行 stopTheWorld()（位于proc.go，用于暂停所有的G，简称STW） 时暂停P，由 STW的M拥有。STW的M甚至在_Pgcstop中也继续使用其P。 从_Prunning过渡到_Pgcstop会导致M释放其P并停放。P保留其运行队列，startTheWorld将在具有非空运行队列的Ps上重新启动调度程序 schedule()。 |
|  _Pdead   |   4   |   **`×`**    |   **`×`**    | **`×`** | P不再被使用，而被放入空闲队列（GOMAXPROCS缩小）。如果GOMAXPROCS增加，我们将重用P。一个死掉的P被剥夺了其大部分资源，尽管还剩下一些东西（例如，跟踪缓冲区）。 |



```ruby
                                             acquirep(p)        
                          不需要使用的P       P和M绑定的时候       进入系统调用       procresize()
new(p)  -----+        +---------------+     +-----------+     +------------+    +----------+
            |         |               |     |           |     |            |    |          |
            |   +------------+    +---v--------+    +---v--------+    +----v-------+    +--v---------+
            +-->|  _Pgcstop  |    |    _Pidle  |    |  _Prunning |    |  _Psyscall |    |   _Pdead   |
                +------^-----+    +--------^---+    +--------^---+    +------------+    +------------+
                       |            |     |            |     |            |
                       +------------+     +------------+     +------------+
                           GC结束            releasep()        退出系统调用
                                             P和M解绑                      
```

P的数量默认等于cpu的个数，很多人认为runtime.GOMAXPROCS可以限制系统线程的数量，但这是错误的，M是按需创建的，和runtime.GOMAXPROCS没有关系。
如果一开始runtime.GOMAXPROCS=10，之后修改成5，那么有5个P不允许使用，那么这些P进入_Pdead 状态。如果再次调整runtime.GOMAXPROCS=10，就会改状态为 _Pgcstop

## G的一生

#### G的创建

```
proc.go
```



```go
// Create a new g running fn with siz bytes of arguments.
// Put it on the queue of g's waiting to run.
// The compiler turns a go statement into a call to this.
// Cannot split the stack because it assumes that the arguments
// are available sequentially after &fn; they would not be
// copied if a stack split occurred.
//go:nosplit
// 新建一个goroutine，
// 用fn + PtrSize 获取第一个参数的地址，也就是argp
// 用siz - 8 获取pc地址
func newproc(siz int32, fn *funcval) {
    argp := add(unsafe.Pointer(&fn), sys.PtrSize)
    pc := getcallerpc()
    // 用g0的栈创建G对象
    systemstack(func() {
        // 真正创建
        newproc1(fn, (*uint8)(argp), siz, pc)
    })
}
```



```go
// Create a new g running fn with narg bytes of arguments starting
// at argp. callerpc is the address of the go statement that created
// this. The new g is put on the queue of g's waiting to run.
// 根据函数参数和函数地址，创建一个新的G，然后将这个G加入队列等待运行
func newproc1(fn *funcval, argp *uint8, narg int32, callerpc uintptr) {
    // 获取当前g
    _g_ := getg()

    if fn == nil {
        _g_.m.throwing = -1 // do not dump full stacks
        throw("go of nil func value")
    }
    _g_.m.locks++ // disable preemption because it can be holding p in a local var
    siz := narg
    siz = (siz + 7) &^ 7

    // We could allocate a larger initial stack if necessary.
    // Not worth it: this is almost always an error.
    // 4*sizeof(uintreg): extra space added below
    // sizeof(uintreg): caller's LR (arm) or return address (x86, in gostartcall).
    // 如果函数的参数大小比2048大的话，直接panic
    // 这里的sys.RegSize是根据系统会有区别的，比如64位就是8字节，32位就是4字节
    if siz >= _StackMin-4*sys.RegSize-sys.RegSize {
        throw("newproc: function arguments too large for new goroutine")
    }

    // 从当前g的m中获取p
    _p_ := _g_.m.p.ptr()
    // 从gfree list获取g
    newg := gfget(_p_)
    // 如果没获取到g，则新建一个
    if newg == nil {
        newg = malg(_StackMin)
        casgstatus(newg, _Gidle, _Gdead) //将g的状态改为_Gdead
        // 添加到allg数组，防止gc扫描清除掉
        allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
    }
    if newg.stack.hi == 0 {
        throw("newproc1: newg missing stack")
    }

    if readgstatus(newg) != _Gdead {
        throw("newproc1: new g is not Gdead")
    }

    totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
    totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
    sp := newg.stack.hi - totalSize
    spArg := sp
    if usesLR {
        // caller's LR
        *(*uintptr)(unsafe.Pointer(sp)) = 0
        prepGoExitFrame(sp)
        spArg += sys.MinFrameSize
    }
    if narg > 0 {
        // copy参数
        memmove(unsafe.Pointer(spArg), unsafe.Pointer(argp), uintptr(narg))
        // This is a stack-to-stack copy. If write barriers
        // are enabled and the source stack is grey (the
        // destination is always black), then perform a
        // barrier copy. We do this *after* the memmove
        // because the destination stack may have garbage on
        // it.
        if writeBarrier.needed && !_g_.m.curg.gcscandone {
            f := findfunc(fn.fn)
            stkmap := (*stackmap)(funcdata(f, _FUNCDATA_ArgsPointerMaps))
            // We're in the prologue, so it's always stack map index 0.
            bv := stackmapdata(stkmap, 0)
            bulkBarrierBitmap(spArg, spArg, uintptr(narg), 0, bv.bytedata)
        }
    }

    memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
    // 下面是对新创建好的g设置各种参数，之后的调度就是根据参数走的
    newg.sched.sp = sp
    newg.stktopsp = sp
    // 保存goexit的地址到sched.pc
    newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
    newg.sched.g = guintptr(unsafe.Pointer(newg))
    gostartcallfn(&newg.sched, fn)
    newg.gopc = callerpc
    newg.startpc = fn.fn
    if _g_.m.curg != nil {
        newg.labels = _g_.m.curg.labels
    }
    if isSystemGoroutine(newg) {
        atomic.Xadd(&sched.ngsys, +1)
    }
    newg.gcscanvalid = false
    // 更改当前g的状态为_Grunnable
    casgstatus(newg, _Gdead, _Grunnable)

    // 生成唯一的goid
    if _p_.goidcache == _p_.goidcacheend {
        // Sched.goidgen is the last allocated id,
        // this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
        // At startup sched.goidgen=0, so main goroutine receives goid=1.
        _p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
        _p_.goidcache -= _GoidCacheBatch - 1
        _p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
    }
    // 分配给g
    newg.goid = int64(_p_.goidcache)
    _p_.goidcache++
    if raceenabled {
        newg.racectx = racegostart(callerpc)
    }
    if trace.enabled {
        traceGoCreate(newg, newg.startpc)
    }
    // 将当前新生成的g，放入p的队列；p是从当前的g的m中获取的；如果队列没满就放在本地队列，否则会放入全局队列
    runqput(_p_, newg, true)

    // 如果有空闲的p 且 m没有处于自旋状态 且 main goroutine已经启动，那么唤醒某个m来执行任务
    if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
        wakep()
    }
    _g_.m.locks--
    if _g_.m.locks == 0 && _g_.preempt { // restore the preemption request in case we've cleared it in newstack
        _g_.stackguard0 = stackPreempt
    }
} 
```

#### G的状态

见`runtime2.go`

|  g.atomicstatus   | value | 执行用户代码 | 位于运行队列 | 拥有栈  | 分配了M | 分配了 P | 含义                                                         |
| :---------------: | :---: | :----------: | :----------: | :-----: | :-----: | :------: | :----------------------------------------------------------- |
|      _Gidle       |   0   |   **`×`**    |   **`×`**    | **`×`** | **`×`** | **`×`**  | 分配了空间，但是并未被初始化                                 |
|    _Grunnable     |   1   |   **`×`**    |   **`√`**    | **`×`** |         |          | goroutine在运行队列（run queue），但是并没有执行用户代码，没有享有栈 |
|     _Grunning     |   2   |   **`○`**    |   **`×`**    | **`√`** | **`√`** | **`√`**  | goroutine可能在执行用户代码（或者做一些其他操作），其拥有栈，不在运行队列，被分配了一个M和一个P |
|     _Gsyscall     |   3   |   **`×`**    |   **`×`**    | **`√`** | **`√`** |          | goroutine正在执行一个系统调用，并没有在执行用户代码，拥有栈，不在运行队列，被分配了一个M |
|     _Gwaiting     |   4   |   **`×`**    |   **`×`**    | **`×`** |         |          | 在运行时被阻塞，并没有在执行用户代码，不在运行队列，但是会在某个地方被记录（比如 channel wait queue），所以在条件允许后会调用 ready() 进入 _Grunnable 状态，并放在运行队列。 **除**通道操作可以在适当的通道锁下读取或写入堆栈的某些部分外，不拥有该堆栈。故在goroutine输入_Gwaiting之后访问堆栈是不安全的 |
| _Gmoribund_unused |   5   |              |              |         |         |          | 当前未使用，但在gdb脚本中进行了硬编码                        |
|      _Gdead       |   6   |   **`×`**    |   **`×`**    | **`○`** | **`×`** | **`×`**  | goroutine当前未被使用，可能直接退出了，在空闲列表中，或者是仅仅被初始化完成，没有执行用户代码，可能拥有或者不拥有堆栈，G和其堆栈被M所拥有，当前的M正在exiting G或者将G从空闲队列拿出来 |
| _Genqueue_unused  |   7   |              |              |         |         |          | 当前未被使用                                                 |
|    _Gcopystack    |   8   |   **`×`**    |   **`×`**    | **`×`** |         |          | goroutine的堆栈正在被移动。未在执行用户代码，不在运行队列，堆栈被将其置为 _Gcopystack 状态的goroutine所拥有 |
|    _Gpreempted    |   9   |   **`×`**    |   **`×`**    | **`×`** |         |          | goroutine停下来从而被一个suspendG抢占，类似 _Gwaiting 状态，但是没有地方负责 ready() 它。一些 suspendG 必须改变状态至 _Gwaiting 来负责调用本G的 ready() |



```ruby
                               --------------------------------------------------------
                               |                      +------------+                  |
                               |      ready           |            |                  |
                               |  +------------------ |  _Gwaiting |                  |
                               |  |                   |            |                  | newproc
                               |  |                   +------------+                  |
                               |  |                         ^ park_m                  |
                               V  V                         |                         |
  +------------+            +------------+  execute   +------------+            +------------+    
  |            |  newproc   |            | ---------> |            |   goexit   |            |
  |  _Gidle    | ---------> | _Grunnable |  yield     | _Grunning  | ---------> |   _Gdead   |      
  |            |            |            | <--------- |            |            |            |
  +------------+            +-----^------+            +------------+            +------------+
                                  |         entersyscall |      ^ 
                                  |                      V      | existsyscall
                                  |                   +------------+
                                  |   existsyscall    |            |
                                  +------------------ |  _Gsyscall |
                                                      |            |
                                                      +------------+
```

最开始是初始化值：0，就是 _Gidle 状态
新建的G都是_Grunnable的，新建G的时候优先从gfree list从获取G，这样可以复用G，所以上图的状态不是完整的，_Gdead通过newproc会变为_Grunnable， 通过go func()的语法新建的G，并不是直接运行，而是放入可运行的队列中，并不能决定其什么时候运行，而是靠调度系统去自发的运行。
_Gdead 也可能直接变为 _Grunnable，比如上面的代码`从gfree list获取g` `newg := gfget(_p_)`的时候，可能拿到的就是 _Gdead 状态的g，之后`更改当前g的状态为_Grunnable` `casgstatus(newg, _Gdead, _Grunnable)`

# 问题

------

看源码的时候，有可能出现只有声明但是没有函数体的函数情况，大致以下三种：

1. **函数体是汇编代码写的**
2. **利用编译指示，来获取真正的函数body，link的本质是把函数的名字link到当前的声明里面**
   比如函数上面写了 go:xxx xxx为 nickname
3. **由编译器帮忙重写**
   汇编代码和代码里面都是看不到实现方式的，相当于代码逻辑都在编译器里面
   比如 runtime.getg()



作者：ChaunhewieTian
链接：https://www.jianshu.com/p/bf46cee74f76
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。



