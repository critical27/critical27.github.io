---
layout: single
title: Distributed lock - part 3
date: 2023-09-04 00:00:00 +0800
categories: 学习
tags: Concurrency Consensus
---

前两篇我们大概讲述了分布式锁的原理和实现，以及围绕Redlock展开的争论。那么这一篇，我们会用TLA+描述一个简单的分布式锁，以及Martin Kleppmann提出的Fencing Token能否解决NPC（Network delay, Process pause, Clock drift）导致分布式锁共识失效的问题。

## Distributed Lock in TLA+

首先定义一下我们的这个分布式锁服务的工作模式：

- 服务端是一个单机进程，网络是可靠的，不会出现丢包和无限长的网络延迟
- 服务端有且只有一个锁给客户端使用（类比都使用Redis中同一个key）
- 服务端可以在任意时刻从请求队列中取一个请求并处理，所有客户端的请求都会进入到同一个请求队列中，服务端一定会回复客户端获取分布式锁的结果
- 客户端使用分布式锁的步骤如下：
    - 往服务端发送获取锁的请求
    - 在响应中检查服务端是否授予分布式锁
    - 如果授权，那么正常使用共享资源，然后释放锁
- 为了模拟分布式锁的TTL，服务端可以在任意时刻将某个客户端已经获取的分布式锁设置为过期

### Invariant

然后是分布式锁服务中用到的一些变量定义

```
variable
    \* Status in distributed lock service: Available, Expired, Acquired
    \* id is the client id which hold the lock, InvalidId means no one
    distLock = [status |-> "Available", id |-> InvalidId],
    request_queue = <<>>,
    responses_queue = [k \in Clients |-> <<>>],

    \* Whether client hold lock by its view
    lock_held = [k \in Clients |-> FALSE];
```

- `distLock`就是这个分布式锁服务提供的唯一一个锁，其状态可能是`Available`, `Expired`, `Acquired`。`id`则是目前获取到分布式锁服务的标识符，如果没有客户端已经获取分布式锁，id为InvalidId。
    - `Available`代表目前没有客户端获取到分布式锁
    - `Acquired`代表目前有客户端已经成功获取到分布式锁
    - `Expired`代表某个客户端已经获取到分布式锁，经过一段时间之后锁过期
- `request_queue`是服务端接收和处理请求的队列
- `responses_queue`则是服务端给每个客户端回复响应的队列
- `lock_held`则是在每个客户端视角，自己是否获取到了分布式锁的标志位

接下来就可以描述这个系统的Invariant了：

```
TypeInvariant ==
  /\ distLock \in {[status |-> "Available", id |-> InvalidId], [status |-> "Expired", id |-> InvalidId]}
                  \union
                  {[status |-> "Acquired", id |-> c] : c \in Clients}
  /\ \A reqEntry \in SeqToSet(request_queue) : reqEntry.type \in {"acquire", "release"} /\ reqEntry.id \in Clients
  /\ \A c \in Clients : \A respEntry \in SeqToSet(responses_queue[c]) : respEntry \in {"granted", "rejected"}
  /\ lock_held \in [Clients -> {TRUE, FALSE}]

AtMostOneClientHoldLock ==
  /\ ~\E c1, c2 \in Clients :
    /\ c1 /= c2
    /\ lock_held[c1] = TRUE
    /\ lock_held[c2] = TRUE
```

- `TypeInvariant`是为了检查下面行为逻辑中修改的变量都是合法值。分布式锁`distLock`的合法状态为：
    - `Available`或者`Expired` ，代表此时没有客户端获取锁，即`id`为`InvalidId`
    - `Acquired`，代表已经有客户端获取了分布式锁，`id`为对应客户端标识符
- `AtMostOneClientHoldLock`则是为了检查任意时刻不会出现两个客户端同时认为自己获取了分布式锁。（也就是safety property）

### 客户端和服务端行为

接下来开始描述客户端：

```cpp
fair process (client \in Clients)
{
C1:
    await lock_held[self] = FALSE;
    request_queue := Append(request_queue, [type |-> "acquire", id |-> self]);
C2:
    await Len(responses_queue[self]) > 0;
    if (Head(responses_queue[self]) = "granted") {
        lock_held[self] := TRUE;
    };
    responses_queue[self] := Tail(responses_queue[self]);
C3:
    await lock_held[self] = TRUE;
    \* use shared resources
    lock_held[self] := FALSE;
    request_queue := Append(request_queue, [type |-> "release", id |-> self]);
}
```

