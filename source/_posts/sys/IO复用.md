---
title: Linux I/O
comments: true
tags:
  - GAP
  - Python
  - I/O
categories: Sys
abbrlink: ddca
date: 2020-05-29 14:19:54
---

本来是写异步编程时提到的IO问题，但是越写越多，越写越杂，加上由于GAP的原因巧妙地错开了计网编程这门专业课，所以干脆新开一篇I/O来介绍这些，也可以作为异步编程的预备篇章。<!-- More -->

常用的I/O包括网络，对主机而言，网络只是一种I/O设备，是数据源和数据接收方。从网络上接收到的数据从网络适配器经过I/O总线和内存总线复制到内存（通常是经过DMA传送）

阻塞：调用函数时，当前线程被挂起，**但不会消耗CPU资源**。
非阻塞：调用函数时，当前进程不会被挂起，而是立即返回。

# syscall

让我们回顾一下 *系统调用* 的知识点（感觉每篇博客前先回顾下基础知识已经变成惯例了orz）

unix环境下，比如read()读取文件实质上调用read函数后是从用户态陷入内核态，由unix内核控制，将数据写到用户态的地址上（即调用者传入的接受数据用的buf地址）成功之后返回用户态。

当进程调用read进行文件读取的时候，会产生一个系统调用，进程被切换到内核态，由于磁盘的读取速度远远比不上CPU，内核会将该进程挂起，对进程来说，此时处于阻塞状态，CPU被内核切换给其他进程去运行，当数据到来的时候会触发中断，再将进入内核态，内核会将准备好的数据写到进程指定的内存buff中，并通过调度算法再次将进程调出来运行，对于进程来说此时read产生返回，完成一次数据读取。

但syscall显然要消耗资源的，首先发起调用时需要进程空间切换：从用户态切换到内核态（linux使用一条触发软中断的特殊汇编指令：int $0x80 来完成）用户态执行了这条指令之后，CPU将会自动进入一个内核初始化时设置好的中断服务例程，CPU会自动判断当前特权级别，如果当前是用户态，则会重新设置相应寄存器，进入最高特权级0，进入内核态，开始根据用户传入的调用号（调用号对应系统调用api）执行相应的调用，调用完成之后会重新设置寄存器，确定返回用户态的时候特权级是3，这就是系统调用的大概流程。

# I/O阻塞

假如现在我们需要实现一个进程，为多个调用者同时提供网络服务，比如web服务器，因为I/O阻塞的缘故，一个进程在一个时刻只能为一个调用者提供服务，等到服务完成，断开连接，才能继续accept下一个连接。那我们考虑开多线程呢？每个线程虽然共享进程的内存空间，但调用栈是独立的，互不影响，每个线程都可以独立执行，处于阻塞状态的线程不会影响其他线程，通过让web服务器实现多线程，每个线程处理一条客户端连接，就可以实现并发访问。

看起来很美好，BUT，独立的调用栈需要耗费内存空间、线程间的切换需要内核介入，耗费CPU资源，当并发数上升之后，这两个问题带来的影响越来越明显：显然系统无法容纳无限多的线程数，线程数上千之后将会使系统进入频繁的任务切换，同时多线程模型还会使数据重入问题变得很复杂，当多个线程同时对同一个数据进入写入时结果会变得不可预测，需要给临界区的数据进行加锁保护，但是锁本身又是通过内核的系统调用来支持的，这会进一步加深系统的负载，所以多线程并不是解决并发的好办法。

# 非阻塞式I/O

非阻塞的I/O不同在于其调用即返回，不会引起进程的等待，进程可以在很短的时间内去遍历read完所有文件描述符，如果文件没准备好数据，它就简单地返回0，这样就可以同时照顾多个文件描述符，而不会陷入某一个描述符的I/O被挂起，导致挂起中的进程无法处理别的描述符的I/O请求。当然I/O由DMA控制，并不需要CPU亲力亲为，但这个线程肯定是非阻不可的，这时候CPU切到其他线程上就好了。

