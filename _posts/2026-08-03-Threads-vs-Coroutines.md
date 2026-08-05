---
layout: single
title: Threads vs Coroutines
date: 2026-08-03 00:00:00 +0800
categories: 学习
tags: C++
---

最近没什么动力更新，这篇文章会简单讨论一下线程和协程两种并发模型该如何挑选。

## Concurrency vs Parallelism

在具体聊线程和协程之前，我们先聊一下两个在中文语境中经常没有被严格区分、甚至会被混用的概念：Concurrency（并发）和 Parallelism（并行）。

Go 语言的大佬 Rob Pike 曾经说过："Concurrency is about dealing with lots of things at once. Parallelism is about doing lots of things at once." 即：

- **Concurrency** 关注的是程序的设计与组织方式，它是一种软件结构化的方法。
- **Parallelism** 关注的是任务是否真的同时执行，它是一种运行时属性。

具体来说，二者的定义更像：

- Concurrency：多个任务在同一时间段内都有进展，但不一定在同一个瞬间同时运行。
- Parallelism：多个任务在同一个时刻真正同时执行。

举个例子，夫妻两个人准备在家做两道菜，可能会出现的情况有：

- Not concurrent, not parallel（也就是 Sequential）：丈夫先做好第一道菜，再做好第二道菜。
- Concurrent, not parallel：妻子把第一道菜放到灶上加热后，开始处理第二道菜的食材。过一会第一道菜加热好了，之后再继续烹饪第二道菜。
- Concurrent and parallel：夫妻两个人同时各做一道菜。

我们把上面例子中的菜肴类比为 CPU 上运行的线程或者任务，而把夫妻二人类比为 CPU：

- 单个 CPU 按时间片调度多个任务执行，尽管执行过程中存在 interleaving，但任一时刻都只有一个任务真正运行；不过从整体上看，多个任务的进度都在推进。这就是 Concurrency。
- 多个 CPU 在同一时刻运行不同任务，则称为 Parallelism。

![figure]({{'/archive/Threads-vs-Coroutines.png' | prepend: site.baseurl}})

> Parallelism 一定是 Concurrency，因为想要同一个时刻运行任务，必然需要调度多个任务。而反过来 Concurrency 不一定是 Parallelism，单个 CPU 也能够按时间片 interleaving 的执行多个任务。
>

上面的定义如果换一种更偏程序设计的说法，Concurrency 关注的是如何组织和调度多个任务。它强调的是如何组织程序，使多个任务可以在一段时间内开始、运行和完成，但不要求它们必须在同一个瞬间一起执行。

相对地，Parallelism 关注的是多个计算是否真正同时执行。它通常意味着利用多处理器或多核，让两个或更多任务在同一时刻并行地运行。

> 从另一个方面说，如果多个任务之间有先后顺序依赖，显然会对 Concurrency 有更高要求。而如果多个任务之间没有任何顺序依赖，则天生满足 Parallelism。
>

理解了 Concurrency 和 Parallelism 的区别之后，接下来就可以进一步看程序里到底是靠什么来承载这些任务。在线程和协程这两种模型里，前者更偏向操作系统提供的原生执行单元，后者则更偏向用户态的轻量级调度抽象。我们先从传统的线程开始。

## Thread

任意时刻，一个线程通常处于以下几种状态之一：

- RUNNING：正在某个 CPU 上执行。
- RUNNABLE：已经可以运行，正在等待被调度。
- BLOCKED：处于等待队列中，例如等待 I/O、futex 或 timer。

操作系统中会由 Scheduler 决定哪个线程在哪个核心上运行，以及能运行多久。至于哪个线程先运行、一次运行多久、何时再次获得 CPU，都没有确定性保证，因此执行过程可能会 interleave。

```cpp
std::thread a({ /* work a */ });
std::thread b({ /* work b */ });
```

既然操作系统中的 Scheduler 会决定每个线程在哪个 CPU 上执行多久，自然就需要在线程之间切换，也就是所谓的 Context Switch。它既可能是非自愿的，比如被抢占（preemption），也可能是自愿的，比如阻塞在 I/O 上。

- Context Switch 会打断线程当前的工作，线程必须等到下一次被调度时才能继续。
- 线程越多，Scheduler 越需要花精力去维持 fairness。
- 如果线程数量超过硬件能高效承载的范围，就会出现 oversubscription。

### Thread Pool + Blocking

