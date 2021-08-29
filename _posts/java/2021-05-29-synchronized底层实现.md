---
layout:     post
title:      "synchronized底层实现"
subtitle:   "exception"
date:       2021-05-29
author:     "CHuiL"
header-img: "img/java-bg.png"
tags:
    - java
---

# 一般描述

synchronized的实现主要利用的Monitor对象，每个类对象都会有一个Monitor对象，在java6之前，monitor的实现完全依靠操作系统内部的互斥锁(Mutex Lock)，需要进行用户态到内核态的切换，是个无差别的重量级操作；    
      
而在java6之后，jvm对其进行优化，实现了锁的升级，即从偏向锁升级到轻量级锁，再进一步升级为重量级锁，同时还有锁粗化和锁消除的优化；    
在没有出现竞争的时候，默认会使用偏向锁，jvm会利用CAS操作在对象的MarkWord部分设置线程ID，表示这个对象偏向当前线程，所以不涉及真正的互斥锁。偏向锁时基于在很多应用场景中，大部分对象生命周期内最多只会被一个线程锁定，所以使用偏向所可以降低开销。    

而在有另外一个线程试图锁定某个偏向锁，JVM就会撤销偏向锁，升级为轻量级锁，轻量级锁会在获取锁的线程栈中开辟一片内存将对象头的Mark Word 复制过去，并将对象头的Mark word CAS设置为指向该内存区域的指针；设置成功就表示该线程获取到了该对象锁，另外尝试获取该锁的线程将自旋等待（因为此时任然不是真正意义上的互斥锁，所以也就没有调用底层来对线程进行挂起和唤醒），只有自旋超过一定时间，或者有第三个线程尝试获取锁，那么锁才会升级为重量级锁   

重量级锁就是利用操作系统Mutex Lock来实现了。对于获取不到锁的线程会进行挂起，释放锁后会将等待的线程唤醒，挂起和唤醒都是开销大的操作。  



# synchronized的原理(Monitor)
## java对象头
synchronize主要实现是依靠Monitor对象实现的，一般而言，synchronized使用的锁对象是存储在Java对象头里的；通过下图，我们可以很直观的看到，对于不同的锁，他们是如何利用Mark Word的； 
![image](/chuil/img/java/synchronized-1.png)

## Monitor
每个对象都存在着一个Monitor与之关联，静态类或者方法使用的Monitor这是该类的Class对象的Monitor；在java虚拟机中，monitor是由ObjectMoniror实现的，主要结构如下
```
ObjectMonitor() {
    _header       = NULL;
    _count        = 0; //记录个数
    _waiters      = 0,
    _recursions   = 0;
    _object       = NULL;
    _owner        = NULL;
    _WaitSet      = NULL; //处于wait状态的线程，会被加入到_WaitSet;如调用Object.wait()方法
    _WaitSetLock  = 0 ;
    _Responsible  = NULL ;
    _succ         = NULL ;
    _cxq          = NULL ;
    FreeNext      = NULL ;
    _EntryList    = NULL ; //处于等待锁block状态的线程，会被加入到该列表
    _SpinFreq     = 0 ;
    _SpinClock    = 0 ;
    OwnerIsThread = 0 ;
  }
```
ObjectMonitor中有两个队列，_WaitSet 和 _EntryList，用来保存ObjectWaiter对象列表( 每个等待锁的线程都会被封装成ObjectWaiter对象)，_owner指向持有ObjectMonitor对象的线程，当多个线程同时访问一段同步代码时，首先会进入 _EntryList 集合，当线程获取到对象的monitor 后进入 _Owner 区域并把monitor中的owner变量设置为当前线程同时monitor中的计数器count加1，若线程调用 wait() 方法，将释放当前持有的monitor，owner变量恢复为null，count自减1，同时该线程进入 WaitSe t集合中等待被唤醒。若当前线程执行完毕也将释放monitor(锁)并复位变量的值，以便其他线程进入获取monitor(锁)。如下图所示
![image](/chuil/img/java/synchronized-2.png)

