---
layout: single
title: Deciphering C++ Coroutines, part 2
date: 2025-11-06 00:00:00 +0800
categories: 学习
tags: C++
---

这篇会继续从`promise_type`的视角，更完整的介绍编译器是如何把协程转换成一个固定的三段式代码，以及`promise_type`是如何自定义了协程的行为。好在有了上一篇对相关概念的介绍，我们终于可以通过demo来研究其中的奥秘了。

## promise_type

首先，我们需要再回顾一下`promise_type`的作用，上一篇我们是这么描述的：

`promise_type`接口指定了自定义协程本身行为的方法。基础库编写者能够自定义：当协程被调用时发生什么、当协程返回时发生什么（无论是通过正常方式还是通过未处理的异常）。这些自定义点可以执行任意逻辑。

除此以外，有一个点上一篇没有展开：`promise_type`还可以通过`await_transform`定义在协程中`co_await`或`co_yield`时行为。注意这里说的是在当前这个协程中进行`co_await`或`co_yield`。

`promise_type`的完整接口如下：

```cpp
struct promise_type {
    // creating coroutine object - mandatory
    ReturnType get_return_object();

    // returns awaitable object - mandatory
    auto initial_suspend();
    auto final_suspend();

    void unhandled_exception();  // mandatory
    // one of below is mandatory and only one must be present
    void return_value(/*type*/);
    void return_void();

    // support for yielding values - returns awaitable
    auto yield_value();

    // modification of the awaitable
    auto await_transform(/*co_await operand*/);
}
```

- `get_return_object`：用于从`promise_type`对象获取对应的`ReturnType`对象。当协程到达其第一个挂起点并且控制流返回给调用方时，调用方将通过调用`get_return_object`获得一个`ReturnType`对象。这些自定义点可以执行任意逻辑。
- `return_void`/`return_value`/`unhandled_exception`：自定义点，用于处理协程到达`co_return`语句时的行为以及异常处理方式。
- `initial_suspend`：自定义点，用于自定义协程体在执行之前的行为，比如是立即执行还是lazily启动。
- `final_suspend`：自定义点，用于协程体执行之后的行为，比如协程由谁在什么时候析构。
- `unhandled_exception`：处理协程执行过程中抛出的异常。
- `return_value`和`return_void`：保存协程的返回值。
- `yield_value`：本质上`co_yield <expr>`会被编译器翻译为`co_await promise.yield_value(<expr>)`，可以通过`yield_value`来自定义`co_yield`的行为。后面如无特殊情况，不会单独再介绍`co_yield`.
- `await_transform`：每次协程内部执行`co_await`时，通过拦截并改写`Awaitable`对象。

## Coroutine body

编译器会把一个协程展开为下面三段式代码：

1. `co_await promise.initial_suspend();`
2. `coroutine body`
3. `co_await promise.final_suspend();`

三段式示意代码展开如下（省略了部分现在不需要关注的细节），在这一篇中我们把其中的`co_await`也展开：

```cpp
// Pretend there's a compiler-generated structure called 'coroutine_frame'
// that holds all of the state needed for the coroutine. Its constructor
// takes a copy of parameters and default-constructs a promise object.
struct coroutine_frame { ... };

ReturnType some_coroutine() {
    auto* f = new coroutine_frame(...);
    auto returnObject = f->promise.get_return_object();
    expanded_coroutine(f);
    return returnObject;
}

void expanded_coroutine(coroutine_frame* f) {
    try {
        // 1. co_await promise.initial_suspend() is expanded below
        {
            auto&& awaitable = f->promise.initial_suspend();
            auto&& awaiter = awaitable;
            if (!awaiter.await_ready()) {
                <suspend-coroutine>
                awaiter.await_suspend(coroutine_handle<promise_type>::from_promise(promise));
                // return to caller
                return;
            }
            <resume-point>
            awaiter.await_resume();
        }

        // 2. coroutine body
        <body-statements>
        f->promise.return_void() or f->promise.return_value(...);
        // destruct all local variables in reverse order
        goto final_suspend_label;
    } catch (...) {
        f->promise.unhandled_exception();
        // destruct all local variables in reverse order
        goto final_suspend_label;
    }

final_suspend_label:
    // 3. co_await promise.final_suspend() is expanded below
    {
        auto&& awaitable = f->promise.final_suspend();
        auto&& awaiter = awaitable;
        if (!awaiter.await_ready()) {
            <suspend-coroutine>
            awaiter.await_suspend(coroutine_handle<promise_type>::from_promise(promise));
            // return to caller
            return;
        }
    }
    <destory couroutine frame>
}
```

