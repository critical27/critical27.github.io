---
layout: single
title: Distributed lock - part 2
date: 2023-08-07 00:00:00 +0800
categories: 学习
tags: Concurrency Consensus
---

书接前文，看大牛们怎么理解分布式锁？

## Argues about Redlock

在Redlock提出之后，Designing Data-Intensive Applications的作者Martin Kleppmann就在博客[How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)提出了质疑，Redlock也作为一个反例被写进了这本书里（只不过没有点名是Redlock）。

Martin Kleppmann举了一个反例，客户端1在获取到分布式锁之后操作共享资源之前，发生了GC，而GC期间锁又过期了。此时客户端2也可以获取到分布式，进而出现了两个客户端同时操作共享资源的问题。

![figure]({{'/archive/DistributedLock-2.png' | prepend: site.baseurl}})

这里面stop-the-world GC只是一种客户端被阻塞的情况，比如文章中提到好几种可能：

- GC: 任何Garbage Collector实现都可能需要stop-the-world
- 操作共享资源时，发生了network delay
- 发生page fault，需要将硬盘上的数据加载进内存
- 进程收到了SIGSTOP或者SIGPAUSE信号
- 时钟漂移，甚至闰秒问题

除了上面的问题，Martin Kleppmann还指出Redlock有以下缺陷，其核心在于Redlock是依赖于本地时间的比较来保证分布式锁互斥的。Redlock主要有两个地方使用了本地时间，可以参照前文理解：

1. Redis需要根据本地时间，决定记录的过期时间
2. 客户端在获取锁成功之后，需要用两次本地之间的差值来计算锁的有效时间

Redis使用`gettimeofday`来获取本地时间，这个时钟不是原子递增的，会出现时钟回退和漂移。因此，而Redlock却重度依赖于本地时间，又或者对网络延迟和超时时间做了时间上的假设，因此不是一个安全的分布式锁实现。原文如下：

> For algorithms in the asynchronous model this is not a big problem: these algorithms generally ensure that their safety properties always hold, without making any timing assumptions. Only liveness properties depend on timeouts or some other failure detector. In plain English, this means that even if the timings in the system are all over the place (processes pausing, networks delaying, clocks jumping forwards and backwards), the performance of an algorithm might go to hell, but the algorithm will never make an incorrect decision.
>
>
> However, Redlock is not like this. Its safety depends on a lot of timing assumptions: it assumes that all Redis nodes hold keys for approximately the right length of time before expiring; that the network delay is small compared to the expiry duration; and that process pauses are much shorter than the expiry duration.
>

博客里面举的一个反例如下，假设ABCDE组成一个5节点的Redis分布式锁集群：

1. 客户端1获取到节点 A、B、C 上的锁
2. 节点C上的时钟发生跳变（比如系统管理员突然把系统时间调大了），导致锁到期
3. 客户端2此时就能获取节点 C、D、E 上的锁
4. 客户端1和2都认为自己持有分布式锁

这个例子其实和Redis文档里面关于持久化一部分的例子几乎如出一辙，问题都出在其中一个节点在不超过TTL的时间内两次上锁成功。只不过文档中是节点发生了重启，而此处则是发生了时钟跳变。

如果我们假设真的不会发生时钟跳变，那么套用前面的GC，一样可以复现问题：

1. 客户端1向A、B、C、D、E发出上锁请求
2. 客户端1在收到上锁的响应之前发生GC，并一直卡住
3. 所有节点锁都过期
4. 客户端2获取锁成功
5. 客户端1此时GC结束，收到之前上锁请求的响应，认为上锁成功
6. 客户端1和2都认为自己持有分布式锁

总结一下，Martin Kleppmann认为Redlock依赖于以下的假设：

- 有上限的网络延迟
- 有上限的进程暂停时间
- 有上限的时钟漂移

只要上面的任何一个假设在真实环境中不满足，Redlock就不能保证safety。

### Fencing token

除此之外，Martin Kleppmann提出了一个解决方案：每次分布式锁返回成功时，需要携带一个自增的token。客户端在操作共享资源时，需要携带这个token。而共享资源可以比较当前已经使用的最大token，拒绝掉携带过期token的请求。通过每次生成的自增token，去除了对本地时钟的依赖。

![figure]({{'/archive/DistributedLock-3.png' | prepend: site.baseurl}})

比如还是前面的例子，客户端1获取了分布式锁，token为33，然后被GC阻塞住。客户端在锁过期之后获取了分布式锁，token为34，并操作了共享资源。当客户端1恢复时，再用token为33发送请求时，会被共享资源拒绝。

然而Redlock本身的限制导致不能使用fencing token来解决这些问题，一个核心点就在于Redlock的多个Redis节点都是主库，并且互相不通信，因此各个节点之间很难生成一个一致的自增token返回给客户端。

### **Is Redlock safe?**

Redis的作者Antirez之后也进行了反驳：

> In this analysis I’ll analyze Martin's analysis so that other experts in the field can check the two documents (the analysis and the counter-analysis), and eventually we can understand if Redlock can be considered safe or not.
>

Antirez的文章总结下来有几个论点：

**论点1：Martin提出的fencing token不是必须的**，这里需要从两个方面进行解释

- 是不是自增id关系不大，比如Redlock使用`/dev/urandom`中的20个字节作为uuid，也能够通过read-modify-write进行比较（冲突概率认为极低）。比如往MySQL中更新记录可以这么做：

    ```
    UPDATE table T SET val = $new_val WHERE id = $id
    ```

