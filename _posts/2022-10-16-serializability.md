---
layout: single
title: Conflict Serializability and View Serializability
date: 2022-10-16 00:00:00 +0800
categories: 学习
tags: TLA+ PlusCal Transaction
---

最近在看SnapshotIsolation的形式化验证，学习到了两个新概念`Conflict Serializability`和`View Serializability`。回顾下可串行化：多个事务执行的结果和这些事务按某种串行执行的结果相同。然后我们分别看下这两种串行化的概念。

## **Conflict Serializable**

首先需要明白多个事务中的操作是否有冲突，如果满足以下所有条件，我们就说两个操作有冲突：

- 属于不同事务
- 操作同一条数据
- 其中至少有一个操作是写操作

> 比如，R1(X)和W2(X)是冲突的，但R1(X)和W2(Y)不是冲突的，R1(X)和W1(X)也不是冲突的，R1(X)和R2(X)也不是冲突的
>

如果能够通过交换事务执行历史中不冲突的操作，使得事务的执行顺序是串行的，那么我们就说这个事务执行历史是满足`Conflict Serializable`。我们看个例子：

| T1   | T2   |
| ---- | ---- |
|      | R(X) |
| R(X) |      |
|      | R(Y) |
|      | W(Y) |
| R(Y) |      |
| W(X) |      |

我们可以执行如下操作

1. 交换T1中的R(X)和T2中的R(Y)
2. 交换T1中的R(X)和T2中的W(Y)

| T1   | T2   |
| ---- | ---- |
|      | R(X) |
|      | R(Y) |
|      | W(Y) |
| R(X) |      |
| R(Y) |      |
| W(X) |      |

交换后的结果如上所示，这样的执行历史就和T2 → T1的执行顺序相同，所以我们说这个事务执行历史满足`Conflict Serializable`。它还有如下性质：

1. `Conflict Serializable`和事务依赖顺序图中没有环是充要条件。这是因为两个事务之间如果存在冲突操作，一定有执行先后顺序要求。而如果事务依赖顺序图中出现了环，就说明两个事务无法定序，也就不能串行执行两个事务，所以无法满足`Conflict Serializable`。

    > 但是是不是出现环了，就一定不能满足`Serializable`呢？答案是否定的，后面会解释。
    >
2. 如果满足`Conflict Serializable`，就一定会满足`View Serializable`。

## View Serializable

大多数的论文或者实现中说的可串行化，基本上是以`Conflict Serializable`为主，为什么需要另外一种可串行化的概念呢？主要原因是在于当事务依赖中出现环之后，就无法判断是不是满足`Serializable`了。

### **View Equivalent**

首先需要理解什么叫`View Equivalent`。我们说两个事务执行历史满足如下三个条件就满足`View Equivalent`：

![figure]({{'/archive/SI-1.png' | prepend: site.baseurl}})

1. Initial Read
    1. 两个事务执行历史中，第一次读必须是同一个事务（上图中都是T1执行第一个读）
    2. 两个事务执行历史中，必须读同一个数据（上图中都是读A）
    3. 所以上面的两个历史满足Initial Read
2. Updated Read
    1. 两个事务执行历史中，如果读数据A，那么数据A必须是由同一个事务更新的
    2. 比如上图中，两个历史都在T3进行Read(A)，但是历史1中是由T2更新的，而历史2中则是由T1更新的，因此不满足Updated Read
3. Final Write
    1. 两个事务执行历史中，最后一次必须是同一个事务（上图中都是T3执行最后一次写）
    2. 两个事务执行历史中，必须写同一个数据（上图中都是写A）
    3. 所以上面的两个历史满足Final Write

如果事务执行历史和串行执行`View Equivalent`，我们就说这个事务执行历史是`View Serializable`。

我们再看一个例子，S的这个执行历史中，事务依赖中出现了环，不满足`Conflict Serializable`。那我们看能否满足`View Serializable`：执行历史S1满足串行执行，那么S和S1是否`View Equivalent`？

![figure]({{'/archive/SI-2.png' | prepend: site.baseurl}})

我们依次检查三个条件：

1. Initial Read：都是T1执行Read(A)，满足
2. Final Write: 都是T3执行Write(A)，满足
3. Updated Read: 不存在，满足

所以是满足`View Serializable`的。

## 如何去验证

鉴于我们通常在说可串行的时候，多数都是`Conflict Serializable`。所以我们尝试在TLA+中来验证一下快照隔离能否满足`Conflict Serializable`。首先我们大概看下事务执行历史的样子，假设一开始数据库为空：

```
WriteSkewAnomalyTest == <<
    [type |-> "begin",  txnId |-> 1, time |-> 1],
    [type |-> "begin",  txnId |-> 2, time |-> 2],
    [type |-> "read",   txnId |-> 1, key |-> "X", val |-> "Empty"],
    [type |-> "read",   txnId |-> 1, key |-> "Y", val |-> "Empty"],
    [type |-> "read",   txnId |-> 2, key |-> "X", val |-> "Empty"],
    [type |-> "read",   txnId |-> 2, key |-> "Y", val |-> "Empty"],
    [type |-> "write",  txnId |-> 1, key |-> "X", val |-> 30],
    [type |-> "write",  txnId |-> 2, key |-> "Y", val |-> 20],
    [type |-> "commit", txnId |-> 1, time |-> 3, updatedKeys |-> {"X"}],
    [type |-> "commit", txnId |-> 2, time |-> 4, updatedKeys |-> {"Y"}]>>
```

上面是一个写偏序的例子，T1和T2都读了”X”和”Y“，然后分别更新”X”和”Y“然后提交。我们后面的说明都是基于类似的事务执行历史来验证是否满足可串行化。然后定义了一些辅助operator：

