---
layout: post
title: ThreadPoolExecutor与BlockQueue源码解读
date: 2020-05-29
Author: baofeidyz
categories: Java
tags: [Java]
comments: true

---


## JDK版本

此文章基于jdk 1.8.0_191

## 需要用到的知识点

### 位运算








| 操作符 | 描述                                                         | 例子                           |
| :----- | :----------------------------------------------------------- | :----------------------------- |
| ＆     | 如果相对应位都是1，则结果为1，否则为0                        | （A＆B），得到12，即0000 1100  |
| \|     | 如果相对应位都是0，则结果为0，否则为1                        | （A \| B）得到61，即 0011 1101 |
| ^      | 如果相对应位值相同，则结果为0，否则为1                       | （A ^ B）得到49，即 0011 0001  |
| 〜     | 按位取反运算符翻转操作数的每一位，即0变成1，1变成0。         | （〜A）得到-61，即1100 0011    |
| <<     | 按位左移运算符。左操作数按位左移右操作数指定的位数。         | A << 2得到240，即 1111 0000    |
| `>>`   | 按位右移运算符。左操作数按位右移右操作数指定的位数。         | A >> 2得到15即 1111            |
| `>>>`  | 按位右移补零操作符。左操作数的值按右操作数指定的位数右移，移动得到的空位以零填充。 | A>>>2得到15即0000 1111         |


## 线程池原理


> 摘自 汪文君. Java高并发编程详解：多线程与架构设计 (Java核心技术系列) (Kindle 位置 2508-2512). 北京华章图文信息有限公司. Kindle 版本. 

所谓线程池通俗的理解就是有一个池子，里面存放着已经创建好的线程，当有任务提交给线程池执行时，池子中的 某个线程会主动执行该任务。如果池子中的线程数量不够应付数量众多的任务时，则需要自动扩充新的线程到池子 中，但是该数量是有限的，就好比池塘的水界线一样。当任务比较少的时候，池子中的线程能够自动回收，释放 资源。为了能够异步地提交任务和缓存未被处理的任务，需要有一个任务队列。

