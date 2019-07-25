---
layout: blog
title: redis线程模型
date: 2019-07-23 20:30:57
tags: [redis, epoll, select]
---
我们都知道Redis虽然以单线程运行，但是却可以支持非常高的并发请求，这里就不得不提Redis的线程模型了。

Redis在单线程下，以Reactor模式开发了一套ae事件库。ae事件库使用常用的IO多路复用机制，内部封装了底层的系统调用，给Redis提供了一组便于使用的接口，实现了Redis高效，高并发的使用特性。

## IO事件库概述

基于Reactor实现的IO框架库主要包括以下几个组件：句柄，事件多路分发器，事件处理器和具体的事件处理器。

### 句柄
当系统内核检测到就绪事件时，通过句柄通知应用程序这个事件。

### 事件多路分发器
事件会在随机时间由系统触发，所以需要循环的等待处理事件，等待事件一般使用IO复用技术实现。事件库将IO复用系统调用函数封装成统一接口，称为事件多路分发器，内部调用的是诸如select，epoll_wait，kevent等函数。同时事件多路分发器提供的还有增加事件和删除事件接口。

### 事件处理器和具体事件处理器
时间处理器执行事件对应的业务逻辑，它通常会对应一个或多个回调函数。在时间循环中，当事件触发时被调用执行。通常，IO框架库会定义一个接口，使用方实现自己的具体事件处理器。

### Reactor
Reactor是框架库的核心，它一般会实现几个主要方法：
- handle_events。这个方法执行事件的循环，包括等待事件，处理就绪事件。
- register_event。这个方法主要是向事件多路分发器中注册事件。
- remove_event。这个方法主要是移除多路事件分发器中的事件。

### IO多路复用系统调用函数
IO多路复用有多重机制，还有不同平台的不同实现，这里主要介绍下两个多路复用函数。

#### select系统调用
select系统调用的用途是：在一段时间内，监听用户感兴趣的文件描述符上的可读，可写和异常等事件。
select系统调用的原型如下：
```c
#include <sys/select.h>
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout);
```

- nfds参数指定了被监听的文件描述符的总数。
- readfds，writefds，exceptfds参数分别指向可读，可写和异常等事件对应的文件描述符集合。应用程序通过参数传入感兴趣的文件描述符，调用返回时，内核将修改它们来通知应用程序哪些文件描述符已经就绪。
- timeout用来设置函数调用超时时间。

select成功时返回就绪(可读，可写和异常)文件描述符的总数。如果在超时时间内没有任何文件描述符就绪，将返回-1，select失败时返回-1并设置errno，如果在调用等待期间，程序接收到信号，则select立即返回，并设置errno为EINTR。

select系统调用是POSIX接口标准，几乎在所有平台都支持。

#### epoll系统调用
epoll是Linux特有的IO多路复用机制，实现上和select有很大不同。上面我们看到select系统调用通过一个函数完成任务，而epoll是通过一组函数完成任务。并且，epoll把用户关心的文件描述符上的事件放在内核里的一个事件表里，不像select每次调用都要重复传入文件描述符集或事件集。epoll需要使用一个额外的文件描述符，来唯一标示内核中的这个事件表。文件描述符使用如下epoll_create函数创建：
```c
#include <sys/epoll.h>
int epoll_create(int size);
```

size参数其实并不起作用，只是提示作用。函数返回的文件描述符将用作其他所有的epoll系统调用的第一个参数，以指定要访问的内核事件表。

epoll_ctl函数用来操作内核事件表，原型如下：
```c
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event* event);
```

fd是要操作的文件描述符，即epoll_create的返回值，op参数指定了操作类型，类型有如下三种：
- EPOLL_CTL_ADD，往事件表中注册fd上的事件。
- EPOLL_CTL_MOD，修改fd上注册的事件。
- EPOLL_CTL_DEL，删除fd上注册的事件。

event成员用来描述事件类型。

epoll_ctl成功时返回0，失败时返回-1并设置errno。

epoll_wait函数在一段超时时间内等待一组文件描述符上的事件，原型如下：
```c
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```

该函数成功时返回就绪的文件描述符个数，失败时返回-1并设置errno。timeout参数指定超时的时间，当为-1时，将永远阻塞，直到某个文件描述符就绪，当timeout为0时，将立即返回。maxevents指定最多监听多少事件，必须大于0。epoll_wait函数如果检测到epfd对应的事件，就会将所有就绪的事件从内核事件表复制到第二个参数events指向的数组中。这个数组只输出epoll_wait检测到的就绪事件，相对select既用于传入注册事件，又用于输出内核检测到的事件，极大提高了索引就绪文件描述符的效率。

## ae事件库源码分析

ae事件库代码量不多，但是却支撑起了Redis的高吞吐量业务场景，作为一个IO框架库，它具有以下几个特点：
- 跨平台支持。支持Linux，Unix，Solaris，Windows。
- 线程安全。ae事件库单线程实现，避免了线程切换产生的问题。
- 基于Reactor模式实现。


### 源码组织结构
ae事件库源码由一下6个文件组成：
- ae.h：事件库头文件。该文件定义了事件的数据结构以及事件循环的容器，事件库用到的常量，还声明了事件库所有的函数原型。
- ae.c：该文件实现了ae事件库的整体框架，根据不同平台特点引入不同的IO复用实现，其主要是对ae.h中定义事件数据结构的操作。
- evport.c：event ports是Solaris平台IO复用机制的一种实现，代码内封装了这种IO复用机制的调用
- epoll.c：epoll是Linux平台IO复用机制的一种实现，代码内封装了epoll系列函数的调用
- kqueue.c：kqueue是Unix平台的IO复用机制的一种实现，代码内封装了kqueue系列函数的调用
- select.c：selecct是POSIX标准IO复用机制的基础实现，是所有平台都会支持的IO复用机制，代码内封装了select函数的调用