大致步骤如下：

1. 构造coroutine frame，包括coroutine frame中的`promise_type`对象。
2. 通过`promise_type`中的`get_return_object`方法得到`ReturnType`对象，`ReturnType`对象在协程第一次挂起或结束时返回给调用方。
3. 之后会`co_await promise.initial_suspend`，通过`promise_type`自定义协程体在执行之前的行为，当`initial_suspend`被恢复时，协程体开始执行。
4. 当协程执行完时，根据返回值的不同，`return_void`或者`return_value`会被调用，结果会被保存在`promise_type`中。如果执行过程中出现异常，则`unhandled_exception`会被调用。之后所有协程函数体重的局部变量都会被析构。
5. 无论返回值是哪种，最终都会跳转到`final_suspend_label`，调用`co_await promise.final_suspend`，通过`promise_type`自定义协程体执行之后的行为。
6. `<destory couroutine frame>`处会析构coroutine frame，具体时机根据`final_suspend`会有所不同，后面会介绍。

下面会详细介绍具体的流程。

### Allocating a coroutine frame

首先，编译器会生成对`operator new`的调用来为coroutine frame分配内存，其中包括：

- `promise_type`对象
- 所有协程参数
- 关于协程当前挂起点的信息以及如何恢复/析构它
- 任何生命周期跨越挂起点的局部变量

协程需要将原始调用方传递给协程函数的所有参数复制到coroutine frame中，以便它们在协程挂起后仍然有效。如果参数是按值传递给协程的，那么这些参数通过调用类型的移动构造函数被复制到coroutine frame中。如果参数是按引用传递给协程的（无论是左值引用还是右值引用），那么只有引用被复制到coroutine frame中。一旦所有参数都被复制到coroutine frame中，协程就会构造promise对象。

类似的，coroutine frame析构时涉及：

1. 调用`promise_type`对象的析构函数。
2. 调用coroutine frame中协程参数析构函数。
3. 调用`operator delete`释放coroutine frame使用的内存。

### Executing coroutine body

1. 获取返回对象

    协程对`promise_type`对象做的第一件事是通过调用`promise.get_return_object()`来获取`ReturnType`对象。当协程首次挂起或运行到完成并将执行返回给调用方后，会把`ReturnType`对象返回给协程调用方。

2. `initial_suspend`

    一旦coroutine frame初始化完成并获得`ReturnType`对象后，接下来执行的是`co_await promise.initial_suspend()`。这允许通过`promise_type`控制协程是应该在执行协程体之前挂起，还是立即开始执行协程体。

    如果协程在`initial_suspend`点挂起，那么它可以稍后通过在协程的`coroutine_handle`上调用`resume()`或`destroy()`来恢复或析构。另外，注意到编译器生成的代码中不会处理`initial_suspend`的`await_resume`返回值，即`co_await promise.initial_suspend()`表达式的结果会被丢弃，因此一般会返回`void`。

    对于许多类型的协程，`initial_suspend()`方法要么返回`std::suspend_always`（协程延迟启动），要么返回`std::suspend_never`（协程立即启动）。

3. 返回给调用方

    当协程第一次被挂起时，或者没有任何一次挂起，则是当协程执行完成时，从`get_return_object()`调用返回的`ReturnType`对象会被返回给协程的调用方。

4. 使用`co_return`从协程返回

    当协程到达`co_return`语句时，它会被转换为`promise.return_void()`或`promise.return_value(<expr>)`，接着是`goto final_suspend_label`。注意，如果执行在没有`co_return`语句的情况下运行到协程的末尾，这相当于在函数体末尾有一个`co_return`。

    具体规则如下：

    - `co_return;`转换为`promise.return_void();`
    - `co_return <expr>;`
        - 如果`<expr>`的类型是`void`，则转换为`<expr>; promise.return_void();`
        - 如果`<expr>`的类型不是`void`，则转换为`promise.return_value(<expr>);`

    随后的`goto final_suspend_label`会导致所有具有局部变量按构造的相反顺序析构。

