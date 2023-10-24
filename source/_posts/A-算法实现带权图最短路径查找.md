---
title: A*算法实现带权图最短路径查找
tags: [pathfinding, dijkstra, A*]
date: 2023-10-23 19:54:49
---

在游戏或现实世界里，我们经常需要找从一个点到另一个点的路径。这个查找不仅仅要关注最短距离，有时也要考虑耗时，路是否好走，是否收费等等因素。
那么如何查找想要的路径呢，我们可以使用一种图搜索算法来实现，将地图表示成计算机程序可以理解的`graph` 图数据结构。 `A*`算法是一个常用的路径搜索算法。下面我们一步步使用 golang来实现这个算法。

## 表示地图

首先，我们如何表示现实或游戏中的地图呢。在人视觉来看是地图就是一个图片，但是如何让算法能理解地图数据呢。哪里可以通行，哪里不能通行。哪里好走，哪里通过不好走，也就是通行成本。 基于以上考虑我们可以定义如下数据结构

```go
type Graph struct {
	width   int
	height  int
	walls   map[Location]bool
	forests map[Location]int
}
```

width 和 height 是图的长度和宽度，这是必须的，我们要知道地图的范围。walls 是墙，里面存的信息是哪里不可通行。forests 是深林沼泽，标识地图哪里通行成本高。以上，我们就定义出了一个简单的地图数据结构。

上面地图结构已经定义出来了，那我们还需要一些图的操作函数提供给算法使用，比如是否可通行，通行成本，通过一个点获取邻接点等等，如下是代码实现：

```go
// 是否在图内
func (g Graph) inBounds(id Location) bool {
	return 0 <= id.X && id.X < g.width && 0 <= id.Y && id.Y < g.height
}

// 是否可通行
func (g Graph) passable(id Location) bool {
	if _, ok := g.walls[id]; ok {
		return false
	}
	return true
}

// 通过一个点获取邻接点
func (g Graph) neighbors(id Location) []Location {
	var result []Location

	for _, dir := range DIRS {
		next := Location{
			X: id.X + dir.X,
			Y: id.Y + dir.Y,
		}

		if g.inBounds(next) && g.passable(next) {
			result = append(result, next)
		}

	}
	if (id.X+id.Y)%2 == 0 {
		util.Reverse(result)
	}

	return result
}

// 到下个点的通行成本
func (g Graph) cost(from_node, to_node Location) int {
	_, ok := g.forests[to_node]
	if ok {
		return 5
	}

	return 1
}
```

以上完成了定义地图数据结构以及辅助函数。这里为了方便理解可视化，咱们定义的网格地图比较简单。

## 算法

- Dijkstra算法，允许我们优先考虑要探索的路径。与其平等地探索所有可能的路径，它更倾向于低成本的路径。我们可以分配较低的成本来鼓励在道路上移动，分配较高的成本来躲避敌人等等。

- A*是对Dijkstra算法的修改，该算法针对单个目的地进行了优化。Dijkstra算法可以找到所有位置的路径；A*查找到一个位置或几个位置中最近的位置的路径。它优先考虑那些似乎更接近目标的路径。

下面我们先实现 Dijkstra 算法，然后再基于 Dijkstra 实现 A* 算法。

基于 Dijkstra 的算法描述，我们需要有以下三点要求

- graph 图数据结构需要知道邻接点之间的移动成本
- queue 队列需要以不同的顺序返回节点
- 搜索时需要跟踪已访问的点，记录成本，并将其提供给队列 queue

### 带权图

上面我们定义图的时候已经定义出了具有权重属性的图了，也就是上面的 forests 属性。通过 cost 函数可以基于权重信息获取移动成本

### Priority Queue

我们使用 通过 golang的 heap 包实现的优先级队列，队列每个元素有两个信息，一个是节点，一个是优先级。我们可以定义比较器，优先返回最高优先级或最低优先级的元素。这里根据需要，我们需要返回最低优先级的元素。以下是队列实现部分代码：

```go
type _item struct {
	value    any // The value of the item; arbitrary.
	priority int // The priority of the item in the queue.
	// The index is needed by update and is maintained by the heap.Interface methods.
	index int // The index of the item in the heap.
}

// A PriorityQueue implements heap.Interface and holds Items.
type _elements []*_item

type priority_queue struct {
	elements *_elements
	compare  _compare
}

type PriorityQueue struct {
	queue *priority_queue
}

func (pq PriorityQueue) Empty() bool {
	if pq.queue.Len() > 0 {
		return false
	}
	return true
}

func (pq PriorityQueue) Put(val any, priority int) {
	item := &_item{
		value:    val,
		priority: priority,
	}
	heap.Push(pq.queue, item)
	pq.queue.update(item, val, priority)
}

func (pq PriorityQueue) Get() any {

	item := heap.Pop(pq.queue).(*_item)

	return item.value
}
```

### 搜索

我们传入五个变量，分别是图结构`graph`，起点`start`，目标点`goal`，存储追踪搜索路径信息变量 `came_from`，存储距离成本信息变量`cost_so_far`。然后开始检索

第一步：
变量初始化，初始化一个优先级队列`frontier`，将起点放入其中。初始化`came_from`和`cost_so_far`

第二步：
判断`frontier`是否为空，不为空循环遍历`frontier`，从中取出待检索节点。

第三步：
判断待检索的点是否是是目标点，如果是直接退出。否则执行第四步。

第四步：
获取当前检索节点的邻接点，遍历所有邻接点，获取到这个邻接点的移动成本。判断对应邻接点是否已有最小移动成本，如果没有或者最小移动成本大于当前新获取的移动成本。则更新最小移动成本变量`cost_so_far`，更新这个邻接点最新的移动成本，记录访问路径，然后将这个邻接点再访问待检索变量`frontier`中。继续第二步。