### 使用synchronized时，怎么利用的monitor?
```
public class TestSynchronized {
    public int i;

    public void syncTask(){
        synchronized (this){ //代码块使用
            i++;
        }
    }
    //修饰方法
    public synchronized void syncTaskMethod(){
        i++;
    }
}

```
反编译后的字节码
```
  public void syncTask();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=3, locals=3, args_size=1
         0: aload_0
         1: dup
         2: astore_1
         3: monitorenter //调用monitorenter进入同步块
         4: aload_0
         5: dup
         6: getfield      #2                  // Field i:I
         9: iconst_1
        10: iadd
        11: putfield      #2                  // Field i:I
        14: aload_1
        15: monitorexit //调用monitorexit退出同步块
        16: goto          24
        19: astore_2
        20: aload_1
        21: monitorexit //用于异常结束时调用，保证代码块结束时一定释放monitor
        22: aload_2
        23: athrow
        24: return
      Exception table:
         from    to  target type
             4    16    19   any
            19    22    19   any
      LineNumberTable:
        line 13: 0
        line 14: 4
        line 15: 14
        line 16: 24
      StackMapTable: number_of_entries = 2
        frame_type = 255 /* full_frame */
          offset_delta = 19
          locals = [ class com/test/demo/Async/TestSynchronized, class java/lang/Object ]
          stack = [ class java/lang/Throwable ]
        frame_type = 250 /* chop */
          offset_delta = 4

  public synchronized void syncTaskMethod();
    descriptor: ()V
    flags: ACC_PUBLIC, ACC_SYNCHRONIZED//该标志表示该方法为同步方法
    Code:
      stack=3, locals=1, args_size=1
         0: aload_0
         1: dup
         2: getfield      #2                  // Field i:I
         5: iconst_1
         6: iadd
         7: putfield      #2                  // Field i:I
        10: return
      LineNumberTable:
        line 19: 0
        line 20: 10
}
SourceFile: "TestSynchronized.java"
```

从上面的字节码可以看到，显示调用synchronized代码块时，会自动调用monitorenter和monitorexit来进行加锁和释放锁；这里会发现最后多了一条monitorexit指令，这时为了在异常结束时，也能自动执行释放锁，这也是使用synchronized的好处，不用担心锁释放的问题。  

而使用synchronized修饰方法时，会在该方法的标志位设置ACC_SYCNRONIZED，用来表示该方法为同步方法，而JVM会根据该标志来进行moniror的加锁和释放锁。

### 偏向锁
当一个线程访问同步代码块并获取锁时，会在Mark Word里存储锁偏向的线程ID。在线程进入和退出同步块时不再通过CAS操作来加锁和解锁，而是检测Mark Word里是否存储着指向当前线程的偏向锁。引入偏向锁是为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径，因为轻量级锁的获取及释放依赖多次CAS原子指令，而偏向锁只需要在置换ThreadID的时候依赖一次CAS原子指令即可。  

一旦出现另外一个线程尝试去获取这个锁时，偏向锁马上结束，根据锁对象目前是否被锁定决定是否撤销偏向锁或是升级为轻量级锁；  

这里注意一个点，因为线程Id是存储在Mark Word 存放hashCode值的地方，所以如果该对象曾经调用过一次计算该对象的hashCode值，那么该值会被存储在Mark Word中，即偏向锁就无法使用了，如果此时该位置存放的是线程Id，则会理解撤销偏向锁状态；
### 轻量级锁
当前锁是偏向锁时，被另外的线程访问，偏向锁就会升级为轻量级锁，其他线程会通过自选的形式尝试获取锁，不会阻塞，从而提高性能；  
轻量级是相对于操作系统互斥量来实现的传统锁而言的；轻量级锁所适应的场景是线程交替执行同步块的情况  
若当前只有一个等待线程，则该线程通过自旋进行等待。但是当自旋超过一定的次数，或者一个线程在持有锁，一个在自旋，又有第三个来访时，轻量级锁升级为重量级锁。
### 轻量级锁的加锁过程
1. 虚拟机首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝，官方称之为 Displaced Mark Word。
2. 拷贝对象头中的Mark Word复制到锁记录中。
3. 拷贝成功后，虚拟机将使用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指针；并将Lock record里的owner指针指向object mark word；如果这个更新动作成功了，那么这个线程就拥有了该对象的锁，并且对象Mark Word的锁标志位设置为“00”，即表示此对象处于轻量级锁定状态，
4. 如果这个更新操作失败了，虚拟机首先会检查对象的Mark Word是否指向当前线程的栈帧，如果是就说明当前线程已经拥有了这个对象的锁，那就可以直接进入同步块继续执行。否则说明多个线程竞争锁，轻量级锁就要膨胀为重量级锁，锁标志的状态值变为“10”，Mark Word中存储的就是指向重量级锁（互斥量）的指针，后面等待锁的线程也要进入阻塞状态。  
![image](/chuil/img/java/synchronized-3.png)

