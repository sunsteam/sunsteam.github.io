---
layout:     post
title:      "复习02 Java中的线程和锁"
subtitle:   ""
date:       2019-08-12 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
---


## 进程与线程

进程是程序代码运行时的实例，线程是程序内部执行代码的一条通道，同进程下的线程共享堆和方法区资源，但有独立的程序计数器、虚拟机栈、本地方法栈。多线程可以提高 cpu 的利用率，在某个线程处于 IO 等待操作时，不至于闲置 CPU 资源。多核时代，每个核心可以处理一个线程，因此常驻的线程池经常维持 CPU 核心数 + 1 个线程，合理利用 cpu 同时减少线程上下文切换的资源消耗。

## 内存模型

- 程序计数器 控制代码逻辑，如：顺序执行、选择、循环、异常处理。并且记录当前线程执行的位置，从而当线程被切换回来的时候能够知道该线程上次运行到哪儿了。如果执行的是 native 方法，那么程序计数器记录的是 undefined 地址，只有执行的是 Java 代码时程序计数器记录的才是下一条指令的地址。程序计数器私有主要是为了线程切换后能恢复到正确的执行位置。
- 虚拟机栈 运行 java 方法
- 虚拟机栈上的栈帧用于存储局部变量表、操作数栈、常量池引用等信息。从方法调用直至执行完成的过程，就对应着一个栈帧在 Java 虚拟机栈中入栈和出栈的过程。
- 本地方法栈 运行 native 方法

为保证线程中的局部变量不被别的线程访问到，虚拟机栈和本地方法栈也是线程私有的。

### 线程的生命周期

- new 初始状态，没有调用 start 方法
- runnable 等待运行或运行中
- blocked 阻塞状态
- waiting 待命状态，等待其他线程通知或中断
- time_waiting 等待状态，等待特定时间后继续执行
- terminated 终止状态，线程执行完毕终止

![](/resource/thread_transfer.png)


### start() 和 run() 区别

start() 方法会讲线程调整到 runnable 状态，并调用 native 方法向 cpu 申请时间片，轮到它执行的时候就会调用 run 方法执行任务逻辑。如果直接手动调用 run 方法，那么就是一个普通对象的方法调用，没有线程切换。

### wait() 和 sleep() 的区别

- sleep 是线程 Thread 类的方法，用于暂停线程执行。wait 是 Object 类的方法，作为锁对象时配合 notify 或 notifyAll 调用，用于线程间的调度和通知。
- sleep 方法没有释放锁，而 wait 方法释放了锁 。
- sleep 后线程会自动恢复执行，wait 后必须要别的线程中调用通知方法，线程才会恢复执行。

### 其他一些方法

- yield 让一个线程进入等待，不一定保证立即生效
- join 插入一个线程，其他线程等它执行完成后再执行，必须先调用 start 再调用 join
- setDaemon 设置一个线程为守护线程，在哪个线程调用就是哪个线程的守护线程，守护线程的生命周期和被守护者保持一致
- setPriority 设置线程申请 CPU 的优先级，默认 5，最高 10，最低 1
- interrupt   设置一个中断标志，告诉线程应该中断了。然后线程运行中可以通过 interrupted 方法获取这个标志，根据业务逻辑做处理，妥善的中断自己。线程是不推荐从外部中断的，所以该方法只是设置标志而已。
- interrupted 获取当前标志位，并清除它。相当于重置到正常状态。

### 死锁问题

死锁是多个线程间持有了对方想获得的资源，同时又在等待对方释放资源，从而导致的无限阻塞状态。

#### 形成死锁的条件

- 一个资源只能被一个线程占有
- 阻塞后不会释放已获得的资源
- 不能强制夺取对方资源
- 多个资源，获取顺序不同，从而导致可能交叉循环的情况

#### 避免死锁的方法

按相同顺序获取资源，要么都拿到要么都拿不到

