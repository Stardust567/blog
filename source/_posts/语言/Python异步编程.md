---
title: Python异步编程
comments: true
tags:
  - GAP
  - Python
  - asyncio
  - concurrency
categories: Python
abbrlink: 92c
date: 2020-05-24 12:24:06
---

python由于GIL（全局锁）的存在，不能发挥多核的优势，这点一直饱受诟病。不过在IO密集型的网络编程里，异步处理比同步处理能提升成百上千倍的效率，弥补了python性能方面的短板。<!-- More -->

同步：按程序定义依次执行事务，如遇阻塞会一直等待，前一个事务执行完毕才进行下一个事务。
异步：执行前一个事务时，如遇阻塞会直接执行第二个事务，不会等待，通过状态、通知、回调完成。

| 并发类型         | 切换决策                      | 处理器数量    |
| :----------- | :-----------------------------| :------------ |
| 抢占式多任务处理（`threading`）| 操作系统决定何时切换Python外部的任务 | 1个 |
| 合作多任务处理（`asyncio`）| 由任务自身决定何时放弃控制权 | 1个 |
| 多进程并行处理（`multiprocessing`）| 所有进程都同时在不同的处理器上运行 |many|

事实上，因为GIL的存在，thread哪怕操作系统这边一切ok，抢不到GIL锁还是白搭，多核跑多线程python属实不太行。用process跑多核，要申请的资源开销还是比较大的。coroutine底层是单线程，虽然开销小，但没法用多核。那么，我们组合一下呗，采取多进程+协程的模式，皆大欢喜。不过考虑到GIL在遇到IO操作时会主动释放，所以python多线程也不是说不能用，只是说对耗CPU的操作不适合。

# yield

用过scrapy的想必都对yield不陌生，generator是一个标准的python协程，由`yield`实现**return+暂定**的功能，`next`实现**再次调用**的功能，`send`实现**再次调用+yield传值**的功能。以最简单的生产者消费者举例：

```python
def producer(n, consumer):
    for i in range(n):
        print("producting:", i)
        consumer.send(i) 
        # 将i发送到yield位置并从该位置调用生成器

def consumer():
    while(True):
        i = yield True
        print("consuming:", i)

c = consumer() # 一个生成器实例
next(c) # 先让c到第一次yield处，不然没法接收send
producer(10, c) # 调用生产者函数
```

> 没有`next(c)`是会报错的哦：TypeError: can't send non-None value to a just-started generator

# asyncio

asyncio在python3.4版本引入到标准库，python3.5又加入了async/await特性。

## 设计

`asyncio` 最大特点就是，底层只有一个线程，多个任务分享运行时间，即多任务并发。asyncio 允许异步任务交出执行权给其他任务，等到其他任务完成，再收回执行权继续往下执行。作为协程，用户态控制切换，所以和线程比省去了切换的开销，当然切换时候还是会保存上下文和栈（*协程的栈空间是可以动态调整的*）可以无锁。

asyncio 模块在单线程上启动一个事件循环（event loop），时刻监听新进入循环的事件，加以处理，并不断重复这个过程，直到异步任务结束。有过I/O多路复用的经验，应该不难理解事件循环。

## 概念

事件循环event_loop：程序会开启一个无限循环，我们把一些任务注册到循环上，任务全部完成时循环终止。

如果一个对象可以在`await`语句中使用，则称该对象为awaitable objec，即可等待对象。
常用的可等待对象有三种，分别是协程对象 **coroutines**、任务 **Tasks** 以及 **Future**（和Task差不多）

### 协程Coroutines

这里的**协程**指**协程函数**（形式为`async def`的函数）和**协程对象**（调用协程函数所返回的对象）

执行引擎遇到`await`命令，就会在异步任务开始执行后，暂停当前 async 函数的执行，把执行权交给其他任务。等到异步任务结束，再把执行权交回 async 函数，继续往下执行。

```python
import asyncio

async def count(n):
# async相当于go的goroutine
    print("One")
    await asyncio.sleep(n)
    print("Two")
    return n

async def main():
    print("Starting...")
    a = await count(1)
    b = await count(2)
    print(f"{a} & {b} End.")

asyncio.run(main())
# async.run()加载async函数，启动事件循环
```

`asyncio.run()` 在事件循环上监听 async 函数`main`的执行，等到 `main` 执行完了，事件循环才会终止。TIPS：如果直接调用`main()`，只会返回一个`coroutine`对象，`main()`方法内的代码不会执行。

此时事件循环loop里只有一个main()，进入main函数后，print完Starting后阻塞main进入count(1)，print完One后阻塞count函数进入sleep，由于loop里没有别的任务，所以相当于同步代码，会一直等sleep结束再继续执行。

> Starting...
> One
> Two
> One
> Two
> 1 & 2 End.

*TIPS: python3.7后才有的`asyncio.run(result)`这样，相当于之前版本的`loop = asyncio.get_event_loop() loop.run_until_complete(result)`*

### 任务Task

一个协程可以通过`asyncio.create_task()`被打包为一个Task，此时会立即把Task添加到事件循环准备运行。

我们把上面的main函数改一下，此时的事件循环相当于loop=[main(), count(1), count(2)]此时当count(1)进行sleep阻塞时，会将执行权交给count(2)，等sleep结束再切回来，达成并发的效果。

```python
async def main():
    print("Starting...")
    task1 = asyncio.create_task(count(1)) # 注意此时是不会转进count()函数的
    task2 = asyncio.create_task(count(2))
    a = await task1 # 此时才会交出执行权，进入异步任务task1，即count()函数
    b = await task2
    print(f"{a} & {b} End.")
```

> Starting...
> One
> One
> Two
> Two
> 1 & 2 End.

