---
layout: single
title: CockroachDB StoreLiveness
date: 2026-08-14 00:00:00 +0800
categories: 学习
tags: TLA+
---

CockroachDB 在 SIGMOD 2026 发了一篇论文《Scalable Leader Leases for Multi-Consensus Groups in CockroachDB》，这一篇主要介绍其核心机制 StoreLiveness，后面有机会再完整梳理一遍论文。

## StoreLiveness

论文要解决的核心问题是：在一个节点同时参与大量 raft group 的场景下，如何让 leader lease 开销低且具备严格的安全性。如果每个 raft group 都独立维护一套租约确认机制，那么每个 raft group 中的心跳和租约续约的开销会随着 raft group 增多而变大。

CockroachDB 的做法是不再直接让每个 raft group 的 leader 来发送心跳以及租约续约的消息，而是直接通过节点和节点之间的心跳来完成。考虑到一个节点上会有多个 raft group，这样做的好处显而易见：节点之间的心跳可以被许多共识组复用，从而把大量心跳和 lease 的维护成本摊薄。

节点间的心跳，也就是论文中提到的 `Liveness Fabric`，为上层 raft 计算 leader lease 的过期时间提供了基础。我们会结合论文中的 TLA+ 形式化验证代码，来介绍其具体原理，注意到在 TLA+ 代码中 `Liveness Fabric` 被称为 `StoreLiveness`。

> 之所以叫 `StoreLiveness`，是指一个节点中可以拥有多个 `Store`，`Store` 之间会发送心跳。但和论文一样，TLA+ 中的代码都将这个模型简化为一个节点只有一个 `Store`，即变成了我们所提到的节点之间的心跳。

我们首先可以先建立这样一个心智模型：

- 集群中的每个节点都可以对其他节点表达“支持”
- 这份支持不是永久的，而是带有一个过期时间
- 上层的 Raft leader lease 会把这份支持作为可以继续持有 leader lease 的依据，从而避免每次读请求都重新走一遍一致性协议

> PS: 节点级别的支持是双向存在的，即同一时刻 `A` 可能支持 `B`，且 `B` 也支持 `A`。比如两个三副本分片 Range1 和 Range2 都在节点 A/B/C 上，且 Range1 的 leader 是 `A`，Range2 的 leader 是 `B`。此时 `B` 支持 `A`，代表 `A` 可以把这个节点级别的支持用于 Range1 的 leader lease 上。与此同时 `A` 支持 `B` 代表 `B` 可以把这个节点级别的支持用于 Range2 的 leader lease 上。

`StoreLiveness` 的核心性质就是系统需要满足 `Support Disjointness invariant`，即两个节点之间的多次支持不能在时间上有重叠：

```text
  \* The Support Disjointness invariant states that, for any requester and supporter,
  \* no two support intervals overlap in time; i.e. a node does not receive support
  \* for a new epoch while support for the previous epoch is still valid.
```

也就是说：
- 一个节点一旦承诺在某个截止时间之前支持另一个节点，就不能在此之前撤销支持，或者转而支持新的节点
- 当旧的一轮支持还没有过期时，新的一轮支持不能提前生效

> 全文中“支持”和 `support` 根据中英文语境不同，可能会穿插使用但含义一致。

## 模型

下面结合 TLA+ 代码来介绍 `StoreLiveness`。

### 节点状态

一个系统中可以包含多个节点，每个节点称为一个 `Store`，每个 `Store` 中包含如下状态：

```text
variables
  max_epoch     = [i \in Nodes |-> 1];
  max_requested = [i \in Nodes |-> 0];
  max_withdrawn = [i \in Nodes |-> 0];
  support_from  = [i \in Nodes |-> [j \in Nodes \ {i} |-> [epoch |-> 1, expiration |-> 0]]];
  support_for   = [i \in Nodes |-> [j \in Nodes \ {i} |-> [epoch |-> 0, expiration |-> 0]]];
  clocks        = [i \in Nodes |-> 1];
  network       = [i \in Nodes |-> EmptyNetwork];
```

这些状态大致可以分成三类。

#### 记录节点之间的支持

节点之间的每次支持，都会对应一个自增的 `epoch`。相关的状态如下：

