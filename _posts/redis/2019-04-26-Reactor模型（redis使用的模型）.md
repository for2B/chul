---
layout:     post
title:      "Reactor模型（redis使用的模型）"
subtitle:   ""
date:       2019-04-26
author:     "CHuiL"
header-img: "img/redis-bg.png"
tags:
    - redis
---

## 文件事件

![image](/chuil/img/redis/19-08-25-1.png)

Reactor模型底层实现还是基于I/O复用的，只是传统的IO复用虽然在技术上解决了单线程模式下高并发的性能问题，能够做到同时并发几十万连接；但是从软件工程层面来说，却有很多问题；一个连接的处理可能需要多个IO操作A1-An，相邻IO之间又有间隔时间;A每次处理完一个IO操作后又重新复用，这样A的处理就不是连续的，在他完成所有操作前可以被其他连接“插队”；这样就需要在连接存在期间保持连接的上下文，然后再下次复用返回时，根据上下文去执行对应的业务处理方法；全部处理完之后在删除；在这种情况下，我们的主程序需要关注各种不同类型的请求，不同的状态下，需要选择的不同处理业务的方法；当这些请求和状态增加时，我们的主程序将会变得十分难以维护；这个时候就需要一种模式框架来解决这些问题；  
Reactor(反应堆)就是解决以上问题的一种途径；如上图所示，反应堆维护着事件的处理，查看一下redis中的事件是如何注册和分发的；

```
//事件类型
typedef struct aeFileEvent {
    int mask; /* one of AE_(READABLE|WRITABLE|BARRIER) *///事件标记
    aeFileProc *rfileProc; //对应该事件的读处理函数
    aeFileProc *wfileProc;//对应该事件的写处理函数
    void *clientData; //client数据
} aeFileEvent;
```

```
//就绪事件
typedef struct aeFiredEvent {
    int fd; //文件描述符
    int mask; //事件标记
} aeFiredEvent;
```

```
//事件循环
typedef struct aeEventLoop {
    。。。。
    aeFileEvent *events; /* 已注册事件数组*/
    aeFiredEvent *fired; /* 已就绪的事件数组 */
    。。。。
} aeEventLoop;
```

```
//创建一个新事件，就是添加一个新的fd;并设置该fd监听的mask-事件类型，以及设置它的处理函数
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];

    if (aeApiAddEvent(eventLoop, fd, mask) == -1) //这里，根据IO复用模型，调用对应的函数，如果是select()，那就将fd添加到set中
        return AE_ERR;
    fe->mask |= mask; //设置监听事件类型
    if (mask & AE_READABLE) fe->rfileProc = proc; //设置处理函数
    if (mask & AE_WRITABLE) fe->wfileProc = proc;//设置处理函数
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

```
//以select()为例
static int aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask) {
    aeApiState *state = eventLoop->apidata;

    if (mask & AE_READABLE) FD_SET(fd,&state->rfds);
    if (mask & AE_WRITABLE) FD_SET(fd,&state->wfds);
    return 0;
}
```
例如，创建监听连接后，注册一个新事件；处理函数是 acceptTcpHandler

```
...
if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
....
```

新建立一个客户连接后，注册一个新事件；处理函数readQueryFromClient

```
if (aeCreateFileEvent(server.el,fd,AE_READABLE,
            readQueryFromClient, c) == AE_ERR)
        {
            close(fd);
            zfree(c);
            return NULL;
        }