![线程池原理图](https://raw.githubusercontent.com/baofeidyz/images/master/img/20190721102700.png)

一个完整的线程池应该具备如下要素：

1. 任务队列：用于缓存提交的任务
2. 线程数量管理功能：一个线程池必须能够很好地管理和控制线程数量，可通过如下三个参数来实现，比如创建 线程池时初始的线程数量 init；线程池自动扩充时最大的线程数量max；在线程池空闲时需要释放线程但是也要维护一定数量的活跃数量或者核心数量core。有了这三个参数，就能够很好地控制线程池中的线程数量，将其维护在一个合理的范围之内，三者之间的关系是 init<= core<= max
3. 任务拒绝策略：如果线程数量已达到上限且任务队列已满，则需要有相应的拒绝策略来通知任务提交者
4. 线程工厂：主要用于个性化定制线程，比如线程设置为守护线程以及设置线程名称等
5. QueueSize：任务队列主要存放提交的Runnable，但是为了防止内存溢出，需要有limit数量对其进行控制
6. Keepedalive 时间：该时间主要决定线程各个重要参数自动维护的时间间隔


## 线程池的五种状态

> 线程池状态示意图以及五种状态的说明摘自CSDN[一只逗比的程序猿](https://blog.csdn.net/L_kanglin/article/details/57411851)

一共有五种，分别是RUNNING、SHUTDOWN、STOP、TIDYING、TERMINATED

线程池状态切换示意图

![线程池状态切换示意图](https://raw.githubusercontent.com/baofeidyz/images/master/img/20190721160003.png)

### RUNNING

状态说明：线程池处在RUNNING状态时，能够接收新任务，以及对已添加的任务进行处理

状态切换：线程池的初始化状态是RUNNING。换句话说，线程池被一旦被创建，就处于RUNNING状态，并且线程池中的任务数为0

### SHUTDOWN

状态说明：线程池处在SHUTDOWN状态时，不接收新任务，但能处理已添加的任务

状态切换：调用线程池的shutdown()接口时，线程池由RUNNING -> SHUTDOWN

> 注：虽然状态已经不是RUNNING了，但是如果任务队列中还有任务的时候，线程池仍然会继续执行，具体分析请见ThreadPoolExecutor.execute()方法解析

### STOP

状态说明：线程池处在STOP状态时，不接收新任务，不处理已添加的任务，并且会中断正在处理的任务

状态切换：调用线程池的shutdownNow()接口时，线程池由(RUNNING or SHUTDOWN ) -> STOP

### TIDYING

状态说明：当所有的任务已终止，ctl记录的”任务数量”为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现

状态切换：当线程池在SHUTDOWN状态下，阻塞队列为空并且线程池中执行的任务也为空时，就会由 SHUTDOWN -> TIDYING。 
当线程池在STOP状态下，线程池中执行的任务为空时，就会由STOP -> TIDYING

### TERMINATED

状态说明：线程池彻底终止，就变成TERMINATED状态

状态切换：线程池处在TIDYING状态时，执行完terminated()之后，就会由 TIDYING -> TERMINATED

## 线程池五种状态的二进制表示

| 线程池状态 | 二进制 |
| ---------- | ------ |
| RUNNING    | 111    |
| SHUTDOWN   | 000    |
| STOP       | 001    |
| TIDYING    | 010    |
| TERMINATED | 011    |

```json
COUNT_BITS :29
RUNNING    :11100000 00000000 00000000 00000000
SHUTDOWN   :00000000 00000000 00000000 00000000
STOP       :00100000 00000000 00000000 00000000
TIDYING    :01000000 00000000 00000000 00000000
TERMINATED :01100000 00000000 00000000 00000000
RUNNING    :-536870912
SHUTDOWN   :0
STOP       :536870912
TIDYING    :1073741824
TERMINATED :1610612736
```

## ThreadPoolExecutor解读

### 构造函数解读

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.acc = System.getSecurityManager() == null ?
            null :
        AccessController.getContext();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
    }
```

先看参数

```java
int corePoolSize,
int maximumPoolSize,
long keepAliveTime,
TimeUnit unit,
BlockingQueue<Runnable> workQueue,
ThreadFactory threadFactory,
RejectedExecutionHandler handler
```

对应含义关系如下

| 参数名          | 类型                       | 备注                                                         |
| --------------- | -------------------------- | ------------------------------------------------------------ |
| corePoolSize    | `int`                      | 核心线程数（如果`allowCoreThreadTimeOut`为true，核心线程将一直存活） |
| maximumPoolSize | `int`                      | 允许创建的最大线程数<br>（如果使用了无界队列LinkedBlockingQueue，这个值会失效，原因在讲解execute方法中提及） |
| keepAliveTime   | `long`                     | 非核心线程闲置时的超时时长（如果`allowCoreThreadTimeOut`为true，这个时长也会用于核心线程） |
| unit            | `TimeUnit`                 | 参数`keepAliveTime`的单位                                    |
| workQueue       | `BlockingQueue<Runnable>`  | 任务队列，可选的[子类](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/BlockingQueue.html)有<br>[ArrayBlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ArrayBlockingQueue.html)<br>[DelayQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/DelayQueue.html)<br>[LinkedBlockingDeque](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedBlockingDeque.html)<br>[LinkedBlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedBlockingQueue.html)<br>[LinkedTransferQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedTransferQueue.html)<br>[PriorityBlockingQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/PriorityBlockingQueue.html)<br>[SynchronousQueue](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/SynchronousQueue.html) |
| threadFactory   | `ThreadFactory`            | 线程工厂，为线程池提供创建新线程的功能（其他构造函数中默认传`Executors.defaultThreadFactory()` |
| handler         | `RejectedExecutionHandler` | 拒绝策略，当队列和线程池都满了就才会根据这个策略进行处理。（默认为AbortPolicy，直接抛出异常），可选[子类](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RejectedExecutionHandler.html)有[ThreadPoolExecutor.AbortPolicy](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.AbortPolicy.html)<br>[ThreadPoolExecutor.CallerRunsPolicy](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.CallerRunsPolicy.html)<br>[ThreadPoolExecutor.DiscardOldestPolicy](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.DiscardOldestPolicy.html)<br>[ThreadPoolExecutor.DiscardPolicy](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.DiscardPolicy.html) |

### 执行方法解读

```java
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
             * Proceed in 3 steps:
             *
             * 1. If fewer than corePoolSize threads are running, try to
             * start a new thread with the given command as its first
             * task.  The call to addWorker atomically checks runState and
             * workerCount, and so prevents false alarms that would add
             * threads when it shouldn't, by returning false.
             *
             * 2. If a task can be successfully queued, then we still need
             * to double-check whether we should have added a thread
             * (because existing ones died since last checking) or that
             * the pool shut down since entry into this method. So we
             * recheck state and if necessary roll back the enqueuing if
             * stopped, or start a new thread if there are none.
             *
             * 3. If we cannot queue task, then we try to add a new
             * thread.  If it fails, we know we are shut down or saturated
             * and so reject the task.
             */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```

这里涉及到`runState`和`workerCount`两个概念，线程池中是利用32位的int变量来表示。

因为线程池的状态总共有五种，2^2 = 4, 2^3 = 8，所以需要占用三位，实际采用的就是**高三位**表示，具体可见[线程池五种状态的二进制表示](#线程池五种状态的二进制表示)

剩下的部分全部都用于记录有效线程数，所以代码中也就规定有效线程数不可大于29位，也就是最大为2^29-1，详见execute()方法解析。

然后我们再来一起看看`execute(Runnable command)`方法是如何运行的。

#### execute()整体逻辑概览

1. 最开始是一个基本的判空逻辑，如果传入的任务是空的则抛出异常
2. 第一个判断：检查当前核心线程数，如果当前线程数小于核心线程则调用`addWorker()`方法创建线程，创建成功返回true
3. 第二个判断：当核心线程池中的所有线程都在运行，此时将线程放到任务中
4. 第三个判断：如果核心线程数已经满了，队列也添加失败了，那么这里就会调用上面提到的拒绝策略，如果我们没有在创建线程池时给出特定的拒绝策略，那么默认的实现就是抛出异常。

```java
throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
```

#### 细节实现

#### 第一个判断：检查当前核心线程数

```java
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
```

##### 获取workerCount

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

这里的`ctl`变量是一个初始值为`RUNNING`的`AtomicInteger`对象，拿到的变量`c`中既存储了当前线程池的状态，又保存了当前线程池中的有效线程数量。

> 画外音：这里的AtomicInteger对象为什么是线程安全的，是因为使用了CAS，具体不谈

##### 判断当前有效线程数量是否大于核心线程数量

然后我们再看`workerCountOf(c)`方法的实现：

```java
    private static int workerCountOf(int c)  { 
        return c & CAPACITY; 
    }
```

`CAPACITY`的值为：

```java
00011111 11111111 11111111 11111111
```

结合[位运算](#位运算)法则，这里`c & CAPACITY`的结果集就是实际的有效线程数量。

##### 创建核心线程数量

当有效线程数量小于核心线程数量的时候，我们需要调用`addWorker(command, true)`的方法去创建线程，这里的参数`ture`就用于标识当前想要创建的线程是核心线程。

然后我们再来看看`addWorker(command, true)`方法的具体实现逻辑。

##### addWork实现逻辑

```java
    private boolean addWorker(Runnable firstTask, boolean core) {
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // Check if queue empty only if necessary.
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;

            for (;;) {
                int wc = workerCountOf(c);
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                if (compareAndIncrementWorkerCount(c))
                    break retry;
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
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                final ReentrantLock mainLock = this.mainLock;
                mainLock.lock();
                try {
                    // Recheck while holding lock.
                    // Back out on ThreadFactory failure or if
                    // shut down before lock acquired.
                    int rs = runStateOf(ctl.get());

                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                        workerAdded = true;
                    }
                } finally {
                    mainLock.unlock();
                }
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

###### `rs`为什么可以表示runstate？

```java
private static int runStateOf(int c)     { return c & ~CAPACITY; }
```

`~CAPACITY`的值为：

```java
11100000 00000000 00000000 00000000
```

所以后面的29位不论怎么样都会变成0，也就是最后的结果集中只会有高三位用于表示runstate的参数

###### 什么时候才会创建线程？

```java
            if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                   firstTask == null &&
                   ! workQueue.isEmpty()))
                return false;
```

哇~这个判断真的是够绕的，我看了老半天，一起来缕一缕

```java
rs >= SHUTDOW
```

代表当前runstate是SHUTDOWN  STOP TIDYING TERMINATED 任意一个，关于五种状态的值可见[线程池五种状态的二进制表示](#线程池五种状态的二进制表示)

```java
! (rs == SHUTDOWN && firstTask == null && ! workQueue.isEmpty())
```

A：在`rs >= SHUTDOW`成立的前提下，如果是`rs != SHUTDOWN`，则整个判断成立。

B：在`rs >= SHUTDOW`成立的前提下，如果是`firstTask != null`，则整个判断成立。

C：在`rs >= SHUTDOW`成立的前提下，如果是` workQueue.isEmpty()`，则整个判断成立。

并且A B C三个是有先后顺序的

再总结一下就是在第一个判断成立的前提下，第二个判断中，只要有一个不成立就会返回false，线程创建失败。

转成白话文就是

> A 当线程池状态为STOP TIDYING TERMINATED时，不会创建线程
>
> B 当线程池为SHUTDOWN时，不允许新建任务
>
> C 当线程池为SHUTDOWN时，且没有新的任务，此时如果任务队列也经没有任务，同样不会创建线程

此时再回头看看[线程池的五种状态](#线程池的五种状态)

---

接着往下看

```java
            for (;;) {
                // 第一步，拿到当前的有效线程数
                int wc = workerCountOf(c);
                // 如果当前有效线程数已经大于等于允许的最大线程数，不允许创建
                // 如果正准备创建的线程是core的线程且大于线程池初始化时设定的线程数，不允许创建
                // 如果正准备创建的线程不是core但大于线程池初始化时设定的最大线程数，不允许创建
                if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                    return false;
                // 利用CAS做自增操作，如果成功了，就跳出循环体开始下一步操作，如果失败则重试
                if (compareAndIncrementWorkerCount(c))
                    break retry;
                // 重新读取当前的runstate，如果当前的runstate和循环体中的runstate有变化，则重新去判断是否需要创建线程
                c = ctl.get();  // Re-read ctl
                if (runStateOf(c) != rs)
                    continue retry;
                // else CAS failed due to workerCount change; retry inner loop
            }
```

###### 线程到底是怎么创建的呢？

经历了层层校验逻辑，我们总算是要准备创建线程了。

```java
    // 标记线程是否启动
	boolean workerStarted = false;
	// 标记线程是否被添加
    boolean workerAdded = false;
    Worker w = null;
    try {
        w = new Worker(firstTask);
        final Thread t = w.thread;
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            // 获取锁
            mainLock.lock();
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get());
				// 再检查一下当前线程池的状态
                // rs < SHUTDOWN 表示线程池的状态为RUNNING就直接创建线程
                // rs == SHUTDOWN && firstTask == null 当前线程已经处于SHUTDOWN且当前没有新建的任务（也就是为了把任务队列执行完）
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    // 如果正准备创建的线程已经处于alive状态，则抛出异常
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();
                    // 否则放到workers中（包含线程池中所有的工作线程，将新构造的工作线程加入到工作线程集合中）
                    workers.add(w);
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)
            addWorkerFailed(w);
    }
```
##### 总结

###### 创建线程的条件：

> 当前线程池状态为RUNNING，当前线程数小于核心线程数或当前线程线程数小于最大线程数。其中最大线程数又分为创建线程池时给定的数量以及程序允许的最大值。

###### 不创建线程的条件：

> 当线程池状态不为RUNNING时，不会接受新的任务，此时如果任务队列还有任务，会把这部分处理完。

#### 第二个判断：当核心线程池中的所有线程都在运行，此时将线程放到任务中

```java
    // 检查当前线程池状态，如果为RUNNING状态，则尝试将任务放到任务队列中	
	if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        // 放置成功以后，再次检查当前的线程池状态，如果当前线程池状态非RUNNING，则尝试将刚刚放入的任务从任务队列中移除
        if (! isRunning(recheck) && remove(command))
            reject(command);
        // 如果当前线程数状态为RUNNING，但是workerCount的值又等于0则传入空任务结束此次创建
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
```



#### 第三个判断：什么时候执行拒绝策略？

```java
    else if (!addWorker(command, false))
        reject(command);
```

当第二个判断不成立，也就是当前线程池状态非RUNNING状态或尝试将任务放到任务队列中失败时，尝试再次创建一个非核心线程，此时线程数需要小于创建线程池时给定的最大值。

> 总结
>
> 结合第二个判断和第三个判断，就可以明白为什么当任务队列是无界的时候，最大线程数不会产生作用了。

### 工作线程的执行

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread(); // 得到当前线程
    Runnable task = w.firstTask; // 得到Worker中的任务task，也就是用户传入的task
    w.firstTask = null; // 将Worker中的任务置空
    w.unlock(); // allow interrupts。 
    boolean completedAbruptly = true; // 标识当前Worker异常结束，默认是异常结束
    try {
        // 如果worker中的任务不为空，执行执行任务
        // 否则使用getTask获得任务。一直循环，除非得到的任务为空才退出
        while (task != null || (task = getTask()) != null) {
            // 如果拿到了任务，给自己上锁，表示当前Worker已经要开始执行任务了，
            // 已经不是处于闲置Worker(闲置Worker的解释请看下面的线程池关闭)
            w.lock();  
            // 在执行任务之前先做一些处理。 
            // 1. 如果线程池已经处于STOP状态并且当前线程没有被中断，中断线程 
            // 2. 如果线程池还处于RUNNING或SHUTDOWN状态，并且当前线程已经被中断了，
            // 重新检查一下线程池状态，如果处于STOP状态并且没有被中断，那么中断线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() &&
                  runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                // 任务执行前需要做什么，ThreadPoolExecutor是个空实现，子类可以自行扩展
                beforeExecute(wt, task); 
                Throwable thrown = null;
                try {
                    // 真正的开始执行任务，这里run的时候可能会被中断，比如线程池调用了shutdownNow方法
                    task.run(); 
                } catch (RuntimeException x) { // 任务执行发生的异常全部抛出，不在runWorker中处理
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    // 任务执行结束需要做什么，ThreadPoolExecutor是个空实现，子类可以自行扩展
                    afterExecute(task, thrown); 
                }
            } finally {
                task = null;
                w.completedTasks++; // 记录执行任务的个数
                w.unlock(); // 执行完任务之后，解锁，Worker变成闲置Worker，等待执行下一个任务
            }
        }
        completedAbruptly = false; // 正常结束
    } finally {
        processWorkerExit(w, completedAbruptly); // Worker退出时执行
    }
}
```

## BlockQueue

![](https://raw.githubusercontent.com/baofeidyz/images/master/img/20190721160225.png)

三个添加元素的方法：

- add:把e加到队列里,添加成功返回true，容量如果满了添加失败会抛出IllegalStateException异常
- offer:表示如果可能的话,将e加到队列里，成功返回true,否则返回false
- put:把e加到BlockingQueue里,如果BlockQueue没有空间,则调用此方法的线程被阻断直到BlockingQueue里面有空间再继续.

三个删除元素的方法：

- poll:取走队列头部的对象,若不能立即取出,则可以等待timeout参数规定的时间,取不到时返回null
- remove:基于对象找到对应的元素并删除，删除成功返回true，否则返回false
- take:取走队列中排在首位的对象,若队列为空,一直阻塞到队列有元素并删除

> 其中：
>
> 队列不接受null 元素。试图add、put 或offer 一个null 元素时，某些实现会抛出NullPointerException。
>
> null 被用作指示poll 操作失败的警戒值。 

|          | 抛出异常             | 特殊值     | 阻塞     | 超时                   |
| :------- | :------------------- | :--------- | :------- | ---------------------- |
| **插入** | `add(e)`             | `offer(e)` | `put(e)` | `offer(e, time, unit)` |
| **移除** | `remove（Object o）` | `poll()`   | `take()` | `poll(time, unit)`     |
| **检查** | `element()`          | `peek()`   | *不可用* | *不可用*               |

| 类型     | 含义                                                         |
| -------- | ------------------------------------------------------------ |
| 抛出异常 | 如果试图的操作无法立即执行，抛一个异常                       |
| 特殊值   | 如果试图的操作无法立即执行，返回一个特定的值(常常是 true / false) |
| 阻塞     | 如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行 |
| 超时     | 如果试图的操作无法立即执行，该方法调用将会发生阻塞，直到能够执行，<br>但等待时间不会超过给定值。返回一个特定值以告知该操作是否成功(典型的是 true / false) |

常见的四个实现：

### ArrayBlockingQueue

![](https://raw.githubusercontent.com/baofeidyz/images/master/img/20190721160411.png)

一个由数组支持的有界阻塞队列。此队列按 **FIFO（先进先出）**原则对元素进行排序。队列的头部是在队列中存在时间最长的元素。队列的尾部 是在队列中存在时间最短的元素。新元素插入到队列的尾部，队列获取操作则是从队列头部开始获得元素。

ArrayBlockingQueue的原理就是使用一个可重入锁和这个锁生成的两个条件对象进行并发控制(classic two-condition algorithm)

ArrayBlockingQueue是一个带有长度的阻塞队列，初始化的时候必须要指定队列长度，且指定长度之后不允许进行修改

#### 属性

```java
  // 存储队列元素的数组，是个循环数组
  final Object[] items;

  // 拿数据的索引，用于take，poll，peek，remove方法
  int takeIndex;

  // 放数据的索引，用于put，offer，add方法
  int putIndex;

  // 元素个数
  int count;

  // 可重入锁
  final ReentrantLock lock;
  // notEmpty条件对象，由lock创建
  private final Condition notEmpty;
  // notFull条件对象，由lock创建
  private final Condition notFull;
```

#### 创建

```java
	// 构造函数要求指定队列大小capacity
	/**
	* capacity和fair，capacity同第一个构造方法，代表队列大小。fair代表该队列的访问策略是否公平。如果为 		
	* true，则按照 FIFO 顺序访问插入或移除时受阻塞线程的队列；如果为 false，则访问顺序是不确定的。这里fair参
	* 数被设置为ReentrantLock的入参，就可以通过ReentrantLock来保证线程访问是否公平。而此构造方法创建了两个
	* Condition，也就是条件，分别是notEmpty和notFull，Condition可以调用wait()和signal()来控制当前现
	* 成等待或者唤醒
	*/
	// 默认fair为false
	public ArrayBlockingQueue(int capacity, boolean fair,
                              Collection<? extends E> c) {
        this(capacity, fair);
        final ReentrantLock lock = this.lock;
        lock.lock(); // Lock only for visibility, not mutual exclusion
        try {
            int i = 0;
            try {
                for (E e : c) {
                    checkNotNull(e);
                    items[i++] = e;
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                throw new IllegalArgumentException();
            }
            count = i;
            putIndex = (i == capacity) ? 0 : i;
        } finally {
            lock.unlock();
        }
    }
```

内部调用的`this(capacity, fair)`方法

```java
    public ArrayBlockingQueue(int capacity, boolean fair) {
        if (capacity <= 0)
            throw new IllegalArgumentException();
        this.items = new Object[capacity];
        lock = new ReentrantLock(fair);
        notEmpty = lock.newCondition();
        notFull =  lock.newCondition();
    }
```

#### 数据的添加

##### add

ArrayBlockingQueue自己并没有实现add方法，而直接调用父类AbstractQueue的add方法

```java
  public boolean add(E e) {
    return super.add(e);
  }
```

内部调用的super.add

```java
    public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw  IllegalStateException("Queue full");
    }
```

实际上最后调用的就是ArrayBlockingQueue自己的offer方法，但是如果offer方法返回结果为false，则抛出IllegalStateException

##### offer

```java
	public boolean offer(E e) {
    		// 不允许元素为空，为空会抛出NullPointerException异常
        checkNotNull(e);
    		// 获取锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            // 元素个数和当前存储队列元素的数组大小相等，就不会再加了，所以会返回false
            if (count == items.length)
                return false;
            else {
              	// 入队操作
                enqueue(e);
                return true;
            }
        } finally {
          	// 释放锁
            lock.unlock();
        }
    }
```

内部调用的enqueue方法

```java
	private void enqueue(E x) {
        // assert lock.getHoldCount() == 1;
        // assert items[putIndex] == null;
        final Object[] items = this.items;
        items[putIndex] = x;
    		// 放数据索引+1，如果数据索引已经与存储队列的元素数量相同则变为0
        if (++putIndex == items.length)
            putIndex = 0;
    		// 元素个数+1
        count++;
    		// 使用条件对象notEmpty通知，比如使用take方法的时，队列中没有数据被阻塞。这个时候队列中新增了一条数据，需要调用signal通知
        notEmpty.signal();
    }
```

##### put

```java
	public void put(E e) throws InterruptedException {
    		// 元素为空抛出NPE异常
        checkNotNull(e);
        final ReentrantLock lock = this.lock;
    		// 加锁，保证调用put方法的时候只有一个线程
        lock.lockInterruptibly();
        try {
          	// 如果队列满了，阻塞当前线程并加入到条件对象notFull的等待队列里
            while (count == items.length)
              	// 线程阻塞并被挂起，同时释放锁
                notFull.await();
          	// 入队
            enqueue(e);
        } finally {
            lock.unlock();
        }
    }
```

疑问点，之前的入队操作的写法是

```java
private void insert(E x) {
    items[putIndex] = x;
    putIndex = inc(putIndex);
    ++count;
    notEmpty.signal(); 
}
```

为什么改了？改之前和改之后有什么区别？

#### 总结

> ArrayBlockingQueue总共有三个添加数据的方法，分别是add、put、offer
>
> - add方法：内部实际调用的是offer方法，如果队列已满则抛出IllegalStateException一场，否则返回true
>
> - offer方法：如果队列满了返回false，成功返回true
>
> - put方法：如果队列满了会阻塞线程，直到有线程消费了队列中的元素
>
> 三个方法内部都使用可重入锁保证原子性

#### 数据的删除

##### poll

```java
    public E poll() {
      	// 加锁，保证调用poll方法的时候只有一个线程
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
          	// 如果count=0，也就是队列中没有元素了，这时候会返回null，否则调用dequeue取元素
            return (count == 0) ? null : dequeue();
        } finally {
            lock.unlock();
        }
    }
```

内部调用的dequeue方法

```java
    private E dequeue() {
        // assert lock.getHoldCount() == 1;
        // assert items[takeIndex] != null;
        final Object[] items = this.items;
        @SuppressWarnings("unchecked")
      	// takeIndex是用于拿数据的索引
        E x = (E) items[takeIndex];
      	// 取出数据以后，这个元素位置为null
        items[takeIndex] = null;
      	// 如果当前拿数据的索引大小已经和元素数组大小相等则置为0，这里的takeIndex之所以自增是因为FIFO原则
        if (++takeIndex == items.length)
            takeIndex = 0;
      	// 一个数据被取出，此时元素个数-1
        count--;
      	// TODO
        if (itrs != null)
            itrs.elementDequeued();
      	// 使用对象notFull通知，如使用put方法放置数据的时候队列满了，被阻塞，这个时候dequeue取出一条数据，队列没满，则可以继续放入
        notFull.signal();
        return x;
    }
```

##### take

```java
    public E take() throws InterruptedException {
        final ReentrantLock lock = this.lock;
        lock.lockInterruptibly();
        try {
          	// 如果队列为空，阻塞当前线程，并加入到条件对象notEmpty的等待队列里
            while (count == 0)
              	// 线程阻塞并被挂起，同时释放锁
                notEmpty.await();
            return dequeue();
        } finally {
            lock.unlock();
        }
    }
```

##### remove

```java
    public boolean remove(Object o) {
        if (o == null) return false;
        final Object[] items = this.items;
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            if (count > 0) {
              	// 放数据索引
                final int putIndex = this.putIndex;
              	// 取数据索引
                int i = takeIndex;
              	// 通过元素类型遍历,找到类型相同的索引位置，i<=元素总大小
                do {
                    if (o.equals(items[i])) {
                        removeAt(i);
                        return true;
                    }
                    if (++i == items.length)
                        i = 0;
                	// 因为putIndex就是最后一个放数据的位置，所以拿数据的索引不能等于它
                  // 但是有没有想过为什么不可以写<=呢？
                } while (i != putIndex);
            }
            return false;
        } finally {
            lock.unlock();
        }
    }
