---
layout: single
title: folly::Fiber
date: 2024-11-06 00:00:00 +0800
categories: 源码 学习
tags: folly
---

研究下folly的Fiber~

## 简述

这篇文章不会对比Fiber和Coroutine，只简单描述一下Fiber，有机会再单独介绍一下。Fiber是用户态线程，多个Fiber可以在一个线程上运行。同一时刻在一个线程上，只会有一个Fiber正在运行，正在执行的Fiber如果阻塞就会把控制权交给另一个能运行的Fiber。当一个Fiber阻塞时，需要一个调度器来决定该运行哪一个Fiber。我们从folly中的这个调度器，也就是`FiberManger`说起。

## FiberManger

### 数据结构

```cpp
/**
 * @class FiberManager
 * @brief Single-threaded task execution engine.
 *
 * FiberManager allows semi-parallel task execution on the same thread. Each
 * task can notify FiberManager that it is blocked on something (via await())
 * call. This will pause execution of this task and it will be resumed only
 * when it is unblocked (via setData()).
 */
class FiberManager : public ::folly::Executor {
  // ...
  Fiber* activeFiber_{nullptr}; /**< active fiber, nullptr on main context */
  /**
   * Same as active fiber, but also set for functions run from fiber on main
   * context.
   */
  Fiber* currentFiber_{nullptr};

  FiberTailQueue readyFibers_; /**< queue of fibers ready to be executed */
  FiberTailQueue* yieldedFibers_{nullptr}; /**< queue of fibers which have
                                      yielded execution */
  FiberTailQueue fibersPool_; /**< pool of uninitialized Fiber objects */

  GlobalFiberTailQueue allFibers_; /**< list of all Fiber objects owned */
  
  folly::AtomicIntrusiveLinkedList<Fiber, &Fiber::nextRemoteReady_>
      remoteReadyQueue_;

  folly::AtomicIntrusiveLinkedList<RemoteTask, &RemoteTask::nextRemoteTask>
      remoteTaskQueue_;
};
```

FiberManager本质上就是一个Executor的实现，不同的接口对应不同调用场景：

- `addTask / addTaskFuture`: 添加一个Task，必须从`FiberManager`的线程调用。
- `addTaskEager / addTaskEagerFuture`: 添加一个紧急Task，必须从`FiberManager`的线程调用。
- `addTaskRemote / addTaskRemoteFuture`: 添加一个Task，可以从其他线程调用。
- `addTaskFinally / addTaskFinallyEager`: 添加一个Task，当这个Task完成之后，会在MainContext中（也就是系统线程）执行一个Final回调。
- `runInMainContext`:
    - 如果是一个Fiber调用该接口，会抢占当前正在执行的Fiber，即`activeFiber_`，执行完Task之后会归还抢占的Fiber继续执行。
    - 如果不是从Fiber调用该接口，会直接执行Task。

每个Fiber会保存一个Task，即添加的回调。每个Fiber有不同的状态，比如正在运行，挂起，被阻塞等等，而FiberManager会管理和调度这些Fiber。

### 获取FiberManager

```cpp
FiberManager& getFiberManager(
    folly::EventBase& evb, const FiberManager::Options& opts) {
  return detail::ThreadLocalCache<EventBase>::get<void>(0, evb, opts);
}
```

之后会先尝试在`ThreadLocalCache`中查找是否有可用的`FiberManager`，如果`ThreadLocalCache`中没有，则尝试去`GlobalCache`中查找。

