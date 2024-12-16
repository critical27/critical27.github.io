---
layout: single
title: Coroutine internals
date: 2024-12-09 00:00:00 +0800
categories: 学习
tags: folly
---

上次研究了Fiber，这次结合C++ Coroutines TS([N4680](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2017/n4680.pdf))和`folly::coro::Baton`，看看Coroutine~

## Coroutines TS

- 三个新关键字：`co_await`, `co_yield`, `co_return`
- 一些新类型：
    - `coroutine_handle`
    - `coroutine_traits`
    - `suspend_always`
    - `suspend_never`
- 协程的基础机制：基础库的开发者可以通过这个机制与协程交互并自定义其行为
- 基于协程的编写异步代码的方式

然而，Coroutines TS 提供的更像是协程的底层工具，这些工具很难被直接使用。相反，基础库的编写者可以基于这些底层工具，提供更加简单易用的高级抽象，比如 [cppcoro](https://github.com/lewissbaker/cppcoro) 或者 [folly::coro](https://github.com/facebook/folly/tree/main/folly/experimental/coro)。

比如，Coroutines TS 实际上并没有定义协程的语义：

- 它没有定义如何生成返回给调用者的值。
- 它没有定义如何处理传递给`co_return`语句的返回值，或者如何处理从协程传播出去的异常。
- 它没有定义应该在哪个线程上恢复协程。

相反，它为基础库提供了一种通用机制，基础库通过实现符合特定接口的类型来定制化协程的行为。因此我们可以拓展出许多不同类型的协程，分别用于各种不同的场合。例如，你可以定义一个异步生成单个值的协程，或者一个 lazily 生成一系列值的协程。

Coroutines TS 定义了两种接口：`Promise`和`Awaitable`。

`Promise`接口可以自定义协程本身的行为。基础库编写者能够自定义：

- 调用协程的行为
- 协程返回时的行为（无论是通过正常方式还是通过未处理的异常）
- 协程中任何`co_await`或`co_yield`表达式的行为。

`Awaitable`接口则指定了`co_await`一个表达式时的语义。当我们`co_await`一个表达式时，代码将被转换为对`Awaitable`对象上的一系列方法的调用，这些方法允许它指定：

- 是否暂停当前协程
- 在挂起后执行某些逻辑以安排之后协程恢复
- 在协程恢复后执行某些逻辑以产生`co_await`表达式的结果

这一篇我们主要先关注`Awaitable`接口。

## Suspend a coroutine

Coroutines TS引入了一个新的单目运算符`co_await`。`co_await`只能在协程使用，这句话有点像废话。因为根据定义，任何包含`co_await`运算符的函数体都将被编译为协程。`co_await <expression>`时会涉及两个类型：`Awaitable` 和 `Awaiter`。

- 支持`co_await`运算符的类型称为`Awaitable`，有一些介绍中也称为 Awaitable concept。
- `Awaiter`则是任何实现了以下三个特殊接口的类型，在`co_await <expr>`时会调用这几个特殊接口：
    - `await_ready`
    - `await_suspend`
    - `await_resume`

我们看下编译器是如何处理`co_await`，并最终将协程挂起的：

- 首先编译器会获取`Awaitable`对象，这里不展开过多介绍，详见Coroutines TS 5.3.8(3)。

```cpp
template<typename P, typename T>
decltype(auto) get_awaitable(P& promise, T&& expr)
{
  if constexpr (has_any_await_transform_member_v<P>)
    return promise.await_transform(static_cast<T&&>(expr));
  else
    return static_cast<T&&>(expr);
}
```

- 根据`Awaitable`对象，获取`Awaiter`对象。

```cpp
template<typename Awaitable>
decltype(auto) get_awaiter(Awaitable&& awaitable)
{
  if constexpr (has_member_operator_co_await_v<Awaitable>)
    return static_cast<Awaitable&&>(awaitable).operator co_await();
  else if constexpr (has_non_member_operator_co_await_v<Awaitable&&>)
    return operator co_await(static_cast<Awaitable&&>(awaitable));
  else
    return static_cast<Awaitable&&>(awaitable);
}
```

- 等待`Awaiter`。这一步是`co_await`的核心流程，其中会调用`Awaiter`实现的三个特殊方法。

整个`co_await <expr>`会被转换为类似下面的代码：

```cpp
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready())
  {
    using handle_t = std::coroutine_handle<P>;

    using await_suspend_result_t =
      decltype(awaiter.await_suspend(handle_t::from_promise(p)));

    <suspend-coroutine>

    if constexpr (std::is_void_v<await_suspend_result_t>)
    {
      awaiter.await_suspend(handle_t::from_promise(p));
      <return-to-caller-or-resumer>
    }
    else
    {
      static_assert(
         std::is_same_v<await_suspend_result_t, bool>,
         "await_suspend() must return 'void' or 'bool'.");

      if (awaiter.await_suspend(handle_t::from_promise(p)))
      {
        <return-to-caller-or-resumer>
      }
    }

    <resume-point>
  }

  return awaiter.await_resume();
}
```

首先是通过`await_ready`判断`Awaiter`是否已经完成异步操作，如果已经完成则不需要再将协程挂起、恢复。

如果没有完成，此时就会进到`<suspend-coroutine>`，编译器会生成一些代码来保存协程的当前状态并准备恢复。会将`<resume-point>`的位置、协程的形参、以及当前寄存器中的值保存到协程帧中（即coroutine frame，一般是经过动态分配到堆上）。此时协程就已经处于挂起状态了。

在返回到调用方或者恢复方之前，编译器生成的代码还会调用`await_suspend`，这个函数是第一个可以观测到协程被挂起的地方。`await_suspend`负责在操作完成后的某个时间点安排协程恢复或销毁。如果`await_suspend`返回`false`，则会将协程在当前线程上立即恢复。

> 协程一旦挂起之后，协程就可以通过`coroutine_handle`进行恢复或者销毁（具体方式后面`coroutine_handle`的部分会介绍）。
> 

`await_suspend`有两种返回值类型：当对`await_suspend`调用返回时，`await_suspend`返回`void`的版本会无条件地将执行返还给协程的调用方，而返回`bool`的版本允许`Awaiter`对象有条件地立即恢复协程，而无需返回给调用方或者恢复方。比如`co_await`的这个异步操作有时候可以同步完成时，如果在`co_await`时已经完成了，那么`await_suspend`就可以返回`false`，进而使协程立即恢复并继续执行。

如果的确需要等待，那么就会进入到`<return-to-caller-or-resumer>`。此时会将执行返还给协程的调用方，具体返还的形式就是将协程的栈帧pop，并恢复调用方的栈帧，注意此时coroutine frame是仍然存在的。

当被挂起的协程被恢复时，会被恢复到`<resume-point>`处，之后就会调用`await_resume`去获取`co_await`的结果。`await_resume`返回值将成为`co_await <expr>`的结果。`await_resume`方法也可能抛出异常，在这种情况下，异常会从`co_await`表达式中传播出去。另外，如果异常从 `await_suspend`中抛出异常，则协程将自动恢复，异常也会从`co_await` 表达式中传播出去，但不会调用`await_resume`。

上面的流程就完成了协程的挂起，那么我们如何恢复这个协程呢？

## Resume a coroutine

细心的你在上面流程中可能也注意到，在`await_suspend`时需要传入一个`coroutine_handle`，它主要作用就是用来恢复或者销毁协程。`coroutine_handle`内部持有一个coroutine frame的指针（注意它不是`coroutine frame`的所有者）。

`coroutine_handle`的主要接口如下：

```cpp
template <>
struct coroutine_handle<void> {
    coroutine_handle() noexcept = default;
    coroutine_handle(nullptr_t) noexcept;
    coroutine_handle& operator=(nullptr_t) noexcept;
    
    explicit operator bool() const noexcept;

    static coroutine_handle from_address(void* a) noexcept;
    void* to_address() const noexcept;

    void operator()() const;
    void resume() const;

    void destroy();
    bool done() const;
};
```

对于不需要返回值的协程，主要提供了几类接口：

- 构造
- `operator bool()`用于检查`coroutine_handle`是否关联了一个协程
- `from_address`和`to_address`用于将`coroutine_handle`和一个函数指针之间进行转换，主要是为了兼容C-style API。
- `operator()`和`resume`用于恢复一个协程继续执行。调用这个接口之后，协程会继续执行，直到协程再次挂起并运行到`<return-to-caller-or-resumer>`。
- `destroy`用于调用方在协程任何一次挂起之后，可以直接销毁协程。内部实际是销毁协程对应的coroutine frame。
- `done`用于检查协程是否执行完

而对于需要返回值的协程，额外提供了两个接口：

```cpp
template <typename Promise>
struct coroutine_handle : coroutine_handle<void> {
    Promise& promise() const noexcept;
    static coroutine_handle from_promise(Promise&) noexcept;
};
```

事实上，这其中有一部分接口，是一般的开发人员不会调用也不不应该调用的：

- `destroy`
- `promise`
- `from_promise`

这些接口一般都是基础库的编写者才需要用到的接口，我们在绝大多数情况下，应该把这些接口视为协程的内部实现。

## folly::coro::Baton

下面我们就看看一个folly::coro协程库是怎么利用C++ Coroutines TS提供的底层工具，构建一个基于协程的`Baton`。关于`Baton`的用法直接参照下面的注释：

```cpp
/// A baton is a synchronisation primitive for coroutines that allows a
/// coroutine to co_await the baton and suspend until the baton is posted by
/// some thread via a call to .post().
///
/// This primitive is typically used in the construction of larger library types
/// rather than directly in user code.
///
/// As a primitive, this is not cancellation-aware.
///
/// The Baton supports being awaited by multiple coroutines at a time. If the
/// baton is not ready at the time it is awaited then an awaiting coroutine
/// suspends. All suspended coroutines waiting for the baton to be posted will
/// be resumed when some thread next calls .post().
///
/// Example usage:
///
///   folly::coro::Baton baton;
///   std::string sharedValue;
///
///   folly::coro::Task<void> consumer()
///   {
///     // Wait until the baton is posted.
///     co_await baton;
///
///     // Now safe to read shared state.
///     std::cout << sharedValue << std::cout;
///   }
///
///   void producer()
///   {
///     // Write to shared state
///     sharedValue = "some result";
///
///     // Publish the value by 'posting' the baton.
///     // This will resume the consumer if it was currently suspended.
///     baton.post();
///   }
```

简单来说就是个生产者消费者模型，消费者通过`co_await`等待`Baton`变成ready状态，如果`Baton`没有ready，则会挂起协程进行等待。直到生产者调用`post`方法，所有被挂起的协程会继续执行。

为了能够完成上述功能，`Baton`内部需要维护一个状态，用来表明这个`Baton`是否已经ready。当`co_await`一个`Baton`时：

- 如果`Baton`没有ready，那么协程就会被挂起，直到`post`被调用。（也就是上面介绍`await_suspend`时提到的有条件的挂起）
- 如果`Baton`已经ready，协程就会直接继续执行

`Baton`提供的主要接口如下所示：

```cpp
class Baton {
 public:
  class WaitOperation;

  /// Initialise the Baton to either the signalled or non-signalled state.
  explicit Baton(bool initiallySignalled = false) noexcept;

  ~Baton();

  bool ready() const noexcept;

  [[nodiscard]] WaitOperation operator co_await() const noexcept {
    return Baton::WaitOperation{*this};
  }

  void post() noexcept;

  void reset() noexcept;

  class WaitOperation；

 private:
  // this  - Baton is in the signalled/posted state.
  // other - Baton is not signalled/posted and this is a pointer to the head
  //         of a potentially empty linked-list of Awaiter nodes that were
  //         waiting for the baton to become signalled.
  mutable std::atomic<void*> state_;
};
```

大多数接口在上面已经提到，没有涉及到的部分包括：

- `reset`是把一个已经ready的`Baton`重置回没有ready的状态
- `std::atomic<void*> state_`的作用就是用来表明Baton是否ready：
    - 当`state_`值等于`this`时，代表已经ready。
    - 当`state_`不等于`this`时，代表没有ready。

从原理上来说，所有正在等待的协程指针，会被维护在一个链表中，当`post`被调用时，就会遍历这个链表并继续执行这些协程。当`state_`不等于`this`时，`state_`中保存的就是链表头，之所以一定用`this`来代表这个ready这个特殊状态，也是因为正在等待的协程链表中的协程指针，肯定不会和`this`相同。

`WaitOperation`就是当`co_await`这个`Baton`时，会返回的`Awaiter`对象。它需要完成的工作有以下几点：

- 它需要知道正在等待的是哪个Baton
- 其次它需要维护所有正在等待这个Baton的协程，并在post调用之后恢复执行这些协程
- 此外它需要保存正在等待这个Baton的协程的coroutine_handle，从而完成协程的恢复
- 最后它既然是Awaiter对象，也就需要实现`await_ready`,`await_suspend`和`await_resume`。

为了实现上述功能，它的实现如下所示：

```cpp
  class WaitOperation {
   public:
    explicit WaitOperation(const Baton& baton) noexcept : baton_(baton) {}

    bool await_ready() const noexcept;

    bool await_suspend(coroutine_handle<> awaitingCoroutine) noexcept;

    void await_resume() noexcept {}

   protected:
    friend class Baton;

    const Baton& baton_;
    coroutine_handle<> awaitingCoroutine_;
    WaitOperation* next_;
  };
```

> 根据我们要实现什么样的协程组件，我们就需要通过这几个接口定制`awaiter`的行为。对于`await_resume`，我们不需要`co_await`一个`Baton`时得到一个返回值，所以不需要任何逻辑。
> 

当我们`co_await`一个`Baton`时，我们就会获取到一个Awaiter对象，也就是`WaitOperation`。按照上面介绍编译器处理`co_wait`时的流程，之后就会调用它的`await_ready`检查是否已经完成异步操作，即`Baton`是否ready，也就是检查`Baton`中的`m_state`是否是`this`：

```cpp
bool Baton::WaitOperation::await_ready() const noexcept {
  return baton_.ready();
}

inline bool Baton::ready() const noexcept {
  return state_.load(std::memory_order_acquire) == static_cast<const void*>(this);
}
```

如果不是ready状态，接下来就会调用`await_suspend`，此处协程已经处于挂起状态，通过`await_suspend`的返回值，可以决定是执行返还给调用方，还是立即恢复协程。

> 这里解释下为什么明明刚通过`await_ready`检查了`Baton`没有ready，并挂起了协程，为什么又可能会立即恢复协程。Baton是多消费者单生产者模型，在这个协程调用`co_await`期间，准确说是`await_ready`和`await_suspend`期间，`Baton`的状态可能已经被修改为ready，此时这个协程就不需要再等待，也就是所谓有条件地立即恢复协程。
> 

```cpp
bool Baton::WaitOperation::await_suspend(coroutine_handle<> awaitingCoroutine) noexcept {
  awaitingCoroutine_ = awaitingCoroutine;
  return baton_.waitImpl(this);
}

bool Baton::waitImpl(WaitOperation* awaiter) const noexcept {
  // Try to push the awaiter onto the front of the queue of waiters.
  const auto signalledState = static_cast<const void*>(this);
  void* oldValue = state_.load(std::memory_order_acquire);
  do {
    if (oldValue == signalledState) {
      // Already in the signalled state, don't enqueue it.
      return false;
    }
    awaiter->next_ = static_cast<WaitOperation*>(oldValue);
  } while (!folly::atomic_compare_exchange_weak_explicit(
      &state_,
      &oldValue,
      awaiter,
      std::memory_order_release,
      std::memory_order_acquire));
  return true;
}
```

这部分大致逻辑是这样的：

1. 保存当前的`coroutine_handle`，以便后续可以恢复这个协程
2. 需要将自身加入到等待的协程列表，也就是这个无锁链表中去。当前协程的指针是通过`awaiter`保存的，`waitImpl`就是要将`awaiter`通过头插法插入到链表头中。
    1. 首先获取当前`state_`。
    2. 检查`Baton`的状态是否已经是ready，即检查`state_`是否为`Baton`的`this`指针。
    3. 如果已经ready，直接返回`false`，代表当前协程不再需要等待，可以立即恢复执行，即执行到`<resume-point>`。
    4. 如果没有ready，则将当前`awaiter`的`next`指针设置成`state_`，即链表头。
    5. 然后尝试CAS将链表头`state_`替换为`awaiter`。
    6. 如果CAS成功代表已经成功加入链表，返回`true`，当前协程就会执行到`<return-to-caller-or-resumer>`，此后返回到`co_await`调用方，等待唤醒。
    7. 如果CAS失败，代表存在并发`co_await`这个`Baton`，需要重新走上述流程

同理，`reset`逻辑就是将`state_`重新设置为`nullptr`，即没有ready的状态。

```cpp
inline void Baton::reset() noexcept {
  // Transition from 'signalled' (ie. 'this') to not-signalled (ie. nullptr).
  void* oldState = this;
  (void)state_.compare_exchange_strong(
      oldState, nullptr, std::memory_order_acq_rel, std::memory_order_relaxed);
}
```

而`post`时候逻辑就是依次遍历这个链表，通过`WaitOperation`中保存的`coroutine_handle`恢复协程的执行。

```cpp
void Baton::post() noexcept {
  void* const signalledState = static_cast<void*>(this);
  void* oldValue = state_.exchange(signalledState, std::memory_order_acq_rel);
  if (oldValue != signalledState) {
    // We are the first thread to set the state to signalled and there is
    // a waiting coroutine. We are responsible for resuming it.
    WaitOperation* awaiter = static_cast<WaitOperation*>(oldValue);
    while (awaiter != nullptr) {
      std::exchange(awaiter, awaiter->next_)->awaitingCoroutine_.resume();
    }
  }
}
```

## Reference

[C++ Coroutines: Understanding operator co_await | Asymmetric Transfer (lewissbaker.github.io)](https://lewissbaker.github.io/2017/11/17/understanding-operator-co-await)

[Coroutine Theory | Asymmetric Transfer (lewissbaker.github.io)](https://lewissbaker.github.io/2017/09/25/coroutine-theory)

[CppCon 2016: James McNellis “Introduction to C++ Coroutines" (youtube.com)](https://www.youtube.com/watch?v=ZTqHjjm86Bw)