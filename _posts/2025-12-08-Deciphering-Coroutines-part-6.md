---
layout: single
title: Deciphering C++ Coroutines, part 6
date: 2025-11-19 00:00:00 +0800
categories: 学习
tags: C++ folly
---

坑越挖越深，这一篇看下Coroutine和C++26引入的Sender。

## P2300

[P2300](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2300r10.html)目前已经已经被确认加入到C++26了，这个提案可以说指明了未来所有异步代码的编写方式。我们简单介绍下这个提案，它主要引入了四个概念：

- `Scheduler`：代表一个执行上下文，也就是在哪里执行一个函数或者任务，可以是线程池、CPU等等。
- `Sender`：代表一个异步产生结果的对象，可以简单理解为一个lazy的等待执行的函数
- `Receiver`：代表一个用于接受异步结果的对象
- `OperatorState`：用于启动异步任务和生命周期管理

它们的相关接口如下：

![figure]({{'/archive/coroutine-18.png' | prepend: site.baseurl}})

这几个概念的关系如下：

- `Scheduler`有一个`schedule`方法，返回一个`Sender`。注意返回值是一个空的`Sender`，只用来表示后续任务在哪里执行。
- `Sender`可以在`starts_on`或者`then`等接口指定实际要执行的任务。但注意，调用`starts_on`或者`then`时并不会开始执行这些任务，即前面提到的`Sender`本质上是一个lazy执行的任务。lazy的优势在于，在真正开始执行这个任务之前，我们可以把多个`Sender`组织在一起，形成一个DAG的形式。
- `Receiver`的相应接口用来保存`Sender`的执行结果：
    - `set_value`：正常执行，传递结果
    - `set_error`：执行错误，传递异常
    - `set_done`：任务被取消
- 有了`Sender`和`Receiver`之后，我们需要通过`connect`将二者连在一起，即告诉给定`Sender`在执行完成或者发生异常后，将相应结果告知给定的`Receiver`，`connect`返回一个`OperatorState`。
- `OperatorState`作用是负责启动异步任务，并保存异步操作的相关状态。

![figure]({{'/archive/coroutine-19.png' | prepend: site.baseurl}})

### Why we use senders?

