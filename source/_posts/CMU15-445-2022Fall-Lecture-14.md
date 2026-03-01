---
title: CMU15-445 2022Fall Lecture 14
date: 2024-02-29 12:22:46
tags:
---
# Lecture 14: Query Planning & Optimization

主要讲了查询的规划以及优化

SQL查询在不同的规划下会有显著的性能上的差异，在先前介绍Join的那一章课已经提到过了

# Query Optimization Techniques

需要注意的是，决定优化的方式、规模与执行查询一样是需要占用时间的，所以查询的优化不在于要做到极致（Query Optimization是一个NP-Hard问题），而是在可能的几个计划之中选取折中的一个

* Heuristics/Rules：

  - Rewrite the query to remove stupid/inefficient things

  - Does not require a cost model

* Cost-based Search
  - Use a cost model to evaluate multiple equivalent plans and pick the one with the lowest cost



## Logical VS. Physical Plans

其实优化查询无非两个步骤：在逻辑上对操作对象进行编排（比如是Join(A,B)还是Join(B,A)，是否要下移谓词等等）、要选择用哪种方式来优化（比如对于nested loop join，是否要更换为nested index join，sort+limit是否要更换为topn算法等等），这就是所谓的逻辑计划与物理计划

## Heuristics/Rules

如果两个关系代数表达式 (Relational Algebra Expressions) 如果能产生相同的 tuple 集合，我们就称二者等价。DBMS 可以通过一些 Heuristics/Rules 来将关系几何表达式转化成成本更低的等价表达式，从而达到查询优化的目的。这些规则通常试用于所有查询，如：

- Predicate Pushdown
- Projections Pushdown

### Predicate Pushdown

Predicate 通常有很高的选择性，可以过滤掉许多无用的数据。将 Predicate 推到查询计划的底部，可以在查询开始时就更多地过滤数据

核心思想如下：

- 越早过滤越多数据越好
- 重排 predicates，使得选择性大的排前面
- 将复杂的 predicate 拆分，然后往下压，如 `X=Y AND Y=3` 可以修改成 `X=3 AND Y=3`

### Replace Cartesian Product

将笛卡尔积替换成Join，不多赘述

### Projections Pushdown

在行存储数据库中，越早过滤掉不用的字段越好，因此将 Projections 操作往查询计划底部推也能够缩小中间结果占用的空间大小

**需要注意的是这种方法对于列存储数据库是不管用的**



## Cost-based Search

除了 Predicates 和 Projections 以外，许多操作没有通用的规则，如 Join：Join 操作既符合交换律又符合结合律，等价关系代数表达式数量庞大，这时候就需要一些成本估算技术，将过滤性大的表作为 Outer Table，小的作为 Inner Table，从而达到查询优化的目的

由于需要遍历各种类型的plan来决定最终的选择，所以这里可以简单划分成三种类型的plan：

* Single relation
* Multiple relations
* Nested sub-queries

### Single relation Query Planning

对于这类查找，只需要用简单的启发式方法譬如顺序扫描、二分搜索、索引扫描等就够用了

具体来说就是将查询拆分成若干个小块，为每一块都生成一个逻辑运算符，为每一个逻辑运算符都生成一组物理运算符，然后依次迭代构建一棵left-deep join tree（就是类似于哈夫曼树一样的结构，两个节点由join作为父节点），这样可以最小化工作量

这里直接去看课上那个Artist、Album、Appears的例子更加直观，整个搜索的过程就是在不断地[枚举、计算代价]

### Multi-relation Query Planning

又分为自顶向下与自底向上两种优化方式，这里重点讲了自顶向下的思路

>  比如一个操作的目标是```A join B join C```，这是我们的目标，也是优化的根节点。接下来需要考虑有哪些方法可以得到这个根节点，比如可以选择```Hash_Join(A join B, C)```或者```SM_Join(A join B, C)```两者之间选择一个代价更小的（这里假设hash join代价更小），然后继续决策有哪些方法可以得到hash join这个节点

### Nested Sub-Queries

