---
title: Redis中多线程的应用
tags: [Redis, 多线程]
date: 2019-12-30 22:28:57
---

说起Redis，熟悉的都知道它是一款单进程单线程的数据库。在保持实现简洁性的同时，最大程度的保证了其IO效率。那Redis只有单线程实现吗？今天就来介绍下Redis中多线程的应用。

## 多线程介绍

&nbsp;&nbsp;多线程对比单线程，可以理解为并行与串行的区别。在单线程中，任务是串行执行的，我们是可以预期的，就像一条单车道的跑道，后面的车永远不会超过前面的车。而在多线程中，就可以理解为有多条跑道，每辆车在某个时间，顺序是不定的，结果是不可预期的。多线程相比于单线程，能充分利用多核处理器的资源，但是也带来了复杂性，用不好，就会造成很多不可预期的问题。

&nbsp;&nbsp;在一些应用中，为了降低代码复杂性，更多的还是使用单线程模式开发程序，比如PHP程序，大多数都是单线程模式。在Redis中，主要功能也是单进程单线程模式实现的，这样的坏处是没法利用多核cpu的资源，但是好处是实现简单。官方推荐的做法是在多核机器上启动多个Redis实例，这样去利用多核cpu资源。

&nbsp;&nbsp;Redis虽然主要功能是单线程实现的，但是也实现了多线程的功能，主要做一些后台任务。比如说数据备份，如果都由一个线程去做，这些数据备份都很费时，可能会阻塞IO，导致请求不可用。所以Redis把数据备份还有关闭文件等分配给多线程去做。

## 实例代码分析

### 线程任务配置

&nbsp;&nbsp;Redis虽然计划以后会加入更多的多线程内容，但是当前版本中，只创建三个线程，去做三个任务。代码如下：

```c
/* Background job opcodes */
#define BIO_CLOSE_FILE    0 /* Deferred close(2) syscall. */
#define BIO_AOF_FSYNC     1 /* Deferred AOF fsync. */
#define BIO_LAZY_FREE     2 /* Deferred objects freeing. */
#define BIO_NUM_OPS       3
```

&nbsp;&nbsp;前面三个常量分别是关闭文件，AOF同步，还有释放对象。这三个任务都是比较费时的操作，所以由多线程去异步延时操作。`BIO_NUM_OPS`是线程的数量，也就是线程初始化时创建多少个线程。

### 线程初始化

&nbsp;&nbsp;在启动Redis服务时，会调用线程初始化函数，创建线程放入线程池。代码如下：

```c
static pthread_t bio_threads[BIO_NUM_OPS];
static pthread_mutex_t bio_mutex[BIO_NUM_OPS];
static list *bio_jobs[BIO_NUM_OPS];
/* 初始化后台系统，创建线程. */
void bioInit(void) {
    pthread_attr_t attr;
    pthread_t thread;
    size_t stacksize;
    int j;

    /* 初始化互斥锁还有队列 */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        pthread_mutex_init(&bio_mutex[j],NULL);
        bio_jobs[j] = listCreate();
    }

    /* 创建线程，把线程任务队列id做参数传入线程执行函数里. */
    for (j = 0; j < BIO_NUM_OPS; j++) {
        void *arg = (void*)(unsigned long) j;
        if (pthread_create(&thread,&attr,bioProcessBackgroundJobs,arg) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize Background Jobs.");
            exit(1);
        }
        bio_threads[j] = thread;
    }
}
```

&nbsp;&nbsp;在代码中，首先初始化了互斥锁，线程任务队列。然后创建了线程，并把线程任务id当参数传入执行函数里。这样，每个线程执行的任务就已经约定好了。

### 线程执行函数

&nbsp;&nbsp;在创建线程时，我们指定了线程的执行函数，那执行函数都会有什么操作呢，接着看代码：