在这几个文件里，ae.c是整个框架的核心，负责事件的循环，处理就绪的文件描述符，还有添加删除关注的文件描述符，evport.c，epoll.c，kqueue.c，select.c这四个文件分别是四种IO复用机制系统调用的封装，它们内部实现的函数都是统一的调用方式，方便框架库根据系统，选择最优的IO复用机制。

### 源码分析

#### ae事件库数据结构介绍

首先我们先看下ae事件库核心数据结构，定义如下：
```c
/* 事件容器 */
typedef struct aeEventLoop {
    int maxfd;   /* 当前注册的最大文件描述符 */
    int setsize; /* 关注的文件描述符的数量 */
    long long timeEventNextId;
    time_t lastTime;     /* 用于检测系统时钟偏差 */
    aeFileEvent *events; /* 注册的事件 */
    aeFiredEvent *fired; /* 就绪的事件 */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
    aeBeforeSleepProc *aftersleep;
} aeEventLoop;
```

这个数据结构存储了存储了程序注册的所有文件描述符，以及事件信息，在ae事件库中，我们基本上是在操作这个数据结构。events是注册的所有的文件描述，fired存储的是就绪的文件描述符，下面来看下它们的结构：

```c
/* 普通文件事件结构 */
typedef struct aeFileEvent {
    int mask; /* AE_(READABLE|WRITABLE|BARRIER)等事件 */
    aeFileProc *rfileProc; 
    aeFileProc *wfileProc;
    void *clientData;
} aeFileEvent;
```

```c
/* 就绪的事件 */
typedef struct aeFiredEvent {
    int fd;
    int mask;
} aeFiredEvent;
```

aeFileEvent是注册的文件描述符对应的信息，包含关注的事件，以及绑定的事件处理器。但是在这个结构里没办法知道事件是否触发。aeFiredEvent结构体存储的是就绪事件，包含事件类型和文件描述符，怎么使用在后面讲。

#### 使用示例
下面我们看下Redis中使用的例子：

```c
int main(int argc, char **argv) {
    struct redisServer server; /* Server global state */
    server.el = aeCreateEventLoop(server.maxclients+CONFIG_FDSET_INCR);
    for (j = 0; j < server.ipfd_count; j++) {
        if (aeCreateFileEvent(server.el, server.ipfd[j], AE_READABLE,
            acceptTcpHandler,NULL) == AE_ERR)
            {
                serverPanic(
                    "Unrecoverable error creating server.ipfd file event.");
            }
    }
    aeMain(server.el);
    aeDeleteEventLoop(server.el);
    return 0;
}
```

程序首先调用aeCreateEventLoop函数创建事件循环容器，做一些初始化工作。然后调用aeCreateFileEvent函数创建文件描述符事件，在这里，注册了文件描述符，关注的事件，还绑定了具体事件处理器。aeCreateFileEvent函数如下：

```c
int aeCreateFileEvent(aeEventLoop *eventLoop, int fd, int mask,
        aeFileProc *proc, void *clientData)
{
    if (fd >= eventLoop->setsize) {
        errno = ERANGE;
        return AE_ERR;
    }
    aeFileEvent *fe = &eventLoop->events[fd];

    if (aeApiAddEvent(eventLoop, fd, mask) == -1)
        return AE_ERR;
    fe->mask |= mask;
    if (mask & AE_READABLE) fe->rfileProc = proc;
    if (mask & AE_WRITABLE) fe->wfileProc = proc;
    fe->clientData = clientData;
    if (fd > eventLoop->maxfd)
        eventLoop->maxfd = fd;
    return AE_OK;
}
```

代码里aeApiAddEvent函数，是不同的IO机制的添加事件的封装。

注册完事件，然后就调用aeMain函数开始事件循环。代码如下：
```c
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS|AE_CALL_AFTER_SLEEP);
    }
}
```

在事件循环里，调用多路复用api函数，返回就绪的事件，如果有就绪的事件，那就执行对应的读写操作，实现代码如下：
```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    numevents = aeApiPoll(eventLoop, tvp);
    for (j = 0; j < numevents; j++) {
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
        int mask = eventLoop->fired[j].mask;
        int fd = eventLoop->fired[j].fd;
        int fired = 0; 
        /* 正常都是先执行可读事件，然后执行可写事件，这样可以尽快响应请求，
         * 但是，某些情况可能需要先执行可写事件，然后执行可读事件，比如一些
         * 往硬盘同步文件，就需要在回复客户端之前执行 */
        int invert = fe->mask & AE_BARRIER;

        if (!invert && fe->mask & mask & AE_READABLE) {
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            fired++;
        }

        if (fe->mask & mask & AE_WRITABLE) {
            if (!fired || fe->wfileProc != fe->rfileProc) {
                fe->wfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
        }

        if (invert && fe->mask & mask & AE_READABLE) {
            if (!fired || fe->wfileProc != fe->rfileProc) {
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
                fired++;
            }
        }

        processed++;
    }
}
```

## 小结
ae事件库封装了不同平台IO复用机制的系统调用，然后在事件库中进行调用，虽然代码量小，但是支撑起了Redis这样的高并发请求特点。
以上，大概就是ae事件库的整个逻辑，篇幅有限，没有展开细说，有些内容待日后补充

