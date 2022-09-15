---
layout: blog
title: go网络模型源码解析
date: 2022-09-14 10:05:06
tags: [Go, net, netpoll, epoll, goroutine]
---

## 前言

基于go语言，开发者通过几行代码就可以创建一个server服务。在这个调用的背后，go基于io多路复用和goroutine schedule集成了一个简洁高性能的网络模型network poller。下面将一步步介绍网络编程基础，然后基于linux平台通过分析源码来了解其实现细节。

## Linux网络编程

在Linux中，一切皆文件，网络编程说白了就是对连接socket文件的I/O（读/写）操作。操作系统的I/O操作是内核空间和用户空间的数据操作。如下图所示：

{% asset_img net_1.jpg 示例图1 %}

由图我们可以看出，网络I/O操作一般会有以下过程：

- 从网卡读取数据 -> 写入内核缓冲区/内核缓冲区复制数据 -> 写入网卡
- 从内核缓冲区复制数据 -> 用户空间读/用户空间复制数据 -> 写入内核缓冲区

Linux服务器编程时，以tcp为例，通过accept接受一个连接。获取socket连接的文件描述符后，通过recv读取请求内容，通过send写入响应。

系统调用Api如下：

```c
#include <sys/types.h>
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
ssize_t recv(int sockfd, void *buf, size_t len, int flags);
ssize_t send(int sockfd, const void *buf, size_t len, int flags);
```

accept调用是从监听队列中接受一个连接，sockfd是监听的socket。成功时返回一个新的连接socket，唯一标识了这个连接，用于后续通信。

recv读取sockfd上的数据，buf和len指定了读缓冲区的位置和大小，flags参数是额外控制，可以阻塞或非阻塞读等等。

send是写入数据，buf和len指定了写缓冲区的位置和大小。


## I/O模型

Unix 下有五种 I/O 模型:

- 阻塞式 I/O
- 非阻塞式 I/O
- I/O 复用(select、poll、epoll)
- 信号驱动式 I/O(SIGIO)
- 异步 I/O(AIO)

### 阻塞I/O & 非阻塞I/O

阻塞I/O会阻塞当前用户进程，比如当程序向内核发起系统调用read()时，进程此时阻塞，等待内核读取磁盘数据到内核缓冲区。非阻塞I/O也就意味着不会阻塞当前进程，不管读没读到，立即返回。

### I/O 复用

I/O复用是指一个线程能同时监听多个文件描述符，直到监听的文件描述符有读写就绪事件触发。I/O复用本身是阻塞等待的，要实现并发，我们还需要依赖多进程/多线程等方法。

Linux中实现I/O复用的系统调用有select、poll和epoll。

#### select

select系统调用如下：

```c
#include <sys/select.h>
int select(int nfds, fd_set* readfds, fd_set* writefds, fd_set* exceptfds, struct timeval* timeout);
```

readfds、writefds、exceptfds分别对应了监听的不同事件的文件描述符集合。调用返回时，内核修改对应就绪的文件描述符来通知应用程序。返回的值指的是就绪的文件描述符数量。

select系统调用支持广泛，但是有以下缺点：

- 允许监听的最大文件描述符有限制，即最大1024，虽然用户可以修改，但是可能导致未知后果
- 每次调用都需要重复把文件描述符集合从用户态拷贝到内核态，这个开销在监听的文件描述符很多时会很大
- 检测就绪事件通过轮询方式扫描整个集合，时间复杂度O(n)，模式低效

#### poll

poll系统调用和select相似，解决了监听数量的限制，其它方面并没有明显提升。

#### epoll

epoll是Linux特有的I/O复用函数。和select、poll有很大区别。epoll把用户要监听的文件描述符放到内核事件表中，不需要每次重复传入，使用一个额外的描述符来表示这个事件表，可以动态的增加、修改、移除注册事件。epoll通过一组函数而不是一个函数实现，系统调用如下：

```c
#include <sys/epoll.h>
int epoll_create(int size);
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
int epoll_wait(int epfd, struct epoll_event* events, int maxevents, int timeout);
```

