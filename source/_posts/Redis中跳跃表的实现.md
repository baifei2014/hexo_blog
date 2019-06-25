---
title: Redis中跳跃表的实现
tags: [Redis, skipList, C]
date: 2019-06-19 11:56:05
---
跳跃表其实也是链表，它在链表的基础上实现了跳跃，以此实现高效率的查找，链表的查询效率是O(N)，而跳跃表在查询时能实现O(logN)的时间复杂度。
## 跳跃表的概念
普通链表结构，如果链表时有序的，在查找时，是逐个搜索的，直到找到对应节点为止，在插入时，也是对应的逻辑，需要逐个对比找到相应的位置。结构如下图所示：
{% asset_img sorted_linked_list.png 跳跃表 %}
以上面列表为例，如果每隔一个元素，增加一个指针，指针指向下下个节点，这样就生成了一个新链表，可以把原始链表称为第一层级，新生成的称为第二层级，逻辑结构如图所示：
{% asset_img skip2node_linked_list.png 跳跃表 %}
比如要搜索22节点，在原始链表里，逐个搜索需要5次才能完成。使用新生成的链表，先在第二层级搜索，确认范围到低层级继续搜索，过程是这样的：
- 先比较7小于22，接着比较下一节点；
- 19小于22，但是26大于22，到下一层级，取19的下一节点比较；
- 22等于22，搜索完成；
新的思路不用再去逐个比较所有元素了，搜索时只需要逐层的缩小范围，就可以找到目标元素位置，这里层级只有两级，比如说19节点在第三级，也是一样的逻辑。
跳跃表就是基于多层级链表的启发而设计出来的，在上图中，是一个理想结构，上一层级是下一层级元素的1/2，元素是均匀的，这样，搜索时效率其实就是顺序存储时的二分查找，在实际应用中，如果要保证这么均匀的结构，意味着要不断调整链表元素位置，造成效率非常低下。所以它的解决方法是每个元素的层级是随机的，这样，就不会影响到别的元素，每次在插入时只需要调整前后元素的指向就行了。
## Redis里跳跃表实现
跳跃表因为它实现简单，并且拥有着媲美红黑树的效率，所以Redis里用它来实现有序集合。
### 跳跃表的定义
下面我们来看下Redis里跳跃表的实现，Redis里跳跃表和节点的定义结构如下：
```c
typedef struct zskiplistNode {
    // member值，可边长字符串
    sds ele;
    // score值
    double score;
    // 回溯指针
    struct zskiplistNode *backward;
    struct zskiplistLevel {
        // 下一节点指针
        struct zskiplistNode *forward;
        unsigned long span;
    } level[];
} zskiplistNode;

typedef struct zskiplist {
    // 头结点和尾节点
    struct zskiplistNode *header, *tail;
    // 跳跃表长度，元素数量
    unsigned long length;
    // 跳跃表层级
    int level;
} zskiplist;
```
每个节点都有一个层级的结构体，指针指向当前层级下一节点，加上backward指针，跳跃表在第一层级时是双向链表结构，这让逆序遍历链表也非常方便。
### 跳跃表新增节点
跳跃表在新增节点时，会计算在每一层级元素将要放置的位置，源码如下：
```c
for (i = zsl->level-1; i >= 0; i--) {
    /* store rank that is crossed to reach the insert position */
    rank[i] = i == (zsl->level-1) ? 0 : rank[i+1];
    while (x->level[i].forward &&
            (x->level[i].forward->score < score ||
                (x->level[i].forward->score == score &&
                sdscmp(x->level[i].forward->ele,ele) < 0)))
    {
        rank[i] += x->level[i].span;
        x = x->level[i].forward;
    }
    update[i] = x;
}
```
根据score值进行排序，由代码也可以看出，Redis中允许节点的score值是可以相等的，这也是Redis种跳跃表实现的特点。
前面我们也讲了，节点的层级为了效率问题，采用了随机获取的方式，源码如下：
```c
int zslRandomLevel(void) {
    int level = 1;
    while ((random()&0xFFFF) < (ZSKIPLIST_P * 0xFFFF))
        level += 1;
    return (level<ZSKIPLIST_MAXLEVEL) ? level : ZSKIPLIST_MAXLEVEL;
}
```
这是以一定的概率去获取层级，在Redis中，这个概率是25%，每次成功晋升，层级加1，同时限制最大层级为64层。
获取当前节点的层级之后，创建节点元素，源码如下：
```c
x = zslCreateNode(level,score,ele);
for (i = 0; i < level; i++) {
    x->level[i].forward = update[i]->level[i].forward;
    update[i]->level[i].forward = x;

    /* update span covered by update[i] as x is inserted here */
    x->level[i].span = update[i]->level[i].span - (rank[0] - rank[i]);
    update[i]->level[i].span = (rank[0] - rank[i]) + 1;
}
```

这一步是往跳跃表里新增节点，这一步和普通链表新增节点的逻辑一样，都是改变节点指向，不同之处在于，由于节点层级不同，需要根据节点层级循环设置节点在每层的节点指向。
整个节点的插入过程就是这样的，它是以尽量均匀的方式平衡节点层级分布。
### 实例观察
下面我们以一个例子观察节点的层级分布，调用样例如下：
```c
int main(int argc, char **argv)
{
    zset *zs = malloc(sizeof(*zs));
    zs->zsl = zslCreate();
    zslInsert(zs->zsl, 10, "你好，JAVA");
    zslInsert(zs->zsl, 20, "你好，PHP");
    zslInsert(zs->zsl, 30, "你好，PYTHON");
    zslInsert(zs->zsl, 150, "你好，dubbo");
    zslInsert(zs->zsl, 130, "你好，spring");
    zslInsert(zs->zsl, 140, "你好，laravel");
    zslInsert(zs->zsl, 40, "你好，go");
    zslInsert(zs->zsl, 50, "你好，scalar");
    zslInsert(zs->zsl, 60, "你好，RUBY");
    zslInsert(zs->zsl, 70, "你好，javascript");
    zslInsert(zs->zsl, 80, "你好，r");
    zslInsert(zs->zsl, 90, "你好，mysql");
    zslInsert(zs->zsl, 100, "你好，mongo");
    zslInsert(zs->zsl, 110, "你好，redis");
    zslInsert(zs->zsl, 120, "你好，memcache");
    return 0;
}
```
最终生成的结构如下图所示：
{% asset_img skip_list_demo.png 跳跃表 %}
## 小结
由结果可以看出，由于是随机的层级，所以层数分布的不是绝对均匀的，不过这也是为了提高效率的做法。
以上，就是跳跃表的概念以及redis跳跃表的实现和新增节点的逻辑。