---
layout: single
title: Deciphering C++ Coroutines, part 1
date: 2025-11-06 00:00:00 +0800
categories: 学习
tags: C++
---

每次看协程的相关介绍，总是被各种繁杂的概念所困扰，之前也尝试过梳理一次，效果也很一般。这次花了不少时间系统的学习了一下，希望能加深一下印象。其中不少的内容都来自于这个[博客](https://lewissbaker.github.io/)，但它罗列了过多的细节，缺少了一个全局视角。直到我前一阵子看到了这个[演讲](https://www.youtube.com/watch?v=J7fYddslH0Q)，才把各个概念串联起来，有了相对清晰的理解。希望对各位有所帮助。

### What is a Coroutine?

一次函数调用被分为两步：调用(Call)和返回(Return)，这里把"抛异常"也广义地归入了返回操作。调用操作会创建一个栈帧，挂起调用函数的执行，并将执行转移到被调用函数的起始位置。返回操作会将返回值传递给调用方，销毁栈帧，然后恢复调用方的执行。

协程在普通函数的基础上，额外具备以下能力：

- 挂起执行并将控制权返回给调用方
- 在被挂起后恢复执行

### What makes a function a coroutine?

如果一个函数包含以下内容，则它是一个协程：

- 一个`co_return`语句
- 一个`co_await`表达式
- 一个`co_yield`表达式

> 一个函数是否为协程从其函数签名上无法区分，这是一个实现细节。
>

在C++中提供的协程是stackless的，当被挂起时，会将控制权转交给调用方。恢复协程继续执行所需的相关信息会保存在一块动态分配的内存中，通常来说会是堆上，因此称为stackless。而之前介绍的Fiber就是stackful的，当Fiber被挂起时，当前的栈帧会被保存在栈上。协程相关信息在内存中的保存位置可以通过指定allocator来指定，不一定是堆上，但一定是动态分配的。

## Coroutines TS

C++ Coroutines TS([N4680](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf))引入了协程的基础机制，开发者可以通过这个机制与协程交互并自定义其行为。然而，Coroutines TS提供的更像是协程的底层工具，这些工具很难被直接使用。相反，基础库的编写者可以基于这些底层工具，提供更加简单易用的高级抽象，比如[cppcoro](https://github.com/lewissbaker/cppcoro)或者[folly::coro](https://github.com/facebook/folly/tree/main/folly/experimental/coro)。

比如，Coroutines TS实际上并没有定义协程的语义：

- 它没有定义如何生成返回给调用者的值。
- 它没有定义如何处理传递给`co_return`语句的返回值，或者如何处理从协程传播出去的异常。
- 它没有定义应该在哪个线程上恢复协程。

相反，它为基础库提供了一种通用机制，基础库通过实现符合特定接口的类型来定制化协程的行为。因此我们可以拓展出许多不同类型的协程，分别用于各种不同的场合。例如，你可以定义一个异步生成单个值的协程，或者一个lazily生成一系列值的协程。

Coroutines TS定义了两种接口：`Promise`和`Awaitable`。

`Promise`接口指定了用于自定义协程本身行为的方法。基础库编写者能够自定义：

- 调用协程的行为
- 协程返回时的行为（无论是通过正常方式还是通过未处理的异常）
- 协程中`co_await`或`co_yield`表达式的行为。

`Awaitable`接口指定了控制`co_await`表达式语义的方法。当我们`co_await`一个表达式时，代码将被转换为对`Awaitable`对象上的一系列方法的调用，这些方法允许它指定：

- 是否挂起当前协程
- 在挂起后执行某些逻辑以安排之后协程恢复
- 在协程恢复后执行某些逻辑以产生`co_await`表达式的结果

这一篇主要会从`co_await`的角度来介绍协程，主要关注`Awaitable`。而下一篇则主要从`Promise`的角度来介绍协程。

## Concept

### ReturnType

协程的概念非常多，为了方便理解，这里直接从一个很简单的例子开始说明。下面例子中`task`是一个协程，它的返回值是一个`folly::coro::Task<void>`。在这篇文章中我们暂时不需要关心它具体是什么，只需要知道它是一个协程的返回类型，我们把它称为`ReturnType`。

```cpp
folly::coro::Task<void> task(int arg42) {
  // ...
  co_return;
}
```

当调用协程时，都会获取到一个`ReturnType`对象。我们前面提到Coroutines TS不会指定协程返回时的行为，所以开发者通过`ReturnType`的接口来定义当协程返回时调用方能够做什么。比如`folly::coro::Task`这个`ReturnType`就提供了一个`scheduleOn`方法来指定这个协程在哪个executor执行。当我们调用其他协程基础库时，首先应当关注的就是`ReturnType`。因为基础库开发者通过`ReturnType`自定义了这个协程的行为。

```cpp

void caller() {
  auto f = task(42).scheduleOn(folly::getCPUExecutor().get()).start();
}

```

### promise_type

每个`ReturnType`中都必须一个`promise_type`，以供编译器使用。这个`promise_type`也就是前面我们所说的`Promise`接口，提供`promise_type`的方式有几种：

1. 内嵌
    - 使用using declaration，比如`folly::coro::Task`这样：

        ```cpp
        class FOLLY_NODISCARD Task {
         public:
          using promise_type = detail::TaskPromise<T>;
          // ...
        }
        ```

    - 内嵌类

        ```cpp
        struct MyTask {
            struct promise_type {
                // ...
            };
        };
        ```

2. 如果无法通过内嵌的形式提供，则可以特化`coroutine_traits`，从而指定指定其中的`promise_type`

    ```cpp
    template<>
    struct coroutine_traits<MyTask> {
        using promise_type = MyPromise;
    };
    ```


> `promise_type`的命名风格是c++标准中规定的。虽然和文章中其他组件的命名风格不一样，我们也沿用这个风格。
>

在这我们先理解下为什么它被称为promise。首先C++标准中提供了`std::future`/`std::promise`，folly提供了功能更加强大的`folly::Future`/`folly::Promise`，本质上都是一个异步的生产者消费者模型。promise是生产者，通过`promise.set_value()`或者`promise.set_exception()`来设置结果。而future是消费者，通过`future.get()`等方法来获取结果。

而在协程中，`promise_type`也是生产者，每个协程都有一个对应的`promise_type`对象，并通过`return_value()`或者`return_void()`来设置结果，又或者通过`unhandled_exception()`来获取异常。和`std::promise`不同的是，协程的`promise_type`对象不会出现在用户代码中，而是由编译器生成代码来调用它的相关接口。

`promise_type`接口指定了自定义协程本身行为的方法。基础库编写者能够自定义：当协程被调用时发生什么、当协程返回时发生什么（无论是通过正常方式还是通过未处理的异常）。

`promise_type`的主要接口如下，为了能更好理解后续的概念，这里会简单介绍一下，相关的代码我们在下一篇再展开。

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

    // ...
}
```

- `get_return_object`：用于从`promise_type`对象获取对应的`ReturnType`对象。当协程到达其第一个挂起点并且控制流返回给调用方时，调用方将通过调用`get_return_object`获得一个`ReturnType`对象。这些自定义点可以执行任意逻辑。
- `return_void`/`return_value`/`unhandled_exception`：自定义点，用于处理协程到达`co_return`语句时的行为以及异常处理方式。
- `initial_suspend`：自定义点，用于自定义协程体在执行之前的行为，比如是立即执行还是lazily启动。
- `final_suspend`：自定义点，用于协程体执行之后的行为，比如协程由谁在什么时候销毁。

实际上，`Promise`是协程代码和协程调用方之间的核心交汇点，它负责管理协程的生命周期，并在内部保存协程的执行结果。这些接口如果现在看起来一头雾水是没有关系，协程本身概念是在太多，无法管中窥豹。接下来还有很多块拼图，只有了解每个拼图块，才能了解全貌。

### Awaitable && Awaiter

下一块拼图是`Awaitable`和`Awaiter`。

`co_await`运算符是一个新的一元运算符，只能在协程的上下文中使用。支持`co_await`运算符的类型称为`Awaitable`，即文章开头所说的`Awaitable`接口。

而`Awaiter`类型是实现了三个特殊方法的类型，这些方法作为`co_await`表达式的一部分被调用：`await_ready`、`await_suspend`和`await_resume`。准确来说，编译器会把`co_await`展开为一段固定的三段式代码（下面的段落会介绍），这段代码会对调用`Awaiter`的这三个方法，进而自定义协程是否需要挂起，挂起时的行为，以及协程恢复时返回什么。因此只要实现了这三个方法的任何类型，都能被编译器正确在`co_await`中展开。

```cpp
struct suspend_always {
    // always suspend
    constexpr bool await_ready() const noexcept {
        return false;
    }
    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
};

struct suspend_never {
    // never suspend
    constexpr bool await_ready() const noexcept {
        return true;
    }
    constexpr void await_suspend(coroutine_handle<>) const noexcept {}
    constexpr void await_resume() const noexcept {}
};

co_await std::suspend_always{};
co_await std::suspend_never{};
```

那怎么理解"支持`co_await`运算符的类型称为`Awaitable`"这句话呢？我们可以认为`Awaitable`的主要作用就是告诉编译器如何获取`Awaiter`，这样`co_await`一个`Awaitable`时，就能正确被编译器展开成一段代码，从而获取到`Awaiter`。

我们可以参照编译器如何获取`Awaitable`和`Awaiter`的完整流程进行理解：

假设等待协程的`promise_type`对象是`promise`，如果`promise_type`类型有一个名为`await_transform`的成员，则首先将`<expr>`传递给对`promise.await_transform(<expr>)`的调用以获得相应的`Awaitable`对象。否则，如果`promise_type`没有`await_transform`成员，则我们直接使用计算`<expr>`的结果作为`Awaitable`对象。

> `await_transform`我们在下一篇详细介绍`promise_type`时会涉及
>

```cpp
template <typename promise_type, typename T>
decltype(auto) get_awaitable(promise_type& promise, T&& expr) {
    if constexpr (has_any_await_transform_member_v<promise_type>)
        return promise.await_transform(static_cast<T&&>(expr));
    else
        return static_cast<T&&>(expr);
}
```

然后，通过`Awaitable`对象来获取`Awaiter`，具体流程是如果`Awaitable`重载了`operator co_await()`，则对这个对象调用`operator co_await`以获得`Awaiter`对象。否则，对象`Awaitable`对象本身就用作`Awaiter`对象。

```cpp
template <typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable) {
    if constexpr (has_member_operator_co_await_v<Awaitable>)
        return static_cast<Awaitable&&>(awaitable).operator co_await();
    else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
        return operator co_await(static_cast<Awaitable&&>(awaitable));
    else
        return static_cast<Awaitable&&>(awaitable);
}
```

- 通过重载`operator co_await`的称为`Awaitable`的一个例子就是`folly::Future`，co_await返回的`FutureAwaiter`提供了Awaiter的三个接口。

    ```cpp
    template <typename T>
    inline detail::FutureAwaiter<T>
    /* implicit */ operator co_await(Future<T>&& future) noexcept {
      return detail::FutureAwaiter<T>(std::move(future));
    }

    template <typename T>
    class FutureAwaiter {
     public:
      explicit FutureAwaiter(folly::Future<T>&& future) noexcept
          : future_(std::move(future)) {}

      bool await_ready() {
        if (future_.isReady()) {
          result_ = std::move(future_.result());
          return true;
        }
        return false;
      }

      T await_resume() { return std::move(result_).value(); }

      Try<drop_unit_t<T>> await_resume_try() {
        return static_cast<Try<drop_unit_t<T>>>(std::move(result_));
      }

      FOLLY_CORO_AWAIT_SUSPEND_NONTRIVIAL_ATTRIBUTES void await_suspend(
          coro::coroutine_handle<> h) {
        // FutureAwaiter may get destroyed as soon as the callback is executed.
        // Make sure the future object doesn't get destroyed until setCallback_
        // returns.
        auto future = std::move(future_);
        future.setCallback_(
            [this, h](Executor::KeepAlive<>&&, Try<T>&& result) mutable {
              result_ = std::move(result);
              h.resume();
            });
      }

     private:
      folly::Future<T> future_;
      folly::Try<T> result_;
    };
    ```


在很多协程的介绍中并不会出现`Awaiter`，而是全部用`Awaitable`来介绍，这的确是一种简化的介绍。在上面的步骤中，编译器很多情况下就会把`Awaitable`作为`Awaiter`，此时如果`Awaitable`中没有实现对应三个接口就会报错。

实际上`Awaiter`一定是`Awaitable`，而`Awaitable`不一定是`Awaiter`。比如标准库中提供的`suspend_always`和`suspend_never`既提供了`Awaiter`的这三个接口，也就支持`co_await`运算符，所以它们既是`Awaiter`又是`Awaitable`。而一个`Awaitable`就不一定是`Awaiter`了，准确来说，`Awaitable`只需要能生成`Awaiter`即可。比如`folly::coro::Task`就只是`Awaitable`，而不是一个`Awaiter`，但它能够生成`Awaiter`。

`Awaiter`的三个接口如下：

```cpp
struct Awaiter {
    bool await_ready();

    auto await_suspend(coroutine_handle<>);
    // or specialize on the promise type: void await_suspend(coroutine_handle<promise_type>);

    auto await_resume();
}
```

- `await_ready` - 自定义点，用于控制`Awaiter`是否已完成并且可以从中获取结果
- `await_suspend` - 自定义点，定义如何等待`Awaiter`（通常是如何恢复它），将在协程即将进入挂起状态之前执行
- `await_resume` - 返回整个`co_await`表达式的结果，将在协程即将唤醒之前执行

## co_await

了解了`promise_type`、`Awaitable`和`Awaiter`之后，我们就能看看编译器是如何展开`co_await`的。我们可以把`co_await`理解为挂起协程的机会，即调用`co_await`是编译器可以挂起协程并将控制流交还给调用方的suspension point。`Awaitable`控制在这些挂起点发生什么。比如它们可以不挂起而继续执行协程，这也是为什么前面说它是挂起的机会。

展开后代码如下，其中`promise`是当前协程的`promise_type`对象：

```cpp
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready()) {
    using handle_t = std::coroutine_handle<promise_type>;

    using await_suspend_result_t =
      decltype(awaiter.await_suspend(handle_t::from_promise(promise)));

    <suspend-coroutine>

    if constexpr (std::is_void_v<await_suspend_result_t>) {
      awaiter.await_suspend(handle_t::from_promise(promise));
      <return-to-caller-or-resumer>
    } else {
      if (awaiter.await_suspend(handle_t::from_promise(promise))) {
        <return-to-caller-or-resumer>
      }
    }

    <resume-point>
  }

  return awaiter.await_resume();
}
```

首先通过前面描述流程获取`Awaitable`和`Awaiter`对象。然后调用`await_ready`判断`Awaiter`是否已经完成异步操作，如果已经完成则不需要再将协程挂起、恢复。如果没有完成，此时就会进到`<suspend-coroutine>`，编译器会生成一些代码来保存协程的当前状态以便之后恢复，会将`<resume-point>`的位置、协程的形参、以及当前寄存器中的值保存到coroutine frame中（即协程帧，一般是动态分配到堆上）。`<suspend-coroutine>`完成后，协程就已经处于挂起状态了。

在返回到调用方或者恢复方之前，编译器生成的代码还会调用`await_suspend`，这个函数是第一个可以观测到协程被挂起的地方，注意`await_suspend`传入的参数是当前协程。

返回`void`的`await_suspend()`会在调用`await_suspend()`返回时无条件地将控制流转移回协程的调用方/恢复方，而返回`bool`的版本允许awaiter对象有条件地立即恢复协程，而不返回给调用方/恢复方。返回`bool`的`await_suspend()`方法可以返回`false`以指示应立即恢复协程并继续执行，也就是把异步操作变为同步操作。

无论哪个版本，如果的确需要挂起，那么就会进入到`<return-to-caller-or-resumer>`。此时会将一个`ReturnType`对象返回给协程的调用方(上面代码中没有直接体现)，并将协程的栈帧出栈，恢复调用方的栈帧，此时控制流回到调用方，且coroutine frame仍然存在。

> 一个coroutine body会被编译器展开为若干次`co_await`，每次`co_await`都是一个挂起点，在任何一个挂起点被挂起，一个对应的`ReturnType`对象就会被返回给协程的调用方。
>

协程一旦挂起之后，协程就可以通过`coroutine_handle`进行恢复或者销毁。恢复的时机和方式取决于`awaiter.await_suspend()`的实现，比较常见的几种实现方式有：

- 当某个异步操作完成时恢复
- 线程池中的其他任务执行完毕时恢复
- 立即恢复
- 对称转移(symmetric transfer)，等介绍`folly::coro::Task`时候我们再展开

无论什么情况，当被挂起的协程被恢复时，会还原寄存器、局部变量以及参数等信息，从`<resume-point>`处继续执行，之后就会调用`await_resume`去获取`co_await`的结果。`await_resume`返回值将成为`co_await <expr>`的结果。`await_resume`方法也可能抛出异常，在这种情况下，异常会从`co_await`表达式中传播出去。如果在`await_suspend`中抛出异常，则协程将自动恢复，异常也会从`co_await`表达式中传播出去，但不会调用`await_resume`。

## Coroutine Handle

注意到展开`co_await`的代码中会调用`await_suspend()`，它有一个`coroutine_handle<promise_type>`类型参数。这个`coroutine_handle`是coroutine frame的一个句柄，可用于恢复协程的执行或销毁coroutine frame。它也可以用来访问协程的`promise`对象。需要注意的是`coroutine_handle`不是智能指针类型，也不持有coroutine frame。

`coroutine_handle`类型的主要接口如下所示：

```cpp
template <>
struct coroutine_handle<void> {
    explicit operator bool() const noexcept;

    static coroutine_handle from_address(void* a) noexcept;
    void* to_address() const noexcept;

    void operator()() const;
    void resume() const;

    void destroy();
    bool done() const;
};

template <typename Promise>
struct coroutine_handle : coroutine_handle<void> {
    Promise& promise() const noexcept;
    static coroutine_handle from_promise(Promise&) noexcept;
};
```

- `resume()`用于恢复挂起时的协程。此时会在`<resume-point>`处重新激活一个挂起的协程。
- `destroy()`方法用于销毁coroutine frame，调用任何在作用域内的变量的析构函数并释放coroutine frame使用的内存。
- `promise`和`from_promise`将在`coroutine_handle`和`promise_type`之间进行转换。

需要强调的是，`coroutine_handle`和`promise_type`一般是只有协程库才需要关心的对象，绝大多数情况下，用户代码都不会直接操作`coroutine_handle`和`promise_type`，可以把二者视为协程的内部实现细节。比如标准库中的`promise`和`from_promise`函数中都调用了内置的函数：

```cpp
static coroutine_handle from_promise(_Promise& __p) {
    coroutine_handle __self;
    __self._M_fr_ptr = __builtin_coro_promise((char*)&__p, __alignof(_Promise), true);
    return __self;
}

_Promise& promise() const {
    void* __t = __builtin_coro_promise(_M_fr_ptr, __alignof(_Promise), false);
    return *static_cast<_Promise*>(__t);
}
```

## Coroutine body

了解了`co_await`的原理后，我们就可以开始了解编译器是如何处理协程。编译器会把一个协程展开为下面三段式代码：

1. `co_await promise.initial_suspend();`
2. `coroutine body`
3. `co_await promise.final_suspend();`

展开之后的代码会涉及到前面我们所说的所有组件，整个流程在下一篇还会再详细介绍，这里只大体描述下关键步骤。

> 也可以看标准草案中的[描述](https://eel.is/c++draft/dcl.fct.def.coroutine#5)
>

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
        co_await f->promise.initial_suspend();
        <body-statements>
        // f->promise.return_void() or f->promise.return_value(...) will be called
    } catch (...) {
        f->promise.unhandled_exception();
    }
final_suspend_label:
    co_await f->promise.final_suspend();
}
```

首先，一旦编译器看到这三个关键字之一，确定这是一个协程，接着就会检查`ReturnType`。并通过`ReturnType`确定`promise_type`类型（通过内嵌或者`coroutine_traits`的形式）。

获取到`promise_type`这个类型后，编译器生成的代码会构造coroutine frame，包括coroutine frame中的`promise_type`对象。接着通过`promise_type`中的`get_return_object`方法得到`ReturnType`对象。`ReturnType`对象在协程第一次挂起或结束时返回给调用方。

之后会`co_await initial_suspend`，当`initial_suspend`被恢复时（或者是`await_suspend`返回`false`时，代表立即恢复），协程体开始执行。也就是说`promise_type`中的`initial_suspend`决定了协程的函数体什么时候开始执行。

当协程执行完时，根据返回值的不同，`return_void`或者`return_value`会被调用。如果执行过程中出现异常，则`unhandled_exception`会被调用。绝大多数的实现，都是通过这几个方法，把协程的执行结果保存在`promise_type`中。

无论哪种情况，最终都会跳转到`final_suspend_label`，这里会`co_await final_suspend`。它会决定coroutine frame由谁来销毁。通常来说`final_suspend`总是会挂起，以便协程库从协程外部对`coroutine_handle`调用`destroy()`。也就是说`promise_type`中的`final_suspend`决定了协程由谁和什么时候销毁。

> 对于`final_suspend`来说，除了挂起，还有一种实现就是对称转移，这块留到分析`folly::coro::Task`时我们再展开
>

## Cheatsheet

了解了上面这些概念后，我们终于可以开始把各个琐碎的细节拼成完整的全景了。我们再整理一下手中的拼图：

1. 编译器会展开`co_await`为一段固定格式的代码，通过`Awaitable`来控制协程是否挂起，挂起时的自定义行为，以及如何获取`co_await`这个表达式的值。
2. 编译器会将协程展开为固定三段式的代码，通过`promise_type`决定协程在启动和停止等关键时间点的行为：
    1. 通过`co_await promise.initial_suspend()`中的`Awaitable`，确定协程体执行之前的行为，比如是立即执行还是lazily启动
    2. 通过`promise`中的`return_void`/`return_value`/`unhandled_exception`，确定协程如何处理返回值和异常
    3. 通过`co_await promise.final_suspend()`中的`Awaitable`，确定协程体执行之后的行为，比如协程由谁在什么时候销毁
3. 协程最终会返回`ReturnType`，一个协程库的开发者会通过`ReturnType`中的接口，定义了用户应该如何使用这个协程。

下面我们开始尝试把这几块拼图合成一个全景。首先我们看下如何把`ReturnType`和`promise_type`拼在一起：

![figure]({{'/archive/coroutine-0.png' | prepend: site.baseurl}})

`promise_type`和`coroutine_handle`可以互相转换，而`promise_type`中有个方法能够返回`ReturnType`。这是如何做到的呢？绝大多数的`ReturnType`都是从`coroutine_handle`构造而得，并且会把`coroutine_handle`作为成员变量保存，这样`ReturnType`可以在合适的时机调用`resume()`使其继续执行：

![figure]({{'/archive/coroutine-1.png' | prepend: site.baseurl}})

> 这里只是个示例，并不是所有`ReturnType`都会提供显示的`resume`接口
>

```cpp
    Caller            Coroutine Internals
      │                       │
      ▼                       ▼
┌────────────┐          ┌───────────┐            ┌──────────────────┐
│ ReturnType │◄─────────│  Promise  │◄──────────►│ coroutine handle │
│            │  create  │           │    bind    │                  │
└────────────┘          └───────────┘            └──────────────────┘
      │                       │                            │
      │owns                   │stores                      │points to
      ▼                       ▼                            ▼
┌────────────┐          ┌───────────┐            ┌──────────────────┐
│ coroutine  │          │ result or │            │ coroutine frame  │
│   handle   │          │ exception │            │                  │
│(not owning)│          └───────────┘            └──────────────────┘
└────────────┘
```

如下图所示，我们已经能够把调用方、`ReturnType`、`coroutine_handle`、`promise_type`串在一起，那`Awaitable`呢？

![figure]({{'/archive/coroutine-2.png' | prepend: site.baseurl}})

为了方便理解，我们也把`Awaitable`和`Awaiter`都简化为`Awaitable`来介绍。

![figure]({{'/archive/coroutine-3.png' | prepend: site.baseurl}})

在上面协程体被展开的代码中，我们可以看到协程调用`Awaiter`的不同方法，确定是挂起还是执行。比如在`await_suspend`时，会传入一个挂起的协程的`coroutine_handle`，进而决定挂起时的行为。而在`coroutine_handle`的`resume`被调用时，协程会被恢复，此时会通过`await_resume`获取到`co_await`的结果，从而使得协程能继续执行。

所以各个组件之间完整的关系如下：

![figure]({{'/archive/coroutine-4.png' | prepend: site.baseurl}})

## Examples

接下来我们通过一些例子，加深对这张图的理解。

### 调用方获取Coroutine产生的数据

假设协程产生了一个42，想要传递到调用方获取。那么首先需要把这个数据保存到`Awaitable`中，也就是`TheAnswer`里。然后根据上图`Awaitable`可以和`promise_type`在`await_suspend`时进行交互，也就能保存到`promise`中。

```cpp
Coroutine f1() {
    co_await TheAnswer{42};
}

struct promise {
    // ...
    int value;
};

TheAnswer::TheAnswer(v): value_(v) {}

void TheAnswer::await_suspend(std::coroutine_handle<promise> h) {
    h.promise().value = value_;
}
```

![figure]({{'/archive/coroutine-5.png' | prepend: site.baseurl}})

最后调用方可以通过`ReturnType`获取到`coroutine_handle`，也就能拿到promise对象，读取到协程产生的数据。

```cpp
struct Coroutine {
    std::coroutine_handle<promise> handle;
    int getAnswer() {
        return handle.promise().value();
    }
};

int main() {
    Coroutien c1 = f1();
    std::cout << "The answer is " << c1.getAnswer();
}
```

![figure]({{'/archive/coroutine-6.png' | prepend: site.baseurl}})

### Coroutine获取调用方生成的数据

假设调用了某个协程，调用方在某个时间点挂起了协程，并想要传递一些数据到协程，当唤醒时，这些数据已经准备好。

一个简单例子如下所示，当`co_await OutsideAnswer`被挂起时，main会继续执行`c1.provide(42)`，其中会把数据保存到`promise`中，并唤醒协程。

```cpp
Coroutine f2() {
    int answer = co_await OutsideAnswer{};
}

void Coroutine::provide(int the_answer {
    handle.promise().value = the_answer;
    handle.resume();
}

int main() {
    Coroutine c1 = f2();
    c1.provide(42);
}
```

而协程获取这个数据也很简单，当被唤醒时，此时数据已经被保存到`promise`了，通过`await_resume`获取即可。

```cpp
struct OutsideAnswer {
    bool await_ready() { return false; }
    void await_suspend(std::coroutine_handle<promise> h) {
        handle = h;
    }
    int await_resume() {
        return handle.promise().value();
    }

    std::coroutine_handle<promise> handle;
};
```

![figure]({{'/archive/coroutine-7.png' | prepend: site.baseurl}})

## At last

这是协程系列的第一篇， 我们主要介绍了协程的相关概念，包括`ReturnType`、`promise_type`、`Awaitable`和`Awaiter`等。接着介绍了编译器是如何处理`co_await`的，以及在`co_await`的基础上时如何处理协程函数体的。由于协程中概念错综复杂，我们也通过一张图把各个概念串联起来。下一篇我们会着重介绍如何通过`promise_type`来自定义协程的行为。

## Reference

[Deciphering C++ Coroutines - A Diagrammatic Coroutine Cheat Sheet - Andreas Weis - CppCon 2022](https://www.youtube.com/watch?v=J7fYddslH0Q)

[Asymmetric Transfer | Some thoughts on programming, C++ and other things.](https://lewissbaker.github.io/)