- `max_epoch[i]`：节点之间交换信息时，节点 `i` 当前感知到的最大 `epoch`
- `support_from[i][j]`：节点 `i` 记录的“自己从节点 `j` 获得的支持”，如果当前没有获得节点 `j` 的支持，则对应 `expiration` 为 0
- `support_for[i][j]`：节点 `i` 记录的“自己给节点 `j` 提供的支持”，如果当前没有支持节点 `j`，则对应 `expiration`为 0

`support_from` 和 `support_for` 正好对应支持关系的两侧：前者是“我认为别人支持我”，后者是“我认为我支持别人”。其中每一条 `support` 记录都包含两个字段：

- `epoch`
- `expiration`：这份支持的过期时间 （逻辑时钟，和 `epoch` 无关）

#### 时间相关状态

每份支持，都有对应的过期时间，采用逻辑时钟（Lamport clock）。相关状态如下：

- `clocks[i]`：节点 `i` 本地的逻辑时间戳
- `max_requested[i]`：节点 `i` 请求在任意时刻请求任意节点的支持时携带过的最大过期时间戳，即节点 `i` 目前要求的支持中最大过期时间
- `max_withdrawn[i]`：节点 `i` 撤销支持时的最大时间戳

这几个时间相关变量的作用主要是防止“重启后时间倒退”或者“新旧支持时间发生重叠”。

> 注意前面说的 `epoch` 更像是用来支持的一个版本号，只在重启、新旧支持切换时涉及。而逻辑时钟主要用于判断支持是否过期，以及逻辑时钟更新，即 Clock propagation。

#### 通信相关状态

- `network`：网络中的消息集合或消息队列

这里的网络模型是可配置的：

- 当 `AllowMsgReordering = TRUE` 时，消息可以乱序到达
- 否则消息按 FIFO 顺序到达

### 判断节点的支持状态

接下来是 `define` 部分，这里会定义一些 operator （可以理解为函数），方便后续模型验证调用，这里挑几个关键的加以解释。

#### 节点之间的支持是否成立

```text
  EpochValid(map, i, j) == map[i][j].expiration /= 0
  \* Has i ever received support from j for the current epoch?
  SupportFromEpochValid(i, j) == EpochValid(support_from, i, j)
  \* Has i ever supported j for the current epoch?
  SupportForEpochValid(i, j) == EpochValid(support_for, i, j)
```

* `SupportFromEpochValid(i, j)`: 在节点 `i` 视角，是否从节点 `j` 处获取过支持？
* `SupportForEpochValid(i, j)`: 在节点 `i` 视角，是否支持过节点 `j`？

由于这里只检查了 `support_from` 和 `support_for` 中的 `expiration` 字段，因此强调的是在历史记录中是否支持过或者被支持过。另外，`support_from` 和 `support_for` 都只保存支持方或者被支持方的记录，并不能确保支持成立。

只有双方关于支持的 epoch 一致，才能认为支持成立，即 `SupportUpheld`：

```text
  \* Is support for i from j upheld?
  SupportUpheld(i, j) == support_from[i][j].epoch = support_for[j][i].epoch
```

`SupportUpheld(i, j)` 的含义是：节点 `i` 认为从节点 `j` 获得的支持，与节点 `j` 提供节点 `i` 的支持，是否属于同一个 `epoch`。如果是，则这一轮支持成立，否则不成立。

#### 节点之间的支持是否过期

```text
  EpochSupportExpired(map, i, j, supporter_time) == map[i][j].expiration <= supporter_time
  \* Is i's support from j (according to i's support_from map) expired (according to j's clock)?
  SupportFromExpired(i, j) == EpochSupportExpired(support_from, i, j, clocks[j])
  \* Is i's support for j (according to i's support_for map) expired (according to i's clock)?
  SupportForExpired(i, j) == EpochSupportExpired(support_for, i, j, clocks[i])
```

这里需要注意是按谁的时钟判断过期：

- `SupportFromExpired(i, j)`：按支持方 `j` 的时钟判断，`i` 从 `j` 获得的支持是否已经过期
- `SupportForExpired(i, j)`：按支持方 `i` 的时钟判断，`i` 给 `j` 的支持是否已经过期

即支持什么时候失效，最终要以支持方的时间视角为准。

#### 节点之间的支持能否撤销