- 如果共享资源已经能通过某种机制拒绝掉过期请求（比如上面写入MySQL），这种情况下就不需要使用分布式锁，直接通过MYSQL中的记录不就好了吗？通常使用分布式锁的时候，都是因为共享资源不能提供类似分布式锁的互斥能力，比如写一个文件。

Antirez的原话如下，这个论点我是比较buy-in的。(~~当然讨论这部分的时候夹杂了很多对于fencing token是一个自增id的辩驳，整篇文章的可读性在我看来其实不是很高~~)

> I want to mention again that, what is strange about all this, is that it is assumed that you always must have a way to handle the fact that mutual exclusion is violated. Actually if you have such a system to avoid problems during race conditions, you probably don’t need a distributed lock at all, or at least you don’t need a lock with strong guarantees, but just a weak lock to avoid, most of the times, concurrent accesses for performances reasons.
>

论点2：其余问题不只是存在于Redlock

针对Martin博文中对Redlock不能应对NPC问题（Network delay, Process pause, Clock drift），Antirez也进行了反驳。

首先是时钟漂移，Redlock的确假设了各个时钟之间的速度是相近的，也就是时钟偏移量是有上限的，但Antirez认为这个时钟模型是符合现实情况的，事实上的确也如此。至于Martin文章中提到的时钟跳变，主要出现在两种情形：

1. 系统管理源手动修改系统时间
2. NTP同步

Antirez关于这部分的反驳让我不禁想起了我司有时候的技术讨论：

1. 不要手动修改时间就好 （原文说要是有人直接修改raft日志不一样会有问题吗）
2. NTP同步是每次不要跳变这么大，一点一点逼近就不会出现跳变了。

> The above two problems can be avoided by “1” not doing this (otherwise even corrupting a Raft log with “echo foo > /my/raft/log.bin” is a problem), and “2” using an ntpd that does not change the time by jumping directly, but by distributing the change over the course of a larger time span.
>

这的确也是一种解决的技术手段，不过这就对对Redis分布式锁进行运维的人员就有要求了，不是所有人都看到过这番争论的。

然后是网络延迟和进程暂停的问题，首先Antirez回顾了下Redlock的基本流程：

```

1. Get the current time.
2. All the steps needed to acquire the lock
3. Get the current time, again.
4. Check if we are already out of time, or if we acquired the lock fast enough.
5. Do some work with your lock.
```

在第一步和第三步分别获取了一次本地时间，如果Network delay和Process pause发生在了第一步和第三步之间，那通过两次本地时间的比较是能够发现的。而第一步和第三步的时间间隔也很短，因此时钟漂移的影响可以忽略不计，但时钟跳变只能按前面所说的方法进行规避了。这里举了一个例子：

当客户端发送请求尝试获取锁时，服务端已经成功授权锁，并发送了响应。然而此时发生网络丢包或者重传，导致当客户端收到响应时候，服务端已经认为过期了。所以客户端在收到响应时，网络延迟早就发生了，客户端也无能为力。客户端能做的就是必须检查我还有多少时间可以用来操作共享资源，这个检查是所有带自动过期的分布式锁算法都需要的。

那么只有Network delay和Process pause发生在第三步以后，才可能出现问题。而Antirez再次强调这是所有分布式锁服务都会遇到的问题，即客户端认为获取锁成功，而服务端已经认为锁过期的情况。即便使用ZooKeeper来作为分布式锁也一样会遇到问题，这里简单介绍Zookeeper是如何实现的分布式锁的。

1. 客户端1和2都尝试创建临时节点ZNode，例如`/lock`
2. 假设客户端1请求先到达，则加锁成功，客户端2加锁失败
3. 客户端1操作共享资源
4. 客户端1删除`/lock`节点，释放锁

ZooKeeper不像Redis那样，需要考虑锁的过期时间问题，它是采用了临时节点，保证客户端 1拿到锁后，只要连接不断，就可以一直持有锁。ZooKeeper的客户端在创建临时节点后，会在和服务端的Session中不断发送心跳来续约，保证服务端。如果客户端的心跳由于crash或者GC终端，那么这个临时节点会自动删除，保证了锁一定会被释放。

我们把之前的异常套用在ZooKeeper上：

1. 客户端1创建临时节点`/lock`成功，拿到了锁
2. 客户端1发生GC，并一直卡住
3. 客户端1无法给Zookeeper发送心跳，Zookeeper认为过期把临时节点`/lock`删除
4. 客户端2创建临时节点`/lock`成功，拿到了锁
5. 客户端1进程此时GC结束
6. 客户端1和2都认为自己持有分布式锁

因此，即便是使用Martin推荐的ZooKeeper来做分布式锁服务，也无法完全避免NPC的问题。（但Antirez只说其他分布式锁都有类似问题，并没有详细解释和举反例）

## 结论

双方其实说的都有道理，这更从侧面验证了实现一个分布式锁服务本质上就是要实现一致性共识。而一个通常意义下的分布式锁服务，是无法在极端情况下保证safety和liveness的。

## Reference

[万字长文说透分布式锁 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/403282013)

[Distributed Locks with Redis | Redis](https://redis.io/docs/manual/patterns/distributed-locks/)

[How to do distributed locking — Martin Kleppmann’s blog](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)

[Is Redlock safe?](http://antirez.com/news/101)

[Redis 实现分布式锁 - 掘金 (juejin.cn)](https://juejin.cn/post/6975069367574888478)

[Redis Redlock 的争论 - 掘金 (juejin.cn)](https://juejin.cn/post/6976538149904678925)

[ZooKeeper: Because Coordinating Distributed Systems is a Zoo (apache.org)](https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html)