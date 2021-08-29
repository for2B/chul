---
layout:     post
title:      "ThreadPoolExecutor java线程池原理分享"
subtitle:   "并发"
date:       2021-08-14
author:     "CHuiL"
header-img: "img/java-bg.png"
tags:
    - java
---


[TOC]

# ThreadPoolExecutor java线程池原理分享

## ThreadPoolExcutor的创建

```
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)
```

- corePoolSize: 线程池允许持有的空闲线程数
- maximumPoolSize：线程池最多持有的线程数
- keepAliveTime：空闲线程阻塞等待新任务的最长等待时间
- unit： keepAliveTime参数的时间单位
- BlockingQueue<Runnable> workQueue：用于保存任务的队列
- ThreadFactory threadFactory:执行创建新线程的工厂
- RejectedExecutionHandler handler：执行被阻塞时使用的处理程序


**allowCoreThreadTimeOut** 参数默认设置为false，如果该值为true，表示及时是corePoolSize数量内的线程，空闲时间超过keepAliveTime也会被回收。

当线程数大于corePoolSize，小于等于maximumPoolSize时，如果线程空闲等待时间超过keepAliveTime，则该线程会被回收。

### 常用的三种任务队列管理策略
- 直接提交：使用==SynchronousQueue==队列。该对列无容量,将直接将任务转交给线程去执行，如果没有立即可执行的线程，将创建一个新的线程。通常需要无限的maximumPoolSizes以避免拒绝新提交的任务.
- 无界队列：使用==LinkedBlockingQueue==队列。任务一直往队列里面加，线程数只有corePoolSize。在瞬时爆发的请求到来时，能起到缓冲的作用，避免瞬时创建大量线程，但是如果处理速度慢于任务入队速度，工作队列可能无限增长。
- 有界队列：使用==ArrayBlockingQueue==队列。队列大小和最大池大小可以相互权衡：大队列和小线程池可以减少cpu的使用率，操作系统资源和上下文切换开销，但是吞吐会降低；小队列则对应的需要更大的线程池，这使得cpu被充分利用，但是要注意，过多的线程可能意味着更多调度，这反而使得吞吐降低。


### Executors提供的几种线程池
FixedThreadPool  
- 这里线程数从头到尾都一直是nThreads个吗？
```
    public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
    }
```

SingleThreadExecutor
```
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```
- 队列能否换成ArrayBlockingQueue队列？


CachedThreadPool
```
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```
- 最大线程是真的能达到Integer.MAX_VALUE个吗？

## 线程池生命周期

### ctl变量
ctl是一个int变量，共32位
![image](/chuil/img/java/tpe-1.png)

所以线程池最高可以有2^29^-1个线程（约5亿），而不是2^31^-1个线程。

```
    private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
    private static final int COUNT_BITS = Integer.SIZE - 3; //SIZE 是32
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
通过ctl我们知道线程池有5个状态
- RUNNING：接收新任务，处理队列中的任务
- SHUTDOWN：不再接受新任务，但是任然处理队列中的任务
- STOP：不再接收新任务，并且不处理队列中的任务。中断正在执行的任务
- TIDYING：所有任务都已经终止。workCount为0，
- TERMINATED：已经完成终止。

![image](/chuil/img/java/tpe-2.png)

## 任务提交

![image](/chuil/img/java/tpe-3.png)

```
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();//重复检查线程池状态
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }

```

## 线程(worker)的生命周期
维护管理线程的类。使用Woker来进行任务执行。线程池中的线程都是以Worker的形式进行工作的。 以下是基本属性，继承了AQS，实现了Runnable。
```
    private final class Worker extends AbstractQueuedSynchronizer implements Runnable
    {

        /** Thread this worker is running in.  Null if factory fails. */
        final Thread thread;
        
        /** Initial task to run.  Possibly null. */
        Runnable firstTask;
        
        /** Per-thread task counter */
        volatile long completedTasks;
    }
