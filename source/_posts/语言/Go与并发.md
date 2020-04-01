---
title: Go与并发
comments: true
tags:
  - GAP
  - Go
  - Tips
categories: Go
abbrlink: 28fa
date: 2020-02-06 09:14:05
---

## goroutine

goroutine是Go并行设计的核心。goroutine算是协程，它比线程更小，十几个goroutine可能体现在底层就是五六个线程，Go语言内部实现了这些goroutine之间的内存共享。执行goroutine只需极少的栈内存(大概是4~5KB)。goroutine比thread更易用、更高效、更轻便。<!--more-->

goroutine是通过Go的runtime管理的一个线程管理器。goroutine通过`go`关键字实现，类似个普通函数。

```Go
package main

import (
    "fmt"
    "runtime"
)

func say(s string) {
    for i := 0; i < 5; i++ {
        runtime.Gosched() //让CPU把时间片让给别人,下次某个时候继续恢复执行该goroutine
        fmt.Println(s)
    }
}

func main() {
    go say("world") //开一个新的Goroutines执行
    say("hello") //当前Goroutines执行
}
```

| 概念         | 说明                                          |
| :----------- | :-------------------------------------------- |
| 进程         | 一个程序对应一个独立程序空间                  |
| 线程         | 一个执行空间，一个进程可以有多个线程          |
| 逻辑处理器   | 执行创建的goroutine，绑定一个线程             |
| 调度器       | Go运行时中的，分配goroutine给不同的逻辑处理器 |
| 全局运行队列 | 所有刚创建的goroutine都会放到这里             |
| 本地运行队列 | 逻辑处理器的goroutine队列                     |

当我们创建一个goroutine的后，会先存放在`全局运行队列`中，等待Go运行时的`调度器`进行调度，把他们分配给其中的一个`逻辑处理器`，并放到这个逻辑处理器对应的`本地运行队列`中，最终等着被`逻辑处理器`执行即可。

这一套管理、调度、执行goroutine的方式称之为Go的并发。那怎么做到Go的并行呢？多创建一个`逻辑处理器`就好了，这样调度器就可以同时分配`全局运行队列`中的goroutine到不同的`逻辑处理器`上并行执行。

```go
func main() {
	var wg sync.WaitGroup
	wg.Add(2)
	go func(){
		defer wg.Done()
		for i:=1;i<100;i++ {
			fmt.Println("A:",i)
		}
	}()
	go func(){
		defer wg.Done()
		for i:=1;i<100;i++ {
			fmt.Println("B:",i)
		}
	}()
	wg.Wait()
}
```

这里的`sync.WaitGroup`其实是一个计数的信号量，使用它的目的是要`main`函数等待两个goroutine执行完成后再结束，不然这两个`goroutine`还在运行的时候，程序就结束了，看不到想要的结果。

`sync.WaitGroup`的使用非常简单，先使用`Add` 方法设置计算器为2，每个goroutine的函数执行完后就调用`Done`方法减1（使用延迟语句defer完成逻辑）。`Wait`方法，在计数器大于0时阻塞，所以`main` 函数会一直等待2个goroutine完成后，再结束。

默认情况下，Go默认是给每个可用的物理处理器都分配一个逻辑处理器，因为我的电脑是4核的，所以上面的例子默认创建了4个逻辑处理器，所以这个例子中同时也有并行的调度。

上面多个goroutine运行在同一个进程里，共享内存数据，设计上遵循：**不通过共享来通信，而通过通信来共享。**

## 竞争

有并发，就有资源竞争，如果两个或者多个goroutine在没有相互同步的情况下，访问某个共享的资源，比如同时对该资源进行读写时，就会处于相互竞争的状态，即并发中的资源竞争。**我们对同一个资源的读写必须是原子化的，即同一时间只能有一个goroutine对共享资源进行读写操作**。

共享资源竞争的问题，Go为我们提供了一个工具帮助我们检查，这个就是`go build -race`命令。我们在当前项目目录下执行这个命令，生成一个可以执行文件，然后再运行这个可执行文件，就可以看到打印出的检测信息。

### mutex

sync包里提供了一种互斥型的锁`sync.Mutex`，可以控制哪些代码同一时段下，只能有一个goroutine访问，被sync互斥锁控制的这段代码范围，被称之为临界区。同时sync包还提供一种读写锁`sync.RWMutex`，设计有`RLock()`等四种方法，下面用经典的读者写者问题来举例RWMutex的用法。