但同时也带来了新的问题：如果进程的业务逻辑需要等待接收完数据才能执行下一步操作，进程就需要不停地循环read直到有数据返回，虽然可以遍历地read完所有文件描述符，实现一对多服务，但此时进程是处于一个大循环中，一直占用着CPU，直到时间片用完被内核强制切换，这个问题比多线程模型更加恐怖，一个长期占用CPU而什么都不做的进程是很窒息的，我们可以用echo程序来演示一下：

```python
import socket

CONN_ADDR = ('127.0.0.1', 9999)
conn_list = []  # 连接列表
sock = socket.socket(socket.AF_INET,socket.SOCK_STREAM)  # 开启socket
sock.setblocking(False)  # 设置为非阻塞
sock.bind(CONN_ADDR)  # 绑定IP和端口到套接字
sock.listen(5)          # 监听，5表示客户端最大连接数
print('start listen')
while True: # 不停轮询
    try:
        conn, addr = sock.accept()  # 被动接受TCP连接，等待连接的到来
        print('connect by ', addr)
        conn_list.append(conn)
        conn.setblocking(False)  # 设置非阻塞
    except BlockingIOError as e: # 把因非阻塞模式抛出的异常pass掉
        pass

    tmp_list = [conn for conn in conn_list]
    for conn in tmp_list:
        try:
            data = conn.recv(1024) # 一次接收的最大数据量即buffer=1024字节
            if data:
                print('收到的数据是{}'.format(data.decode()))
                conn.send(data.capitalize())
            else:
                print('close conn',conn)
                conn.close()
                conn_list.remove(conn)
                print('还有客户端=>',len(conn_list))
        except IOError:
            pass
```

可以看到server端是由程序编写的逻辑不断轮询所有connection没有停歇，这样无论client有没有数据传递，CPU都会一直被这个服务run起来，虽然这样可以做到一对多服务，但会不断地询问内核，占用大量的CPU时间，看起来并不划算。

```python
import socket
client = socket.socket()
client.connect(('localhost',9999))
while True:
    msg = input(">>:").strip()
    if len(msg) == 0: continue # 空消息recv会收不到
    elif(msg=='exit'): break # 输入exit时断开连接
    client.send(msg.encode("utf-8"))
    data = client.recv(1024)
    print("recv:",data.decode())
client.close()
```

> TCP黏包：1. 由Nagle算法造成的发送端的粘包：当我们提交一段数据给TCP发送时,TCP并不立刻发送此段数据,而是等待一小段时间,看看在等待期间是否还有要发送的数据,若有则会一次把这两段数据发送出去。2. 接收端接收不及时造成的接收端粘包:TCP会把接收到的数据存在自己的缓冲区中,然后通知应用层取数据.当应用层由于某些原因不能及时的把TCP的数据取出来,就会造成TCP缓冲区中存放了几段数据。
>
> TCP黏包本质上是因为tcp传输的是一段数据流，程序并不知道中间哪里应该有分割。
>
> 解决方案：1. 连续的send中间加上sleep(0.5)或者每次send后紧跟个recv清空buffer这样。2. 封包，即给一段数据加上包头，人为的将传输的数据流用标识分割开来。

一种能想到的解决办法是在大循环中加个sleep休眠，比如每休眠1ms交出执行权，再醒过来遍历文件描述符，这可以一定程度上解决CPU占用的问题，但是失去了实时性，并没有彻底解决问题。

# I/O多路复用

那么我们能不能同时监听多个I/O文件描述符的事件，监听的时候是处于阻塞的，当有事件到来的时候，系统才过来唤醒进程去读数据，如果能做到这样，进程就即不会空转CPU，又可以同时监听到所有I/O，在UNIX下，select模型实现了这样的想法，先来看一下select的形式。

## select

我们先来简单感受下select()函数，最大支持1024个描述符，采用bitmap表征需要监听的描述符们。比如有fd_list = [1, 2, 4, 7]那么bitmap为01101001表示需要监听这四个描述符。

