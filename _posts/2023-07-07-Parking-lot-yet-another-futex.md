---
layout: single
title: Parking lot - yet another futex
date: 2023-07-07 00:00:00 +0800
categories: 学习
tags: linux folly
---

Parking lot vs Futex?

## Futex

Futex是linux提供的一个系统调用，主要作用是进行线程或者进程之间同步。很多线程同步的primitive都是基于futex实现的，比如mutex和condition variable。其本质上就是一个4字节整形数(称为futex word)进行读取，和期望值进行比较，如果futex word和期望值相同，就在内核态进行阻塞等待。futex word的值完全是由用户管理，因此也是在用户态对其进行各种原子操作，最终决定是否需要交给内核态进行阻塞等待。

futex这个系统调用的原型如下所示：

```cpp
long syscall(SYS_futex,
             uint32_t* uaddr,
             int futex_op,
             uint32_t val,
             const struct timespec* timeout, /* or: uint32_t val2 */
             uint32_t* uaddr2,
             uint32_t val3);
```

通过比较`uaddr`地址的这个futex word，是否和期望值`val`相同：

- 如果相同，表示需要进入内核态阻塞等待，当前线程会进入sleep状态
- 如果不相同，表示不满足阻塞等待的条件，立即返回

这个系统调用中，`读取 - 比较 - 阻塞`是在内核中原子的完成。内核对每个`uaddr`所指向的futex word都维护了一个队列。内核中有一个hashmap，保存了futex word地址到队列的映射。每次进行系统调用，并决定需要等待时，就会在对应队列中增加一个waiter，然后就会进入sleep状态等待唤醒。

而当需要唤醒futex对应的一个或多个waiter时候，仍然是通过这个系统调用，传入相同futex word的地址，将`op`设置为相应唤醒操作，就能够唤醒之前进入阻塞等待状态的waiter。