```cpp
template <typename EventBaseT>
class ThreadLocalCache {
 public:
  template <typename LocalT>
  static FiberManager& get(
      uint64_t token, EventBaseT& evb, const FiberManager::Options& opts) {
    return instance()->template getImpl<LocalT>(token, evb, opts);
  }
  
  // ...
  
  template <typename LocalT>
  FiberManager& getImpl(
      uint64_t token, EventBaseT& evb, const FiberManager::Options& opts) {
    eraseImpl();

    auto key = make_tuple(&evb, token, std::type_index(typeid(LocalT)));
    auto& fmPtrRef = map_[key];
    if (!fmPtrRef) {
      fmPtrRef = &GlobalCache<EventBaseT>::template get<LocalT>(key, evb, opts);
    }

    DCHECK(fmPtrRef != nullptr);

    return *fmPtrRef;
  }
  
  // ...
  folly::F14NodeMap<Key<EventBaseT>, FiberManager*> map_;
};
```

`GlobalCache`中的相关代码如下，如果仍然找不到，则会构造一个FiberManager。

```cpp
template <typename EventBaseT>
class GlobalCache {
 public:
  template <typename LocalT>
  static FiberManager& get(
      const Key<EventBaseT>& key,
      EventBaseT& evb,
      const FiberManager::Options& opts) {
    return instance().template getImpl<LocalT>(key, evb, opts);
  }
  
  // ...
  
  template <typename LocalT>
  FiberManager& getImpl(
      const Key<EventBaseT>& key,
      EventBaseT& evb,
      const FiberManager::Options& opts) {
    bool constructed = false;
    SCOPE_EXIT {
      if (constructed) {
        evb.runOnDestruction(makeOnEventBaseDestructionCallback(key));
      }
    };

    std::lock_guard<std::mutex> lg(mutex_);

    auto& fmPtrRef = map_[key];

    if (!fmPtrRef) {
      constructed = true;
      auto loopController = std::make_unique<EventBaseLoopController>();
      loopController->attachEventBase(evb);
      fmPtrRef = std::make_unique<FiberManager>(
          LocalType<LocalT>(), std::move(loopController), opts);
    }

    return *fmPtrRef;
  }
  
  std::mutex mutex_;
  folly::F14NodeMap<Key<EventBaseT>, std::unique_ptr<FiberManager>> map_;
};
```

`FiberManger`的构造函数如下：

```cpp
template <typename LocalT>
FiberManager::FiberManager(
    LocalType<LocalT>,
    std::unique_ptr<LoopController> loopController__,
    Options options)
    : loopController_(std::move(loopController__)),
      stackAllocator_(options.guardPagesPerStack),
      options_(preprocessOptions(std::move(options))),
      exceptionCallback_(defaultExceptionCallback),
      fibersPoolResizer_(*this),
      localType_(typeid(LocalT)) {
  loopController_->setFiberManager(this);
}
```

此处会将`FiberManger`和`LoopManager`关联上，常用的`LoopManager`有几种：

- `EventBaseLoopController`
- `ExecutorLoopController`
- `SimpleLoopController`

### 向FiberManager添加Task

```cpp
template <typename F>
void FiberManager::addTask(F&& func, TaskOptions taskOptions) {
  readyFibers_.push_back(
      *createTask(std::forward<F>(func), std::move(taskOptions)));
  ensureLoopScheduled();
}

inline void FiberManager::ensureLoopScheduled() {
  if (isLoopScheduled_) {
    return;
  }

  isLoopScheduled_ = true;
  loopController_->schedule();
}
```

`createTask`会获取一个Fiber，并把Task保存在Fiber中。之后会将Fiber保存到`readyFibers_`中，即当前可以执行但还未执行的的Fiber，然后调用`LoopManager`开始调度，不同`LoopManager`的实现不同：

