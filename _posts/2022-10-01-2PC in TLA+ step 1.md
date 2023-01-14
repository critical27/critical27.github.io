---
layout: single
title: 2PC in TLA+, step 1
date: 2022-10-01 00:00:00 +0800
categories: 学习
tags: TLA+ PlusCal Transaction
---

2PC广泛运用于各种分布式系统，然而就是这样一个看似平平无奇的算法，想要正确实现，绝不是一件容易的事情。这个系列应该会陆续写三篇，第一篇主要会在TLA+中描述一个简单的2PC算法。

## Two-phase-commit

我们沿用经典论文Consensus on Transaction Commit里的描述。一个事务有两种类型的参与者：

- Resource Manager: 下面简称RM，负责实际执行事务，所有RM必须就事务是committed或者aborted达成共识。
- Transaction Manager: 下面简称TM，负责推动并决策事务commit或者abort。

然后简单描述下2PC流程，TM给所有RM发送prepare消息，当TM确定所有RM都成功转换成prepared状态后，TM就向所有RM发送commit消息，所有RM收到commit决议后就会进入committed状态。如果TM发现不是所有RM都成功转换成prepared状态，TM则会决定abort事务，向所有RM发送abort消息，所有RM收到abort决议后就会进入aborted状态。

上述流程没有详细说corner case的处理，我们会直接在动手实践中会发现并尝试解决这些case。

## TLA+ specification

### Models

```
variable
    rmState = [rm \in RM |-> "working"],
    \* RM收到的消息队列 FIFO
    rmMsg = [rm \in RM |-> <<>>],
    tmState = [state |-> "init"];
```

为了实现以及避免状态空间爆炸，我们的specification大致模型如下：

1. RM的状态包括5种`{"working", "prepared", "committed", "aborted", "unavailable"}`(本文中都不会用unavailable状态)。每个RM只能查看或者修改自身的状态，而不能查看或者修改其他RM的状态。
2. 每个RM有一个FIFO消息队列用于接收TM发送的消息。
3. TM的状态包括4种`{"init", "commit", "abort", "unavailable"}`(本文中都不会用unavailable状态)。
4. TM会通过每个RM的消息队列发送对应指令，TM可以查看任意RM的状态（但不能修改），并根据RM的状态可能会修改自身状态。

### Definitions

定义一些operator方便后续使用

```
define {
    canCommit == \A rm \in RM : rmState[rm] \in {"prepared"}
    canAbort == \E rm \in RM : rmState[rm] \in {"aborted"}
}
```

- canCommit: 判断一个事务能否commit的条件就是所有RM都处于prepared状态。
- canAbort: 判断一个事务是否需要abort的条件就是存在RM处于aborted状态。

> 其实TM判断是否需要abort还有其他情况，比如RM因为某种原因无法prepared（比如锁冲突、RM挂掉等），本文中暂时不涉及，后面两篇会逐渐完善。
> 

### Invariants

```
Consistency ==
  \A rm1, rm2 \in RM : ~ /\ rmState[rm1] = "aborted"
                         /\ rmState[rm2] = "committed"

TypeOK ==
  /\ rmState \in [RM -> {"working", "prepared", "committed", "aborted", "unavailable"}]
  /\ tmState["state"] \in {"init", "commit", "abort", "unavailable"}

Completed == <> (\A rm \in RM : rmState[rm] \in {"committed","aborted"})
```

两个Invariants:

- Consistency: 所有RM对于事务的最终状态必须达成共识，要么是`committed`，要么是`aborted`。
- TypeOK: 用来检查rmState和tmState都是合法值。

一个Temporal Property:

- Completed: 注意有个`<>`代表eventual，也就是所有RM的状态都变成了`committed`或者`aborted`。

### RM

到这里可以开始实现RM和TM的逻辑了，我们先看RM的状态转换图

![figure]({{'/archive/2PC-step1-1.png' | prepend: site.baseurl}})

- working → prepared: 收到了TM发送的preapre消息
- prepared → committed: 收到了TM发送的commit消息
- prepared → aborted 和 working → aborted: 收到了TM发送的abort消息