不过一般我们编程时候更习惯用list把Tasks简单的封装一下，但是要记住create_task会将task加入事件循环，但事件循环是由asyncio.run(main())创建，所以create_task必须要写在main()函数里面，不然会报错。

```python
async def main():
    task_list = [
        asyncio.create_task(count(1)),
        asyncio.create_task(count(2)),
    ]
    # 如果不设timeout，await会等待所有协程执行完毕，并将所有协程的返回值存到done(type:set)
    # timeout表示此处最多等待的秒，完成的协程返回值写入到done中，未完成则写到pending中
    done, pending = await asyncio.wait(task_list, timeout=None)
    print(done, pending)
```

> One
> One
> Two
> Two
> `{<Task finished coro=<count() done, defined at 0521.py:3> result=1>, <Task finished coro=<count() done, defined at 0521.py:3> result=2>}`
> set()

## API

### gather()

`asyncio.gather()` 将多个异步任务包装成一个新的异步任务，必须等到内部的多个异步任务都执行结束，这个新的异步任务才会结束。在实例中就相当于两个count()函数并发执行。

```python
import asyncio

async def count():
    print("One")
    await asyncio.sleep(1)
    print("Two")

async def main():
    await asyncio.gather(count(), count())

asyncio.run(main())
```
> One
> One
> Two
> Two

### async with

这是个异步上下文管理器，`async with aiohttp.ClientSession() as session:`可以类比下python文件操作的上下文管理器`open with url as f:`，进入时候进行封装好的`__enter__()`操作，退出时候进行封装好的`__exit__()`操作，异步操作也一样，通过`__aenter__()`和`__aexit__()`来对`async with`语句中的环境进行控制。

以数据库连接为例，进入`async with AsyncContextManager() as f:`时会执行`__aenter__(self)`连接数据库并把return的self赋给f，当异步完成后会执行` __aexit__(self, exc_type, exc, tb)`来关闭数据库。

```python
import asyncio

class AsyncContextManager:
    def __init__(self):
        self.conn = None
    async def do_something(self):
        # 异步操作数据库
        return "Hello World"
    async def __aenter__(self):
        # 异步链接数据库
        self.conn = await asyncio.sleep(1)
        return self
    async def __aexit__(self, exc_type, exc, tb):
        # 异步关闭数据库链接
        await asyncio.sleep(1)
        
async def func():
    async with AsyncContextManager() as f:
        result = await f.do_something()
        print(result)
asyncio.run(func())
```

## 实例

让我们再次回到经典的生产者消费者模型

```python
import asyncio
import itertools as it
import os
import random
import time

async def makeitem(size: int = 5) -> str:
    return os.urandom(size).hex()

async def randsleep(a: int = 1, b: int = 5, caller=None) -> None:
    i = random.randint(0, 10)
    if caller:
        print(f"{caller} sleeping for {i} seconds.")
    await asyncio.sleep(i)

async def produce(name: int, q: asyncio.Queue) -> None:
    n = random.randint(0, 10)
    for _ in it.repeat(None, n):  # Synchronous loop for each single producer
        await randsleep(caller=f"Producer {name}")
        i = await makeitem()
        t = time.perf_counter()
        await q.put((i, t))
        print(f"Producer {name} added <{i}> to queue.")

async def consume(name: int, q: asyncio.Queue) -> None:
    while True:
        await randsleep(caller=f"Consumer {name}")
        i, t = await q.get()
        now = time.perf_counter()
        print(f"Consumer {name} got element <{i}>"
              f" in {now-t:0.5f} seconds.")
        q.task_done()

async def main(nprod: int, ncon: int):
    q = asyncio.Queue()
    producers = [asyncio.create_task(produce(n, q)) for n in range(nprod)]
    consumers = [asyncio.create_task(consume(n, q)) for n in range(ncon)]
    await asyncio.gather(*producers)
    await q.join()  # Implicitly awaits consumers, too
    for c in consumers:
        c.cancel()

if __name__ == "__main__":
    import argparse
    random.seed(444)
    parser = argparse.ArgumentParser()
    parser.add_argument("-p", "--nprod", type=int, default=5)
    parser.add_argument("-c", "--ncon", type=int, default=10)
    ns = parser.parse_args()
    start = time.perf_counter()
    asyncio.run(main(**ns.__dict__))
    elapsed = time.perf_counter() - start
    print(f"Program completed in {elapsed:0.5f} seconds.")
```

# aiohttp

在编写爬虫应用时，需要通过网络IO去请求目标数据，这种情况适合使用异步编程来提升性能。

```python
import aiohttp
import asyncio

async def fetch(session, url):
    print("发送请求：", url)
    try:
        async with session.get(url, verify_ssl=False) as response:
            text = await response.text()
            print("得到结果：", url, len(text))
    except aiohttp.client_exceptions.ClientConnectorError as e:
        print(e)
        
async def main():
    async with aiohttp.ClientSession() as session:
        url_list = [
            'https://stardust567.github.io',
            'https://stardust567.top',
            'https://jotang.club'
        ]
        tasks = [asyncio.create_task(fetch(session, url)) for url in url_list]
        await asyncio.wait(tasks)
        
if __name__ == '__main__':
    asyncio.run(main())
```

因为我中间那个域名还没申请https所以会有error改成http就好了（不过之后域名到期我可能没钱供就是了orz）

> 发送请求： https://stardust567.github.io
> 发送请求： https://www.stardust567.top
> 发送请求： https://jotang.club
> 得到结果： https://stardust567.github.io 76565
> 得到结果： https://jotang.club 6852
> Cannot connect to host www.stardust567.top:443 ssl:False [Connect call failed ('118.25.70.50', 443)]