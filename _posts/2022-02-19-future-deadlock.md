---
layout: single
title: folly::Future deadlock example
date: 2022-02-19 00:00:00 +0800
categories: C++ folly
tags: C++ folly async
---

Future的第一弹。

## 使用folly::Future姿势不对导致死锁

直接上代码，简化实现如下所示：

```c++
nebula::cpp2::ErrorCode Host::startSendSnapshot() {
  CHECK(!lock_.try_lock());
  if (!sendingSnapshot_) {
    sendingSnapshot_ = true;
    part_->snapshot_->sendSnapshot(part_, addr_)
        .thenValue([self = shared_from_this()](auto&& status) {
          std::lock_guard<std::mutex> g(self->lock_);
          self->sendingSnapshot_ = false;
          ...
        });
  } else {
    ...
  }
}
```

`startSendSnapshot`函数一定会拿到`lock_`，然后检查是否已经在发送存量数据：

* 如果没有正在发送，设置`sendingSnapshot_`并开始发送
* 如果已经正在发送，则什么也不做

`sendSnapshot`函数会返回一个`Future<Status>`用来表示是否发送成功。这个函数中的锁`lock_`其实就是用来防护`sendingSnapshot_`这个变量，当发送完snapshot后，我们会把再次去拿锁，并置为false。

乍一看这个代码实际上并没有太多违和感，但最近在测试的时候发现了一个有趣的死锁现象，不是每次能够复现，但是测试时间足够是能够稳定复现的。死锁时候pstack对应的栈看[这里]({{'/archive/future-deadlock-pstack.txt' | prepend: site.baseurl}})。

pstack中的Frame 24(`Host.cpp:337`)就对应`thenValue`这一行，然后一直在尝试拿`lock_`，然后整个程序就hang住了。由于`sendSnapshot`是个异步的函数，按理来说不会死锁啊。但是仔细思考一下，如果`thenValue`里的回调如果和`startSendSnapshot`在同一个线程执行不就死锁了吗？因为同一个线程对一个`std::mutex`上了两次锁...

## 对应的Future实现

### thenValue

我们可以看一看folly中Future的相关实现，首先是`thenValue`，其实里面比较简单，对于`thenValue`里的回调封装了一下，然后调用`thenImplementation`。

```c++
template <class T>
template <typename F>
Future<typename futures::detail::valueCallableResult<T, F>::value_type>
Future<T>::thenValue(F&& func) && {
  // 回调的wrapper
  auto lambdaFunc = [f = static_cast<F&&>(func)](
                        Executor::KeepAlive<>&&, folly::Try<T>&& t) mutable {
    return futures::detail::wrapInvoke(std::move(t), static_cast<F&&>(f));
  };
  using R = futures::detail::tryExecutorCallableResult<T, decltype(lambdaFunc)>;
  // 不能inline执行
  return this->thenImplementation(
      std::move(lambdaFunc), R{}, futures::detail::InlineContinuation::forbid);
}
```

需要注意的是里面`tryExecutorCallableResult`是个`type_traits`的类，folly的Future实现中有相当多模板匹配的方法，我们只列出这次`thenValue`匹配的对应代码：

```c++
template <
    typename T,
    typename F,
    typename = std::enable_if_t<is_invocable_v<F, Executor*, Try<T>&&>>>
struct tryExecutorCallableResult {
  typedef detail::argResult<true, F, Executor::KeepAlive<>&&, Try<T>&&> Arg;
  typedef isFutureOrSemiFuture<typename Arg::Result> ReturnsFuture;
  typedef typename ReturnsFuture::Inner value_type;
  typedef Future<value_type> Return;
};

// 这个类用来保存thenValue中回调的相关信息
// 比如上面的例子中的回调 有一个参数status 没有返回语句 Result类型是void
template <bool isTry_, typename F, typename... Args>
struct argResult {
  using Function = F;
  // 参数列表
  using ArgList = ArgType<Args...>;
  // 返回类型
  using Result = invoke_result_t<F, Args...>;
  // 参数个数
  using ArgsSize = index_constant<sizeof...(Args)>;
  static constexpr bool isTry() { return isTry_; }
};

// isFutureOrSemiFuture实际就是判断传入的类是不是Future或者SemiFuture
// 文章开头示例中的sendSnapshot.thenValue中的这个回调没有返回值类型，所以特化为isFutureOrSemiFuture<void>
// 对应匹配的应该是这一条 所以Inner和Return在这个例子中都是folly::Unit
template <typename T>
struct isFutureOrSemiFuture : std::false_type {
  using Inner = lift_unit_t<T>;
  using Return = Inner;
};
```