```c
void *bioProcessBackgroundJobs(void *arg) {
    struct bio_job *job;
    unsigned long type = (unsigned long) arg;
    sigset_t sigset;

    /* 检查类型是否合法. */
    if (type >= BIO_NUM_OPS) {
        serverLog(LL_WARNING,
            "Warning: bio thread started with wrong type %lu",type);
        return NULL;
    }

    /* 设置线程可以在任何时候被杀死. */
    pthread_setcancelstate(PTHREAD_CANCEL_ENABLE, NULL);
    pthread_setcanceltype(PTHREAD_CANCEL_ASYNCHRONOUS, NULL);

    pthread_mutex_lock(&bio_mutex[type]);
    /* 阻塞信息保证只有主线程才能收到信号. */
    sigemptyset(&sigset);
    sigaddset(&sigset, SIGALRM);
    if (pthread_sigmask(SIG_BLOCK, &sigset, NULL))
        serverLog(LL_WARNING,
            "Warning: can't mask SIGALRM in bio.c thread: %s", strerror(errno));

    while(1) {
        listNode *ln;

        /* The loop always starts with the lock hold. */
        if (listLength(bio_jobs[type]) == 0) {
            pthread_cond_wait(&bio_newjob_cond[type],&bio_mutex[type]);
            continue;
        }
        /* 从队列中弹出任务. */
        ln = listFirst(bio_jobs[type]);
        job = ln->value;
        /* 解锁，因为现在已经知道只有一个任务在处理.*/
        pthread_mutex_unlock(&bio_mutex[type]);

        /* 根据任务类型执行任务. */
        if (type == BIO_CLOSE_FILE) {
            close((long)job->arg1);
        } else if (type == BIO_AOF_FSYNC) {
            redis_fsync((long)job->arg1);
        } else if (type == BIO_LAZY_FREE) {
            /* What we free changes depending on what arguments are set:
             * arg1 -> free the object at pointer.
             * arg2 & arg3 -> free two dictionaries (a Redis DB).
             * only arg3 -> free the skiplist. */
            if (job->arg1)
                lazyfreeFreeObjectFromBioThread(job->arg1);
            else if (job->arg2 && job->arg3)
                lazyfreeFreeDatabaseFromBioThread(job->arg2,job->arg3);
            else if (job->arg3)
                lazyfreeFreeSlotsMapFromBioThread(job->arg3);
        } else {
            serverPanic("Wrong job type in bioProcessBackgroundJobs().");
        }
        zfree(job);

        /* Lock again before reiterating the loop, if there are no longer
         * jobs to process we'll block again in pthread_cond_wait(). */
        pthread_mutex_lock(&bio_mutex[type]);
        listDelNode(bio_jobs[type],ln);
        bio_pending[type]--;

        /* Unblock threads blocked on bioWaitStepOfType() if any. */
        pthread_cond_broadcast(&bio_step_cond[type]);
    }
}
```

&nbsp;&nbsp;在线程执行函数里，首先会判断当前任务队列是否为空，如果为空，阻塞等待直到有队列中有新任务。如果队列中有任务，取出队列中第一个任务，然后根据任务类型执行不同的内容。当执行完成后，在队列中删除这个任务，这里我们还可以看出，这是具有ack机制的任务队列。

### 添加任务到队列

&nbsp;&nbsp;前面说了线程初始化还有线程执行函数，下面我们就要看下如何添加任务到队列了。还是直接看代码：

```c
void bioCreateBackgroundJob(int type, void *arg1, void *arg2, void *arg3) {
    struct bio_job *job = zmalloc(sizeof(*job));

    job->time = time(NULL);
    job->arg1 = arg1;
    job->arg2 = arg2;
    job->arg3 = arg3;
    pthread_mutex_lock(&bio_mutex[type]);
    listAddNodeTail(bio_jobs[type],job);
    bio_pending[type]++;
    pthread_mutex_unlock(&bio_mutex[type]);
}
```

&nbsp;&nbsp;创建后台任务函数支持四个参数，第一个参数是任务类型，也就是我们前面说的那三种任务类型，后面三个都是任务参数。这里，由于是多线程的，所以，添加任务到队列时使用了互斥锁。申请锁后，将任务添加到队列末尾，然后解锁。



## 小结

&nbsp;&nbsp;这篇文章大致介绍了线程初始化，创建队列任务，任务消费者(线程执行函数)的逻辑。可以看出，里面涉及到了多线程编程的基础知识，包括对共享变量的访问加锁等逻辑。通过多线程，原本耗时的任务也转到后台异步执行了。










