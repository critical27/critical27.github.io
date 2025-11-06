---
layout: single
title: folly::Future defer
date: 2022-03-19 00:00:00 +0800
categories: 源码
tags: C++ folly async
---

Future的第二弹。

## introduction

**只有`SemiFutre`才有`defer`这一系列函数**:

* defer
* deferValue
* deferError

下面是比较重要的一段注释

```c++
  /// Defer work to run on the consumer of the future.
  /// Function must take a Try as a parameter.
  /// This work will be run either on an executor that the caller sets on the
  /// SemiFuture, or inline with the call to .get().
  ///
  /// All forms of defer will run the continuation inline with the execution of
  /// the  previous callback in the chain if the callback attached to the
  /// previous future that triggers execution of func runs on the same executor
  /// that func would be executed on.
```

简而言之:

1. defer的函数要么在producer的线程执行(也就是调用SemiFuture::setValue的线程)，要么在consumer的线程执行(也就是调用SemiFuture::get或者SemiFuture::via的线程)
2. 如果执行上一个回调的executor，和defer的函数执行的executor，是同一个executor，defer的函数会inline执行

## example

我们看下面一个简单的例子，`innerResult`只会在`std::move(sf).get()`之后才会被设置为7。

```c++
TEST(SemiFuture, Defer) {
  {
    std::atomic<int> innerResult{0};
    Promise<int> p;
    auto sf = p.getSemiFuture().deferValue([&](int a) { innerResult = a; });
    p.setValue(7);
    ASSERT_EQ(innerResult, 0);
    sleep(1);
    ASSERT_EQ(innerResult, 0);
    // deferValue的callback实际是在SemiFuture::get的时候调用的
    std::move(sf).get();
    ASSERT_EQ(innerResult, 7);
  }
  /*
  {
    auto ioPool = std::make_shared<IOThreadPoolExecutor>(1);
    std::atomic<int> innerResult{0};
    Promise<int> p;
    auto sf = p.getSemiFuture().via(ioPool.get()).thenValue([&](int a) { innerResult = a; });
    p.setValue(7);
    sleep(1);
    // thenValue的callback在p.setValue时候就会执行
    ASSERT_EQ(innerResult, 7);
    std::move(sf).get();
    ASSERT_EQ(innerResult, 7);
  }
  */
}
```

我们分别看下在`deferValue`，`setValue`和`get`这三步分别发生了什么。

### SemiFuture::defer

先看`auto sf = p.getSemiFuture().deferValue([&](int a) { innerResult = a; });`

```c++
template <class T>
template <typename F>
SemiFuture<typename futures::detail::valueCallableResult<T, F>::value_type>
SemiFuture<T>::deferValue(F&& func) && {
  return std::move(*this).defer(
      [f = static_cast<F&&>(func)](folly::Try<T>&& t) mutable {
        return futures::detail::wrapInvoke(std::move(t), static_cast<F&&>(f));
      });
}

template <class T>
template <typename F>
SemiFuture<typename futures::detail::tryCallableResult<T, F>::value_type>
SemiFuture<T>::defer(F&& func) && {
  // 检查当前semifuture是否有deferedExecutor
  auto deferredExecutorPtr = this->getDeferredExecutor();
  // 有则copy没有则create
  futures::detail::KeepAliveOrDeferred deferredExecutor = [&]() {
    if (deferredExecutorPtr) {
      return futures::detail::KeepAliveOrDeferred{deferredExecutorPtr->copy()};
    } else {
      // 这个SemiFuture没有deferredExecutor 所以会创建一个
      auto newDeferredExecutor = futures::detail::KeepAliveOrDeferred(
          futures::detail::DeferredExecutor::create());
      this->setExecutor(newDeferredExecutor.copy());
      return newDeferredExecutor;
    }
  }();

  // 重新构造一个SemiFuture (用同一个core 把func以Inline加到这个SemiFuture后面)
  auto sf = Future<T>(this->core_).thenTryInline(static_cast<F&&>(func)).semi();
  this->core_ = nullptr;
  // Carry deferred executor through chain as constructor from Future will
  // nullify it
  // 设置defer的函数后面执行的executor
  sf.setExecutor(std::move(deferredExecutor));
  return sf;
}
```