![figure]({{'/archive/SI-3.png' | prepend: site.baseurl}})

1. `BeginOp`和`CommitOp`用来在事务执行历史中找到特定事务的begin和commit操作
2. `CommittedTxns`和`AbortedTxns`用来在事务执行历史中找到commit或者abort的事务
3. `ReadsByTxn`和`WritesByTxn`用来在事务执行历史中找到特定事务的读集合或者写集合

### 冲突的定义

首先是三种冲突：写写冲突、写读冲突和读写冲突，详细可参见[原始论文的2.5.1小节](https://ses.library.usyd.edu.au/bitstream/2123/5353/1/michael-cahill-2009-thesis.pdf)。下面是摘抄的核心描述：

```
* T1 produces a version of x, and T2 produces a later version of x 
  (this is a ww-dependency);
* T1 produces a version of x, and T2 reads this (or a later) version of x
  (this is a wr-dependency);
* T1 reads a version of x, and T2 produces a later version of x
 (this is a rw-dependency, also known as an anti-dependency, and is the only
  case where T1 and T2 can run concurrently).
```

对应实现如下：

- 写写冲突检测两个事务T1和T2是否写入同一个key，且`commit_ts(T1) < commit_ts(T2)`
- 写读冲突检测是否存在T1先写入一个key，之后T2读同一个key，且`commit_ts(T1) < begin_ts(T2)`
- 读写冲突检测是否存在T1先读一个key，之后T2写同一个key，且`begin_ts(T1) < commit_ts(T2)`

> 原始论文中提到，只有读写冲突时可以并发的，所写写冲突这里并没有检测T1和T2是否时间上有overlap。
>

![figure]({{'/archive/SI-4.png' | prepend: site.baseurl}})

### Conflict Serializable

定义好冲突之后，我们来看如何定义`Conflict Serializable`。根据其定义，我们只需要在MVSG中（可以理解为事务依赖画成的有向图，图中的每一条边都是前面三种冲突中的一种）是否存在环即可）。所以事务间的冲突我们可以如下描述，也就是检查任何两个已提交事务是否存在写写冲突、写读冲突或者读写冲突。当然，检测是否满足`Conflict Serializable`，就是检查`SerializationGraph`中是否存在环。

![figure]({{'/archive/SI-5.png' | prepend: site.baseurl}})

下面判环的这个函数是从[这里](https://github.com/pron/amazon-snapshot-spec/blob/master/serializableSnapshotIsolation.tla)借鉴~~(抄)~~过来的

> 至今不太会在TLA+里面写递归，说不出的诡异
>

![figure]({{'/archive/SI-6.png' | prepend: site.baseurl}})

### Invariant

![figure]({{'/archive/SI-7.png' | prepend: site.baseurl}})

最后终于到了Invariant: `IsConflictSerializable`用来检查给定的事务执行历史是否满足`Conflict Serializable`，方式就是前面提到的在MVSG中判断是否存在环。

> `ReadOnlyAnomaly`是用来验证在快照隔离级别下，只读事务其实也可能违反可串行化。原始论文在[这里](https://www.cs.umb.edu/~poneil/ROAnom.pdf)。具体我们下一篇再介绍，这里只解释下`ReadOnlyAnomaly`的实现，它用来检查如下情况是否会出现：如果一个事务执行历史不满足`Conflict Serializable`，但是去掉了一个只读事务后反而满足`Conflict Serializable`。

### 验证时间

```
\* 写偏序 不多介绍
WriteSkewAnomalyTest == <<
    [type |-> "begin",  txnId |-> 1, time |-> 1],
    [type |-> "begin",  txnId |-> 2, time |-> 2],
    [type |-> "read",   txnId |-> 1, key |-> "X", val |-> "Empty"],
    [type |-> "read",   txnId |-> 1, key |-> "Y", val |-> "Empty"],
    [type |-> "read",   txnId |-> 2, key |-> "X", val |-> "Empty"],
    [type |-> "read",   txnId |-> 2, key |-> "Y", val |-> "Empty"],
    [type |-> "write",  txnId |-> 1, key |-> "X", val |-> 30],
    [type |-> "write",  txnId |-> 2, key |-> "Y", val |-> 20],
    [type |-> "commit", txnId |-> 1, time |-> 3, updatedKeys |-> {"X"}],
    [type |-> "commit", txnId |-> 2, time |-> 4, updatedKeys |-> {"Y"}]>>
```

然后在TLC中检查`IsConflictSerializable(WriteSkewAnomalyTest)`，结果是FALSE

![figure]({{'/archive/SI-8.png' | prepend: site.baseurl}})

## 写在最后

这篇文章介绍了`Conflict Serializable`和`View Serializable`，然后简单的说明了下如何在TLA+中来验证事务执行历史能否满足`Conflict Serializable`。我们下一篇会在这一篇的基础上，实现一个快照隔离，并实际的来检测快照隔离能否满足可串行化。

## Reference

[ROAnomSIGMREC1 (umb.edu)](https://www.cs.umb.edu/~poneil/ROAnom.pdf)

[https://ses.library.usyd.edu.au/bitstream/2123/5353/1/michael-cahill-2009-thesis.pdf](https://ses.library.usyd.edu.au/bitstream/2123/5353/1/michael-cahill-2009-thesis.pdf)

[https://github.com/will62794/snapshot-isolation-spec](https://github.com/will62794/snapshot-isolation-spec)

[https://github.com/pron/amazon-snapshot-spec](https://github.com/pron/amazon-snapshot-spec)
