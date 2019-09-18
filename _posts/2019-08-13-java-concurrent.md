---
layout:     post
title:      "复习03 Java中的并发处理"
subtitle:   ""
date:       2019-08-13 12:00:00
author:     "Yomii"
catalog:    true
tags:
    - Java
---


## 原子类

使用了 CAS 操作的乐观锁，主要操作通过 Unsafe 类来进行。原理是写操作之前比较旧值和当前值是否一致，一致表示没人改过，可以更新，不一致表示改过了，不可执行，然后进行自旋重试。它是利用 cpu 指令集提供的 compare and swap 指令进行操作，效率高，适合多读少写场景。如果写过多，一直重试会造成 cpu 资源浪费。

Java 提供了 AtomicInteger、AtomicLong、AtomicReference 和他们的数组类型来满足常用的非锁同步需求。并且在其他并发的处理实现中中也可以结合他们使用。CAS 的操作对象 value 需要加上 `volatile` 确保修改后多线程可见。

```java
public class AtomicInteger extends Number implements java.io.Serializable {
    private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

    ....

}
```

### CAS 的 ABA 问题

假设比较时，变量已经被改成过其他值，又改回原值，那么原子类是无法判断出来的，因此 AtomicStampedReference 增加了一个 stamp 标准来满足这样的判断需求。

## 线程池

线程池维持一定数量的常驻线程 + 一个任务队列来处理多任务，减少开关线程造成的消耗。构造指定了不同的核心线程数、任务队列、最大线程数、线程存活时间、处理量满载之后的策略等。其他还提供了核心线程预加载、核心线程是否可销毁等优化配置，满足多种场景需要。



线程池的实现包括以下几点：


```java
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3;
    private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

    // runState is stored in the high-order bits
    private static final int RUNNING    = -1 << COUNT_BITS;
    private static final int SHUTDOWN   =  0 << COUNT_BITS;
    private static final int STOP       =  1 << COUNT_BITS;
    private static final int TIDYING    =  2 << COUNT_BITS;
    private static final int TERMINATED =  3 << COUNT_BITS;

    // Packing and unpacking ctl
    private static int runStateOf(int c)     { return c & ~CAPACITY; }
    private static int workerCountOf(int c)  { return c & CAPACITY; }
    private static int ctlOf(int rs, int wc) { return rs | wc; }
```

这个 `ctl` 变量同时维护线程池状态和线程数量，这样设计的好处是两者可以保证是原子操作同时更改。高三位表示状态，其他位表示线程数量。这种设计方式是可以借鉴的。

```java
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        /**
         * This class will never be serialized, but we provide a
         * serialVersionUID to suppress a javac warning.
         */
        private static final long serialVersionUID = 6138294804551838833L;

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        /** Per-thread task counter */
        volatile long completedTasks;

        /**
         * Creates with given first task and thread from ThreadFactory.
         * @param firstTask the first task (null if none)
         */
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        /** Delegates main run loop to outer runWorker  */
        public void run() {
            runWorker(this);
        }

       .....
    }
```

Worker 内部类实现了 AbstractQueuedSynchronizer ，内部包含一个 Thread，一个初始化的任务。当线程池判断需要增加线程的时候，他其实启动的是 worker，并给他一个初始化任务，worker 会创建自己的工作线程处理任务，处理完成之后他不会关闭，而是查看任务队列是否还有任务，有的话取出来继续工作。线程的资源同步和状态切换工作都是由它对 AQS 框架的实现完成的。

### 线程池中的线程创建 / 销毁

首先线程的创建逻辑是这样的:

当前线程数 < 核心线程数 --> addWorker
当前线程数 >= 核心线程数 && 队列没满 --> 添加到队列等待 worker 取出
当前线程数 >= 核心线程数 && 队列已满 && 最大线程数未满 --> addWorker
当前线程数 >= 核心线程数 && 队列已满 && 最大线程数已满 --> 执行满载处理策略

销毁的逻辑

