---
title: JDK1.8 线程池 源码解析
permalink: /posts/jdk1.8-executor.html
index_img: /img/jdk1.8-executor.png
lang: cn
mermaid: true
math: true
date: 2025-12-04 13:22:00
updated: 2025-12-04 13:22:00
excerpt: 以JDK1.8源码视角解析线程池的原理与设计哲学
categories:
- Java
tags:
- 源码解析
- 线程池
---



# UML



````mermaid
classDiagram

    class Executor {
        <<interface>>
        +execute()
    }
    class ExecutorService {
        <<interface>>
        +awaitTermination()
				+invokeAll()
				+invokeAny()
				+isShutdown()
				+isTerminated()
				+shutdown()
				+shutdownNow()
				+submit()
    }
    class AbstractExecutorService {
        <<abstract>>
        +...()
        +newTaskFor()
    }
    
    class ThreadPoolExecutor 


    Executor <|-- ExecutorService
    ExecutorService <|.. AbstractExecutorService
    AbstractExecutorService <|-- ThreadPoolExecutor

````



# 构造参数介绍

这里先介绍参数最全的构造参数

```java
public ThreadPoolExecutor(int corePoolSize,  
                          int maximumPoolSize,  
                          long keepAliveTime,  
                          TimeUnit unit,  
                          BlockingQueue<Runnable> workQueue,  
                          ThreadFactory threadFactory,  
                          RejectedExecutionHandler handler) {}
```

| 核心参数                            | 说明                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| `int corePoolSize`                  | 核心线程数<br />常驻线程数，空闲也不会回收<br>除非设置 `allowCoreThreadTimeOut = true` |
| `int maximumPoolSize`               | 最大线程数<br>当任务队列满时，会创建新线程直至此上线         |
| `long keepAliveTime`                | 非核心线程允许空闲的时间，超过则回收                         |
| `TimeUnit unit`                     | `keepAliveTime`的时间单位                                    |
| `BlockingQueue<Runnable> workQueue` | 任务队列，保存等待执行任务的阻塞队列<br>核心线程满之后的，会先加入任务队列 |
| `ThreadFactory threadFactory`       | 线程工厂<br>用于设置线程名称、优先级、守护状态、创建方式等   |
| `RejectedExecutionHandler handler`  | 拒绝策略<br>当 任务队列已满 且 达到最大线程数 触发           |

参数最少的构造器如下：

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```

这里有2个默认值

- 默认线程工厂 `Executors.defaultThreadFactory()`
- 默认拒绝策略 `defaultHandler = new AbortPolicy()`



# 任务提交流程

`ThreadPoolExecutor#execute` 方法整体流程

1. 判断 当前线程数 workers.size <  corePoolSize ？
   1. 为真，则创建核心线程并执行任务
   2. 为假，判断任务队列 workQueue 是否已满？
      1. 未满，则加入 workQueue
      2. 已满，再判断 当前线程数 workers.size < maximumPoolSize？
         1. 为真，则创建临时线程线程执行任务
         2. 为假，则触发拒绝策略 handler



# 其他重要参数

## 线程池运行状态与有效线程数

线程池的运行状态 runState(rs) 与 有效线程数 workerCount(wc)

都是打包在一个32位原子整数 ctl 中控制实现的，就是用一个数字能同时表示2个含义

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

其中**高3位**用来表示运行状态rs，**低29位**表示有效线程数wc



线程池的运行状态有5个，都是32位的整数，其中`COUNT_BITS=29`

```java
private static final int COUNT_BITS = Integer.SIZE - 3;

private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;
```

下面详细解释下各状态的含义以及二进制表示，其中省略位都是0