```c
#undef __NFDBITS
#define __NFDBITS	(8 * sizeof(unsigned long))  // 一个unsigned long类型的bit数，8*8=64
#undef __FD_SETSIZE
#define __FD_SETSIZE	1024                    // 最大支持的fd数量，固定为1024
#undef __FDSET_LONGS
#define __FDSET_LONGS	(__FD_SETSIZE/__NFDBITS)  // 1024bits需要几个long类型，1024/64=16 

typedef struct {
	unsigned long fds_bits [__FDSET_LONGS];  
} __kernel_fd_set;
// fd_set实际上就是unsigned long数组，支持1024bits
//每个bit对应fd号，所以select下最大只能支持1024个fd
typedef __kernel_fd_set fd_set; 

int select(int nfds, fd_set *readfds, fd_set *writefds,
    fd_set *exceptfds, struct timeval *timeout);

void FD_CLR(int fd, fd_set *set);	// 清除fd的标志
int  FD_ISSET(int fd, fd_set *set);	// 判断事件是否被置位
void FD_SET(int fd, fd_set *set);	// 设置fd标志位
void FD_ZERO(fd_set *set);			// 清空所有标志位
```

对`select()`函数，我们可以简单介绍一下它的函数签名：

- **int nfds**：需要监听一个描述符数组fds=[8, 9 ,10]那么nfds=max(fds)+1=11即需要传11进去。
- **fd_set *readfds**：文件读事件的监听标志设置，比如说想监听fd=10这个文件的读事件，就使用FD_SET这个宏，把readfds的第10位置为1；如果有事件发生，内核会重新设置标志位，所以返回之后调用者可以通过遍历readfds来查看哪些文件产生了事件。
- **writefds**是写事件，**exceptfds**是异常事件，原理同上。
- **timeout**：超时设置，如果设为NULL，则不管超时，如果没有事件发生，select会一直阻塞下去。
- return：返回产生的事件总量，返回0表示超时，返回-1表示有错误发生，通过errno来查询错误类型。

### 过程

用户态服务调用select时候会阻塞，然后把IO任务交给内核，所以内核需要先把bitmaps复制到自己的空间中。然后快速循环三个bitmaps找出我们标记的需要监听的fd，把关心的事件写到wait.key里，同时将进程信息写进pwq中并赋给wait.private，这样被唤醒时内核就能通过wait找到相应进程。将fd和wait绑定到poll里，即f_op->poll(file,wait)~~具体数据结构和流程可以看源码或者引用~~。然后驱动程序执行poll流程，这个回调是每个设备的驱动程序要实现的内容，里面会返回一个mask来标志当前事件是否已经发生。把wait挂载到等待队列后，会根据返回的mask来判断是否有已经产生的事件，如果有的话就设置到copy进内核态的位图rset里，这是作为结果来返回给用户态的。所有fd都搞定之后，直接睡眠，进程被切走，直到有事件产生，设备驱动通过等待队列把进程唤醒。比如磁盘把数据通过DMA写到内存了，会产生一个硬件中断，内核收到中断后会交给磁盘驱动来处理，驱动程序会判断一下当前产生的事件是否在关注的事件里面，不是的话啥也不用管了，继续让进程睡眠；如果有，会找到这个文件对应的等待队列，然后调用wakeup来唤醒他们。 被唤醒后会再进入一次遍历，把所有fd都访问一遍poll，取出mask，来设置好产生的事件标志位修改传进来的位图rset，然后再把这些数据通过copy_to_user写向用户空间，结束整个系统调用，用户态取得了select的结果。

### 例子

```c
sockfd = socket(AF_INET, SOCK_STREAM, 0);
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_port = htons(2000);
addr.sin_addr.s_addr = INADDR_ANY;
bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
listen(sockfd, 5);

for(int i=0;i<5;i++) {
	memset(&client, 0, sizeof(client));
	addrlen = sizeof(client);
	fds[i] = accept(sockfd, (struct sockaddr*)&client, &addrlen);
	if(fds[i] > max) max = fds[i]; 
}

while(1) {
	FD_ZERO(&rset);
	for(int i=0;i<5;i++) FD_SET(fds[i], &rset); // 初始化位图rset
	puts("round again.");
	select(max+1, &rset, NULL, NULL, NULL); // 只监听读事件
	// 发生相应fd的读事件，rset在相应fd置位，select返回，程序往下走
	for(int i=0;i<5;i++) { // 遍历找到被置位的fd，进行read和处理操作
		if(FD_ISSET(fds[i], &rset)) {
			memset(buffer, 0, MAXBUF);
			read(fds[i], buffer, MAXBUF);
			puts(buffer);
		}
	}
}
```