首先检查当前semifuture是否有DeferredExecutor，有则copy没有则create

```c++
class FutureBase {
  ...
  DeferredExecutor* getDeferredExecutor() const {
    return getCore().getDeferredExecutor();
  }
  ...
};

DeferredExecutor* CoreBase::getDeferredExecutor() const {
  if (!executor_.isDeferred()) {
    return {};
  }

  return executor_.getDeferredExecutor();
}

DeferredExecutor* KeepAliveOrDeferred::getDeferredExecutor() const noexcept {
  switch (state_) {
    case State::Deferred:
      return deferred_.get();
    case State::KeepAlive:
      return nullptr;
  }
  assume_unreachable();
}
```

然后重新构造一个SemiFuture (用同一个core 把func以Inline形式加到这个SemiFuture后面)

`auto sf = Future<T>(this->core_).thenTryInline(static_cast<F&&>(func)).semi();`

```c++
template <class T>
template <typename F>
Future<typename futures::detail::tryCallableResult<T, F>::value_type>
Future<T>::thenTryInline(F&& func) && {
  auto lambdaFunc = [f = static_cast<F&&>(func)](
                        folly::Executor::KeepAlive<>&&,
                        folly::Try<T>&& t) mutable {
    return static_cast<F&&>(f)(std::move(t));
  };
  using R = futures::detail::tryExecutorCallableResult<T, decltype(lambdaFunc)>;
  return this->thenImplementation(
      std::move(lambdaFunc), R{}, futures::detail::InlineContinuation::permit);
}
```