epoll_create创建一个epoll实例返回文件描述符，通过操作这个返回的文件描述符来操作事件表。
epoll_ctl就是操作事件表，可以添加，修改，删除注册事件。
epoll_wait用于在一端超时时间内等待一组文件描述符上的事件。如果检测到事件，将所有就绪事件从内核事件表复制到第二个参数数组里，并返回就绪的文件描述符个数。

相对于select和poll，epoll无需轮询整个集合检测就绪事件，通过回调方式在文件描述符就绪时由内核将就绪事件移入就绪队列，非常高效。

### 信号驱动I/O & 异步I/O

信号驱动I/O应用进程使用sigaction系统调用，内核立即返回，应用进程可以继续执行，也就是说等待数据阶段应用进程是非阻塞的。内核在数据到达时向应用进程发送 SIGIO 信号，应用进程收到之后在信号处理程序中调用 recvfrom 将数据从内核复制到应用进程中。

异步I/O，即I/O操作不会引起进程阻塞。发起aio_read请求后，内核会直接返回。等数据就绪，发送信号到进程处理数据。进程不需要再次发起读取数据的系统调用，因为内核会把数据复制到用户空间再通知进程处理，整个过程不存在任何阻塞。

## go源码解析

Go提供了功能完备的标准网络库：net包，net包的实现相当之全面，http\tcp\udp均有实现且对用户提供了简单友好的使用接口。

go拉起一个http server如下：

```golang
func main() {
	http.HandleFunc("/hello", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, "Hello, playground")
	})

	log.Println("Starting server...")
	l, err := net.Listen("tcp", "localhost:8080")
	if err != nil {
		log.Fatal(err)
	}
	go func() {
		log.Fatal(http.Serve(l, nil))
	}()
}
```

### Listen

```golang
func (lc *ListenConfig) Listen(ctx context.Context, network, address string) (Listener, error) {
	addrs, err := DefaultResolver.resolveAddrList(ctx, "listen", network, address, nil)
	// ...
	switch la := la.(type) {
	case *TCPAddr:
		l, err = sl.listenTCP(ctx, la)
	case *UnixAddr:
		l, err = sl.listenUnix(ctx, la)
	// ...
	}
	// ...
	return l, nil
}
```

上面代码主要两个作用，先是解析协议名称和地址取得Internet协议族地址列表。然后从地址列表中取得满足条件的地址进行实际监听操作, 具体根据传入的协议族来确定。


顺着listenTCP往下看，调用链路是listenTCP->internetSocket->socket，socket函数定义如下：

```golang
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr, ctrlFn func(string, string, syscall.RawConn) error) (fd *netFD, err error) {
	s, err := sysSocket(family, sotype, proto)
	// ...
	if laddr != nil && raddr == nil {
		switch sotype {
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
			if err := fd.listenStream(laddr, listenerBacklog(), ctrlFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		case syscall.SOCK_DGRAM:
			if err := fd.listenDatagram(laddr, ctrlFn); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
	}
	// ...
	return fd, nil
}
```

这一步有个重要操作就是创建套接字，也就是sysSocket调用。我们详细看下实现：

```golang
func sysSocket(family, sotype, proto int) (int, error) {
	// ...
	s, err := socketFunc(family, sotype, proto)
	// ...
	if err = syscall.SetNonblock(s, true); err != nil {
		poll.CloseFunc(s)
		return -1, os.NewSyscallError("setnonblock", err)
	}
	return s, nil
}
```

代码里首先走系统调用（Linux系统Api也就是socket调用）创建一个socket，然后通过I/O函数（fcntl）设置这个socket文件描述符为非阻塞的。最后返回这个socket。

然后接着回到上一步调用，创建完socket后，继续调用listenStream监听socket。

```golang
func (fd *netFD) listenStream(laddr sockaddr, backlog int, ctrlFn func(string, string, syscall.RawConn) error) error {
	// ... 
	if err = syscall.Bind(fd.pfd.Sysfd, lsa); err != nil {
		return os.NewSyscallError("bind", err)
	}
	if err = listenFunc(fd.pfd.Sysfd, backlog); err != nil {
		return os.NewSyscallError("listen", err)
	}
	if err = fd.init(); err != nil {
		return err
	}
	// ...
	return nil
}
```