5. 处理从协程体传播出的异常

    如果协程体中抛出了异常，则异常会被捕获，并在`catch`块内调用`promise.unhandled_exception()`方法。通常实现常会调用`std::current_exception()`来捕获异常并将其存储起来，之后通过`promise_type`的相关接口检查是否在协程运行过程中出现异常。

6. `final_suspend`

    一旦协程体执行完成，并且调用`return_void()`、`return_value()`或`unhandled_exception()`处理了返回结果后，协程有机会在将控制流返回给调用方/恢复方之前执行一些额外的逻辑。即协程通过执行`co_await promise.final_suspend()`执行自定义逻辑，例如发布结果、发出完成信号或Resume continuation（下面会介绍，简单来说就是当前协程执行完时唤醒另一个协程继续执行），也允许协程在析构coroutine frame之前挂起。

    如果协程在`final_suspend`点挂起，对这个协程调用`resume()`是未定义行为，唯一能做的就是`destroy()`。也可以注意到，编译器生成的代码中，`final_suspend`是没有恢复点的，它对应的`Awaitable`的`await_resume`永远不会被调用。另外`final_suspend`必须是`noexcept`。

7. 析构coroutine frame

    在被展开的代码，最后一部分就是析构整个coroutine frame。注意到协程体已经完全执行完毕，没有更多的用户代码需要执行，实际上就是`final_suspend`的`await_ready`决定了coroutine frame什么时候被析构：

    - `await_ready`返回`true`，代表不挂起并立即析构
    - `await_ready`返回`false`，代表挂起并等待外部调用析构

    虽然标准允许协程在`final_suspend`点不挂起，但从工程时间角度应当挂起。这样会要求从协程外部对协程调用`.destroy()`，常见手段是某个RAII对象的析构函数。


## Demo

我们用一些实际例子来理解`promise_type`如何自定义协程的行为。

### Coroutine state machine

首先是一个Hello World示例，通过它可以进一步了解编译器是如何将协程展开的。

```cpp
#include <coroutine>
#include <iostream>

struct ReturnType {
    struct promise_type {
        ReturnType get_return_object() {
            return ReturnType{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() {
            return {};
        }
        std::suspend_always final_suspend() noexcept {
            return {};
        }
        void return_void() {}
        void unhandled_exception() {}
    };

    explicit ReturnType(std::coroutine_handle<promise_type> h) : handle(h) {}

    ~ReturnType() {
        if (handle) {
            handle.destroy();
        }
    }

    std::coroutine_handle<promise_type> handle;
};

ReturnType hello(const std::string& to, int times) {
    for (int i = 0; i < times; ++i) {
        std::cout << "Hello " << to << "\n";
    }
    co_return;
}

int main() {
    auto coro = hello("World", 3);
    return 0;
}
```

注意到运行这个程序是不会打印Hello World的，原因是`initial_suspend`返回了`suspend_always`，想让它打印可以改成`suspend_never`。即`promise_type`通过`initial_suspend`自定义了协程是延迟执行还是立即执行。

我们把这个代码放到cppinsight中，勾选上`Show coroutine transformation`即可。

对于每个协程，都会生成它对应的coroutine frame，其中包含了promise，以及协程恢复和析构时的回调，以及协程的几个函数。此外注意到还有一个`__suspend_index`用来保存当前协程是在第几个挂起点挂起，以便恢复时能够正确执行剩余逻辑。

```cpp
struct __helloFrame
{
  void (*resume_fn)(__helloFrame *);
  void (*destroy_fn)(__helloFrame *);
  std::__coroutine_traits_impl<ReturnType>::promise_type __promise;
  int __suspend_index;
  bool __initial_await_suspend_called;
  const std::basic_string<char, std::char_traits<char>, std::allocator<char> > & to;
  int times;
  int i;
  std::suspend_always __suspend_30_12;      // initial_suspend
  std::suspend_always __suspend_30_12_1;    // final_suspend
};
```

协程的代码被展开成下面的流程：

1. 构造coroutine frame
2. 保存协程参数
3. 构造`promise_type`对象
4. 设置恢复和析构时的回调
5. 调用三段式展开函数`__helloResume`
6. 返回`ReturnObject`对象