### 轻量级锁的解锁过程
1. 通过CAS操作尝试把线程中复制的Displaced Mark Word对象替换当前的Mark Word。
2. 如果替换成功，整个同步过程就完成了。
3. 如果替换失败，说明有其他线程尝试过获取该锁（此时锁已膨胀），那就要在释放锁的同时，唤醒被挂起的线程。

### 重量级锁
升级为重量级锁时，锁标志的状态值变为“10”，此时Mark Word中存储的是指向重量级锁的指针（即monitor的地址)，此时等待锁的线程都会进入阻塞状态。Synchronized是通过对象内部的一个叫做监视器锁（monitor）来实现的。但是监视器锁本质又是依赖于底层的操作系统的Mutex Lock来实现的。mutex lock使用(可能)需要利用到系统调用，需要从用户态转换到内核态，开销大，并且获取不到锁的线程会被阻塞，无差别的阻塞与唤醒也是开销很大的操作。


### 锁消除
如下代码，这段代码并没有并发竞争的环境，但是由于使用了Vector的add方法，该方法使用synchronized进行了同步。
```
public void vectorTest(){
    Vector<String> vector = new Vector<String>();
    for(int i = 0 ; i < 10 ; i++){
        vector.add(i + "");
    }

    System.out.println(vector);
}
```
在运行这段代码时，JVM可以明显检测到变量vector没有逃逸出方法vectorTest()之外，所以JVM可以大胆地将vector内部的加锁操作消除。

### 锁粗化
同样是上面的代码，如果这段代码确实存在并发竞争的环境，vector每次add的时候都需要加锁操作，JVM检测到对同一个对象（vector）连续加锁、解锁操作，会合并一个更大范围的加锁、解锁操作，即加锁解锁操作会移到for循环之外。


## 关于重量级锁的实现 Mutex Lock
mutex lock互斥锁主要用于实现内核中的互斥访问功能。mutex lock内核互斥锁是在原子 API 之上实现的，但这对于内核用户是不可见的。对它的访问必须遵循一些规则：同一时间只能有一个任务持有互斥锁，而且只有这个任务可以对互斥锁进行解锁。互斥锁不能进行递归锁定或解锁。一个互斥锁对象必须通过其API初始化，而不能使用memset或复制初始化。一个任务在持有互斥锁的时候是不能结束的。互斥锁所使用的内存区域是不能被释放的。使用中的互斥锁是不能被重新初始化的。并且互斥锁不能用于中断上下文。但是互斥锁比当前的内核信号量选项更快，并且更加紧凑，因此如果它们满足您的需求，那么它们将是您明智的选择。



```
struct mutex {
 /* 1: unlocked, 0: locked, negative: locked, possible waiters */
        atomic_t                count;
        spinlock_t              wait_lock;
        struct list_head        wait_list;
        #ifdef CONFIG_DEBUG_MUTEXES
        struct thread_info      *owner;
        const char              *name;
        void                    *magic;
  #endif
  #ifdef CONFIG_DEBUG_LOCK_ALLOC
        struct lockdep_map      dep_map;
 #endif
 };
```
各字段解释：  
1、atomic_t count。指示互斥锁的状态：1 没有上锁，可以获得 0 被锁定，不能获得  负数 被锁定，且可能在该锁上有等待进程 初始化为没有上锁。

2、spinlock_t wait_lock。 等待获取互斥锁中使用的自旋锁。在获取互斥锁的过程中，操作会在自旋锁的保护中进行。初始化为为锁定。

3、struct list_head wait_list。 等待互斥锁的进程队列。


> mutex lock的底层实现是基于cpu和硬件原子操作

CPU如果提供一些用来构建锁的atomic指令，譬如x86的CMPXCHG（加上LOCK prefix），能够完成atomic的compare-and-swap （CAS），用这样的硬件指令就能实现spin lock。本质上LOCK前缀的作用是锁定系统总线（或者锁定某一块cache line）来实现atomicity，可以了解下基础的缓存一致协议譬如MSEI。简单来说就是，如果指令前加了LOCK前缀，就是告诉其他核，一旦我开始执行这个指令了，在我结束这个指令之前，谁也不许动。这样便实现了一次只能有一个核对同一个内存地址赋值。


## 参考
[深入理解Java并发之synchronized实现原理](https://blog.csdn.net/javazejian/article/details/72828483)  
[刨根问底synchronized | 锁系列-Java中的锁](https://cloud.tencent.com/developer/article/1082708)  
[jdk源码剖析二: 对象内存布局、synchronized终极原理](https://www.cnblogs.com/dennyzhangdd/p/6734638.html#_label0_0)  
[深入分析Synchronized原理](https://www.cnblogs.com/aspirant/p/11470858.html)