| 状态         | 二进制      | 意义                                                         |
| ------------ | ----------- | ------------------------------------------------------------ |
| `RUNNING`    | 1100...0000 | 接受**新execute**的任务；执行**已入队**的任务                |
| `SHUTDOWN`   | 0000...0000 | 不接受**新execute**的任务；但执行**已入队**的任务；中断所有空闲的线程 |
| `STOP`       | 0010...0000 | 不接受**新execute**的任务；不执行**已入队**的任务；中断所有的线程 |
| `TIDYING`    | 0100...0000 | 所有线程停止；`workerCount`数量为0；将执行钩子方法 `terminated()` |
| `TERMINATED` | 0110...0000 | `terminated()`方法执行完毕                                   |

观察可知，这5个状态都是只有高3位不同，低29位都是0



这样在使用初始化方法`ctlOf()`，传入目标状态与有效线程数，在按位或运算下，其效果就是

- 不管 wc 高3位如何，结果的高3位就取决于 rs 的高3位
- 因为 rs 低29位都是0，所以结果的低29位取决于 wc 的低29位

```java
private static int ctlOf(int rs, int wc) { return rs | wc; }
```



线程池也提供了2个打包方法，用来分别提取 `ctl` 中运行状态与有效线程数的方法

```java
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

其中 `CAPACITY`就是$2^29$ = 0001...1111 (二进制下低位29个1)正是当做`ctl`中的区分位，具体作用如下

| 方法            | 方法体          | 带入CAPACITY的值                       | 作用            |
| --------------- | --------------- | -------------------------------------- | --------------- |
| `runStateOf`    | `c & ~CAPACITY` | `c & 11100000000000000000000000000000` | 提取`ctl`高3位  |
| `workerCountOf` | `c & CAPACITY`  | `c & 00011111111111111111111111111111` | 提取`ctl`低29位 |

就这样线程池将运行状态与有效线程数量2个属性，打包在一个32位的整数中

并提供了初始化与分别解包提取的方法



## 线程池中的锁

```java
private final ReentrantLock mainLock = new ReentrantLock();
```

这把可重入锁在很多地方都会使用到。

比如对线程集合 `workers` 的访问与操作、读取曾最大线程数、读取已完成任务数等。



## 线程集合 

```java
private final HashSet<Worker> workers = new HashSet<Worker>();
```

用来保存当前线程池中的所有线程;

可通过该集合对线程池中的线程进行**中断**, **遍历**等;

创建新的线程时, 要添加到该集合, 移除线程, 也要从该集合中移除对应的线程;

对该集合操作都需要`mainLock`锁.



## 2个可监控的数量

```java
// 线程池中曾经达到的最大线程数
private int largestPoolSize;
// 线程池中已完成的任务数
private long completedTaskCount;
```

都是通过 `mainLock` 来获取保证数据的原子性和可见性



# 源码解析

## execute(): 将任务提交到线程池

```java
    public void execute1(Runnable command) {
        // 任务非空检查
        if (command == null)
            throw new NullPointerException();

        // 获得ctl的int值
        int c = ctl.get();

        // 情况1：当前线程数 < 核心线程数
        if (workerCountOf(c) < corePoolSize) {
            // 尝试添加一个核心类型的新worker, 作为核心线程池的线程，然后使用核心线程执行任务
            if (addWorker(command, true))
                // 添加 worker execute 方法退出
                return;

            // 添加worker作为核心线程失败, 重新获取ctl的int值
            c = ctl.get();
        }

        //  情况2：任务可以添加至工作队列
        if (isRunning(c) && workQueue.offer(command)) {
            // double-check, 再次获取ctl的值
            int recheck = ctl.get();
            // 线程池不是RUNNING状态并且当前task从workerQueue被移除成功
            if (!isRunning(recheck) && remove(command))
                // 执行拒绝策略
                reject(command);
                // 线程池中的workerCount为0
            else if (workerCountOf(recheck) == 0)
                // 启动一个非核心线程, 由于这里的task参数为null, 该线程会从workerQueue拉去任务
                addWorker(null, false);
        }
      
        // 情况3：尝试使用非核心线程执行任务，否则触发拒绝策略
        else if (!addWorker(command, false))
            reject(command);
    }