上面创建完socket，我们指定了地址族，但是没有指定使用该地址族中哪个地址。所以这里走系统调用Bind给socket命名，也就是将socket与socket地址绑定。只有命名后客户端才能连接到。

命名完后，执行系统调用Listen创建一个监听队列存放待处理的客户连接。backlog是提示内核监听队列的最大长度，如果超长后新连接请求会被放弃。执行完这两个系统调用之后，这个socket其实已经可以开始接受客户端连接了。但是前面也提到过，如果用accept接受连接不采取额外处理，程序就只能依次处理串行工作。

为了实现并发，go会怎么做，接着往下看，fd.init调用可以说在做最重要的事情，完成网络事件库的初始化并将socket注册进事件表。调用链路init->Init->init：

```golang
// internal/poll/fd_poll_runtime.go
func (pd *pollDesc) init(fd *FD) error {
	serverInit.Do(runtime_pollServerInit)
	ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
	if errno != 0 {
		return errnoErr(syscall.Errno(errno))
	}
	pd.runtimeCtx = ctx
	return nil
}
```

runtime_pollServerInit会调用netpoll网络事件库创建一个netpoller，调用链路是poll_runtime_pollServerInit->netpollGenericInit->netpollinit，具体实现代码如下：

```golang
func netpollinit() {
	epfd = epollcreate1(_EPOLL_CLOEXEC)
	if epfd < 0 {
		epfd = epollcreate(1024)
		if epfd < 0 {
			println("runtime: epollcreate failed with", -epfd)
			throw("runtime: netpollinit failed")
		}
		closeonexec(epfd)
	}
}
```

走系统调用创建了一个内核事件表。


回到上一层代码，runtime_pollOpen是调用netpoll里netpollopen注册事件，代码如下：

```golang
func netpollopen(fd uintptr, pd *pollDesc) int32 {
	var ev epollevent
	ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
	*(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
	return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

这里是往内核事件表添加监听的文件描述符，这里可以看出了监听了socket的可读，可写，连接关闭事件，以及设置了epoll的工作模式为ET模式，也就是边沿触发。

epoll对文件描述符的操作有两种模式，LT电平触发和ET边沿触发。LT是默认的工作模式，采用LT时，事件如果检测到通知应用程序没被处理，会重复通知，直到事件被处理。而ET工作模式要求事件通知应用程序后，必须立即处理，后续不再通知这个事件。ET降低了事件重复触发次数。是一种高效的工作模式。

回到代码，这里在事件表注册上了socket的监听，Listen工作已经做完了。

### Accept

http调用Serve接受连接请求，代码如下：

```golang
func (srv *Server) Serve(l net.Listener) error {
	// ...
	for {
		rw, err := l.Accept()
		// ...
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks) // before Serve can return
		go c.serve(connCtx)
	}
}
```

可以看出，Serve入参是上一步Listen操作创建的Listener句柄来接受连接。然后为每个连接创建一个goroutine处理请求响应，主go程继续阻塞等待。底层的系统调用是如何处理呢，接着往下看：


```golang
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
	// ...
	if err := fd.pd.prepareRead(fd.isFile); err != nil {
		return -1, nil, "", err
	}
	for {
		s, rsa, errcall, err := accept(fd.Sysfd)
		// ...
		case syscall.EAGAIN:
			if fd.pd.pollable() {
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		// ...
		}
		return -1, nil, errcall, err
	}
}
```

上面代码中，prepareRead首先检查socket是否可读，然后进入for循环等待连接。for循环里走系统调用接受连接，因为前面设置socket是非阻塞的，所以如果当前没有连接，调用waitRead等待socket读就绪。

#### runtime_pollWait

waitRead调用其实还是走到了netpoll网络事件库，链路是waitRead -> runtime_pollWait -> poll_runtime_pollWait，poll_runtime_pollWait代码如下：

```golang
func poll_runtime_pollWait(pd *pollDesc, mode int) int {
	errcode := netpollcheckerr(pd, int32(mode))
	if errcode != pollNoError {
		return errcode
	}
	// As for now only Solaris, illumos, and AIX use level-triggered IO.
	if GOOS == "solaris" || GOOS == "illumos" || GOOS == "aix" {
		netpollarm(pd, mode)
	}
	for !netpollblock(pd, int32(mode), false) {
		errcode = netpollcheckerr(pd, int32(mode))
		if errcode != pollNoError {
			return errcode
		}
		// Can happen if timeout has fired and unblocked us,
		// but before we had a chance to run, timeout has been reset.
		// Pretend it has not happened and retry.
	}
	return pollNoError
}

