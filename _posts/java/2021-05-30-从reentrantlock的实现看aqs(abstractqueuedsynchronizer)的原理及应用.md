---
layout:     post
title:      "从reentrantlock的实现看aqs(abstractqueuedsynchronizer)的原理及应用"
subtitle:   "并发"
date:       2021-05-30
author:     "CHuiL"
header-img: "img/java-bg.png"
tags:
    - java
---

[TOC]
# java concurrent包中常用同步工具原理




## java AQS(AbstractQueuedSynchronizer)
AQS是一个框架，用于帮助实现依赖于FIFO等待队列（Thread）的阻塞锁和相关同步器；  

他是一个抽象类，有个单原子int值表示状态，有个队列来管理等待线程。

他提供基本的获取和更改状态值的方法，并在内部实现了尝试获取失败后，对当前线程进行入队和阻塞的操作，以及在释放之后，对阻塞队列里线程执行唤醒操作。而具体得获取获取和释放的规则，由具体实现类决定。

AQS还支持独占和共享两种模式
- 独占模式：同一时刻只能有一个线程获得锁。
- 共享模式：同一时刻可能有多个线程持有锁。

同时AQS为实现可中断锁，有限时间阻塞，以及tryLock提供了支持。


以下是最基本的数据结构
```
    /**
     * Head of the wait queue, lazily initialized.
     */
    private transient volatile Node head;

    /**
     * Tail of the wait queue. After initialization, modified only via casTail.
     */
    private transient volatile Node tail;

    /**
     * The synchronization state.
     */
    private volatile int state;
```


### AQS主要方法

独占锁 | 共享锁 | 需要具体类实现 | 作用
---|--- |--- |---
acquire(int arg) | acquireShared(int arg) | 否 | 获取锁
release(int arg) | releaseShared(int arg) |否 | 释放锁
tryAcquire(int arg) | tryAcquireShared(int arg) | 是 | 具体获取锁的逻辑
tryRelease(int arg) | tryReleaseShared(int arg) |是 | 具体释放锁的逻辑
acquireQueued(final Node node, int arg) | doAcquireShared(int arg) | 否 | 获取锁失败后调用，阻塞线程以及唤醒后尝试获取锁
unparkSuccessor(h) | doReleaseShared() |否 | 唤醒线程
tryAcquireNanos(int arg, long nanosTimeout) | tryAcquireSharedNanos(int arg, long nanosTimeout) | 否 |有限时间阻塞的去获取锁
acquireInterruptibly(int arg) | acquireSharedInterruptibly(int arg) | 否 | 可中断的获取锁
doAcquireInterruptibly(int arg) | doAcquireSharedInterruptibly(int arg) | 否 | 阻塞线程以及唤醒后尝试获取锁- 阻塞可中断
doAcquireNanos(int arg, long nanosTimeout) | doAcquireSharedNanos(int arg, long nanosTimeout) | 否 | 阻塞线程以及唤醒后尝试获取锁-有限时间阻塞

状态值的操作
- void setState(int newState)
- int getState()
- boolean compareAndSetState(int expect, int update)


### 独占模式获取锁
![image](/chuil/img/java/aqs-1.png)


### 独占锁释放锁流程
![image](/chuil/img/java/aqs-2.png)


### 共享锁获取锁
```
private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```
共享锁获取锁的大致逻辑与独占锁是一样的，不同的地方在于`setHead(node) -> setHeadAndPropagate(node, r)`

```
    private void setHeadAndPropagate(Node node, int propagate) {
        Node h = head; // Record old head for check below
        setHead(node);
        if (propagate > 0 || h == null || h.waitStatus < 0 ||
            (h = head) == null || h.waitStatus < 0) {
            Node s = node.next;
            if (s == null || s.isShared())
                doReleaseShared();
        }
    }
```

在一定条件下调用了`doReleaseShared()`来唤醒后继的节点。这是因为在共享锁模式下，锁可以被多个线程所共同持有，既然当前线程已经拿到共享锁了，那么就可以直接通知后继节点来拿锁，而不必等待锁被释放的时候再通知。

### 共享锁释放

