---
layout: single
title: Deciphering C++ Coroutines, part 3
date: 2025-11-06 00:00:00 +0800
categories: 学习
tags: C++
---

Asymmetric transfer vs Symmetric transfer.

## Asymmetric transfer reviewed

上一篇结尾时我们提到：Resume continuation就是Asymmetric transfer，为什么会称为非对称转移呢？

我们首先回顾下上一篇例子中，控制流是如何在`caller`和`callee`之间转移的。

```cpp
SimpleTask<int> caller() {
    std::cout << "  > caller()\n";
    int result = co_await callee();
    std::cout << "  > caller: result = " << result << "\n";
    co_return result * 2;
}
```

- caller → callee

```cpp
void await_suspend(std::coroutine_handle<> continuation) noexcept {
    handle_.promise().continuation_ = continuation;
    handle_.resume();
}
```

通过在`co_await callee`中的`Awaiter::await_suspend`，将`caller`的`coroutine_handle`保存到`callee`的promise中，然后调用`.resume()`恢复`callee`执行。注意`callee`的协程栈会在当前调用栈的基础上展开，只有当`callee`执行完成之后，相关的栈才会释放。

- callee → caller

```cpp
void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
    auto continuation = h.promise().continuation_;
    if (continuation) {
        continuation.resume();
    }
}
```

当callee的协程体执行完之后，在`co_await callee_promise.final_suspend`时，在对应的`FinalAwaiter::await_suspend`中，通过promise获取到caller的`coroutine_handle`，然后调用`.resume()`恢复`caller`执行。注意`caller`的协程栈会在当前调用栈的基础上展开，只有当`caller`执行完成之后，相关的栈才会释放。

> 在上一篇提到过，这里再强调下：无论谁来调用`.resume`都一样，`.resume`的调用方在调用时，都会像一个普通函数调用一样，在当前栈的基础上增长出恢复的协程栈。
>

乍一看，都是在某个`Awaiter`的`await_suspend`中恢复了另一个协程。但其实二者有以下不同：

1. 前者是父协程恢复子协程，后者是子协程恢复父协程。
2. 前者是发生在`co_await`表达式时，后者则是协程函数体执行完，控制流到`final_suspend`时。

