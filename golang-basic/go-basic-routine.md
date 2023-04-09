## GO之协程
#### 1. Golang 线程和协程的区别
对于 **进程、线程**，都是有内核进行调度，有 CPU 时间片的概念，进行 抢占式调度（有多种调度算法）  
对于 **协程**(用户级线程，这是对内核透明的，也就是系统并不知道有协程的存在，是完全由用户自己的程序进行调度的，因为是由用户程序自己控制，那么就很难像抢占式调度那样做到强制的 CPU 控制权切换到其他进程/线程，通常只能进行 协作式调度，需要协程自己主动把控制权转让出去之后，其他协程才能被执行到。  

*goroutine和协程区别*  
本质上，goroutine 就是协程。 不同的是，Golang 在 runtime、系统调用等多方面对 goroutine 调度进行了封装和处理，当遇到长时间执行或者进行系统调用时，会主动把当前 goroutine 的CPU (P) 转让出去，让其他 goroutine 能被调度并执行，也就是 Golang 从语言层面支持了协程。Golang 的一大特色就是从语言层面原生支持协程，在函数或者方法前面加 go关键字就可创建一个协程。  

其他方面的比较:
- 内存消耗方面
  每个 goroutine (协程) 默认占用内存远比 Java 、C 的线程少。  
  goroutine：2KB   
　线程：8MB  
- 线程和 goroutine 切换调度开销方面  
  线程/goroutine 切换开销方面，goroutine 远比线程小  
  线程：涉及模式切换(从用户态切换到内核态)、16个寄存器、PC、SP...等寄存器的刷新等。   
  goroutine：只有三个寄存器的值修改 - PC / SP / DX。  

#### 2. 协程底层实现原理
线程是操作系统的内核对象，多线程编程时，如果线程数过多，就会导致频繁的上下文切换，这些 cpu 时间是一个额外的耗费。所以在一些高并发的网络服务器编程中，使用一个线程服务一个 socket 连接是很不明智的。于是操作系统提供了基于事件模式的异步编程模型。用少量的线程来服务大量的网络连接和I/O操作。但是采用异步和基于事件的编程模型，复杂化了程序代码的编写，非常容易出错。因为线程穿插，也提高排查错误的难度。

协程，是在应用层模拟的线程，他避免了上下文切换的额外耗费，兼顾了多线程的优点。简化了高并发程序的复杂度。举个例子，一个高并发的网络服务器，每一个socket连接进来，服务器用一个协程来对他进行服务。代码非常清晰，而且兼顾了性能。goroutine建立在操作系统线程基础之上，它与操作系统线程之间实现了一个多对多(M:N)的两级线程模型。即M个goroutine运行在N个操作系统线程之上，内核负责对这N个操作系统线程进行调度，而这N个系统线程又负责对这M个goroutine进行调度和运行。

协程和线程的原理是一样的，当 a线程 切换到 b线程 的时候，需要将 a线程 的相关执行进度压入栈，然后将 b线程 的执行进度出栈，进入 b线程 的执行序列。协程只不过是在 应用层 实现这一点。但是，协程并不是由操作系统调度的，协程是基于线程的。内部实现上，维护了一组数据结构和 n 个线程，真正的执行还是线程，协程执行的代码被扔进一个待执行队列中，由这 n 个线程从队列中拉出来执行。这就解决了协程的执行问题。那么协程是怎么切换的呢？答案是：golang 对各种 io函数 进行了封装，这些封装的函数提供给应用程序使用，而其内部调用了操作系统的异步 io函数，当这些异步函数返回 busy 或 bloking 时，golang 利用这个时机将现有的执行序列压栈，让线程去拉另外一个协程的代码来执行，利用并封装了操作系统的异步函数。包括 linux 的 epoll、select 和 windows 的 iocp、event 等。

由于golang是从编译器和语言基础库多个层面对协程做了实现，所以，golang的协程是目前各类有协程概念的语言中实现的最完整和成熟的。

#### 3. goroutine 的生命周期
- 创建：当我们使用关键字 go 启动一个函数时，Go 会在当前的 goroutine 上下文中创建一个新的 goroutine，并在新的 goroutine 中执行该函数。
- 运行：一旦 goroutine 被创建，它就开始运行。在运行时，goroutine 将执行该函数的代码，直到函数执行完毕或者遇到了 return、panic、fatal error 等情况导致程序崩溃。
- 结束：当函数执行完毕或者遇到了上述的情况时，该 goroutine 就会结束。此时，该 goroutine 的资源，包括堆栈和其它上下文信息，将被释放。如果在 goroutine 中创建了其它的 goroutine，那么这些 goroutine 也将随着父 goroutine 的结束而结束。

需要注意的是，Go 语言的垃圾回收机制会自动管理内存，并在合适的时候回收不再使用的内存。因此，在 goroutine 结束时，不需要手动释放资源。

#### 4. 让 goroutine 并行执行
通过设置 runtime.GOMAXPROCS(1)，强制让 goroutine 在一个逻辑处理器上并发执行。用同样的方式，我们可以设置逻辑处理器的个数等于物理处理器的个数，从而让 goroutine 并行执行(物理处理器的个数得大于 1)。
下面的代码可以让逻辑处理器的个数等于物理处理器的个数：
```
runtime.GOMAXPROCS(runtime.NumCPU())
```
其中的函数 NumCPU 返回可以使用的物理处理器的数量。因此，调用 GOMAXPROCS 函数就为每个可用的物理处理器创建一个逻辑处理器。注意，从 Golang 1.5 开始，GOMAXPROCS 的默认值已经等于可以使用的物理处理器的数量了。

#### 5. 等待 goroutine 完成任务
协程示例代码：
```
package main

import (
    "time"
    "fmt"
)

func say(s string) {
    for i := 0; i < 3; i++ {
        time.Sleep(100 * time.Millisecond)
        fmt.Println(s)
    }
}

func main() {
    go say("hello world")
    fmt.Println("over!")
}
```
输出的结果为：
```
over!
```
因为 goroutine 以***非阻塞***的方式执行，它们会随着程序(主线程)的结束而消亡，所以程序输出字符串 "over!" 就退出了，这可不是我们想要的结果。

##### 5.1 使用 channel
通过 channel 达到等待 goroutine 结束的目的，运行下面的代码：
```
func main() {
    done := make(chan bool)
    go func() {
        for i := 0; i < 3; i++ {
            time.Sleep(100 * time.Millisecond)
            fmt.Println("hello world")
        }
        done <- true
    }()

    <-done
    fmt.Println("over!")
}
```
输出的结果也是：
```
hello world
hello world
hello world
over!
```
这种方法的特点是执行多少次 done <- true 就得执行多少次 <-done。

##### 5.2 WaitGroup
WaitGroup 用来等待单个或多个 goroutines 执行结束。在主逻辑中使用 WaitGroup 的 Add 方法设置需要等待的 goroutines 的数量。在每个 goroutine 执行的函数中，需要调用 WaitGroup 的 Done 方法。最后在主逻辑中调用 WaitGroup 的 Wait 方法进行阻塞等待，直到所有 goroutine 执行完成。

使用方法可以总结为下面几点：
- 创建一个 WaitGroup 实例，比如名称为：wg
- 调用 wg.Add(n)，其中 n 是等待的 goroutine 的数量
- 在每个 goroutine 运行的函数中执行 defer wg.Done()
- 调用 wg.Wait() 阻塞主逻辑

```
package main

import (
    "fmt"
    "sync"
    "net/http"
)

func main() {
    var urls = []string{
        "https://www.baidu.com/",
        "https://www.cnblogs.com/",
    }

    var wg sync.WaitGroup

    for _, url := range urls {
        wg.Add(1)
        go fetch(url, &wg)
    }

    wg.Wait()
}

func fetch(url string, wg *sync.WaitGroup) (string, error) {    defer wg.Done()
    resp, err := http.Get(url)
    if err != nil {
        fmt.Println(err)
        return "", err
    }
    fmt.Println(resp.Status)
    return resp.Status, nil
}
```

#### 6. 协程池
高并发场景下，会启动大量协程进行业务处理，此时如果使用协程池可以复用对象，减少协程池内存分配的效率与创建协程池点创建开销，提高协程的执行效率。字节官方开源了gopkg库提供的 `gopool` 协程池实现。

**协程池实现原理**  
线程池结构：
```
type pool struct {
   // pool 的名字，打 metrics 和打 log 时用到
   name string
   // pool 的容量，也就是最大的真正在工作的 goroutine 的数量
   // 为了性能考虑，可能会有小误差
   cap int32
   // 配置信息
   config *Config
   // 任务链表
   taskHead  *task
   taskTail  *task
   taskLock  sync.Mutex
   taskCount int32
 
   // 记录正在运行的 worker 数量
   workerCount int32
 
   // 用来标记是否关闭
   closed int32
 
   // worker panic 的时候会调用这个方法
   panicHandler func(context.Context, interface{})
 }
 ```
 gopool最核心的方法：
```
func (p *pool) CtxGo(ctx context.Context, f func()) {
   // 首先是一个对象池，避免task对象反复分配
   t := taskPool.Get().(*task)
   // 拿到的对象做一些变量的设置
   t.ctx = ctx
   t.f = f
   
   // 这里的加上task锁，把生成的task绑定到task链表上
   p.taskLock.Lock()
   if p.taskHead == nil {
     p.taskHead = t
     p.taskTail = t
   } else {
     p.taskTail.next = t
     p.taskTail = t
   }
   p.taskLock.Unlock()
   // 解锁之后做一些统计信息变更
   atomic.AddInt32(&p.taskCount, 1)
   
   // 这个地方的逻辑是不是放到一开头判断一下也可以，
   // 或者上下都判断一下；
   // 这里写在下面可能考虑到上面的lock和taskpool的get可能会有一点时间的等待，
   // 所以如果我来写可能方法的入口和这里都会判断一下吧。不过这里也不是什么重点问题。

   // 如果 pool 已经被关闭了，就 panic
   if atomic.LoadInt32(&p.closed) == 1 {
     panic("use closed pool")
   }
   
   // 下面的注释作者的意图都已经说明了，
   // 解决的问题就是协程池goroutine需不需要加的问题；
   // 以及协程池是不是啥都没有的问题
 
   // 满足以下两个条件：
   // 1. task 数量大于阈值
   // 2. 目前的 worker 数量小于上限 p.cap
   // 或者目前没有 worker
   if (atomic.LoadInt32(&p.taskCount) >= p.config.ScaleThreshold && p.WorkerCount() < atomic.LoadInt32(&p.cap)) || p.WorkerCount() == 0 {
     p.incWorkerCount() // 加一个工作协程
     w := workerPool.Get().(*worker) // 对象池的优化
     w.pool = p
     w.run() // 让worker跑起来。
   }
 }
 ```
 worker是如何执行的:
 ```
  func (w *worker) run() {
   // 通过goroutine异步执行
   go func() {
     for {
       //select {
       //case <-w.stopChan:
       // w.close()
       // return
       //default:
       
       // 这里是从task池中获取一个task
       var t *task
       w.pool.taskLock.Lock()
       if w.pool.taskHead != nil {
         t = w.pool.taskHead
         w.pool.taskHead = w.pool.taskHead.next
         atomic.AddInt32(&w.pool.taskCount, -1)
       }
       if t == nil {
         // 如果没有任务要做了，就释放资源，退出
         w.close()
         w.pool.taskLock.Unlock()
         w.Recycle()
         return
       }
       w.pool.taskLock.Unlock()
       // 这里就是成功的获取一个任务，然后准备开始执行这个任务
       func() {
         // 防止任务panic做的一些打点逻辑
         defer func() {
           if r := recover(); r != nil {
             logs.CtxFatal(t.ctx, "GOPOOL: panic in pool: %s: %v: %s", w.pool.name, r, debug.Stack())
             if w.pool.config.EnablePanicMetrics {
               panicMetricsClient.EmitCounter(panicKey, 1, metrics.T{Name: "pool", Value: w.pool.name})
             }
             // 这里如果没有设置panicHandler可能会有空指针
             w.pool.panicHandler(t.ctx, r)
           }
         }()
         // 执行函数
         t.f()
       }()
       // 这个task已经做完了，回收对象
       t.Recycle()
       //}
     }
   }()
 }
 ```
- 接受一个新任务，放到任务链表中，使用锁控制保证并发安全
- 如果 task 数量大于阈值且当前的 worker 数量小于上限 p.cap 或者目前没有 worker，那么新建一个工作协程来执行任务。
- 任务执行过程中，使用 for 循环不断遍历 task 链表，如果链表不为空，则从链表中拿任务执行。链表为空，协程关闭，工作协程数减一。

#### 7. 协程调度
所谓的对goroutine的调度，是指程序代码按照一定的算法在适当的时候挑选出合适的goroutine并放到CPU上去运行的过程，这些负责对goroutine进行调度的程序代码我们称之为goroutine调度器。

##### 7.1 PMG 协程并发模型

Go语言中支撑整个scheduler实现的主要有4个重要结构，分别是M、G、P、Sched。
- Sched结构就是调度器，它维护有存储M和G的队列以及调度器的一些状态信息等。
- M结构是Machine，系统线程，它由操作系统管理的，goroutine就是跑在M之上的；M是一个很大的结构，里面维护小对象内存cache（mcache）、当前执行的goroutine、随机数发生器等等非常多的信息。
- P结构是Processor，处理器，它的主要用途就是用来执行goroutine的，它维护了一个goroutine队列，即runqueue。Processor是让我们从N:1调度到M:N调度的重要部分。
- G是goroutine实现的核心结构，它包含了栈，指令指针，以及其他对调度goroutine很重要的信息，例如其阻塞的channel。

> Processor 数量是在启动时被设置为环境变量GOMAXPROCS的值，或者通过运行时调用函数GOMAXPROCS()进行设置。Processor数量固定意味着任意时刻只有GOMAXPROCS个线程在运行go代码。

> M：Machine的简称，在linux平台上是用clone系统调用创建的，其与用linux pthread库创建出来的线程本质上是一样的，都是利用系统调用创建出来的OS线程实体。M的作用就是执行G中包装的并发任务。Go运行时系统中的调度器的主要职责就是将G公平合理的安排到多个M上去执行。

当正在运行的goroutine阻塞的时候，例如进行系统调用，会再创建一个系统线程（M1），当前的M线程放弃了它的Processor，P转到新的线程中去运行。这样就填补了这个进入系统调用的M的空缺，始终保证 有GOMAXPROCS个工作线程在干活了。

总结起来：
在 Go 进程启动之后，Go 会尝试建立若干个 M，也就是若干个物理线程，接着：
- 每个物理线程在建立之后，都要进入调度函数，一个M调度goroutine执行的过程是一个loop。
- M会从P的local queue弹出一个Runable状态的goroutine来执行，如果P的local queue为空，就会执行work stealing；如果实在找不到就会自动去睡眠。
  
我们通过 go func()来创建一个goroutine；
- 有两个存储goroutine的队列，一个是局部调度器P的local queue、一个是全局调度器数据模型schedt的global queue。
- 新创建的goroutine会先保存在local queue，如果local queue已经满了就会保存在全局的global queue；
- 创建 G 之后，发现有闲置的 P 就会尝试唤醒物理线程。

这个时候，G 的创建就结束了。

- M 从睡眠状态被唤醒之后，就要绑定一个 P。一个 M 必须持有一个P，M 与 P 是1：1的关系。绑定失败还是要回去睡觉。绑定成功了就会回到调度函数，继续尝试获取一个可运行的 G。
- 一切完美，直接运行 G 指定的代码。
- 当 G 执行了非阻塞调用或者网络调用之后，调度程序会将 G 保存上下文并切出 M，M 会运行下一个 runable 的 G
- 当 G 获得了想要的数据后，sysmon 线程会将 G 放入队列当中，等待着调度运行。
- 当 M 执行某一个 goroutine 时候如果发生了 syscall 或则其余阻塞操作。这种操作并不像非阻塞调用一样可以暂停 G，因为 M 物理线程大概率已经沉入内核，没有办法运行下一个 G，这个系统调用只能占用一个物理线程。但是这个时候 M 实际上可能只是等待内核的 IO 数据等等，并不会占用 CPU。
- 这时候，sysmon 线程会检测到 M 已经阻塞，如果当前有一些 G 在执行，runtime 会把这个线程 M 从 P 中摘除(detach)，然后再创建一个新的操作系统的线程(如果有空闲的线程可用就复用空闲线程)，尝试让操作系统调度这个新的物理线程来占用这个 CPU，保障并发度；
- 当系统调用结束时候，这个 M 会尝试获取一个空闲的 P 执行。如果获取不到 P，那么这个线程 M 会 park 它自己(休眠)，加入到空闲线程中，然后这个 goroutine 会被放入 schedt 的global queue。
- 当 G 运行结束之后，M 再次回到调度程序，并尝试获取新的 G，无法获取那就接着去睡眠，等待着新的 G 来唤醒它。

简单的说，调度的本质是不断的监控各个线程的运行状况，如果发现某个线程已经阻塞了，那么就要唤醒一个已有的线程或者新建一个线程，尝试让操作系统调度这个物理线程跑满所有的 CPU。

##### 7.2 抢占式调度
Go还只是引入了一些很初级的抢占，并没有像操作系统调度那么复 杂，没有对goroutine分时间片，设置优先级等。

只有长时间阻塞于系统调用，或者运行了较长时间才会被抢占。runtime会在后台有一个检测线程，它会检测这些情况，并通 知goroutine执行调度。

**sysmon**  
前面讲Go程序的初始化过程中有提到过，runtime开了一条后台线程，运行一个sysmon函数。这个函数会周期性地做epoll操 作，同时它还会检测每个P是否运行了较长时间。

如果检测到某个P状态处于Psyscall超过了一个sysmon的时间周期(20us)，并且还有其它可运行的任务，则切换P。

如果检测到某个P的状态为Prunning，并且它已经运行了超过10ms，则会将P的当前的G的stackguard设置为 StackPreempt。这个操作其实是相当于加上一个标记，通知这个G在合适时机进行调度。

**morestack 的修改**  
前面说的，将stackguard设置为StackPreempt实际上是一个比较trick的代码。我们知道Go会在每个函数入口处比较当前的栈 寄存器值和stackguard值来决定是否触发morestack函数。

将stackguard设置为StackPreempt作用是进入函数时必定触发morestack，然后在morestack中再引发调度。

所以，到目前为止Go的抢占式调度还是很初级的，比如一个goroutine运行了很久，但是它并没有调用另一个函数，则它不会 被抢占。当然，一个运行很久却不调用函数的代码并不是多数情况。