```
    private void doReleaseShared() {
        for (;;) {
            Node h = head;
            if (h != null && h != tail) {
                int ws = h.waitStatus;
                if (ws == Node.SIGNAL) {
                    if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                        continue;            // loop to recheck cases
                    unparkSuccessor(h);
                }
                else if (ws == 0 &&!compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                    continue;                // loop on failed CAS
            }
            if (h == head) //由于获取锁成功后也会调用该方法，所以head可能发生变化，一旦发生变化，那么他会继续执行唤醒操作
                break;
        }
    }
```
该方法调用的地方
- acquireShared方法的末尾
- releaseShared方法中

在共享锁中，持有共享锁的线程可以有多个，这些线程都可以调用releaseShared方法释放锁；而这些线程想要获得共享锁，则它们必然曾经成为过头节点，或者就是现在的头节点。因此，如果是在releaseShared方法中调用的doReleaseShared，可能此时调用方法的线程已经不是头节点所代表的线程了，头节点可能已经被易主好几次了。


## AQS的具体应用-java常用同步工具类

### ReentrantLock
常见的独占锁，可以用来代替synchronized实现锁机制，相比synchronized，有更多灵活的功能，可响应中断，超时，尝试获取锁，以及公平和非公平两种类型的锁。


#### 公平锁的tryAcquire()
```
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (!hasQueuedPredecessors() && //查询是否有任何线程等待获取的时间比当前线程长。
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) { //重入
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

#### 非公平锁中的tryAcquire()
```
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

公平与非公平的对比 主要区别就在于有没有多执行一步检查`!hasQueuedPredecessors()`

#### ReentrantLock的tryRealease()
```
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
```



### CountDownLatch
闭锁，相当于一扇门，在达到结束状态之前，这扇门一直是关闭的，所有线程都无法进入，到达结束状态之后，这扇门就会打开，所有线程都可以进入；后面不会再关闭；
- await() : 使当前线程等待直到闩锁倒计时为零
- countDown(): 递减锁存器的计数，如果计数达到零，则释放所有等待的线程。

```
    public void await() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    ...
    public void countDown() {
        sync.releaseShared(1);
    }
```

```
private static final class Sync extends AbstractQueuedSynchronizer {
    private static final long serialVersionUID = 4982264981922014374L;
    
    //初始化时直接调用AQS设置状态值
    Sync(int count) {
        setState(count);
    }

    int getCount() {
        return getState();
    }
    
    
    //调用await之后，会调用该方法，判断是否能获取成功，这里就是判断当前状态值是否0。
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    //在调用countDown方法后，会调用该方法，将状态值-1，如果状态值不为0，则返回false，AQS不唤醒阻塞的线程，为0时释放锁，唤醒阻塞线程
    protected boolean tryReleaseShared(int releases) {
        // Decrement count; signal when transition to zero
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0; //不为0，说明还没达到释放锁的条件
        }
    }
}
```

### Semaphore
信号量；它允许通过控制一定数量的permit，来达到限制资源访问的目的。他有公平和非公平两种模式

- acquire():请求permit，permit-1，若permit为0，阻塞当前线程直到permit不为0；
- release()：释放permit，，permit+1，若permit，如果有线程阻塞请求，则会进行唤醒操作。


```
    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }
    ...
    public void release() {
        sync.releaseShared(1);
    }
```



```
    abstract static class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 1192457210091910933L;

        Sync(int permits) {
            setState(permits);
        }

        final int getPermits() {
            return getState();
        }

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
        ....
    }
```


```
    static final class FairSync extends Sync {
        private static final long serialVersionUID = 2014338818796000944L;

        FairSync(int permits) {
            super(permits);
        }

        protected int tryAcquireShared(int acquires) {
            for (;;) {
                if (hasQueuedPredecessors())
                    return -1;
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
    }
```




### 总结
java同步工具原理基本都是依靠AQS实现的，而AQS最关键的地方,其实就是以下三点
- 状态值
- 阻塞队列管理
- CAS



## java cas原理

> Unsafe类：Unsafe类通过JNI的方式访问本地的C++实现库从而使java具有了直接操作内存空间的能力

