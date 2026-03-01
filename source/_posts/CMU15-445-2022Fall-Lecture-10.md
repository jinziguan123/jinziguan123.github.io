---
title: CMU15-445 2022Fall Lecture 10
date: 2024-02-28 12:48:49
tags: CMU15-445
categories: 笔记
---
# Lecture 10: Sorting & Aggregation Algorithms

本章主要介绍了几种排序与聚合算法的思想

# Sorting

在关系型数据库中，tuple之间并没有特定的顺序关系，因此在进行诸如group by, distinct, partition by, order by, join之类的操作时，需要对tuple进行排序
如果内存足够放下这些数据，那么使用qsort进行排序即可；然而内存未必能够放下这么多的tuple，因此在这种情况下需要使用external sorting，能够根据需要将排序溢出到磁盘，并且倾向于顺序I/O

## Top-N Heap Sorting

* 如果一个查询包含一个带有limit的order by，则DBMS只需要查找到前N个元素即可，也就是使用Top-N Heap Sort。理想情况是将前N个元素放入内存，这样DBMS只需要维护一个优先队列即可

> 例如我们有以下查询：```select * from enrolled order by sid fetch first 4 rows with ties```，我们需要输出学号前四小的学生信息，如果学号相同，则并列输出。排序方法如下：
>
> * 首先创建大小为4的堆
> * 遍历数据，如果堆未满，则直接放入堆；否则，如果fetch指向的数据已经在堆中，则将堆的大小扩大到两倍（因为要求with ties）并且放入堆；如果该数据不在堆中，则判断是否是我们需要的数据：如果是，则将堆首的相同元素全部pop，放入堆；如果不是，则跳过该fetch

## External Merge Sort

### Two-way Merge Sort

* 最基本的两路归并排序：在排序阶段从磁盘中读取page到内存，排序结束后写回磁盘；在合并阶段，他使用三个缓冲页——从磁盘中读取两个page到其中两个frame，并且将其合并到第三个frame，每当第三个frame被填满，就会将其写回磁盘，并且替换成一个空的page。其中每一组排好序的page被称为一个run，每一次遍历称为一个pass
* 如果参与排序的page数量为N，则该算法一共进行了$$1+\lceil\log_2N\rceil$$次遍历，总IO成本为$$2N\times (pass nums)$$

### General（K-way） Merge Sort

* 该算法的一般版本允许DBMS使用三个以上的缓冲页

  B表示缓冲页总数，在排序阶段，一次可以读取B个page，并且将$$\lceil \frac{N}{B} \rceil$$个排好序的run写回磁盘；在合并阶段，可以在每个通道中合并最多B-1个runs，将结果放入最后一个缓冲页并且写回磁盘

* 在一般的版本中，一共需要遍历$$1 + \lceil log_{B-1} \lceil \frac{N}{B} \rceil  \rceil$$次排序，总IO开销依然是$$2N\times (pass nums)$$

### Double Buffering Optimization

* 优化思想是：假设我们从原本有的4个缓冲页扩大到了8个，那么如果还是按照原本3+1的思路进行串行的归并排序和写回，就会浪费一部分的IO请求等待时间。双缓冲优化使用多线程，组成两个3+1：当第一组缓冲区正在进行归并排序的时候，第二组缓冲区已经开始从磁盘中预读取page到frame中，这样一旦第一组IO结束，即可马上开始下一组IO的运算

### Using B+ Tree

* 对于DBMS，可以使用聚类B+索引来帮助排序，因为在B+树的叶子结点中，所有数据的存储都是有序的，IO的访问也都是顺序的，可以有效减小计算开销，这比外部归并排序要来的高效
* 反之，如果是非聚类的B+索引，那么使用它就不是一个很好的选择（因为数据不连续），几乎所有访问都要从磁盘中读取而非buffer pool

# Aggregation

在执行聚合运算的时候，往往是将一个或者多个tuple的值折叠成一个标量值，例如```select distinct cid from enrolled where grade in ('B','C) order by cid```

总体来说有两种实现聚合的方法：(1)排序，(2)散列

## Sorting

* DBMS首先根据group by语句对tuple进行排序，如果buffer pool够用，则直接使用qsort，否则使用外部归并算法。然后DBMS会对排好序的数据进行顺序扫描来计算聚合
* 在执行排序聚合，重要是要对查询操作进行排序，以效率最大化。例如：如果查询需要一个过滤器，最好先执行过滤器，然后对过滤后的数据进行排序，以减少数据量

## Hashing

* 由于在计算聚合的时候，散列的计算成本总是要比排序低得多，所以DBMS在扫描表的时候会填充一个临时的哈希表（ephemeral hash table），对于每一条记录，都会检查哈希表中是否已经有该条目，并且进行适当的修改。如果哈希表太大，DBMS就会将其溢出到磁盘上（？）。完成这个任务有两个阶段：

  * Phase #1 - Partition：

    使用哈希函数h1，根据目标的hash key将tuple分割到不同的磁盘分区，这样所有匹配的tuple都会被分配到同一片区域，然后DBMS会通过输出缓冲区将分区溢出到磁盘上

  * Phase #2 - ReHash：

    对于磁盘上的每一个分区，将其page读入内存，并且根据第二个哈希函数h2，建立一个哈希表，由此把所有匹配的tuple都聚集到一起。如果hash表太小了以至于无法容纳所有数据，那么可以考虑将当前分区重新进行分割，或者混杂其他基于排序或者基于散列的算法

* 在ReHash阶段，DBMS可以将需要输出的聚合存储为（GroupByKey -> RunningValue）的配对，RunningValue的内容取决于聚合函数

  > 例如如果我们想要输出```select cid, AVG(gpa) from enrolled```，可以将AVG存储为(COUNT,SUM)的形式

* 而对于一个新插入的tuple

  * 如果它能够找到一个匹配的GroupByKey，则适当更新RunningValue
  * 否则插入一个新的(GroupByKey -> RunningValue)

