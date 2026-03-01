---
title: CMU15-445 2022Fall Lecture 6
date: 2024-02-28 12:45:09
tags: CMU15-445
categories: 笔记
---
# Letcure 6: Memory Management

本章主要介绍了数据库中内存管理的相关内容，主要引入了缓冲池的概念

# Introduction

DBMS的任务就是管理内存以及从磁盘写入写出数据。数据库如果希望对磁盘上的数据进行处理，则必须首先将数据移动到内存中

因此我们需要设计一个DBMS，满足以下几个要求：

* 其能够处理的数据量超出内存的大小
* 最小化磁盘上执行查询所带来的低效率问题
* 使得所有的操作看起来都像是在磁盘上进行的一样

这个问题可以从时间上和空间上两个方面来考虑：

* **空间上**：我们需要将Page写在磁盘的哪一个部分？这是为了保证数据的空间局部性
* **时间上**：我们应该何时进行Page的写入写出？这是为了尽可能减少从磁盘上读取数据的停顿次数

# Buffer Pool

* 执行引擎在请求Page的时候会先去buffer pool查询，如果没有，则buffer pool会执行替换算法将目标Page换入其中。此时脏页会被写会磁盘，而非脏页会被直接舍弃。可以参考cache在计算机硬件中的作用
* 缓冲池是一个从磁盘上读取数据的内存区域，该内存区域被组织称一个个固定大小的页面阵列，被称为frame。每个frame保存的都是磁盘中的一个Page，因此为了方便调度，我们需要Page Table（页表）。
* 页表是一个Hash表，用来跟踪处于buffer pool中的Page，我们可以通过页表以及上面的page_id来确认此Page是否还在buffer pool中。当然页表还会记录一些元数据
  * Dirty Flag：标志位，表示这个Page是否被改写过，这是为了保证数据持久化
  * Pin Count：正在引用或者正在访问该Page的线程的数量，如果这个meta data非零，那么我们当下就不应该将这个Page驱逐出buffer pool

# Lock vs. Latch

由于我们并不希望我们在移动Page的时候被其他线程干扰（比如修改这个Page上的数据），所以我们需要对Page进行上锁操作

这涉及到了数据库中Lock和Latch的一些区别：

* lock
  1. 保护当前事务的索引内容不会受到其他事务的影响
  2. lock在整个事务的执行期间都会被持有
  3. DBMS需要在发生冲突的时候回滚变更
* latch
  1. 保护索引内部数据不会被其他线程影响
  2. 仅在线程对索引进行某个操作的时刻被持有（一般持有时间很短）
  3. DBMS不需要对数据的更新进行回滚
  4. latch有两种mode，分别为Read Mode和Write Mode，其中Read Mode可以同时被多个线程持有，但Write Mode不行

# Page Table vs. Page Directory

这里需要辨析两个概念，页表和页目录虽然看上去很像，但本质上有很大区别：

* Page Table：内存中的一个哈希表，并不需要持久化，即哪怕丢弃了，只需要重新建立一个就行
* Page Directory：从page_id到数据库文件中页面位置的映射，对于页目录的所有改变都必须要记录到磁盘上，以允许DBMS在重启之后能够发现，否则会破坏数据的一致性

# Allocation Policies

数据库中的内存是根据两个策略来分配给buffer pool的

* Global Policies：
  全局策略处理DBMS应该作出的决定，以有利于正在执行的整个工作负载
  他考虑到所有活动的事务，以找到分配内存的最佳策略
* Local Policies：
  本地策略作出的决定会使得单个查询或事务的执行更快
  本地策略将框架分配给特定的事务而不考虑事务的并发行为

# Buffer Pool Optimization

有许多方法可以来优化buffer pool

## Multiple Buffer Pool

我们可以分配多块内存区域以支持多个Buffer Pool，每个区域可以有自己的页表，并且自己一套page id和frame的映射关系，这样可以更好地运用局部策略，并且根据不同的类型来分配page的置换策略（例如index或者table data），这样也能减少访问缓存池的不同线程之间对于页表锁的争夺

有两种办法来将Page映射到不同的buffer pool中：

* Objects IDs：
  对象ID涉及扩展记录标识(extending the record IDs)，使其具有一个对象标识符(object identifier)。然后，通过对象标识符，可以维持一个从对象到特定缓冲池的映射关系。
* Hashing：
  哈希也是老生常谈的方法了，不展开

## Pre-Fetching

DBMS也可以通过基于查询计划的预取页面来进行优化。然后，当第一组页面被处理时，第二组页面可以被预取到缓冲池中。**这种方法是DBMS在连续访问许多页面时常用的**

## Scan Sharing

查询游标可以有效减少对于数据的访问

> Q1:select sum(val) from A
> Q2:select avg(val) from A limit 100
> 对于以上两个查询，如果Q1在执行到一半的时候，Q2加入了进来，那么就可以将Q2定位到Q1此时的游标位置，当游标遍历结束之后，Q2再针对前面没有被查询到的数据进行访问

## Buffer Pool Bypass

对于需要线性读取大量数据的查询操作，可以选择不将获取的页面存储在buffer pool中，可以随意丢弃

此外，对于中间数据（类似于join和sorting），也可以这么做

# Buffer Replacement Policies

buffer pool在写入新页而没有空间时，需要执行evict操作给新页腾位子
替换策略的目标是准确性、正确性、并发性、和元数据的开销

## Least Recently Used（LRU）

对于缓存池中每个page维护一个时间戳，时间戳记录着每个page最后被访问的时刻。当DBMS需要驱逐（evict）一个page时，选择时间戳最早的page执行驱逐。一般来说我们可以让page按照时间戳排序（优先级队列）以减少搜索的时间。

## Clock

CLOCK策略是LRU的一个近似值，不需要每页有单独的时间戳。在CLOCK策略中，每个页面被赋予一个参考位。当一个页面被访问时，设置为1
为了形象地说明这一点，可以用 "时钟指针 (clock hand)"将页面组织在一个圆形的缓冲区中。在清扫时检查一个页面的位是否被设置为1，如果是，则设置为0，如果不是，则驱逐它。通过这种方式，时钟指针在驱逐之间记住了位置

## Alternative

无论是LRU还是Clock，都会受到sequential flooding的影响：在进行顺序访问的时候，DBMS会不断把连续的page读入内存，但这些正在使用的page是我们最不想要的（因为我们不过是为了查找到自己想要的page，迫不得已才把之前的page读入buffer pool）
有三种方法可以解决LRU和Clock的缺点：

* LRU-K：
  frame组成的链表被分为两个区域：young list和old list，当old list中的page被再一次访问到，则将其放入young list的头部；而新加入buffer pool的页面会被放入old list的头部
  它跟踪最后K个引用的历史作为时间戳，并计算出后续访问的间隔。这个历史记录被用来预测一个页面下次被访问的时间
* Localization per Query
  DBMS跟踪每一个查询的访问情况，在每个查询的基础上选择哪些页面要被驱逐。这使得每次查询对缓冲池的污染最小化
* Priority Hints
  优先级提示允许事务根据查询执行过程中每个页面的上下文，告诉缓冲池页面是否重要

# Dirty Page

DBMS有两种办法来处理有脏位的页面：

* Fast Path：直接丢弃掉buffer pool中任何非脏页
* Slow Path：将脏页写回磁盘

避免不必要的页面写回的方法是**后台写入**
通过后台写入，DBMS可以周期性地通过页表来写回脏页到磁盘上
当一个脏页被后台写入，DBMS可以要么将这个page驱逐出去，或者保持其在buffer pool中的位置但是重置其dirty flag