只有当节点 `i` 之前确实支持过节点 `j`，并且这份支持按 `i` 自己的时钟已经过期时，`i` 才允许撤销这份支持。
```text
  \* Can i withdraw support for j?
  CanInvalidateSupportFor(i, j) == SupportForEpochValid(i, j) /\ SupportForExpired(i, j)
```

也就是说，支持方不能在承诺的支持期限内反悔。

### Invariants

`StoreLiveness` 要满足 `Support Disjointness invariant`，即给定两个节点的多次支持不能有时间上的重叠。在 TLA+ 中对这样的安全性检查主要 invariant 来约束，当后续模型检查时，会对状态空间中的所有状态进行检查，只要任意时刻任意一个 invariant 不满足，TLA+ 的 model checker 就会输出系统是通过怎样的路径到达了非法状态。

代码中将 `Support Disjointness invariant` 拆成了两部分 `DurableSupportInvariant` 和 `SupportProvidedLeadsSupportAssumedInvariant` (对应论文中的 4.1 小结)，其余几个 invariant 都是由实现直接约束的。

* DurableSupportInvariant

```text
  \* If we ever had support for the current i=>j epoch, then either support
  \* is still upheld or the support we have received had expired according to
  \* j's clock.
  \*
  \* Durable support is the central safety property of the algorithm.
  DurableSupportInvariant ==
    \A i \in Nodes:
      \A j \in Nodes \ {i}:
        SupportFromEpochValid(i, j) =>
          (SupportUpheld(i, j) \/ SupportFromExpired(i, j))
```

`StoreLiveness` 核心的安全性质。

它表达的是：如果节点 `i` 从节点 `j` 获得了支持”，那么一定满足下面两者之一：

- 要么这份支持现在仍然成立，即 `SupportUpheld(i, j)`
- 要么这份支持在支持方 `j` 的时间视角下已经过期，即 `SupportFromExpired(i, j)`

它要防止的情况是：被支持方还以为支持仍然有效，但支持方其实已经悄悄撤销了支持，甚至转而支持别的节点。只有确保这种情况不会发生，上层 leader lease 的正确性才有保障。

* SupportProvidedLeadsSupportAssumedInvariant

```text
  \* If support for i from j is provided in en epoch, the end time of support
  \* known to the supporter (j) must be greater than or equal to the end time
  \* of support known to the supportee (i).
  \*
  \* This is a structural invariant in the algorithm used to provide safety.
  SupportProvidedLeadsSupportAssumedInvariant ==
    \A i \in Nodes:
      \A j \in Nodes \ {i}:
        SupportUpheld(i, j) =>
          support_from[i][j].expiration <= support_for[j][i].expiration
```

这一条比较容易理解：如果一份支持当前成立，那么支持方记录的过期时间，必须大于等于被支持方记录的过期时间。否则就会出现被支持方还以为支持有效、而支持方已经认为它过期的情况。

* CurrentEpochLeadsSupportedEpochsInvariant

```text
  \* A node's current epoch leads its supported epoch by all other nodes.
  \*
  \* This is a structural invariant in the algorithm used to provide safety.
  CurrentEpochLeadsSupportedEpochsInvariant ==
    \A i \in Nodes:
      \A j \in Nodes \ {i}:
        max_epoch[i] >= support_from[i][j].epoch
```

由于节点接受消息时，会更新本地的 `max_epoch`。因此，每个节点当前已知的 `max_epoch`，必须大于等于 `support_from` 中记录的那些 `epoch`。由下面算法实现直接保证。

* WithdrawnSupportMinimumEpochInvariant

```text
  \* The minimum epoch assigned to store liveness support after support has
  \* been withdrawn from a prior epoch leads the supportee's support_from epoch
  \* by exactly 1.
  \*
  \* This is a structural invariant in the algorithm used to provide safety.
  WithdrawnSupportMinimumEpochInvariant ==
    \A i \in Nodes:
      \A j \in Nodes \ {i}:
        (support_for[i][j].epoch > support_from[j][i].epoch /\ support_for[i][j].expiration = 0) =>
          support_for[i][j].epoch = support_from[j][i].epoch + 1
```

这一条约束的是“撤销之后如何进入下一轮”。

如果节点 `i` 之前支持过 `j`，之后把这份支持撤销了，那么：

- `support_for[i][j].expiration = 0` 表示旧支持已经被清空
- 同时新的 `epoch` 必须正好比对方已知的旧 `epoch` 大 1