```

内部调用的removeAt方法

```java
    void removeAt(final int removeIndex) {
        // assert lock.getHoldCount() == 1;
        // assert items[removeIndex] != null;
        // assert removeIndex >= 0 && removeIndex < items.length;
        final Object[] items = this.items;
      	// 如果要删除数据的索引位置就是拿数据索引位置，直接将takeIndex索引位置上的数据，然后takeIndex+1
        if (removeIndex == takeIndex) {
            // removing front item; just advance
            items[takeIndex] = null;
            if (++takeIndex == items.length)
                takeIndex = 0;
            count--;
            if (itrs != null)
                itrs.elementDequeued();
        } else {
            // an "interior" remove

            // slide over all others up through putIndex.
          	// 如果要删除数据的索引位置不是takeIndex，则需要移动元素位置，更新putIndex
            final int putIndex = this.putIndex;
            for (int i = removeIndex;;) {
                int next = i + 1;
                if (next == items.length)
                    next = 0;
                if (next != putIndex) {
                    items[i] = items[next];
                    i = next;
                } else {
                    items[i] = null;
                    this.putIndex = i;
                    break;
                }
            }
            count--;
            if (itrs != null)
                itrs.removedAt(removeIndex);
        }
      	// 删除以后通知阻塞线程，比如put方法
        notFull.signal();
    }