compareAndSetInt方法
```
/**
 * Atomically updates Java variable to {@code x} if it is currently
 * holding {@code expected}.
 *
 * <p>This operation has memory semantics of a {@code volatile} read
 * and write.  Corresponds to C11 atomic_compare_exchange_strong.
 *
 * @return {@code true} if successful
 */
@HotSpotIntrinsicCandidate
public final native boolean compareAndSetInt(Object o, long offset,int expected,int x);
```

查看openJDK中的源码 `jdk9/hotspot/src/share/vm/prims/unsafe.cpp`  
尾段有个保存方法对应关系的静态数组：
```
static JNINativeMethod jdk_internal_misc_Unsafe_methods[] = {
    ......
    
    {CC "compareAndSetInt",   CC "(" OBJ "J""I""I"")Z",  FN_PTR(Unsafe_CompareAndSetInt)},
    {CC "compareAndSetLong",  CC "(" OBJ "J""J""J"")Z",  FN_PTR(Unsafe_CompareAndSetLong)},
    {CC "compareAndExchangeObject", CC "(" OBJ "J" OBJ "" OBJ ")" OBJ, FN_PTR(Unsafe_CompareAndExchangeObject)},
    {CC "compareAndExchangeInt",  CC "(" OBJ "J""I""I"")I", FN_PTR(Unsafe_CompareAndExchangeInt)},
    {CC "compareAndExchangeLong", CC "(" OBJ "J""J""J"")J", FN_PTR(Unsafe_CompareAndExchangeLong)},
    ......
}
```
找到对应的`Unsafe_CompareAndExchangeInt

```
UNSAFE_ENTRY(jboolean, Unsafe_CompareAndSetInt(JNIEnv *env, jobject unsafe, jobject obj, jlong offset, jint e, jint x)) {
  oop p = JNIHandles::resolve(obj);
  //根据成员变量value反射后计算出的内存偏移值offset去内存中取指针addr
  jint* addr = (jint *)index_oop_from_field_offset_long(p, offset); 

  return (jint)(Atomic::cmpxchg(x, addr, e)) == e;
} UNSAFE_END
```

`Atomic::cmpxchg`该方法真正调用的由当时具体执行的操作系统和内核决定，这里选取linux_x86的实现
`jdk9/hotspot/src/os_cpu/linux_x86/vm/atomic_linux_x86`

```
//检查是否为单核
#define LOCK_IF_MP(mp) "cmp $0, " #mp "; je 1f; lock; 1: "
....

inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value, cmpxchg_memory_order order) {
  //该方法用于获取当前系统处理器核心数
  int mp = os::is_MP();
  __asm__ volatile (LOCK_IF_MP(%4) "cmpxchgl %1,(%3)"
                    : "=a" (exchange_value)
                    : "r" (exchange_value), "a" (compare_value), "r" (dest), "r" (mp)
                    : "cc", "memory");
  return exchange_value;
}
```

宏定义用`"cmp $0, " #mp "`检查核心是否为单核：
- 是：跳到1f，执行CPU指令cmpxchgl %1,(%3)。1f的意思是1after
- 不是：则跳到1f前先通过lock给==总线上锁==，令物理处理器的其他核心不能通过总线访存，保证指令操作的原子性。


### 总线锁作用的效果
- 处理器使用LOCK#信号达到锁定总线，来解决原子性问题，当一个处理器往总线上输出LOCK#信号时，其它处理器的请求将被阻塞，此时该处理器独占共享内存。
- 其它CPU对内存的读写请求都会被阻塞，直到锁释放
- lock期间的写操作会回写已修改的数据到主内存，同时通过缓存一致性协议让其它CPU相关缓存行失效




## 参考

- [逐行分析AQS源码(3)——共享锁的获取与释放](https://segmentfault.com/a/1190000016447307)
- [从ReentrantLock的实现看AQS的原理及应用](https://tech.meituan.com/2019/12/05/aqs-theory-and-apply.html)
- [The java.util.concurrent Synchronizer Framework](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
- [聊聊CPU的LOCK指令](https://albk.tech/%E8%81%8A%E8%81%8ACPU%E7%9A%84LOCK%E6%8C%87%E4%BB%A4.html)    
- [Java CAS底层实现详解](https://www.jianshu.com/p/0e312402f6ca)
- [Unsafe类源码解析](https://zhuanlan.zhihu.com/p/81027072)