---
layout: single
title: Deciphering C++ Coroutines, part 4
date: 2025-11-19 00:00:00 +0800
categories: 学习
tags: C++ folly
---

本来想直接介绍`folly::coro::Task`的，但鉴于上一篇展示了太多的”术”，这一片会从宏观视角，理解一个通过协程实现的异步任务，到底需要实现什么东西，以及为什么需要这么实现，所谓“道”。在此基础上，可能会穿插一些`folly::coro::Task`的内容。这篇很多内容都是总结于这个[演讲](https://www.youtube.com/watch?v=qfKFfQSxvA8)，也可以看这个更详尽的[版本](https://www.youtube.com/watch?v=lKUVuaUbRDk)。

### Mental Model

假设要实现如下一个任务，即`main`依次调用`spawn_task`生成一个任务，其中又依次调用了`outer_function`，`middle_function`和最里层的`inner_function`。

![figure]({{'/archive/coroutine-10.png' | prepend: site.baseurl}})

```cpp
void spawn_task() {
    // ...
    Result r = outer_function();
}

Result outer_function() {
    PartialResult r = middle_function();
    return Result::from_partial_result(r);
}

PartialResult middle_function() {
    auto r = inner_function();
    return PartialResult::from_io_result(r);
}

IoResult inner_function() {
    auto data = blocking_io(...); // this could take some time
    return IoResult::from_io_data(data);
}
```

最里层的`inner_function`会执行一些异步IO操作，当这个异步IO完成时，我们希望能唤醒整个任务中还没执行完成的部分继续执行。

如果想支持异步执行这个任务，那么整个任务中的每一层可能都要进行相应接口改造，使得`outer_function`/`middle_function`/`inner_function`都变成一个异步任务。下面是一些常用的实现方式：

- 线程(thread)：通过`std::async`的方式启动新线程，通过`std::future`获取结果。缺点是线程context switch比较繁重，如果有多个任务之间需要协同，需要额外同步机制。
- 有栈协程(fiber)：当需要执行异步操作时，通过fiber主动让出线程并挂起，使其他fiber能够运行。当异步操作完成时，通过调度器能够恢复被挂起的fiber。优势是fiber之间的切换比线程切换更轻量，但需要一些额外栈空间。
- 无栈协程(coroutine)：通过实现协程的相关接口，自定义什么时候挂起，以及什么时候由谁恢复。但注意到，每次只能将一个协程挂起。如果我们将`inner_function`挂起，那除了自身的挂起点信息之外，同时还需要保存其调用方`middle_function`/`outer_function`的挂起点信息。

所以这一篇我们核心要解释的就是，如果通过coroutine来实现这样一个异步任务，需要提供什么样的能力。

## Async Task

我们将上面的同步调用改造为一个协程，每个协程都返回一个`Async`对象，代表是一个异步任务(Async Task)。

> `folly::coro::Task`本质上就是一个我们所实现的这个`Async`异步任务，只不过为了更好介绍异步任务的核心原理，省略了一些不是那么重要的细节
>

```cpp
Async<IoResult> inner_function() {
    // ...
}

Async<PartialResult> middle_function() {
    auto r = co_await inner_function();
    co_return PartialResult::from_io_result(r);
}
```

上面代码可以等价展开为如下的形式：

```cpp
Async<IoResult> inner_function() {
    // ...
}

Async<PartialResult> middle_function() {
    Async<IoResult> awaitable = inner_function();
    IoResult r = co_await awaitable;
    co_return PartialResult::from_io_result(r);
}
```

注意到，`Async`作为一个异步任务，它本身是一个协程的返回值。它又可以在其他协程中被`co_await`，比如上面的`middle_function`中`co_await inner_function()`。

因此`Async`既是`ReturnType`，又是`Awaitable`。即`Async`要能满足以下能力：

- `ReturnType`：作为协程函数的返回值
- `Awaitable`：可以被 `co_await`

> `folly::coro::Task`本身是个`ReturnType`，又提供了嵌套类`folly::coro::Task::Awaiter`作为`Awaitable`，本质上一样。
>

### 1. How to resume inner coroutine

首先我们需要解决的第一个问题就是，我们如何恢复异步任务中剩余没有执行完成的部分继续执行。我仍然以上面的例子为例，描述下可能的执行流程（如果看不太懂，需要看下前几篇博客）：

当`middle_function`开始执行时，它会`co_await inner_function()`。如果`Awaitable`对应的`await_ready`决定要挂起并调用`await_suspend`，此时`middle_function`的调用方是能拿到一个`Async`对象的。绝大多数实现中，`Async`对象中都会保存对应的`coroutine_handle`。

此时，一种简单的想法就是，直接从外向内恢复，即调用这个`coroutine_handle`的`resume`方法继续执行。这样做有几个问题，注意到此时`inner_function`还没有执行完成：

- 协程一旦被恢复，就无法再次挂起（除非遇到新的`co_await`）
- 从外部无法知道需要恢复内层函数多少次才能完成
- 外层协程恢复时，内层结果可能还未就绪，无法执行`co_return`

也就是说，当我们恢复outer coroutine，即这里的`middle_function`时，我们无法确保`inner_function`已经执行完成，也就无法获取到`inner_function`的返回值`r`，更没有办法执行`co_return`。并且在恢复`middle_function`继续执行后，已经无法控制协程是否再次挂起了（取决于是否内部还有其他`co_await`）。

正确的`co_await`语义是：当被`co_await`的协程完成，能获取到其结果时，唤醒调用方exactly once。

所以，当我们挂起一个基于协程的异步任务时，所需要的上下文，远远多余单个函数。我们不仅需要整个异步任务之间的调用栈，并且需要要以某种形式将多个互相调用的协程串联起来，使得他们能按预期执行顺序执行。在了解基于协程的异步任务解决方案之前，我们先不妨看下普通函数调用是如何继续执行任务中还未完成的部分。

函数调用时，汇编层面都会有prologue，由被调用方负责保存调用方的基地址%rbp：

```nasm
pushq %rbp
movq %rsp, %rbp
```

callee返回时，汇编层面有epilogue，由被调用方恢复调用方的基地址%rbp：

```nasm
leave
ret
```

如下图所示，当从最外层函数调用到最里层时，基地址的指针则是由内指向外，形成了一个单链表，即我们一般所说的调用栈。

![figure]({{'/archive/coroutine-11.png' | prepend: site.baseurl}})

对于普通函数而言，stack frame能满足如下要求：

- 创建最外层函数
- 调用内层函数
- 从内层函数恢复外层函数继续执行
- 传递返回结果

但对于基于协程的异步任务而言，需要额外支持几个功能：

- [ ]  创建最外层协程
- [ ]  调用内层协程，并且需要在co_await callee时建立callee → caller的关系（后面会解释）
- [ ]  挂起最内层协程
- [ ]  恢复最内层协程
- [ ]  从内层协程恢复外层协程
- [ ]  传递最终结果

我们接下来看看异步任务如何满足几个需求。

我们可以分析下这个异步任务需要提供什么样的接口，以及如何通过这些接口完成上面所述的功能。注意到`middle_function`本身是一个协程，它的`ReturnType`是一个`Async`对象。而在`middle_function`中它又`co_await`另一个`Async`对象，即`Async`还是一个`Awaitable`对象（任何支持`co_await`的对象都是`Awaitable`）。因此，Async需要同时扮演`ReturnType`和`Awaitable`。

```cpp
Async<IoResult> inner_function() {
    // ...
}

Async<PartialResult> middle_function() {
    Async<IoResult> awaitable = inner_function();
    IoResult r = co_await awaitable;
    co_return PartialResult::from_io_result(r);
}
```

`Async`作为`ReturnType`时，需要维护对应协程的句柄，比如上面例子中`awaitable`对象作为一个`Async<IoResult>`需要保存`inner_function`的`coroutine_handle`。因此`Async`的构造函数如下所示：

```cpp
template<typename T>
struct Async {
    struct promise_type {
        Async<T> get_return_object() {
            auto h = std::coroutine_handle<promise_type>::from_promise(*this);
            return Async<T>{h};
        }
    };

    // ReturnType part of Async
    std::coroutine_handle<promise_type> self;
    Async(std::coroutine_handle<promise_type> h) : self(h) {}
};
```

`Async`作为`Awaitable`时，它需要能建立协程之间的调用链。这里的关键就是`Awaitable`中的`await_suspend`接口。它的参数是调用方的`coroutine_handle`，即它对应`co_await`了内层协程（下文都称为`callee`）的协程（下文都称为`caller`）。并且在`callee`还没有准备好时，`caller`希望被挂起。

在一个异步任务中，`await_suspend`接口的核心功能就是建立从`callee`到`caller`的联系。准确说就是告诉`callee`：“`caller`是你的调用者，当你的结果就绪时，应该恢复的是这个协程“。

因此在大多数异步任务的实现中，在`Awaitable`的`await_suspend`中，需要将`caller`的`coroutine_handle`保存到`callee`的`promise`中。对称转移在此基础上会返回`callee`的`coroutine_handle`。

```cpp
template<typename T>
struct Async {
    struct promise_type {
        /*...*/
        std::coroutine_handle<> my_caller;
    };

    // ReturnType part of Async
    std::coroutine_handle<promise_type> self;

    // Awaitable part of Async
    bool await_ready() {
        return false;
    }

    T await_resume() {/*...*/}

    auto await_suspend(std::coroutine_handle<> handle) {
        self.promise().my_caller = handle;
        // Asymmetric transfer will return void
        // return;
        // Symmetric transfer will return callee's handle
        return self;
    }
};
```

> 这部分`folly::coro::Task`对应的代码
>
>
> ```cpp
>     template <typename Promise>
>     FOLLY_NOINLINE auto await_suspend(
>         coroutine_handle<Promise> continuation) noexcept {
>       DCHECK(coro_);
>       auto& promise = coro_.promise();
>
>       promise.continuation_ = continuation;
>
>       auto& calleeFrame = promise.getAsyncFrame();
>       calleeFrame.setReturnAddress();
>
>       if constexpr (detail::promiseHasAsyncFrame_v<Promise>) {
>         auto& callerFrame = continuation.promise().getAsyncFrame();
>         folly::pushAsyncStackFrameCallerCallee(callerFrame, calleeFrame);
>         return coro_;
>       } else {
>         folly::resumeCoroutineWithNewAsyncStackRoot(coro_);
>         return;
>       }
>     }
> ```
>

到这我们成功建立了`callee`到`caller`的联系，如下图所示，从左到右依次为`inner_function`/`middle_function`/`outer_function`，并且将`caller`的`coroutine_handle`保存到了`callee`的`promise`中。

![figure]({{'/archive/coroutine-12.png' | prepend: site.baseurl}})

接下来，我们看下如何支持从内层协程恢复外层协程。其实这个功能在上一篇我们已经非常详细的解释过了，即在`co_await promise.final_suspend()`阶段中，通过`Awaitable::await_suspend`对称转移完成。

1. 首先是`callee`（这里的`inner_function`）在执行完成时，会通过`promise.return_value`或者`promise.return_void`接口传递返回值。之后进入到最后一次挂起点，即`co_await promise.final_suspend()`，此时会返回一个`Awaitable`对象（即下图中的`ResumeCaller`）。

    ![figure]({{'/archive/coroutine-13.png' | prepend: site.baseurl}})

2. 通过`Awaitable::await_suspend`对称转移到`caller`（这里的`middle_function`）。

    ```cpp
    struct promise_type{
        /*...*/
        std::coroutine_handle<> my_caller;
        auto final_suspend() noexcept {
            return ResumeCaller{};
        }
    };

    struct ResumeCaller {
        bool await_ready() { return false; }
        void await_resume() { /* will never be called! */ }
        coroutine_handle<> await_suspend(coroutine_handle<promise_type> h) {
            // Symmetric Transfer!
            return h.promise().my_caller;
        }
    };
    ```

    ![figure]({{'/archive/coroutine-14.png' | prepend: site.baseurl}})


1. 之后`middle_function`通过`await_resume`获取到的返回值，并继续执行剩余逻辑，此时`inner_function`处于等待销毁的状态。

    ![figure]({{'/archive/coroutine-15.png' | prepend: site.baseurl}})

2. 当`middle_function`执行完成时，会将`inner_function`对应协程销毁

    ![figure]({{'/archive/coroutine-16.png' | prepend: site.baseurl}})


到这我们已经完成了协程所需的两个需求。

- [ ]  创建最外层协程
- [x]  调用内层协程，并且需要在调用时建立callee → caller的关系
- [ ]  挂起最内层协程
- [ ]  恢复最内层协程
- [x]  从内层协程恢复外层协程
- [ ]  传递最终结果

### 2. How to perform async io and resume innermost coroutine

接下来要解决的问题是如何恢复最内层执行IO的协程。由于具体的IO是由操作系统完成的，必须要有某种机制，使得在操作系统通知用户态的进程IO完成之后，能够恢复协程继续执行。

简单来说，在操作系统层面，异步IO一般会通过`libaio`或者是`io_uring`完成，而基础库一般会在此基础上封装，提供易于使用的异步IO接口，比如`boost::asio`和`folly::SimpleAsyncIO`。接下来我们通过梳理`folly::SimpleAsyncIO`的主干代码，理解如何在异步IO完成后恢复最内层的IO协程。

1. 用户代码在协程中执行IO操作，`SimpleAsyncIO`默认使用`libaio`来完成异步IO，`coro::Baton`用来挂起和恢复协程。

    ```cpp
    Async<IoResult> inner_function(int fd, void* buf, size_t size, off_t start) {
        SimpleAsyncIO aio;
        int result = co_await aio.co_pread(fd, buf, size, start);
        co_return IoResult::from(result);
    }

    folly::coro::Task<int> SimpleAsyncIO::co_pread(int fd, void* buf, size_t size, off_t start) {
        folly::coro::Baton done;
        int result;
        pread(fd, buf, size, start, [&done, &result](int rc) {
            result = rc;
            done.post();
        });
        co_await done;
        co_return result;
    }
    ```

2. 本质上就是注册了一个IO完成时的回调，然后像操作系统提交异步IO请求。即通过`setNotificationCallback`设置当IO完成时，向一个Executor中调度用户的回调，也就是上面代码中的`done.post()`。

    ```cpp
    void SimpleAsyncIO::submitOp(Function<void(AsyncBaseOp*)> preparer,
                                 SimpleAsyncIOCompletor completor) {
        std::unique_ptr<AsyncBaseOp> opHolder = getOp();
        if (!opHolder) {
            completor(-EBUSY);
            return;
        }

        // Grab a raw pointer to the op before we create the completion lambda,
        // since we move the unique_ptr into the lambda and can no longer access
        // it.
        AsyncBaseOp* op = opHolder.get();

        preparer(op);

        op->setNotificationCallback([this,
                                     completor{std::move(completor)},
                                     opHolder{std::move(opHolder)}](AsyncBaseOp* op_) mutable {
            CHECK(op_ == opHolder.get());
            int rc = op_->result();

            // 当IO完成 在线程池中调度用户回调
            completionExecutor_->add(
                    [rc, completor{std::move(completor)}]() mutable { completor(rc); });

            putOp(std::move(opHolder));
        });
        asyncIO_->submit(op); // 提交到内核（libaio/io_uring）
    }
    ```

    我们不关心具体异步IO是怎么完成的，重点关注如何从系统层面获取到IO完成事件，从而执行这个回调，从而恢复协程的。

3. `SimpleAsyncIO`在构造函数中会注册事件监听，即当IO完成时，内核会让`pollFd()`这个fd变成可读状态，调用`libeventCallback`这个回调。

    ```cpp
    SimpleAsyncIO::SimpleAsyncIO(Config cfg)
            : maxRequests_(cfg.maxRequests_),
              completionExecutor_(cfg.completionExecutor_),
              terminating_(false) {
        // ...
        if (cfg.evb_) {
            initHandler(cfg.evb_, NetworkSocket::fromFd(asyncIO_->pollFd()));
        } else {
            evb_ = std::make_unique<ScopedEventBaseThread>();
            initHandler(evb_->getEventBase(), NetworkSocket::fromFd(asyncIO_->pollFd()));
        }
        registerHandler(EventHandler::READ | EventHandler::PERSIST);
    }

    void EventHandler::initHandler(EventBase* eventBase, NetworkSocket fd) {
        ensureNotRegistered(__func__);
        event_.eb_event_set(fd.data, 0, &EventHandler::libeventCallback, this);
        setEventBase(eventBase);
    }
    ```

4. `SimpleAsyncIO`中的`EventBase`会通过一个while循环，不断检查是否有事件完成（参见`EventBase::loop()`），最终调度用户态回调。整个调用链如下：

    ```cpp
    EventBase::loop() // 检测到 pollFd 可读
    └─ libeventCallback // libevent调用这个回调
       └─ SimpleAsyncIO::handlerReady()
          └─ AsyncBase::pollCompleted()
             └─ AsyncBase:::doWait()
                └─ AsyncBase::complete(op, result)
                   └─ AsyncBaseOp::complete()
                      └─ cb_(this)  // 执行在构造时通过notificationCallback设置的回调
                         └─ completionExecutor_->add(...)  // 调度用户态回调 即Baton.post()
    ```

    下面是比较关键的函数：

    ```cpp
    void EventHandler::libeventCallback(libevent_fd_t fd, short events, void* arg) {
      auto handler = reinterpret_cast<EventHandler*>(arg);
      assert(fd == handler->event_.eb_ev_fd());
      (void)fd; // prevent unused variable warnings

      auto observer = handler->eventBase_->getExecutionObserver();
      if (observer) {
        observer->starting(reinterpret_cast<uintptr_t>(handler));
      }

      // this can't possibly fire if handler->eventBase_ is nullptr
      handler->eventBase_->bumpHandlingTime();

      handler->handlerReady(uint16_t(events));

      if (observer) {
        observer->stopped(reinterpret_cast<uintptr_t>(handler));
      }
    }

    void SimpleAsyncIO::handlerReady(uint16_t events) noexcept {
        if (events & EventHandler::READ) {
            // All the work (including putting op back on free list) happens in the
            // notificationCallback, so we can simply drop the ops returned from
            // pollCompleted. But we must still call it or ops never complete.
            while (asyncIO_->pollCompleted().size()) {
                ;
            }
        }
    }
    ```


所以到这我们已经知道如何在协程中完成异步IO并挂起，并且如何在IO完成时唤醒协程继续执行。

- [ ]  创建最外层协程
- [x]  调用内层协程，并且需要在co_await callee时建立callee → caller的关系
- [x]  挂起最内层协程
- [x]  恢复最内层协程
- [x]  从内层协程恢复外层协程
- [ ]  传递最终结果

### 3. Symmetric transfer reviewed

在解决剩余的问题前，我们再review一个问题：当一个异步任务中又执行了其他的异步任务，如何使他们都在相同的上下文执行，比如让`caller`和`callee`都调度到同一组线程池执行。或者换成更通用的场景，如何让`caller`传递任意信息到`callee`。

答案跟之前建立`callee -> caller`的联系一样，在`caller`执行`co_await callee()`时，通过异步任务中的`Awaitable`的`await_suspend`，可以从`caller`的`promise`对象传递任意信息到`callee`。

```cpp
auto await_suspend(std::coroutine_handle<> handle) {
    self.promise().my_caller = handle;
    self.promise().some_thing_else = h.promise().some_thing_else;
    return self;
}
```

通常来说，通过协程实现的异步任务都会通过对称转移，将控制权交给`callee`，这是为什么？这里的本质在于当`caller`执行`co_await callee()`时，都会先将`callee`挂起（`await_ready`返回`false`），注意此时`callee`可能还缺少一些上下文信息。紧接着，在`await_suspend`中，`caller`将上下文给`callee`。

我们分析这一时刻`caller`和`callee`所期望的行为：

- `caller`肯定是希望要被挂起，因为`callee`还没有执行完成
- `callee`此时已经处于挂起状态，但它想要开始执行。

因此在对称转移中，在上下文传递完成之后，就应该返回`callee`的`coroutine_handle`，从而恢复`callee`执行（非对称转移则是手动`resume`）。从另一种角度上说，正因为存在这种上下文转递的需求，同时我们又希望协程的控制流和普通函数调用几乎一样，使得C++中提供了对称转移这样的协程底层机制。

### 4. How to spawn a async task

最后就是如何启动一个异步任务，并获取其最终结果了，这块我们直接用`folly::coro::Task`为例。`folly::coro::Task`是lazy启动的（对应的`promise_type`中`initial_suspend`为`suspend_always`）。

启动`Task`的形式有两种：

第一种是从协程中启动，即在另一个`Task`中`co_await`另一个Task。

```cpp
folly::coro::Task<int> callee() {
  co_return 42;
}

folly::coro::Task<int> caller() {
  int result = co_await callee();
  co_return result;
}
```

主要流程如下：

1. `caller`执行到`co_await callee()`
2. 编译器调用`caller.promise.await_transform(callee())`，获取到对应的`Awaiter`。其中注意在`co_viaIfAsync`中，将子任务的`Executor`设置为了父任务的`Executor`。

    ```cpp
    template <typename Awaitable>
    auto await_transform(Awaitable&& awaitable) {
        bypassExceptionThrowing_ = bypassExceptionThrowing_ == BypassExceptionThrowing::REQUESTED
                                           ? BypassExceptionThrowing::ACTIVE
                                           : BypassExceptionThrowing::INACTIVE;

        return folly::coro::co_withAsyncStack(folly::coro::co_viaIfAsync(
                executor_.get_alias(),
                folly::coro::co_withCancellation(cancelToken_,
                                                 static_cast<Awaitable&&>(awaitable))));
    }

    friend auto co_viaIfAsync(Executor::KeepAlive<> executor, Task<T>&& t) noexcept {
        DCHECK(t.coro_);
        // Child task inherits the awaiting task's executor
        t.setExecutor(std::move(executor));
        return Awaiter{std::exchange(t.coro_, {})};
    }

    void setExecutor(folly::Executor::KeepAlive<>&& e) noexcept {
        DCHECK(coro_);
        DCHECK(e);
        coro_.promise().executor_ = std::move(e);
    }

    ```

3. 编译器调用`Awaiter::await_suspend`，其内部就是标准的对称转移实现：
    1. 将`caller`的`coroutine_handle`保存到`callee`的`promise`中
    2. 返回`callee`的`coroutine_handle`，恢复callee执行
    3. 值得注意的是它还设置了一个`AsyncStackFrame`用于保存协程之间的调用关系，这块内容下一篇会再展开介绍

    ```cpp
    template <typename Promise>
    FOLLY_NOINLINE auto await_suspend(coroutine_handle<Promise> continuation) noexcept {
        DCHECK(coro_);
        auto& promise = coro_.promise();

        // 1. 保存父协程的coroutine_handle
        promise.continuation_ = continuation;

        // 2. 设置AsyncStackFrame
        auto& calleeFrame = promise.getAsyncFrame();
        calleeFrame.setReturnAddress();

        if constexpr (detail::promiseHasAsyncFrame_v<Promise>) {
            auto& callerFrame = continuation.promise().getAsyncFrame();
            // 如果父协程也有AsyncStackFrame 那么就子协程加入到调用关系中
            folly::pushAsyncStackFrameCallerCallee(callerFrame, calleeFrame);
            return coro_;
        } else {
            folly::resumeCoroutineWithNewAsyncStackRoot(coro_);
            return;
        }
    }
    ```

4. 之后`callee`就开始在`caller`的`Executor`上开始执行

第二种则是，从非协程启动：

```cpp
folly::coro::Task<int> myTask() {
  co_return 42;
}

void normalFunction() {
  auto future = myTask()
      .scheduleOn(executor）
      .start();
  int result = std::move(future).get();

  // or
  // int result = folly::coro::blockingWait(myTask().scheduleOn(executor));
}
```

主要步骤是：

1. 通过`Task::scheduleOn(executor)`获取到一个`TaskWithExecutor`对象，Executor会保存在协程的promise中。
2. 调用`TaskWithExecutor::start()`启动一个辅助协程`startImpl`

    ```cpp
    FOLLY_NOINLINE SemiFuture<lift_unit_t<StorageType>> start() && {
        folly::Promise<lift_unit_t<StorageType>> p;

        auto sf = p.getSemiFuture();

        std::move(*this).startImpl(
                [promise = std::move(p)](Try<StorageType>&& result) mutable {
                    promise.setTry(std::move(result));
                },
                folly::CancellationToken{},
                FOLLY_ASYNC_STACK_RETURN_ADDRESS());

        return sf;
    }

    template <typename F>
    detail::InlineTaskDetached startImpl(TaskWithExecutor task, F cb) {
        try {
            cb(co_await folly::coro::co_awaitTry(std::move(task)));
        } catch (...) {
            cb(Try<StorageType>(exception_wrapper(std::current_exception())));
        }
    }
    ```

    `startImpl`这个协程实际就是`co_await`了这个`TaskWithExecutor`对象。核心步骤在`TaskWithExecutor::Awaiter::await_suspend`中，即通过`promise.executor_->add(...)`调度协程在给定的`Executor`上启动。

    ```cpp
    template <typename Promise>
    void await_suspend(coroutine_handle<Promise> continuation) noexcept {
        DCHECK(coro_);
        auto& promise = coro_.promise();
        DCHECK(promise.executor_);

        auto& calleeFrame = promise.getAsyncFrame();
        calleeFrame.setReturnAddress();

        promise.continuation_ = continuation;

        promise.executor_->add([coro = coro_, ctx = RequestContext::saveContext()]() mutable {
            RequestContextScopeGuard contextScope{std::move(ctx)};
            folly::resumeCoroutineWithNewAsyncStackRoot(coro);
        });
    }
    ```


协程启动的流程就这么多，而当协程执行完毕，通过`Task::Awaiter::await_resume`或者`TaskWithExecutor::Awaiter::await_resume`，即可获取到其结果。

```cpp
T await_resume() {
    DCHECK(coro_);
    // Eagerly destroy the coroutine-frame once we have retrieved the result.
    SCOPE_EXIT {
        std::exchange(coro_, {}).destroy();
    };
    return std::move(coro_.promise().result()).value();
}
```

- [x]  创建最外层协程
- [x]  调用内层协程，并且需要在co_await callee时建立callee → caller的关系
- [x]  挂起最内层协程
- [x]  恢复最内层协程
- [x]  从内层协程恢复外层协程
- [x]  传递最终结果



## At last

之前的三篇文章，更多是从C++标准提供了什么样的协程底层机制，而这一篇则是运用这些机制，分析一个基于协程的异步任务需要什么接口、该怎么实现以及为什么需要这样实现。但协程的底层机制实在是有些复杂，所以也才会先有前三篇介绍各种细节，最后在这才能加以总结了。

## Refernce

[Manage Asynchronous Control Flow With C++ Coroutines - Andreas Weis](https://www.youtube.com/watch?v=lKUVuaUbRDk)