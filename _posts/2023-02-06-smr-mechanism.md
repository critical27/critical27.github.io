---
layout: single
title: Differences between SMR mechanisms
date: 2023-02-06 00:00:00 +0800
categories: 论文 实践
tags: C++ Concurrency
---

### Hazard pointer vs shared_ptr

相同之处：管理对象的生命周期

不同之处：一言以蔽之，Hazard pointer是线程安全的，但`shared_ptr`不是

- 为什么`hazard pointer`是线程安全的？

首先读线程不会修改任何指针所指向内容，并且会将指针记录进全局列表中，其他线程是不能对全局列表中的指针所指向的内容进行修改的（包括释放指针所指向的内存）。然后任何写线程，只会释放同时满足以下两个条件的指针所指向内存：

1. 该指针是在当前写线程所替换下来的旧版本指针
2. 该指针没有其他读线程在使用（即没有出现在全局列表中）

读写线程通过这些约束，保证了`hazard pointer`是线程安全的。

- 为什么智能指针不是线程安全的？

不管是`unique_ptr`还是`shared_ptr`都只能管理智能指针中的那个对象的生命周期（而不是智能指针本身），也就是通过引用计数的方式发现如果智能指针中的对象不再被使用，就会回收对应内存。但如果有多个线程同时修改智能指针，将这个指针指向不同对象，又或者其中夹杂若干读取智能指针的操作，这显然不是线程安全的，只能通过外部的锁等机制进行同步。

### Hazard pointers vs Epoch-based reclamation

|  | EBR | Hazard pointers |
| --- | --- | --- |
| 核心原理 | 只有全部线程同意，才能推进epoch，过期epoch的指针一定可以安全回收。 | 读线程告知其他线程我正在使用特定指针，其他线程不可以修改或释放这个指针指向的对象，但可以将指针指向其他对象。 |
| 粒度 | 粗 | 细 |
| 速度 | 快 只需要获取全局epoch即可 | 慢 需要CAS操作一个全局链表 |
| 内存释放内存时机 | 特定epoch没有被任何线程所引用 | thread local列表达到一定长度 |
| 内存释放是否会被阻塞 | 会 | 不会 |

这里解释几个方面

- 粒度：EBR一旦进入资源临界区之后，可以操作任意数量和类型的对象指针，只要在退出资源临界区之前，将指针放到epoch对应的列表中即可。但Hazard pointers每次进入资源临界区之前都需要先获取特定类型对象的指针，之后的操作也限定于这个指针，退出资源临界区之前，将指针放到thread_local的列表中。所以从这个角度上说，EBR是粗粒度，而Hazard pointers是细粒度。
- 速度：这里主要对比进入和退出资源临界区的速度
    - EBR只需要获取一次全局epoch

        ```cpp
          void acquire() {
            active_flags_[get_thread_id()].active_ = true;
            CPU_BARRIER();
            local_epoches_[get_thread_id()].epoch_ = global_epoch_.epoch_;
          }

          void release() {
            active_flags_[get_thread_id()].active_ = false;
          }
        ```

    - Hazard pointers则需要通过CAS操作来在全局链表中获取一个节点

        ```cpp
          HazptrNode* acquire() {
            auto p = head_;
            while (p != nullptr) {
              if (p->active_ || !CAS(&(p->active_), false, true)) {
                p = p->next_;
                continue;
              }
              return p;
            }

            HazptrNode* node = new HazptrNode();
            VLOG(2) << "[Node] new node " << node;
            node->active_ = true;

            HazptrNode* oldHead;
            do {
              oldHead = head_;
              node->next_ = oldHead;
            } while (!CAS(&head_, oldHead, node));
            return node;
          }

          void release(HazptrNode* node) {
            (node->pHazard_).store(nullptr, std::memory_order_release);
            node->active_ = false;
          }
        ```

- 内存释放是否会被阻塞

除了粒度和速度之外，二者最关键的不同之处就在于能否及时回收内存。EBR由于某个线程可能在特定epoch长时间不退出资源临界区，导致内存回收不及时。另外如果大量线程在不断尝试进入和退出资源临界区，也可能导致全局epoch较难推进。所以EBR通常的使用场景在于内存对象的生命周期非常短的情况下。而Hazard pointers则不存在阻塞的问题，任何一个线程所使用的对象，不会阻塞其他线程的内存回收。

### Hazard pointers vs RCU

二者的主要区别同样是在于粒度和是否阻塞。

- 粒度：Hazard pointers如之前所说，只保护单个指针。但RCU是保护可以保护资源临界区中的所有指针指向内存不被释放。
- RCU和EBR一样，如果某个线程在资源临界区的时间过长，会阻止资源临界区中的对象回收。所以RCU对于同样也是期望读临界区的时间足够短。
- RCU可以在读操作时做到wait-free，而Hazard pointers只能做到lock-free

> Quoted from folly:
So roughly: RCU is simple, but an all-or-nothing affair.  A single rcu_reader can block all reclamation. Hazptrs will reclaim exactly as much as possible, at the cost of extra work writing traversal code
>

[p0461r1.pdf (open-std.org)](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/p0461r1.pdf) 这个论文比较了几种延时内存回收的机制

![figure]({{'/archive/SMR.png' | prepend: site.baseurl}})