1. `C1`: 如果自己认为没有获取到分布式锁，往服务端发送一个请求（如前文所说，不模拟丢包、乱序等网络扰乱，因此直接往服务端的队列中增加一个请求即可）。请求的内容中`type |-> "acquire"`代表是一个获取分布式锁的请求，`id |-> self`则是客户端自身的id。
2. `C2`: 发送请求之后，客户端一直等待服务端回复。如果服务端回复 "granted"，就代表自己已经成功获取分布式锁。并将这个response从队列中移除。
3. `C3`: 客户端开始使用资源，随后向服务端发送释放分布式锁的请求。

然后是服务端，直至所有客户端都完成，一直执行如下两个操作中的一个：

- 从请求队列中获取一个消息，然后开始处理（对应标签S1, S2, S3）
- 将分布式锁设置为`Expired`过期状态（对应标签S4）

```
fair process (service = Service)
variable req = <<>>, resp = <<>>;
{
SERVICE_MAIN:
    while (~ \A cli \in Clients : pc[cli] = "Done") {
        either {
            await Len(request_queue) > 0;
        S1:
            req := Head(request_queue);
            request_queue := Tail(request_queue);
        S2:
            if (req.type = "acquire") {
                if (distLock.status \in {"Available", "Expired"}) {
                    distLock := [status |-> "Acquired", id |-> req.id];
                    resp := "granted";
                } else {
                    resp := "rejected";
                };
            S3:
                responses_queue[req.id] := Append(responses_queue[req.id], resp);
            } else if (req.type = "release") {
                if (distLock.status = "Acquired" /\ distLock.id = req.id) {
                    distLock := [status |-> "Available", id |-> InvalidId];
                }
            }
        } or {
            await distLock.status = "Acquired";
        S4:
            distLock := [status |-> "Expired", id |-> InvalidId];
        }
    }
}
```

1. `S1`: 从请求队列中取第一个请求用于处理。
2. `S2`: 如果请求是获取分布式锁的请求，检查能否授予客户端分布式锁。如果处于`Available`或者`Expired`就可以授权，更新自身状态。否则就拒绝授权。随后给客户端回复响应。
3. `S3`: 将回复加入到客户端的响应队列中。如果请求是释放分布式锁的请求，那么检查当前分布式锁是否是被对应客户端锁持有。如果是，则释放分布式锁，否则忽略这个请求（一个客户端不能释放其他客户端持有的分布式锁）。
4. `S4`: 为了模拟客户端持有的分布式锁过期，如果某个客户端已经获取分布式锁，服务端可以在任意时刻将锁设置为过期状态。

> 由于TLA+ 在同一个标签下的所有操作都是原子完成的，而现实情况中，服务端处理请求和回复响应（分别对应S2和S3）并不是一个原子操作，所以将这两个操作分别放到不同标签下。
>

### TLC验证

我们用两个客户端的情况来验证一下目前的这个分布式锁是否满足safety，很快TLC就提示：`Invariant AtMostOneClientHoldLock is violated`，即出现了两个客户端同时认为自己获取到分布式锁这个不合理现象，违背了共识。我们看下error trace：

> 标红的部分代表上一个状态和当前状态的diff，省略掉一些不重要的状态
>

![figure]({{'/archive/DistributedLock-4.png' | prepend: site.baseurl}})

![figure]({{'/archive/DistributedLock-5.png' | prepend: site.baseurl}})

- (State 1)初始化，所有客户端都没有获取分布式锁
- (State 2)客户端C1发送了获取分布式锁的请求
- (State 5)服务端检查分布式锁状态为Available，因此将分布式锁授予给C1
- (State 6)服务端将授予分布式锁的回复加入到客户端C1的响应队列中。由于某些特殊情况（比如网络延迟，GC等），C1此时还未说到对应回复。
- (State 8)服务端此时认为分布式分布式锁已经过期，因此将分布式锁设置为过期状态
- (State 9)客户端C1此时收到了回复，认为自己持有了分布式锁
- (State 10)客户端C2发送了获取分布式锁的请求
- (State 13)服务端检查分布式锁状态为Expired，因此将分布式锁授予给C2
- (State 14)服务端回复客户端C2
- (State 15)客户端C2收到回复，认为自己持有了分布式锁