```


## addWorker(): 创建启动线程并执行任务

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (; ; ) {
        int c = ctl.get();
        // 线程池运行状态
        int rs = runStateOf(c);

        // 必要时检查工作队列是否为空
        if (rs >= SHUTDOWN &&
                !(rs == SHUTDOWN &&
                        firstTask == null &&
                        !workQueue.isEmpty()))
            return false;

        for (; ; ) {
            // 检查当前有效线程数，以下3种情况任意一种则退出并返回 false
            // 1.大于等于 容量数是2的29次方
            // 2.添加的是核心线程，当前有效线程数量 大于等于 设定的核心线程数量
            // 3.添加的是空闲线程，当前有效线程数量 大于等于 设定的最大线程数量
            int wc = workerCountOf(c);
            if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
          
            // 以 CAS 的方式增加 wc 数量1，成功则跳出 retry 块
            if (compareAndIncrementWorkerCount(c))
                break retry;
          
           // CAS 失败，如果 rs 发生变化，则再次进入 retry 块
            c = ctl.get();  // Re-read ctl
            if (runStateOf(c) != rs)
                continue retry;
            // else CAS failed due to workerCount change; retry inner loop
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {
        // 将任务从 Runnable 包装成 Worker
        w = new Worker(firstTask);
        // 从 Worker 中获取任务线程对象 t
        final Thread t = w.thread;
        if (t != null) {
            // 获取 mainLock 并开锁
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {
                // 再次检查 rs
                int rs = runStateOf(ctl.get());
               
                // rs 是 RUNNING 状态 或者 在 SHUTDOWN 前提下任务为空
                if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                    
                    // 任务线程已被启动则抛异常
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                  
                    // 将当前任务添加到任务列表 workers 中
                    workers.add(w);
                    
                    // 检查更新最大线程数
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    // 更新 workder 添加标志位
                    workerAdded = true;
                }
            } finally {
                // 释放锁
                mainLock.unlock();
            }
            // workder 添加成功，则启动任务线程，并更新 workder 启动标志位
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        // 任务线程未启动，执行添加 workder 失败逻辑
        if (!workerStarted)
            addWorkerFailed(w);
    }
    // 返回 workder 启动标志位
    return workerStarted;
}
```



## Worker对象: 包装过的线程任务

### 什么是 Worker

上文提到线程集合的对象是`Worker`而不是`Thread`

而Worker更像是一个"打包好"的员工：

- 里面装着线程 Thread
- 本地要干的第一个任务
- 已完成的任务计数
- 再顺带自带一把小锁（AQS）

在 JDK1.8 里 Worker 是 ThreadPoolExecutor 的内部类，大概声明如下：

```java
// 伪代码，保留结构
private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable {

    final Thread  thread;        // 真正干活的线程
    Runnable      firstTask;     // 刚创建时要跑的第一个任务
    volatile long completedTasks;// 这个 worker 累计完成的任务数

    // ...
}
```

这里涉及几个关键的信息：

1. Worker 不是单纯的 Thread，而是 线程 +状态 + 锁 的组合体
2. 是 Runnable ，线程启动后会执行自身的 run() 方法
3. 继承了 AQS 实现了一个简易的 0/1互斥锁，用来判断当前线程是否在正在执行任务



### 构造方法

```java
Worker(Runnable firstTask) {
    setState(-1); // inhibit interrupts until runWorker
    this.firstTask = firstTask;
    this.thread = getThreadFactory().newThread(this);
}
```

说明点：

1. setState(-1): 将AQS的state设置成-1，这是一个特殊值避免真正runWorker前被中断或者回收
2. firstTask: 就是在调用 execute() 时，新建 Worker 时新建的 task
3. Thread = new thread(this)：线程启动后会执行 Worker.run() 而不是直接提交的任务



### 使用AQS实现的小锁

Worker 继承了 AQS，但只用到了**独占模式**，而且也很简单：就是一个 0/1 状态的非重入锁。