判断是否销毁线程的逻辑在 `getTask` 方法中，判断条件比较复杂，整体如下。最后 getTask() 的返回结果为 null，runWorker 任务不会继续执行，直接进 finally 代码块中执行 processWorkerExit ，从 workers 的 HashSet 容器中移除这个工作线程。

```java
private Runnable getTask() {
        boolean timedOut = false; // Did the last poll() time out?

        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
                decrementWorkerCount();
                return null;
            }

            int wc = workerCountOf(c);

            // Are workers subject to culling?
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c))
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true;
            } catch (InterruptedException retry) {
                timedOut = false;
            }
        }
    }

```

```java
final void runWorker(Worker w) {
        Thread wt = Thread.currentThread();
        Runnable task = w.firstTask;
        w.firstTask = null;
        w.unlock(); // allow interrupts
        boolean completedAbruptly = true;
        try {
            while (task != null || (task = getTask()) != null) {
                w.lock();
                // If pool is stopping, ensure thread is interrupted;
                // if not, ensure thread is not interrupted.  This
                // requires a recheck in second case to deal with
                // shutdownNow race while clearing interrupt
                if ((runStateAtLeast(ctl.get(), STOP) ||
                     (Thread.interrupted() &&
                      runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                    wt.interrupt();
                try {
                    beforeExecute(wt, task);
                    Throwable thrown = null;
                    try {
                        task.run();
                    } catch (RuntimeException x) {
                        thrown = x; throw x;
                    } catch (Error x) {
                        thrown = x; throw x;
                    } catch (Throwable x) {
                        thrown = x; throw new Error(x);
                    } finally {
                        afterExecute(task, thrown);
                    }
                } finally {
                    task = null;
                    w.completedTasks++;
                    w.unlock();
                }
            }
            completedAbruptly = false;
        } finally {
            processWorkerExit(w, completedAbruptly);
        }
    }
```

关键判断语句可以理解为如下

```java
/*
* 大于最大线程数量基本不会发生，可以忽略
* 
* 可以理解为 允许核心线程销毁或当期工作线程大于核心线程数，并且任务队列已经无新任务，并且当期线程数大于 1
* 接下来就会做线程数递减，直到不满足判断条件
*
*/
if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty()))
```

### 线程池任务的执行

简单来说就是，worker 自己的 firstWork 判断是不是 null，是的话从 getTask 方法取一个，假如没有新任务可取，`workQueue.poll()` 或 `workQueue.take()` 方法的实现队列中会用 AQS 的 `await()` 通知线程阻塞，假如取出了任务，那么 worker 中的 AQS 实现会把线程上锁，然后执行 `task.run()`，完成之后任务执行计数器递增，解锁线程。


## AbstractQueuedSynchronizer (AQS)

AQS 把线程封装为节点 (Node)，各节点包含等待竞争、取消、等待唤醒、共享锁等锁的候选者状态，而 AQS 通过 CAS 操作让线程排队竞争头节点 (head) 的位置来实现加锁功能，竞争失败的都用 CAS 操作去队尾 (tail) 节点列队，形成一个 FIFO 队列。这样就形成了以下队列

head 当前获得锁线程 -》 第二名 for 循环自旋竞争锁 标记自己状态为 SINGNAL -》后续节点判断前一个已经是 SINGNAL，通过 LockSupport 类（底层 Unsafe 类）的 park 方法进入阻塞

head 节点释放锁后把 state 置为 0，通过 unpark 唤醒下个节点获取锁

共享锁的实现: AQS 在共享锁的时候判断 tryAcquireShared 的返回值，如果 < 0 ，那么共享锁的传播会停止，后面的就阻塞了

![](resource/aqs.jpg)


贴一下 Node 的 waitState 注释