- `EventBaseLoopController`
    
    ```cpp
    inline void EventBaseLoopController::schedule() {
      if (eventBase_ == nullptr) {
        // In this case we need to postpone scheduling.
        awaitingScheduling_ = true;
      } else {
        // Schedule it to run in current iteration.
    
        if (!eventBaseKeepAlive_) {
          eventBaseKeepAlive_ = getKeepAliveToken(eventBase_);
        }
        // EventBaseLoopController本身就是个callback
        eventBase_->getEventBase().runInLoop(&callback_, true);
        awaitingScheduling_ = false;
      }
    }
    
    void EventBase::runInLoop(
        LoopCallback* callback,
        bool thisIteration,
        std::shared_ptr<RequestContext> rctx) {
      dcheckIsInEventBaseThread();
      callback->cancelLoopCallback();
      callback->context_ = std::move(rctx);
      if (runOnceCallbacks_ != nullptr && thisIteration) {
        runOnceCallbacks_->push_back(*callback);
      } else {
        loopCallbacks_.push_back(*callback);
      }
    }
    ```
    
    调用`runInLoop`将`EventBaseLoopController`中的`ControllerCallback`作为一个回调注册到`EventBase`中。
    
    ```cpp
    class ControllerCallback : public folly::EventBase::LoopCallback {
     public:
      explicit ControllerCallback(EventBaseLoopController& controller)
          : controller_(controller) {}
    
      void runLoopCallback() noexcept override { controller_.runLoop(); }
    
     private:
      EventBaseLoopController& controller_;
    };
    ```
    
- `ExecutorLoopController`
    
    ```cpp
    inline void ExecutorLoopController::schedule() {
      // add() is thread-safe, so this isn't properly optimized for addTask()
      if (!executorKeepAlive_) {
        executorKeepAlive_ = getKeepAliveToken(executor_);
      }
      auto guard = localCallbackControlBlock_->trySchedule();
      if (!guard) {
        return;
      }
      executor_->add([this, guard = std::move(guard)]() {
        if (guard->isCancelled()) {
          return;
        }
        runLoop();
      });
    }
    ```
    

## Fiber调度和执行

`FiberManager`调度Fiber的核心函数如下所示：

1. 首先一直运行`readyFibers_`中的Fiber，直到`readyFibers_`为空，这部分Fiber是通过`FiberManager::addTask`接口添加的Fiber。
2. 然后将`remoteReadyQueue_`中的Fiber都取出并执行，这部分Fiber是通过`Fiber::resume`添加的Fiber。
3. 最终将`remoteTaskQueue_`中的Fiber取出并执行，这部分Fiber是通过`FiberManager::addTaskRemote`添加的Fiber。

```cpp
inline void FiberManager::loopUntilNoReadyImpl() {
  runFibersHelper([&] {
    SCOPE_EXIT { isLoopScheduled_ = false; };

    bool hadRemote = true;
    while (hadRemote) {
      while (!readyFibers_.empty()) {
        auto& fiber = readyFibers_.front();
        readyFibers_.pop_front();
        runReadyFiber(&fiber);
      }

      auto hadRemoteFiber = remoteReadyQueue_.sweepOnce(
          [this](Fiber* fiber) { runReadyFiber(fiber); });

      if (hadRemoteFiber) {
        ++remoteCount_;
      }

      auto hadRemoteTask =
          remoteTaskQueue_.sweepOnce([this](RemoteTask* taskPtr) {
            std::unique_ptr<RemoteTask> task(taskPtr);
            auto fiber = getFiber();
            if (task->localData) {
              fiber->localData_ = *task->localData;
            }
            fiber->rcontext_ = std::move(task->rcontext);

            fiber->setFunction(std::move(task->func), TaskOptions());
            runReadyFiber(fiber);
          });

      if (hadRemoteTask) {
        ++remoteCount_;
      }

      hadRemote = hadRemoteTask || hadRemoteFiber;
    }
  });
}
```

要理解Fiber的执行逻辑，我们需要先理解Fiber的不同状态。

```cpp
  enum State {
    INVALID, /**< Does't have task function */
    NOT_STARTED, /**< Has task function, not started */
    READY_TO_RUN, /**< Was started, blocked, then unblocked */
    RUNNING, /**< Is running right now */
    AWAITING, /**< Is currently blocked */
    AWAITING_IMMEDIATE, /**< Was preempted to run an immediate function,
                             and will be resumed right away */
    YIELDED, /**< The fiber yielded execution voluntarily */
  };
```