```cpp
ReturnType hello(const std::basic_string<char, std::char_traits<char>, std::allocator<char> > & to, int times)
{
  /* Allocate the frame including the promise */
  /* Note: The actual parameter new is __builtin_coro_size */
  __helloFrame * __f = reinterpret_cast<__helloFrame *>(operator new(sizeof(__helloFrame)));
  __f->__suspend_index = 0;
  __f->__initial_await_suspend_called = false;
  __f->to = std::forward<const std::basic_string<char, std::char_traits<char>, std::allocator<char> > &>(to);
  __f->times = std::forward<int>(times);

  /* Construct the promise. */
  new (&__f->__promise)std::__coroutine_traits_impl<ReturnType>::promise_type{};

  /* Forward declare the resume and destroy function. */
  void __helloResume(__helloFrame * __f);
  void __helloDestroy(__helloFrame * __f);

  /* Assign the resume and destroy function pointers. */
  __f->resume_fn = &__helloResume;
  __f->destroy_fn = &__helloDestroy;

  /* Call the made up function with the coroutine body for initial suspend.
     This function will be called subsequently by coroutine_handle<>::resume()
     which calls __builtin_coro_resume(__handle_) */
  __helloResume(__f);


  return __f->__promise.get_return_object();
}
```

三段式函数被展开为`__helloResume`。可以看到，本质上协程体就变成了一个状态机，即协程在挂起时，准确说是在调用`await_suspend`之后，会设置corourinte frame中的挂起点的下标`__suspend_index`，之后会返回给调用方。而之后每次调用`coroutine_handle::resume()`时，都会调用这个函数中，并通过`__suspend_index`跳转到相应的恢复点并继续执行。

具体逻辑如下：

```cpp
void __helloResume(__helloFrame * __f)
{
  try
  {
    /* Create a switch to get to the correct resume point */
    switch(__f->__suspend_index) {
      case 0: break;
      case 1: goto __resume_hello_1;
      case 2: goto __resume_hello_2;
    }

    /* co_await insights.cpp:30 */
    __f->__suspend_30_12 = __f->__promise.initial_suspend();
    if(!__f->__suspend_30_12.await_ready()) {
      __f->__suspend_30_12.await_suspend(std::coroutine_handle<ReturnType::promise_type>::from_address(static_cast<void *>(__f)).operator std::coroutine_handle<void>());
      __f->__suspend_index = 1;
      __f->__initial_await_suspend_called = true;
      return;
    }

__resume_hello_1:
    __f->__suspend_30_12.await_resume();
    for(__f->i = 0; __f->i < __f->times; ++__f->i) {
      std::operator<<(std::operator<<(std::operator<<(std::cout, "Hello "), __f->to), "\n");
    }

    /* co_return insights.cpp:34 */
    __f->__promise.return_void();
    /* co_return insights.cpp:30 */
    __f->__promise.return_void()/* implicit */;
    goto __final_suspend;
  } catch(...) {
    if(!__f->__initial_await_suspend_called) {
      throw ;
    }

    __f->__promise.unhandled_exception();
  }

__final_suspend:

  /* co_await insights.cpp:30 */
  __f->__suspend_30_12_1 = __f->__promise.final_suspend();
  if(!__f->__suspend_30_12_1.await_ready()) {
    __f->__suspend_30_12_1.await_suspend(std::coroutine_handle<ReturnType::promise_type>::from_address(static_cast<void *>(__f)).operator std::coroutine_handle<void>());
    __f->__suspend_index = 2;
    return;
  }

__resume_hello_2:
  __f->destroy_fn(__f);
}
```

其余部分的代码就不再重复解释，对应前面的流程理解即可。

### Resume continuation

了解了这个状态机后以及`co_await promise.initial_suspend()`自定义协程执行之前的行为之后，下面开始介绍协程通过`co_await promise.final_suspend()`在协程函数体执行后自定义行为。其中一种常见行为就是Resume continuation，即在当前协程执行完成时，唤醒其他协程继续执行，从而实现协程之间的控制流转移。我们用下面的例子来解释：