主要作用是支持撤销之后的中间状态，以及新旧支持的 `epoch` 必须连续，由下面算法实现直接保证。

## Algorithm

整个算法分为几部分，我们先理解每个节点能执行的操作，再看节点主循环，会比较容易把整个脉络串起来。

### 节点的操作

每个节点能执行的操作有（对应 TLA+ 中的 macro 部分）：

#### 发送心跳

心跳的含义是：发送方向接收方请求在某个 `epoch` 上继续支持我，直到过期时间 `expiration`。

> 注意心跳的发送方对应被支持方，心跳的接收方对应支持方。下面可能会混用。

```text
macro forward(clock, time)
begin
  if clock < time then
    clock := time;
  end if;
end macro

macro send_heartbeat(to)
begin
  with interval \in HeartbeatIntervals do
    forward(max_requested[self], clocks[self] + interval);
    send_msg(to, [
      type       |-> MsgHeartbeat,
      from       |-> self,
      epoch      |-> support_from[self][to].epoch,
      expiration |-> max_requested[self],
      now        |-> clocks[self]
    ]);

  end with;
end macro
```

`forward(max_requested[self], clocks[self] + interval)` 保证本次请求的支持过期时间一定大于当前本地逻辑时钟。

心跳的主要字段为：

- `type`：消息类型
- `from`：心跳发送方节点 id
- `epoch`：被支持方本地 `support_from` 中记录的 `epoch`
- `expiration`：请求对方支持到这个时间点
- `now`：心跳发送方当前的逻辑时间

#### 返回心跳响应

心跳请求是“被支持方希望对方如何支持自己”，而心跳响应返回的是“支持方自己记录的支持状态”。

```text
macro send_heartbeat_resp(to)
begin
  send_msg(to, [
    type       |-> MsgHeartbeatResp,
    from       |-> self,
    epoch      |-> support_for[self][to].epoch,
    expiration |-> support_for[self][to].expiration,
    now        |-> clocks[self]
  ]);
end macro
```

心跳响应中的主要字段为：

- `type`：消息类型
- `from`：心跳接受方节点 id
- `epoch`：支持提供方的 `support_for` 中的 `epoch`，即支持方愿意在哪个 `epoch` 给予支持
- `expiration`：支持提供方的 `support_for` 中的 `expiration`，即支持方至少提供支持到这个逻辑时间戳
- `now`：心跳接收方当前的逻辑时间

#### 接收消息

不论是心跳还是心跳响应，二者都是一种消息，需要节点先接收对应消息，然后再加以处理。

接受消息时，如果允许乱序，则接收方可以每次处理任意一条未处理消息，如果不允许乱序，则按 FIFO 处理先收到的消息。

不论怎样从网络中取哪条消息，都需要根据消息中的逻辑时间戳，更新本地时间（`forward(clocks[self], msg.now)`）。

```text
macro recv_msg()
begin
  if AllowMsgReordering then
    with recv \in network[self] do
      network[self] := network[self] \ {recv};
      msg := recv;
    end with;
  else
    msg := Head(network[self]);
    network[self] := Tail(network[self]);
  end if;
  \* Clock propagation is necessary for the Support Disjointness Invariant.
  forward(clocks[self], msg.now);
end macro
```

#### 重启

```text
macro restart()
begin
  if AllowClockRegressionOnRestart then
    clocks[self] := Max({max_withdrawn[self], max_requested[self], clocks[self] - 1});
  else
    clocks[self] := Max({max_withdrawn[self], max_requested[self], clocks[self]});
  end if;
  max_epoch[self]    := max_epoch[self] + 1;
  support_from[self] := [j \in Nodes \ {self} |-> [epoch |-> max_epoch[self], expiration |-> 0]];
end macro
```

节点重启后会做三件事：

- 保证自己的时钟不会回退到 `max_withdrawn` 或 `max_requested` 之前
- 将 `max_epoch` 自增，表示从一个新的轮次重新开始
- 把 `support_from` 重置到新的 `epoch`，并清空已有的支持过期时间，代表不再承认之前的支持

注意这里清空了 `support_from`，可以理解为该节点不再信任之前所获取的所有支持，需要用新的 `epoch` 重新获取支持。

而另一方面，重启并没有清空 `support_for`，这是因为该节点之前给于其他节点的支持可能仍处于有效时间之内，并不能随意撤销，否则会违反 `DurableSupportInvariant`。