各个状态的转换如下：

![figure]({{'/archive/Fiber-1.png' | prepend: site.baseurl}})

- (1) Fiber初始状态为INVALID，`FiberManager::addTask`之后切换为NOT_STARTED。
- (2) 在执行具体Task回调之前，Fiber状态会切换为RUNNING。
- (3) 在执行具体Task回调之后，Fiber状态会切换为INVALID。
- (4) `FiberManager::yield`可以挂起当前Fiber，其状态切换为YIELDED。
- (5) 正在执行的Task可能也是异步的，如果执行过程被阻塞，会切换到AWAITING状态。
- (6) `FiberManager::runInMainContext`接口会将当前Fiber修改为AWAITING_IMMEDIATE状态，即直接抢占系统线程，执行传入的回调，然后在切换到之前的Fiber。
- (7) 被挂起的Fiber，在所有其他Fiber都执行完或者被阻塞后，会得到重新运行运行的机会，此时状态会切换为READY_TO_RUN。
- (8) 被阻塞的Fiber不再阻塞后，会通过`Fiber::resume`接口将状态修改为READY_TO_RUN，以便再次运行。
- (9) 对应第(6)步，被抢占的Fiber会停留在AWAITING_IMMEDIATE状态，直到`FiberManager::runInMainContext`执行完之后，状态会切换为READY_TO_RUN，以便再次运行。
- (10) 主动或被动被阻塞的Fiber即 (789)，都会在抢占完成之后，重新切换状态为RUNNING再次运行。

### Fiber如何阻塞以及恢复

在开头提到正在执行的Fiber如果阻塞就会把控制权交给另一个能运行的Fiber，我们举一个例子来立即。如果有一个异步接口返回了一个Future：

```cpp
folly::Future<Response> asyncCallFuture(Request request);
```

通常当我们调用`Future::get`时，就会阻塞线程，而在Fiber中调用`Future::get`时，不再会阻塞线程，而会自动切换到其他可以运行的Fiber。

```cpp
fiberManager.addTask([]() {
  ...
  auto response = asyncCallFuture(request).get();
// Now response holds response returned by the async call
...
}
```

我们看下阻塞时都发生了什么，阻塞的路径如下：

```cpp
Future::wait or SemiFuture::wait
-> futures::detail::waitImpl
  -> fibers::Baton::wait
    -> fibers::Baton::waitFiber
```

实际上Future兼容了Fiber中的Baton，当调用`Future::wait`或者`SemiFuture::wait`时，就会检查当前线程是否存在`FiberManager`，以及`FiberManager`中是否有正在运行的Fiber。如果都满足，表示这是一个Fiber线程，调用`waitFiber`进行等待。否则调用普通的`waitThread`等待系统线程。

> `Future::get`也会调用`Future::wait`
> 

```cpp
// folly/futures/Future-inl.h
template <class FutureType, typename T = typename FutureType::value_type>
void waitImpl(FutureType& f) {
  if (std::is_base_of<Future<T>, FutureType>::value) {
    f = std::move(f).via(&InlineExecutor::instance());
  }
  // short-circuit if there's nothing to do
  if (f.isReady()) {
    return;
  }

  Promise<T> promise;
  auto ret = convertFuture(promise.getSemiFuture(), f);
  // 这里的FutureBatonType就是fibers::Baton
  FutureBatonType baton;
  f.setCallback_([&baton, promise = std::move(promise)](
                     Executor::KeepAlive<>&&, Try<T>&& t) mutable {
    promise.setTry(std::move(t));
    baton.post();
  });
  f = std::move(ret);
  baton.wait();
  assert(f.isReady());
}

// folly/fibers/Baton-inl.h
void Baton::wait() {
  wait([]() {});
}

template <typename F>
void Baton::wait(F&& mainContextFunc) {
  auto fm = FiberManager::getFiberManagerUnsafe();
  if (!fm || !fm->activeFiber_) {
    mainContextFunc();
    return waitThread();
  }

  return waitFiber(*fm, std::forward<F>(mainContextFunc));
}
```

