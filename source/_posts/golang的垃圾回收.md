---
layout: blog
title: golang的垃圾回收
date: 2023-07-26 14:32:24
tags: [golang, gc]
---

## 介绍


```golang
// memoryLimitHeapGoal returns a heap goal derived from memoryLimit.
func (c *gcControllerState) memoryLimitHeapGoal() uint64 {
	// Start by pulling out some values we'll need. Be careful about overflow.
	var heapFree, heapAlloc, mappedReady uint64
	for {
		heapFree = c.heapFree.load()                         // 空闲且未清理的内存。
		heapAlloc = c.totalAlloc.Load() - c.totalFree.Load() // 正在使用的堆对象字节。
		mappedReady = c.mappedReady.Load()                   // 未释放的映射内存总量。
		if heapFree+heapAlloc <= mappedReady {
			break
		}
		// 未释放的映射内存总量不可能超过堆内存，
		// 但由于这些统计数据是独立更新的，
		// 因此我们可能会观察到仅包括某些值的部分更新。 我们似乎打破了这个不变量。 
		// 但是这种情况必然是暂时的，直接重试。 如果持续存在计算错误，我们将陷入死锁。
	}

	// Below we compute a goal from memoryLimit. There are a few things to be aware of.
	// Firstly, the memoryLimit does not easily compare to the heap goal: the former
	// is total mapped memory by the runtime that hasn't been released, while the latter is
	// only heap object memory. Intuitively, the way we convert from one to the other is to
	// subtract everything from memoryLimit that both contributes to the memory limit (so,
	// ignore scavenged memory) and doesn't contain heap objects. This isn't quite what
	// lines up with reality, but it's a good starting point.
	//
	// In practice this computation looks like the following:
	//
	//    memoryLimit - ((mappedReady - heapFree - heapAlloc) + max(mappedReady - memoryLimit, 0)) - memoryLimitHeapGoalHeadroom
	//                    ^1                                    ^2                                   ^3
	//
	// Let's break this down.
	//
	// The first term (marker 1) is everything that contributes to the memory limit and isn't
	// or couldn't become heap objects. It represents, broadly speaking, non-heap overheads.
	// One oddity you may have noticed is that we also subtract out heapFree, i.e. unscavenged
	// memory that may contain heap objects in the future.
	//
	// Let's take a step back. In an ideal world, this term would look something like just
	// the heap goal. That is, we "reserve" enough space for the heap to grow to the heap
	// goal, and subtract out everything else. This is of course impossible; the definition
	// is circular! However, this impossible definition contains a key insight: the amount
	// we're *going* to use matters just as much as whatever we're currently using.
	//
	// Consider if the heap shrinks to 1/10th its size, leaving behind lots of free and
	// unscavenged memory. mappedReady - heapAlloc will be quite large, because of that free
	// and unscavenged memory, pushing the goal down significantly.
	//
	// heapFree is also safe to exclude from the memory limit because in the steady-state, it's
	// just a pool of memory for future heap allocations, and making new allocations from heapFree
	// memory doesn't increase overall memory use. In transient states, the scavenger and the
	// allocator actively manage the pool of heapFree memory to maintain the memory limit.
	//
	// The second term (marker 2) is the amount of memory we've exceeded the limit by, and is
	// intended to help recover from such a situation. By pushing the heap goal down, we also
	// push the trigger down, triggering and finishing a GC sooner in order to make room for
	// other memory sources. Note that since we're effectively reducing the heap goal by X bytes,
	// we're actually giving more than X bytes of headroom back, because the heap goal is in
	// terms of heap objects, but it takes more than X bytes (e.g. due to fragmentation) to store
	// X bytes worth of objects.
	//
	// The third term (marker 3) subtracts an additional memoryLimitHeapGoalHeadroom bytes from the
	// heap goal. As the name implies, this is to provide additional headroom in the face of pacing
	// inaccuracies. This is a fixed number of bytes because these inaccuracies disproportionately
	// affect small heaps: as heaps get smaller, the pacer's inputs get fuzzier. Shorter GC cycles
	// and less GC work means noisy external factors like the OS scheduler have a greater impact.

	memoryLimit := uint64(c.memoryLimit.Load())

	// Compute term 1.
	nonHeapMemory := mappedReady - heapFree - heapAlloc

	// Compute term 2.
	var overage uint64
	if mappedReady > memoryLimit {
		overage = mappedReady - memoryLimit
	}

	if nonHeapMemory+overage >= memoryLimit {
		// We're at a point where non-heap memory exceeds the memory limit on its own.
		// There's honestly not much we can do here but just trigger GCs continuously
		// and let the CPU limiter reign that in. Something has to give at this point.
		// Set it to heapMarked, the lowest possible goal.
		return c.heapMarked
	}

	// Compute the goal.
	goal := memoryLimit - (nonHeapMemory + overage)

	// Apply some headroom to the goal to account for pacing inaccuracies.
	// Be careful about small limits.
	if goal < memoryLimitHeapGoalHeadroom || goal-memoryLimitHeapGoalHeadroom < memoryLimitHeapGoalHeadroom {
		goal = memoryLimitHeapGoalHeadroom
	} else {
		goal = goal - memoryLimitHeapGoalHeadroom
	}
	// Don't let us go below the live heap. A heap goal below the live heap doesn't make sense.
	if goal < c.heapMarked {
		goal = c.heapMarked
	}
	return goal
}
```