```

#### 总结

> 三个删除方法，分别是poll，take，remove
>
> - poll方法对于队列为空的情况，返回null，否则返回队列头部元素
> - remove方法取的元素是基于对象的下标值，删除成功返回true，否则返回false
> - poll方法和remove方法不会阻塞线程
> - take方法对于队列为空的情况，会阻塞并挂起当前线程，直到有数据加入到队列中
> - 三个删除方法内部都会调用notFull.signal方法通知正在等待队列满情况下的阻塞线程

### LinkedBlockingQueue

![](https://raw.githubusercontent.com/baofeidyz/images/master/img/20190721160443.png)

内部以一个链式结构(链接节点)对其元素进行存储，链表是单向链表，满足FIFO(先进先出)原则。

#### 属性

```java
    /** The capacity bound, or Integer.MAX_VALUE if none */
		// 容量大小
    private final int capacity;

    /** Current number of elements */
		// 元素个数
    private final AtomicInteger count = new AtomicInteger();

    /**
     * Head of linked list.
     * Invariant: head.item == null
     */
		// 头节点
    transient Node<E> head;

    /**
     * Tail of linked list.
     * Invariant: last.next == null
     */
		// 尾节点
    private transient Node<E> last;

    /** Lock held by take, poll, etc */
		// 读锁
    private final ReentrantLock takeLock = new ReentrantLock();

    /** Wait queue for waiting takes */
		// 读锁的条件对象
    private final Condition notEmpty = takeLock.newCondition();

    /** Lock held by put, offer, etc */
		// 写锁
    private final ReentrantLock putLock = new ReentrantLock();

    /** Wait queue for waiting puts */
		// 写锁的条件对象
    private final Condition notFull = putLock.newCondition();