实际上`tryExecutorCallableResult`经过`template specification`如下

```c++
// T为sendSnapshot的返回的Future<T>中的T
template <
    typename T,
    typename F,
    typename = std::enable_if_t<is_invocable_v<F, Executor*, Try<T>&&>>>
struct tryExecutorCallableResult {
  typedef detail::argResult<true, F, Executor::KeepAlive<>&&, Try<T>&&> Arg;
  typedef isFutureOrSemiFuture<void> ReturnsFuture;
  typedef folly::Unit value_type;
  typedef Future<value_type> Return;
```

### thenImplementation

由于`ReturnsFuture`是`std::false_type`，所以匹配到下面这个`thenImplementation`，返回值类型是`R::Return`也就是`Future<T>`。

```c++
// Variant: returns a value
// e.g. f.then([](Try<T>&& t){ return t.value(); });
template <class T>
template <typename F, typename R>
typename std::enable_if<!R::ReturnsFuture::value, typename R::Return>::type
FutureBase<T>::thenImplementation(
    F&& func, R, futures::detail::InlineContinuation allowInline) {
  static_assert(R::Arg::ArgsSize::value == 2, "Then must take two arguments");
  typedef typename R::ReturnsFuture::Inner B;

  // step 1: 构造Future对应的Promise, B实际是folly::Unit类型 (也就是例子里面thenValue的返回类型 由于没有返回值 所以是folly::Unit)
  Promise<B> p;
  p.core_->initCopyInterruptHandlerFrom(this->getCore());

  // grab the Future now before we lose our handle on the Promise
  // step 2
  auto sf = p.getSemiFuture();
  // step 3: 设置Future在哪个executor上执行后面的回调
  sf.setExecutor(folly::Executor::KeepAlive<>{this->getExecutor()});
  // step 4
  auto f = Future<B>(sf.core_);
  sf.core_ = nullptr;

  // step 5: 同时还会设置Core的状态
  this->setCallback_(
      [state = futures::detail::makeCoreCallbackState(
           std::move(p), static_cast<F&&>(func))](
          Executor::KeepAlive<>&& ka, Try<T>&& t) mutable {
        // t是上一个回调执行的结果
        if (!R::Arg::isTry() && t.hasException()) {
          state.setException(std::move(ka), std::move(t.exception()));
        } else {
          // 如果上一个执行结果没有抛异常 就调用then后面的函数
          auto propagateKA = ka.copy();
          state.setTry(std::move(propagateKA), makeTryWith([&] {
                         return detail_msvc_15_7_workaround::invoke(
                             R{}, state, std::move(ka), std::move(t));
                       }));
        }
      },
      allowInline);
  return f;
}
```

这个函数主要分为以下几步：

1. 构造一个`Promise`
2. 获取`Promise`对应的`SemiFuture`
3. 设置`SemiFuture`的`Exectuor`
4. 由于要返回的是`Future`，所以将`SemiFuture`中的`Core`设置给`Future`(`Core`里面包含executor)
5. 要返回的`Future`中的`Core`已经构造完成，设置对应的`Callback`，设置这个`Future`里面Core的状态并返回

前两步代码如下

```c++
template <class T>
Promise<T>::Promise() : retrieved_(false), core_(Core::make()) {}

Core() : CoreBase(State::Start, 2) {}

template <class T>
SemiFuture<T> Promise<T>::getSemiFuture() {
  if (retrieved_) {
    throw_exception<FutureAlreadyRetrieved>();
  }
  retrieved_ = true;
  return SemiFuture<T>(&getCore());
}

explicit SemiFuture(Core* obj) : Base(obj) {}
```

第三步：`sf.setExecutor(folly::Executor::KeepAlive<>{this->getExecutor()});`。

这一步要做的是设置`then`所返回的`Future`的executor，这个executor和`thenValue`之前的`Future`(也就是`sendSnapshot`返回的`Future`)的executor是同一个。

```c++
Executor* CoreBase::getExecutor() const {
  if (!executor_.isKeepAlive()) {
    return nullptr;
  }
  // executor_是KeepAliveOrDeferred
  return executor_.getKeepAliveExecutor();
}

/// Call only from consumer thread, either before attaching a callback or
/// after the callback has already been invoked, but not concurrently with
/// anything which might trigger invocation of the callback.
void CoreBase::setExecutor(KeepAliveOrDeferred&& x) {
  DCHECK(
      state_ != State::OnlyCallback &&
      state_ != State::OnlyCallbackAllowInline);
  executor_ = std::move(x);
}

Executor* KeepAliveOrDeferred::getKeepAliveExecutor() const noexcept {
  switch (state_) {
    case State::Deferred:
      return nullptr;
    case State::KeepAlive:
      // 返回一个executor
      return keepAlive_.get();
  }
  assume_unreachable();
}
```