```

### addWorker() - 线程创建
```
    private boolean addWorker(Runnable firstTask, boolean core) {
        //重复循环检查状态，设置ctl值，设置成功则退出循环；
        //线程池状态不允许创建worker或者线程数量限制，则直接返回
        retry:
        for (;;) {
            int c = ctl.get();
            int rs = runStateOf(c);

            // SHUTDOWN状态不接收新任务，但是还需要执行队列里的任务，所以SHUTDOWN状态下如果队列里还有任务，则应该允许创建worker
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
                    t.start(); //执行worker
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

### runWork() - 线程执行
![image](/chuil/img/java/tpe-4.png)
```
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

### getTask() 决定线程生死，生则返回可执行任务，死则返回null

![image](/chuil/img/java/tpe-5.png)

```
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

### processWorkerExit()处理worker退出
```
    private void processWorkerExit(Worker w, boolean completedAbruptly) {
        if (completedAbruptly) // 该值为true，说明不是正常的线程退出，没有执行ctl的值更新
            decrementWorkerCount();

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            completedTaskCount += w.completedTasks;
            workers.remove(w);
        } finally {
            mainLock.unlock();
        }

        tryTerminate();

        int c = ctl.get();
        // running，shutdown状态，线程意外退出时判断
        if (runStateLessThan(c, STOP)) {
            if (!completedAbruptly) {
                //min 理解为最少应该持有的线程数
                int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
                if (min == 0 && ! workQueue.isEmpty())
                    min = 1;
                if (workerCountOf(c) >= min)
                    return; // replacement not needed
            }
            addWorker(null, false);
        }
    }
```

## woker的锁的用途
这里的锁，主要实现的是不可重入的互斥锁。以防止worker自身调用了调整pool的动态方法之后，将自身给中断掉。比如调用setPoolCoreSize()，setKeepAliveTime()等;

worker上锁和释放锁的地方，主要在runWorker中。在获取到task并且执行前，需要上锁，然后在执行task完毕之后释放锁。 

这里主要强调的是一个不可重入。  
setPoolCoreSize()调用之后，可以重新设置corePoolSize()。如果新的值比原来的小，那么当前多余的worker就需要被中断。这一步的操作，就是通过遍历workers，然后尝试将==等待中的空闲线程中断==。以让他在getTask()循环中重新判断决定work是否保留。
```
    private void interruptIdleWorkers(boolean onlyOne) {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers) {
                Thread t = w.thread;
                if (!t.isInterrupted() && w.tryLock()) {
                    try {
                        t.interrupt();
                    } catch (SecurityException ignore) {
                    } finally {
                        w.unlock();
                    }
                }
                if (onlyOne)
                    break;
            }
        } finally {
            mainLock.unlock();
        }
    }
```
假设这里使用了可重入的锁，那么一个worker自身tryLock()自身的锁会成功，那么他会自己将自己中断掉。所以要避免这种事情的发送。

而这种情况，可能发生在runWorker循环中。观察前面的代码，在worker上锁之后，执行任务前后，会调用beforeExecute和afterExecute，这两个方法实在锁住期间调用的，这里面就可能执行了调整pool的方法，导致worker自己中断了自己。

## 如何中断任务？

```
    public List<Runnable> shutdownNow() {
        List<Runnable> tasks;
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            checkShutdownAccess();
            advanceRunState(STOP);
            interruptWorkers();
            tasks = drainQueue();
        } finally {
            mainLock.unlock();
        }
        tryTerminate();
        return tasks;
    }
    

    private void interruptWorkers() {
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            for (Worker w : workers)
                w.interruptIfStarted();
        } finally {
            mainLock.unlock();
        }
    }
    
    void interruptIfStarted() {
            Thread t;
            if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                }
            }
        }
```
前面interruptIdleWorkers的中断是只有获取锁成功之后才会中断该worker，即空闲的worker，而这里是中断所有的worker。  

但是中断仅仅只是发送中断信号而已，如果具体的任务执行中没有对中断信号进行响应，那么该中断信号依旧是无效的。可能会有任务会一直执行，始终不会退出。