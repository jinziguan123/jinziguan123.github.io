---
title: CMU15-445 2022Fall Lecture 11
date: 2024-02-28 12:49:07
tags: CMU15-445
categories: 笔记
---
# Lecture 11: Join Algorithms

本章介绍了join算法的思想和实现

# Join Operator

* 首先是join在底层是如何实现的：这里有两种方式

  例如对于MySql语句：```select R.id, S.date from R join S on R.id = S.id where S.value > 100```

  * Early Materialization：

    这种方法一开始就将所有我们需要用到的具体数据都读取好放入内存，详细一点来讲就是把R.id和S.data直接作为新的tuple插入表中，这样一来，该查询的所有后续操作都不用再回到磁盘中读取数据

  * Late Materialization：

    与前者相对的，滞后具象方法在一开始只会将匹配到的tuple的Record IDS放入表中，等到上层的操作需要获取数据的时候再去磁盘中拿。这种方法对于列存储的DBMS比较友好，因为最终除了我们需要的数据以外，我们不会读取到任何无关的数据

# Join vs. Cross-Product

这里需要进行一下连接操作与笛卡尔积之间的比较：

相对而言笛卡尔积是一种更加低效的方法，因为对于两张大小分别为m和n的表，一次笛卡尔积操作需要用两个for循环来遍历两张表，最终获得一张m*n大小的输出表

> 例如```select * from s cross join e```就是一个笛卡尔积操作，最终输出的结果一来没有规律可言，而来耗费时间更长
>
> 而```select * from s join e on s.id = e.id```是一个join操作，输出结果明显会更小更快

# Join Algorithms

## Nested Loop Join

### Naive Nested Loop Join

* 这是最简单的join算法，遍历两个表中的所有tuple，如果两两匹配，则输出

* 显然这是一个糟糕的算法，如果R表有M个page、m个tuple，S表有N个page、n个tuple，则对于一次join，开销将会是$$M + (m \times N)$$

  > 例如R有1000页，100000个tuple；S有500页，40000个tuple
  >
  > 那么做一次join的IO次数为：$$1000 + (100000 \times 500) = 50001000$$次
  >
  > 如果一次IO要0.1ms，那么join一次就要将近1.3小时
  >
  > 即使使用大小比较小的S表作为outer loop的主体，最终也要近乎1.1小时

### Block Nested Loop Join

朴素的循环算法没有充分利用缓冲池，由此引出了块嵌套循环

* 假设我们缓冲区大小为B，则我们使用B-2个缓冲区用来装载外表（即外侧遍历的表），用一个缓冲区装载内表，最后一个缓冲区用来输出，写成伪代码如下：

  > ``````python
  > for each page pR(Belong to R) in range(B-2):
  > 	for each page pS(Belong to S) in range(1):
  > 		for each tuple r in pR:
  > 		    for each tuple s in pS:
  >   		        if r matches s then emit
  > ``````

* 这个算法的IO复杂度为$$M + \lceil M / (B-2) \rceil \times N$$

  > **如果外循环可以完全装入内存：**
  >
  > 加入一次IO要0.1ms，那么总时长只要0.15秒

### Index Nested Join Loop

上述两种算法，性能瓶颈在于：对于外循环中的每一个tuple，都需要遍历一次内循环中的tuple来进行判断，因此我们可以使用索引进行优化

* 具体做法为：在关系S的连接属性上建立索引，对于R中的每一个元组，根据索引找到对应的S中元组进行连接
* 假设在索引上查找的代价为C，则IO复杂度为：$$M + m \times C$$

## Sort-Merge Join

如果我们手上的两张表都是有序的，那么join工作就会简单很多；如果他们不是有序的，那就可以考虑使用上节课讲到的排序算法让他们变得有序

* 具体流程如下：

  * Phase #1: Sort

    * 对两张表以join所使用到的key作为关键字进行排序

    * 如果内存放不下这么多page，则需要使用外部排序

  * Phase #2: Merge

    * 对两张排好序的表进行配对，这样外循环的元素就不用每次都遍历一遍内循环的元素了
    * 可能会需要根据join的类型进行回溯，这里具体

    > 比如表R：
    >
    > | id   | name |
    > | ---- | ---- |
    > | 100  | Andy |
    > | 200  | GZA  |
    > | 200  | GZA  |
    >
    > 表S：
    >
    > | id   | value | cdate     |
    > | ---- | ----- | --------- |
    > | 100  | 2222  | 10/4/2023 |
    > | 100  | 9999  | 10/4/2023 |
    > | 200  | 8888  | 10/4/2023 |
    > | 400  | 7777  | 10/4/2023 |
    >
    > 当R表遍历到第二行，而S表遍历到第三行时，会输出这组tuple，S表的指针会向下移动到第四行，此时发现R表中仍然有id=200的tuple存在，则需要进行回溯，将指针退回第三行

* IO开销如下：

  > Sort Cost(R) : $$2M \times (1 + \lceil log_{B-1}{\lceil M / B \rceil} \rceil)$$
  >
  > Sort Cost(S) : $$2N \times (1 + \lceil log_{B-1}{\lceil N / B \rceil} \rceil)$$
  >
  > Merge Cost : $$M + N$$
  >
  > ->Total Cost : Sort + Merge
  >
  > 
  >
  > 例如R有1000页，100000个tuple；S有500页，40000个tuple
  >
  > 则一共需要IO次数7500次，如果每次IO需要0.1ms，则一共需要0.75s

* 排序join算法的最坏情况为：如果两张表的所有tuple都一模一样，那么回溯的次数将会大大增加，开销会来到$$(M \times N) + (sort cost)$$

## Hash Join

* 哈希join算法也有两个阶段：

  * Phase #1: Build

    首先使用哈希函数h1扫描外层表来获取一张哈希表，其中哈希方式可以任选，只不过在实际应用中，线性探查法是效果最好的

  * Phase #2: Probe

    扫描内层表并且使用哈希函数h1来跳转并且寻找匹配的tuple

### Hash Table Contents

对于一张哈希表，我们不仅仅要记录哈希值，还要记录对应的key，以防发生冲突时形成错误的match

此外，有些DBMS还会记录下来tuple的value，这取决于其使用的策略是一开始所提到的Early Materialization 还是Later Materialization

### Optimization : Probe Filter

对于Hash Join的一种常见的优化方式是使用Bloom Filter。这是一种概率性的数据结构，存放在CPU的cache中，Bloom Filter会在创建哈希表的时候判断：这个key是否存在于内层表中，他可能会将不存在误判为存在，但不会将存在的key误判为不存在

这样一来，当我们想要进入哈希表进行查找的时候，我们可以首先访问一下Bloom Filter，可以加快join的速度

### Partition Hash Join

* 有的时候哈希表没有办法装进内存，这个时候我们可以使用聚类哈希join，大概意思就是在build之前就先对输入的relation进行分类，然后针对每一个partition进行前面提到的hash join算法

* Recursive Partition: 关于这一部分其实有点没太看懂，好像是说要用两个哈希函数来对两个表分别进行递归聚类，然后看tuple是否匹配（？）

* IO开销如下：

  > 如果我们不使用递归聚类，那么
  >
  > Partition Phase: $$2(M+N)$$次IO
  >
  > Probe Phase: $$M+N$$次IO

#### Optimization : Hybrid Hash Join

如果一个表中的key是skewed的（大概意思就是这个key在这个表中非常多），那么DBMS会将这个partition视作热点分区，将其保存在内存而非写回磁盘