### setCallback

然后我们就来到了最重要的第五步，这一步会设置当前这个`Future`的状态和它后面要执行的`callback`。

```c++
template <class T>
template <class F>
void FutureBase<T>::setCallback_(
    F&& func, futures::detail::InlineContinuation allowInline) {
  throwIfContinued();
  getCore().setCallback(
      static_cast<F&&>(func), RequestContext::saveContext(), allowInline);
}

```c++
  template <class F>
  void setCallback(
      F&& func,
      std::shared_ptr<folly::RequestContext>&& context,
      futures::detail::InlineContinuation allowInline) {
    Callback callback = [func = static_cast<F&&>(func)](
                            CoreBase& coreBase,
                            Executor::KeepAlive<>&& ka,
                            exception_wrapper* ew) mutable {
      auto& core = static_cast<Core&>(coreBase);
      if (ew != nullptr) {
        core.result_ = Try<T>{std::move(*ew)};
      }
      func(std::move(ka), std::move(core.result_));
    };

    setCallback_(std::move(callback), std::move(context), allowInline);
  }
```

核心代码是在`CoreBase::setCallback_`里，这里涉及到`Core`这个类的有限状态机转换，后面可能会单独整理出来。简单来说一个Future起始状态转换成完成状态，需要`producer thread`调用`setResult`，还需要`consumer thread`调用`setCallback`，两者都完成后，Future就已经拿到结果，并可以继续链式执行下去。

需要说明的是`producer`和`consumer`如果是在两个线程，谁先执行是不一定的，所以可以看到下面代码会根据`cas`的结果做不同处理。

```c++
void CoreBase::setCallback_(
    Callback&& callback,
    std::shared_ptr<folly::RequestContext>&& context,
    futures::detail::InlineContinuation allowInline) {
  DCHECK(!hasCallback());

  // 构造Callback和Context 会在其它函数反复使用
  // using Callback = folly::Function<void(CoreBase&, Executor::KeepAlive<>&&, exception_wrapper* ew)>;
  // using Context = std::shared_ptr<RequestContext>;
  ::new (&callback_) Callback(std::move(callback));
  ::new (&context_) Context(std::move(context));

  auto state = state_.load(std::memory_order_acquire);
  State nextState = allowInline == futures::detail::InlineContinuation::permit
      ? State::OnlyCallbackAllowInline
      : State::OnlyCallback;

  if (state == State::Start) {
    // 将状态cas为OnlyCallbackAllowInline或者OnlyCallback
    // cas成功就直接返回 consume thread会继续推动状态机
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

  // cas失败 也就是在调用state_.load(std::memory_order_acquire)和cas之间状态发生了改变
  // 根据最新状态 判断FSM进入到哪个状态
  if (state == State::OnlyResult) {
    state_.store(State::Done, std::memory_order_relaxed);
    // producer thread已经生成了结果 直接调用callback
    doCallback(Executor::KeepAlive<>{}, state);
    return;
  }

  if (state == State::Proxy) {
    return proxyCallback(state);
  }

  terminate_with<std::logic_error>("setCallback unexpected state");
}
```

根据死锁时的pstack可以看到，我们进入了`doCallback`这个函数。这可能发生在两种情况下：

1. `consumer`获取状态时已经是`OnlyResult`
2. `consumer`第一次拿state时是`Start`状态(对应`auto state = state_.load(std::memory_order_acquire);`这行)，而当在想把state通过`cas`操作改为`OnlyCallback`时失败了，然后会发现`producer`已经把state改为了`OnlyResult`，所以`consumer`可以直接调用callback，所以进入到`doCallback`。

这也就解释了为什么测试时候不是稳定复现的原因，我们可以再详细列下几种情况，Future起始状态都是`Start`

* case 1: 先`setCallback` 再`setResult`

```
       setCallback                 setResult
Start -------------> onlyCallback -----------> Done
```

* case 2: 先`setResult` 再`setCallback`

```
       setResult               setCallback
Start -----------> onlyResult -------------> Done
```

* case 3: `setResult`和`setCallback`有并发

```
上面是setResult
                    set state = onlyResult
Start -----------------------------------------------------------------------------------------------> Done
        load state                          cas failed    load state again      set state = Done
        state == Start                                    state == OnlyResult   doCallback

下面是setCallback
```

case2和case3都会出现死锁。

修掉bug的方法也很简单，要么使用`SemiFuture`，要么由于`lock_`只是用来保护一个bool变量，可以换成`atomic_bool`去掉锁就好了。
