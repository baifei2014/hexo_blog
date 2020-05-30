---
title: redis6.0多线程IO
tags: [redis, 多线程, IO]
date: 2020-05-29 21:52:51
---

## 前言

这个月redis有了一次大版本更新，最新版本已经到了6.0，这其中的有很多值得关注的更新，包括重新设计客户端侧缓存，RDB文件复制完不再使用立即删除，更好用的权限控制，还有多线程IO等等。这篇文章重点聊聊多线程IO。


## redis是多线程的吗

我在之前的文章里也写过redis多线程信息，这次再简单说下redis多线程。很长时间以来，单线程redis深入人心。并且redis作者也说，redis作者也说，redis的瓶颈不在cpu，而是内存和网络。的确，redis主要服务都是通过单线程完成的，但是redis从4.0之后版本已经在部分功能里开始使用了多线程了。redis作者之前也曾提到，redis计划更广泛的使用多线程，真香定理无处不在。这次更新的版本里，在网络IO里也开始使用多线程了，用于和客户端通信，当然，使用多线程是可选项，也可以选择不适用多线程IO。所以，以后在讨论redis，我们准确的描述应该就是redis是一款多线程应用程序。

## 多线程vs多线程

客观来讲，单线程程序实现相对于多线程来讲，较为简单，不用去考虑各种线程安全的操作，因为都是有序执行的，也不用去考虑死锁。但也是缺点，没法有效利用多cpu机器的计算资源，一个redis实例只能使用到单核的算力，对于现在动辄16核、32核的机器来说，实在是有点浪费资源。所以之前推荐的方案里，就是在一台机器上启动多个redis实例。这种做法虽然可行，但是提高了运维成本，本来是一个实例，现在要部署这么多实例。

对于Redis来讲，现在一个直接的优化方向就是网络IO，redis本身数据结构相对简单，不管是数据的修改还是查询，都不会耗费太多时间。网络IO恰恰是一个可以优化的点，原先单线程网络IO，只能利用一个核。现在使用多线程进行IO，可以提高网络IO效率，支持更大并发量。

## 网络IO多线程实现

### 线程初始化

redis启动的时候，会初始化IO线程，初始化IO线程的代码如下：

```c
/* Initialize the data structures needed for threaded I/O. */
void initThreadedIO(void) {
    io_threads_active = 0; /* We start with threads not active. */

    /* Don't spawn any thread if the user selected a single thread:
     * we'll handle I/O directly from the main thread. */
    if (server.io_threads_num == 1) return;

    if (server.io_threads_num > IO_THREADS_MAX_NUM) {
        serverLog(LL_WARNING,"Fatal: too many I/O threads configured. "
                             "The maximum number is %d.", IO_THREADS_MAX_NUM);
        exit(1);
    }

    /* Spawn and initialize the I/O threads. */
    for (int i = 0; i < server.io_threads_num; i++) {
        /* Things we do for all the threads including the main thread. */
        io_threads_list[i] = listCreate();
        if (i == 0) continue; /* Thread 0 is the main thread. */

        /* Things we do only for the additional threads. */
        pthread_t tid;
        pthread_mutex_init(&io_threads_mutex[i],NULL);
        io_threads_pending[i] = 0;
        pthread_mutex_lock(&io_threads_mutex[i]); /* Thread will be stopped. */
        if (pthread_create(&tid,NULL,IOThreadMain,(void*)(long)i) != 0) {
            serverLog(LL_WARNING,"Fatal: Can't initialize IO thread.");
            exit(1);
        }
        io_threads[i] = tid;
    }
}
```

由代码可以看出，创建线程的数量是根据 `server.io_threads_num` 决定的。每个线程会分配一个双向链表队列，要处理的客户端连接会放在这个队列容器里，按顺序执行。线程还会绑定一个任务处理函数 `IOThreadMain` ，这个函数就是线程要执行的工作内容。线程处理函数内容也不多，简单来讲就是从队列中取出客户端连接，向连接中写入数据。下面我们看下这个函数的实现，代码如下：