如果对线程数量不加限制，随着线程数越来越多，Context Switch 所占用的时间就会越长。另外由于 Context Switch 是一个内核操作，其开销不可忽视，甚至最终会出现 Context Switch 消耗大量 CPU 的情况。

为了限制线程数量，一个常见方法就是线程池。其典型实现都是，调用方将任务提交到线程池中的队列中，而线程池中的工作线程从队列中获取任务并执行。

- 能提供的 Parallelism，通常受限于线程池中的线程数量。
- 但每个线程拿到任务后通常会一路执行到结束，中间没有更细粒度的 interleaving。
- 还没轮到执行的任务，只能先停留在线程池队列里。
- 这也意味着整体 Concurrency 会受到限制。

### Event loop + Non-blocking

不过，如果提交到线程池中的任务大多都是 I/O bound 的，那么工作线程里的很多时间其实都会消耗在等待 I/O 上。这样一来，虽然线程数量并不少，但真正能持续推进任务的执行时间却比较有限，整体吞吐和资源利用率也会受到影响。

为了解决这个问题，一个常见的办法就是 Event loop。它的本质是由单个线程不断等待新的事件到来，再将事件分发给对应的 handler 处理。通过这种 Non-blocking 的方式，可以用较少的线程实现较高的并发。每个 Event loop 大致都在执行如下伪代码：

```cpp
while (true) {
    auto events = epoll_wait();
    for (event : events) {
        event.callback();  // do small, non-blocking work
    }
}
```

但 Non-blocking 的一个缺点是：既然它不阻塞等待 I/O 完成，就必须有某种机制感知 I/O 何时结束；而在结束后，又往往需要通过回调继续处理后续逻辑。因此，大量回调很容易让代码变得难以维护。

## Coroutine

后来，C++20 引入了协程，以及在此基础上发展出来的 Structured Concurrency。它的一大优势是：在能力上依然保留了 Non-blocking，但在写法上却更接近 Blocking 代码，不再需要层层回调：

```cpp
Task coroutine(...) {
	// ...
  co_await BlockingRead(...);
  // ...
}
```

> 关于协程，可以参考之前整理的一系列[文章](https://critical27.github.io/%E5%AD%A6%E4%B9%A0/Deciphering-Coroutines-part-1/)。
>

协程和线程一样也需要调度，不同的是，协程的切换发生在 User Space。编译器会为协程生成对应的状态机，协程的恢复与挂起本质上就是状态机的切换。这种“Context Switching”的成本通常接近一次函数调用，而且由应用程序自己决定何时恢复它的执行。

不过需要注意的是，协程库在用户空间提供的 scheduler，相比内核中的线程 scheduler，往往缺少更强的 fairness 保证。如果上面的 coroutine 里不是 `co_await BlockingRead(...)`，而是直接进行了阻塞调用，那么关联的所有协程都可能一起被卡住。因此对于协程这种 Cooperative multitasking 而言，所有协程都必须遵守一些基本规则，比如尽量避免在协程内部直接进行阻塞调用。

## Conclusion

从上面的讨论可以看出，线程和协程各自优化的问题并不相同，因此具体采用哪种模型，通常还是要结合 workload 来看。对于一个新系统来说：

* 对于 I/O bound 的 workload，瓶颈不在 CPU，而在等待数据返回。这种情况会更适合协程，因为它的挂起与恢复相比线程切换更轻量，挂起后也能让 CPU 去处理其他事情。
* 对于 CPU bound 的 workload，瓶颈主要就在 CPU 本身，因此直接使用线程往往就足够了。

而对于既有系统，情况往往没法这么简单地一刀切。当前采用某种并发模型，背后也许已经有更深层次的历史包袱、工程约束或者性能考量，还需要结合具体场景进一步分析。当然，本文讨论的主要还是高并发系统；如果系统对性能和资源利用率没有那么苛刻，那么传统的线程和线程池在很多场景下也已经足够用了。

## Reference

- [Threads vs Coroutines — Why C++ Has Two Concurrency Models - Conor Spilsbury - CppCon 2025](https://www.youtube.com/watch?v=txffplpsSzg)
- [ByteByteGo | Concurrency vs Parallelism](https://bytebytego.com/guides/concurrency-is-not-parallelism/)
- [Concurrency vs Parallelism | Concurrency Interview | AlgoMaster.io](https://algomaster.io/learn/concurrency-interview/concurrency-vs-parallelism)
