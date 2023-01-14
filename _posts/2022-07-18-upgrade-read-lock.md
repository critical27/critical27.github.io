---
layout: single
title: How to upgrade read lock to write lock
date: 2022-07-18 00:00:00 +0800
categories: 学习
tags: C++ folly
---

什么是`Upgrade Lock`?

## Upgrade read lock to write lock

### 怎么从读锁升级为写锁？

假设有线程已经拿到了读锁，进行了一些条件检查，发现需要进行一些修改，我们是否能够将读锁”升级“为写锁？

如果允许类似操作，在多线程情况就会出现死锁，比如如下情况

```cpp
* thread 1 acquire read lock
* thread 2 acquire read lock
* thread 1 ask to upgrade lock to write // wait t2 release read lock
* thread 2 ask to upgrade lock to write // wait t1 release read lock
```

如果我们需要在某些条件进行写入，folly的SharedMutex里面写了几种解决办法：

1. Check the condition under a shared lock and release it. Then maybe check the condition again under a unique lock and maybe do the mutation.
2. Check the condition once under an upgradeable lock. Then maybe upgrade the lock to a unique lock and do the mutation.
3. Check the condition and maybe perform the mutation under a unique lock.

这里多出来一种概念upgradeable lock，它其实是一种read lock和write lock之间的中间状态，我们下面会详细介绍它的作用。

三种解决办法的代码

```cpp
TEST(Doodle, Basic) {
    folly::SharedMutex m;
    // 1.
    {
        {
            folly::SharedMutex::ReadHolder rh(m);
            LOG(INFO) << "Do check";
        }
        {
            folly::SharedMutex::WriteHolder wh(m);
            LOG(INFO) << "Do modify";
        }
    }
    // 2.
    {
        folly::SharedMutex::UpgradeHolder uh(m);
        LOG(INFO) << "Do check";
        folly::SharedMutex::WriteHolder wh(std::move(uh));
        LOG(INFO) << "Do modify";
    }
    // 3.
    {
        folly::SharedMutex::WriteHolder wh(m);
        LOG(INFO) << "Do check";
        LOG(INFO) << "Do modify";
    }
}
```

## Upgrade lock

`folly::SharedMutex`还提供了`Upgrade lock`(其实更准确的名称是Upgradable lock)，它有一些特点：

1. 和写锁一样，同一时间一个`SharedMutex`最多只有一个线程能够获取`Upgrade lock`（和写锁也不能共存）
2. 一个`Upgrade lock`可以和任意个读锁共存
3. 一个`Upgrade lock`可以原子的升级为写锁

我们再分析下引入Upgrade lock之后的影响，`SharedMutex`任何时候都满足如下两个条件

```cpp
num_threads_holding_exclusive + num_threads_holding_upgrade <= 1
num_threads_holding_exclusive == 0 || num_threads_holding_shared == 0
```

从上面的条件也可以看出`Upgrade lock`只能和读锁共存，不能和写锁共存

> `boost::shared_mutex`和`folly::SharedMutex`在有Upgradelock时候都允许新的读，而`folly::RWSpinLock`不允许
> 