> futex系统调用的参数和操作较多，这里没有展开介绍，之后有机会再完整阐述。可以参考[futex(2) - Linux manual page (man7.org)](https://man7.org/linux/man-pages/man2/futex.2.html)
> 

Futex虽然提供了等待和唤醒的能力，也就能基于它实现其他的同步原语。比如Semaphore, POSIX mutex和condition variable都是基于futex实现的。但是考虑到每次futex在决定要阻塞等待时，都会将当前线程挂起，势必会带来context switch。因此几乎所有的mutex等同步原语都不会只用futex来进行阻塞等待（想象一下每次获取mutex失败都会进行线程切换带来的代价），一个比较通用的做法如下：

- 在用户态进行spin，不断检查期望条件是否成立
- 如果超过了最大的spin次数，此时会进行若干次yield，交出cpu控制权，再检查期望条件是否成立
- 如果超过了最大yield次数，此时才会通过futex等待期望条件成立

通过多种重试和等待策略，保证了在抢占不严重的情况下，能够最快速度执行。只有当需要长时间等待时，才会通过futex进行等待。

另外一点，上面也提到futex是一个系统调用，并且是由内核来维护futex到waiter队列的映射。既然futex的全称是fast user-space mutex，但决定是否阻塞等待的这一个过程仍然是在内核态完成的，有没有办法在用户态完成这一步呢？

这就是本文要介绍的parking lot。

## Parking lot

### What is parking lot?

从本质上来说，parking lot就是另一种futex的实现，我们沿用futex word的概念，二者区别和相同之处在于：

- 二者提供的都是基于futex word进行同步的功能，但parking lot只支持线程间同步，而futex支持线程和进程之间的同步。
- parking lot中，futex word到waiter queue的hashmap，以及各个waiter queue都是在用户态维护。futex中，则是在内核维护。
- 由于上述数据结构在不同地方进行维护，因此parking lot是在用户态决定是否需要阻塞等待，futex则是在内核态决定。二者最终阻塞等待的方式类似，都是通过挂起线程，进入sleep进行等待。
- parking lot一般是基于`std::mutex`和`std::condition_variable`完成阻塞等待，因此可移植性会比futex更好。

### Why parking lot?

既然parking lot和futex提供的功能类似，那为什么还要重复造轮子呢？我总结有以下几点原因：

- 出于性能原因，需要自己实现mutex、condition_variable等等同步原语，此时需要类似futex的底层机制。这篇[文章](https://webkit.org/blog/6161/locking-in-webkit/)最早提出parking lot的概念，并且详细的介绍了在webkit中的mutex, condition variable以及底层parking lot的实现原理和背后的思考，非常值得阅读。
- futex作为一个系统调用，参数过于复杂。其实是不适合应用程序直接使用的，更多是使用基于futex的mutex等来间接使用。而parking lot则不限制只能比较int32，而是通过传入的lambda函数检查是否需要阻塞等待，因此其接口会更友好。
- 上面提到的可移植性。

`Folly`提供了`Futex`这个类，其本质就是在linux系统中就使用内核提供的futex，而其他平台上则使用自己的实现的`ParkingLot`模拟了内核的`futex`，我们之后会看下这个类的实现。

而rust中parking lot似乎比较流行，甚至差一点就把基于parking lot的mutex合入到rust标准库之中了，可以参见

- <https://github.com/rust-lang/rust/issues/93740>
- <https://github.com/rust-lang/rust/pull/56410>

这也侧面说明了从功能上，parking lot是完全能够胜任futex的。

### Parking lot implementation

这里以Folly的实现为例，WebKit的实现稍微有些复杂，感兴趣的可以移步[WebKit/Source/WTF/wtf/ParkingLot.h at main · WebKit/WebKit · GitHub](https://github.com/WebKit/WebKit/blob/main/Source/WTF/wtf/ParkingLot.h)学习，另外二者的接口并不完全相同。

首先我们看下接口，每个parking lot有一个全局递增分配的id，核心接口就是`park`和`unpark`，`park_for`和`park_until`只是额外增加了一个超时参数。

```cpp
template <typename Data = Unit>
class ParkingLot {
  const uint64_t lotid_;
  ParkingLot(const ParkingLot&) = delete;

  struct WaitNode : public parking_lot_detail::WaitNodeBase {
    const Data data_;

    template <typename D>
    WaitNode(uint64_t key, uint64_t lotid, D&& data)
        : WaitNodeBase(key, lotid), data_(std::forward<D>(data)) {}
  };

 public:
  ParkingLot() : lotid_(parking_lot_detail::idallocator++) {}

  template <typename Key, typename D, typename ToPark, typename PreWait>
  ParkResult park(const Key key, D&& data, ToPark&& toPark, PreWait&& preWait) {
    return park_until(
        key,
        std::forward<D>(data),
        std::forward<ToPark>(toPark),
        std::forward<PreWait>(preWait),
        std::chrono::steady_clock::time_point::max());
  }
  
  // park_for
  // park_until

  template <typename Key, typename Unparker>
  void unpark(const Key key, Unparker&& func);
};
```

我们具体看看`park_until`的模板参数：

- `Key`是一个变量的地址，用来计算哈希，找到对应的队列
- `D`是用来在unpark时候检查这个waiter是否需要继续等待，在part时只是保存进`WaitNode`中
- `ToPark`是一个lambda函数，如果返回true则当前线程会进入阻塞等待，返回false则表示不满足等待条件，直接返回
- `PreWait`也是一个lambda函数，用于在真正等待之前再额外执行一些逻辑
- `<Clock, Duration>`组成一个`time_point`，用于指定最长等待时间

```cpp
template <typename Data>
template <
    typename Key,
    typename D,
    typename ToPark,
    typename PreWait,
    typename Clock,
    typename Duration>
ParkResult ParkingLot<Data>::park_until(
    const Key bits,
    D&& data,
    ToPark&& toPark,
    PreWait&& preWait,
    std::chrono::time_point<Clock, Duration> deadline) {
  auto key = hash::twang_mix64(uint64_t(bits));
  auto& bucket = parking_lot_detail::Bucket::bucketFor(key);
  WaitNode node(key, lotid_, std::forward<D>(data));

  {
    // A: Must be seq_cst.  Matches B.
    bucket.count_.fetch_add(1, std::memory_order_seq_cst);

    std::unique_lock<std::mutex> bucketLock(bucket.mutex_);

    if (!std::forward<ToPark>(toPark)()) {
      bucketLock.unlock();
      bucket.count_.fetch_sub(1, std::memory_order_relaxed);
      return ParkResult::Skip;
    }

    bucket.push_back(&node);
  } // bucketLock scope

  std::forward<PreWait>(preWait)();

  auto status = node.wait(deadline);

  if (status == std::cv_status::timeout) {
    // it's not really a timeout until we unlink the unsignaled node
    std::lock_guard<std::mutex> bucketLock(bucket.mutex_);
    if (!node.signaled()) {
      bucket.erase(&node);
      return ParkResult::Timeout;
    }
  }

  return ParkResult::Unpark;
}
```

其实介绍完上面的参数，其逻辑已经非常清晰了：

1. 根据`bits`计算哈希，确定这个地址对应的bucket （所有bucket是全局静态构造好的，数量固定），构造`WaitNode`
2. 获取bucket的mutex（之后可能会向bucket的链表中添加`WaitNode`，因此需要上锁）
3. 执行`toPark`，检查是否需要等待，如果不需要则直接返回`ParkResult::Skip`
4. 将`WaitNode`加入到bucket的链表中，然后释放锁（critical section就是需要操作bucket的部分）
5. 调用`preWait`
6. 调用`WaitNode::wait`进入真正等待（具体实现下面会解释）
7. 根据等待的结果判断是超时还是被唤醒，如果是超时且确定这个`WaitNode`没有被唤醒过，那么就返回`ParkResult::Timeout`
8. 否则就返回`ParkResult::Unpark`代表被唤醒

而唤醒的代码很类似，获取到`bits`对应的bucket，然后遍历bucket的链表，找到正确的`WaitNode`，根据`func`确定是否要唤醒这个`WaitNode`。

```cpp
template <typename Data>
template <typename Key, typename Func>
void ParkingLot<Data>::unpark(const Key bits, Func&& func) {
  auto key = hash::twang_mix64(uint64_t(bits));
  auto& bucket = parking_lot_detail::Bucket::bucketFor(key);
  // B: Must be seq_cst.  Matches A.  If true, A *must* see in seq_cst
  // order any atomic updates in toPark() (and matching updates that
  // happen before unpark is called)
  std::atomic_thread_fence(std::memory_order_seq_cst);
  if (bucket.count_.load(std::memory_order_seq_cst) == 0) {
    return;
  }

  std::lock_guard<std::mutex> bucketLock(bucket.mutex_);

  for (auto iter = bucket.head_; iter != nullptr;) {
    auto node = static_cast<WaitNode*>(iter);
    iter = iter->next_;
    if (node->key_ == key && node->lotid_ == lotid_) {
      auto result = std::forward<Func>(func)(node->data_);
      if (result == UnparkControl::RemoveBreak ||
          result == UnparkControl::RemoveContinue) {
        // we unlink, but waiter destroys the node
        bucket.erase(node);

        node->wake();
      }
      if (result == UnparkControl::RemoveBreak ||
          result == UnparkControl::RetainBreak) {
        return;
      }
    }
  }
}
```

`func`就是用户需要根据节点数据，决定是否需要唤醒以及唤醒哪些waitNode的逻辑，`func`的返回值有几种可能：
* `RetainContinue`代表不唤醒当前`WaitNode`，并继续遍历bucket链表
* `RemoveContinue`代表唤醒当前`WaitNode`，并继续遍历bucket链表
* `RetainBreak`代表不唤醒当前`WaitNode`，不继续遍历bucket链表
* `RemoveBreak`代表唤醒当前`WaitNode`，不继续遍历bucket链表


可以看到`ParkingLot`这个类主要作用就是管理变量地址到bucket的映射关系，然后在bucket的脸变中增加或者删除一个`WaitNode`。具体的等待和唤醒操作都是在`WaitNode`中完成的。

`WaitNode`的相关实现如下，本质就是一个双向链表的节点，每个`WaitNode`通过`mutex`和`condition_variable`完成等待和唤醒。

```cpp
struct WaitNodeBase {
  const uint64_t key_;
  const uint64_t lotid_;
  WaitNodeBase* next_{nullptr};
  WaitNodeBase* prev_{nullptr};

  // tricky: hold both bucket and node mutex to write, either to read
  bool signaled_;
  std::mutex mutex_;
  std::condition_variable cond_;

  WaitNodeBase(uint64_t key, uint64_t lotid)
      : key_(key), lotid_(lotid), signaled_(false) {}

  template <typename Clock, typename Duration>
  std::cv_status wait(std::chrono::time_point<Clock, Duration> deadline) {
    std::cv_status status = std::cv_status::no_timeout;
    std::unique_lock<std::mutex> nodeLock(mutex_);
    while (!signaled_ && status != std::cv_status::timeout) {
      if (deadline != std::chrono::time_point<Clock, Duration>::max()) {
        status = cond_.wait_until(nodeLock, deadline);
      } else {
        cond_.wait(nodeLock);
      }
    }
    return status;
  }

  void wake() {
    std::lock_guard<std::mutex> nodeLock(mutex_);
    signaled_ = true;
    cond_.notify_one();
  }

  bool signaled() { return signaled_; }
};
```

### Parking lot usage

最后我们可以看下如何使用parking lot，前面提到过Folly内部的Futex类就封装了内核的futex和parking lot，用于在不同系统使用。

```cpp
FutexResult futexWaitImpl(
    const Futex<std::atomic>* futex,
    uint32_t expected,
    system_clock::time_point const* absSystemTime,
    steady_clock::time_point const* absSteadyTime,
    uint32_t waitMask) {
#ifdef __linux__
  return nativeFutexWaitImpl(
      futex, expected, absSystemTime, absSteadyTime, waitMask);
#else
  return emulatedFutexWaitImpl(
      futex, expected, absSystemTime, absSteadyTime, waitMask);
#endif
}
```

在非linux环境下，就是使用`ParkingLot`模拟了futex，里面就是调用`park_untill`，如果`*futex == expected`，那么就进入sleep等待，和futex所实现的功能是一样的。

```cpp
template <typename F>
FutexResult emulatedFutexWaitImpl(
    F* futex,
    uint32_t expected,
    system_clock::time_point const* absSystemTime,
    steady_clock::time_point const* absSteadyTime,
    uint32_t waitMask) {
  static_assert(
      std::is_same<F, const Futex<std::atomic>>::value ||
          std::is_same<F, const Futex<EmulatedFutexAtomic>>::value,
      "Type F must be either Futex<std::atomic> or Futex<EmulatedFutexAtomic>");
  ParkResult res;
  if (absSystemTime) {
    res = parkingLot.park_until(
        futex,
        waitMask,
        [&] { return *futex == expected; },
        [] {},
        *absSystemTime);
  } else if (absSteadyTime) {
    res = parkingLot.park_until(
        futex,
        waitMask,
        [&] { return *futex == expected; },
        [] {},
        *absSteadyTime);
  } else {
    res = parkingLot.park(
        futex, waitMask, [&] { return *futex == expected; }, [] {});
  }
  switch (res) {
    case ParkResult::Skip:
      return FutexResult::VALUE_CHANGED;
    case ParkResult::Unpark:
      return FutexResult::AWOKEN;
    case ParkResult::Timeout:
      return FutexResult::TIMEDOUT;
  }

  return FutexResult::INTERRUPTED;
}
```

## Reference

[Locking in WebKit | WebKit](https://webkit.org/blog/6161/locking-in-webkit/)

[futex(2) - Linux manual page (man7.org)](https://man7.org/linux/man-pages/man2/futex.2.html)