### 节点主循环

有 `Nodes` 个节点，每个节点都是一个状态机。下面这段伪代码可以看成是每个节点不断执行的主循环：

> 为了方便理解，这里对原有的 TLA+ 代码进行了一些简化

```text
begin Loop:
  while TRUE do
    either
      TickClock:
        clocks[self] := clocks[self] + 1;
    or
      TickClockAndSendHeartbeats:
        clocks[self] := clocks[self] + 1;
        with i \in Nodes \ {self} do
          send_heartbeat(i);
        end with;
    or
      Restart:
        restart();
    or
      WithdrawSupport:
        withdraw();
    or
      recv_msg();

      if msg.type = MsgHeartbeat then
        ReceiveHeartbeat:
          \* 处理心跳...
          \* 返回心跳响应...
      elsif msg.type = MsgHeartbeatResp then
        ReceiveHeartbeatResp:
          \* 处理心跳响应...
      else
        assert FALSE;
      end if;
    end either;
  end while;
```

在一次循环里，一个节点可能执行的动作包括：

1. `TickClock`：推进本地时钟
2. `TickClockAndSendHeartbeats`：推进时钟后给其他节点发送心跳
3. `Restart`：节点重启
4. `WithdrawSupport`：当某个支持已经按本地时钟过期，撤销这个支持
5. `recv_msg()`：取出并处理一条消息。如果是心跳消息，则处理心跳并返回心跳响应。如果是心跳响应消息，则处理该响应。

这里的 `either ... or ...` 很重要，它表示只要某个分支满足条件，模型检查就会把这个分支纳入状态空间。不论有几个分支成立，虽然每次运行时只能走其中一个分支，但最终模型检查会确保所有可能的状态空间都能够覆盖，从而验证 invariant 是否始终成立。

主循环中，我们重点关注处理心跳和心跳响应以及撤销支持的几部分，其余已经在前面有所覆盖。

#### 撤销支持

当节点发现当前给于其他节点的支持，对比本地逻辑时钟已经过期，则可以撤销支持。对应前面的 `WithdrawnSupportMinimumEpochInvariant`：

```text
      WithdrawSupport:
        with expired \in CanInvalidateSupportForSet(self) do
          support_for[self][expired].epoch      := support_for[self][expired].epoch + 1 ||
          support_for[self][expired].expiration := 0;
          forward(max_withdrawn[self], clocks[self]);
        end with;
```

具体撤销的步骤是：
- 把 `support_for` 中记录的 `expiration` 清零，表示旧支持已经撤销
- 把 `support_for` 中记录的 `epoch` 增加 1，为下一轮支持做准备
- 用当前时钟推进 `max_withdrawn`，为后续重启后的时钟下界提供依据

#### 处理心跳

收到心跳消息时，其中的 `epoch` 表示的是：心跳发送方正在从心跳接收方请求第 `msg.epoch` 轮支持。于是，心跳接收方要拿这个 `msg.epoch` 去和本地记录的 `support_for[self][msg.from].epoch` 做比较，也就是比较“对方认为我在第几轮支持它”和“我自己认为我在第几轮支持它”是否一致。

对应代码如下：

```text
      if msg.type = MsgHeartbeat then
        ReceiveHeartbeat:
          if support_for[self][msg.from].epoch = msg.epoch then
            \* Forward the expiration to prevent regressions due to out-of-order
            \* delivery of heartbeats.
            forward(support_for[self][msg.from].expiration, msg.expiration);
          elsif support_for[self][msg.from].epoch < msg.epoch then
            assert support_for[self][msg.from].expiration < msg.expiration;
            \* This assertion is part of the Support Disjointness invariant.
            \* We assert that the requestor of support with this new epoch has
            \* a clock that exceeds the expiration of the previous epoch.
            assert support_for[self][msg.from].expiration < clocks[self];
            support_for[self][msg.from].epoch      := msg.epoch ||
            support_for[self][msg.from].expiration := msg.expiration;
          end if;

          send_heartbeat_resp(msg.from);
```

具体可以分为三种情况：

* 两边记录的 epoch 相同

```text
support_for[self][msg.from].epoch = msg.epoch
```

说明双方都还在同一轮支持里，用 `forward` 更新支持的过期时间即可（可能因为消息乱序到达，避免过期时间没有正确更新）。

