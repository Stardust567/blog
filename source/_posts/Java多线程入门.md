---
title: Java多线程入门
date: 2018-12-12 15:58:28
tags: [大二,Java,多线程]
categories: Java
---
关于Java多线程的简单入门知识
注意协调不同线程驱动的任务之间对资源的使用
<!-- More -->
### 并发的概念
1. **进程：**每个进程都有独立的代码和数据空间（进程上下文），进程间的切换会有较大的开销，一个进程包含1--n个线程。（进程是资源分配的最小单位）
2. **线程：**同一类线程共享代码和数据空间，每个线程有独立的运行栈和程序计数器(PC)，线程切换开销小。（线程是cpu调度的最小单位）
3. **更快的执行：** 并发可以将任务分解到多个CPU上执行，但并发通常是提高运行在单处理器上的程序的性能。为什么实际运用会这么反直觉？——因为**阻塞**，当某个任务因为程序控制范围之外的条件（比如I/O）不能继续执行时，如果没有并发，那么整个主进程都将因此停止下来，直到外部条件发生变化。
4. **Java的并发：** Java的并发系统与操作系统不同，会共享类如内存和I/O这样的资源，因此Java多线程最基本的困难就在于协调不同线程驱动的任务之间对资源的使用，以使得这些资源不会同时被多个任务访问。
 * Java的线程机制是在由执行程序表示的单一进程中创建任务。
 * 对于资源各个线程是抢占式的，调度机制会周期性地中断线程，将上下文切换到另一个线程。

### 多线程实现
1. 继承Thread类
 ```Java 
class Thread1 extends Thread{
    private String name;
    public Thread1(String name) {
       this.name = name;
    }
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println(name + "运行: " + i);
            try {
                sleep((int) Math.random() * 10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
public class Main {
 
    public static void main(String[] args) {
        Thread1 mTh1=new Thread1("A");
        Thread1 mTh2=new Thread1("B");
        mTh1.start();
        mTh2.start();
    }

}
```
 程序启动运行main时候，java虚拟机启动一个进程，主线程main在main()调用时候被创建。随着调用两个对象的start方法，启动另外两个线程，这样整个应用就在多线程下成功运行了。*注意：start()方法的调用后并不是立即执行多线程代码，而是使得该线程变为可运行态（Runnable），什么时候运行是由系统决定的。*
2. 实现java.lang.Runnable接口
```Java 
class Thread2 implements Runnable{
    private String name;
    public Thread2(String name) {
        this.name=name;
    }
    @Override
    public void run() {
        for (int i = 0; i < 5; i++) {
            System.out.println(name + "运行  :  " + i);
            try {
                Thread.sleep((int) Math.random() * 10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
public class Main {
 
    public static void main(String[] args) {
        new Thread(new Thread2("C")).start();
        new Thread(new Thread2("D")).start();
    }

}
```
 Thread2类通过实现Runnable接口，使得该类有了多线程类的特征。run（）方法是多线程程序的一个约定。所有的多线程代码都在run方法里面。
 在启动的多线程的时候，需要先通过Thread类的构造方法Thread(Runnable target) 构造出对象，然后调用Thread对象的start()方法来运行多线程代码。

**实际上所有的多线程代码都是通过运行Thread的start()方法来运行的。因此，不管是扩展Thread类还是实现Runnable接口来实现多线程，最终还是通过Thread的对象的API来控制线程的。**
实现Runnable接口比继承Thread类所具有的优势：
- 适合多个相同的程序代码的线程去**处理同一个资源**
- 可以**避免java中的单继承的限制**
- 增加程序的健壮性，代码可以被多个线程共享，代码和数据独立
- 线程池只能放入实现Runable或callable类线程，不能直接放入继承Thread的类