### ThreadLocal

ThreadLocal 是一个工具类，用来维护线程独有的变量，实际上这些变量被保存在线程 Thread 类中的 threadLocals 变量中。该变量的类型 `ThreadLocal.ThreadLocalMap` 是一个自定义的 HashMap，简单实现了 Hash 算法和一个 Entry 数组，内部类 Entry 继承了 `WeakReference<ThreadLocal<T>>` ，每个实例以自身为 key，保存了一个 Object 类型的 value，取出时转换为 T 。

比较有意思的是计算数组位置采用了独特的 Hash 算法，每个 hash 都是新的、唯一的、并且能平均分布，因此不会造成碰撞。实现如下

```java
    /**
     * ThreadLocals rely on per-thread linear-probe hash maps attached
     * to each thread (Thread.threadLocals and
     * inheritableThreadLocals).  The ThreadLocal objects act as keys,
     * searched via threadLocalHashCode.  This is a custom hash code
     * (useful only within ThreadLocalMaps) that eliminates collisions
     * in the common case where consecutively constructed ThreadLocals
     * are used by the same threads, while remaining well-behaved in
     * less common cases.
     */
    private final int threadLocalHashCode = nextHashCode();

    /**
     * The next hash code to be given out. Updated atomically. Starts at
     * zero.
     */
    private static AtomicInteger nextHashCode =
        new AtomicInteger();

    /**
     * The difference between successively generated hash codes - turns
     * implicit sequential thread-local IDs into near-optimally spread
     * multiplicative hash values for power-of-two-sized tables.
     */
    private static final int HASH_INCREMENT = 0x61c88647;

    /**
     * Returns the next hash code.
     */
    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

```

## synchronized 关键字

可以修饰实例方法和静态方法，两者的锁对象不同，实例方法和静态方法调用不会互相阻塞。
而使用带参的 synchronized 方法块，并将相应锁对象传入后调用，那么会与对应的被修饰的方法相冲突。

- 修饰代码块:   可指定锁对象对代码块加锁
- 修饰实例方法: 锁为当前对象 `this`，等同于 `synchronized (this)`
- 修饰静态方法: 锁为当前的类字节码文件，等同于 `synchronized (A.class)`

### synchronized 关键字原理

在编译好的字节码文件中查看可以发现，synchronized 代码块被转换为 `monitorenter` 和 `monitorexit` 表示进入和退出同步。当执行 monitorenter 指令时，线程试图获取锁也就是获取 monitor(monitor 对象存在于每个 Java 对象的对象头中，synchronized 锁便是通过这种方式获取锁的，也是为什么 Java 中任意对象可以作为锁的原因) 的持有权。当计数器为 0 则可以成功获取，获取后将锁计数器设为 1 也就是加 1。相应的在执行 monitorexit 指令后，将锁计数器设为 0，表明锁被释放。如果获取对象锁失败，那当前线程就要阻塞等待，直到锁被另外一个线程释放为止。

synchronized 修饰方法时并没有 monitorenter 指令和 monitorexit 指令，取得代之的确实是方法上添加了 ACC_SYNCHRONIZED 标识，该标识指明了该方法是一个同步方法，JVM 通过该 ACC_SYNCHRONIZED 访问标志来辨别一个方法是否声明为同步方法，从而执行相应的同步调用。

### synchronized 在 1.6 中的优化

1.6 之前单纯使用 monitor，监视器锁本质又是依赖于底层的操作系统的 Mutex Lock 来实现，1.6 后增加多重锁机制，逐步升级，优化了竞争不激烈情况下的消耗。锁的状态保存在对象的头文件中，有一个标志位记录锁状态，另外其他的数据位描述锁的信息，称为 Mark Word，标志状态包含：无锁状态、偏向锁、轻量级锁、重量级锁、GC 标记。