```java
// Lock methods
//
// The value 0 represents the unlocked state.
// The value 1 represents the locked state.

// 当前线程是否持持有 Worker 锁 
// 0：没有线程获取该锁，可以尝试加锁
// 1：已经有线程获取了该锁，此时
protected boolean isHeldExclusively() {
    return getState() != 0;
}

// 尝试获取锁：state 从 0 -> 1，成功后将当前持有锁的线程设置为当前线程
// 形参 unused 没有用到
protected boolean tryAcquire(int unused) {
    if (compareAndSetState(0, 1)) {
        setExclusiveOwnerThread(Thread.currentThread());
        return true;
    }
    return false;
}

// 释放锁：state 置回 0，并且将当前持有锁的线程设置为 null
// 形参 unused 没有用到
protected boolean tryRelease(int unused) {
    setExclusiveOwnerThread(null);
    setState(0);
    return true;
}

// 对外暴露的几个封装方法
public void lock() { acquire(1);}
public boolean tryLock() { return tryAcquire(1);}
public void unlock() { release(1);}
public boolean isLocked() { return isHeldExclusively();}
```

可以直接把它理解为：**Worker 自己身上有一把“我现在是否在执行任务”的互斥锁。**

后面线程池很多逻辑（比如 interruptIdleWorkers）都要依赖这把锁的状态——成功 tryLock() 说明当前 worker 是空闲的。



### run(): 真正的线程任务runWorker()

Worker 作为 Runnable，它的 run() 非常简单：

```java
public void run() {
    runWorker(this);
}
```

也就是说：

- **真正的任务循环逻辑在** **ThreadPoolExecutor.runWorker(Worker w)** **里**
- Worker 只是把自己（包含 thread / firstTask / 锁）交给 runWorker()



这在设计上有两个好处：

1. Worker 只负责“**绑定线程和状态**”
2. 核心调度逻辑统一放在 ThreadPoolExecutor 里，便于管理、扩展和覆写钩子方法（beforeExecute / afterExecute）



###  runWorker(): Worker 主要工作流程

执行提交的task或死循环从`BlockingQueue`获取task.

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    // 关键：把 state 从 -1 解锁成 0，此后可以被中断 / 回收
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        // 当传入的task不为null, 或者task为null但是从BlockingQueue中获取的task不为null
        while (task != null || (task = getTask()) != null) {
            // 标记：从这里开始，worker 处于“忙碌”状态
            w.lock();
            // 线程池状态如果为STOP, 或者当前线程是被中断并且线程池是STOP状态, 或者当前线程不是被中断;
            // 则调用interrupt方法中断当前线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                            runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                wt.interrupt();
            try {
                // 钩子
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 真正执行线程任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x;
                    throw x;
                } catch (Error x) {
                    thrown = x;
                    throw x;
                } catch (Throwable x) {
                    thrown = x;
                    throw new Error(x);
                } finally {
                    // 钩子
                    afterExecute(task, thrown);
                }
            } finally {
                // task赋值为null, 下次从BlockingQueue中获取task
                task = null;
                w.completedTasks++;
                // 标记：本次任务结束，worker 变“空闲”
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```



### getTask():  从任务队列中获取task

getTask() 负责从队列里取任务并控制 worker 的回收。

runWorker 里如果 getTask() 返回了 null，就跳出 while 循环，最终由 processWorkerExit 把这个 Worker 清理掉。

```java
private Runnable getTask() {
    // BlockingQueue 的 poll方法是否已经超时
    boolean timedOut = false; // Did the last poll() time out?

    for (; ; ) {
        int c = ctl.get();
        int rs = runStateOf(c);

        // 如果线程池状态>=SHUTDOWN,并且BlockingQueue为null;
        // 或者线程池状态>=STOP
        // 以上两种情况都减少工作线程的数量, 返回的task为null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        // 当前线程是否需要被淘汰
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
            // BlockingQueue 的 poll 方法超时会直接返回 null
            // BlockingQueue 的 take 方法, 如果队列中没有元素, 当前线程会wait, 直到其他线程提交任务入队唤醒当前线程.
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