因此客户端C1和C2都认为自己持有了分布式锁，违背了同一时间只有一个客户端持有分布式锁的Invariant，二者可能同时操作共享资源。其实仔细观察这个error trace，和[上一篇](https://critical27.github.io/%E5%AD%A6%E4%B9%A0/Distributed-lock-part-2/)中的Martin Kleppmann举的例子几乎如出一辙，都是客户端1在获取分布式锁之后，由于各种问题（error trace中是客户端1没有收到回复，而在原始的例子中是客户端1出现了GC），导致服务端认为分布式锁过期，进而将分布式锁授予给了另一个客户端。

![figure]({{'/archive/DistributedLock-6.png' | prepend: site.baseurl}})

### Fencing Token

我们来验证一下Martin Kleppmann说的Fencing Token能否解决上面出现的问题。

首先我们需要引入token，用`history`保存使用共享资源时的token，将其加入到系统的变量中

```
variable
    \* Status in distributed lock service: Available, Expired, Acquired
    \* id is the client id which hold the lock, InvalidId means no one
    distLock = [status |-> "Available", id |-> InvalidId],
    request_queue = <<>>,
    responses_queue = [k \in Clients |-> <<>>],

    \* Whether client hold lock by its view
    lock_held = [k \in Clients |-> FALSE],

    \* The token usage history in shared resource's view
    history = <<>>;
```

在服务端会记录token，每次授予分布式锁时，会自增这个token，并且会将token告知客户端，以便客户端携带这个token操作共享资源。

```
fair process (service = Service)
variable req = <<>>, resp = <<>>, service_token = 1;
{
SERVICE_MAIN:
    while (~ \A cli \in Clients : pc[cli] = "Done") {
        either {
            await Len(request_queue) > 0;
        S1:
            req := Head(request_queue);
            request_queue := Tail(request_queue);
        S2:
            if (req.type = "acquire") {
                if (distLock.status \in {"Available", "Expired"}) {
                    distLock := [status |-> "Acquired", id |-> req.id];
                    resp := [status |-> "granted", token |-> service_token];
                    service_token := service_token + 1;
                } else {
                    resp := [status |-> "rejected"];
                };
            S3:
                responses_queue[req.id] := Append(responses_queue[req.id], resp);
            } else if (req.type = "release") {
                if (distLock.status = "Acquired" /\ distLock.id = req.id) {
                    distLock := [status |-> "Available", id |-> InvalidId];
                }
            }
        } or {
            await distLock.status = "Acquired";
        S4:
            distLock := [status |-> "Expired", id |-> InvalidId];
        }
    }
}
```

在客户端则保存服务端发送的token，当且仅当目前的token比共享资源已经见过的最大token还要大时，才能使用共享资源。（也就是共享资源能够拒绝掉过期请求的能力）

```
fair process (client \in Clients)
\* 0 means invalid
variable token = 0;
{
C1:
    await lock_held[self] = FALSE;
    request_queue := Append(request_queue, [type |-> "acquire", id |-> self]);
C2:
    await Len(responses_queue[self]) > 0;
    if (Head(responses_queue[self]).status = "granted") {
        lock_held[self] := TRUE;
        token := Head(responses_queue[self]).token;
    };
    responses_queue[self] := Tail(responses_queue[self]);
C3:
    if (token /= 0) {
        \* Lock aquired, use the share resource with given token
        if (Len(history) = 0 \/ history[Len(history)] < token) {
            history := Append(history, token);
        }
    };
C4:
    await lock_held[self] = TRUE;
    lock_held[self] := FALSE;
    request_queue := Append(request_queue, [type |-> "release", id |-> self]);
}
```

最后就是Invariant，那么已知我们不能避免两个客户端同时自认为获取到了分布式锁，我们可以换一种角度来验证safety，即每次使用共享资源时，token都必须自增。

```
TokenIsMonotonical ==
  ~\E i, j \in DOMAIN history :
    /\ i < j
    /\ history[i] >= history[j]
```

在TLC中验证通过

![figure]({{'/archive/DistributedLock-7.png' | prepend: site.baseurl}})

### 结论

我们在这一篇中用TLA+对分布式锁服务进行了建模，成功找到了因为NPC问题导致两个客户端同时认为自己获取到了分布式锁的反例，也基于FencingToken对我们的模型进行了修正，事实也证明如果共享资源如果具有拒绝掉过期资源的能力，分布式锁的safety就能得到保证。整个分布式锁系列相关的文章也就到此结束~

## Reference

[How to do distributed locking — Martin Kleppmann’s blog](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)