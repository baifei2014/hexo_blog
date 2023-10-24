---
layout: blog
title: golang context理解
date: 2023-04-18 17:17:26
tags: [golang, context, timeout]
---

## 前言
在使用 golang 开发应用时，我们会看到项目里函数大量的Context传参。可能初次接触时会困惑，为啥要对这个对象传来传去，Context 究竟是什么，有啥作用。下面就来聊一聊 Context 的实现及作用。

## Context 定义

Context 是应用执行的上下文信息，在 golang 里有标准库实现，最初的Context是于2014年放在官方实验性的golang.org/x/net/context这个库里，并于2016年在Go 1.7版本加入到官方标准库context package。在程序执行时，通过传递 Context，可以让主协程和子协程拥有相同的上下文信息。实现协程通信，数据传递。


## Context 原理
goroutine 的创建和调用关系往往是有着层级关系的，顶部的 Goroutine 应有办法主动关闭其创建的子 goroutine 的执行。为了实现这种关系，Context 结构也应该像一棵树，叶子节点须总是由根节点衍生出来的。

第一个创建 Context 的 goroutine 被称为 root 节点：root 节点负责创建一个实现 Context 接口的具体对象，并将该对象作为参数传递至新拉起的 goroutine 作为其上下文。下游 goroutine 继续封装该对象并以此类推向下传递。

Context 可以安全的被多个 goroutine 使用。开发者可以把一个 Context 传递给任意多个 goroutine 然后 cancel 这个 context 的时候就能够通知到所有的 goroutine。

Context 接口定义如下:

```golang
type Context interface {
	// Deadline returns the time when work done on behalf of this context
	// should be canceled. Deadline returns ok==false when no deadline is
	// set. Successive calls to Deadline return the same results.
	Deadline() (deadline time.Time, ok bool)

	// Done returns a channel that's closed when work done on behalf of this
	// context should be canceled. Done may return nil if this context can
	// never be canceled. Successive calls to Done return the same value.
	// The close of the Done channel may happen asynchronously,
	// after the cancel function returns.
	Done() <-chan struct{}

	// If Done is not yet closed, Err returns nil.
	// If Done is closed, Err returns a non-nil error explaining why:
	// Canceled if the context was canceled
	// or DeadlineExceeded if the context's deadline passed.
	// After Err returns a non-nil error, successive calls to Err return the same error.
	Err() error

	// Value returns the value associated with this context for key, or nil
	// if no value is associated with key. Successive calls to Value with
	// the same key returns the same result.
	Value(key any) any
}
```

- Deadline 会返回这个 context 何时会被取消的具体时间。 如果没有设置超时时间，返回 ok 是 false，否则是 true。

- Done 方法在 Context 被取消或超时时返回一个 close 的 channel，close 的 channel 可以作为广播通知，告诉给 context 相关的函数要停止当前工作然后返回。

- Err 方法返回 context 为什么被取消。

- Value 返回 context 相关的数据

## 取消信号传递

context 对象的 WithCancel 函数能够从父节点再派生出一个子 context 和一个取消函数，执行这个取消函数，可以取消这个派生出的context 以及其子 context，如果 goroutine 里有传递这个 context，也会同步收到信号，我们可以用如下代码定义一个具有超时的 context 对象：

```golang
ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
defer cancel()
```

WithTimeout 函数返回派生出的子 context以及一个取消函数，取消函数在结束时会自动执行，取消函数代码如下：

```golang
func (c *cancelCtx) cancel(removeFromParent bool, err error) {
	if err == nil {
		panic("context: internal error: missing cancel error")
	}
	c.mu.Lock()
	if c.err != nil {
		c.mu.Unlock()
		return // already canceled
	}
	c.err = err
	d, _ := c.done.Load().(chan struct{})
	if d == nil {
		c.done.Store(closedchan)
	} else {
		close(d)
	}
	for child := range c.children {
		// NOTE: acquiring the child's lock while holding parent's lock.
		child.cancel(false, err)
	}
	c.children = nil
	c.mu.Unlock()

	if removeFromParent {
		removeChild(c.Context, c)
	}
}
```

在取消函数里，会递归的取消所有子孙 context。如下代码展示了在协程里监听 context 的取消信号

```golang
func doSomething(ctx context.Context) {

	// 模拟耗时操作
	for i := 0; i < 100; i++ {
		select {
		case <-ctx.Done():
			fmt.Println("收到取消信号，停止执行")
			return
		default:
			fmt.Println("执行中...")
			time.Sleep(1 * time.Second)
		}
	}

	fmt.Println("任务执行完成")
}

func main() {
	// 创建一个带有超时的context，超时时间设置为3秒
	ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
	defer cancel()

	// 启动一个goroutine执行任务
	go doSomething(ctx)

	// 主goroutine等待一段时间
	time.Sleep(20 * time.Second)
	fmt.Println("主程序执行完毕")
}
```

在协程里，监听 context 的通道消息，当 context 超过3秒取消时，子协程会收到信号后退出。

## 总结

context通过派生层级结构，可以实现取消信号传递，并发安全的移除关闭子 context，及时回收资源。






