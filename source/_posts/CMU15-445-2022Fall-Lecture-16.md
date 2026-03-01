---
title: CMU15-445-2022Fall-Lecture-16
date: 2024-03-19 16:35:51
tags:
---
# Lecture 16: Two-Phase Locking

本章主要介绍了数据库事务处理中的2-PL协议

上节课介绍了如何使用WW、WR、RW三种冲突来判断事物之间是否是串行化的，但是在现实中，这种方法是不切实际的，因为我们不可能提前预知所有事物到底什么时候到达，以及事务具体要做什么，而在现实生活中，事务有可能会源源不断地发送到数据库，这就意味着数据库每时每刻都需要将新的事务纳入串行化的判断范畴

因此需要一种协议来确保DBMS在不知道事务的具体内容、具体到达时间的情况下，依然能够作出正确的判断，2-PL协议就是一种通过合理地加锁来达到这种目的的协议



# Lock Type

首先需要再一次强调一下数据库中两种类型锁的区别

|                    | Locks                                             | Latches                     |
| ------------------ | ------------------------------------------------- | --------------------------- |
| Separate           | User transactions                                 | Threads                     |
| Protect            | Database Contents                                 | In-Memory Data Structures   |
| During             | Entire Transactions                               | Critical Sections           |
| Modes              | Shared, Exclusive, Update, Intention              | Read, Write                 |
| Handle deadlock by | Detection & Resolution Waits-for, Timeout, Aborts | Avoidance Coding Discipline |
| Kept in            | Lock Manager                                      | Protected Data Structure    |

本章关注的是Locks，其有两种基本类型：

* S-Lock：共享锁（读锁）
* X-Lock：互斥锁（写锁）

两者的兼容矩阵如下所示：

|                        | S-LOCK (shared) | X-LOCK (exclusive) |
| ---------------------- | --------------- | ------------------ |
| **S-LOCK (shared)**    | ✅               | ❌                  |
| **X-LOCK (exclusive)** | ❌               | ❌                  |

从上节课我们可以知道，WW、WR、RW都可能导致事务冲突，因此只有RR的时候，两把锁是可以兼容的

DBMS有一个名为Lock Manager的模块，专门负责分配、回收事务的锁。每当事务申请加锁或者升级锁的时候，都需要向其发送请求，而Lock Manager内部还维护着一个Lock Table，记录每一个事务所持有的锁（在实验中以哈希表的形式呈现），Lock Manager会根据Lock Table上面的记录来判断是给该事务分配锁，还是拒绝请求，以此确保事务排列的正确性和并发性



# Two-Phase Locking

2-PL帮助数据库在运行过程中决定某个事务是否可以访问某条数据，并且 2PL 的正常工作并不需要提前知道所有事务的执行内容，仅仅依靠已知的信息即可

## Growing & Shrinking

这个协议定义了两个阶段：Growing 和 Shrinking

* 在growing阶段，事务可以按需申请获取锁，Lock Manager可以决定分配与否
* 在shrinking阶段，事务就只能够释放锁，而不能够获取新的锁或者升级已有的锁,**也就是说，一旦事务释放了锁，那么它就再也无法获得锁了**

这样的协议保证了事物之间**一定不会出现有向环路，也就是一定是可串行化的**（具体证明可以参考[这篇文章]([Transaction management：两阶段锁（two-phase locking） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/59535337))）

## Cascading Aborts

尽管2-PL能够确保事务之间的串行性，但其可能会引发一种新的问题：联级中止

如果T1产生了Abort，那么已经读取了T1写入数据的T2就变成了脏读，这种情况下，DBMS需要将有过脏读的事务也反馈为Abort，而这些中止可能进而使得其它正在进行的事务级联地中止，这个过程就是所谓的级联中止

##  Strong Strict 2-PL 

为了解决联级中止的问题，我们可以将2-PL进行改进，也被称为Rigorous 2-PL：

* growing阶段和2-PL相同
* shrinking阶段，每一个事务**只有在自身结束之后才能够释放所有的锁**，无论最终该事务是提交还是中止

通过比较Non-2PL、2PL和Rigorous-2PL，可以看出，三者的并发程度越来越低，但是安全性是越来越高的（其中2PL和Rigourous-2PL的安全性是一致的，但是2PL可能会引发联级中止）



# Deadlock Detection & Prevention

2PL协议无法解决死锁问题，和操作系统中常见的解决死锁问题的方法一样，可以从事后检测（Detection）和事前阻止（Prevention）来考虑

## Deadlock Detection

DBMS会维护一张waits-for graph，用来记录多个事务之间的相互等待关系，一旦图中出现了环，那么DBMS就需要考虑如何去打破这个环

### Deadlock Handling

当 DBMS 检测到死锁时，它会选择一个 "受害者" (事务)，将该事务回滚，打破环形依赖，而这个 "受害者" 将依靠配置或者应用层逻辑重试或中止。这里有两个设计决定：

1. 检测死锁的频率
2. 如何选择合适的 "受害者"

