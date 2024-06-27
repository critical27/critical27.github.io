---
layout: single
title: folly::Executor::KeepAlive
date: 2024-04-17 00:00:00 +0800
categories: 源码 学习
tags: folly Concurrency
---

之前一直没有总结过`Executor`中的`KeepAlive`，最近因为工作中使用`Executor`又碰到了一些问题，趁热打铁一下。

## KeepAlive

 它的作用根据注释来看就是一个`Executor`的安全指针， 只要`Executor`的某个`KeepAlive`存在， `Executor`的析构函数就会被阻塞，直到所有`KeepAlive`对象都析构。

### 数据结构

我们首先看下`KeepAlive`的接口和数据结构：

```cpp
class Executor {
 public:
  // Workaround for a linkage problem with explicitly defaulted dtor t22914621
  virtual ~Executor() {}

  /// Enqueue a function to executed by this executor. This and all
  /// variants must be threadsafe.
  virtual void add(Func) = 0;

  // ...

  /**
   * Executor::KeepAlive is a safe pointer to an Executor.
   * For any Executor that supports KeepAlive functionality, Executor's
   * destructor will block until all the KeepAlive objects associated with that
   * Executor are destroyed.
   * For Executors that don't support the KeepAlive functionality, KeepAlive
   * doesn't provide such protection.
   *
   * KeepAlive should *always* be used instead of Executor*. KeepAlive can be
   * implicitly constructed from Executor*. getKeepAliveToken() helper method
   * can be used to construct a KeepAlive in templated code if you need to
   * preserve the original Executor type.
   */
  template <typename ExecutorT = Executor>
  class KeepAlive : private detail::ExecutorKeepAliveBase {
   public:
    using KeepAliveFunc = Function<void(KeepAlive&&)>;

    KeepAlive() = default;

    ~KeepAlive() {
      static_assert(
          std::is_standard_layout<KeepAlive>::value, "standard-layout");
      static_assert(sizeof(KeepAlive) == sizeof(void*), "pointer size");
      static_assert(alignof(KeepAlive) == alignof(void*), "pointer align");

      reset();
    }

    // ...
    // 各种copy/move constructor/operator

    // reset本质上只是引用计数减1
    void reset() noexcept {
      if (Executor* executor = get()) {
        auto const flags = std::exchange(storage_, 0) & kFlagMask;
        if (!(flags & (kDummyFlag | kAliasFlag))) {
          executor->keepAliveRelease();
        }
      }
    }

    explicit operator bool() const { return storage_; }

    // 获取KeepAlive关联的executor
    ExecutorT* get() const {
      return reinterpret_cast<ExecutorT*>(storage_ & kExecutorMask);
    }

    ExecutorT& operator*() const { return *get(); }

    ExecutorT* operator->() const { return get(); }

   // ...

   private:
    friend class Executor;
    template <typename OtherExecutor>
    friend class KeepAlive;

    KeepAlive(ExecutorT* executor, uintptr_t flags) noexcept
        : storage_(reinterpret_cast<uintptr_t>(executor) | flags) {
      assert(executor);
      assert(!(reinterpret_cast<uintptr_t>(executor) & ~kExecutorMask));
      assert(!(flags & kExecutorMask));
    }

    explicit KeepAlive(uintptr_t storage) noexcept : storage_(storage) {}

    //  Combined storage for the executor pointer and for all flags.
    uintptr_t storage_{reinterpret_cast<uintptr_t>(nullptr)};
  };

 public:
  // Exectuor对外暴露的接口
  // ...
 protected:
  // ...

  // Acquire a keep alive token. Should return false if keep-alive mechanism
  // is not supported.
  virtual bool keepAliveAcquire() noexcept;
  // Release a keep alive token previously acquired by keepAliveAcquire().
  // Will never be called if keepAliveAcquire() returns false.
  virtual void keepAliveRelease() noexcept;

  // ...
};
```

`KeepAlive`本身只有一个成员变量，即用来保存`Executor`的地址的一个`uintptr_t`。 要实现其安全指针的特性，关键就在于重载`Executor`中这两个方法`keepAliveAcquire`和`keepAliveRelease`。

如果`Executor`的`keepAliveAcquire`方法返回false，代表这个`Executor`不支持KeepAlive机制，例如`InlineExecutor`，`QueuedImmediateExecutor`等。而如果`Executor`的`keepAliveAcquire`方法返回true，则需要在`Executor`内部，通过诸如引用计数的方式，保证`Executor`的生命周期。典型例子就是`Executor`一旦通过`add`添加若干回调之后，`Executor`在这些回调执行完之前不能析构。

