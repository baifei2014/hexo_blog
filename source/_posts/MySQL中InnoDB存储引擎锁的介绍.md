---
layout: post
title: MySQL中InnoDB存储引擎锁的介绍
date: 2019-06-24 20:15:48
tags: [MySQL, InnoDB, LOCK]
---
## 共享锁和排它锁
&nbsp;&nbsp;InnoDB实现了两种类型的行级锁，分别是共享锁和排它锁。

- 共享锁(s)允许事务读取一行
- 排它锁(x)允许事务更新或删除一行

&nbsp;&nbsp;如果事务T1在r行持有共享锁，事务T2在r行申请锁操作会有如下现象：
- 事务T2可以被分配共享锁，T1和T2在r行都拥有共享锁。
- 事务T2不能被分配排它锁

&nbsp;&nbsp;如果事务T1在r行拥有排它锁，事务T2在r行哪种类型的锁都获取不到了，只能等待事务T1释放r行的锁。

&nbsp;&nbsp;事务T1和T2获取锁的过程如下：
![事务T1](https://jlh-1255362804.cos.ap-beijing.myqcloud.com/artical/20190712-ex1.png)
![事务T2](https://jlh-1255362804.cos.ap-beijing.myqcloud.com/artical/20190712-ex2.png)
&nbsp;&nbsp;事务T1获取共享锁后，事务T2在同一行可以获取共享锁，但是不能获取排它锁，事务T2在尝试获取排它锁时会阻塞等待事务T1释放锁，直到等待超时。

## 意向锁
&nbsp;&nbsp;InnoDB支持行级和表级这样不同粒度的锁定。为了实现在不同粒度级别实现锁定，InnoDB使用了意向锁。意向锁是表级锁，它表明事务稍后对表中行所加锁的类型（共享锁或排它锁），下面讲了两种类型的意向锁：
- 意向共享锁(IS)意味着事务会在表个别行设置共享锁。
- 意向排它锁(IX)意味着事务会在表个别行设置排它锁。

&nbsp;&nbsp;例如，SELECT ... LOCK IN SHARE MODE设置的是意向共享锁，SELECT ... FOR UPDATE设置的是意向排它锁。

&nbsp;&nbsp;意向锁协议如下：
- 事务在可以获取表中行的共享锁之前，它必须首先获取表的意向共享锁或更严格的锁。
- 事务在可以获取表中行的排它锁之前，它必须首先获取表的意向排它锁。
&nbsp;&nbsp;下面表里总结了表级锁类型的兼容性

待增加表格

## 间隙(Gap)锁
&nbsp;&nbsp;间隙锁锁住了索引记录之前的间隙记录，或锁住了整个表。比如说，SELECT c1 FROM t WHERE c1 BETWEEN 10 AND 20 FOR UPDATE;这会避免其它事务新增t.c1列值为15的记录，无论这个列是否存在这个值，因为间隙范围内所有存在的值都被锁定了。

&nbsp;&nbsp;下面展示了间隙锁在实际应用中的例子：
![示例3](https://jlh-1255362804.cos.ap-beijing.myqcloud.com/artical/20190712-ex3.png)
![示例4](https://jlh-1255362804.cos.ap-beijing.myqcloud.com/artical/20190712-ex4.png)
&nbsp;&nbsp;在事务1中，更新完user_no为102的记录后加锁，这时事务2中新增user_no为102的记录就会阻塞等待，如果事务1释放锁，会执行成功，否则直到等待超时。这里起作用的就是间隙锁，它避免了幻读。

&nbsp;&nbsp;间隙可能是单条记录，多条记录，甚至是空的。

&nbsp;&nbsp;上面示例的间隙锁展示了单条记录，下面来看下多个间隙：

![示例3](https://jlh-1255362804.cos.ap-beijing.myqcloud.com/artical/20190712-ex5.png)
![示例4](https://jlh-1255362804.cos.ap-beijing.myqcloud.com/artical/20190712-ex6.png)
&nbsp;&nbsp;在事务1中，当执行update更新user_no为95的记录时，虽然没有这个记录，但还是在前后都加了间隙锁，这个范围是[90,102]，所以当在事务2中执行新增时，如果列值是90到102范围内的，都会阻塞等待直到释放锁，否则就执行超时，但是在范围外就能执行成功，这是在user_no是有索引的情况，如果user_no列没有索引，间隙锁会锁住整张表

&nbsp;&nbsp;间隙锁是并发和性能之间平衡的一部分。

&nbsp;&nbsp;获取行级锁的语句使用唯一索引查找唯一记录不需要间隙锁(这不包括查找条件仅包含多列唯一索引的部分列，这种情况时，间隙锁会起作用)，例如，如果id列有唯一索引，下列语句仅仅id值为100的行被锁定，无论其它sessions里增加多少条记录前面的间隙都不会受影响。

```sql
SELECT * FROM child WHERE id = 100;
```
&nbsp;&nbsp;如果id不是索引或没有唯一索引，这个语句就会锁住前面的间隙。

&nbsp;&nbsp;需要注意的一点是，不同的事务中冲突的锁可以拥有一个间隙。例如，事务A在一个间隙可以拥有共享间隙锁(gap S-lock)，同时，事务B在相同的间隙可以持有排它间隙锁(gap X-lock)。冲突间隙锁被允许的原因是如果索引中的一个记录被删除了，不同事务中间隙锁持有的记录必须被合并。

&nbsp;&nbsp;InnoDB中间隙锁是“完全被限制的”，这意味着间隙锁的目的只是避免其他事务新增记录到间隙中。间隙锁可以同时存在。一个事务的间隙锁没法避免别的事务在同一间隙上执行间隙锁，这里不同于共享和排它间隙锁，它们之间互相没有冲突，并且扮演相同的作用。

&nbsp;&nbsp;间隙锁可以被声明隐藏。如果改变事务隔离级别到已提交读(READ COMMITED)或隐藏innodb_locks_unsafe_for_binlog系统变量(现已废弃)。在这些情况下，查找和索引扫描间隙就被隐藏了，只有在外键和检查重复键时才会使用。

&nbsp;&nbsp;在使用已提交读隔离级别或设置innodb_locks_unsafe_for_binlog时会有一些别的影响。MySQL在评估WHERE条件列之后，会释放没有匹配到记录的锁。拿UPDATE来说，InnoDB执行一个semi-consistent读，它返回一个最新提交的版本号，让MySQL决定是否满足UPDATE的WHERE条件。



> semi-consistent read是read committed与consistent read两者的结合。一个update语句，如果读到一行已经加锁的记录，此时InnoDB返回记录最近提交的版本，由MySQL上层判断此版本是否满足update的where条件。若满足，则MySQL会重新发起一次读操作，此时会读取行的最新版本(并加锁)。

## 参考资料
- [MySQL参考手册](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html)
