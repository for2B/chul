---
layout:     post
title:      "java thread中断"
subtitle:   "并发"
date:       2021-08-09
author:     "CHuiL"
header-img: "img/java-bg.png"
tags:
    - java
---

Java 程序中有不止一条执行线程，只有当所有的线程都运行结束的时候，这个 Java 程序才算运行结束。

官方的话给你描述一下：当所有的非守护线程运行结束时，或者其中一个线程调用了 System.exit() 方法时，这个 Java 程序才能运行结束。

线程中断即线程运行过程中被其他线程给打断了，它与 stop 最大的区别是：stop 是由系统强制终止线程，而线程中断则是给目标线程发送一个中断信号，如果目标线程没有接收线程中断的信号并结束线程，线程则不会终止，具体是否退出或者执行其他逻辑由目标线程决定。
我们来看下线程中断最重要的 3 个方法，它们都是来自 Thread 类！


#### 1、java.lang.Thread#interrupt
中断目标线程，给目标线程发一个中断信号，线程被打上中断标记。


#### 2、java.lang.Thread#isInterrupted()
判断目标线程是否被中断，不会清除中断标记。


#### 3、java.lang.Thread#interrupted
判断目标线程是否被中断，会清除中断标记。

#### 示例1（中断失败）
```
private static void test1() {
    Thread thread = new Thread(() -> {
        while (true) {
            Thread.yield();
        }
    });
    thread.start();
    thread.interrupt();
}
```
请问示例1中的线程会被中断吗？答案：不会，因为虽然给线程发出了中断信号，但程序中并没有响应中断信号的逻辑，所以程序不会有任何反应。

#### 示例2：（中断成功）
```
private static void test2() {
    Thread thread = new Thread(() -> {
        while (true) {
            Thread.yield();

            // 响应中断
            if (Thread.currentThread().isInterrupted()) {
                System.out.println("青秧线程被中断，程序退出。");
                return;
            }
        }
    });
    thread.start();
    thread.interrupt();
}
```

#### 实例3：

```
private static void test3() throws InterruptedException {
    Thread thread = new Thread(() -> {
        while (true) {
            // 响应中断
            if (Thread.currentThread().isInterrupted()) {
                System.out.println("青秧线程被中断，程序退出。");
                return;
            }

            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                System.out.println("青秧线程休眠被中断，程序退出。");
            }
        }
    });
    thread.start();
    Thread.sleep(2000);
    thread.interrupt();
}
```
sleep() 方法被中断后会清除中断标记，所以循环会继续运行。。