```go
var (
	mutex sync.RWMutex
	wg sync.WaitGroup
	content string
)

func read() {
	defer wg.Done()
	mutex.RLock()
	fmt.Println(content)
	mutex.RUnlock()
}

func write(text string) {
	defer wg.Done()
	mutex.Lock()
	content += text
	mutex.Unlock()
}

func main() {
	wg.Add(4)
	go write("hhh")
	go read()
	go write("hhh")
	go read()
	wg.Wait()
}
```

### channels

在多个goroutine并发中，我们不仅可以通过原子函数和互斥锁保证对共享资源的安全访问，消除竞争的状态，还可以通过使用通道，在多个goroutine发送和接受共享的数据，达到数据同步的目的。

goroutine间数据的通信机制为channel，可以与Unix shell中双向管道做类比：可以通过它发送或者接收值，**这些值只能是channel类型**。定义一个channel时，需要定义发送到channel的值的类型。**必须使用make 创建channel：**

```Go
ci := make(chan int)
cs := make(chan string)
cf := make(chan interface{})
```

channel通过操作符`<-`来接收和发送数据

```Go
ch <- v    // 发送v到channel ch.
v := <-ch  // 从ch中接收数据，并赋值给v
```

默认情况下，channel接收和发送数据都是阻塞的，除非另一端已经准备好，这样就使得Goroutines同步变的更加的简单，而不需要显式的lock。所谓阻塞，也就是如果读取（value := <-ch）它将会被阻塞，直到有数据接收。其次，任何发送（ch<-5）将会被阻塞，直到数据被读出。无缓冲channel是在多个goroutine之间同步很棒的工具。

### Buffered Channels

上面为Go默认的非缓存类型的channel，Go也允许指定channel的缓冲大小，即channel可以存储多少元素。

`ch:= make(chan bool, 4)`创建了可以存储4个元素的bool 型channel。这个channel 中前4个元素可以无阻塞的写入。当写入第5个元素时，代码将会阻塞，直到其他goroutine从channel 中读取一些元素，腾出空间。

```Go
ch := make(chan type, value)
```

当 value = 0 时，channel 是无缓冲阻塞读写的，当value > 0 时，channel 有缓冲、是非阻塞的，直到写满 value 个元素才阻塞写入。我们可以用最为经典的生产者消费者来举例buffered channels的用法。

```Go
var ch chan int = make(chan int, 20)

func produce(max int, num int) {
	for i := 0; i < max; i++ {
		fmt.Println("Send", num, ": ", i)
		ch <- i
		time.Sleep(5)
	}
}

func consumer(max int, num int) {
	for i := 0; i < max; i++ {
		v := <-ch
		fmt.Println("Receive", num, ": ", v)
	}
}

func main() {
	go produce(5, 1)
	go consumer(10, 1)
	go consumer(5, 2)
	go produce(10, 2)
	time.Sleep(1 * time.Second)
}
```

#### Range

`for i := range c`能够不断的读取channel里面的数据，直到该channel被显式的关闭。

#### Close

我们可以通过内置函数`close`关闭channel，**一般在生产方使用**。关闭channel之后就无法再发送任何数据了，在消费方可以通过语法`v, ok := <-ch`测试channel是否被关闭。如果ok返回false，那么说明channel已经没有任何数据并且已经被关闭。

#### Select

如果存在多个channel的时候，Go里面提供了一个关键字`select`，通过`select`可以监听channel上的数据流动。

`select`默认是阻塞的，只有当监听的channel中有发送或接收可以进行时才会运行，当多个channel都准备好的时候，select是随机的选择一个执行的。

```Go
func fibonacci(c, quit chan int) {
    x, y := 1, 1
    for {
        select {
        case c <- x:
            x, y = y, x + y
        case <-quit:
            fmt.Println("quit")
            return
        }
    }
}

func main() {
    c := make(chan int)
    quit := make(chan int)
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-c)
        }
        quit <- 0
    }()
    fibonacci(c, quit)
}
```

在`select`里面还有default语法，`select`其实就是类似switch的功能，default就是当监听的channel都没有准备好的时候，默认执行的（select不再阻塞等待channel）。

```Go
select {
case i := <-c:
    // use i
default:
    // 当c阻塞的时候执行这里
}
```

#### 超时