1. 偏向锁 无竞争条件下使用，检查是否可偏向，可以的话 CAS 更改一次 Mark Word 中的 ThreadID，成功则执行代码，不行的话就说明有竞争，拆掉偏向锁，升级为轻量级锁
   
2. 轻量级锁 虚拟机栈中事先创建了一个 Lock Record 空间，用于存储锁对象目前的 Mark Word 的拷贝。
   加锁时的步骤：
   - 拷贝对象的 Mark Word 内容到这个空间
   - 同时替换锁对象 Mark Word 中的内容为指向 Lock Record 的指针
   - 将锁记录空间内标志所有者的指针指向对象的 Mark work
   - 一系列操作成功则表明申请锁成功，失败则锁升级为重量级锁，线程进入自旋，其他请求资源的线程阻塞。
   轻量级锁虽然操作多，但不涉及系统底层切换，因此相对 Mutex Lock 效率还是更高。


同时按优化操作可以分为：

- 锁消除 虚拟机确定代码不满足多线程访问条件的情况下去除同步操作指令
- 锁升级 低等级锁根据竞争情况升级为更高等级锁，只能升级不会降级
- 锁粗化 虚拟机判断多条语句内部都是相同的锁的同步方法，直接把他们合并，减少进出锁的次数，比如 StringBuffer 执行多次 append，那么就会合并加锁执行
- 自旋   指等待锁释放的线程不立即阻塞，而是通过 cpu 调用一个空转指令来自旋，由于一般释放锁的速度都较快，因此成本比上下文切换低，如果自旋完毕后还是获取不到锁，再阻塞自己
- 自适应自旋 默认的自旋次数是固定的 10 次，因此后面增加了自适应自旋的机制，会根据之前这个申请这个锁时自旋是否完成来灵活设定自旋指令次数。如果完成过 10 次，那么说明需要更长时间，增加次数，如果从未完成过，那么减少自旋次数

## ReentrantLock 与 synchronized 特点比较

他们都是可重入锁，可重入是指一个线程在获取一个锁之后还没有释放，假设要再获取一次相同的锁，那么还可以获取到。ReentrantLock 是 java 层面的，synchronized 是在虚拟机层面实现的。1.6 的优化后，两者的性能相差不大，官方推荐在满足需求的情况下使用 synchronized 实现。

ReentrantLock 最主要是提供了一些高级功能，比如:

1. 等待可中断
2. 可实现公平锁，通过 ReentrantLock(boolean fair) 构造方法来制定是否是公平的。
3. 可选择性的通知机制。ReentrantLock 可以设置多个 Condition ，该接口抽象了多线程间需要交互（等待、通知）的情形，因此一个锁可以对不同情形下的某个或某几个线程单独唤醒（signal/signalAll）了。


## volatile 关键字

由于线程在工作的时候要将数据读入 cpu 进行运算，而 cpu 和内存直接还存在高速缓存，cpu 实际上是和自己的高速缓存进行交互的，因此多线程情况下就可能存在读入工作内存后，主内存内容变化的情况。volatile 字面意思上就是 ` 容易变化的 `，它的作用就是告诉 cpu，该变量的值需要从主内存读写，从而保证了多线程下最新写入的数据的可见性。另外它还有防止 cpu 做指令重排的作用。

### 工作原理

cpu、工作内存和主内存交互指令的情形可以分为下面 4 种，他们之间的工作也是并行的，cpu 一边读写，工作内存就在保存、加载。volatile 实际上保证了写后必保存，读前必加载这样一个操作。

write 写入工作内存
save  保存到主内存

read  从工作内存读入
load  从主内存加载到工作内存

### 修饰对象变量时的作用

初始化对象分为创建内存空间、初始化对象、指向引用 3 个步骤，由于 cpu 指令重排的作用，3 个动作的顺序有可能打乱，多线程访问时，可能出现已经指向了对象，但还未初始化完成的情况，导致空指针，加上 volatile 有防止指令重排的作用，避免了异常情况。