```cpp
#include <coroutine>
#include <iostream>
#include <utility>

template<typename T>
struct SimpleTask {
    struct promise_type {
        SimpleTask get_return_object() {
            return SimpleTask{std::coroutine_handle<promise_type>::from_promise(*this)};
        }

        std::suspend_always initial_suspend() { return {}; }

        struct FinalAwaiter {
            bool await_ready() noexcept { return false; }

            void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
                std::cout << "      > FinalAwaiter: coroutine " << h.address()
                          << " completed, ready to resume continuation\n";
                auto continuation = h.promise().continuation_;
                if (continuation) {
                    std::cout << "      > FinalAwaiter: resume continuation -> "
                              << continuation.address() << "\n";
                    continuation.resume();
                }
            }

            void await_resume() noexcept {}
        };

        FinalAwaiter final_suspend() noexcept { return {}; }

        void return_value(T v) {
            std::cout << "      > promise: return_value(" << v << ")\n";
            value_ = v;
        }
        void unhandled_exception() { exception_ = std::current_exception(); }

        T value_;
        std::exception_ptr exception_{};
        std::coroutine_handle<> continuation_{};
    };

    std::coroutine_handle<promise_type> handle_;

    explicit SimpleTask(std::coroutine_handle<promise_type> h) : handle_(h) {}

    ~SimpleTask() {
        std::cout << "      > ~SimpleTask: destruct\n";
        if (handle_) {
            std::cout << "      > ~SimpleTask: handle_.destroy() "
                      << handle_.address() << "\n";
            handle_.destroy();
        }
    }

    SimpleTask(const SimpleTask &) = delete;
    SimpleTask(SimpleTask&& other) noexcept
        : handle_(std::exchange(other.handle_, {})) {}

    struct Awaiter {
        explicit Awaiter(std::coroutine_handle<promise_type> h) : handle_(h) {}

        ~Awaiter() {
            if (handle_) {
                std::cout << "    > Awaiter: ~Awaiter() handle_.destroy() "
                          << handle_.address() << "\n";
                handle_.destroy();
            }
        }

        bool await_ready() const noexcept {
            return false;
        }

        void await_suspend(std::coroutine_handle<> continuation) noexcept {
            std::cout << "    > Awaiter: await_suspend() - save continuation "
                      << continuation.address() << "\n";
            // Store the continuation in the SimpleTask's promise so that the final_suspend()
            // knows to resume this coroutine when the task completes.
            handle_.promise().continuation_ = continuation;
            // Then we resume the SimpleTask's coroutine, which is currently suspended
            // at the initial-suspend-point (ie. at the open curly brace).
            handle_.resume();
        }

        T await_resume() {
            std::cout << "    > Awaiter: await_resume() - continuation resumed (coroutine "
                      << handle_.address() << " completed)\n";
            return std::move(handle_.promise().value_);
        }

        std::coroutine_handle<promise_type> handle_{};
    };

    Awaiter operator co_await() && {
        std::cout << "  > SimpleTask: operator co_await()\n";
        return Awaiter{std::exchange(handle_, {})};
    }
};

SimpleTask<int> callee() {
    std::cout << "      > callee()\n";
    co_return 42;
}

SimpleTask<int> caller() {
    std::cout << "  > caller()\n";
    int result = co_await callee();
    std::cout << "  > caller: result = " << result << "\n";
    co_return result * 2;
}

int main() {
    auto task = caller();
    std::cout << "> main: start caller coroutine " << task.handle_.address() << "\n";
    task.handle_.resume();
    std::cout << "> main: caller coroutine completed, final result = "
              << task.handle_.promise().value_ << "\n";
    return 0;
}
```

我们先不管具体实现细节，理解下主干代码：`SimpleTask`是一个延迟启动的协程，在`main()`中调用了`caller()`，并手动启动了这个协程。在`caller()`协程中，又`co_await`了另一个协程`callee()`，并最终使用`co_await callee()`的返回值，返回结果`result * 2`。

接下来分析`SimpleTask`是如何实现Resume continuation。在`SimpleTask`中，实现了以下组件：

- 重载了`operator co_await`，每当`co_await`一个`SimpleTask`时，会调用嵌套类`Awaiter`，自定义协程被挂起时的行为。注意只有`caller()`中`co_await callee()`时会构造`Awaiter`并调用相关接口，`caller`协程自身是被`main()`手动启动的。
- `SimpleTask`作为一个`ReturnType`，它内嵌了一个`promise_type`。`promise_type`中`initial_suspend`是`suspend_always`，而`final_suspend`则有所不同，又实现了一个新的`Awaitable`，即`FinalAwaiter`。正是通过`FinalAwaiter`，自定义了协程之后的行为。

为了更好的说明`SimpleTask`是实现Resume continuation的原理，按照我们之前描述的流程，`caller`会展开为如下代码，其中稍微调整和简化了其中一些步骤以便于理解。