`waitFiber`的相关实现如下：

- 在调用`Baton::wait`时会指定`mainContextFunc`，即在系统线程执行的回调，此处为空
- 在调用`Baton::waitFiber`时会指定FiberManager的`awaitFunc_`：
    - 初始化`FiberWaiter`，并通过`Baton::setWaiter`保存到Baton中
    - 当Fiber不再阻塞后 ，即变成READY_TO_RUN时，会通过`FiberWaiter`最终调用`Fiber::resume`

```cpp
class Baton::FiberWaiter : public Baton::Waiter {
 public:
  void setFiber(Fiber& fiber) {
    DCHECK(!fiber_);
    fiber_ = &fiber;
  }

  void post() override { fiber_->resume(); }

 private:
  Fiber* fiber_{nullptr};
};

// ...

template <typename F>
void Baton::waitFiber(FiberManager& fm, F&& mainContextFunc) {
  FiberWaiter waiter;
  auto f = [this, &mainContextFunc, &waiter](Fiber& fiber) mutable {
    waiter.setFiber(fiber);
    setWaiter(waiter);

    mainContextFunc();
  };

  fm.awaitFunc_ = std::ref(f);
  fm.activeFiber_->preempt(Fiber::AWAITING);
}

void Baton::setWaiter(Waiter& waiter) {
  auto curr_waiter = waiter_.load();
  do {
    if (LIKELY(curr_waiter == NO_WAITER)) {
      continue;
    } else if (curr_waiter == POSTED || curr_waiter == TIMEOUT) {
      waiter.post();
      break;
    } else {
      throw std::logic_error("Some waiter is already waiting on this Baton.");
    }
  } while (!waiter_.compare_exchange_weak(
      curr_waiter, reinterpret_cast<intptr_t>(&waiter)));
}
```

恢复的路径相对比较简单：

```cpp
Baton::post
-> Baton::postHelper
  -> Baton::FiberWaiter::post
    -> Fiber::resume
```

最终调用`Fiber::resume`，将其加入到`readyFibers_`中，之后就会被调度运行。

```cpp
void Fiber::resume() {
  DCHECK_EQ(state_, AWAITING);
  state_ = READY_TO_RUN;

  if (LIKELY(threadId_ == localThreadId())) {
    fiberManager_.readyFibers_.push_back(*this);
    fiberManager_.ensureLoopScheduled();
  } else {
    fiberManager_.remoteReadyInsert(this);
  }
}
```

其他的状态转换都比较简单，不再过多阐述，核心逻辑都在`FiberManager::runReadyFiber`中。

### Fiber如何执行

FiberManager的核心功能在前面已经描述过了，但还遗留一个问题，我们开头提到Fiber是用户态线程，多个Fiber可以在一个线程上运行。这是如何实现的呢？

首先我们看Fiber的构造，此处会构造一个`FiberImpl`，即`boost::context::detail::fcontext_t`，实际上就是一个boost提供的用户态线程上下文切换的context。

```cpp
Fiber::Fiber(FiberManager& fiberManager)
    : fiberManager_(fiberManager),
      fiberStackSize_(fiberManager_.options_.stackSize),
      fiberStackLimit_(fiberManager_.stackAllocator_.allocate(fiberStackSize_)),
      fiberImpl_([this] { fiberFunc(); }, fiberStackLimit_, fiberStackSize_) {
  fiberManager_.allFibers_.push_back(*this);

```

上下文切换需要的context会保存在栈上，这块内存也是在构造`Fiber`时分配，起始地址为`fiberStackLimit_`，大小为`fiberStackSize_`，连同传入的回调`fiberFunc`，一并构造`FiberImpl`。

> 栈默认大小为16K，因此官方文档也提醒了我们需要控制Fiber的数量，以免context占用过多内存
> 