### 缺点

1. 每次调用select都需要把用户态的数据复制一遍到内核态，读/写/异常三种事件总共有384字节需要复制，这是固定的，即使只监听一个fd。
2. 挂等待队列跟唤醒之后的事件收集都需要遍历整个fdset，复杂度为N。
3. 最长只能监听1024个fd，偏要自定义超过1024的话需要遍历数组两次，效率降低。

## poll

不同于select使用三个位图来表示三个fdset的方式，poll使用一个pollfd的指针实现。pollfd包含了要监视的event和发生的revent，不再使用select的“参数-值”传递方式。

```c
struct pollfd {
	int fd;
	short events;  // 事件，宏定义了POLLIN, POLLOUT
	short revents; // 对events的回馈，初始值为0
};

for (int i=0;i<5;i++) {
    memset(&client, 0, sizeof(client));
    addrlen = sizeof(client);
    pollfds[i].fd = accept(sockfd, (struct sockaddr*)&client, &addrlen);
    pollfds[i].events = POLLIN;
}
while(1) {
    puts("round again.");
    poll(pollfds, 5, 50000); // 5个fds， 超时时间50000
    // 发生事件时，会置相关的pollfds[i].revents = POLLIN这样
    for(int i=0;i<5;i++) {
        if(pollfds[i].revents & POLLIN) {
            pollfds[i].revents = 0;
            memset(buffer, 0, MAXBUF);
            read(pollfds[i].fd, buffer, MAXBUF);
            puts(buffer);
        }
    }
}
```

本质上，poll仅仅只是把select的固定数组换成可变长数组，以此来突破文件数量的限制，其他还是跟select一样。

## epoll

epoll是linux下关于poll的提升版，epoll使用一个文件描述符管理多个描述符，最大的优点就是**不再轮询所有描述符**(N)，只需要查看相关描述符(k)就可，把复杂度从O(N)降到了常数级O(k)，所以它适用于高并发但连接活跃度低的情况，比如浏览网页。但是如果并发性不高但连接活跃度高的话，还是select比较合适一点，比如连线游戏。

### 改进

- 把epoll当成一个文件对象，对相关fd事件，仅需复制一次数据到内核，便可在epoll文件生命周期中一直使用
- 可动态地添加/删除/修改事件，不再像select/poll那样每次都要重设好所有事件标志
- 内核态在监听过程中不再需要遍历所有fd的事件，在事件触发阶段即被写入一个链表中随时可以返回

### 步骤

- 使用epoll_create创建一个epoll实例，他在内核中创建一个匿名的inode节点，生成一个文件描述符epfd返回给用户，用户之后的操作都基于这个epoll文件描述符
- 使用epoll_ctl对需要关注的文件fd进行注册，内核会把这个fd的信息挂在一颗红黑树下面，同时向文件操作符调用poll，注册等待队列
- 使用epoll_wait进行事件监听，当有事件产生时，epoll回调会自动将文件事件插入到一个就绪列表中，然后唤醒进程，向用户空间返回一个就绪列表

```c
struct epoll_event events[5];
int epfd = epoll_create(10); // 创建一个epfd白板，10无明确意义

for(int i=0;i<5;i++) {
    static struct epoll_event ev;
    memset(&client, 0, sizeof(client));
    addrlen = sizeof(client);
    ev.data.fd = accept(sockfd, (struct sockaddr*)&client, &addrlen);
    ev.events = EPOLLIN;
    epoll_ctl(epfd, EPOLL_CTL_ADD, ev.data.fd, &ev); 
    // 将fd和events添加（EPOLL_CTL_ADD）到epfd，用户态和内核态共享epfd
}

while(1) {
    puts("round again.");
    nfds = epoll_wait(epfd, events, 5, 10000);
    // 返回触发事件fd数目，epoll会把触发的fd放到events队首
    for(int i=0;i<nfds;i++) {
        memset(buffer, 0, MAXBUF);
        read(events[i].data.fd, buffer, MAXBUF);
        puts(buffer);
    }
}
```