```

ArrayBlockingQueue只有1个锁，添加数据和删除数据的时候只有1个可以被执行，不允许并行操作

但LinkedBlockingQueue有2个锁，读和写各有一把，添加数据和删除数据两个操作可以并行。

#### 创建

```java
    public LinkedBlockingQueue(Collection<? extends E> c) {
        this(Integer.MAX_VALUE);
        final ReentrantLock putLock = this.putLock;
        putLock.lock(); // Never contended, but necessary for visibility
        try {
            int n = 0;
            for (E e : c) {
                if (e == null)
                    throw new NullPointerException();
                if (n == capacity)
                    throw new IllegalStateException("Queue full");
                enqueue(new Node<E>(e));
                ++n;
            }
            count.set(n);
        } finally {
            putLock.unlock();
        }
    }
```

内部调用的this和enqueue方法

```java
    public LinkedBlockingQueue(int capacity) {
        if (capacity <= 0) throw new IllegalArgumentException();
        this.capacity = capacity;
      	// last和head节点都是null
        last = head = new Node<E>(null);
    }
```

```java
    private void enqueue(Node<E> node) {
        // assert putLock.isHeldByCurrentThread();
        // assert last.next == null;
        last = last.next = node;
    }
    static class Node<E> {
        E item;

        /**
         * One of:
         * - the real successor Node
         * - this Node, meaning the successor is head.next
         * - null, meaning there is no successor (this is the last node)
         */
        Node<E> next;

        Node(E x) { item = x; }
    }