```cpp
FiberImpl(
    folly::Function<void()> func, unsigned char* stackLimit, size_t stackSize)
    : func_(std::move(func)) {
  auto stackBase = stackLimit + stackSize;
  stackBase_ = stackBase;
  fiberContext_ =
      boost::context::detail::make_fcontext(stackBase, stackSize, &fiberFunc);
}
```

在构造中会生成上下文，后续当切换到`fiberContext_`时，就会开始执行`fiberFunc`，即下面的函数，其中的`func_`就是添加的Task回调。

```cpp
[[noreturn]] void Fiber::fiberFunc() {
#ifdef FOLLY_SANITIZE_ADDRESS
  fiberManager_.registerFinishSwitchStackWithAsan(
      nullptr, &asanMainStackBase_, &asanMainStackSize_);
#endif

  while (true) {
    DCHECK_EQ(state_, NOT_STARTED);

    threadId_ = localThreadId();
    if (taskOptions_.logRunningTime) {
      prevDuration_ = std::chrono::microseconds(0);
      currStartTime_ = std::chrono::steady_clock::now();
    }
    state_ = RUNNING;

    try {
      if (resultFunc_) {
        DCHECK(finallyFunc_);
        DCHECK(!func_);

        resultFunc_();
      } else {
        DCHECK(func_);
        func_();
      }
    } catch (...) {
      fiberManager_.exceptionCallback_(
          std::current_exception(), "running Fiber func_/resultFunc_");
    }

    if (UNLIKELY(recordStackUsed_)) {
      auto newHighWatermark = fiberManager_.recordStackPosition(
          nonMagicInBytes(fiberStackLimit_, fiberStackSize_));
      VLOG(3) << "Max stack usage: " << newHighWatermark;
      CHECK_LT(newHighWatermark, fiberManager_.options_.stackSize - 64)
          << "Fiber stack overflow";
    }

    state_ = INVALID;

    fiberManager_.deactivateFiber(this);
  }
}
```

触发上下文切换的逻辑也比较简单，当FiberManager在`readyFibers_`中找到可以执行的Fiber之后，就将当前线程切换到Fiber的上下文：

```cpp
FiberManager::runReadyFiber
-> FiberManager::activateFiber
  -> FiberImpl::activate

void activate() {
  auto transfer = boost::context::detail::jump_fcontext(fiberContext_, this);
  fiberContext_ = transfer.fctx;
  auto context = reinterpret_cast<intptr_t>(transfer.data);
  DCHECK_EQ(0, context);
}
```

这里截了个图，看看执行`auto transfer = boost::context::detail::jump_fcontext(fiberContext_, this);`前后的上下文。执行前：

![figure]({{'/archive/Fiber-2.png' | prepend: site.baseurl}})

执行后就直接开始执行`fiberFunc`了：

![figure]({{'/archive/Fiber-3.png' | prepend: site.baseurl}})

当Fiber执行完之后，就会调用`FiberManager::deactivateFiber`，恢复到运行这个Fiber之前的上下文。

```cpp
Fiber::fiberFunc
-> FiberManager::deactivateFiber
  -> FiberImpl::deactivate

void deactivate() {
  auto transfer =
      boost::context::detail::jump_fcontext(mainContext_, nullptr);
  mainContext_ = transfer.fctx;
  fixStackUnwinding();
  auto context = reinterpret_cast<intptr_t>(transfer.data);
  DCHECK_EQ(this, reinterpret_cast<FiberImpl*>(context));
}
```

## 写在最后

folly的Fiber的实现有点意思，除了提供了通常意义上的用户态线程以及上下文切换之外，最棒的一点在于它兼容了folly的Future，改造成本大大降低。另外如果想要进一步了解Fiber的实现原理，可以看看这篇[博客](https://agraphicsguynotes.com/posts/fiber_in_cpp_understanding_the_basics/)。