```cpp
SimpleTask<int> caller() {
    auto caller_frame = new coroutine_frame(...);
    auto caller_promise = f->promise;
    auto caller_return_object = caller_frame->get_return_object();
    using handle_t = std::coroutine_handle<SimpleTask<int>::promise_type>;

    co_await caller_promise.initial_suspend();

    try {
        // co_await callee() is expanded below:
        auto&& awaitable = callee();
        auto&& awaiter = awaitable.operator co_await();
        if (!awaiter.await_ready()) {
            awaiter.await_suspend(handle_t::from_promise(caller_promise));
            return caller_return_object;
        }
        // when callee() is resumed
        auto result = awaiter.await_resume();

        co_return result * 2;
    } catch (...) {
        // ...
    }

    co_await caller_promise.final_suspend();
    // ...
}
```

接下来我们看下具体执行流程：

```
1. main调用caller()
   ├─ 创建caller()协程
   ├─ promise_type中的initial_suspend为suspend_always，协程会被挂起返回SimpleTask
   ├─ 返回SimpleTask
   └─ 通过SimpleTask中的coroutine_handle手动恢复caller()协程继续执行

2: caller()执行co_await callee()，此时callee会挂起
   ├─ 创建callee()协程
   ├─ 对callee()的返回对象(即SimpleTask对象)调用operator co_await，
   │  返回值为一个Awaiter，注意callee()协程的coroutine_handle转交给这个Awaiter，
   │  即return Awaiter{std::exchange(handle_, {})};
   ├─ Awaiter::await_ready返回false，即callee需要挂起
   └─ Awaiter::await_suspend(caller's coroutine handle)，
      ├─ 保存caller的coroutine_handle到callee_promise中
      └─ 随后立即恢复callee
```

注意，由于当前控制流是在执行`caller`这个协程函数体，即在`caller`的视角，`callee`只是一个普通函数，所以`Awaiter::await_suspend`传入的参数是`caller()`的`coroutine_handle`。

此处的`await_suspend`是理解Resume continuation的关键：

它将`caller()`协程的`coroutine_handle`保存到被等待协程的promise中，也就是`callee()`协程的promise中，记为`callee_promise`。通过`callee()`协程的promise，`caller()`协程就能在之后的步骤中被恢复。

随后立即恢复`callee()`协程（`Awaiter`的构造中会将传入的`coroutine_handle`保存为成员变量，即`callee()`协程的`coroutine_handle`），这个操作由更复杂的机制触发，这里只是为了简化实现而立即恢复。

```cpp
void await_suspend(std::coroutine_handle<> continuation) noexcept {
    // Store the `continuation` in promise so that the final_suspend()
    // knows to resume `continuation` coroutine when current task completes.
    std::cout << "    > Awaiter: await_suspend() - save continuation "
              << continuation.address() << "\n";
    handle_.promise().continuation_ = continuation;
    // Then we resume current task coroutine, which is currently suspended
    // at the initial-suspend-point (ie. at the open curly brace).
    handle_.resume();
}
```

随后，恢复`Awaiter`中`coroutine_handle`对应的协程，即`callee()`协程继续执行。

```
3. callee()被恢复，继续执行，完成时唤醒caller
   ├─ co_return 42
   │  └─ 调用callee_promise.return_value(42)
   └─ co_await callee_promise.final_supend，即co_await FinalAwaiter{}
      ├─ FinalAwaiter::await_ready返回false, callee会被挂起
      └─ FinalAwaiter::await_suspend(callee's coroutine handle)
         └─ 恢复caller执行
```

当`callee`协程体执行完成时，最后会`co_await callee_promise.final_suspend()`，此时控制流是可以交还给`caller`。注意到在第2步中，`Awaiter::await_suspend(caller's coroutine handle)`已经将`caller()协程`的`coroutine_handle`保存到了`callee_promise`中。此时`callee`协程执行完成，可以在`FinalAwaiter::await_suspend`中并直接读取出`caller`协程对应的`coroutine_handle`，然后恢复：

```cpp
void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
    // current coroutine is now suspended at the final-suspend point

    // In this case, `h` is current coroutine, aka callee's coroutine_handle
    // `continuation` is caller's coroutine_handle
    auto continuation = h.promise().continuation_;
    if (continuation) {
        continuation.resume();
    }
}
```