```go
func DijkstraSearch(graph Graph, start, goal Location, came_from map[Location]Location, cost_so_far map[Location]int) {
	frontier := priority.New(priority.Lesser)
	frontier.Put(start, 0)

	came_from[start] = start
	cost_so_far[start] = 0

	for !frontier.Empty() {
		current, ok := frontier.Get().(Location)
		if !ok {
			continue
		}

		if current.equal(goal) {
			break
		}

		for _, next := range graph.neighbors(current) {
			new_cost := cost_so_far[current] + graph.cost(current, next)
			if _, ok := cost_so_far[next]; !ok || new_cost < cost_so_far[next] {
				cost_so_far[next] = new_cost
				came_from[next] = current
				frontier.Put(next, new_cost)
			}
		}
	}
}
```

检索完成后，因为 `came_from`存储的是途径点的前向节点，所以从目标点反向推算，就可获取从起点到目标点的最小移动成本路径。获取路径代码如下：

```go
func ReconstructPath(start, goal Location, came_from map[Location]Location) []Location {
	var path []Location

	current := goal

	for !current.equal(start) {
		path = append(path, current)
		current = came_from[current]
	}

	path = append(path, start)
	util.Reverse(path)
	return path
}
```

### 效果

下面我们 定义一个图数据，然后将它的搜索路径简单绘制出来，以下是生成图以及绘制路径的代码

```go
func MakeDiagram4() Graph {
	grid := NewGraph(10, 10)
	addRect(grid, 1, 7, 4, 9)

	grid.forests = map[Location]int{
		{3, 4}: 1, {3, 5}: 1, {4, 1}: 1, {4, 2}: 1,
		{4, 3}: 1, {4, 4}: 1, {4, 5}: 1, {4, 6}: 1,
		{4, 7}: 1, {4, 8}: 1, {5, 1}: 1, {5, 2}: 1,
		{5, 3}: 1, {5, 4}: 1, {5, 5}: 1, {5, 6}: 1,
		{5, 7}: 1, {5, 8}: 1, {6, 2}: 1, {6, 3}: 1,
		{6, 4}: 1, {6, 5}: 1, {6, 6}: 1, {6, 7}: 1,
		{7, 3}: 1, {7, 4}: 1, {7, 5}: 1,
	}

	return grid
}

func print(s string) {
	fmt.Printf("%-3s", s)
}

func DrawGrid(graph Graph, path []Location) {
	newpath := map[Location]bool{}
	for _, location := range path {
		newpath[location] = true
	}

	for y := 0; y != graph.height; y++ {
		for x := 0; x != graph.width; x++ {
			id := Location{x, y}
			if _, ok := graph.walls[id]; ok {
				print("#")
			} else if _, ok := newpath[id]; ok {
				print("@")
			} else {
				print(".")
			}
		}
		fmt.Println()
	}
}
```

然后我们执行调用，具体调用如下：

```go
graph := implementions.MakeDiagram4()

start := implementions.Location{X: 1, Y: 4}
goal := implementions.Location{X: 8, Y: 5}

came_from := map[implementions.Location]implementions.Location{}
cost_so_far := map[implementions.Location]int{}

implementions.DijkstraSearch(graph, start, goal, came_from, cost_so_far)

path := implementions.ReconstructPath(start, goal, came_from)

implementions.DrawGrid(graph, path)
```

实际效果如下所示：

```
.  .  .  @  @  @  @  .  .  .  
.  .  .  @  .  .  @  @  .  .  
.  .  .  @  .  .  .  @  @  .  
.  .  @  @  .  .  .  .  @  .  
.  @  @  .  .  .  .  .  @  .  
.  .  .  .  .  .  .  .  @  .  
.  .  .  .  .  .  .  .  .  .  
.  #  #  #  .  .  .  .  .  .  
.  #  #  #  .  .  .  .  .  .  
.  .  .  .  .  .  .  .  .  .
```

### A* 算法

A* 算法是在前面 Dijkstra 的基础上加的一个启发函数，让搜索路径优先向目标点搜索。启发函数代码：

```go
func heuristic(a, b Location) int {
	return int(math.Abs(float64(a.X-b.X)) + math.Abs(float64(a.Y-b.Y)))
}
```

A* 算法

```go
func AStarSearch(graph Graph, start, goal Location, came_from map[Location]Location, cost_so_far map[Location]int) {
	frontier := priority.New(priority.Lesser)
	frontier.Put(start, 0)

	came_from[start] = start
	cost_so_far[start] = 0

	for !frontier.Empty() {
		current, ok := frontier.Get().(Location)
		if !ok {
			continue
		}

		if current.equal(goal) {
			break
		}

		for _, next := range graph.neighbors(current) {
			new_cost := cost_so_far[current] + graph.cost(current, next)
			if _, ok := cost_so_far[next]; !ok || new_cost < cost_so_far[next] {
				cost_so_far[next] = new_cost
				priority := new_cost + heuristic(next, goal)
				frontier.Put(next, priority)
				came_from[next] = current
			}
		}
	}
}
```

可以看出，A* 算法和前面并无太大不同，但是有了启发函数，就可以实现，又快又准的找到最优路径。

### 相关源码

[Priority Queue](https://github.com/baifei2014/jqueue "Priority Queue")
[Pathfinding](https://github.com/baifei2014/pathfinding "pathfinding")


### 参考资料

[Introduction to the A* Algorithm](https://www.redblobgames.com/pathfinding/a-star/introduction.html "Introduction to the A* Algorithm")
[Implementation of A* ](https://www.redblobgames.com/pathfinding/a-star/implementation.html "Implementation of A* ")