> RWSpinLock's upgradeable mode blocks new readers, while SharedMutex's doesn't. Both semantics are reasonable. The boost documentation doesn't explicitly talk about this behavior (except by omitting any statement that those lock modes conflict), but the boost implementations do allow new readers while the upgradeable mode is held.  See [Boost's implematation](https://github.com/boostorg/thread/blob/master/include/boost/thread/pthread/shared_mutex.hpp)

`boost::shared_mutex`就是这里检查是否能上读锁的时候，只检查exclusive
> 
> 
> ```cpp
> bool can_lock_shared () const
> {
>     return ! (exclusive || exclusive_waiting_blocked);
> }
> ```
> 

想要使用`Upgrade lock`，最简单的方式就是`UpgradeHolder`，从提供的接口来看，要么直接获取upgrade lock，要么就从写锁降级（可以从`UpgradeHolder`升级为`WriteHolder`），而不能从读锁升级。

```cpp
class FOLLY_NODISCARD UpgradeHolder {
  UpgradeHolder() : lock_(nullptr) {}

 public:
  explicit UpgradeHolder(SharedMutexImpl* lock) : lock_(lock) {
    if (lock_) {
      lock_->lock_upgrade();
    }
  }

  explicit UpgradeHolder(SharedMutexImpl& lock) : lock_(&lock) {
    lock_->lock_upgrade();
  }

  // Downgrade from exclusive mode
  explicit UpgradeHolder(WriteHolder&& writer) : lock_(writer.lock_) {
    assert(writer.lock_ != nullptr);
    writer.lock_ = nullptr;
    lock_->unlock_and_lock_upgrade();
  }

  UpgradeHolder(UpgradeHolder&& rhs) noexcept : lock_(rhs.lock_) {
    rhs.lock_ = nullptr;
  }

  UpgradeHolder& operator=(UpgradeHolder&& rhs) noexcept {
    std::swap(lock_, rhs.lock_);
    return *this;
  }

  UpgradeHolder(const UpgradeHolder& rhs) = delete;
  UpgradeHolder& operator=(const UpgradeHolder& rhs) = delete;

  ~UpgradeHolder() {
    unlock();
  }

  void unlock() {
    if (lock_) {
      lock_->unlock_upgrade();
      lock_ = nullptr;
    }
  }

 private:
  friend class WriteHolder;
  friend class ReadHolder;
  SharedMutexImpl* lock_;
};
```

### UpgradeHolder获取锁

检查并等待`state & kHasSolo`是否为0，如果为0就CAS置上`kHasU`，阻止后续的exclusive操作。

> 可以看到`UpgradeHolder`构造时压根没有拿到exclusive ownership，所以类似这样的使用方式是错误的 https://github.com/vesoft-inc/nebula/pull/622/files
> 

```cpp
void lock_upgrade() {
  WaitForever ctx;
  (void)lockUpgradeImpl(ctx);
  OwnershipTrackerBase::beginThreadOwnership();
  // For TSAN: treat upgrade locks as equivalent to read locks
  annotateAcquired(annotate_rwlock_level::rdlock);
}

template <class WaitContext>
bool lockUpgradeImpl(WaitContext& ctx) {
  uint32_t state;
  do {
    if (!waitForZeroBits(state, kHasSolo, kWaitingU, ctx)) {
      return false;
    }
  } while (!state_.compare_exchange_strong(state, state | kHasU));
  return true;
}
```

### UpgradeHolder释放锁

尝试唤醒其他exclusive/upgrade线程

```cpp
void unlock_upgrade() {
  annotateReleased(annotate_rwlock_level::rdlock);
  OwnershipTrackerBase::endThreadOwnership();
  auto state = (state_ -= kHasU);
  assert((state & (kWaitingNotS | kHasSolo)) == 0);
  wakeRegisteredWaiters(state, kWaitingE | kWaitingU);
}
```

### UpgradeHolder升级为WriteHolder

`kHasU`已经被set，其实就是尝试clear `kHasU`，并set `kHasE`

```cpp
// Promotion from upgrade mode
explicit WriteHolder(UpgradeHolder&& upgrade) : lock_(upgrade.lock_) {
  assert(upgrade.lock_ != nullptr);
  upgrade.lock_ = nullptr;
  lock_->unlock_upgrade_and_lock();
}

void unlock_upgrade_and_lock() {
  // no waiting necessary, so waitMask is empty
  WaitForever ctx;
  (void)lockExclusiveImpl(0, ctx);
  annotateReleased(annotate_rwlock_level::rdlock);
  annotateAcquired(annotate_rwlock_level::wrlock);
}

// Performs an exclusive lock, waiting for state_ & waitMask to be
// zero first
template <class WaitContext>
bool lockExclusiveImpl(uint32_t preconditionGoalMask, WaitContext& ctx) {
  uint32_t state = state_.load(std::memory_order_acquire);
  if (LIKELY((state & (preconditionGoalMask | kMayDefer | kHasS)) == 0 &&
             state_.compare_exchange_strong(state, (state | kHasE) & ~kHasU))) {
    return true;
  } else {
    return lockExclusiveImpl(state, preconditionGoalMask, ctx);
  }
}
```

### WriteHolder降级为UpgradeHolder

clear kWaitingNotS, kWaitingS, kPrevDefer, kHasE

set kHasU

```cpp
// Downgrade from exclusive mode
explicit UpgradeHolder(WriteHolder&& writer) : lock_(writer.lock_) {
  assert(writer.lock_ != nullptr);
  writer.lock_ = nullptr;
  lock_->unlock_and_lock_upgrade();
}

void unlock_and_lock_upgrade() {
  annotateReleased(annotate_rwlock_level::wrlock);
  annotateAcquired(annotate_rwlock_level::rdlock);
  // We can't use state_ -=, because we need to clear 2 bits (1 of
  // which has an uncertain initial state) and set 1 other.  We might
  // as well clear the relevant wake bits at the same time.
  auto state = state_.load(std::memory_order_acquire);
  while (true) {
    assert((state & ~(kWaitingAny | kPrevDefer | kAnnotationCreated)) == kHasE);
    auto after = (state & ~(kWaitingNotS | kWaitingS | kPrevDefer | kHasE)) + kHasU;
    if (state_.compare_exchange_strong(state, after)) {
      if ((state & kWaitingS) != 0) {
        futexWakeAll(kWaitingS);
      }
      return;
    }
  }
}
```