检测死锁的频率越高，陷入死锁的事务等待的时间越短，但消耗的 cpu 也就越多。所以这是个典型的 trade-off，通常有一个调优的参数供用户配置

选择 "受害者" 的指标可能有很多：事务持续时间、事务的进度、事务锁住的数据数量、级联事务的数量、事务曾经重启的次数等等。在选择完 "受害者" 后，DBMS 还有一个设计决定需要做：完全回滚还是回滚到足够消除环形依赖即可

## Deadlock Prevention

Deadlock prevention 是一种事前行为，采用这种方案的 DBMS 无需维护 waits-for graph，也不需要实现 detection 算法，而是在事务尝试获取其它事务持有的锁时直接决定是否需要将其中一个事务中止。

通常 prevention 会按照事务的年龄来赋予优先级，事务的时间戳越老，优先级越高。有两种 prevention 的策略： 

* Old Waits for Young：如果 requesting txn 优先级比 holding txn 更高则等待后者释放锁；更低则自行中止
* Young Waits for Old：如果 requesting txn 优先级比 holding txn 更高则后者自行中止释放锁，让前者获取锁，否则 requesting txn 等待 holding txn 释放锁

无论是 Old Waits for Young 还是 Young Waits for Old，只要保证 prevention 的方向是一致的，就能阻止死锁发生，其原理类似哲学家就餐设定顺序的解决方案：先给哲学家排个序，遇到获取刀叉冲突时，顺序高的优先



# Lock Granularity

当事务在获取锁的时候，DBMS可以决定分配给它的锁的粒度（到底是给它锁住一整张表？还是一个tuple？还是一个page？），DBMS需要尽可能给事务分配最少的锁，并且在并行性和开销之间进行权衡（更少，但粒度更大的锁 vs. 更多，但粒度更细的锁）

## Intention Locks

### Definition

意向锁是一种表级别的锁，用于表明事务将要对某个对象（表、页、行）加锁。它被用来表示一个事务在获取该对象的锁之前是否会先获取其他类型的锁

> 来自百度百科的定义：
>
> 如果另一个任务试图在该表级别上应用共享或排它锁，则受到由第一个任务控制的表级别意向锁的阻塞。第二个任务在锁定该表前不必检查各个页或行锁，而只需检查表上的意向锁

### Why

那么为什么我们需要意向锁呢？举个例子：现在有事务A已经获取了表t的某一行的互斥锁，而此时表B想要获取这张表的表共享锁，由于两把锁互斥，所以B在试图对表t施加锁的时候必须保证：

* 当前没有其他事务持有 users 表的排他锁
* 当前没有其他事务持有 users 表中任意一行的排他锁 

而为了确保第二个条件成立，B就需要检查表t的每一行进行判断。很明显效率非常低，而有了意向锁之后，情况就不一样了，我们只需要检查这张表的IS或者IX锁是否空闲，那么就能直接判断能够进行后续的加锁操作，类比来说，意向锁就是这张表所有锁的哨兵

### Lock Types

* Intention-Shared Lock

  如果我们只需要读取表R的某些行，我们可以在这些行上加上S锁。对于整个表R, 表的任何行都可以被读取。这种情况下，可以给表R加上IS锁。从而，如果此时有另一个事务需要读取表R的某些行或是整个表R, 就能根据R上的IS锁直接做出判断，无需遍历表的每一行

* Intention-Exclusive Lock

  如果我们需要写表R的某些行，此时可以给表R加上IX锁以表达这个意思。例如，表R加上IX锁后，就不能申请R上的S锁，因为IX锁表明某个事务正在修改表R的某些行

* Shared+Intention-Exclusive Lock

  共享意向排它锁，是S锁和IX锁的结合，适用于以下场景：如果需要修改表中的某些行，但需要读取整个表，这时候就可以给整张表加上SIX锁。可以看到它与IX锁的区别：加入SIX锁后，不能修改表的其它行，因为需要读整张表。

  在MySQL中，并没有SIX锁；但在Oracle、SQL Server中有这种锁，此类数据库有更为复杂的树状组织

### Compatibility Matrix

这里给出意向锁、读写锁之间的兼容矩阵，竖为事务T1已经持有的锁，横为T2想要申请的锁：

|         | IS   | IX   | S    | SIX  | X    |
| ------- | ---- | ---- | ---- | ---- | ---- |
| **IS**  | ✅    | ✅    | ✅    | ✅    | ❌    |
| **IX**  | ✅    | ✅    | ❌    | ❌    | ❌    |
| **S**   | ✅    | ❌    | ✅    | ❌    | ❌    |
| **SIX** | ✅    | ❌    | ❌    | ❌    | ❌    |
| **X**   | ❌    | ❌    | ❌    | ❌    | ❌    |

## Locking Protocol

对于一个事务，其需要在数据库层次结构的最高一级获取合适的锁：

* 如果想要获取一个结点的S或者IS锁，事务必须至少获取其双亲结点的IS锁
* 如果想要获取一个结点的X、IX或者SIX锁，则必须至少获取其双亲结点的IX锁