```c
void *IOThreadMain(void *myid) {
    /* The ID is the thread number (from 0 to server.iothreads_num-1), and is
     * used by the thread to just manipulate a single sub-array of clients. */
    long id = (unsigned long)myid;
    char thdname[16];

    snprintf(thdname, sizeof(thdname), "io_thd_%ld", id);
    redis_set_thread_title(thdname);
    redisSetCpuAffinity(server.server_cpulist);

    while(1) {
        /* Wait for start */
        for (int j = 0; j < 1000000; j++) {
            if (io_threads_pending[id] != 0) break;
        }

        /* Give the main thread a chance to stop this thread. */
        if (io_threads_pending[id] == 0) {
            pthread_mutex_lock(&io_threads_mutex[id]);
            pthread_mutex_unlock(&io_threads_mutex[id]);
            continue;
        }

        serverAssert(io_threads_pending[id] != 0);

        if (tio_debug) printf("[%ld] %d to handle\n", id, (int)listLength(io_threads_list[id]));

        /* Process: note that the main thread will never touch our list
         * before we drop the pending count to 0. */
        listIter li;
        listNode *ln;
        listRewind(io_threads_list[id],&li);
        while((ln = listNext(&li))) {
            client *c = listNodeValue(ln);
            if (io_threads_op == IO_THREADS_OP_WRITE) {
                writeToClient(c,0);
            } else if (io_threads_op == IO_THREADS_OP_READ) {
                readQueryFromClient(c->conn);
            } else {
                serverPanic("io_threads_op value is unknown");
            }
        }
        listEmpty(io_threads_list[id]);
        io_threads_pending[id] = 0;

        if (tio_debug) printf("[%ld] Done\n", id);
    }
}
```

处理函数首先设置线程的标题，然后指定线程运行的cpu，下面就是while循环进入阻塞模式，不停的从自己的任务队列中取出客户端连接，判断是读还是写，执行对应的操作，这样，原先由一个线程执行的任务，现在就可以由多个线程执行了，那这里就要好奇了，任务是怎么分配的呢？我们往下看。

### 任务添加

队列任务添加的逻辑在 `handleClientsWithPendingWritesUsingThreads` 函数里，这个函数在beforeSleep里被调用，beforeSleep函数会绑定在ae事件容器上，因为ae事件库每次都要使用poll机制获取就绪事件。这段时间会阻塞执行，相当于主线程进入睡眠，所以每次ae事件库调用poll获取就绪事件之前，都要调用beforeSleep函数处理已经就绪的客户端请求。

上面说了 `handleClientsWithPendingWritesUsingThreads` 调用的时机，接下来我们看下这个函数的具体实现，代码如下：

```c
int handleClientsWithPendingWritesUsingThreads(void) {
    /* Distribute the clients across N different lists. */
    listIter li;
    listNode *ln;
    listRewind(server.clients_pending_write,&li);
    int item_id = 0;
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        c->flags &= ~CLIENT_PENDING_WRITE;
        int target_id = item_id % server.io_threads_num;
        listAddNodeTail(io_threads_list[target_id],c);
        item_id++;
    }

    /* Give the start condition to the waiting threads, by setting the
     * start condition atomic var. */
    io_threads_op = IO_THREADS_OP_WRITE;
    for (int j = 1; j < server.io_threads_num; j++) {
        int count = listLength(io_threads_list[j]);
        io_threads_pending[j] = count;
    }

    /* Also use the main thread to process a slice of clients. */
    listRewind(io_threads_list[0],&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);
        writeToClient(c,0);
    }
    listEmpty(io_threads_list[0]);

    /* Wait for all the other threads to end their work. */
    while(1) {
        unsigned long pending = 0;
        for (int j = 1; j < server.io_threads_num; j++)
            pending += io_threads_pending[j];
        if (pending == 0) break;
    }
    if (tio_debug) printf("I/O WRITE All threads finshed\n");

    /* Run the list of clients again to install the write handler where
     * needed. */
    listRewind(server.clients_pending_write,&li);
    while((ln = listNext(&li))) {
        client *c = listNodeValue(ln);

        /* Install the write handler if there are pending writes in some
         * of the clients. */
        if (clientHasPendingReplies(c) &&
                connSetWriteHandler(c->conn, sendReplyToClient) == AE_ERR)
        {
            freeClientAsync(c);
        }
    }
    listEmpty(server.clients_pending_write);
    return processed;
}
```

从代码中可以看出，主线程从当前待处理的连接中取模平均的分配到每个线程任务队列中。分配完后，主线程会执行第一个线程中的任务，然后再重复检查当前server上还有没有要处理的连接请求。最后，也就意味着当前可处理请求都已经处理完了，返回处理的个数，执行完成，这也就是任务分配的逻辑。


## 小结

这里的网络io多线程处理任务也是通过共享变量的方式实现的，在一段生产者的代码中往队列容器中写入数据，在消费者中也就是队列执行函数中处理分配的任务。

对于线程数的设置，官方的建议是不要设置的太大，熟悉多线程编程的应该知道，线程超过内核数，会带来上下文切换的开销。最后反而不能利用多核cpu的资源。