控制流会返回到`caller()`协程中继续执行：

```cpp
4. caller继续执行
   ├─ 此时callee已经执行完成，通过Awaiter::await_resume读取callee的结果
   ├─ result = 42
   ├─ 调用caller_promise.return_value(84)
   └─ co_await caller_promise.final_suspend，即co_await FinalAwaiter{}
      ├─ FinalAwaiter::await_ready返回false, caller会被挂起
      └─ FinalAwaiter::await_suspend(caller's coroutine handle)
```

即`co_await callee()`对应的`Awaiter::await_resume`会被调用，从而读取到`co_await callee()`的返回值42。注意在这个过程中，`co_await callee()`对应的`Awaiter`会在`await_resume`之后，就出作用域并析构`callee()`协程的coroutine frame，而`callee()`协程的返回值`SimpleTask`也会在`co_await callee()`执行完成之后析构，注意这个`SimpleTask`中的`coroutine_handle`为空（之前已经交给对应的`Awaiter`了）。

当`caller`协程体执行完成时，最后会`co_await caller_promise.final_suspend()`，注意`call_promise`中的`continuation_`为空，即没有协程在等待`caller`执行完成，因此对应的`FinalAwaiter::await_suspend`中什么都不会执行。

此时`caller`协程体执行完，控制流交还给`main`，通过`SimpleTask`拿到对应的promise，也就能读取到`caller`协程的返回值。最终，`main`中的`SimpleTask`析构，销毁`caller`的coroutine frame。

整个例子的输出可能是这样的，可以对照着上述流程加深理解：

- `0x57edddf392b0`是`caller()`协程的`coroutine_handle`
- `0x57edddf39720`是`callee()`协程的`coroutine_handle`

```cpp
> main: start caller coroutine 0x57edddf392b0
  > caller()
  > SimpleTask: operator co_await()
    > Awaiter: await_suspend() - save continuation 0x57edddf392b0
      > callee()
      > promise: return_value(42)
      > FinalAwaiter: coroutine 0x57edddf39720 completed, ready to resume continuation
      > FinalAwaiter: continuation.resume() -> 0x57edddf392b0
    > Awaiter: await_resume() - continuation resumed (coroutine 0x57edddf39720 completed)
    > Awaiter: ~Awaiter() handle_.destroy() 0x57edddf39720
      > ~SimpleTask: destruct
  > caller: result = 42
      > promise: return_value(84)
      > FinalAwaiter: coroutine 0x57edddf392b0 completed, ready to resume continuation
> main: caller coroutine completed, final result = 84
      > ~SimpleTask: destruct
      > ~SimpleTask: handle_.destroy() 0x57edddf392b0
```

整体上Resume continuation分为两部分：

- `Awaiter`连接了等待者`caller`和被等待者`callee`，即把等待者的`coroutine_handle`保存到了被等待者的promise中，即将等待者注册为被等待者的continuation
- 而`promise_type`中的`FinalAwaiter`，通过自定义协程执行完成后的行为，使得被等待协程`callee`完成时能自动恢复`caller`等待者的执行。

因此Resume continuation能够让协程之间自动形成调用链，使代码在保持同步风格的同时，实现异步操作。整个调用链中不需要传递回调函数，每当被等待者完成时，就能自动恢复调用者继续执行。但是Resume continuation会造成栈的深度迅速增长。我们可以把这个例子的几个步骤串联起来，得到类似下面的调用栈。其中Resume continuation发生在`continuation.resume()`这一步，从`callee`栈上又生长出了`caller`的栈。而从调用关系上，明明是`caller`调用了`callee`。

```
main()
└─ caller协程体 // 通过手动调用caller.handle_.resume()
   └─ Awaiter::await_suspend(caller's handle) // Awaiter指co_await callee()对应的awaiter
      └─ handle_.resume() // 立即恢复了callee
         └─ callee协程体
            └─ FinalAwaiter::await_suspend(callee's handle) // co_await callee_promise.final_supend
              └─ continuation.resume() // 恢复caller 即所谓resume continuation
                  └─ caller协程体
                     └─ Awaiter::await_resume() // Awaiter指co_await callee()对应的awaiter
```

