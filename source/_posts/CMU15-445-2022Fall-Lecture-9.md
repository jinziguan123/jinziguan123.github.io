---
title: CMU15-445 2022Fall Lecture 9
date: 2024-02-28 12:48:06
tags: CMU15-445
categories: 笔记
---
# Lecture 9: Index Concurrency Control

本章主要介绍了数据库中几种闩锁的概念和简单的实现（非代码）

# Locks vs. Latches

* lock
  1. 保护当前事务的索引内容不会受到其他事务的影响
  2. lock在整个事务的执行期间都会被持有
  3. DBMS需要在发生冲突的时候回滚变更
* latch
  1. 保护索引内部数据不会被其他线程影响
  2. 仅在线程对索引进行某个操作的时刻被持有（一般持有时间很短）
  3. DBMS不需要对数据的更新进行回滚
  4. latch有两种mode，分别为Read Mode和Write Mode，其中Read Mode可以同时被多个线程持有，但Write Mode不行

# Latch Implementation

## Blocking OS Mutex

* 依赖OS内置的互斥机制，由**用户空间的自旋latch**以及**OS的互斥锁**组成。当DBMS无法获得用户空间的latch时，会试图进入内核态并获取更加昂贵的mutex，如果还是无法获取，则会被阻塞
* OS mutex在DBMS中不是一个好的选择，因为会被OS介入，而且开销比较大
  * 例子：std::mutex  
  * 优点：使用简单
  * 缺点：消耗大，不可扩展

## Test-and-Set Spin Latch(TAS)

* 自旋锁比mutex更有效，因为DBMS可以控制自旋锁在无法获取latch的情况下的下一步动作：比如可以继续尝试获取或者允许OS取消调度
  * 例子：std::atomic\<T>
  * 优点：上锁更高效，一个```std::atomic_flag latch```指令即可上锁
  * 缺点：没有扩展性，对于cache和OS并不友好

## Reader-Writer Latches

* mutex和自旋锁并不区分读写（对于数据库而言是不好的），DBMS需要一种可以支持并发读取的方法，这样如果程序有大量读取的需求就可以获得更好的效果
* 读写锁允许latch以read或者write的mode进行等待，可以跟踪有多少线程持有该latch，以及有多少线程在等待获取latch。对于read锁，我们可以定义其在任何情况下都可以获得这把latch，也可以指定其只有在等待write的线程为空的时候才能获取；对于write锁，则需要等待read锁被全部释放
  * 例子：std::shared_mutex
  * 优点：能够并发读取
  * 缺点：必须要额外维护两个队列：read线程和write线程以防止饥饿现象，内存开销会更大

# Hash Table Latching

* 由于Hash数据结构本身的特性，所有线程在访问的时候都是按照顺序自上而下的，每次也都只会访问一个slot，因此不会出现死锁现象
* 按照粒度的大小，Hash Table Latching可以被分成两种：
  * Page Latches：每个Page都用一把大锁锁住，每个线程在访问Page之前获取一个Read或者Write锁，这样的做法会降低程序的并发性，但是由于线程访问每个slot的速度很快，因为只需要一把锁即可实现
  * Slot Latches：Page中的每个slot都有自己的锁，所以读写线程可以同时访问一个Page的不同slot，提高了并发度，但同时也增加了存储和计算开销

# B+ Tree Latching

* B+树的锁主要为了防止以下问题：
  * 不同线程同时修改同一个结点
  * 当一个线程对结点进行插入/删除，导致结点出现了split/merge现象时，另一个线程正在遍历树
* Latch Crabbing/Couping Protocol（锁存耦合协议）允许多个线程并发访问B+树，具体规则如下：
  * 获取父节点锁
  * 获取子结点锁
  * 如果子结点被认为是安全的（不会发生split、merge、再分配），则释放父节点的锁
* Basic Latch Crabbing Protocol
  * Search：从根结点开始向下，获取子结点锁->释放父节点锁，重复此步骤
  * Insert/Delete：从根结点向下按需获取x个结点的latch，一旦孩子结点被锁住了，检查是否安全，如果安全，则释放所有祖先的锁
  * 释放锁的顺序在逻辑上不重要，然而在现实中越靠近根结点的锁需要更先释放，否则会造成性能的下降
* Better Latch Crabbing Protocol：
  * 在改进的算法中，每一个线程都会默认到达目标结点的路径是安全的，并且以不断抓去Read锁的形式来到达，最终检验是否安全。如果不安全，则停止操作，重新开始，只不过重新遍历会抓去Write锁
  * Search：与朴素的算法一致
  * **Insert/Delete**：跟Search一样，不断获取和释放READ latch。最后在叶子结点上设置WRITE latch。如果叶子不安全，则重来，这次用基本算法的思路
* Leaf Node Scan
  * 前两种算法都是自上而下的，线程无法获取更上一层的锁存器，因此不会出现死锁现象
  * 然而扫描叶子结点的时候很容易出现死锁，当两个线程以相反的方向遍历到了相邻的叶子结点，就会出现死锁，而Index Latch本身并不支持死锁的检测和预防
  * 因为，唯一能够解决这个问题的方式就是通过code discipline，叶子结点的兄弟结点的锁存器必须遵循“no wait”原则。B+树必须面对锁存器获取失败的情况，一般会选择终止操作，重新启动