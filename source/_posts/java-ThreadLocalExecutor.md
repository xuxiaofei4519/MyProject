---
title: ThreadLocalExecutor解析
date: 2018-08-12 20:33:38
copyright: true
tags:
 - java
categories:
 - java
---

{% cq %}
通过学习Btrace想要追踪线程池状态，公司常用ThreadPoolExecutor线程池，由于对其内部原理与实现细节不够了解，因此本文对线程池进行简要分析
{% endcq %}
<!-- more -->

### 构造函数
{% codeblock lang:java %}
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
{% endcodeblock %}
- corePoolSize 核心线程数
- maximumPoolSize 最大线程数
- keepAliveTime 线程空闲时间
- TimeUnit 时间单位
- BlockingQueue<Runnable> 等待队列类型
- ThreadFactory 线程工厂
- RejectedExecutionHandler 拒绝策略

### 状态定义
{% codeblock lang:java %}
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
{% endcodeblock %}
> ThreadPoolExecutor是将Int的高3位用来标志当前线程池的状态，低29位标志当前线程池的线程数。由此我们可以得知最大线程数是2^29-1。
- RUNNING -1左移29位即11111....111111 << (29) 最终高3位是111
- SHUTDOWN 0左移29位00000.....000000 <<  (29) 最终高3位是000
- STOP 1左移29位000000.....00001 << (29) 最终高3位是001
- TIDYING 2左移29位000000...00002 << (29) 最终高3位是002
- TERMINATED 3左移29位000000...003 << (29) 最终高3位是003
- runStateOf 获取当前线程池的状态
- workerCountOf 获取当前线程池线程数目
- ctlOf 根据线程数和线程池状态计算代表当前线程状态的int值

### 线程池执行流程
- 1.调用ThreadPoolExecutor的execute方法提交线程，如果线程池内的线程小于核心线程数，那么就创建新线程执行任务
- 2.如果线程池内的线程数大于或等于核心线程数，那么就会将该线程加入任务队列，如果不能加入任务队列，那么如果当前线程池内的线程数小于最大线程数，那么创建新线程执行任务。
- 3.如果当前线程池内的线程数超出maximumPoolSize，那么将执行拒绝策略(RejectedExecutionHandler)

### execute方法
{% codeblock lang:java %}
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    /*
     * 三个步骤:
     *
     * 1.如果运行的线程少于corepoolol大小，请尝试以给定的任务作为第一个任务启动一个新线程。
     * 对addWorker的调用原子性地检查运行状态和workerCount，从而通过返回false防止假警报，假警报会在不应该的情况下添加线程。
     *
     * 2.如果一个任务可以成功排队，那么我们仍然需要再次检查是否应该添加一个线程(因为自上次检查以来已有线程死亡)，
     * 或者在进入该方法后池关闭。因此，我们重新检查状态，如果有必要，如果停止则回滚排队，如果没有，则启动一个新线程。
     *
     * 3.如果不能对任务进行排队，则尝试添加一个新任务线程。如果它失败了，我们知道我们已经关闭或饱和了所以拒绝这个任务。
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
{% endcodeblock %}
> 对上面内容做详细分析(重要):
> 1.首先判断当前线程池的状态以及当前线程的数目，判断当前线程的数目是否小于核心线程池的限制，如果小于，就新建线程添加任务
> 2.如果超过核心线程池限制，那么判断线程池状态是否是Running，并且将任务添加到任务队列，等待线程处理，如果这时候线程数为0，那么就新建空线程，执行队列中的任务
> 3.如果往任务队列中没有添加成功，那么就说明当前线程超出最大线程数限制，执行拒绝策略。

### addWorker方法
{% codeblock lang:java %}
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
{% endcodeblock %}

- 1.首先对线程池，任务队列进行前期检查
- 2.对当前线程池中线程数的校验
- 3.新建Worker，Worker类集成AbstractQueuedSynchronizer并且实现Runnable接口，说明Worker可以执行任务并且支持AQS(独占锁)
- 4.添加任务的时候使用可重入锁锁住当前操作，防止并发产生一些脏读问题，导致线程池内的参数不准确，进而导致线程执行任务失败。
- 5.添加任务成功后,就执行该线程，内部最终调用Runnable的run方法
- 6.通过返回boolean类型数据，告诉调用者添加任务是否成功。


### Shutdown方法
{% codeblock lang:java %}
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(SHUTDOWN);
        interruptIdleWorkers();
        onShutdown(); // hook for ScheduledThreadPoolExecutor
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
{% endcodeblock %}

- 1.判断是否拥有操作这些线程的权限
- 2.设置线程池的状态为SHUTDOWN
- 3.中断所有空闲线程(即没有执行任务的线程)
- 4.onShutdown() 该类中并实现该方法
- 5.尝试转换为最终状态TERMINATED

### 总结
> 本文只是简要分析了ThreadPoolExecutor的大体流程，以及具体的一些细节，由于其核心方法addWorker还有疑问，因此只是对其简要分析，后期会更新本文追加其详细内容。