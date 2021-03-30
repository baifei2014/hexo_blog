---
layout: post
title: 关于go内存模型
date: 2020-02-18 21:16:07
tags: [golang, 内存模型, goroutine]
---

## 介绍

go内存模型指定在以下几种条件下，在一个goroutine中读取一个变量可以保证监听到不同goroutine对相同变量写操作产生的值。

## 建议

程序中多个goroutine在修改同时访问的数据时必须有序进行。

为了实现有序访问、保护数据，可以通过信道操作或语言原生的并发包比如 `sync` 和 `sync/atomic` 来实现。

## Synchronization

### Initialization

程序初始化运行在一个goroutine中，但是这个goroutine可能会创建其它的goroutine并发执行。

如果p包导入了q包，那q包init函数执行完成在p包所有代码之前。

main函数执行再所有的init函数执行完成之后。

### Goroutine creation

go语言启动一个新的goroutine在goroutine执行之前，也就是说goroutine不会立即执行。

举个例子，下面这个程序：

```go
var a string

func f() {
	print(a)
}

func hello() {
	a = "hello, world"
	go f()
}
```

调用hello将会在未来某个时间点输出“hello, world”。

### Goroutine destruction

goroutine的退出不保证会发生在任何事件之前。举个例子，下面这个程序：

```go
var a string

func hello() {
	go func() { a = "hello" }()
	print(a)
}
```

分配变量a没有遵守任何同步事件，所以不保证会被其他的goroutine监听到。事实上，激进的编译器可能会删除整个go语句。

如果goroutine中的变化想要被其它goroutine监听到，使用同步机制比如锁或channel通信去创建一个相对的执行顺序。

### Channel communication

channel通信是goroutine之间实现同步的主要方法。每次发送在指定的channel上并且匹配有一个相应的接收channel。

channel的发送发生在该channel相应接收完成之前。

这个程序：

```go
var c = make(chan int, 10)
var a string

func f() {
	a = "hello, world"
	c <- 0
}

func main() {
	go f()
	<-c
	print(a)
}
```

可以保证输出“hello, world”，写操作发生在c发送之前，发生在c上对应接收完成之前，发生在输出之前。

通道关闭发生在由于通道关闭而返回零值的接收之前。

前面的例子，替换 `c <- 0` 为 `close(c)` 有同样的保证效果。

无缓冲channel中接收在该channel发送完成之前。

下面这个程序：

```go
var c = make(chan int)
var a string

func f() {
	a = "hello, world"
	<-c
}

func main() {
	go f()
	c<-0
	print(a)
}
```

同样能够保证输出“hello, world”，写操作发生在接收执行之前，发生在发送之前之前，发生在输出之前。

如果channel是有缓冲的 (e.g., c = make(chan int, 1)) ，那么程序就不会保证输出“hello, world”

### Locks

sync包里实现了两种锁类型，分别是 `sync.Mutex` 和 `sync.RWMutex`。

对于 `sync.Mutex` 或 `sync.RWMutex` 类型的变量l来说，如果 `n<m` ，第n次调用l.Unlock()一定发生在第m次调用l.Lock()返回之前。

下面这个程序：

```go
var l sync.Mutex
var a string

func f() {
	a = "hello, world"
	l.Unlock()
}

func main() {
	l.Lock()
	go f()
	l.Lock()
	print(a)
}
```

保证会输出“hello, world”，因为第一次调用l.Unlock()发生在第二次调用l.Lock()返回之前。

对于 `sync.RLock` 或 `sync.RWMutex` 类型的变量l，第n次调用 `l.RLock` 发生在第n次调用 `l.Unlock` 之后，并且 `l.RUnlock` 发生在第n+1次调用 `l.Lock` 之前。
























