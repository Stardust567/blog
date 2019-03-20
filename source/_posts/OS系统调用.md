---
title: OS系统调用
copyright: true
tags:
  - 大二
  - OS
  - 进程
  - 系统调用
categories: OS
abbrlink: 22713
date: 2019-03-18 17:33:13
---
Linux内核中设置了一组用于实现各种系统功能的子程序，称为*系统调用（system call）*。同时它还提供些C语言函数库，这些库对系统调用进行包装和扩展。由于这些库函数与系统调用的关系非常紧密，习惯上把这些函数也称为系统调用。*本文是作者在大二写OS作业时整理的，安利文章[系统调用跟我学](https://www.ibm.com/developerworks/cn/linux/kernel/syscall/part3/index.html)这篇写得真的很舒服，本文很多内容都源于此。*
<!-- More -->
一般情况下进程不能访问内核所占内存也不能调用内核函数，但有一般就会有例外，系统调用就是例外。它的原理是进程先用适当的值填充寄存器，然后调用一个特殊的指令，该指令会跳到事先定义的内核中的一个位置*（该位置用户进程可读但不可写）*。
Intel CPU中，这个由中断0x80实现。硬件知道一旦你跳到*这个位置（system_call）*，你就不再是限制模式下运行的用户，而“成为”了操作系统的内核~~接下来你就可以为所欲为~~。这个过程会检查系统调用号，该号码告诉内核进程请求哪种服务。然后查看*系统调用表(sys_call_table)*找到所调用的内核函数入口地址。接着调用函数，等返回后做一些系统检查，最后返回到进程（或到其他进程，如果这个进程时间用尽）。

**进程是可并发执行的程序在一个数据集合上的运行过程。**

为防止和正常的返回值混淆，系统调用并不直接返回错误码，而是将错误码放入一个名为errno的全局变量中。若系统调用失败，可以读出errno的值来锁定问题。errno不同数值所代表的错误消息定义在errno.h中，可以通过命令`man 3 errno`来察看它们。
需要注意的是，errno的值只在函数发生错误时设置，如果函数不发生错误，errno的值就无定义，并不会被置为0。另外，在处理errno前最好先把它的值存入另一个变量，因为在错误处理过程中，即使像printf()这样的函数出错时也会改变errno的值。

## getpid
在2.4.4版内核中，getpid是第20号系统调用，其在Linux函数库中的原型是：
>`#include<sys/types.h> /* 提供类型pid_t的定义 */`
>`#include<unistd.h> /* 提供函数的定义 */`
>`pid_t getpid(void);`
>
getpid的作用很简单，就是返回当前进程的进程ID
```C

#include<unistd.h>
main()
{
    printf("The current process ID is %d\n",getpid());
}
```
>The current process ID is 1980
>
*注意，该程序的定义里并没包含头文件sys/types.h，这是因为我们在程序中没有用到pid_t类型，pid_t类型即为进程ID的类型。事实上，在i386架构上（就是我们一般PC计算机的架构），pid_t类型是和int类型完全兼容的，我们可以就把它当做个整形，比如用"%d"把它打印出来。*

## fork
在2.4.4版内核中，fork是第2号系统调用，其在Linux函数库中的原型是：
>`#include<sys/types.h> /* 提供类型pid_t的定义 */`
>`#include<unistd.h> /* 提供函数的定义 */`
>`pid_t fork(void);`
>
fork系统调用的作用是复制一个进程。当一个进程调用它，完成后就出现两个几乎一模一样的进程，我们也由此得到了一个新进程。
```C

#include<sys/types.h>
#inlcude<unistd.h>
main()
{
    pid_t pid;    
    /*此时仅有一个进程*/
    pid=fork();
    /*此时已经有两个进程在同时运行*/
    if(pid<0) printf("error in fork!");
      else if(pid==0)
          printf("I am the child process, my process ID is %d\n",getpid());
      else
          printf("I am the parent process, my process ID is %d\n",getpid());
}
```
>I am the parent process, my process ID is 1991
>I am the child process, my process ID is 1992
>
看这个程序的时候，头脑中必须首先了解一个概念：在语句pid=fork()之前，只有一个进程在执行这段代码，但在这条语句之后，就变成两个进程在执行了，这两个进程的代码部分完全相同，将要执行的下一条语句都是if(pid==0)
两个进程中，原先就存在的那个被称作“父进程”，新出现的那个被称作“子进程”。父子进程的区别除了进程标志符PID不同外，变量pid的值也不相同，pid存放的是fork的返回值。fork调用的一个奇妙之处就是它仅仅被调用一次，却能够返回两次，它可能有三种不同的返回值：
1. 在父进程中，fork返回新创建子进程的进程ID；
2. 在子进程中，fork返回0；
3. 如果出现错误，fork返回一个负值(当前的进程数已经达到了系统规定的上限*这时errno的值被设置为EAGAIN*或系统内存不足*这时errno的值被设置为ENOMEM*）。

## exit
在2.4.4版内核中，exit是第1号调用，其在Linux函数库中的原型是：
>`#include<stdlib.h>`
>`void exit(int status);`
>
这个系统调用是用来终止一个进程的。无论在程序中的什么位置，只要执行到exit系统调用，进程就会停止剩下的所有操作，清除包括PCB在内的各种数据结构，并终止本进程的运行。
```C

#include<stdlib.h>
main()
{
    printf("this process will exit!\n");
    exit(0);
    printf("never be displayed!\n");
}
```
>this process will exit!
>
进程在`exit(0)`处直接终止，并不会打印后面的printf。exit带有一个整型的参数status，可以用这个参数传递进程结束时的状态，比如正常结束的为0。

## _exit
_exit在Linux函数库中的原型是：
>`#include<unistd.h>`
>`void _exit(int status);`
>
和exit比较一下，`exit()`函数定义在stdlib.h中，而`_exit()`定义在unistd.h中。
但两者真正的区别在于exit()函数在调用exit系统调用之前要检查文件的打开情况，把文件缓冲区中的内容写回文件，也就是所谓的“清理I/O缓冲”。
举个例子，如果同样的两行printf，加上不同的终止调用，会发生什么：
>`main(){`
>`    printf("output begin\n");`
>`    printf("content in buffer");`
>`    exit(0); # _exit(0);`
>`}`
>
`exit(0)`会完成两句printf,但`_exit(0)`可能完成第一句printf就终止了。这应该不难理解，系统先把前两个printf存入buff，然后一边I/O一边往后执行，如果是`_exit(0)`不管缓存死活，那就直接结束了啊。
但是exit后的进程并不是就灰飞烟灭了，它有个让人毛骨悚然的名字，*僵尸进程（Zombie）*

## 进程状态
下面简单介绍下进程的状态：分linux内核代码定义的状态和常用状态。
由Linux内核代码宏定义出的“TASK_REPORT”：
>`/* get_task_state() */`
>`#define TASK_REPORT	(TASK_RUNNING | TASK_INTERRUPTIBLE | `
>`			  TASK_UNINTERRUPTIBLE | __TASK_STOPPED |>`
>`             __TASK_TRACED | EXIT_ZOMBIE | EXIT_DEAD)`
>
可以得到七个基本的进程状态，即：
**R运行状态TASK_RUNNING**进程要么正在执行，要么正要准备执行。
**S可中断睡眠状态TASK_INTERRUPTIBLE**进程被阻塞，直到某个条件变为真。条件一旦达成，进程的状态就被设置为TASK_RUNNING。
**D不可中断睡眠状态TASK_UNINTERRUPTIBLE**进程不会立即响应信号。这种状态一般用于内核某些不能被打断的进程，比如等待磁盘或网络I / O的设备驱动程序使用。
**T停止状态__TASK_STOPPED**可以通过发送SIGSTOP信号给进程来停止进程，可以发送SIGCONT信号让进程继续运行
**t追踪状态__TASK_TRACED**进程被debugger等进程监视。处于TASK_TRACED状态的进程不能响应SIGCONT信号而被唤醒。
**Z僵尸状态EXIT_ZOMBIE**进程的执行被终止，但是其父进程还没有使用wait()等系统调用来获知它的终止信息。此时进程几乎的所有资源将被回收，没有任何可执行代码，也不能被调度，仅留下task_struct结构（以及少数资源）记载了些供人凭吊的信息。
**X死亡状态EXIT_DEAD**进程的最终状态，即将被销毁，ls都没了的那种。
常用的状态转换图：
![](https://i.loli.net/2019/03/18/5c8fa09144527.jpg)


## wait
wait的函数原型是：
>`#include <sys/types.h> /* 提供类型pid_t的定义 */`
>`#include <sys/wait.h>`
>`pid_t wait(int *status)`
>
进程一旦调用了wait，就立即阻塞自己，由wait自动分析是否当前进程的某个子进程已经退出，如果让它找到了这样一个已经变成僵尸的子进程，wait就会收集这个子进程的信息，并把它彻底销毁后返回；如果没有找到这样一个子进程，wait就会一直阻塞在这里，直到有一个出现为止。
参数status用来保存被收集进程退出时的一些状态，它是一个指向int类型的指针。但如果我们对这个子进程是如何死掉的毫不在意，只想把这个僵尸进程消灭掉（多数情况如此），我们就可以设定这个参数为NULL:`pid = wait(NULL);`如果成功，wait会返回被收集的子进程的PID，如果调用进程没有子进程，调用就会失败，此时wait返回-1*errno被置为ECHILD*。
如果想知道status，那就准备个int指针，也可以`int status`然后传`wait(&status)`这时就可以调用专门的*宏（macro）*来获取信息：
1. WIFEXITED(status) 这个宏用来指出子进程是否为正常退出的，如果是，它会返回一个非零值。
2. WEXITSTATUS(status) 这个宏用来提取子进程的返回值，如果子进程调用exit(5)退出，WEXITSTATUS(status)就会返回5*（注意，如果进程不是正常退出的，也就是说，WIFEXITED返回0，这个值就毫无意义）*

## waitpid
waitpid的函数原型是：
>`#include <sys/types.h> /* 提供类型pid_t的定义 */`
>`#include <sys/wait.h>`
>`pid_t waitpid(pid_t pid,int *status,int options)`
>
just多了pid和options两个参数：
* 参数 pid 为欲等待的子进程的识别码：
	* pid < -1 ；等待进程组 ID 为 pid 绝对值的进程组中的任何子进程；
	* pid = -1 ；等待任何子进程，此时 waitpid() 相当于 wait()。实际上，wait()就是 pid = -1、options = 0 的waitpid()
	* pid = 0 ；等待进程组 ID 与当前进程相同的任何子进程（也就是等待同一个进程组中的任何子进程）；
	* pid > 0 ；等待任何子进程 ID 为 pid 的子进程，只要指定的子进程还没有结束，waitpid() 就会一直等下去。
* 参数 options提供一些额外的选项来控制 waitpid()：
	* WNOHANG；如果没有任何已经结束了的子进程，则马上返回，不等待；
	* WUNTRACED；如果子进程进入暂停执行的情况，则马上返回，但结束状态不予理会；
	* 也可以将这两个选项组合起来使用，使用 OR 操作
	* 如果不想使用这两个选项，也可以直接把 options 设为0，则 waitpid() 会一直等待，直到有进程退出
* waitpid()的返回值，有三种：
	* 正常返回时，waitpid() 返回收集到的子进程的PID；
	* 如果设置了 WNOHANG，而调用 waitpid() 时，没有发现已退出的子进程可收集，则返回0；
	* 如果调用出错，则返回 -1，这时erron 会被设置为相应的值以指示错误所在。（当 pid 所指示的子进程不存在，或此进程存在，但不是调用进程的子进程， waitpid() 就会返回出错，这时 erron 被设置为 ECHILD）

```C

#include <sys/types.h>
#include <sys/wait.h>
#include <unistd.h>
main()
{
    pid_t pc, pr;
         
    pc=fork();
    if(pc<0)     /* 如果fork出错 */
        printf("Error occured on forking.\n");
    else if(pc==0){     /* 如果是子进程 */
        sleep(10);  /* 睡眠10秒 */
        exit(0);
    }
    /* 如果是父进程 */
    do{
        pr=waitpid(pc, NULL, WNOHANG);  /* WNOHANG参数:waitpid不会在这等待 */
        if(pr==0){          /* 如果没有收集到子进程 */
            printf("No child exited\n");
            sleep(1);
        }
    }while(pr==0);              /* 没有收集到子进程，就回去继续尝试 */
    if(pr==pc)
        printf("successfully get child %d\n", pr);
    else
        printf("some error occured\n");
}
```
>No child exited
No child exited
No child exited
No child exited
No child exited
No child exited
No child exited
No child exited
No child exited
No child exited
successfully get child 1526
>
我们让父进程和子进程分别睡眠了10秒钟和1秒钟，代表它们分别作了10秒钟和1秒钟的工作。父子进程都有工作要做，父进程利用工作的简短间歇察看子进程的是否退出，如退出就收集它。

## exec
*Question*:既然所有新进程都是由fork产生的，而且由fork产生的子进程和父进程几乎完全一样，那岂不是意味着系统中所有的进程都应该一模一样了吗？而且，就我们的常识来说，当我们执行一个程序的时候，新产生的进程的内容应就是程序的内容才对。是我们理解错了吗？
实际上在Linux中，exec指的是一组函数，一共有6个，分别是：
>`#include <unistd.h>`
>`int execl(const char *path, const char *arg, ...);`
>`int execlp(const char *file, const char *arg, ...);`
>`int execle(const char *path, const char *arg, ..., char *const envp[]);`
>`int execv(const char *path, char *const argv[]);`
>`int execvp(const char *file, char *const argv[]);`
>`int execve(const char *path, char *const argv[], char *const envp[]);`
>
*其中只有execve是真正意义上的系统调用，其它都是在此基础上经过包装的库函数。*

exec函数族的作用是根据指定的文件名找到可执行文件，并用它来取代调用进程的内容，换句话说，就是在调用进程内部执行一个可执行文件。这里的可执行文件既可以是二进制文件，也可以是任何Linux下可执行的脚本文件。

与一般情况不同，exec函数族的函数执行成功后不会返回，因为调用进程的实体，包括代码段，数据段和堆栈等都已经被新的内容取代，只留下进程ID等一些表面上的信息仍保持原样，颇有些神似"三十六计"中的"金蝉脱壳"。看上去还是旧的躯壳，却已经注入了新的灵魂。只有调用失败了，它们才会返回一个-1，从原程序的调用点接着往下执行。

现在我们应该明白了，Linux下是如何执行新程序的，每当有进程认为自己不能为系统和拥护做出任何贡献了，他就可以发挥最后一点余热，调用任何一个exec，让自己以新的面貌重生；或者，更普遍的情况是，如果一个进程想执行另一个程序，它就可以fork出一个新进程，然后调用任何一个exec，这样看起来就好像通过执行应用程序而产生了一个新进程一样。

事实上第二种情况被应用得如此普遍，以至于Linux专门为其作了优化，我们已经知道，fork会将调用进程的所有内容原封不动的拷贝到新产生的子进程中去，这些拷贝的动作很消耗时间，而如果fork完之后我们马上就调用exec，这些辛辛苦苦拷贝来的东西又会被立刻抹掉，这看起来非常不划算，于是人们设计了一种*写时拷贝（copy-on-write）*技术，使得fork结束后并不立刻复制父进程的内容，而是到了真正实用的时候才复制，这样如果下一条语句是exec，它就不会白白作无用功了，也就提高了效率。

在学习它们之前，先来了解一下我们习以为常的main函数。
>`int main(int argc, char *argv[], char *envp[])`
>
参数argc指出了运行该程序时命令行参数的个数，数组argv存放了所有的命令行参数，数组envp存放了所有的环境变量。环境变量指的是一组值，从用户登录后就一直存在，很多应用程序需要依靠它来确定系统的一些细节，我们最常见的环境变量是PATH，它指出了应到哪里去搜索应用程序，如/bin；HOME也是比较常见的环境变量，它指出了我们在系统中的个人目录。环境变量一般以字符串"XXX=xxx"的形式存在，XXX表示变量名，xxx表示变量的值。

值得一提的是，argv数组和envp数组存放的都是指向字符串的指针，这两个数组都以一个NULL元素表示数组的结尾。

我们可以通过以下这个程序来观看传到argc、argv和envp里的都是什么东西：
```C
int main(int argc, char *argv[], char *envp[])
{
    printf("\n### ARGC ###\n%d\n", argc);
    printf("\n### ARGV ###\n");
    while(*argv)
        printf("%s\n", *(argv++));
    printf("\n### ENVP ###\n");
    while(*envp)
        printf("%s\n", *(envp++));
    return 0;
}
```
编译`cc main.c -o main`然后运行，故意加几个没有任何作用的命令行参数`./main -xx 000` 
>\### ARGC ###
3
\### ARGV ###
./main
-xx
000
\### ENVP ###
PWD=/home/lei
REMOTEHOST=dt.laser.com
HOSTNAME=localhost.localdomain
QTDIR=/usr/lib/qt-2.3.1
LESSOPEN=|/usr/bin/lesspipe.sh %s
KDEDIR=/usr
USER=lei
LS_COLORS=
MACHTYPE=i386-redhat-linux-gnu
MAIL=/var/spool/mail/lei
INPUTRC=/etc/inputrc
LANG=en_US
LOGNAME=lei
SHLVL=1
SHELL=/bin/bash
HOSTTYPE=i386
OSTYPE=linux-gnu
HISTSIZE=1000
TERM=ansi
HOME=/home/lei
PATH=/usr/local/bin:/bin:/usr/bin:/usr/X11R6/bin:/home/lei/bin
_=./main
>
我们看到，程序将"./main"作为第1个命令行参数，所以共有3个命令行参数。这可能与平时习惯的说法有些不同。
现在回过头来看一下exec函数族，先把注意力集中在execve上：
>`int execve(const char *path, char *const argv[], char *const envp[]);`
>
对比一下main函数的完整形式，就会发现这两个函数里的argv和envp是完全一一对应的关系。execve第1个参数path是被执行应用程序的完整路径，第2个参数argv就是传给被执行应用程序的命令行参数，第3个参数envp是传给被执行应用程序的环境变量。
留心看一下这6个函数还可以发现，前3个函数都是以execl开头的，后3个都是以execv开头的，它们的区别在于，execv开头的函数是以"char *argv[]"这样的形式传递命令行参数，而execl开头的函数采用了我们更容易习惯的方式，把参数一个一个列出来，然后以一个NULL表示结束。这里的NULL的作用和argv数组里的NULL作用是一样的。

在全部6个函数中，只有execle和execve使用了char *envp[]传递环境变量，其它的4个函数都没有这个参数，这并不意味着它们不传递环境变量，这4个函数将把默认的环境变量不做任何修改地传给被执行的应用程序。而execle和execve会用指定的环境变量去替代默认的那些。

还有2个以p结尾的函数execlp和execvp，咋看起来，它们和execl与execv的差别很小，事实也确是如此，除execlp和execvp之外的4个函数都要求，它们的第1个参数path必须是一个完整的路径，如"/bin/ls"；而execlp和execvp的第1个参数file可以简单到仅仅是一个文件名，如"ls"，这两个函数可以自动到环境变量PATH制定的目录里去寻找。
```C

#include <unistd.h>
main()
{
    char *envp[]={"PATH=/tmp",
            "USER=lei",
            "STATUS=testing",
            NULL};
    char *argv_execv[]={"echo", "excuted by execv", NULL};
    char *argv_execvp[]={"echo", "executed by execvp", NULL};
    char *argv_execve[]={"env", NULL};
    if(fork()==0)
        if(execl("/bin/echo", "echo", "executed by execl", NULL)<0)
            perror("Err on execl");
    if(fork()==0)
        if(execlp("echo", "echo", "executed by execlp", NULL)<0)
            perror("Err on execlp");
    if(fork()==0)
        if(execle("/usr/bin/env", "env", NULL, envp)<0)
            perror("Err on execle");
    if(fork()==0)
        if(execv("/bin/echo", argv_execv)<0)
            perror("Err on execv");
    if(fork()==0)
        if(execvp("echo", argv_execvp)<0)
            perror("Err on execvp");
    if(fork()==0)
        if(execve("/usr/bin/env", argv_execve, envp)<0)
            perror("Err on execve");
}
```
程序里调用了2个Linux常用的系统命令，echo和env。echo会把后面跟的命令行参数原封不动的打印出来，env用来列出所有环境变量。

由于各个子进程执行的顺序无法控制，所以有可能出现一个比较混乱的输出--各子进程打印的结果交杂在一起，而不是严格按照程序中列出的次序。
>executed by execl
PATH=/tmp
USER=lei
STATUS=testing
executed by execlp
excuted by execv
executed by execvp
PATH=/tmp
USER=lei
STATUS=testing
>
果然不出所料，execle输出的结果跑到了execlp前面。

在平时的编程中，如果用到了exec函数族，一定记得要加错误判断语句。因为与其他系统调用比起来，exec很容易受伤，被执行文件的位置，权限等很多因素都能导致该调用的失败。最常见的错误是：
* 找不到文件或路径，此时errno被设置为ENOENT；
* 数组argv和envp忘记用NULL结束，此时errno被设置为EFAULT；
* 没有对要执行文件的运行权限，此时errno被设置为EACCES。

## 总结
下面就让我用一些形象的比喻，来对进程短暂的一生作一个小小的总结：

随着一句fork，一个新进程呱呱落地，但它这时只是老进程的一个克隆。
然后随着exec，新进程脱胎换骨，离家独立，开始了为人民服务的职业生涯。
人有生老病死，进程也一样，它可以是自然死亡，即运行到main函数的最后一个"}"，从容地离我们而去；也可以是自杀，自杀有2种方式，一种是调用exit函数，一种是在main函数内使用return，无论哪一种方式，它都可以留下遗书，放在返回值里保留下来；它还甚至能可被谋杀，被其它进程通过另外一些方式结束他的生命。
进程死掉以后，会留下一具僵尸，wait和waitpid充当了殓尸工，把僵尸推去火化，使其最终归于无形。

这就是进程完整的一生。