有的时候一个查询的嵌套太深，如果对于每一个嵌套都分别优化，而没有一个整体的优化方向的话，最终会导致计划十分低效。因此可以考虑将一个嵌套查询扁平化，或者至少减少其嵌套层数，尽可能水平展开

这就延伸出了两种方法：

* 重写查询，降低语句之间的关联性或者将其扁平化
* 分解嵌套查询，将其结果保存在临时的表中

#### Rewrite

直接举个例子

```mysql
select name from sailors as s
	where exists(
		select * from reservers as r
  		where s.sid = r.sid
  		and r.day = '2022.1.1'
)
```

可以被改写成

```mysql
select name from sailors as s, reservers as r
	where s.sid = r.sid 
		and r.day = '2022.1.1'
```

#### Decomposing Queries

同样举个例子

```mysql
SELECT S.sid, MIN(R.day)
 	FROM sailors S, reserves R, boats B
		WHERE S.sid = R.sid
 			AND R.bid = B.bid
 			AND B.color = 'red'
 			AND S.rating = (SELECT MAX(S2.rating)
 		FROM sailors S2)
	GROUP BY S.sid
		HAVING COUNT(*) > 1
```

可以改写成

```mysql
SELECT MAX(rating) FROM sailors

SELECT S.sid, MIN(R.day)
 	FROM sailors S, reserves R, boats B
		WHERE S.sid = R.sid
 			AND R.bid = B.bid
 			AND B.color = 'red'
 			AND S.rating = (SELECT MAX(S2.rating)
 		FROM sailors S2)
	GROUP BY S.sid
		HAVING COUNT(*) > 1
```

毕竟```(SELECT MAX(S2.rating)```对于整个查询来说只是一个常量

### Cost Estimation

一个查询需要花费多长时间，取决于许多因素 

- CPU: Small cost; tough to estimate
- Disk: #block transfers
- Memory: Amount of DRAM used
- Network: #messages

但本质上取决于：**整个查询过程需要读入和写出多少 tuples**

因此 DBMS 需要保存每个 table 的一些统计信息，如 attributes、indexes 等信息，有助于估计查询成本。值得一提的是，不同的 DBMS 的搜集、更新统计信息的策略不同

### Statistics

对于任意table R，DBMS都保存了关于R的一些相关信息比如：

* $$N_{R}$$：R中tuple的数量
* $$V(A, R)$$：R中A属性的不同取值的个数
* $$A_{max}, A_{min}$$：A属性的最大值和最小值

利用上面这些数据可以得到R中A属性的每一个值所对应的平均记录个数
$$
SC(A, R) =N_R / V(A, R)
$$
利用以上信息，就可以针对不同的predicate，预估不同的selectivity：

* Equality
* Range
* Negation
* Conjunction
* Disjunction

#### Equality Predicate

```mysql
select * from people where age = 2
```

假设people中有5个人，所有的age一共有5个取值，则$$N_R$$=5，$$V(age, people)$$=5：
$$
sel(A = constant) = SC(P) / V(A, R) = \frac{1}{5}
$$

#### Range Predicate

```mysql
select * from people where age >= 2
```

则可以利用最大值最小值来估计：
$$
sel(A >= a) = (A_{max} - a) / (A_{max} - A_{min})
$$

#### Negation Predicate

```mysql
select * from people where age != 2
```

其实就是Equality Predicate取个补集
$$
sel(not P) = 1 - sel(P) = 1 - SC(age = 2) = \frac{4}{5}
$$

#### Conjunc Predicate

```mysql
select * from people 
	where age = 2
		and name like 'A%'
```

如果两个predicate是相互独立的话
$$
sel(P1 \bigwedge P2) = sel(P1) \times sel(P2)
$$

#### Disjunction Predicate

```mysql
select * from people 
	where age = 2
		or name like 'A%'
```

如果两个predicate相互独立，则有：
$$
sel(P1 \bigvee P2) = sel(P1) + sel(P2) - sel(P1 \bigwedge P2) = sel(P1) + sel(P2) - sel(P1) \times sel(P2)
$$