`stdexec`已经证明了P2300的可行性，可以在[这里](https://godbolt.org/z/3cseorf7M)感受一下中通过`Sender`来完成异步任务的能力：

- 异步任务能够指定在哪个`Scheduler`上执行
- 多个异步任务能够链式执行，甚至组织成一个DAG的形式

```cpp
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>

int main()
{
    // Declare a pool of 3 worker threads:
    exec::static_thread_pool pool(3);

    // Get a handle to the thread pool:
    auto sched = pool.get_scheduler();

    // Describe some work:
    // Creates 3 sender pipelines that are executed concurrently by passing to `when_all`
    // Each sender is scheduled on `sched` using `on` and starts with `just(n)` that creates a
    // Sender that just forwards `n` to the next sender.
    // After `just(n)`, we chain `then(fun)` which invokes `fun` using the value provided from `just()`
    // Note: No work actually happens here. Everything is lazy and `work` is just an object that statically
    // represents the work to later be executed
    auto fun = [](int i) { return i*i; };
    auto work = stdexec::when_all(
        stdexec::starts_on(sched, stdexec::just(0) | stdexec::then(fun)),
        stdexec::starts_on(sched, stdexec::just(1) | stdexec::then(fun)),
        stdexec::starts_on(sched, stdexec::just(2) | stdexec::then(fun))
    );

    // Launch the work and wait for the result
    auto [i, j, k] = stdexec::sync_wait(std::move(work)).value();

    // Print the results:
    std::printf("%d %d %d\n", i, j, k);
}
```

但事实上，我第一次看这个草案的相关介绍时，我的第一反应是为什么需要这东西（据说该草案据说在标准委员会投票时也有非常大的争议），而且从好几个方面我都产生了怀疑：

1. 标准库已经有promise/future了，更别提folly的Promise/Future也能组织成DAG的形式
2. 标准库已经有协程了，不是说协程是C++20之后编写异步代码的方式吗
3. 为什么P2300引入这么多新概念，有必要吗

关于这些问题，我推荐去看下这篇[文章](https://ericniebler.com/2024/02/04/what-are-senders-good-for-anyway/)。这里浅谈一下我的理解：

1. 按照P2300这个草案，通过各种算法（比如`then`/`when_all`等），把各个Sender串联起来，构建出非常复杂的任务，这一点的确和folly的Promise/Future一样。但一个核心区别在于Sender是一个lazy任务，它把任务调度和具体任务执行分离开来，更加灵活一些。
2. P2300的设计理念遵循[Structured Concurrency](https://www.youtube.com/watch?v=1Wy5sq3s2rg)，并且提供了无缝衔接协程的能力（我们后面会用实际代码来解释这一点），保证了父协程一定晚于子协程结束，避免了shared state带来的资源管理问题。
3. P2300的出现，使得所有异步任务有了相同抽象，不同库的异步任务可能具体实现方式不同（比如回调、future等），而P2300使得不同异步任务都能统一为`Sender`/`Receiver`的实现，从而具备将不同库的异步任务串联起来的能力。

不过这块内容牵涉面太广，就不再展开了，感兴趣的可以看看引用的这些文章。我们的重点还是研究协程和`Sender`。

## Use coroutine as sender

事实上，协程和Sender的关系并不是互相取代，而是Sender从几方面加强了协程：

- 协程可以作为Sender使用，也就能使用Sender提供的各种算法：
    - then/let_value/let_error/let_done
    - when_all
    - repeat_effect/retry
    - schedule_after
    - …
- 协程已经比其他异步代码形式朝Structured Concurrency的方向已经迈了一大步，但这些约束都不是强制性的，比如协程可以创建任务但不等待完成就返回，生命周期管理需要程序员手动保证。但P2300则全方位满足了Structured Concurrency的需求：
    - 生命周期嵌套：子任务的生命周期严格在父任务内
    - 错误传播：子任务的异常能正确传播到父任务
    - 资源清理：所有资源在正确时机清理
    - 取消支持：父任务取消能传播到所有子任务
    
    有关这一块内容，有机会再单独展开介绍其具体原理。
    
- 性能上，Eric Niebler和Lewis Baker的博客都提到了把协程作为Sender时，coroutine frame就不需要动态分配了，这块没有深究。

下面会通过一个简单例子，看看如何把协程当成一个`Sender`使用。代码中的`SimpleTask`就是之前介绍对称转移时的[协程](https://godbolt.org/z/n896xMrdG)，没有做任何修改。

```cpp
#include <coroutine>
#include <iostream>
#include <utility>

#include <exec/static_thread_pool.hpp>
#include <stdexec/execution.hpp>

template <typename T>
struct SimpleTask {
    struct promise_type {
        SimpleTask get_return_object() {
            return SimpleTask{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() {
            return {};
        }

        struct FinalAwaiter {
            bool await_ready() noexcept {
                return false;
            }

            std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<promise_type> h) noexcept {
                auto continuation = h.promise().continuation_;
                if (continuation) {
                    return continuation;
                }
                return std::noop_coroutine();
            }

            void await_resume() noexcept {}
        };

        FinalAwaiter final_suspend() noexcept {
            return {};
        }

        void return_value(T v) {
            value_ = v;
        }
        void unhandled_exception() {
            exception_ = std::current_exception();
        }

        T value_;
        std::exception_ptr exception_{};
        std::coroutine_handle<> continuation_{};
    };

    std::coroutine_handle<promise_type> handle_;

    explicit SimpleTask(std::coroutine_handle<promise_type> h) : handle_(h) {}

    ~SimpleTask() {
        if (handle_) {
            handle_.destroy();
        }
    }

    SimpleTask(const SimpleTask&) = delete;
    SimpleTask(SimpleTask&& other) noexcept : handle_(std::exchange(other.handle_, {})) {}

    struct Awaiter {
        explicit Awaiter(std::coroutine_handle<promise_type> h) : handle_(h) {}

        ~Awaiter() {
            if (handle_) {
                handle_.destroy();
            }
        }

        bool await_ready() const noexcept {
            return false;
        }

        std::coroutine_handle<> await_suspend(std::coroutine_handle<> continuation) noexcept {
            handle_.promise().continuation_ = continuation;
            return handle_;
        }

        T await_resume() {
            return std::move(handle_.promise().value_);
        }

        std::coroutine_handle<promise_type> handle_{};
    };

    Awaiter operator co_await() && {
        return Awaiter{std::exchange(handle_, {})};
    }
};

SimpleTask<int> callee() {
    co_return 42;
}

SimpleTask<int> caller(int i) {
    int result = co_await callee();
    std::cout << "caller: " << std::this_thread::get_id() << std::endl;
    co_return result* i;
}

struct DummyReceiver {
    using receiver_concept = stdexec::receiver_t;

    void set_value(int v) && noexcept {
        std::cout << "DummyReceiver set_value as " << v << std::endl;
    }
    void set_error(std::exception_ptr) && noexcept {
        std::terminate();
    }
    void set_stopped() && noexcept {
        std::terminate();
    }
};

int main() {
    {
        std::cout << "Sender/Receiver style" << std::endl;
        // The caller coroutine will be implicitly co_awaited in main thread
        std::cout << "main: " << std::this_thread::get_id() << std::endl;
        stdexec::sender auto sender = caller(2);
        stdexec::receiver auto receiver = DummyReceiver{};
        // Connect the sender and receiver
        stdexec::operation_state auto op = stdexec::connect(std::move(sender), receiver);
        // Start the operation asynchronously
        stdexec::start(op);
    }
    {
        std::cout << "Asynchronously executed in thread pool" << std::endl;
        exec::static_thread_pool pool{3};
        auto scheduler = pool.get_scheduler();
        // `starts_on` returns a sender, `when_all` returns a sender too
        auto work = stdexec::when_all(stdexec::starts_on(scheduler, caller(1)),
                                      stdexec::starts_on(scheduler, caller(2)),
                                      stdexec::starts_on(scheduler, caller(3)));
        // `sync_wait` will block until all senders complete, which has a internal receiver to
        auto [i, j, k] = stdexec::sync_wait(std::move(work)).value();
    }
}

```

注意到：为什么`SimpleTask`没有实现`Sender`的相关接口，但是能被当成`Sender`使用？这背后的原理是，`stdexec`会检查`SimpleTask`能否满足`Awaitable`，即`SimpleTask`能否被`co_await`。然后通过`stdexec`中的`__connect_awaitable_t`将`SimpleTask`封装成Sender，具体流程如下：

1. `__connect_awaitable_t`的`operator()`能接受任意`Awaitable`和`Receiver`
    
    ```cpp
    template <class _Receiver, __awaitable<__promise_t<_Receiver>> _Awaitable>
    requires receiver_of<_Receiver, __completions_t<_Receiver, _Awaitable>>
    auto operator()(_Awaitable&& __awaitable, _Receiver __rcvr) const -> __operation_t<_Receiver> {
        return __co_impl(static_cast<_Awaitable&&>(__awaitable), static_cast<_Receiver&&>(__rcvr));
    }
    ```
    
    此处会通过`requires receiver_of<_Receiver, __completions_t<_Receiver, _Awaitable>>`检查`Awaitable`和`Receiver`是否匹配：
    
    `__completions_t<_Receiver, _Awaitable>`会声明`Awaitable`作为`Sender`时，它可能会调用`Receiver`的哪些方法，而`receiver_of`里会检查`Receiver`能否处理这些方法。即`completion_signatures`是`Sender`和`Receiver`之间的一个contract。
    
    ```cpp
    using __completions_t = completion_signatures<
      set_value_t() or set_value_t(T), // according to return type of Awatiable
      set_error_t(std::exception_ptr),
      set_stopped_t()
    >
    ```
    
2. `__co_impl`的具体实现如下，本质上就是在`co_await`传入的`Awaitable`对象。
    
    ```cpp
    static auto __co_impl(_Awaitable __awaitable, _Receiver __rcvr) -> __operation_t<_Receiver> {
        using __result_t = __await_result_t<_Awaitable, __promise_t<_Receiver>>;
        std::exception_ptr __eptr;
        STDEXEC_TRY {
            if constexpr (same_as<__result_t, void>)
                co_await (co_await static_cast<_Awaitable&&>(__awaitable),
                          __co_call(set_value, static_cast<_Receiver&&>(__rcvr)));
            else
                co_await __co_call(set_value,
                                   static_cast<_Receiver&&>(__rcvr),
                                   co_await static_cast<_Awaitable&&>(__awaitable));
        }
        STDEXEC_CATCH_ALL {
            __eptr = std::current_exception();
        }
        co_await __co_call(set_error,
                           static_cast<_Receiver&&>(__rcvr),
                           static_cast<std::exception_ptr&&>(__eptr));
    }
    ```
    
3. 当协程执行完成时，会获取到`Awaitable`的执行结果，然后调用`set_value`，如果发生异常时则调用`set_error`。

以`set_value`为例，对应`__co_impl`中核心代码就是：

```cpp
co_await __co_call(set_value,
                   static_cast<_Receiver&&>(__rcvr),
                   co_await static_cast<_Awaitable&&>(__awaitable));
```

本质上等价于：

```cpp
auto result = co_await static_cast<_Awaitable&&>(__awaitable);
set_value(std::move(receiver), result);
```

即，首先`co_await`传入的`Awaitable`，也就是`SimpleTask`，获取到其执行结果。然后调用`receiver.set_value(result)`，将结果传给Receiver。

回顾整个例子，可以发现`SimpleTask`并没有添加任何代码，那么`stdexec`是如何识别到`SimpleTask`，然后将其作为`Sender`来使用的呢？

## CPO

要回答这个问题，我们先从CPO(Customization Point Object)说起。CPO是C++20引入的一类特殊函数对象，用来解决如何安全、可扩展地让用户提供自定义行为。

在C++20之前，自定义行为主要依赖于：

1. 模板特化
2. ADL(Argument-Dependent Lookup)

模板特化比较简单，比如给自定义类型拓展`std::hash`就是通过模板特化实现的。ADL的一个常见例子就是：

```cpp
void foo(auto& a, auto& b) {
    using std::swap;
    swap(a, b); // ADL
}
```

如果没有`using std::swap`，编译器只会在当前作用域和参数相关命名空间查找`swap`。如果有`using std::swap`，则`std::swap`也会被引入当前作用域，参与重载决议。如果没有找到自定义`swap`，`std::swap`作为兜底方案会被选中。不难发现，通过ADL来自定义行为，是借助函数重载来完成的。一旦没有using相关的命名空间，会导致自定义重载没有被调用，也容易出现二义性调用，或者选中意料之外的版本。

而CPO本身则是一个特殊的函数对象，编译器会在编译器决定`operator()`的行为，避免了意外的重载。具体来说，标准库或者其他库，可以通过重载`operator()`，或者`if constexpr`的形式，告知用户应当如何自定义这些行为。

我们前面提到`stdexec`中会：

1. 通过`connect`将`SimpleTask`封装成`Sender`
2. 通过`set_value`将`Awaitable`结果传递给`Receiver`

这两个行为实际上都是通过CPO实现的，我们以`set_value`为例，结合`stdexec`代码来理解下CPO。

首先，`set_value`实际是个函数对象，其类型是`set_value_t`。`Receiver`的其他两个接口`set_error`和`set_stopped`也是一样的。

```cpp
  using __rcvrs::set_value_t;
  using __rcvrs::set_error_t;
  using __rcvrs::set_stopped_t;
  inline constexpr set_value_t set_value{};
  inline constexpr set_error_t set_error{};
  inline constexpr set_stopped_t set_stopped{};
```

`set_value_t`的代码如下：

```cpp
template <class _Receiver, class... _As>
concept __set_value_member = requires(_Receiver &&__rcvr, _As &&...__args) {
    static_cast<_Receiver &&>(__rcvr).set_value(static_cast<_As &&>(__args)...);
};

struct set_value_t {
    template <class _Fn, class... _As>
    using __f = __minvoke<_Fn, _As...>;

    // Receiver has set_value as member function
    template <class _Receiver, class... _As>
        requires __set_value_member<_Receiver, _As...>
    STDEXEC_ATTRIBUTE(host, device, always_inline)
    void operator()(_Receiver &&__rcvr, _As &&...__as) const noexcept {
        static_assert(noexcept(static_cast<_Receiver &&>(__rcvr).set_value(
                              static_cast<_As &&>(__as)...)),
                      "set_value member functions must be noexcept");
        static_assert(__same_as<decltype(static_cast<_Receiver &&>(__rcvr).set_value(
                                        static_cast<_As &&>(__as)...)),
                                void>,
                      "set_value member functions must return void");
        static_cast<_Receiver &&>(__rcvr).set_value(static_cast<_As &&>(__as)...);
    }

    // Receiver doesn't have set_value as member function
    template <class _Receiver, class... _As>
        requires(!__set_value_member<_Receiver, _As...>) &&
                tag_invocable<set_value_t, _Receiver, _As...>
    STDEXEC_ATTRIBUTE(host, device, always_inline)
    void operator()(_Receiver &&__rcvr, _As &&...__as) const noexcept {
        static_assert(nothrow_tag_invocable<set_value_t, _Receiver, _As...>);
        (void)tag_invoke(
                *this, static_cast<_Receiver &&>(__rcvr), static_cast<_As &&>(__as)...);
    }
};
```

这里对`operator()`提供了两种重载：

1. 如果有`set_value`这个成员函数，则直接调用`receiver.set_value()`。比如我们前面示例中，`DummyReceiver`就提供了`set_value`成员函数，当`Sender`执行完毕，就会通过这个成员函数，将结果告知给`Receiver`。
2. 如果没有`set_value`这个成员函数，但支持通过以`tag_invoke`进行自定义，那么就调用`tag_invoke`。我们在下面例子就能看到，`tag_invoke`就是一个函数模版，只要用户提供了这个函数模板，就能通过标签派发(tag dispatch)调用相应函数。

> 同理，`connect`时会通过相同的机制调用`__connect_awaitable_t`的`operator()`
> 

到这也就介绍完了`stdexec`如何通过CPO提供了让用户自定义接口的能力。接下来，我们以`SimpleTask`为例，看看如何在用户代码中自定义这些行为。`SimpleTask`的代码省略，和之前一样。

```cpp
namespace stdexec {
template <class S>
struct completion_signatures_of;

template <class T>
struct completion_signatures_of<SimpleTask<T>> {
    using type = completion_signatures<set_value_t(T),
                                       set_error_t(std::exception_ptr),
                                       set_stopped_t()>;
};
}  // namespace stdexec

struct TagInvokeReceiver {
    using receiver_concept = stdexec::receiver_t;
    int value = 0;
};

void tag_invoke(stdexec::set_value_t, TagInvokeReceiver&& r, int v) noexcept {
    r.value = v * 100;
    std::cout << "TagInvokeReceiver set_value as " << r.value << std::endl;
}
void tag_invoke(stdexec::set_error_t, TagInvokeReceiver&&, std::exception_ptr) noexcept {}
void tag_invoke(stdexec::set_stopped_t, TagInvokeReceiver&&) noexcept {}

template <class R>
struct TaskOp {
    R receiver_;
    SimpleTask<int> task_;
    void start() noexcept {
        try {
            if (!task_.handle_.done()) {
                task_.handle_.resume();
            }
            auto v = task_.handle_.promise().value_;
            stdexec::set_value(std::move(receiver_), v);
        } catch (...) {
            stdexec::set_error(std::move(receiver_), std::current_exception());
        }
    }
};

template <class R>
auto tag_invoke(stdexec::connect_t, SimpleTask<int>&& task, R&& r) -> TaskOp<std::decay_t<R>> {
    return TaskOp<std::decay_t<R>>{std::forward<R>(r), std::move(task)};
}

int main() {
    stdexec::sender auto sender = caller(2);
    stdexec::operation_state auto op = stdexec::connect(std::move(sender), TagInvokeReceiver{});
    stdexec::start(op);
}

```

首先我们通过`completion_signatures_of`，声明了`SimpleTask`对应的`Receiver`需要提供哪些方法。然后自定义了一个`Receiver`类`TagInvokeReceiver`，之后就用这个类来保存`SimpleTask`的执行结果。此处`using receiver_concept = stdexec::receiver_t`是为了告知`stdexec`它可以被用作`Receiver`。

```cpp
struct TagInvokeReceiver {
    using receiver_concept = stdexec::receiver_t;
    int value = 0;
};
```

之后就提供了对应的`tag_invoke`函数，这样，`stdexec`在使用`set_value_t`对象时，就能调用到我们提供的`tag_invoke`函数了：

```cpp
void tag_invoke(stdexec::set_value_t, TagInvokeReceiver&& r, int v) noexcept {
    r.value = v * 100;
    std::cout << "TagInvokeReceiver set_value as " << r.value << std::endl;
}
```

同理，`connect`也是一个CPO，我们通过另一个`tag_invoke`函数，自定义了每次`connect`一个`SimpleTask`和`Receiver`时，都返回`TaskOp`。

```cpp
template <class R>
auto tag_invoke(stdexec::connect_t, SimpleTask<int>&& task, R&& r) -> TaskOp<std::decay_t<R>> {
    return TaskOp<std::decay_t<R>>{std::forward<R>(r), std::move(task)};
}
```

最后，`stdexec::start`本身也是个CPO，如果没有通过`tag_invoke`自定义的话，就是调用传入对象的`start`方法，在我们例子中也就是`TaskOp`的`start`方法。运行这个代码，就会发现SimpleTask的结果被传给了`TagInvokeReceiver`。

```cpp
TagInvokeReceiver set_value as 8400
```

## At last

这篇文章差不多就到此结束了。P2300提出了一种统一的抽象来完成异步任务，这些概念需要一定时间去消化。而从业务代码上而言，如果你不需要去实现一个Sender（这通常是基础库需要做的事），而是只需要使用一个`Sender`，那倒不需要太大改工，只需要`co_await`一个`Sender`，或者是调用一个类似`sync_wait`的方法即可。但鉴于P2300饱受争议，又是一个刚采纳没多久的提案，目前只有几个POC的库支持了P2300，比如stdexec和libunifex，其余的基础库比如Boost、Folly和Abseil还处于没支持或者很早期的阶段。从一个基础架构的开发者的角度来说，一个项目一般不会使用很多种异步代码的编写方式，异步代码的技术栈更换说不定远远跟不上新的C++标准更新速度，这未免会让很多人敬而远之。

## Reference

[P2300R10: `std::execution`](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2300r10.html)

[https://github.com/NVIDIA/stdexec](https://github.com/NVIDIA/stdexec)

[Working with Asynchrony Generically: A Tour of C++ Executors (part 1/2) - Eric Niebler - CppCon 21](https://www.youtube.com/watch?v=xLboNIf7BTg)

[Structured Concurrency: Writing Safer Concurrent Code with Coroutines... - Lewis Baker - CppCon 2019 - YouTube](https://www.youtube.com/watch?v=1Wy5sq3s2rg)

[What are Senders Good For, Anyway? – Eric Niebler](https://ericniebler.com/2024/02/04/what-are-senders-good-for-anyway/)

[A Universal Async Abstraction for C++ | cor3ntin](https://cor3ntin.github.io/posts/executors/)