func netpollblock(pd *pollDesc, mode int32, waitio bool) bool {
	gpp := &pd.rg
	if mode == 'w' {
		gpp = &pd.wg
	}

	// set the gpp semaphore to pdWait
	for {
		// Consume notification if already ready.
		if gpp.CompareAndSwap(pdReady, 0) {
			return true
		}
		if gpp.CompareAndSwap(0, pdWait) {
			break
		}

		// Double check that this isn't corrupt; otherwise we'd loop
		// forever.
		if v := gpp.Load(); v != pdReady && v != 0 {
			throw("runtime: double wait")
		}
	}

	// need to recheck error states after setting gpp to pdWait
	// this is necessary because runtime_pollUnblock/runtime_pollSetDeadline/deadlineimpl
	// do the opposite: store to closing/rd/wd, publishInfo, load of rg/wg
	if waitio || netpollcheckerr(pd, mode) == pollNoError {
		gopark(netpollblockcommit, unsafe.Pointer(gpp), waitReasonIOWait, traceEvGoBlockNet, 5)
	}
	// be careful to not lose concurrent pdReady notification
	old := gpp.Swap(0)
	if old > pdWait {
		throw("runtime: corrupted polldesc")
	}
	return old == pdReady
}

func gopark(unlockf func(*g, unsafe.Pointer) bool, lock unsafe.Pointer, reason waitReason, traceEv byte, traceskip int) {
	if reason != waitReasonSleep {
		checkTimeouts() // timeouts may expire while two goroutines keep the scheduler busy
	}
	mp := acquirem()
	gp := mp.curg
	status := readgstatus(gp)
	if status != _Grunning && status != _Gscanrunning {
		throw("gopark: bad g status")
	}
	mp.waitlock = lock
	mp.waitunlockf = unlockf
	gp.waitreason = reason
	mp.waittraceev = traceEv
	mp.waittraceskip = traceskip
	releasem(mp)
	// can't do anything that might move the G between Ms here.
	mcall(park_m)
}

func park_m(gp *g) {
	_g_ := getg()

	if trace.enabled {
		traceGoPark(_g_.m.waittraceev, _g_.m.waittraceskip)
	}

	casgstatus(gp, _Grunning, _Gwaiting)
	dropg()

	if fn := _g_.m.waitunlockf; fn != nil {
		ok := fn(gp, _g_.m.waitlock)
		_g_.m.waitunlockf = nil
		_g_.m.waitlock = nil
		if !ok {
			if trace.enabled {
				traceGoUnpark(gp, 2)
			}
			casgstatus(gp, _Gwaiting, _Grunnable)
			execute(gp, true) // Schedule it back, never returns.
		}
	}
	schedule()
}
```

netpoll不会hang住go程的执行，判断没有就绪事件，会将go程由_Grunning转为_Gwaiting，然后继续下一个go程的调度。

另外主go程不断的调用findrunnable，也就是底层调用netpoll寻找就绪事件，执行可运行的goroutine。

## 对比PHP

通过前面的分析，可以看出对比php来说，go的网络I/O完全实现了自主控制调度。基于netpoll事件控制和runtime的GMP调度，实现了简洁高效的网络库。

PHP-FPM模型如下图所示：

{% asset_img net_2.jpg 示例图2 %}

go网络模型如下图所示：

{% asset_img net_3.jpg 示例图3 %}

## 总结

Go的网络编程模式及其简洁高效，但是也有其缺点。go为每个请求创建goroutine，虽然创建goroutine不像fork进程那样成本高，但在海量连接场景下也会过度消耗资源而影响调度。目前Linux平台上主流的高性能网络库/框架中，大都采用Reactor模式。Redis也是类似Reactor模式的网络模型，只是它完全由一个线程处理请求响应。当然，大多数业务场景下没有这么苛刻的性能要求，最好的使用方式还是按照go最佳实践来。