```text
forward(support_for[self][msg.from].expiration, msg.expiration);
```

* 接收方记录的 epoch 更旧

```text
support_for[self][msg.from].epoch < msg.epoch
```

说明心跳发送方由于重启已经进入了一个更大的 `epoch`，因此它不再沿用旧的支持记录，而是要在新的轮次上重新请求支持。

这时模型会检查两个断言：

```text
\* 新一轮支持的过期时间，必须比旧一轮的过期时间更靠后。
assert support_for[self][msg.from].expiration < msg.expiration;
\* 确保两轮支持的时间上没有重叠
assert support_for[self][msg.from].expiration < clocks[self];
```

第一个断言要求：新一轮支持的过期时间必须晚于旧一轮。

第二个断言直接对应 `Support Disjointness Invariant`，它要求：当接收方准备接受这轮新支持时，
上一轮支持已经过期。只有这样，新旧两轮支持区间才不会重叠。

这个断言成立的推导过程如下，记心跳发送方为 `A`，心跳接收方为 `B`：

* `B` 曾经支持过 `A`，其过期时间为 `support_for[B][A].expiration`，记为 `E_old`
* `E_old` 是由心跳信息中的 `expiration` 而得，实际上也就是发送方的某个 `max_requested[A]` （参见发送心跳的部分）
* 由于 `max_requsted` 只会自增，所以在 `A` 重启之前一定有 `E_old <= max_requested[A]`
* `A` 重启后，一定能保证时钟 `clocks[A] >= max_requested[A]` （参见重启部分）
* 此时有 `E_old <= max_requested[A] <= clocks[A]`，即 `E_old <= clocks[A]`
* 重启后，当 `A` 发送心跳时，本地逻辑时钟 `clocks` 会自增，所以心跳中的时间戳满足 `E_old < msg.now`
* `B` 收到心跳时，会更新自己的逻辑时钟，因此 `msg.now <= clocks[B]`
  ```text
  \* Clock propagation is necessary for the Support Disjointness Invariant.
  forward(clocks[self], msg.now);
  ```
* 因此 `B` 收到 `A` 重启之后的心跳后，可以得到 `E_old < msg.now <= clocks[B]`
* 也就是 `support_for[self][msg.from].expiration < clocks[self]` 成立

因此，即便被支持方在重启后，在新的 epoch 上要求获取支持时，支持方的本地时钟已经大于旧 epoch 的支持过期时间，确保了两轮支持没有时间上的重叠，也就能安全的提供新的一轮支持。
之后会根据心跳消息更新 `support_for[self][msg.from].epoch` 和 `support_for[self][msg.from].expiration`。

* 接收方记录的 epoch 更大

如果本地记录 `support_for[self][msg.from].epoch > msg.epoch` 比 `msg.epoch` 更大，例如支持方已经撤销了旧支持（对应 `WithdrawSupport` 部分），那么这条心跳会被忽略。

无论是上述哪种情况，心跳接收方最后都会返回心跳响应，其中会携带自己记录的：

- `support_for[self][msg.from].epoch`
- `support_for[self][msg.from].expiration`

也就是说，心跳响应返回的是“支持方认定的支持状态”。

#### 处理心跳响应

处理心跳响应时，其中的 `epoch` 表示的是：心跳接收方正在向心跳发起提供第 `msg.epoch` 轮支持（`msg.epoch` 来自于心跳接受方的 `support_for`）。

此时心跳发起方收到响应时，主要任务是将自身的 `support_from` 和心跳接受方的 `support_for` 中记录的信息对齐。对应代码如下：

```text
      elsif msg.type = MsgHeartbeatResp then
        ReceiveHeartbeatResp:
          if max_epoch[self] < msg.epoch then
            max_epoch[self] := msg.epoch;
          end if;
          if support_from[self][msg.from].epoch = msg.epoch then
            \* Forward the expiration to prevent regressions due to out-of-order
            \* delivery of heartbeat responses.
            forward(support_from[self][msg.from].expiration, msg.expiration);
          elsif support_from[self][msg.from].epoch < msg.epoch then
            assert support_from[self][msg.from].epoch = msg.epoch - 1;
            assert msg.expiration = 0;
            \* This assertion is part of the Support Disjointness invariant.
            \* We assert that support for the previous epoch has expired before
            \* increasing the epoch and forgetting the previous epoch's expiration.
            \* We check the expiration wrt clocks[self] because, by the propagation
            \* of clocks via messages, we know that clocks[self] <= clock[msg.from].
            assert support_from[self][msg.from].expiration <= clocks[self];
            support_from[self][msg.from].epoch      := msg.epoch ||
            support_from[self][msg.from].expiration := msg.expiration;
          end if;
```