后面可以参考[之前的分析](https://critical27.github.io/c++/2022/02/18/future-deadlock.html)，这个core的初始状态时`Start`，因为是`defer`调用的是`thenTryInline`允许inline执行，所以core的状态会被设置为`State::OnlyCallbackAllowInline`，然后就返回，到这里`defer`就完成了，本质上只是注册了一个callback。

```c++
void CoreBase::setCallback_(
    Callback&& callback,
    std::shared_ptr<folly::RequestContext>&& context,
    futures::detail::InlineContinuation allowInline) {
  DCHECK(!hasCallback());

  // 保存callback和context 以便后面执行
  ::new (&callback_) Callback(std::move(callback));
  ::new (&context_) Context(std::move(context));

  auto state = state_.load(std::memory_order_acquire);
  State nextState = allowInline == futures::detail::InlineContinuation::permit
      ? State::OnlyCallbackAllowInline
      : State::OnlyCallback;

  if (state == State::Start) {
    if (folly::atomic_compare_exchange_strong_explicit(
            &state_,
            &state,
            nextState,
            std::memory_order_release,
            std::memory_order_acquire)) {
      return;
    }
    assume(state == State::OnlyResult || state == State::Proxy);
  }

  ...
}
```

### Promise::setValue

对于`then`系列的函数，当callback已经设置好后，一旦调用`Promise::setValue`，core的状态就会从`OnlyCallback`转换为`Done`，然后开始执行`then`的回调。

我们来看看`defer`系列是如何处理的：

首先找到对应的core，然后发现core的状态是`OnlyCallbackAllowInline`，调用`doCallback`

```c++
void CoreBase::setResult_(Executor::KeepAlive<>&& completingKA) {
  DCHECK(!hasResult());

  auto state = state_.load(std::memory_order_acquire);
  switch (state) {
    case State::Start:
      if (folly::atomic_compare_exchange_strong_explicit(
              &state_,
              &state,
              State::OnlyResult,
              std::memory_order_release,
              std::memory_order_acquire)) {
        return;
      }
      assume(
          state == State::OnlyCallback ||
          state == State::OnlyCallbackAllowInline);
      FOLLY_FALLTHROUGH;

    case State::OnlyCallback:
    case State::OnlyCallbackAllowInline:
      state_.store(State::Done, std::memory_order_relaxed);
      doCallback(std::move(completingKA), state);
      return;
    case State::OnlyResult:
    case State::Proxy:
    case State::Done:
    case State::Empty:
    default:
      terminate_with<std::logic_error>("setResult unexpected state");
  }
}
```

在`doCallback`里面会调用这个`doAdd`，把之前`defer`的函数加入到exeutor的队列中。

**这里实际上是`deferValue`和`thenValue`最重要的差别之处**：

* `defer`系列的函数由于指定了`deferredExecutor`都只会把函数调用`DeferredExecutor::addFrom`(只是加到executor的队列中而不会实际执行)
* 而`then`系列函数如果是inline则直接在当前线程调用这个函数，如果不是inline则调用`Executor::add`，由对应executor执行。

```c++
void CoreBase::doCallback(
    Executor::KeepAlive<>&& completingKA, State priorState) {
  DCHECK(state_ == State::Done);

  // 把之前在defer()里面设置的executor取出来
  auto executor = std::exchange(executor_, KeepAliveOrDeferred{});

  // Customise inline behaviour
  // If addCompletingKA is non-null, then we are allowing inline execution
  auto doAdd = [](Executor::KeepAlive<>&& addCompletingKA,
                  KeepAliveOrDeferred&& currentExecutor,
                  auto&& keepAliveFunc) mutable {
    if (auto deferredExecutorPtr = currentExecutor.getDeferredExecutor()) {
      // 由于有deferredExecutor 所以会从这走
      deferredExecutorPtr->addFrom(
          std::move(addCompletingKA), std::move(keepAliveFunc));
    } else {
      // If executors match call inline
      // 而then系列的函数都会走下面分支
      auto currentKeepAlive = std::move(currentExecutor).stealKeepAlive();
      if (addCompletingKA.get() == currentKeepAlive.get()) {
        // inline则直接调用函数
        keepAliveFunc(std::move(currentKeepAlive));
      } else {
        // 否则调用Executor::add
        std::move(currentKeepAlive).add(std::move(keepAliveFunc));
      }
    }
  };

  ...
}
```

`addFrom`函数的作用如下:

* run func inline if there is a stored executor and completingKA matches the stored executor
* enqueue func into the stored executor if one exists
* store func until an executor is set otherwise

其中`DeferredExecutor`有四种状态

```c++
enum class State {
  EMPTY,          // 初始状态
  HAS_FUNCTION,   // 已经设置了需要defer的函数
  HAS_EXECUTOR,   // 已经设置了负责执行的executor
  DETACHED        // 函数和executor都有就会变为DETACHED
};
```

```c++
void DeferredExecutor::addFrom(
    Executor::KeepAlive<>&& completingKA,
    Executor::KeepAlive<>::KeepAliveFunc func) {
  auto state = state_.load(std::memory_order_acquire);
  // 已经detach 不知道这个func还会不会执行 按我理解是不执行了
  if (state == State::DETACHED) {
    return;
  }

  // If we are completing on the current executor, call inline, otherwise
  // add
  auto addWithInline =
      [&](Executor::KeepAlive<>::KeepAliveFunc&& addFunc) mutable {
        if (completingKA.get() == executor_.get()) {
          addFunc(std::move(completingKA));
        } else {
          executor_.copy().add(std::move(addFunc));
        }
      };

  if (state == State::HAS_EXECUTOR) {
    addWithInline(std::move(func));
    return;
  }
  DCHECK(state == State::EMPTY);
  func_ = std::move(func);
  if (folly::atomic_compare_exchange_strong_explicit(
          &state_,
          &state,
          State::HAS_FUNCTION,
          std::memory_order_release,
          std::memory_order_acquire)) {
    return;
  }
  DCHECK(state == State::DETACHED || state == State::HAS_EXECUTOR);
  if (state == State::DETACHED) {
    std::exchange(func_, nullptr);
    return;
  }
  addWithInline(std::exchange(func_, nullptr));
}
```

对于我们这个例子里，`DeferredExecutor`的状态是`EMPTY`，然后会设置函数，然后转换为`HAS_FUNCTION`退出，等待后续设置了Executor时调用。

### SemiFuture::get

我们会发现`defer`的回调还没有执行，那么是在哪执行的呢？

```c++
template <class T>
T SemiFuture<T>::get() && {
  return std::move(*this).getTry().value();
}

template <class T>
Try<T> SemiFuture<T>::getTry() && {
  wait();
  auto future = folly::Future<T>(this->core_);
  this->core_ = nullptr;
  return std::move(std::move(future).result());
}

template <class T>
SemiFuture<T>& SemiFuture<T>::wait() & {
  if (auto deferredExecutor = this->getDeferredExecutor()) {
    // Make sure that the last callback in the future chain will be run on the
    // WaitExecutor.
    Promise<T> promise;
    auto ret = promise.getSemiFuture();
    setCallback_(
        [p = std::move(promise)](Executor::KeepAlive<>&&, auto&& r) mutable {
          p.setTry(std::move(r));
        });
    auto waitExecutor = futures::detail::WaitExecutor::create();
    deferredExecutor->setExecutor(waitExecutor.copy());
    while (!ret.isReady()) {
      waitExecutor->drive();
    }
    waitExecutor->detach();
    this->detach();
    *this = std::move(ret);
  } else {
    futures::detail::waitImpl(*this);
  }
  return *this;
}
```

可以发现在`wait`的时候如果有`deferredExecutor`，就会新建一个`Promise`，然后再创建一个`WaitExecutor`来执行之前推迟的回调。

而触发`defer`的函数的入口是在`waitExecutor->drive();`这里

```c++
class WaitExecutor final : public folly::Executor {
  ...

  void drive() {
    baton_.wait();
    fibers::runInMainContext([&]() {
      baton_.reset();
      auto funcs = std::move(queue_.wlock()->funcs);
      for (auto& func : funcs) {
        std::exchange(func, nullptr)();
      }
    });
  }

  ...
};
```

在`drive`里面实际就是把之前`defer`的函数都一次取出来然后开始执行。

`fibers::runInMainContext`是之前看`fiber`时候遇到的函数，下面是README里面的介绍：

fibers::runInMainContext will switch to the stack of the system thread (main context), run the functor passed to it and then switch back to the fiber-task stack.

IMPORTANT: Make sure you don't do any blocking calls on main context though. It will suspend the whole system thread, not just the fiber-task which was running.
Remember that it's fine to use fibers::runInMainContext in general purpose functions (those which may be called both from fiber-task and non from fiber-task). When called in non-fiber-task context fibers::runInMainContext would simply execute passed functor right away.

简单来说，`runInMainContext`就是在system thread执行这个函数。

```c++
template <typename F>
invoke_result_t<F> inline runInMainContext(F&& func) {
  auto fm = FiberManager::getFiberManagerUnsafe();
  if (UNLIKELY(fm == nullptr)) {
    // 走的这里
    return runNoInline(std::forward<F>(func));
  }
  return fm->runInMainContext(std::forward<F>(func));
}

// 实际上就是调用这个func
template <class F>
FOLLY_NOINLINE invoke_result_t<F> runNoInline(F&& func) {
  return func();
}
```

`drive`里面调用的`func`之后就依次触发这几个地方的回调：

```
`CoreBase::doCallback`里面的`doAdd`添加的那个lambda
  -> 这个lambda里面又触发了`Core::setCallback`里面的那个callback
    -> 然后是`FutureBase<T>::thenImplementation`里面的`this->setCallback_`添加的lambda
      -> 然后是`Future<T>::thenTryInline`里面的lambda
        -> 最后才到我们代码里的`[&](int a) { innerResult = a; }`
```