```

在readQueryFromClient中会为客户连接建立一个client对象，包含很多客户连接相关的信息，也即上面提到的上下文环境；然后会read出用户命令，执行命令；执行完后会将客户端输出数据写入缓冲区，然后将clien对象加入到队列中，在每次重新循环前将他们取出并注册写出事件（下面代码中的beforesleep）；
  
  主程序一直循环

  ```
  void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
  ```
  在aeProcessEvent中

```
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;
    ......
    //这里就是io多路复用，可以是Select（），也可以是epoll，返回就绪fd个数
        numevents = aeApiPoll(eventLoop, tvp);
    ...
    //根据返回个数，循环就绪fire事件数组，根据mask标记执行对应的处理函数，这里就相当于事件分发器了
        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int fired = 0; /* Number of events fired for current fd. */
            int invert = fe->mask & AE_BARRIER;

            if (!invert && fe->mask & mask & AE_READABLE) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }

            /* Fire the writable event. */
            if (fe->mask & mask & AE_WRITABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            /* If we have to invert the call, fire the readable event now
             * after the writable one. */
            if (invert && fe->mask & mask & AE_READABLE) {
                if (!fired || fe->wfileProc != fe->rfileProc) {
                    fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                    fired++;
                }
            }

            processed++;
        }
    }
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
```

![image](/chuil/img/redis/19-08-25-2.png)


## 时间事件
起初一直不理解redis这个模式是如何做到既可以监控文件事件（socket）是否就绪，又可以监控定时事件；因为在监控的过程中如果没有新连接到来是会被阻塞的；那如果长时间没有客户来，那对单线程来说不就一直都阻塞在这上面了吗？关键的地方在于调用numevents = aeApiPoll(eventLoop, tvp);这句复用的这句话中，有个关键的参数tvp;详细了解之后就会明白了
 //时间事件结构
 ```
 typedef struct aeTimeEvent {
    long long id; /* time event identifier. */
   // 事件的到达时间
    long when_sec; /* seconds */
    long when_ms; /* milliseconds */
      // 事件处理函数
    aeTimeProc *timeProc;
     // 事件释放函数
    aeEventFinalizerProc *finalizerProc;
    void *clientData;
    struct aeTimeEvent *prev;
    struct aeTimeEvent *next;
} aeTimeEvent;
```

时间事件是链表结构进行存储的，为时间事件注册新事件的
```
long long aeCreateTimeEvent(aeEventLoop *eventLoop, long long milliseconds,
        aeTimeProc *proc, void *clientData,
        aeEventFinalizerProc *finalizerProc)
{
    long long id = eventLoop->timeEventNextId++;
    aeTimeEvent *te;

    te = zmalloc(sizeof(*te));
    if (te == NULL) return AE_ERR;
    te->id = id;
    aeAddMillisecondsToNow(milliseconds,&te->when_sec,&te->when_ms);
    //添加到链表中
    te->timeProc = proc;
    te->finalizerProc = finalizerProc;
    te->clientData = clientData;
    te->prev = NULL;
    te->next = eventLoop->timeEventHead;
    if (te->next)
        te->next->prev = te;
    eventLoop->timeEventHead = te;
    return id;
}
```
![image](/chuil/img/redis/19-08-25-3.png)

然后再上述aeProcessEvent函数中，每次进行复用阻塞前，会先获取一下距离当前最近的时间事件，然后获取一下时间间隔tvp；如果tvp大于0，那么将他设置为复用阻塞的时间，让复用在这间隔内监控，到期后出来，这样就可以执行定时任务了而不会一直阻塞；如果tvp小于0，说明现在就有定时任务要执行，设置tvp为0，复用不等待；

```
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;
    
    if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
      shortest = aeSearchNearestTimer(eventLoop);//获取最近的时间事件
       if (shortest) {
            long now_sec, now_ms;

            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;

            /* How many milliseconds we need to wait for the next
             * time event to fire? */
            long long ms = //求时间间隔
                (shortest->when_sec - now_sec)*1000 +
                shortest->when_ms - now_ms;

            if (ms > 0) {
                tvp->tv_sec = ms/1000;
                tvp->tv_usec = (ms % 1000)*1000;
            } else {
                tvp->tv_sec = 0;//间隔小于=0，所以tvp设置为0
                tvp->tv_usec = 0; 
            }
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }
      
    ......
    //这里就是io多路复用，可以是Select（），也可以是epoll，返回就绪fd个数
        numevents = aeApiPoll(eventLoop, tvp);
    ...
    //根据返回个数，循环就绪fire事件数组，根据mask标记执行对应的处理函数，这里就相当于时间分发器了
        for (j = 0; j < numevents; j++) {
       ......
        }
    }
    /* Check time events */
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop); //执行时间事件

    return processed; /* return the number of processed file/time events */
}
```


processTimeEvent函数的大致思路如下
1. 记录最新一次执行这个函数的时间，用于处理系统时间被修改产生的问题。
2. 遍历链表找出所有 when_sec 和 when_ms 小于现在时间的事件。
3. 执行事件对应的处理函数。
4. 检查事件类型，如果是周期事件则刷新该事件下一次的执行事件。
5. 否则从列表中删除事件。	
