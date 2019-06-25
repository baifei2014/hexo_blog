---
title: Java使用LinkedHashMap实现LRU字典
tags: [Java, LRU]
date: 2019-03-12 15:32:45
---
LRU全拼Least recently used，即最近最少使用，根据历史访问记录淘汰响应的数据。当前在互联网架构中使用非常广泛的存储系统Redis，在内存超过限制内存时，其中一种淘汰策略就使用到LRU算法，这里闲话一下Redis，作为当前使用非常普遍的存储系统中间件，这就要求后端工程师不仅要学会如何使用Redis，更要求我们了解实现细节，这对发现问题，正确使用Redis都是非常有帮助的，而今天这篇文章，也帮助我们了解Redis中使用到的淘汰策略。

## LRU算法
LRU算法使用到key/value的字典，以及链表，链表按照一定的顺序排列，当某个元素被访问时，它在链表中的位置会移动到表头，整个链表的顺序就是按访问顺序排列的，当长度超过最大限制长度了，就会删除最近最少访问的元素。在Java里面，刚好有数据结构满足我们说的情况，这就是LinkedHashMap。

### LinkedHashMap
LinkedHashMap继承自HashMap，实现了Map接口，类的属性定义如下：
``` java
/**
 * 双向链表排序属性，如果为true，按访问排列，如果为false，按插入顺序排列
 */
final boolean accessOrder;
/**
 * 双向链表头节点
 */
transient LinkedHashMap.Entry<K,V> head;
/**
 * 双向链表尾节点
 */
transient LinkedHashMap.Entry<K,V> tail;
```
在LinkedHashMap中，元素为链式存储，每个节点都有个前驱节点和后继节点，源代码如下：
``` java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```
可以看出，Entry类继承的还是HashMap里的Node类，所以结构基本和HashMap节点一样，只是在原先基础上增加了两个变量，一个存储前驱节点，一个存储后继节点。
说完了节点的数据结构，接下来看下get一个节点时都会有哪些操作，首先先看下get函数及相关代码：
``` java
public V get(Object key) {
    Node<K,V> e;
    if ((e = getNode(hash(key), key)) == null)
        return null;
    if (accessOrder)
        afterNodeAccess(e);
    return e.value;
}
void afterNodeAccess(Node<K,V> e) { // move node to last
    LinkedHashMap.Entry<K,V> last;
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a != null)
            a.before = b;
        else
            last = b;
        if (last == null)
            head = p;
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```
关键代码在于afterNodeAccess函数的调用，当accessOrder值为true时，调用afterNodeAccess函数对节点进行调整，其核心就是类尾部指针指向访问节点。这样，每次访问，就会将访问节点放在尾部，那头部就是最近最少访问的。
说完了get操作，再来看下put操作，由于LinkedHashMap中没有实现put函数，所以调用的是HashMap的put函数，put函数进行了一系列操作，代码如下：
``` java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```
put操作大致流程是：
* 1，根据key获取hash值，根据哈希值及桶的长度获取在桶中的位置；
* 2，如果没有发生哈希碰撞，直接将节点放入存储单元
* 3，如果发生哈希碰撞，采用链地址法解决冲突，如果链地址法解决冲突元素超过限定值，采用红黑树优化；
* 4，如果元素个数超过阈值，对桶进行扩容；
* 5，调用节点新增函数；
在第五步时，LinkedHashMap重写了这个调用函数，相关源码信息如下：
``` java
void afterNodeInsertion(boolean evict) { // possibly remove eldest
    LinkedHashMap.Entry<K,V> first;
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```
put操作时，evict传参始终为true，所以当链表不为空时，如果removeEldestEntry函数返回值为true，就会移除链表头部节点，在LinkedHashMap中实现的removeEldestEntry函数返回值始终为false，所以我们只需要重写这个函数，当超过容量时，返回true，就会按最近最少访问原则移除元素。

### LRULinkedHashMap
依据对LinkedHashMap的了解，我们可以实现一个LRU Cache功能的类，代码如下：
``` java
package dragon.likecho.com;

import java.util.*;

public class LRULinkedHashMap<K, V> extends LinkedHashMap<K, V> {
    private int capacity;
    private static final long serialVersionUid = 1L;
    LRULinkedHashMap(int capacity) {
        super(16, 0.75f, true);
        this.capacity = capacity;
    }
    @Override
    public boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > this.capacity;
    }
}
```
代码中有几个点：
* 1，继承于LinkedHashMap；
* 2，定义容量，并调用父类构造函数，设置accessOrder值为true；
* 3，重写removeEldestEntry函数，判断当前容器大小是否超过容量限制，如果超过，返回true，执行移除节点操作；
通过这几步，我们就实现了一个新的容器，具有LRU特点。