```java
static final class Node {
        /** Marker to indicate a node is waiting in shared mode */
        static final Node SHARED = new Node();
        /** Marker to indicate a node is waiting in exclusive mode */
        static final Node EXCLUSIVE = null;

        /** waitStatus value to indicate thread has cancelled */
        static final int CANCELLED =  1;
        /** waitStatus value to indicate successor's thread needs unparking */
        static final int SIGNAL    = -1;
        /** waitStatus value to indicate thread is waiting on condition */
        static final int CONDITION = -2;
        /**
         * waitStatus value to indicate the next acquireShared should
         * unconditionally propagate
         */
        static final int PROPAGATE = -3;

        /**
         * Status field, taking on only the values:
         *   SIGNAL:     The successor of this node is (or will soon be)
         *               blocked (via park), so the current node must
         *               unpark its successor when it releases or
         *               cancels. To avoid races, acquire methods must
         *               first indicate they need a signal,
         *               then retry the atomic acquire, and then,
         *               on failure, block.
         *   CANCELLED:  This node is cancelled due to timeout or interrupt.
         *               Nodes never leave this state. In particular,
         *               a thread with cancelled node never again blocks.
         *   CONDITION:  This node is currently on a condition queue.
         *               It will not be used as a sync queue node
         *               until transferred, at which time the status
         *               will be set to 0. (Use of this value here has
         *               nothing to do with the other uses of the
         *               field, but simplifies mechanics.)
         *   PROPAGATE:  A releaseShared should be propagated to other
         *               nodes. This is set (for head node only) in
         *               doReleaseShared to ensure propagation
         *               continues, even if other operations have
         *               since intervened.
         *   0:          None of the above
         *
         * The values are arranged numerically to simplify use.
         * Non-negative values mean that a node doesn't need to
         * signal. So, most code doesn't need to check for particular
         * values, just for sign.
         *
         * The field is initialized to 0 for normal sync nodes, and
         * CONDITION for condition nodes.  It is modified using CAS
         * (or when possible, unconditional volatile writes).
         */
        volatile int waitStatus;
}
```

`waitStatus` 字段为同步状态，其中 state > 0 为有锁状态，每次加锁就在原有 state 基础上加 1，即代表当前持有锁的线程加了 state 次锁，反之解锁时每次减 1，当 waitStatus = 0 为无锁状态；

### 子类需要实现的方法

独占锁

```java
// 获取锁方法
protected boolean tryAcquire(int arg) {
  throw new UnsupportedOperationException();
}
// 释放锁方法
protected boolean tryRelease(int arg) {
  throw new UnsupportedOperationException();
}
```

共享锁

```java
// 获取锁方法
protected int tryAcquireShared(int arg) {
  throw new UnsupportedOperationException();
}
// 释放锁方法
protected boolean tryReleaseShared(int arg) {
  throw new UnsupportedOperationException();
}
```


### 使用 AQS 机制的组件

- ReentrantLock
- Semaphore
- CountDownLatch
- CyclicBarrier


### ReentrantLock 中公平锁的实现

代码比较简单，由于使用了 AQS 机制，只需要实现独占锁的 2 个方法就行了，非公平锁就是用的父类 Sync 的实现，公平锁的区别是多了用 hasQueuedPredecessors 判断一下自己等的时间是不是最长了。另外 ReentrantLock 中 `lock` 方法会调用 `tryAcquire` 实现，`tryLock` 方法是非公平锁，没抢到就立即返回失败。

```java
    /**
     * Sync object for fair locks
     */
    static final class FairSync extends Sync {
        private static final long serialVersionUID = -3000897897090466540L;

        final void lock() {
            acquire(1);
        }

        /**
         * Fair version of tryAcquire.  Don't grant access unless
         * recursive call or no waiters or is first.
         */
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 区别是在这里使用了 hasQueuedPredecessors 判断是否有比当前线程等待的时间更长的线程
                // 没有的话并且竞争成功之后设置一个标志位设置当前线程为独占
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            }
            // 这边是 state 已经不为 0 的时候，能到这里表示已经被当前线程独占了，所以下面 + 1 的时候不需要用 CAS 了
            else if (current == getExclusiveOwnerThread()) {
                int nextc = c + acquires;
                if (nextc < 0)
                    throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

    // 就是判断下前面没等着的节点了
    public final boolean hasQueuedPredecessors() {
        // The correctness of this depends on head being initialized
        // before tail and on head.next being accurate if the current
        // thread is first in queue.
        Node t = tail; // Read fields in reverse initialization order
        Node h = head;
        Node s;
        return h != t &&
            ((s = h.next) == null || s.thread != Thread.currentThread());
    }

```