这里可以分两类情况。

如果提供支持一方和自身（需要被支持的一方）记录的信息一致，也就是 `support_from[self][msg.from].epoch = msg.epoch`，则更新支持的过期信息即可。

如果支持方已经进入更大的 `epoch`，即 `support_from[self][msg.from].epoch < msg.epoch`，说明支持方已经撤销了这轮支持。此时模型会检查如下断言（参照 `WithdrawSupport`），确保两轮支持的 `epoch` 连续：

```text
assert support_from[self][msg.from].epoch = msg.epoch - 1;
assert msg.expiration = 0;
```

接下来的这条断言，直接对应 `Support Disjointness Invariant`：

```text
assert support_from[self][msg.from].expiration <= clocks[self];
```

它要求心跳请求方在接受新轮次的支持之前，按本地的时钟，确保旧支持已经过期。否则就会出现“被支持方认定旧支持仍然有效，但支持方已经开始新一轮支持”的重叠问题。

这个断言成立的推导过程如下，记心跳发送方为 `A`，心跳接收方为 `B`：

* 某一个时刻 `B` 支持过 `A`, 假设 `A` 中 `support_from[A][B]` 中 `expiration` 为 `E_old`
* 在 `B` 的本地时间大于等于 `E_old` 之后，`B` 就能撤回之前的支持
* 之后 `B` 在收到 `A` 的心跳，对应的心跳响应中，会携带更大的 `epoch`，且 `support_for[B][A].expiration` 为 0, 此时满足 `E_old <= msg.now`
* `A` 在收到心跳响应时，由于会更新本地逻辑时间戳，因此 `E_old <= msg.now <= clocks[A]`
* 即 `support_from[self][msg.from].expiration <= clocks[self]` 成立

因此，被支持的一方也能确保上一轮的支持已经过期，才能接受新的支持。

### 完整视角

到这里，整个算法的主线就比较清楚了。

- `ReceiveHeartbeat` 站在支持方视角，保证“我给你的新支持不会和旧支持重叠”
- `ReceiveHeartbeatResp` 站在被支持方视角，保证“我接受你的新轮次之前，旧支持确实已经过期”

也就是说，`Support Disjointness` 其实是被两侧共同维护的：

- 支持方会确保不会过早发出下一轮支持，也不会在承诺的过期时间之前撤销当前支持
- 被支持方确保不会过早接受下一轮支持

而这正是 `StoreLiveness` 能支撑上层 leader lease 的关键。只要底层支持区间是可持续、且不同轮次之间不重叠的，上层就可以安全地把这些支持转化为互斥的 lease 区间。

## 如何将 StoreLiveness 应用到 raft leader lease

到这我们已经基本梳理清了 `StoreLiveness` 的流程，它能确保新旧支持时间上不重叠，且已经承诺的支持不会提前撤销。有了这个基础，再应用到 raft leader lease 就比较容易了。对于某个 raft group 的 leader，如果这个 raft group 所在的节点中的多数都在支持它，且这些支持都到某个时间 `T` 才过期，那么该 leader 能确信：在 `T` 之前，这些支持者不能撤销对我的支持，也就能确保 `T` 之前不会出现新的 leader，自然也就能保证读的安全性。

另一方面，如果一个节点上有很多 raft group，也就会有很多 raft group 的 leader，而 leader lease 的维护需要开销。而如果使用 `StoreLiveness` 把支持关系下沉到节点级别后，一个节点级别的支持就可以被多个 raft group 所复用，从每个 raft group 各自维护 lease，变成了一套节点级别的支持。这也正是这篇论文的核心所在。

## Reference

* [Scalable Leader Leases For Multi Consensus Groups in CockroachDB](https://dl.acm.org/doi/10.1145/3788853.3803081)
* [StoreLiveness](https://github.com/cockroachdb/cockroach/tree/master/docs/tla-plus/StoreLiveness)