```

enqueue的class方法

```java
	private void enqueue(Node<E> paramNode){
      this.last = (this.last.next = paramNode);
    }
```

另外还画了一个图

![](http://qiniu.losergzr.cn/picgo/20190401180533.png)

#### 数据的添加

三个方法，分别是add offer put

##### add

LinkedBlockingQueue和ArrayBlockingQueue一样，同样没有实现add方法，所以会直接调用父类AbstractQueue的add方法

```java
		public boolean add(E e) {
        if (offer(e))
            return true;
        else
            throw new IllegalStateException("Queue full");
    }
```

实际上也就是调用LinkedBlockingQueue的offer方法，失败则抛出异常

##### offer

> 关于这里使用到的锁实际上涉及到java的monitor MESA模型，这里就不展开讲了，有兴趣的小伙伴戳[极客时间-管程：并发编程的万能钥匙](https://time.geekbang.org/column/article/7db3be62eae600db0d947f428185c06a/share?code=4e5ChyziCCisKCETno4QJHdW0BXcHigtqXBCNDlJuIE%3D&from=singlemessage&isappinstalled=0&oss_token=)，有二十个小伙伴可以免费读哦～打不开的话复制地址用微信打开哦😊
>
> 可重入锁指的是线程可以重复获取同一把锁，ReentrantLock有一个带布尔值fair的构造函数，true表示公平锁，反之则是非公平锁。指的是条件变量的等待队列唤醒策略，公平锁是唤醒等待时间最长的，非公平锁则不会保证

```java
    public boolean offer(E e) {
      	// 不允许空元素
        if (e == null) throw new NullPointerException();
        final AtomicInteger count = this.count;
      	// 如果满了就返回false，but这个大小可是2^31-1=2147483647
        if (count.get() == capacity)
            return false;
        int c = -1;
        Node<E> node = new Node<E>(e);
      	// 拿到写锁
        final ReentrantLock putLock = this.putLock;
      	// 对写操作加锁
        putLock.lock();
        try {
            if (count.get() < capacity) {
                enqueue(node);
              	// 元素个数+1
                c = count.getAndIncrement();
                if (c + 1 < capacity)
                  	// 如果容量还没满，在对象notFull唤醒正在等待的线程，表示可以再往队列里加数据了
                    notFull.signal();
            }
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
      // todo
        return c >= 0;
    }
```

##### put

```java
    public void put(E e) throws InterruptedException {
        if (e == null) throw new NullPointerException();
        // Note: convention in all put/take/etc is to preset local var
        // holding count negative to indicate failure unless set.
        int c = -1;
        Node<E> node = new Node<E>(e);
      	// 拿到写锁
        final ReentrantLock putLock = this.putLock;
        final AtomicInteger count = this.count;
      	// 对写操作加锁
        putLock.lockInterruptibly();
        try {
            /*
             * Note that count is used in wait guard even though it is
             * not protected by lock. This works because count can
             * only decrease at this point (all other puts are shut
             * out by lock), and we (or some other waiting put) are
             * signalled if it ever changes from capacity. Similarly
             * for all other uses of count in other wait guards.
             */
          	// 如果当前容量已经满了，则阻塞并挂起当前线程
            while (count.get() == capacity) {
                notFull.await();
            }
          	// 入队操作
            enqueue(node);
          	// 元素+1
            c = count.getAndIncrement();
            if (c + 1 < capacity)
              	// 如果容量还没满，在放锁的条件对象notFull唤醒正在等待的线程
                notFull.signal();
        } finally {
            putLock.unlock();
        }
        if (c == 0)
            signalNotEmpty();
    }
```

#### 总结

> add offer put方法与ArrayBlockingQueue特性一致，只是底层实现不同
>
> - add方法：内部实际调用的是offer方法，如果队列已满则抛出IllegalStateException一场，否则返回true
>
> - offer方法：如果队列已满则返回false，成功返回true
>
> - put方法：如果队列满了会阻塞线程，直到有线程消费了队列中的元素
>
> ArrayBlockingQueue中放入数据阻塞的时候，需要消费数据才能唤醒，而LinkedBlockingQueue中放入数据阻塞的时候，因为它内部有2个锁，可以并行执行放入数据和消费数据，不仅在消费数据的时候进行唤醒插入阻塞的线程，同时在插入的时候如果容量还没满，也会唤醒插入阻塞的线程

#### 数据的删除

##### poll

```java
    public E poll() {
        final AtomicInteger count = this.count;
        if (count.get() == 0)
            return null;
        E x = null;
        int c = -1;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lock();
        try {
            if (count.get() > 0) {
                x = dequeue();
                c = count.getAndDecrement();
                if (c > 1)
                    notEmpty.signal();
            }
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
      	// 成功就会返回实际的元素，否则返回null
        return x;
    }
```

内部调用的dequeue方法

```java
    private E dequeue() {
        // assert takeLock.isHeldByCurrentThread();
        // assert head.item == null;
        Node<E> h = head;
        Node<E> first = h.next;
        h.next = h; // help GC
        head = first;
        E x = first.item;
        first.item = null;
        return x;
    }
```

不是特别好理解的话，就看图吧 

![](https://raw.githubusercontent.com/baofeidyz/images/master/img/20190721160516.png)

##### take

```java
    public E take() throws InterruptedException {
        E x;
        int c = -1;
        final AtomicInteger count = this.count;
        final ReentrantLock takeLock = this.takeLock;
        takeLock.lockInterruptibly();
        try {
            while (count.get() == 0) {
              	// 阻塞等待
                notEmpty.await();
            }
            x = dequeue();
            c = count.getAndDecrement();
            if (c > 1)
                notEmpty.signal();
        } finally {
            takeLock.unlock();
        }
        if (c == capacity)
            signalNotFull();
        return x;
    }
```

##### remove

```java
    public boolean remove(Object o) {
        if (o == null) return false;
      	// remove操作的位置不固定，所以需要对两个锁都进行加锁
        fullyLock();
        try {
            for (Node<E> trail = head, p = trail.next;
                 p != null;
                 trail = p, p = p.next) {
              	// 判断是否找到对象
                if (o.equals(p.item)) {
                  	// 修改节点的链接信息，同时调用notFull的signal方法
                    unlink(p, trail);
                    return true;
                }
            }
            return false;
        } finally {
            fullyUnlock();
        }
    }
```

内部调用的fullyLock方法 unlink方法以及fullyUnlock方法

```java
  void fullyLock() {
      putLock.lock();
      takeLock.lock();
  }
  void unlink(Node<E> p, Node<E> trail) {
    // assert isFullyLocked();
    // p.next is not changed, to allow iterators that are
    // traversing p to maintain their weak-consistency guarantee.
    p.item = null;
    trail.next = p.next;
    if (last == p)
      last = trail;
    // 判断当前元素总数是否等于最大值
    if (count.getAndDecrement() == capacity)
      notFull.signal();
  }
  void fullyUnlock() {
    takeLock.unlock();
    putLock.unlock();
  }
```

#### 总结

> take poll方法的特性与ArrayBlockingQueue一致，唯一不同的是remove方法中会同时对取 拿两个锁进行加锁

ArrayBlockingQueue因为内部实现是通过数组实现，所以其在初始化的时候必须要指定大小，且不可变，在删除操作的时候会移动元素。

LinkedBlockingQueue内部实现是通过单向链表实现，本身没有边界，但默认最大值为2^31-1。可同时操作读和写，在删除操作时需要给读写同时加锁。

### DelayQueue

Delayed 元素的一个无界阻塞队列，只有在延迟期满时才能从中提取元素。该队列的头部 是延迟期满后保存时间最长的 Delayed 元素。如果延迟都还没有期满，则队 列没有头部，并且 poll 将返回 null。当一个元素的 getDelay(TimeUnit.NANOSECONDS) 方法返回一个小于等于 0 的值时，将发生到期。即使无法使用 take 或 poll 移除未到期的元素，也不会将这些元素作为正常元素对待

### SynchronousQueue

SynchronousQueue 是一个特殊的队列，它的内部同时只能够容纳单个元素。如果该队列已有一元素的话，试图向队列中插入一个新元素的线程将会阻塞，直到另一个线程将该元素从队列中抽走。同样，如果该队列为空，试图向队列中抽取一个元素的线程将会阻塞，直到另一个线程向队列中插入了一条新的元素。

## 拒绝策略

主要有四种

### ThreadPoolExecutor.AbortPolicy

默认的ThreadPoolExecutor.AbortPolicy   处理程序遭到拒绝将抛出运行时RejectedExecutionException

```java
    public static class AbortPolicy implements RejectedExecutionHandler {
        /**
         * Creates an {@code AbortPolicy}.
         */
        public AbortPolicy() { }

        /**
         * Always throws RejectedExecutionException.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         * @throws RejectedExecutionException always
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```

### ThreadPoolExecutor.CallerRunsPolicy

线程调用运行该任务的 execute 本身。此策略提供简单的反馈控制机制，能够减缓新任务的提交速度

```java
	public static class CallerRunsPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code CallerRunsPolicy}.
         */
        public CallerRunsPolicy() { }

        /**
         * Executes task r in the caller's thread, unless the executor
         * has been shut down, in which case the task is discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                r.run();
            }
        }
    }
```

### ThreadPoolExecutor.DiscardPolicy

 不能执行的任务将被删除

```java
    public static class DiscardPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardPolicy}.
         */
        public DiscardPolicy() { }

        /**
         * Does nothing, which has the effect of discarding task r.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        }
    }
```

### ThreadPoolExecutor.DiscardOldestPolicy

 如果执行程序尚未关闭，则位于工作队列头部的任务将被删除，然后重试执行程序（如果再次失败，则重复此过程）

```java
	public static class DiscardOldestPolicy implements RejectedExecutionHandler {
        /**
         * Creates a {@code DiscardOldestPolicy} for the given executor.
         */
        public DiscardOldestPolicy() { }

        /**
         * Obtains and ignores the next task that the executor
         * would otherwise execute, if one is immediately available,
         * and then retries execution of task r, unless the executor
         * is shut down, in which case task r is instead discarded.
         *
         * @param r the runnable task requested to be executed
         * @param e the executor attempting to execute this task
         */
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            if (!e.isShutdown()) {
                e.getQueue().poll();
                e.execute(r);
            }
        }
    }
```

## 参考资料

[极客时间-Java并发编程实战](https://time.geekbang.org/column/intro/159?code=4e5ChyziCCisKCETno4QJHdW0BXcHigtqXBCNDlJuIE%3D&from=singlemessage&isappinstalled=0)

[《Java高并发编程详解：多线程与架构设计 (Java核心技术系列) 》](https://www.amazon.cn/dp/B07D9KBGR6/ref=sr_1_1?__mk_zh_CN=亚马逊网站&keywords=多线程与架构设计&qid=1554224223&s=gateway&sr=8-1)

[Java阻塞队列ArrayBlockingQueue和LinkedBlockingQueue实现原理分析](https://fangjian0423.github.io/2016/05/10/java-arrayblockingqueue-linkedblockingqueue-analysis/)

[LinkedBlockingQueue源码解析](https://www.cnblogs.com/java-zhao/p/5135958.html)