> 关于stack overflow，可以看这篇[博客](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer)里举的[例子](https://godbolt.org/z/gy5Q8q)。
>

有没有其他办法能够既保证协程之间形成调用链，而又不会造成stack overflow呢，答案就是上一篇提到的对称转移，具体的方法要等到下一篇再揭晓了。

### await_transform

最后一部分，我们再总结一下`promise_type`如何通过`await_transform`定义在协程的体中`co_await`或`co_yield`时行为，注意是当前协程中。

首先，`await_transform`是在`co_await`中获取`Awaitable`这一步会被调用：

```cpp
template <typename promise_type, typename T>
decltype(auto) get_awaitable(promise_type& promise, T&& expr) {
    if constexpr (has_any_await_transform_member_v<promise_type>)
        return promise.await_transform(static_cast<T&&>(expr));
    else
        return static_cast<T&&>(expr);
}
```

`await_transform`的本质作用都是自定义某些类型在co_await时的行为，比如

- 让原本不是`Awaitable`的类型变成`Awaitable`

    例如，一个返回类型为`std::optional<T>`的协程的`promise_type`可以提供一个`await_transform()`重载，该重载接受`std::optional<U>`参数并返回一个`Awaitable`类型，这个`Awaitable`类型要么返回`U`类型的值，要么在被等待的值是`std::nullopt`时挂起协程。

    ```cpp
    template <typename T>
    class optional_promise {
        template <typename U>
        auto await_transform(std::optional<U>& value) {
            class awaiter {
                std::optional<U>& value;
            public:
                explicit awaiter(std::optional<U>& x) noexcept : value(x) {}
                bool await_ready() noexcept {
                    return value.has_value();
                }
                void await_suspend(std::coroutine_handle<>) noexcept {}
                U& await_resume() noexcept {
                    return *value;
                }
            };
            return awaiter{value};
        }
    };
    ```

- 通过将`await_transform`重载声明为`deleted`来禁止等待某些类型

    例如，一个返回类型为`std::generator<T>`的协程的`promise_type`类型可能会声明`await_transform()`为`deleted`。也就是禁止在这个协程内使用`co_await`，而只能使用`yield_value`（对于generator类型的协程是合理的）。

- folly通过`await_transform`，提供了一个magic value，即`co_await co_current_executor`获取当前协程在哪个executor上执行，

    ```cpp
    // Special placeholder object that can be 'co_await'ed from within a Task<T>
    // or an AsyncGenerator<T> to obtain the current folly::Executor associated
    // with the current coroutine.
    //
    // Note that for a folly::Task the executor will remain the same throughout
    // the lifetime of the coroutine. For a folly::AsyncGenerator<T> the current
    // executor may change when resuming from a co_yield suspend-point.
    //
    // Example:
    //   folly::coro::Task<void> example() {
    //     Executor* e = co_await folly::coro::co_current_executor;
    //     e->add([] { do_something(); });
    //   }

    class TaskPromiseBase {
      // ...
      auto await_transform(co_current_executor_t) noexcept {
        return ready_awaitable<folly::Executor*>{executor_.get()};
      }
      // ...
    };

    template <typename T = void>
    class ready_awaitable {
      static_assert(!std::is_void<T>::value, "base template unsuitable for void");

     public:
      explicit ready_awaitable(T value) //
          noexcept(noexcept(T(FOLLY_DECLVAL(T&&))))
          : value_(static_cast<T&&>(value)) {}

      bool await_ready() noexcept { return true; }
      void await_suspend(coroutine_handle<>) noexcept {}
      T await_resume() noexcept(noexcept(T(FOLLY_DECLVAL(T&&)))) {
        return static_cast<T&&>(value_);
      }

     private:
      T value_;
    };
    ```


## At last

这一篇我们了解了编译器如何将协程体展开为有限状态机，以及如何从`promise_type`的视角自定义协程行为，包括`initial_suspend`、`final_suspend`和`await_transform`。结合代码示例，我们介绍了Resume continuation的简单实现，这实际上就是协程的非对称转移（Asymmetric transfer）。如果你对这个术语还不太理解也没关系，下一篇介绍对称转移（Symmetric transfer）时自然就会明白两者的区别。

## Reference

[C++ Coroutines: Understanding the promise type | Asymmetric Transfer](https://lewissbaker.github.io/2018/09/05/understanding-the-promise-type)

[C++ Coroutines: Understanding Symmetric Transfer | Asymmetric Transfer](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer)