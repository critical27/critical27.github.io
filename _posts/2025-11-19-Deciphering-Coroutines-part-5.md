---
layout: single
title: Deciphering C++ Coroutines, part 5
date: 2025-11-19 00:00:00 +0800
categories: 学习
tags: C++ folly
---

coroutine简化了异步代码的编写难度，但在debug时，却无法还原协程之间的异步调用链。这一篇我们研究下`folly::AsyncStackFrame`是如何记录协程之间的调用关系的。注意本文中所说的“协程“，如无特殊说明，都是指代`folly::coro::Task`。

## Background

我们在上一篇其实有介绍，通过C++标准的协程底层机制，能够建立起协程之间的调用关系，从而实现一个完整的异步任务。比如在`callee`的promise对象中记录`caller`的`coroutine_handle`，进而使得`callee`在执行完成后，能够恢复`caller`继续执行。

![figure]({{'/archive/coroutine-17.png' | prepend: site.baseurl}})

第一种想法是：既然我们可以建立起`caller`和`callee`的调用关系，只要能获取到`callee`的`coroutine_handle`，也就能获取到其promise中的任意对象， 包括`caller`的`coroutine_handle`。也就能将两个`coroutine_handle`连接在一起，从而建立起协程的调用栈。

```cpp
template<typename T>
struct TaskPromise {
  std::coroutine_handle<void> continuation;
  Try<T> result;
};

struct __coro_frame {
    void (*resume_fn)(void*);
    void (*destroy_fn)(void*);
    std::__coroutine_traits_impl<ReturnType>::promise_type __promise;
    int __suspend_index;
    bool __initial_await_suspend_called;
    // ...
};
```

然而，这个办法在当`folly::coro::Task<T>`中的`T`超过某个对齐大小的阈值之后，编译器就会在`resume_fn`和`destroy_fn`这两个指针之后，插入若干字节的padding。这时我们就无法再获取到协程帧中的promise对象以及其中的成员变量。

第二种想法是：获取到某个协程的`coroutine_handle`后，直接去查看其`resume_fn`的实现，进而推断出promise的位置，从而获取到其中的成员变量。然而这种方法要么是需要调试信息，同时不同编译器生成的底层汇编也不同，很难以统一形式处理。

第三种想法是：既然我们可以建立起`caller`和`callee`的调用关系，自然也就能在把协程之间的调用关系，以某种形式保存在协程的promise的任意成员变量中。和第一种的区别在于，它并不会直接尝试从`coroutine_handle`读取promise对象中的内容。事实上，folly的`AsyncStackFrame`就是作为一个成员变量，保存在协程的promise中。它和另一个类`AsyncStackRoot`一起，组成了一个链表，记录了协程的调用关系。通过遍历这个链表，也就能恢复出完整的调用栈。

在具体介绍原理之前，不妨看个例子：

```cpp
void baz() {
    // ...
}

folly::coro::Task<void> bar() {
    co_return baz();
}

folly::coro::Task<void> foo() {
    co_await bar();
}

int main() {
    folly::CPUThreadPoolExecutor executor{1};
    folly::coro::blockingWait(foo().scheduleOn(&executor));
    return 0;
}
```

如果我们去gdb里面在`baz`函数加上断点，看到的调用栈可能是下面这样的：

```cpp
#0  baz () at /home/doodle.wang/source/folly/folly/experimental/coro/test/BlockingWaitTest.cpp:375
#1  0x00005555557e067f in bar(_Z3barv.Frame *) (frame_ptr=0x7ffff0005210) at /home/doodle.wang/source/folly/folly/experimental/coro/test/BlockingWaitTest.cpp:378
#2  0x00005555557ecd8d in std::__n4861::coroutine_handle<void>::resume (this=0x7ffff6ff5ec8) at /usr/include/c++/13/coroutine:135
...
#5  0x0000555555813466 in folly::coro::TaskWithExecutor<void>::Awaiter::await_suspend<folly::coro::detail::BlockingWaitPromise<void> >(std::__n4861::coroutine_handle<folly::coro::detail::BlockingWaitPromise<void> >)::{lambda()#1}::operator()() (__closure=0x7ffff6ff60e0)
    at /home/doodle.wang/source/folly/folly/experimental/coro/Task.h:526
...
#10 folly::ThreadPoolExecutor::runTask (this=0x555555b2b750, thread=..., task=...) at /home/doodle.wang/source/folly/folly/executors/ThreadPoolExecutor.cpp:100
#11 0x000055555585de53 in folly::CPUThreadPoolExecutor::threadRun (this=0x555555b2b750, thread=...)
    at /home/doodle.wang/source/folly/folly/executors/CPUThreadPoolExecutor.cpp:326
...
#22 0x000055555588a0f8 in std::thread::_Invoker<std::tuple<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}> >::operator()() (
    this=0x7ffff00083a0) at /usr/include/c++/13/bits/std_thread.h:299
#23 0x0000555555880fa0 in std::thread::_State_impl<std::thread::_Invoker<std::tuple<folly::NamedThreadFactory::newThread(folly::Function<void ()>&&)::{lambda()#1}> > >::_M_run() (this=0x7ffff0008390) at /usr/include/c++/13/bits/std_thread.h:244
...
```

注意到`foo`是没有出现在调用栈中的。虽然是`main`调用了`foo`，但`foo`是被调度在一个`Executor`上的执行的，因此只能看到执行`foo`的`Executor`的相关调用栈。我们更想要的是一个完整的调用栈，而不区分是不是协程：

```cpp
- baz
- bar
- foo
- main
```

我们不妨分析下`AsyncStackFrame`要还原出完整调用栈会遇到哪些情况，即包括：

- [x]  普通函数调用普通函数
- [ ]  普通函数调用协程
- [ ]  协程调用协程
- [x]  协程调用普通函数

普通函数调用普通函数是通过对`%rbp`和`caller`的返回地址的压栈出栈操作完成的，大致原理在上一篇我们介绍过，不熟悉的可以回顾下。而协程调用普通函数和普通函数调用普通函数本质上没有什么区别，因此这两种情况都不再赘述。

> 实际上，`folly::AsyncStackFrame`不仅仅是支持协程，也支持追踪`folly::Future`这样的回调形式的异步调用栈，但鉴于我们这一系列都是分析协程，所以也都以协程为例。剩下篇幅中普通调用栈和同步栈是同义词，异步调用栈和协程调用栈是同义词。
>

## AsyncStackFrame

普通函数的调用关系，是通过`%rbp`和返回地址串联起来，最终形成了普通调用栈，也称为同步栈。同理，为了还原协程之间的调度栈，我们需要记录下来`caller`被挂起在什么位置，以便`callee`执行完成之后恢复`caller`继续执行，本质上和同步栈是一样的。有了这些返回地址后，加上二进制的调试信息，就能把指令地址映射回对应的函数名，最终定位到源代码的文件和行号。

为了实现这个目标，有两个核心问题需要解决：

- `callee`协程如何获知`caller`协程的返回地址，并保存在`AsyncStackFrame`中
- 如何将`caller`和`callee`的`AsyncStackFrame`串联起来

这里直接展示下`AsyncStackFrame`的数据结构：

```cpp
struct AsyncStackFrame {
    AsyncStackFrame* parentFrame = nullptr;
    void* instructionPointer = nullptr;
    AsyncStackRoot* stackRoot = nullptr;
};
```

三个成员变量主要用于保存以下信息：

1. `parentFrame`：`AsyncStackFrame`单链表，在协程被挂起时会更新，记录当前协程是从被哪个协程被调用的，形成异步调用链。
2. `instructionPointer`：在协程被挂起时会更新，记录当前协程下次恢复时要执行的代码地址
3. `stackRoot`：记录当前协程属于哪个`EventLoop`（也可以理解为异步操作），只有正在执行的协程中`stackRoot`才非空。主要作用是连接普通调用栈和异步调用栈。

然后，我们分为协程调用协程和普通函数调用协程两种情况，分别介绍其具体原理。

### **Obtaining the return-address of an async-stack frame**

对于第一个问题”`callee`协程如何获知`caller`协程的返回地址”，理论上只要我们能获取到某个协程帧，就能根据其中的`suspend_index`，得知当前协程挂起在哪个位置。即当协程恢复时，状态机函数`resume_fn`要从哪个地址开始继续执行。这块内容前期篇介绍过，不再赘述。

```cpp
struct __coro_frame {
    void (*resume_fn)(void*);
    void (*destroy_fn)(void*);
    std::__coroutine_traits_impl<ReturnType>::promise_type __promise;
    int __suspend_index;
    bool __initial_await_suspend_called;
    // ...
};
```

然而，编译器可能会根据协程内部`co_await`数量的多少，会把状态机函数中根据`suspend_index`进行跳转的汇编代码处理成[不同的形式](https://godbolt.org/z/994b4M)：

1. 对于比较小的协程，会直接通过`cmp`指令比较`suspend_index`
2. 而比较大的协程，则会直接使用jump table进行跳转（本质上就是通过jump table优化`switch`语句性能）

而不同编译器处理的方式差别更大，故而这种方式虽然可行，但是过于复杂。

而`AsyncStackFrame`的处理方式非常简单：我们之前介绍对称转移的时候说过，在`Awaitable`的`await_suspend`方法中可以建立起`caller`协程和`callee`协程的调用关系，并且可以将`caller`的相关信息传递给`callee`。那么，只要能以某种获取到`caller`协程的的返回地址（也就是`caller`调用`co_await callee()`的下一条指令地址），也就能传递给`callee`，达成建立协程调用栈的目的。

而具体获取到返回地址的方式就是编译器的内置函数`__builtin_return_address`，他可以传入一个整数`n`，从而获取第`n`个stack frame的返回地址。因此`caller`的返回地址可以通过如下形式进行传递：

```cpp
template<typename T>
auto Task::Awaiter::await_suspend(std::coroutine_handle<> continuation) {
    coro_.promise().getAsyncFrame().instructionPointer = __builtin_return_address(0);
    // ...
    return coro_;
}
```

> PS：如果`await_suspend`被inline了，那么`__builtin_return_address`就会获取到错误的返回地址，因此一般需要禁止inline这个函数。
>

### **Hooking up the stack-frames in a chain**

至于如何”将`caller`和`callee`的`AsyncStackFrame`串联起来”，我们可以如法炮制。也在`Awaiter::await_suspend`中更新`callee`的`parentFrame`指针。通过这样的形式，形成了一个到`callee -> caller -> ...`的链表，当我们通过打断点等类似手段，发现CPU正在执行任何一个协程时，就能沿着这个链表，恢复整个协程之间的调用关系。

具体形式如下：

```cpp
template<typename Promise>
auto Task<T>::Awaiter::await_suspend(std::coroutine_handle<Promise> continuation) {
    auto& callerFrame = continuation.promise().getAsyncFrame();
    auto& calleeFrame = coro_.promise().getAsyncFrame();
    calleeFrame.parentFrame = &callerFrame;
    calleeFrame.instructionPointer = __builtin_return_address(0);
    // ...
    return coro_;
}
```

### **Finding the top async stack-frame**

前面两步中，我们在`Awaitable`的`await_suspend`中，更新了`parentFrame`和`instructionPointer`两个字段，从而建立起了协程之间的调用关系。但此处还遗留了一个问题：如何获取到当前正在执行的协程的`AsyncStackFrame`。

> 对于普通函数的stack frame不存在这个问题，只需要读取`%rbp`就知道topmost frame。
>

具体解决办法如下：在`folly::coro::Task`对称转移部分代码中，我们知道要将控制流交给哪个协程，自然也能把正在执行的协程的这个信息记录下来。只不过，考虑到我们不会直接读取promise中的`AsyncStackFrame`，因此是不直接存在其中的。实际的做法是，将当前正在执行的协程的`AsyncStackFrame`，保存在一个`thread_local`变量中，它的类型是`AsyncStackRoot`。数据结构如下：

```cpp
struct AsyncStackRoot {
    // Pointer to the currently-active AsyncStackFrame for this event
    // loop or callback invocation. May be null if this event loop is
    // not currently executing any async operations.
    std::atomic<AsyncStackFrame*> topFrame{nullptr};
    // Pointer to the next event loop context lower on the current
    // thread's stack.
    AsyncStackRoot* nextRoot = nullptr;
    // Pointer to the stack-frame and return-address of the function
    // call that registered this AsyncStackRoot on the current thread.
    // This is generally the stack-frame responsible for executing async
    // callbacks (typically an event-loop).
    void* stackFramePtr = nullptr;
    void* returnAddress = nullptr;
};
```

- `topFrame`：指向当前正在执行的协程`AsyncStackFrame`。
- `nextRoot`：用于串联多个`EventLoop`，从而形成AsyncStackRoot的链表，即`EventLoop A`创建了`EventLoop B`，则`B`的`AsyncStackRoot`的`nextRoot`指向`A`的`AsyncStackRoot`。
- `stackFramePtr` & `returnAddress`：都用于记录普通调用栈的信息。`stackFramePtr`记录注册这个`AsyncStackRoot`时的同步栈帧(即当时的`%rbp`)，`returnAddress`用于记录对应返回地址。

对于正在执行的协程，`AsyncStackRoot`中的`topFrame`和`AsyncStackFrame`中的`stackRoot`互相指向对方：

- `AsyncStackRoot`中的`topFrame`指向当前正在执行的协程
- `AsyncStackFrame`中的`stackRoot`指向当前线程正在使用的`AsyncStackRoot`

而对于被挂起的协程，`AsyncStackFrame`中的`stackRoot`为空指针。

下面通过一个具体例子具体介绍`AsyncStackRoot`的作用：

```cpp
void compute_something() {
    // ...
}

folly::coro::Task<void> coro1() {
    compute_something();
    co_return;
}

void func1() {
    folly::coro::blockingWait(coro1());
}

folly::coro::Task<void> coro2() {
    func1();
    co_return;
}

void main() {
    folly::coro::blockingWait(coro2());
}
```

和文章一开始的例子不同的是，这个例子并没有指定协程在哪个`Executor`上执行。`main`启动了嵌套的两个协程`coro2`和`coro1`，两个协程实际上是在同一个线程上执行的。每次`folly::coro::blockingWait`时，都会创建一个事件循环，不断推动异步任务执行直至完成。

```cpp
FOLLY_NOINLINE T getVia(folly::DrivableExecutor *executor,
                        folly::AsyncStackFrame &parentFrame) && {
    folly::Try<detail::lift_lvalue_reference_t<T>> result;
    auto &promise = coro_.promise();
    promise.setTry(&result);

    // ...

    executor->add([coro = coro_, rctx = RequestContext::saveContext()]() mutable {
        RequestContextScopeGuard guard{std::move(rctx)};
        folly::resumeCoroutineWithNewAsyncStackRoot(coro);
    });
    while (!promise.done()) {  // <- EventLoop
        executor->drive();
    }
    return std::move(result).value();
}
```

每个线程可能会启动多个`EventLoop`，每个`EventLoop`负责推动多个协程执行，在一个线程上，同一时刻只有一个协程正在执行，其余协程都处于挂起状态。为了得到正在执行的协程的`AsyncStackFrame`，我们需要将当前线程正在通过哪个`EventLoop`执行哪个协程以某种形式记录下来。

具体的办法是：每当启动一个`EventLoop`，就会创建一个`AsyncStackRoot`。在协程切换时，也就是`caller`执行`co_await callee`，以及`callee`执行`co_await promise.final_suspend`时，把当前线程正在执行的协程信息保存到`AsyncStackRoot`中。示意代码如下：

```cpp
// caller -> callee
template <typename T>
auto Task<T>::Awaiter::await_suspend_impl(std::coroutine_handle<> continuation,
                                          AsyncStackFrame &callerFrame) {
    auto &calleeFrame = coro_.promise().getAsyncFrame();
    calleeFrame.parentFrame = &callerFrame;
    calleeFrame.instructionPointer = __builtin_return_address(0);

    auto *stackRoot = callerFrame.stackRoot;
    stackRoot->topFrame = &calleeFrame;
    calleeFrame.stackRoot = stackRoot;
    callerFrame.stackRoot = nullptr;

    // ...
    return coro_;
}

// calee -> caller
template <typename T>
auto TaskPromise<T>::FinalAwaiter::await_suspend(std::coroutine_handle<Promise> h) noexcept {
    auto &promise = h.promise();

    AsyncStackFrame &calleeFrame = promise.getAsyncFrame();
    AsyncStackFrame *callerFrame = calleeFrame.parentFrame;
    AsyncStackRoot *stackRoot = calleeFrame.stackRoot;

    stackRoot->topFrame = callerFrame;
    callerFrame->stackRoot = stackRoot;
    callee->stackRoot = nullptr;

    // ...
    return promise.continuation;
}
```

可以看到，无论是在`caller -> callee`还是`callee -> caller`的对称转移处理过程中，都会：

- 将`AsyncStackRoot`中的`topFrame`指向即将正在执行的协程
- 将即将执行的协程的`AsyncStackFrame`中`stackRoot`置为非空，即将被挂起的协程的`AsyncStackFrame`中`stackRoot`置为空指针。

到这里我们还剩一个问题没有解释：一个线程可能有多个`AsyncStackRoot`，给定一个线程，如何获取到正在运行的`AsyncStackRoot`呢？答案就是将每个线程正在使用的`AsyncStackRoot`保存到thread local storage中。从宏观上看，`AsyncStackFrame`和`AsyncStackRoot`的完整关系如下：

```cpp
//  Stack Register
//      |
//      V
//  Stack Frame       currentStackRoot (TLS)
//      |                   |
//      V                   V
//  Stack Frame <----- AsyncStackRoot -----> AsyncStackFrame -----> AsyncStackFrame -> ...
//      |   (stackFramePtr) |      (topFrame)            (parentFrame)
//      V                   |
//  Stack Frame             |(nextRoot)
//      :                   |
//      V                   V
//  Stack Frame <----- AsyncStackRoot -----> AsyncStackFrame -----> AsyncStackFrame -> ...
//      |   (stackFramePtr) |      (topFrame)            (parentFrame)
//      V                   X
//  Stack Frame
//      :
//      V
```

- `AsyncStackFrame`和`AsyncStackFrame`之间通过`parentFrame`连接起来
- `AsyncStackRoot`通过`topFrame`保存当前正在执行的协程
- `AsyncStackRoot`和`AsyncStackRoot`之间则通过`nextRoot`连接起来，即`EventLoop A`创建了`EventLoop B`，则`B`的`AsyncStackRoot`的`nextRoot`指向`A`的`AsyncStackRoot`
- 当前线程正在使用的`AsyncStackRoot`，则保存到TLS中

> 延伸的问题是，在进程外（比如gdb中），如何线程找到对应的`AsyncStackRoot`？通常来说，每个线程的`control-block`中会保存线程号、进程号、优先级以及我们所关心的TLS等，`control-block`的指针则会保存在`fs`寄存器中。在gdb中，可以根据该寄存器，找到线程对应的TLS，也就能找到对应的`AsyncStackRoot`。相关代码可以参考folly中的`AsyncStackRootHolder`。
>

### **Finding the stack-frame that corresponds to an async-frame activation**

到这我们已经完全了解了协程之间的调用栈是如何组织的。最后一个问题就是如何处理普通函数调用协程，即如何把普通调用栈和协程调用栈连在一起，答案就是前面提到的`stackFramePtr`指针。仍以文章开头的代码为例：

```cpp
void baz() {
    // ...
}

folly::coro::Task<void> bar() {
    co_return baz();
}

folly::coro::Task<void> foo() {
    co_await bar();
}

int main() {
    folly::CPUThreadPoolExecutor executor{1};
    folly::coro::blockingWait(foo().scheduleOn(&executor));
    return 0;
}
```

沿着协程的`AsyncStackFrame`调用栈，我们可以获取到如下的调用栈：

```
- baz
- bar
- foo
```

而我们期望的完整调用栈是：

```cpp
- baz
- bar
- foo
- main
```

即，如果在某个线程中，通过`coroutine_handle`恢复了一个协程后，需要将此处的普通调用栈和协程调用栈连在一起。也就是把普通调用栈的相关信息，保存到`AsyncStackRoot`中的`stackFramePtr`和`returnAddress`字段中。每当一个线程创建一个`EventLoop`，即异步操作时，就会通过`AsyncStackRoot`记录下普通调用栈中的栈帧位置，以便后续从协程调用栈再切换回普通调用栈。代码的调用关系如下：

```cpp
blockingWait
  -> BlockingWaitTask::get or BlockingWaitTask::getVia
    -> resumeCoroutineWithNewAsyncStackRoot
```

实际工作是由`ScopedAsyncStackRoot`这个类以RAII的形式设置和恢复的：

1. 创建一个`AsyncStackRoot`：
    1. 其`stackFramePtr`字段指向普通调用栈的地址，即`FOLLY_ASYNC_STACK_FRAME_POINTER`，实际是调用`__builtin_frame_address`。之后就能通过这个指针，在恢复完整调用栈时，从协程调用栈再切换回普通调用栈。（参照后续说明）
    2. 其`returnAddress`字段指向普通调用栈的下一条指令地址
    3. 更新TLS中的`AsyncStackRoot`为调用`blockingWait`线程的`AsyncStackRoot`
2. 更新`AsyncStackRoot`中的`topFrame`为要恢复协程的`AsyncStackFrame`
3. 恢复协程执行
4. 协程执行完成，将TLS中的`AsyncStackRoot`还原为调用`blockingWait`线程的`AsyncStackRoot`

```cpp
FOLLY_NOINLINE void resumeCoroutineWithNewAsyncStackRoot(
    coro::coroutine_handle<> h, folly::AsyncStackFrame& frame) noexcept {
  // In ScopedAsyncStackRoot's constructor, it will:
  // 1. create a AsyncStackRoot with
  //      stackFramePtr = FOLLY_ASYNC_STACK_FRAME_POINTER()
  //      returnAddress = FOLLY_ASYNC_STACK_RETURN_ADDRESS()
  // 2. update TLS AsyncStackRoot as current AsyncStackRoot
  detail::ScopedAsyncStackRoot root;
  root.activateFrame(frame);
  h.resume();

  // In ScopedAsyncStackRoot's destructor, it will:
  // 1. restore TLS AsyncStackRoot to nextRoot of current AsyncStackRoot's
}

ScopedAsyncStackRoot::ScopedAsyncStackRoot(
    void* framePointer = FOLLY_ASYNC_STACK_FRAME_POINTER(),
    void* returnAddress = FOLLY_ASYNC_STACK_RETURN_ADDRESS()) noexcept {
  root_.setStackFrameContext(framePointer, returnAddress);
  root_.nextRoot = currentThreadAsyncStackRoot.get();
  currentThreadAsyncStackRoot.set(&root_);  // update thread local AsyncStackRoot
}

ScopedAsyncStackRoot::~ScopedAsyncStackRoot() {
  assert(currentThreadAsyncStackRoot.get() == &root_);
  assert(root_.topFrame.load(std::memory_order_relaxed) == nullptr);
  currentThreadAsyncStackRoot.set_relaxed(root_.nextRoot);
}

inline void AsyncStackRoot::setStackFrameContext(
    void* framePtr, void* ip) noexcept {
  stackFramePtr = framePtr;
  returnAddress = ip;
}
```

有了普通调用栈的返回地址后，我们就可以把普通调用栈和协程调用栈连接在一起。准确来说，所有`folly::coro::Task`暴露的接口，最终都会调用`blockingWait`。因此无论怎么使用`folly::coro::Task`，普通函数调用协程都会被保存下来。

我们结合这个链表梳理一遍还原整个调用栈的过程：

```
Stack Register
    |
    V
Stack Frame       currentStackRoot (TLS)
    |                   |
    V                   V
Stack Frame <----- AsyncStackRoot -----> AsyncStackFrame -----> AsyncStackFrame -> ...
    |   (stackFramePtr) |      (topFrame)            (parentFrame)
    V                   |
Stack Frame             |(nextRoot)
    :                   |
    V                   V
Stack Frame <----- AsyncStackRoot -----> AsyncStackFrame -----> AsyncStackFrame -> ...
    |   (stackFramePtr) |      (topFrame)            (parentFrame)
    V                   X
Stack Frame
    :
    V
```

1. 读取TLS中的`AsyncStackRoot`。
2. 处理普通函数调用：沿着`%rbp`和返回地址，恢复同步调用栈。直到发现当前stack frame的`%rbp`和当前`AsyncStackRoot`的`stackFramePtr`指向同一个位置（说明在这里启动了一个异步操作），不再沿着`%rbp`遍历。这里就是普通调用栈和协程调用栈切换的地方。
3. 通过`AsyncStackRoot`的`topFrame`找到正在执行的协程对应的`AsyncStackFrame`，并不断沿着`parentFrame`延伸协程调用栈。
4. 直至某个`AsyncStackFrame`的`parentFrame`为空，说明当前`AsyncStackRoot`的整条调用链已经遍历完成。
5. 此时会根据`AsyncStackRoot`的`stackFramePtr`切换到普通调用栈。
6. 之后就又从第2步开始重复上述流程。

所以恢复出来的调用栈可能是交替出现的：`同步栈 -> 异步栈 -> 同步栈 -> 异步栈 -> ...`。

> PS：也就是说，`AsyncStackRoot`中的`topFrame`用于解决从同步调用栈切换到异步调用栈，而`stackFramePtr`用于解决从异步调用栈切换到同步调用栈。
>

在下面图中的序号代表在最终还原出来的调用栈中的序号：

```
Stack Register
    |
    V
Stack Frame(0)   currentStackRoot (TLS)
    |                  |
    V                  V
Stack Frame(3) <- AsyncStackRoot  -> AsyncStackFrame(1) -> AsyncStackFrame(2) -> X
    |                  |
    V                  |
Stack Frame(4)         |
    :                  |
    V                  V
Stack Frame(7) <- AsyncStackRoot  -> AsyncStackFrame(5) -> AsyncStackFrame(6) -> X
    |                  |
    V                  X
Stack Frame(8)
    :
    V
```

## Example

到这`AsyncStackFrame`的核心逻辑都介绍完了，folly把恢复异步调用栈都封装到了这个[gdb脚本](https://github.com/facebook/folly/blob/main/folly/coro/scripts/co_bt.py)中。

```cpp
void baz() {
    // ...
}

folly::coro::Task<void> bar() {
    co_return baz();
}

folly::coro::Task<void> foo() {
    co_await bar();
}

int main() {
    folly::CPUThreadPoolExecutor executor{1};
    folly::coro::blockingWait(foo().scheduleOn(&executor));
    return 0;
}
```

对于上面的例子，如果在`baz`处打上断点，输入`co_bt`就能得到如下的调用栈：

```cpp
>>> co_bt
#0  0x00005555557dfa61 in baz() () at /home/doodle.wang/source/folly/folly/experimental/coro/test/BlockingWaitTest.cpp:376
#1  0x00005555557dfcdf in bar(bar()::_Z3barv.Frame*) [clone .actor] () at /home/doodle.wang/source/folly/folly/experimental/coro/test/BlockingWaitTest.cpp:379
#2  0x00005555557e0232 in foo(foo()::_Z3foov.Frame*) [clone .actor] () at /home/doodle.wang/source/folly/folly/experimental/coro/test/BlockingWaitTest.cpp:382
#3  0x00005555557ec3fb in std::__n4861::coroutine_handle<void>::resume() const () at /usr/include/c++/13/coroutine:135
#4  0x00005555557e00fa in foo(foo()::_Z3foov.Frame*) [clone .actor] () at /home/doodle.wang/source/folly/folly/experimental/coro/test/BlockingWaitTest.cpp:383
...
#7  0x00005555557e03dc in main () at /home/doodle.wang/source/folly/folly/experimental/coro/test/BlockingWaitTest.cpp:388
#8  0x00007ffff782a1ca in ??? () at ???:0
#9  0x00007ffff782a28b in __libc_start_main () at ???:0
#10 0x00005555557da2f5 in _start () at ???:0
```

完结！

## Reference

[Async stack traces in folly: Introduction](https://developers.facebook.com/blog/post/2021/09/16/async-stack-traces-folly-Introduction/)

[Async Stacks: Making Senders and Coroutines Debuggable - Ian Petersen & Jessica Wong - CppCon 2024 - YouTube](https://www.youtube.com/watch?v=nHy2cA9ZDbw)