有时候会出现goroutine阻塞的情况，为了避免整个程序进入阻塞，可以利用select来设置超时

```Go
func main() {
    c := make(chan int)
    o := make(chan bool)
    go func() {
        for {
            select {
                case v := <- c:
                    println(v)
                case <- time.After(5 * time.Second):
                    println("timeout")
                    o <- true
                    break
            }
        }
    }()
    <- o
}
```

### runtime goroutine

runtime包中有几个处理goroutine的函数：

- Goexit

  退出当前执行的goroutine，但是defer函数还会继续调用

- Gosched

  让出当前goroutine的执行权限，调度器安排其他等待的任务运行，并在下次某个时候从该位置恢复执行。

- NumCPU

  返回 CPU 核数量

- NumGoroutine

  返回正在执行和排队的任务总数

- GOMAXPROCS

  用来设置可以并行计算的CPU核数的最大值，并返回之前的值。

## 并发控制

控制并发有两种经典的方式，一种是WaitGroup，另外一种就是Context。

### WaitGroup

如上面用到的，waitGroup尤其适用于多个goroutine协同做一件事情的时候，因为每个goroutine做的都是这件事情的一部分，只有全部的goroutine都完成，这件事情才算是完成，即wait模式。

### Context

对于一个网络请求Request，每个Request都需要开启一个goroutine做一些事情，这些goroutine又可能会开启其他的goroutine。所以我们需要一种可以跟踪goroutine的方案，才可以达到控制他们的目的，Go提供了Context，即goroutine的上下文。

`context.Background()` 返回一个空的Context，这个空的Context一般用于整个Context树的根节点。我们使用`context.WithCancel(parent)`函数，创建一个可取消的子Context，然后当作参数传给goroutine使用，这样就可以使用这个子Context跟踪这个goroutine。

在goroutine中，使用select调用`<-ctx.Done()`判断是否要结束，如果接受到值的话，就可以返回结束goroutine了；如果接收不到，就会继续进行监控。

`cancel`函数是由调用`context.WithCancel(parent)`函数生成子Context的时候返回的，，它是`CancelFunc`类型的。调用它就可以发出取消指令，然后我们的监控goroutine就会收到信号，就会返回结束。

`Done`是Context接口常用的方法，如果Context取消的时候，我们就可以得到一个关闭的chan，关闭的chan是可以读取的，所以只要可以读取的时候，就意味着收到Context取消的信号了

```go
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	go watch(ctx,"【监控1】")
	go watch(ctx,"【监控2】")
	go watch(ctx,"【监控3】")

	time.Sleep(10 * time.Second)
	fmt.Println("可以了，通知监控停止")
	cancel()
	//为了检测监控过是否停止，如果没有监控输出，就表示停止了
	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context, name string) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(name,"监控退出，停止了...")
			return
		default:
			fmt.Println(name,"goroutine监控中...")
			time.Sleep(2 * time.Second)
		}
	}
}
```

示例中启动了3个监控goroutine进行不断的监控，每一个都使用了Context进行跟踪，当我们使用`cancel`函数通知取消时，这3个goroutine都会被结束。这就是Context的控制能力，它就像一个控制器一样，按下开关后，所有基于这个Context或者衍生的子Context都会收到通知，这时就可以进行清理操作了，最终释放goroutine，优雅的解决了goroutine启动后不可控的问题。

#### Context接口

`Deadline`方法是获取设置的截止时间的意思，第一个返回式是截止时间，到了这个时间点，Context会自动发起取消请求；第二个返回值ok==false时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消。

`Done`方法返回一个只读的chan，类型为`struct{}`，我们在goroutine中，如果该方法返回的chan可以读取，则意味着parent context已经发起了取消请求，我们通过`Done`方法收到这个信号后，就应该做清理操作，然后退出goroutine，释放资源。

`Err`方法返回取消的错误原因，因为什么Context被取消。Done和Err两者搭配起来就是常用经典用法：

```go
 func Stream(ctx context.Context, out chan<- Value) error {
  	for {
  		v, err := DoSomething(ctx)
  		if err != nil {
  			return err
  		}
  		select {
  		case <-ctx.Done():
  			return ctx.Err()
  		case out <- v:
  		}
  	}
  }
```

`Value`方法获取该Context上绑定的值，是一个键值对，所以要通过一个Key才可以获取对应的值，这个值一般是线程安全的。