> 无论`Executor`是否支持KeepAlive机制，`KeepAlive`的使用方式都是和`Executor`的指针一样的。
>

`KeepAlive`的引用计数原理之后我们会详细分析，在此之前，我们先简单理解`KeepAlive`的使用方式。

### 构造KeepAlive

获取`KeepAlive`的方式分成几类:

- 从`Executor*`隐式转换

    ```cpp
    /* implicit */ KeepAlive(ExecutorT* executor) {
      *this = getKeepAliveToken(executor);
    }
    ```

- `Executor`的static方法

    ```cpp
    class Executor {
     public:
      // ...

      template <typename ExecutorT>
      static KeepAlive<ExecutorT> getKeepAliveToken(ExecutorT* executor) {
        static_assert(
            std::is_base_of<Executor, ExecutorT>::value,
            "getKeepAliveToken only works for folly::Executor implementations.");
        if (!executor) {
          return {};
        }
        folly::Executor* executorPtr = executor;
        if (executorPtr->keepAliveAcquire()) {
          // 构造一个正常的KeepAlive对象
          return makeKeepAlive<ExecutorT>(executor);
        }
        // 构造一个没有keep-alive机制的KeepAlive对象, 所谓的dummy
        return makeKeepAliveDummy<ExecutorT>(executor);
      }

      template <typename ExecutorT>
      static KeepAlive<ExecutorT> getKeepAliveToken(ExecutorT& executor) {
        static_assert(
            std::is_base_of<Executor, ExecutorT>::value,
            "getKeepAliveToken only works for folly::Executor implementations.");
        return getKeepAliveToken(&executor);
      }

    };
    ```

- 全局函数

    ```cpp
    /// Returns a keep-alive token which guarantees that Executor will keep
    /// processing tasks until the token is released (if supported by Executor).
    /// KeepAlive always contains a valid pointer to an Executor.
    template <typename ExecutorT>
    Executor::KeepAlive<ExecutorT> getKeepAliveToken(ExecutorT* executor) {
      static_assert(
          std::is_base_of<Executor, ExecutorT>::value,
          "getKeepAliveToken only works for folly::Executor implementations.");
      return Executor::getKeepAliveToken(executor);
    }

    template <typename ExecutorT>
    Executor::KeepAlive<ExecutorT> getKeepAliveToken(ExecutorT& executor) {
      static_assert(
          std::is_base_of<Executor, ExecutorT>::value,
          "getKeepAliveToken only works for folly::Executor implementations.");
      return getKeepAliveToken(&executor);
    }

    template <typename ExecutorT>
    Executor::KeepAlive<ExecutorT> getKeepAliveToken(
        Executor::KeepAlive<ExecutorT>& ka) {
      return ka.copy();
    }
    ```


以上接口如果`Executor`支持KeepAlive机制，最终都会调用到`Executor`中的`makeKeepAlive`

```cpp
  template <typename ExecutorT>
  static KeepAlive<ExecutorT> makeKeepAlive(ExecutorT* executor) {
    static_assert(
        std::is_base_of<Executor, ExecutorT>::value,
        "makeKeepAlive only works for folly::Executor implementations.");
    return KeepAlive<ExecutorT>{executor, uintptr_t(0)};
  }
```

仔细看下构造函数和相关的flag，在构造时会将`storage_`初始化为`Executor*`按位或`flag`。

```cpp
KeepAlive(ExecutorT* executor, uintptr_t flags) noexcept
    : storage_(reinterpret_cast<uintptr_t>(executor) | flags) {
  assert(executor);
  assert(!(reinterpret_cast<uintptr_t>(executor) & ~kExecutorMask));
  assert(!(flags & kExecutorMask));
}

class ExecutorKeepAliveBase {
 public:
  //  A dummy keep-alive is a keep-alive to an executor which does not support
  //  the keep-alive mechanism.
  static constexpr uintptr_t kDummyFlag = uintptr_t(1) << 0;

  //  An alias keep-alive is a keep-alive to an executor to which there is
  //  known to be another keep-alive whose lifetime surrounds the lifetime of
  //  the alias.
  static constexpr uintptr_t kAliasFlag = uintptr_t(1) << 1;

  static constexpr uintptr_t kFlagMask = kDummyFlag | kAliasFlag;
  static constexpr uintptr_t kExecutorMask = ~kFlagMask;
};
```

`flag`之所以可以保存在低位的原因是：64位内存地址的实际上只使用低48位，剩余的高16位是没有使用的。因此将`Executor*`转为一个unsigned int之后低16个bit是没有用的，因此可以将这些`flag`保存在低位中，后续在获取`Executor`的指针时会再去掉`flag`。

### 使用KeepAlive

使用`KeepAlive`的方式就和一个`Executor*`一样：

```cpp
    ExecutorT* get() const {
      return reinterpret_cast<ExecutorT*>(storage_ & kExecutorMask);
    }

    ExecutorT& operator*() const { return *get(); }

    ExecutorT* operator->() const { return get(); }
```

### 释放KeepAlive

当一个`KeepAlive`不再需要时，我们可以通过将其析构，或者主动调用`reset`来释放这个`KeepAlive`。`reset`有两个作用：

- 释放`KeepAlive`内部保存的`Executor*`指针，即之后再也无法通过这个`KeepAlive`调用`Executor`的接口
- 如果`Executor`支持KeepAlive机制，就会调用`keepAliveRelease`，在对应Executor内部进行引用计数的维护。

```cpp
~KeepAlive() {
  static_assert(std::is_standard_layout<KeepAlive>::value, "standard-layout");
  static_assert(sizeof(KeepAlive) == sizeof(void*), "pointer size");
  static_assert(alignof(KeepAlive) == alignof(void*), "pointer align");

  reset();
}

void reset() noexcept {
  if (Executor* executor = get()) {
    auto const flags = std::exchange(storage_, 0) & kFlagMask;
    if (!(flags & (kDummyFlag | kAliasFlag))) {
      executor->keepAliveRelease();
    }
  }
}
```

### Example

在解释更多内部原理之前，我们通过一个简单测试看看KeepAlive的使用方式。

一个常见的错误使用方式就是，如果获取了一个`KeepAlive`之后，没有将其释放，那么对应的`Executor`是永远不会释放（这也正是`KeepAlive`存在的意义所在）。例如下面的代码因为`ka`仍然持有着`pool`的指针，就会永远被阻塞在`pool`的析构（stop或者join也是一样的）

```cpp
TEST(KeepAlive, Case1) {
    auto pool = std::make_unique<folly::CPUThreadPoolExecutor>(2);

    folly::Executor::KeepAlive<> ka = pool.get();
    ka->add([]{ std::cout << "Some work" << std::endl; });

    // the program will be blocked forever
    pool.reset();
    std::cout << "Won't be printed" << std::endl;
}
```

解决的办法也很简单，显示或者隐式的释放掉`KeepAlive`即可。

```cpp
TEST(KeepAlive, Case2) {
    auto pool = std::make_unique<folly::CPUThreadPoolExecutor>(2);

    {
      folly::Executor::KeepAlive<> ka = pool.get();
      ka->add([]{ std::cout << "Some work" << std::endl; sleep(1); });
      // Explicitly reset() will work as well
      // ka.reset();
    }

    // After KeepAlive released, the executor could be release as well
    pool.reset();
}
```

随之带来的另一个常见问题：如果我们对一个`KeepAlive`调用过`reset`之后，就不能再通过`KeepAlive`再调用任何`Executor`的接口了，比如下面的`ka`如果在`reset`之后，再调用`add`就会crash。

```cpp
TEST(KeepAlive, Case3) {
    auto pool = std::make_unique<folly::CPUThreadPoolExecutor>(2);

    folly::Executor::KeepAlive<> ka = pool.get();

    ka->add([]{ std::cout << "Some work" << std::endl; });
    ka.reset();

    // Will crash because of KeepAlive have been reset, the executor is nullptr
    // ka->add([] { std::cout << "Will crash" << std::endl; });
}

```

### KeepAlive的引用计数原理

最常用的`CPUThreaPoolExecutor`和`IOThreadPoolExecutor`都继承自`ThreadPoolExecutor`，`ThreadPoolExecutor`继承自`DefaultKeepAliveExecutor`。

1. 引用计数加一

    我们以`CPUThreaPoolExecutor`为例，在调用`add`时，在将`CPUTask`加入到对应队列之前，会先获取一个`KeepAlive`，此时会调用`keepAliveAcquire`，内部实现是引用计数加一（参见下面`DefaultKeepAliveExecutor`的实现）。因此在这些`CPUTask`执行完成之前，`Executor`不会被析构。

    ```cpp
    template <bool withPriority>
    void CPUThreadPoolExecutor::addImpl(
        Func func,
        int8_t priority,
        std::chrono::milliseconds expiration,
        Func expireCallback) {
      if (withPriority) {
        CHECK(getNumPriorities() > 0);
      }
      CPUTask task(
          std::move(func), expiration, std::move(expireCallback), priority);
      if (auto queueObserver = getQueueObserver(priority)) {
        task.queueObserverPayload() =
            queueObserver->onEnqueued(task.context_.get());
      }

      // It's not safe to expect that the executor is alive after a task is added to
      // the queue (this task could be holding the last KeepAlive and when finished
      // - it may unblock the executor shutdown).
      // If we need executor to be alive after adding into the queue, we have to
      // acquire a KeepAlive.
      bool mayNeedToAddThreads = minThreads_.load(std::memory_order_relaxed) == 0 ||
          activeThreads_.load(std::memory_order_relaxed) <
              maxThreads_.load(std::memory_order_relaxed);
      folly::Executor::KeepAlive<> ka = mayNeedToAddThreads
          ? getKeepAliveToken(this)
          : folly::Executor::KeepAlive<>{};

      auto result = withPriority
          ? taskQueue_->addWithPriority(std::move(task), priority)
          : taskQueue_->add(std::move(task));

      if (mayNeedToAddThreads && !result.reusedThread) {
        ensureActiveThreads();
      }
    }
    ```

    > 当然，有一个疑惑点在于，并没有把KeepAlive传入到CPUTask中，而这个KeepAlive对象在出作用域之后就析构。
    >
2. 引用计数减一

    在`KeepAlive`析构或者`reset`时，会调用`keepAliveRelease`接口，内部实现是引用计数减一（参见下面`DefaultKeepAliveExecutor`的实现）

3. 如何确保`KeepAlive`已经释放

    首先`keepAliveCount_`为引用计数，而`DefaultKeepAliveExecutor`内部自己会持有一个`KeepAlive`。`keepAliveCount_`就等于自身的`KeepAlive`加外部构造的`KeepAlive`的数量。当`keepAliveCount_`为0时，代表所有`KeepAlive`已经释放，此时`Executor`可以析构。

    ```cpp
    class DefaultKeepAliveExecutor : public virtual Executor {

      // ...

     protected:
      void joinKeepAlive() {
        DCHECK(keepAlive_);
        // 释放自己所持有的KeepAlive
        keepAlive_.reset();
        // 等待所有KeepAlive都释放
        keepAliveReleaseBaton_.wait();
      }

      // ...

     private:
      // 引用计数
      struct ControlBlock {
        std::atomic<ssize_t> keepAliveCount_{1};
      };

      // ...

      bool keepAliveAcquire() noexcept override {
        auto keepAliveCount =
            controlBlock_->keepAliveCount_.fetch_add(1, std::memory_order_relaxed);
        // We should never increment from 0
        DCHECK(keepAliveCount > 0);
        return true;
      }

      void keepAliveRelease() noexcept override {
        auto keepAliveCount =
            controlBlock_->keepAliveCount_.fetch_sub(1, std::memory_order_acq_rel);
        DCHECK(keepAliveCount >= 1);

        // 当keepAliveCount为1时 代表keepAliveCount_为0 也就是Executor可以被析构了
        if (keepAliveCount == 1) {
          keepAliveReleaseBaton_.post(); // std::memory_order_release
        }
      }

      std::shared_ptr<ControlBlock> controlBlock_{std::make_shared<ControlBlock>()};
      Baton<> keepAliveReleaseBaton_;
      // 每次构造DefaultKeepAliveExecutor时, 引用计数初始化为1
      KeepAlive<DefaultKeepAliveExecutor> keepAlive_{makeKeepAlive(this)};
    };
    ```

    其本质就是有一个原子变量进行引用计数，当引用计数变为0时，也就是确保没有`KeepAlive`还在使用`Executor`之后，会调用`keepAliveReleaseBaton_.post()`通知`Executor`可以`stop`或者`join`或者析构。

    - `joinKeepAlive`中会释放`DefaultKeepAliveExecutor`内部持有的这个`KeepAlive`对象，并通过`keepAliveReleaseBaton_`等待所有`KeepAlive`释放。
    - 当构造`KeepAlive`时，就会调用`keepAliveAcquire`，引用计数加一。
    - 当析构`KeepAlive`时，就会调用`keepAliveRelease`，引用计数减一。
    - 当引用计数`keepAliveCount_`为0时（注意`keepAliveRelease`中是通过`keepAliveCount == 1来判断的`），所有KeepAlive已经释放，此时会通过`keepAliveReleaseBaton_`告知`joinKeepAlive`完成。

到这里，`KeepAlive`的原理基本已经清晰了~