正是因为有这些控制流传递过程中的不同，它才被称为Asymmetric transfer。除此之外，在调用栈上也能看到控制流从`caller`传递给`callee`，又传递给了`caller`，造成调用栈的层次不断变深。即每次调用`.resume()`时，我们都会创建一个新的stack frame给要恢复的协程，这会导致实际运用中出现[stack overflow](https://godbolt.org/z/gy5Q8q)。

```
main()
└─ caller协程体
   └─ Awaiter::await_suspend(caller's handle)
      └─ handle_.resume() // caller -> callee
         └─ callee协程体
            └─ FinalAwaiter::await_suspend(callee's handle)
              └─ continuation.resume() // callee -> caller
                  └─ caller协程体
                     └─ Awaiter::await_resume()
```

实际运行上一篇给出的[例子](https://godbolt.org/z/c3ocoPeM6)，也能在gdb中也能看到如下的调用栈，出现了`caller → callee → caller`这样的嵌套执行，以及`.resume()`时都会生成新的stack frame。

![figure]({{'/archive/coroutine-8.png' | prepend: site.baseurl}})

## Symmetric transfer

我们看下Symmetric transfer是怎么解决`.resume()`时栈变深的问题。Symmetric transfer最初是在这篇标准[提案](https://wg21.link/P0913R0)被提议，最终被纳入到标准中的。其核心是：当一个协程需要切换到另一个协程时，不再通过显式的调用`resume()`来恢复协程，而是通过返回一个`coroutine_handle`来告知，控制流应该交给哪个协程，然后跳转到这个协程继续执行。

主要引入的修改就是`Awaiter::await_suspend`，我们之前提到过它有两种类型的返回值：

- 返回`void`的`await_suspend()`会无条件地将控制流转移回协程的调用方/恢复方。
- 返回`bool`的`await_suspend()`在返回`false`时表示立即恢复协程并继续执行，而返回true会将控制流转移回协程的调用方/恢复方。

实际上还有第三种情况，也就是这个提案中提议的：

- 返回一个`std::coroutine_handle<T>`。即返回一个 `std::coroutine_handle<T>`，表明控制流应该对称地移交到由返回的`coroutine_handle`所标识的协程。另外提供了一个特殊标识
`std::noop_coroutine_handle`，用于标识没有要恢复的协程，控制流会直接返回给当前协程的调用方或者恢复方。

也就是说，在Symmetric transfer中，我们只是简单地挂起一个协程并恢复另一个协程。它和Asymmetric transfer的一个重要区别在于：两个协程之间没有隐含的调用者/被调用者关系，当一个协程挂起时，它可以将执行流移交给任何被挂起的协程（包括它自己），并且在下次挂起或完成时，不一定要将执行流移交回之前的那个协程。

### co_await reviewed

根据[标准](https://eel.is/c++draft/expr.await#5.1.1)来说，Symmetric transfer并不要做很多修改，只需要在协程需要挂起时，在对应的`Awaiter::await_suspend`中返回一个 `coroutine_handle`，表明控制流应该对称地移交到由返回的`coroutine_handle`所标识的协程。进一步的问题是，控制流是怎么交给这个协程的？

要回答这个问题，我们需要了解编译器在Symmetric transfer情况是如何展开co_await。

```cpp
{
  auto&& value = <expr>;
  auto&& awaitable = get_awaitable(promise, static_cast<decltype(value)>(value));
  auto&& awaiter = get_awaiter(static_cast<decltype(awaitable)>(awaitable));
  if (!awaiter.await_ready()) {
    using handle_t = std::coroutine_handle<promise_type>;

    <suspend-coroutine>

    auto next = awaiter.await_suspend(handle_t::from_promise(promise));
    next.resume();
    return; // <return-to-caller-or-resumer>

resume_point_label:
    <resume-point>
  }

  return awaiter.await_resume();
}
```

注意在第一篇中我们介绍`co_await`被展开的代码中，当`await_suspend`返回`void`或者`true`时，会直接`<return-to-caller-or-resumer>`，即控制流会被返回给调用方/恢复方。

而在Symmetric transfer中，按我们上面所说的语义，它需要恢复`await_suspend`返回的`coroutine_handle`，并且恢复这个协程即`next.resume();`。而`<return-to-caller-or-resumer>`部分则会被编译器处理为一个`return;`语句。

也就是说，当我们通过`.resume()`恢复一个协程时，在这个协程体执行过程中，又通过对称转移调用了另一个`.resume()`，而之后的`return;`语句会使控制流回到最初`.resume()`的调用方。注意到两次`.resume()`的函数声明完全一致，本质上就是一个[tail call](https://en.wikipedia.org/wiki/Tail_call)。

### co_await reviewed again

不幸的是，我前面隐藏了一些事实，编译器生成的实际代码并不和上面的示例完全一致。准确说，编译器生成的代码中并没有`next.resume()`，我们下面就会说到这一点。

```cpp
SimpleTask<int> caller() {
    std::cout << "  > caller()\n";
    int result = co_await callee();
    std::cout << "  > caller: result = " << result << "\n";
    co_return result * 2;
}
```

我们这里直接给出一个对称转移的[例子](https://godbolt.org/z/n896xMrdG)，整体流程和上一篇介绍Asymmetric transfer时一样，只修改了`await_suspend`部分代码（具体改动在后面）。可以把代码中`caller`协程放到cppinsight中，恢复函数会被展开如下的示意代码，这里为了容易理解做了一些命名调整和逻辑简化：

```cpp
void caller_resume(__callerFrame* frame) {
    try {
        switch (frame->suspend_index) {
            case 0: break;
            case 2: goto resume_point_1;
            case 4: goto resume_point_2;
            case 6: goto resume_point_3;
        }

        // initial_suspend
        frame->initial_awaiter = frame->promise.initial_suspend();
        if (!frame->initial_awaiter.await_ready()) {
            frame->initial_awaiter.await_suspend(
                std::coroutine_handle<SimpleTask<int>::promise_type>::from_address(frame)
            );
            frame->suspend_index = 2;
            frame->initial_await_called = true;
            return;
        }

    resume_point_1:
        frame->initial_awaiter.await_resume();
        std::cout << "  > caller()\n";

        // co_await callee()
        frame->callee_awaiter = callee().operator co_await();
        if (!frame->callee_awaiter.await_ready()) {
            auto next = frame->callee_awaiter.await_suspend(
                std::coroutine_handle<SimpleTask<int>::promise_type>::from_address(frame)
            );
            if (next.address()) {
                frame->suspend_index = 4;
                return;
            }
        }

    resume_point_2:
        frame->result = frame->callee_awaiter.await_resume();
        std::cout << "  > caller: result = " << frame->result << "\n";
        frame->promise.return_value(frame->result * 2);
        goto final_suspend;

    } catch (...) {
        if (!frame->initial_await_called) throw;
        frame->promise.unhandled_exception();
    }

final_suspend:
    // final_suspend
    frame->final_awaiter = frame->promise.final_suspend();
    if (!frame->final_awaiter.await_ready()) {
        auto next = frame->final_awaiter.await_suspend(
            std::coroutine_handle<SimpleTask<int>::promise_type>::from_address(frame)
        );
        if (next.address()) {
            frame->suspend_index = 6;
            return;
        }
    }

resume_point_3:
    frame->destroy_fn(frame);
}

```

和上一篇的介绍一样，`caller`的协程体被展开为一个有限状态机，通过在挂起时调整协程的状态，以便在恢复这个协程时，能跳转到正确的位置继续执行。我们聚焦到关心的对称转移部分：

1. `Awaiter::await_suspend`返回了一个`coroutine_handle`，即代码中的`next`，控制流应该交给`coroutine_handle`对应的协程。
2. 如果`next`是一个非空值，即`next`的确指向一个协程，此时会设置挂起点下标，就立刻返回了。此处会发生一些**神奇**的事情，包括恢复`next`协程，最终控制流会返回到当前协程的调用方或者恢复方。（注意`next.resume()`没有显示出现在这段代码中）
3. 之后，当`caller`协程被最终唤醒时，会继续从`resume_point_2`开始执行。

为什么说神奇的事情呢，因为甚至[标准草案](https://eel.is/c++draft/expr.await#note-1)都说的闪烁其词：

> If the type of *await-suspend* is std::coroutine_handle<Z>, *await-suspend*.resume() is evaluated[.](https://eel.is/c++draft/expr.await#5.1.1.sentence-1) This resumes the coroutine referred to by the result of *await-suspend*[.](https://eel.is/c++draft/expr.await#5.1.1.sentence-2) Any number of coroutines can be successively resumed in this fashion, eventually returning control flow to the current coroutine caller or resumer ([[dcl.fct.def.coroutine]](https://eel.is/c++draft/dcl.fct.def.coroutine))[.](https://eel.is/c++draft/expr.await#5.1.1.sentence-3)
>

以及这是最初对称转移的提案[P0913R0](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0913r1.html)中的描述：

> If that expression has type `std::experimental::coroutine_handle<Z>` and evaluates to a value *s*, the coroutine referred to by *s* is resumed as if by a call *s*`.resume()`. [*Note:* Any number of coroutines may be successively resumed in this fashion, eventually returning control flow to the current coroutine caller or resumer (8.4.4) *-- end note*]
>

标准和草案中都提到，当在`Awaiter::await_suspend`中返回一个`coroutine_handle`，它会被恢复。但也提到了，这期间可能会恢复若干个协程，并且最终控制流会回到当前协程的调用方或恢复方。

关于所谓**神奇**的事情，我们在这只需要知道，编译器实际生成的代码中不会直接出现类似`next.resume()`的调用，而是会以直接跳转的形式，恢复对称转移返回的`coroutine_handle`对应协程执行。这里需要再补充一些背景知识，具体的机制，我们会结合代码再介绍。

从另一方面角度来说，协程体之所以会被编译器处理为固定的一个格式，也正是因为对称转移。即在`std::coroutine_handle::resume()`中，又需要以某种形式调用另一个`std::coroutine_handle::resume()`并返回。为了避免栈的增长，于是想以tail call优化的形式解决这个问题。

Tail call可以理解为：

> For compilers generating assembly directly, tail-call elimination is easy: it suffices to replace a call opcode with a jump one, after fixing parameters on the stack.
>

也就是，在条件允许的情况下，编译器可以用一个`jump`指令，替代`call`和`ret`指令。

```nasm
; before
foo:
  call B
  call A
  ret

; after
foo:
  call B
  jmp  A
```

tail call优化的前提如下所示，而正是为了使得协程能够满足tail call的条件，编译器才把协程体处理为前述的格式：

- 调用约定(calling convention)支持尾调用，并且调用方和被调用方的调用约定相同：可以看到编译器中把整个协程分为了两部分，构造和初始化coroutine的一个函数（被称为`ramp`），以及包含协程体、协程状态机的函数（称为`body`，比如上面的`caller_resume`）。编译器这样处理，就能保证能保证调用约定的要求。
- 返回类型相同：都是调用`std::coroutine_handle::resume()`，返回类型都是`void`
- 在返回到调用方之前，不需要在调用之后执行任何non-trivial析构函数：协程中所有生命周期可能会跨越挂起点的所有对象，都会被保存在coroutine frame中，不需要调用回调。而生命周期不跨越挂起点的对象，比如局部变量，都会在挂起之前已经析构。
- ~~调用不在try/catch 块内部~~：我们可以看到coroutine body中是有try/catch的，而按这篇[博客](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer)里的说法，而编译器通过前述把`.resume()`从`body`部分挪出去的手段，使协程满足了这个要求。

### Demo

了解了对称转移的原理之后，我们结合[demo](https://godbolt.org/z/n896xMrdG)再看下具体改动。大体流程和上一篇中介绍Asymmetric transfer时几乎完全一致，其中只有`Awaiter::await_suspend`和`FinalAwaiter::await_suspend`的地方稍有不同（即转移控制流的地方），下面会具体分析。

按照唯二的不同点如下：

- `Awaiter::await_suspend`：
    - Asymmetric transfer中，`caller`在`co_await callee();`时发现要保存`caller`的`coroutine_handle`，然后手动恢复`callee`继续执行。
    - 而Symmetric transfer中，我们不再手动恢复子协程`callee`，而是直接返回它的`coroutine_handle`。

```cpp
/*
// Asymmetric transfer
void await_suspend(std::coroutine_handle<> continuation) noexcept {
    handle_.promise().continuation_ = continuation;
    handle_.resume();
}
*/

// Symmetric transfer
std::coroutine_handle<> await_suspend(std::coroutine_handle<> continuation) noexcept {
    handle_.promise().continuation_ = continuation;
    return handle_;
}
```

- `FinalAwaiter::await_suspend`：
    - Asymmetric transfer中，当`callee`协程体执行完时，检查promise中是否有设置过要恢复的协程，如果有则直接恢复。
    - 而Symmetric transfer中，当`callee`协程体执行完时，检查promise中是否有设置过要恢复的协程，如果有则返回它的`coroutine_handle`，代表控制流要交给这个协程（即例子中的`caller`），否则返回一个空值。

```cpp
/*
// Asymmetric transfer
void await_suspend(std::coroutine_handle<promise_type> h) noexcept {
    auto continuation = h.promise().continuation_;
    if (continuation) {
        continuation.resume();
    }
}
*/

// Symmetric transfer: Return the next coroutine to transfer to, or noop if none.
std::coroutine_handle<> await_suspend(std::coroutine_handle<promise_type> h) noexcept {
    auto continuation = h.promise().continuation_;
    if (continuation) {
        return continuation;
    }
    return std::noop_coroutine();
}
```

总结一下两处改动，都是在`await_suspend`中返回一个`coroutine_handle`，代表控制流需要需要转移到这个协程。

实际运行这个例子，在gdb中可以发现Symmetric transfer和Asymmetric transfer的调用栈不同，虽然仍然出现了`caller → callee → caller`这样的嵌套执行，但是并不会出现`.resume()`这样的stack frame了。

![figure]({{'/archive/coroutine-9.png' | prepend: site.baseurl}})

下面我们会直接走读这个demo的汇编，查看关键步骤的栈帧和堆的状态，揭开对称转移的真实面目。

## Symmetric transfer details

### Backgrounds

首先，`caller`的协程帧数据结构如下，部分变量命名做过调整：

```cpp
struct __callerFrame {
  // +0x00
  void (*resume_fn)(__callerFrame*);      // 协程状态机函数 即恢复函数指针
  void (*destroy_fn)(__callerFrame*);     // 析构函数指针

  // +0x10
  SimpleTask<int>::promise_type promise {
    int value;
    std::exception_ptr exception;
    std::coroutine_handle<> continuation_;
  };

  // +0x28
  std::coroutine_handle<SimpleTask<int>::promise_type> self_handle; // 自身coroutine_handle

  // +0x30
  int16_t suspend_index;                  // 挂起点下标 主要用于标识协程状态
  bool needs_free;                        // 是否需要释放
  char initial_await_called;              // initial_suspend是否已调用

  // +0x34
  std::suspend_always initial_awaiter;    // initial_suspend的Awaiter

  // +0x38
  int result;                             // 生命周期跨越挂起点的局部变量

  // +0x40
  SimpleTask<int>::Awaiter callee_awaiter; // co_await callee()的Awaiter

  // +0x48
  SimpleTask<int> callee_task;            // callee协程的ReturnType对象
                                          // 同时也提供了operator co_await

  // +0x50+
  SimpleTask<int>::promise_type::FinalAwaiter final_awaiter;
                                          // final_suspend的Awaiter
};
```

这其中对于理解对称转移最重要的就是`resume_fn`这个函数指针。每个协程都有一个状态机函数，每次协程开始执行或者被恢复时，都会调用这个状态机函数。而协程当前的状态用`suspend_index`来表示，即协程当前在哪个挂起点被挂起。

比如`caller`协程的状态机函数就是上面的`caller_resume`，协程会根据`suspend_index`跳转到`caller_resume`中的不同位置。

> 实际生成的汇编代码中，caller协程的状态机函数demangle之后命名为`caller(caller()::_Z6callerv.Frame*) [clone .actor]`
>

```cpp
        switch (frame->suspend_index) {
            case 0: break;
            case 2: goto resume_point_1;
            case 4: goto resume_point_2;
            case 6: goto resume_point_3;
        }
```

注意到上面给出的状态机代码中，`suspend_index`没有奇数的原因：偶数代表正常挂起，而奇数代表协程需要销毁。由于挂起点有多个，因此需要从不同的状态进行相应清理的逻辑也不同。比如`caller`协程的`suspend_index`对应的完整状态如下：

```
suspend_index = 0:  初始状态
suspend_index = 1:  销毁时从状态0清理
suspend_index = 2:  initial_suspend被挂起
suspend_index = 3:  销毁时从状态2清理
suspend_index = 4:  co_await callee()挂起
suspend_index = 5:  销毁时从状态4清理
suspend_index = 6:  final_suspend被挂起
suspend_index = 7:  销毁时从状态6清理
```

状态机函数可能会被调用多次，每次调用时协程处于不同被挂起的挂起点处。除此以外，在汇编代码中，状态机函数与普通函数并没有什么不同，比如都有prologue，即每次函数调用时都有对`%rbp`和`%rsp`的相应压栈操作：

```nasm
0000000000001838 <_Z6callerPZ6callervE16_Z6callerv.Frame.actor>:
    1838:  endbr64
    183c:  push   %rbp
    183d:  mov    %rsp,%rbp
    1840:  push   %rbx
    1841:  sub    $0x28,%rsp
    1845:  mov    %rdi,-0x28(%rbp)
```

这里再多提一点，状态机函数的只有一个参数，即协程的coroutine frame指针。每次调用状态机函数，都会把这个指针都保存到了`-0x28(%rbp)`处。

### main → caller

接下来，我们梳理整个demo的执行流程，完整的汇编参见[这里](https://gist.github.com/critical27/0a95391b8df0df2de3d60d792f547325)（为了便于理解，编译器指定`-O0`）。`caller`的状态机函数入口地址为`1838`，`callee`的状态机函数入口地址为`13fb`。

1. `main`调用`caller()`，创建协程

    ```nasm
    0000000000001c94 <main>:
    1c94:  endbr64
    1c98:  push   %rbp
    1c99:  mov    %rsp,%rbp
    1c9c:  push   %rbx
    1c9d:  sub    $0x18,%rsp
    1ca1:  mov    %fs:0x28,%rax
    1caa:  mov    %rax,-0x18(%rbp)
    1cae:  xor    %eax,%eax
    1cb0:  lea    -0x20(%rbp),%rax
    1cb4:  mov    %rax,%rdi
    1cb7:  call   16e6 <_Z6callerv>
    1cbc:  lea    0x137d(%rip),%rax   ; return address
    ```

    调用`caller()`后状态如下所示：

    ```
    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ caller() constructor                    │
    │ - return addr: 0x1cbc (main)            │ ← pushed by call at 0x1cb7
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame) has not been constructed yet
    ```

    在`16e6`开始的接下来一段汇编中，会做几件事：

    - 分配coroutine frame所需要的内存
    - 创建`promise`
    - 调用`promise.get_return_object()`
    - 第一次调用caller协程的状态机函数

    ```nasm
    00000000000016e6 <_Z6callerv>:
    16e6:  endbr64
    16ea:  push   %rbp
    16eb:  mov    %rsp,%rbp
    16ee:  push   %rbx
    16ef:  sub    $0x38,%rsp
    16f3:  mov    %rdi,-0x38(%rbp)
    16f7:  mov    %fs:0x28,%rax
    1700:  mov    %rax,-0x18(%rbp)
    1704:  xor    %eax,%eax
    1706:  movq   $0x0,-0x20(%rbp)
    170e:  movb   $0x0,-0x21(%rbp)
    1712:  movb   $0x0,-0x22(%rbp)
    1716:  mov    $0x58,%eax          ; 88 bytes for coroutine frame
    171b:  mov    %rax,%rdi
    171e:  call   1150 <_Znwm@plt>    ; operator new
    1723:  mov    %rax,-0x20(%rbp)    ; -0x20(%rbp) = caller_frame pointer
    1727:  mov    -0x20(%rbp),%rax
    172b:  movb   $0x1,0x32(%rax)
    172f:  mov    -0x20(%rbp),%rax    ; %rax = caller_frame pointer
    1733:  lea    0xfe(%rip),%rdx     ; rdx = 1838 即caller的状态机函数地址 (resume_fn)
    173a:  mov    %rdx,(%rax)
    173d:  mov    -0x20(%rbp),%rax    ; %rax = caller_frame pointer
    1741:  lea    0x519(%rip),%rdx    ; rdx = 1c61 即caller的销毁函数地址 (destroy_fn)
    1748:  mov    %rdx,0x8(%rax)
    174c:  mov    -0x20(%rbp),%rax    ; %rax = caller_frame pointer
    1750:  add    $0x10,%rax          ; %rax = &(caller_frame->promise)
    1754:  mov    %rax,%rdi
    1757:  call   204c <...>          ; 调用promise构造函数
    175c:  movb   $0x1,-0x21(%rbp)
    1760:  mov    -0x20(%rbp),%rax
    1764:  lea    0x10(%rax),%rdx     ; rdx = &(caller_frame->promise)
    1768:  mov    -0x38(%rbp),%rax    ; %rax = 返回值地址
    176c:  mov    %rdx,%rsi
    176f:  mov    %rax,%rdi
    1772:  call   231a <>             ; 调用promise.get_return_object()
    1777:  movb   $0x1,-0x22(%rbp)
    177b:  mov    -0x20(%rbp),%rax    ; %rax = caller_frame pointer
    177f:  movw   $0x0,0x30(%rax)     ; caller_frame->suspend_index = 0
    1785:  mov    -0x20(%rbp),%rax
    1789:  mov    %rax,%rdi
    178c:  call   1838 <...>          ; 第一次调用caller的状态机函数
    1791:  jmp    181a <...>
    ```

    调用状态机后的状态如下所示：

    ```
    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ caller() constructor                    │
    │ - return addr: 0x1cbc (main)            │ ← pushed by call at 0x1cb7
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x1791                   │ ← pushed by call at 0x178c
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 0
    ```

2. `caller`协程的`suspend_index`初始值为`0`。由于`initial_suspend`是`suspend_always`，因此在`co_await promise.initial_suspend`时，`await_ready`返回`false`，代表会被挂起，而`await_suspend`返回`void`，代表无条件将控制流返回给调用方，且在返回之前`suspend_index`被设置为`2`。

    状态机函数返回后，依次执行返回地址`1791`的代码，最终由返回到`main`函数。

    ```nasm
    1791:  jmp    181a
    ; ...
    181a:  mov    -0x18(%rbp),%rax
    181e:  sub    %fs:0x28,%rax
    1827:  je     182e
    182e:  mov    -0x38(%rbp),%rax    ; rax = &task (return value)
    1832:  mov    -0x8(%rbp),%rbx
    1836:  leave
    1837:  ret                        ; return to main (0x1cbc)
    ```

    此时栈帧中只有`main`：

    ```
    ┌─────────────────────┐ ← High Address
    │ main() stack frame  │
    └─────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 2
    ```

3. 之后在`main`函数中通过`coroutine_handle`手动恢复`caller`，`call`指令的返回地址是`1d10`。

    ```nasm
    ; 省略main中部分代码...
    1d04:  lea    -0x20(%rbp),%rax    ; %rax = caller()的返回对象 即SimpleTask<int>
                                      ; 其中只有一个成员变量即caller协程的coroutine_handle
    1d08:  mov    %rax,%rdi
    1d0b:  call   26a6 <...>          ; 调用caller's coroutine_handle.resume()

    1d10:  lea    0x1349(%rip),%rax
    ; ...
    ```

    ```nasm
    00000000000026a6 <_ZNKSt7__n486116coroutine_handleIN10SimpleTaskIiE12promise_typeEE6resumeEv>:
    26a6:  endbr64
    26aa:  push   %rbp
    26ab:  mov    %rsp,%rbp
    26ae:  sub    $0x10,%rsp
    26b2:  mov    %rdi,-0x8(%rbp)
    26b6:  mov    -0x8(%rbp),%rax
    26ba:  mov    (%rax),%rax        ; %rax = calle_frame pointer
    26bd:  mov    (%rax),%rdx        ; %rdx = *(%rax) = caller->resume_fn = 1838
    26c0:  mov    %rax,%rdi
    26c3:  call   *%rdx              ; 调用caller状态机函数
    26c5:  nop
    26c6:  leave
    26c7:  ret
    ```

    调用后的状态如下所示：

    ```
    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ ← pushed by call at 0x26c3
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 2
    ```


### caller → callee

1. `caller`协程继续执行，`suspend_index`为`2`，跳转到如下汇编继续执行：

    ```nasm
    195b:  mov    -0x28(%rbp),%rax    ; %rax = caller coroutine frame
    195f:  movb   $0x1,0x33(%rax)     ; 设置initial_await_called = 1
    1963:  mov    -0x28(%rbp),%rax
    1967:  add    $0x34,%rax          ; %rax = &(caller_frame->initial_awaiter)
    196b:  mov    %rax,%rdi
    196e:  call   1f22 <...>          ; 调用initial_awaiter.await_resume();
    1973:  lea    0x16a0(%rip),%rax   ; 加载字符串 "  > caller()\n"
    197a:  mov    %rax,%rsi
    197d:  lea    0x36bc(%rip),%rax   ; 加载 std::cout
    1984:  mov    %rax,%rdi
    1987:  call   1140 <...>          ; 输出字符串
    ```

2. `co_await callee()`，对应汇编如下，会通过`call 12a9`创建`callee`协程

    ```nasm
    198c:  mov    -0x28(%rbp),%rax
    1990:  add    $0x48,%rax          ; %rax = &(caller_frame->callee_task)
    1994:  mov    %rax,%rdi
    1997:  call   12a9 <_Z6calleev>   ; 调用callee()创建协程
    199c:  mov    -0x28(%rbp),%rax
    19a0:  lea    0x48(%rax),%rdx     ; %rdx = &(caller_frame->callee_task)
    19a4:  mov    -0x28(%rbp),%rax
    19a8:  add    $0x40,%rax          ; %rax = &(caller_frame->callee_awaiter)
    19ac:  mov    %rdx,%rsi
    19af:  mov    %rax,%rdi
    19b2:  call   244c <...>          ; 调用operator co_await()
    19b7:  mov    -0x28(%rbp),%rax
    19bb:  add    $0x40,%rax          ; %rax = &(caller_frame->callee_awaiter)
    19bf:  mov    %rax,%rdi
    19c2:  call   2550 <...>          ; 调用callee_awaiter.await_ready()
    19c7:  xor    $0x1,%eax
    19ca:  test   %al,%al
    19cc:  je     1a0b <...>          ; 如果await_ready返回true 跳转到状态4
    ```

3. 由于`await_ready`返回`false`，调用`Awaiter::await_suspend`对称转移至`callee`

    ```cpp
    std::coroutine_handle<> await_suspend(std::coroutine_handle<> continuation) noexcept {
        handle_.promise().continuation_ = continuation;
        return handle_;
    }
    ```

    对应汇编如下，在`Awaiter::await_suspend`中，会把`caller`的`coroutine_handle`保存到`callee`的promise中。

    ```nasm
    19ce:  mov    -0x28(%rbp),%rax
    19d2:  movw   $0x4,0x30(%rax)     ; caller_frame->suspend_index = 4
    19d8:  mov    -0x28(%rbp),%rax
    19dc:  lea    0x40(%rax),%rbx     ; %rbx = &(caller_frame->callee_awaiter)
    19e0:  mov    -0x28(%rbp),%rax
    19e4:  add    $0x28,%rax          ; %rax = caller的coroutine_handle(当前协程句柄)
    19e8:  mov    %rax,%rdi
    19eb:  call   2132 <...>          ; 转换为coroutine_handle<void>
    19f0:  mov    %rax,%rsi
    19f3:  mov    %rbx,%rdi
    19f6:  call   2564 <...>          ; 调用callee_awaiter.await_suspend()
    19fb:  mov    %rax,-0x20(%rbp)    ; %rax = 对称转移返回的协程句柄 (即callee的coroutine_handle)
    19ff:  jmp    1b66 <...>          ; 准备跳转到callee
    1a04:  mov    $0x0,%ebx
    1a09:  jmp    1a27 <...>
    ```

    跳转前的状态如下所示，注意`19d2`处已经把`suspend_index`改为4：

    ```
    Before symmetric transfer (caller -> callee)

    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ ← pushed by call at 0x26c3
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4
    ```

    具体跳转到`callee`的汇编代码如下。注意在`19fb`时已经把`callee`的`coroutine_handle`保存在`-0x20(%rbp)`了，然后通过`coroutine_handle::address()`获取到`callee`的`coroutine frame`地址，即`callee`的状态机函数`callee_resume`的入口地址，并保存到`%rdx`中，最终通过`call *%rdx`跳转至`13fb`。

    ```nasm
    1b66:  endbr64
    1b6a:  lea    -0x20(%rbp),%rax    ; rax = &(callee's coroutine_handle)
    1b6e:  mov    %rax,%rdi
    1b71:  call   1dd4 <...address>   ; 调用coroutine_handle::address
                                      ; %rax = callee_frame pointer
    1b76:  mov    (%rax),%rdx         ; rdx = *(callee_frame) = callee's resume_fn函数地址 即状态机函数
    1b79:  mov    %rax,%rdi           ; rdi = callee_frame pointer
    1b7c:  call   *%rdx               ; ★ indirect tail call调用callee的状态机函数 ★
    1b7e:  jmp    1c46 <cleanup>
    ```

    > `1b66`开始的这段汇编，是编译器对`caller`协程生成的一段对称转移通用指令，后面还会再见到一次，只不过根据`await_suspend`的返回值不同，最终跳转的位置也不同。
    >

    跳转后的状态如下所示，注意`caller`此时处于被挂起状态，而`callee`的`coroutine frame`还没有创建。

    ```
    After symmetric transfer (caller -> callee)

    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ ← pushed by call at 0x26c3
    ├─────────────────────────────────────────┤
    │ callee_resume (not started)             │
    │ - symmetric transferred from 0x1b7c     │
    │ - return addr: 0x1b7e                   │
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4
      - suspened

    - callee frame (__calleeFrame) has not been constructed yet
    ```

    需要注意的是，`call *%rdx`所跳转的函数，是`callee`的状态机函数，它也会通过对称转移，使控制流切换到其他协程上。但是在执行`call *%rdx`时，下一条指令地址`1b7e`的确会被压栈，然后跳转到`*%rdx`处，只不过其返回地址`1b7e`对应的代码并像普通函数调用返回后立马执行，而可能是会被tail call优化，绕一个大圈回来。


### callee → caller

1. 创建`callee`，这一步和之前一样，分配coroutine frame，构造`promise`，调用`promise.get_return_object()`。

    ```cpp
    SimpleTask<int> callee() {
        std::cout << "      > callee()\n";
        co_return 42;
    }
    ```

    经过`co_await initial_suspend`和函数体，通过`promise_type`的`return_value`接口保存了返回值`42`。并且此时`callee`协程已经执行完成，会将其状态机函数置为`nullptr`，后续不能再被调用。（但coroutine frame还没有释放，释放时机下面会讲）

    ```nasm
    1543:  call   20c4 <...>          ; 调用promise.return_value
    1548:  nop
    1549:  mov    -0x28(%rbp),%rax    ; rax = callee_frame pointer
    154d:  movq   $0x0,(%rax)         ; callee_frame->resume_fn = nullptr
                                      ; 即标识callee协程已完成 状态机函数后续不能再被调用
    ```

    最终进入到`co_await final_suspend`阶段。`FinalAwaiter`的`await_ready`返`回false`，于是在`await_suspend`处再次对称转移。

    ```cpp
    std::coroutine_handle<> await_suspend(std::coroutine_handle<promise_type> h) noexcept {
        auto continuation = h.promise().continuation_;
        if (continuation) {
            return continuation;
        }
        return std::noop_coroutine();
    }
    ```

    前面的汇编就不展开了，只详细看对称转移部分：

    ```nasm
    157b:  endbr64
    157f:  mov    -0x28(%rbp),%rax    ; %rax = callee_frame pointer
    1583:  movw   $0x4,0x30(%rax)     ; callee_frame->suspend_index = 4
    1589:  mov    -0x28(%rbp),%rax
    158d:  lea    0x35(%rax),%rdx     ; %rdx = &(callee_frame->final_awaiter)
    1591:  mov    -0x28(%rbp),%rax
    1595:  mov    0x28(%rax),%rax     ; %rax = callee的coroutine_handle(当前协程句柄)
    1599:  mov    %rax,%rsi
    159c:  mov    %rdx,%rdi
    159f:  call   21e2 <...>          ; 调用FinalAwaiter::await_suspend()
    15a4:  mov    %rax,-0x20(%rbp)    ; %rax = 对称转移返回的协程句柄(即caller的coroutine_handle)
    15a8:  lea    -0x20(%rbp),%rax    ; %rax = &(caller's coroutine_handle)
    15ac:  mov    %rax,%rdi
    15af:  call   1dd4 <...address>   ; 调用coroutine_handle::address
                                      ; %rax = caller_frame pointer
    15b4:  mov    (%rax),%rdx         ; %rdx = *(caller_frame) = caller's resume_fn函数地址 即状态机函数
    15b7:  mov    %rax,%rdi           ; %rdi = caller_frame pointer
    15ba:  call   *%rdx               ; ★ indirect tail call调用caller状态机函数 ★
    15bc:  jmp    1698 <cleanup>
    ```

    原理和上面一次对称转移一样，都是获取`await_suspend`的返回值，通过`coroutine_handle::address()`获取到返回值，即`caller`的`coroutine frame`地址，也就是`caller`的状态机函数`caller_resume`的入口地址，并保存到`%rdx`中，最终通过`call *%rdx`跳转至`1838`。跳转前后的状态如下所示：

    ```
    Before symmetric transfer (callee -> caller)

    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ ← pushed by call at 0x26c3
    ├─────────────────────────────────────────┤
    │ callee_resume                           │
    │ - symmetric transferred from 0x1b7c     │
    │ - return addr: 0x1b7e                   │
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4
      - suspended

    - callee frame (__calleeFrame)
      - suspend_index = 4
      - resume_fn = nullptr

    After symmetric transfer (callee -> caller)

    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ ← pushed by call at 0x26c3
    ├─────────────────────────────────────────┤
    │ callee_resume                           │
    │ - symmetric transferred from 0x1b7c     │
    │ - return addr: 0x1b7e                   │
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - symmetric transferred from 0x15ba     │
    │ - return addr: 0x15bc                   │
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4

    - callee frame (__calleeFrame)
      - suspend_index = 4
      - resume_fn = nullptr
      - suspended
    ```

    注意`caller`的状态机函数被再次调用，因此又会对`%rbp`和`%rsp`进行相应操作，出现了一个新的栈帧。

    ```nasm
    0000000000001838 <_Z6callerPZ6callervE16_Z6callerv.Frame.actor>:
        1838:	endbr64
        183c:	push   %rbp
        183d:	mov    %rsp,%rbp
    ```

    准确来说，`callee`协程当然是被挂起的，而`caller`协程是正在执行的，只不过`caller_resume`这个函数由于被优化为了一系列tail call，导致在栈上出现了两次。

2. `caller`状态机会继续执行，此时`suspend_index`为4，跳转到如下代码。主要逻辑就是完成`co_await callee()`的善后，此时`callee`已经执行完成，析构了caller coroutine frame中的`callee_awaiter`和`callee_task`。

    ```nasm
    1a0b:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1a0f:  add    $0x40,%rax          ; %rax = &(caller_frame->callee_awaiter)
    1a13:  mov    %rax,%rdi
    1a16:  call   2630 <...>          ; 调用caller_frame->callee_awaiter.await_resume()
    1a1b:  mov    -0x28(%rbp),%rdx
    1a1f:  mov    %eax,0x38(%rdx)     ; 保存到caller_frame->result中
    1a22:  mov    $0x1,%ebx
    1a27:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1a2b:  add    $0x40,%rax          ; %rax = &(caller_frame->callee_awaiter)
    1a2f:  mov    %rax,%rdi
    1a32:  call   24d4 <...>          ; 调用callee_awaiter的析构函数
    1a37:  cmp    $0x1,%ebx
    1a3a:  jne    1a43 <...>          ; %ebx为1 不跳转
    1a3c:  mov    $0x1,%ebx
    1a41:  jmp    1a48 <...>
    1a43:  mov    $0x0,%ebx
    1a48:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1a4c:  add    $0x48,%rax          ; %rax = &(caller_frame->callee_task)
    1a50:  mov    %rax,%rdi
    1a53:  call   2352 <...>          ; 调用callee_task的析构函数
    1a58:  cmp    $0x1,%ebx
    1a5b:  jne    1b32 <...>          ; %ebx为1 不跳转
    1a61:  nop
    ```

    注意在析构caller coroutine frame中的`callee_awaiter`，也就是析构`co_await callee()`生成的`Awaiter`对象时，`callee`的coroutine frame会被释放。大致调用路径如下：

    ```nasm
    1a32:  call 24d4  ; 调用Awaiter::~Awaiter()
      ↓
    2544:  call 2770  ; 调用coroutine_handle<SimpleTask<int>::promise_type>::destroy()
      ↓
    278e:  call *%rdx ; callee.Frame.destroy (0x16b3)
      ↓
    16df:  call 13fb  ; callee.Frame.actor (最后一次调用状态机函数，清理)
      ↓
    释放callee的coroutine frame
    ```

    具体过程如下，不想深究的可以调到下一步骤。首先在`coroutine_handle<SimpleTask<int>::promise_type>::destroy()`中，先根据`coroutine_handle`获取coroutine frame指针，再获取coroutine frame中的销毁函数指针，最后调用。

    ```nasm
    0000000000002770 <_ZNKSt7__n486116coroutine_handleIN10SimpleTaskIiE12promise_typeEE7destroyEv>:
    2770:  endbr64
    2774:  push   %rbp
    2775:  mov    %rsp,%rbp
    2778:  sub    $0x10,%rsp
    277c:  mov    %rdi,-0x8(%rbp)
    2780:  mov    -0x8(%rbp),%rax     ; %rax = coroutine handle's pointer
    2784:  mov    (%rax),%rax         ; %rax = &(couroutine frame)
                                      ; 即读取coroutine_handle中的coroutine frame指针
                                      ; 等同于调用coroutine_handle::address()
    2787:  mov    0x8(%rax),%rdx      ; %rdx = &(frame->destory_fn)
    278b:  mov    %rax,%rdi
    278e:  call   *%rdx               ; 调用destory_fn 对于callee来说是16b3
    2790:  nop
    2791:  leave
    2792:  ret
    ```

    之后，在销毁函数`callee(callee()::_Z6calleev.Frame*) [clone .destroy]`中设置`suspend_index`的最低位为1，并最后一次调用状态机函数进行清理。

    ```nasm
    00000000000016b3 <_Z6calleePZ6calleevE16_Z6calleev.Frame.destroy>:
    16b3:  endbr64
    16b7:  push   %rbp
    16b8:  mov    %rsp,%rbp
    16bb:  sub    $0x10,%rsp
    16bf:  mov    %rdi,-0x8(%rbp)
    16c3:  mov    -0x8(%rbp),%rax
    16c7:  movzwl 0x30(%rax),%eax     ; %rax = callee_frame->suspend_index
    16cb:  or     $0x1,%eax           ; 设置suspend_index的最低位为1 表示已销毁
    16ce:  mov    %eax,%edx
    16d0:  mov    -0x8(%rbp),%rax
    16d4:  mov    %dx,0x30(%rax)      ; callee_frame->suspend_index = 5
    16d8:  mov    -0x8(%rbp),%rax
    16dc:  mov    %rax,%rdi
    16df:  call   13fb <...>          ; 最后一次调用状态机函数
    16e4:  leave
    16e5:  ret
    ```

    最后，在状态机函数中析构promise，并调用`operator delete`释放内存

    ```nasm
    00000000000013fb <_Z6calleePZ6calleevE16_Z6calleev.Frame.actor>:
    ; ...
    142b:  mov    -0x28(%rbp),%rax
    142f:  movzwl 0x30(%rax),%eax
    1433:  movzwl %ax,%eax
    1436:  cmp    $0x5,%eax
    1439:  je     15c1 <>             ; suspsend_index = 5则跳转

    ; ...
    15c1:  jmp    15d6 <>

    ; ...
    15d6:  mov    -0x28(%rbp),%rax
    15da:  add    $0x10,%rax
    15de:  mov    %rax,%rdi
    15e1:  call   208a <...>          ; 调用promise的析构函数
    15e6:  mov    -0x28(%rbp),%rax
    15ea:  movzbl 0x32(%rax),%eax
    15ee:  movzbl %al,%eax
    15f1:  test   %eax,%eax
    15f3:  je     1698 <...>
    15f9:  mov    -0x28(%rbp),%rax
    15fd:  mov    %rax,%rdi
    1600:  call   1130 <_ZdlPv@plt>   ; 调用operator delete释放内存
    1605:  jmp    1698 <...>

    ; ...
    1698:  nop
    1699:  mov    -0x18(%rbp),%rax
    169d:  sub    %fs:0x28,%rax
    16a6:  je     16ad
    16a8:  call   1170 <__stack_chk_fail@plt>
    16ad:  mov    -0x8(%rbp),%rbx
    16b1:  leave
    16b2:  ret
    ```

    此时的栈状态如下所示，即callee的coroutine frame已经不存在了，但其状态机函数还存在于栈上。虽然coroutine frame已经不存在，那状态机函数还怎么执行呢？这里需要说明的是，callee的状态机函数剩余还未执行的部分，只是一些清理逻辑且会迅速返回，而不会再读取coroutine frame中的内容。

    ```nasm
    After callee coroutine frame destructed

    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ ← pushed by call at 0x26c3
    ├─────────────────────────────────────────┤
    │ callee_resume                           │
    │ - symmetric transferred from 0x1b7c     │
    │ - return addr: 0x1b7e                   │
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - symmetric transferred from 0x15ba     │
    │ - return addr: 0x15bc                   │
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4
      - callee_awaiter destructed
      - callee_task destructed
    ```

3. 之后`caller`协程继续执行`std::cout << "  > caller: result = " << result << "\n";`，略过相应汇编。最终`caller`协程返回`result * 2`，对应汇编如下

    ```nasm
    1aa4:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1aa8:  add    $0x10,%rax          ; %rax = &(caller_frame->promise.value)
    1aac:  mov    -0x28(%rbp),%rdx    ; %rdx = caller_frame pointer
    1ab0:  mov    0x38(%rdx),%edx     ; %rdx = caller_frame->result
    1ab3:  add    %edx,%edx           ; result * 2
    1ab5:  mov    %edx,%esi
    1ab7:  mov    %rax,%rdi
    1aba:  call   20c4 <...>          ; calle_frame->promise.return_value()
    1abf:  nop
    ```


### caller → main

1. 之后，`caller`协程将进入到`co_await final_suspend`阶段。

    ```nasm
    1ac0:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1ac4:  movq   $0x0,(%rax)         ; caller_frame->resume_fn = nullptr
                                      ; 即标识caller协程已完成 状态机函数后续不能再被调用
    1acb:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1acf:  add    $0x10,%rax          ; %rax = &(caller_frame->promise)
    1ad3:  mov    %rax,%rdi
    1ad6:  call   21be <...>          ; 调用SimpleTask<int>::promise_type::final_suspend()
    1adb:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1adf:  add    $0x50,%rax          ; %rax = &(caller_frame->final_awaiter)
    1ae3:  mov    %rax,%rdi
    1ae6:  call   21ce <...>          ; 调用final_awaiter.await_ready() 返回值为false
    1aeb:  xor    $0x1,%eax           ; 取反后 %rax = 1
    1aee:  test   %al,%al
    1af0:  je     1b1f <...>          ; await_ready返回false 不会跳转
    1af2:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1af6:  movw   $0x6,0x30(%rax)     ; caller_frame->suspend_index = 6
    1afc:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1b00:  lea    0x50(%rax),%rdx     ; %rax = &(caller_frame->final_awaiter)
    1b04:  mov    -0x28(%rbp),%rax    ; %rax = caller_frame pointer
    1b08:  mov    0x28(%rax),%rax     ; %rax = caller's coroutine handle
    1b0c:  mov    %rax,%rsi
    1b0f:  mov    %rdx,%rdi
    1b12:  call   21e2 <...>          ; 调用final_awaiter.await_suspend()
    1b17:  mov    %rax,-0x20(%rbp)    ; %rax = std::noop_coroutine
    1b1b:  jmp    1b66 <...>

    ```

    由于`caller`协程并没有指定`continuation`，所以在`final_awaiter.await_suspend()`时会返回`std::noop_coroutine`。再次跳转到我们在第4步中提到处理对称转移的通用序列`1b66`处。

2. 由于`std::noop_coroutine`本质上就是一个dummy coroutine frame，根据[提案](https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0913r0.html)中所述，`std::noop_coroutine的address`函数返回值非空，但其状态机函数什么都不会执行，于是直接跳转到`caller`状态机函数的收尾处。

    ```nasm
    1b66:  endbr64
    1b6a:  lea    -0x20(%rbp),%rax    ; %rax = std::noop_coroutine
    1b6e:  mov    %rax,%rdi
    1b71:  call   1dd4 <...>          ; 对noop_coroutine调用address() 会返回一个dummy coroutine frame
    1b76:  mov    (%rax),%rdx         ; %rdx = &(dummy coroutine's resume_fn)
    1b79:  mov    %rax,%rdi
    1b7c:  call   *%rdx               ; 调用dummy coroutine的状态机函数
                                      ; 本质上什么都不会执行
    1b7e:  jmp    1c46 <...>          ; 跳转至1c46返回
    ```

3. 最终在`caller`状态机函数返回

    ```nasm
    ; epilogue
    1c46:  nop
    1c47:  mov    -0x18(%rbp),%rax
    1c4b:  sub    %fs:0x28,%rax
    1c54:  je     1c5b <...>
    1c56:  call   1170 <__stack_chk_fail@plt>
    1c5b:  mov    -0x8(%rbp),%rbx
    1c5f:  leave
    1c60:  ret
    ```

    `ret`后的状态为：

    ```nasm
    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ ← pushed by call at 0x26c3
    ├─────────────────────────────────────────┤
    │ callee_resume                           │
    │ - symmetric transferred from 0x1b7c     │
    │ - return addr: 0x1b7e                   │
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - symmetric transferred from 0x15ba     │
    │ - return addr: 0x15bc                   │ <- cpu is here
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4
      - callee_awaiter destructed
      - callee_task destructed
    ```

    此时CPU下一条要执行指令是其返回地址指向的`15bc`，很快又再次在`16b2`处`ret`。

    ```nasm
    15bc:  jmp    1698 <...>

    ; ...
    1698:  nop
    1699:  mov    -0x18(%rbp),%rax
    169d:  sub    %fs:0x28,%rax
    16a6:  je     16ad
    16a8:  call   1170 <__stack_chk_fail@plt>
    16ad:  mov    -0x8(%rbp),%rbx
    16b1:  leave
    16b2:  ret
    ```

    `ret`后的状态为：

    ```nasm
    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ ← pushed by call at 0x26c3
    ├─────────────────────────────────────────┤
    │ callee_resume                           │
    │ - symmetric transferred from 0x1b7c     │
    │ - return addr: 0x1b7e                   │ <- cpu is here
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4
      - callee_awaiter destructed
      - callee_task destructed
    ```

    此时CPU下一条要执行指令是其返回地址指向的`1b7e`，这里又跳转到1c46处（代码上面出现过），会再次`ret`。

    ```nasm
    1b7e:  jmp    1c46 <...>          ; 跳转至1c46返回
    ```

    `ret`后状态为：

    ```nasm
    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ ← pushed by call at 0x1d0b
    ├─────────────────────────────────────────┤
    │ caller_resume                           │
    │ - return addr: 0x26c5                   │ <- cpu is here
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4
      - callee_awaiter destructed
      - callee_task destructed
    ```

    此时CPU下一条要执行指令是其返回地址指向的`26c5`，再次`ret`。

    ```nasm
    26c5:  nop
    26c6:  leave
    26c7:  ret
    ```

    `ret`后状态为：

    ```nasm
    ┌─────────────────────────────────────────┐ ← High Address
    │ main()                                  │
    ├─────────────────────────────────────────┤
    │ coroutine_handle.resume()               │
    │ - return addr: 0x1d10                   │ <- cpu is here
    └─────────────────────────────────────────┘ ← Low Address (rsp)

    Heap State:
    - caller frame (__callerFrame)
      - suspend_index = 4
      - callee_awaiter destructed
      - callee_task destructed
    ```

    此时`caller`协程也执行完成，接下来会将`caller`的coroutine frame释放。大致调用路径如下：

    ```nasm
    1d65: call 2352  ; 调用caller协程的返回值析构 SimpleTask::~SimpleTask()
      ↓
    23db: call 2770  ; 调用coroutine_handle<SimpleTask<int>::promise_type>::destroy()
      ↓
    278e: call *%rdx ; caller.Frame.destroy (0x1c61)
      ↓
    1c8d: call 1838  ; caller.Frame.actor (最后一次调用状态机函数，清理)
      ↓
    释放caller的coroutine frame
    ```

    执行完这一些列操作后，最终就回返回到`main`了：

    ```nasm
    1db4:  mov    -0x8(%rbp),%rbx
    1db8:  leave
    1db9:  ret
    ```


## Asymmetric transfer vs Symmetric transfer

希望上面冗长的流程没有吓跑正在阅读的你，事实上我也花了将近一周的时间，才把短短100行左右的demo对应汇编大致分析了一遍。但不可否认的是，通过汇编，的确加深了我对协程整个执行流程的理解。最后我们总结下非对称转移和对称转移的核心。

在非对称转移中，每次`coroutin_handle.resume()`都是一个普通的函数调用。正如文章开头我们分析的，如果协程中又调用了其他协程的`coroutin_handle.resume()`，就会导致栈的深度线性增长，甚至出现stack overflow。

而对称转移中，编译器会生成如下所示的对称转移汇编。虽然形式上通过`call *%rdx`这样的indirect tail call，仍然会导致栈的深度加深，但是其返回地址，也就是`call *%rdx`的下一条指令，都会跳转到一段清理的逻辑中。

```nasm
endbr64
lea    -0x20(%rbp),%rax    ; rax = &(coroutine_handle)
mov    %rax,%rdi
call   1dd4 <...address>   ; 调用coroutine_handle::address
                           ; %rax = frame pointer
mov    (%rax),%rdx         ; rdx = *(frame pointer) = resume_fn函数地址 即状态机函数
mov    %rax,%rdi           ; rdi = frame pointer
call   *%rdx               ; indirect tail call调用对应协程的状态机函数
jmp    1c46 <cleanup>
```

然而，对称转移并不能完全解决栈深度线性增长的问题，比如A co_await B，B co_await C，C co_await D，一直这样下去，对称转移中的`call *%rdx`也会导致栈深度增长，并不能完全避免stack overflow。

但对称转移的优势在于，虽然栈确实会增长，但一旦协程执行完成，就能通过`jmp cleanup`并`ret`的形式快速清理，而不需要像非对称转移那样层层返回。

到这为止，关于协程的基础介绍应该告一段落，如果有下一篇的话，会研究下`folly::coro::Task`协程库。

## Reference

[C++ Coroutines: Understanding Symmetric Transfer | Asymmetric Transfer](https://lewissbaker.github.io/2020/05/11/understanding_symmetric_transfer)

[[expr.await]](https://eel.is/c++draft/expr.await#5.1.1)