### Semaphore 

原理是构造中传入一个信号数量，每有一个线程获取到就往下减 1，然后传递给下个线程获取，直到 remaining 值 < 0，AQS 中的调用就会停止，并且后面队列转入阻塞，释放的时候就是反过来，上层再唤醒。

加锁、解锁是 Sync 中的下面 2 个方法

```java
    final int nonfairTryAcquireShared(int acquires) {
        for (;;) {
            int available = getState();
            int remaining = available - acquires;
            if (remaining < 0 ||
                compareAndSetState(available, remaining))
                return remaining;
        }
    }

    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            if (compareAndSetState(current, next))
                return true;
        }
    }

```

### CountDownLatch

一般有两种业务上的用法，我觉得形象点可以成为发令枪和终点线，再加一种判断死锁条件是否成立的技巧性用法。

- 发令枪，所有线程先 `await` 好，之后 `coutdown` 方法设置 `state` 为 0，所有线程一起跑，重点在同时起跑
- 终点线，有一个处理最终任务的线程在等待，所有业务线程不管何时跑，把业务跑完就等待或终止，调用一次 `countDown`，最后一个线程（选手）处理完之后，裁判（之前在等待的线程）被唤醒，执行收尾工作，一般是各业务的结果汇总之类的。

原理上是一个直接把 AQS 的 `state` 作为倒数器，大于 0 表示上了锁，等于 0 表示没锁。初始化后是上了锁的，await 自然拿不到，countdown 到 0 之后，会做 unpark 操作唤醒，然后各个线程就能分享锁了。

另外，它的内部类 Sync 是这样实现的，因此，只要 `state = 0` ，那么所有队列中的线程都能启动。

```java
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }
```

`await` 的内部调用了下面这个方法，先试着拿锁，拿不到就去后面排队

```java

    public final void acquireSharedInterruptibly(int arg)
            throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        if (tryAcquireShared(arg) < 0)
            doAcquireSharedInterruptibly(arg);
    }
```

### CyclicBarrier

以下引用自 [javaGuide: CyclicBarrier 和 CountDownLatch 区别][javaGuide]，总结的比较好，直接用了。

CyclicBarrier 和 CountDownLatch 功能有点像，但又有不同。

CountDownLatch 是计数器，只能使用一次，而 CyclicBarrier 的计数器提供 reset 功能，可以多次使用。但是我不那么认为它们之间的区别仅仅就是这么简单的一点。我们来从 jdk 作者设计的目的来看，javadoc 是这么描述它们的：

>CountDownLatch: A synchronization aid that allows one or more threads to wait until a set of operations being performed in other threads completes.(CountDownLatch: 一个或者多个线程，等待其他多个线程完成某件事情之后才能执行；) CyclicBarrier : A synchronization aid that allows a set of threads to all wait for each other to reach a common barrier point.(CyclicBarrier : 多个线程互相等待，直到到达同一个同步点，再继续一起执行。

对于 CountDownLatch 来说，重点是 “一个线程（多个线程）等待”，而其他的 N 个线程在完成“某件事情” 之后，可以终止，也可以等待。而对于 CyclicBarrier，重点是多个线程，在任意一个线程没有完成，所有的线程都必须等待。

CountDownLatch 是计数器，线程完成一个记录一个，只不过计数不是递增而是递减，而 CyclicBarrier 更像是一个阀门，需要所有线程都到达，阀门才能打开，然后继续执行。

![](/resource/CyclicBarrier_vs_CountDownLatch.png)

[javaGuide]: https://snailclimb.gitee.io/javaguide/#/java/Multithread/AQS?id=_53-cyclicbarrier%e5%92%8ccountdownlatch%e7%9a%84%e5%8c%ba%e5%88%ab