### 线程状态切换
![](https://i.loli.net/2018/12/12/5c1109a616e2e.jpg)
1. 新建状态（New）：新创建了一个线程对象。
2. 就绪状态（Runnable）：线程对象创建后，其他线程调用了该对象的start()方法。该状态的线程位于可运行线程池中，变得可运行，等待获取CPU的使用权。
3. 运行状态（Running）：就绪状态的线程获取了CPU，执行程序代码。
4. 阻塞状态（Blocked）：阻塞状态是线程因为某种原因放弃CPU使用权，暂时停止运行。直到线程进入就绪状态，才有机会转到运行状态。阻塞的情况分三种：
 * 等待阻塞：运行的线程执行wait()方法，JVM会把该线程放入等待池中。(wait会释放持有的锁)
 * 同步阻塞：运行的线程在获取对象的同步锁时，若该同步锁被别的线程占用，则JVM会把该线程放入锁池中。
 * 其他阻塞：运行的线程执行sleep()或join()方法，或者发出了I/O请求时，JVM会把该线程置为阻塞状态。当sleep()状态超时、join()等待线程终止或者超时、或者I/O处理完毕时，线程重新转入就绪状态。_（注意,sleep是不会释放持有的锁）_
5. 死亡状态（Dead）：线程执行完了或者因异常退出了run()方法，该线程结束生命周期。

### 线程调度
1. **调整线程优先级：**优先级高的线程会获得较多的运行机会。
 Java线程的优先级用整数表示，取值范围是1~10
 Thread类有以下三个静态常量：
>`static int MAX_PRIORITY;//线程可以具有的最高优先级，取值为10。`
>`static int MIN_PRIORITY;//线程可以具有的最低优先级，取值为1。`
>`static int NORM_PRIORITY;//分配给线程的默认优先级，取值为5。`
>
 Thread类的`setPriority()`和`getPriority()`方法分别用来设置和获取线程优先级。
 每个线程都有*默认优先级*，主线程的默认优先级为`Thread.NORM_PRIORITY`
 线程的优先级有*继承关系*，比如A线程中创建了B线程，那么B将和A具有相同的优先级。
 _JVM提供了10个线程优先级，但与常见的操作系统不能很好的映射。如果希望程序能移植到各个操作系统中，应该仅仅使用Thread类有以下三个静态常量作为优先级，这样能保证同样的优先级采用了同样的调度方式。_

2. **线程睡眠：**`Thread.sleep(long millis)`方法，使线程转到阻塞状态。
 millis：设定睡眠的时间(ms)当睡眠结束后，转为就绪（Runnable）状态。
 
3. **线程等待：**Object类中的`wait()`方法，导致当前的线程等待，直到其他线程调用此对象的`notify() `方法或`notifyAll()`唤醒方法。这个两个唤醒方法也是Object类中的方法，行为等价于调用`wait(0)`一样。

4. **线程让步：**`Thread.yield()`方法，暂停当前正在执行的线程对象，把执行机会让给相同或者更高优先级的线程。
 
5. **线程加入：**`join()`方法，等待其他线程终止。
 在当前线程中调用另一个线程的join()方法，则当前线程转入阻塞状态，直到另一个进程运行结束，当前线程再由阻塞转为就绪状态。
 
6. **线程唤醒：**Object类中的`notify()`方法，唤醒在此对象监视器等待的单个线程。
 如果所有线程都在此对象上等待，则会选择唤醒其中一个线程（选择是任意性的，并在对实现做出决定时发生）
 线程通过调用其中一个 wait 方法，在对象的监视器上等待。 直到当前的线程放弃此对象上的锁定，才能继续执行被唤醒的线程。被唤醒的线程将以常规方式与在该对象上主动同步的其他所有线程进行竞争。类似的方法还有一个`notifyAll()`，唤醒在此对象监视器上等待的所有线程。

### 常用函数
1. `Thread.sleep(long millis)`：<font color=SlateGray >在指定的毫秒数内让当前正在执行的线程休眠（暂停执行）</font>
2. `Thread.join()`：<font color=SlateGray >指等待该线程终止。</font>
 该线程是指的主线程等待子线程的终止。也就是在子线程调用了join()方法后面的代码，只有等到子线程结束了才能执行。
 *在很多情况下，主线程生成并起动了子线程，如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束，但是如果主线程处理完其他的事务后，需要用到子线程的处理结果，也就是主线程需要等待子线程执行完成之后再结束，这个时候就要用到join()方法了。*
3. `Thread.yield()`：<font color=SlateGray >暂停当前正在执行的线程对象，并执行其他线程。</font>
 yield()应该做的是让当前运行线程回到可运行状态，以允许具有相同优先级的其他线程获得运行机会。因此，使用yield()的目的是让相同优先级的线程之间能适当的轮转执行。但是，实际中无法保证yield()达到让步目的，因为让步的线程还有可能被线程调度程序再次选中。
4. `Thread.setPriority()`: <font color=SlateGray >更改线程的优先级。</font>
5. `Obj.wait()`：<font color=SlateGray >暂停当前线程，释放CPU控制权，释放对象锁的控制。</font>
 <font color=MediumPurple >与Obj.notify()必须要与synchronized(Obj)一起使用，也就是wait,与notify是针对已经获取了Obj锁进行操作</font>：
 * 从语法角度来说就是Obj.wait(),Obj.notify必须在synchronized(Obj){...}语句块内。
 * 从功能上来说wait就是说线程在获取对象锁后，主动释放对象锁，同时本线程休眠。直到有其它线程调用对象的notify()唤醒该线程，才能继续获取对象锁，并继续执行。但有一点需要注意的是notify()调用后，并不是马上就释放对象锁的，而是在相应的synchronized(){}语句块执行结束，自动释放锁后，JVM会在wait()对象锁的线程中随机选取一线程，赋予其对象锁，唤醒线程，继续执行。

 线程类方法：
>sleep(): 强迫一个线程睡眠Ｎ毫秒。 
 isAlive(): 判断一个线程是否存活。 
 join(): 等待线程终止。 
 activeCount(): 程序中活跃的线程数。 
 enumerate(): 枚举程序中的线程。 
 currentThread(): 得到当前线程。 
 isDaemon(): 一个线程是否为守护线程。 
 setDaemon(): 设置一个线程为守护线程。(用户线程和守护线程的区别在于，是否等待主线程依赖于主线程结束而结束) 
 setPriority(): 设置一个线程的优先级。

### 线程同步
1. synchronized关键字的作用域： 
 * 是某个对象实例内，synchronized aMethod(){}可以防止多个线程同时访问这个对象的synchronized方法（如果一个对象有多个synchronized方法，只要一个线程访问了其中的一个synchronized方法，其它线程不能同时访问这个对象中任何一个synchronized方法）。这时，不同的对象实例的synchronized方法是不相干扰的。也就是说，其它线程照样可以同时访问相同类的另一个对象实例中的synchronized方法； 
 * 是某个类的范围，synchronized static aStaticMethod{}防止多个线程同时访问这个类中的synchronized static 方法。它可以对类的所有对象实例起作用。

 *synchronized关键字是不能继承的，继承类需要你显式的指定它的某个方法为synchronized方法；*
2. 线程同步的TIPS
 * 线程同步的目的是为了保护多个线程反问一个资源时对资源的破坏。
 * 线程同步方法是通过锁来实现，每个对象都有且仅有一个锁，这个锁与一个特定的对象关联，线程一旦获取了对象锁，其他访问该对象的线程就无法再访问该对象的其他非同步方法。
 * 对于静态同步方法，锁是针对这个类的，锁对象是该类的Class对象。静态和非静态方法的锁互不干预。一个线程获得锁，当在一个同步方法中访问另外对象上的同步方法时，会获取这两个对象锁。
 * 对于同步，要时刻清醒在哪个对象上同步，这是关键。
 * 编写线程安全的类，需要时刻注意对多个线程竞争访问资源的逻辑和安全做出正确的判断，对“原子”操作做出分析，并保证原子操作期间别的线程无法访问竞争资源。
 * 当多个线程等待一个对象锁时，没有获取到锁的线程将发生阻塞。
3. 代码示例
 创造一个`Object Lock`来充当锁，
 用Lock类的变量lockon作为是否循环wait()语句的条件：
```Java 
class Lock{
    int lockon;
    Lock(){
        this.lockon = 1;
    }
}

class ThTest implements Runnable{
    Lock lock;
    ThTest(Lock lock){
        this.lock = lock;
    }
    public void run() {
        synchronized(lock) {
            while(lock.lockon==1) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println(lock.lockon+"Lock is down.
                        Next it will be on.");
            try {
                Thread.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            lock.lockon = 1;
            lock.notify();
        }
    }
}

class ThRest implements Runnable{
    Lock lock;
    ThRest(Lock lock){
        this.lock = lock;
    }
    public void run() {
        synchronized(lock) {
            while(lock.lockon==0) {
                try {
                    lock.wait();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
            System.out.println(lock.lockon+"Lock is on.
                        Next it will be down.");
            try {
                Thread.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            lock.lockon = 0;
            lock.notify();
        }
    }
}

public class Test8_3 {
    public static void main(String args[]) {
        Lock lock = new Lock();
        ThTest thtest = new ThTest(lock);
        ThRest threst = new ThRest(lock);
        new Thread(thtest).start();
        new Thread(threst).start();
    }
}```