epoll_ctl支持三种操作：插入事件、删除事件、修改事件。这三种操作都可以在epoll_fd的生命周期内任意调用，得益于红黑树高效的查找与插入性能，这可以在lgN的时间内完成。

### LT/ET

epoll有水平触发（level trigger，LT，epoll默认工作模式）与边缘触发（edge trigger，ET）两种工作模式。

LT模式下，只要内核缓冲区中还有未读数据，就会一直返回描述符的就绪状态，即不断地唤醒应用进程。在ET模式下， 缓冲区从不可读变成可读，会唤醒应用进程，缓冲区数据变少的情况，则不会再唤醒应用进程。

#### 水平触发
对于**读**操作：只要缓冲内容不为空，LT模式返回读就绪。
对于**写**操作：只要缓冲区还不满，LT模式会返回写就绪。

#### 边缘触发
对于**读**操作：缓冲区由空变为不空的时候。当有新数据到达时。当缓冲区有数据可读，且应用进程对相应的描述符进行EPOLL_CTL_MOD 修改EPOLLIN事件时。
对于**写**操作：当缓冲区由不可写变为可写时。当有旧数据被发送走。当缓冲区有空间可写，且应用进程对相应的描述符进行EPOLL_CTL_MOD 修改EPOLLOUT事件时。

### python api

使用python的api时通常考虑封装好的`selectors`库，`selectors.DefaultSelector()`会基于当前运行环境自动选择多路复用的方式，比如windows下是`selectors.SelectSelector`，linux下是`selectors.EpollSelector`，非常方便。我们用selectors库改写下之前的echo服务端。

```python
import selectors
import socket

mysel = selectors.DefaultSelector()
keep_running = True

def read(connection, mask):
    "Callback for read events"
    global keep_running

    client_address = connection.getpeername()
    print('read({})'.format(client_address))
    data = connection.recv(1024)
    if data:
        # A readable client socket has data
        print('  received {!r}'.format(data))
        connection.sendall(data)
    else:
        # Interpret empty result as closed connection
        print('  closing')
        mysel.unregister(connection)
        connection.close()
        # Tell the main loop to stop
        keep_running = False


def accept(sock, mask):
    "Callback for new connections"
    new_connection, addr = sock.accept()
    print('accept({})'.format(addr))
    new_connection.setblocking(False)
    mysel.register(new_connection, selectors.EVENT_READ, read)


server_address = ('localhost', 9999)
print('starting up on {} port {}'.format(*server_address))
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setblocking(False)
server.bind(server_address)
server.listen(5)

mysel.register(server, selectors.EVENT_READ, accept)

while keep_running: # # 事件循环不断地调用select获取被激活的socket
    print('waiting for I/O')
    for key, mask in mysel.select(timeout=1):
        callback = key.data
        callback(key.fileobj, mask)

print('shutting down')
mysel.close()
```

### 问题

在一个多核机器上（假设是32核），有多个进程在同时等待tcp的accept事件，这时候一个连接起来，网络栈将会唤醒所有在这个socket上监听的进程，但是只会有一个进程是accept成功的，其他进程失败之后再次进入睡眠，如果是在一个qps特别高的业务中，一个连接进来就会唤醒一大片进程，这就是惊群问题，严重影响性能。

不过在4.5内核中，linux已经触发了这个问题，他为epoll增加了一个EPOLLEXCLUSIVE标志位，增加了这个选项之后，内核每次只会唤醒一个进程，其实在内核里面的实现也非常简单，就是在文件poll的唤醒中进行了互斥进程的唤醒，如果唤醒的第一个进程被调协了EPOLLEXCLUSIVE标志，则唤醒之后马上退出，但是这接着会带来负载不平衡的问题。

# Reference:

> [1] Unifix. (2011, Oct 31). epoll原理剖析: select&poll. [Web log post]. Retrieved from https://medium.com/@heshaobo2012/epoll原理剖析-3-select-poll-8d23b0a12906
>
> [2] Unifix. (2011, Dec 17). epoll原理剖析: epoll. [Web log post]. Retrieved from https://medium.com/@heshaobo2012/epoll原理剖析-3-epoll-bf9cdcf5e50
