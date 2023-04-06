## GO语音之协程
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

协程，是在应用层模拟的线程，他避免了上下文切换的额外耗费，兼顾了多线程的优点。简化了高并发程序的复杂度。举个例子，一个高并发的网络服务器，每一个socket连接进来，服务器用一个协程来对他进行服务。代码非常清晰，而且兼顾了性能。

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