根据上面的状态转换，我们开始实现RM，最外层的while循环用来推动RM状态机，当`rmState[self]`变为`committed`或者`aborted`就代表这个事务结束了。

> 由于有多个RM，所以用rmState[self]代表自身这个RM的状态
> 

```
fair process (RManager \in RM)
variables msg = <<>>;
{
RM_MAIN:
    while (rmState[self] \notin {"committed", "aborted"}) {
        recv(msg);
        if (rmState[self] = "working" /\ msg.type = "prepare") {
            rmState[self] := "prepared";
        } else if (rmState[self] = "prepared" /\ msg.type = "commit") {
            rmState[self] := "committed";
        } else if ((rmState[self] = "working" \/
                     (rmState[self] = "prepared" /\ tmState["state"] = "abort"))
                   /\ msg.type = "abort") {
            rmState[self] := "aborted";
        } else {
            assert FALSE;
        }
    }
}
```

然后RM就不断从消息队列开始接收消息，每次收到一个消息之后就将其从队列中移除。（事实上为了简单，不将其从队列中移除也是可以的，一个好处是减少状态搜索空间，另一个是这甚至能模拟网络问题导致消息重发- -b）

```
macro recv(msg) {
    await Len(rmMsg[self]) > 0;
    msg := Head(rmMsg[self]);
    rmMsg[self] := Tail(rmMsg[self]);
}
```

然后根据自身状态和收到的消息类型，依据上面的状态转换图开始转换。最后加的assert FALSE是为了确保不存在其他的状态转换路径和非法消息。

### TM

一个事务只有一个TM，TM做的事情就是给RM发送消息以推动事务。

```
fair process (TManager = 0)
variables idx = 1;
{
TM_PREPARE:
    call broadcast([type |-> "prepare"]);
TM_DECIDE:
    either {
        await canCommit;
        tmState["state"] := "commit";
        call broadcast([type |-> "commit"]);
    } or {
        await canAbort;
        tmState["state"] := "abort";
        call broadcast([type |-> "abort"]);
    }
}
```

首先就是给所有RM发送prepare消息，然后开始根据RM的状态决定事务是`commit`还是`abort`，通过`await canCommit`和`await canAbort`等待RM的条件满足`commit`或者`abort`之后。当满足条件后，TM修改事务的状态，并给所有RM发送消息，最后当RM收到消息后会将自身状态修改为`committed`或者`aborted`。

> 事实上在上面这个TM实现上，是不会abort事务的，主要是因为canAbort只有当存在RM处于aborted状态时才会为真，而只有TM发送abort消息才会使RM转换为aborted状态。我们会在后面几篇中激活这个路径。
> 

> 给所有RM发送消息的实现如下，需要使用procedure是因为macro里面不能包含while循环，只有procedure可以。发送消息本身就是往对应RM的消息队列中添加一个消息。
> 

```

macro send(dst, msg) {
    rmMsg[dst] := Append(rmMsg[dst], msg);
}

procedure broadcast(message) {
TM_B1:
    idx := 1;
TM_B2:
    while (idx <= Cardinality(RM)) {
        send(idx, message);
        idx := idx + 1;
    };
    return;
}
```

## Model checking

![figure]({{'/archive/2PC-step1-2.png' | prepend: site.baseurl}})

我们设置有三个RM，然后添加对应的Invariants和Temporal Properties，然后就可以运行了，在这个模型下其实状态很少，只有212个。主要原因就是不论是RM还是TM都不会出现crash的情况。在后面的几篇中，我们会逐步的模拟真实场景，不断完善这个2PC算法。

![figure]({{'/archive/2PC-step1-3.png' | prepend: site.baseurl}})

## Reference

- <https://lamport.azurewebsites.net/video/videos.html>
- <https://www.microsoft.com/en-us/research/uploads/prod/2004/01/twophase-revised.pdf>
- <http://muratbuffalo.blogspot.com/2017/12/tlapluscal-modeling-of-2-phase-commit_14.html>
