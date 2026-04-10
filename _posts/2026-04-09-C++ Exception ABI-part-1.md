---
layout: single
title: C++ Exception Handling ABI, part 1
date: 2026-04-09 00:00:00 +0800
categories: 学习
tags: C++
---

C++ 异常处理的第一篇，这一篇先对栈展开的流程建立一个基本概念，下一篇再补充一些深入细节。

## Itanium C++ ABI

要聊 C++ 的异常处理，首先我们得了解 Itanium C++ ABI。这个术语中，Itanium 是一个曾经想取代 x86，但目前已经退出市场的处理器架构。虽然处理器架构失败了，但其副产品 Itanium C++ ABI 早就不只服务于 Itanium 架构，而是逐渐演化成类 Unix 平台上 C++ ABI 的事实标准。今天我们在 x86-64、AArch64 等平台上看到的 C++ 异常处理、RTTI、`dynamic_cast` 等运行时行为，都遵循这套规范。

对于异常处理，虽然 Itanium ABI 最初是为 C++ 设计的，但它是建立在更底层的组件之上，这些组件都是语言无关的：

- System V ABI
- DWARF
- libunwind

因此，任何语言只要按照 Itanium ABI 来实现，都可以使用同样的异常处理 ABI 机制。 所以 Itanium C++ ABI 实际可以拆分成两层：

- Level 1: Base ABI，定义语言无关的栈展开机制
- Level 2: C++ ABI，在 Level 1 之上补充 C++ 语义，比如 `throw` / `catch` 等

Base ABI 描述语言无关的栈展开过程，并定义 `_Unwind_*` API。常见实现如下所示，它们也就是通常所说的 `unwinder` ：

- `libgcc` 中的 `libgcc_s.so.1` （`libgcc_s`是gcc运行时一部分）
- `libunwind`: https://github.com/libunwind/libunwind
- `llvm` 中的 `libunwind`: https://github.com/llvm/llvm-project/tree/main/libunwind

C++ ABI则和 C++ 语言本身相关，定义了 `__cxa_*` API（例如 `__cxa_allocate_exception`、`__cxa_throw`、`__cxa_begin_catch` 等），以及如何通过这些 API，实现 C++ 的 `throw` / `catch` 语法。常见实现有：

- `libstdc++` 中的 `libsupc++` (support library for C++): 除了上面提到的 `__cxa_*` API 之外，还提供 RTTI 以及 `dynamic_cast`。`libsupc++` 提供了 C++ 中所有动态类型相关的实现
- `llvm` 中的 `c++abi`: https://github.com/llvm/llvm-project/tree/main/libcxxabi

> 这些库的名字很容易让人混淆。`libstdc++` = C++ 标准库 + `libsupc++`，其中 `libsupc++` 提供了异常处理所需的 `__cxa_*` API。`libgcc` 则是编译器生成代码时依赖的底层运行时支持库，其中包含异常处理需要的 `_Unwind_*` API。
>

| ABI 层级 | 作用 | GCC 常见实现 | LLVM 常见实现 |
| --- | --- | --- | --- |
| Level 1: Base ABI（语言无关） | 定义`_Unwind_*` API，负责栈展开 | `libgcc_s.so.1`（隶属 `libgcc`） | `libunwind` |
| Level 2: C++ ABI（语言相关） | 负责将 `throw` / `catch` 语义转换为对应`__cxa_*` API调用 | `libsupc++`（隶属 `libstdc++`） | `libc++abi` |

下面会先简单描述两层 API 的主要作用，之后再分章节详细介绍。

第一层规定了如何做栈展开（stack unwinding），是一套语言无关的通用接口。它主要包括以下几部分（具体内容先看不懂也没关系）：

- `_Unwind_Exception` 异常对象结构
- `_Unwind_*` API，这里只列出最重要的两个：
    - `_Unwind_RaiseException`
    - `_Unwind_Resume`
- 如何进行栈展开，整个过程分成搜索和清理两个阶段，后面会详细介绍
- `personality`：在栈展开过程中，`unwinder` 会询问当前栈帧的 `personality` 能不能处理这个异常：
    - 如果能，应该跳到哪个 `catch` 块，从哪条指令开始继续执行
    - 如果不能，当前栈帧是否需要额外清理工作，比如清理栈上对象

概括一下，这一层定义的是：由 `_Unwind_RaiseException` 负责执行两阶段栈展开，而栈展开过程中涉及的语言相关概念，例如 `catch` 块、离开作用域后的对象析构等，都由 `personality` 封装。正因为如此，这套 ABI 才能支持多语言，并允许它们和 C++ 一起工作。

第二层就是我们通常所说的 C++ ABI。它定义的是 C++ 语言特性在运行时的实现规则和接口，大致可以分成几部分：

- `__cxa_exception`: C++ 的异常对象结构，其中包含 Level 1 里的 `_Unwind_Exception`
- `__cxa_* API`: 异常运行时 API。它本质上是把 C++ 异常相关的语法（`throw` / `catch`）翻译成运行时的函数调用
    - `__cxa_begin_catch`
    - `__cxa_end_catch`
    - `__cxa_allocate_exception`
    - `__cxa_throw`
- RTTI：动态类型相关的支持，主要包括：
    - `std::type_info`
    - `typeid`运算符
    - 类型比较方式
    - `dynamic_cast`

> 异常处理过程中需要比较 `throw` 出来的异常类型和 `catch` 声明的类型是否匹配，因此 RTTI 也会参与异常处理。
>

## Level 1: Base ABI

### _Unwind_Exception

数据结构如下：

```cpp
// Level 1
struct _Unwind_Exception {
  _Unwind_Exception_Class exception_class; // an identifier, used to tell whether the exception is native
  _Unwind_Exception_Cleanup_Fn exception_cleanup;
  _Unwind_Word private_1; // zero: normal unwind; non-zero: forced unwind, the _Unwind_Stop_Fn function
  _Unwind_Word private_2; // saved stack pointer
} __attribute__((aligned));
```

`exception_class` 和 `exception_cleanup` 由 Level 2 中负责抛异常的 API 设置。Level 1 并不关心 `exception_class` 的具体含义，而是把它原样传给 `personality`，再由后者判断当前异常是 `native exception` 还是 `foreign exception` （可以简单理解为 C++ 运行时抛出的异常为 `native exception`，其他语言抛出的异常为 `foreign exception`，这块文章最后会再补充一些）。

`exception_class` 用来表示这个异常对象属于哪种语言和运行时，前4个字节一般表示厂商，而后4个字节表示语言。例如，`libc++abi` 的 `__cxa_throw` 会把 `exception_class` 设成表示 `"CLNGC++\\0"` 的 `uint64_t`，而 `libsupc++` 使用的是表示 `"GNUCC++\\0"` 的 `uint64_t`。`exception_cleanup` 保存对应异常对象的析构函数，会在出 `catch` 作用域时，由 Level 2 的 API 调用。

栈展开过程中需要的相关信息，比如给定 IP 或者 SP 寄存器如何获取上一个栈帧的 IP 和 SP，则是由具体实现定义。对于 ELF，栈展开的相关信息都保存在 `.eh_frame` 和 `.eh_frame_hdr`中。这部分原理不影响理解栈展开的主要流程，我们在下一篇再详细介绍。

### API

`_Unwind_RaiseException` 负责执行异常的栈展开。这个函数没有通常意义上的 return 语句，控制权最终要么转移给匹配到的 `catch` 块，要么在无法 `catch` 时转移给相应清理代码，从而析构局部对象的代码。整个过程分成两个阶段：`search phase`（搜索阶段）和 `cleanup phase`（清理阶段）。

- 在搜索阶段，要找出能够处理该异常的 `catch`，并把对应栈帧的栈指针记录到 `private_2`
    - 根据 IP、SP 以及其他已保存寄存器，沿着调用链逐帧回溯
    - 对每个栈帧，如果没有对应 `personality` 就跳过；如果有，就传入 `_UA_SEARCH_PHASE` 作为参数并调用它
    - 如果 `personality` 返回 `_URC_CONTINUE_UNWIND`，表示继续向上搜索
    - 如果 `personality` 返回 `_URC_HANDLER_FOUND`，表示找到了匹配的 `catch` 块，将对应栈帧保存到 `private_2`。
    - 过程中如果发现 ABI 层面不匹配，此时搜索停止
- 在清理阶段，要先跳转搜索阶段遍历过程中，没有捕获异常的栈帧的清理代码（通常是局部变量析构），最后再把控制权转交给搜索阶段找到的 `catch` 块
    - 同样根据 IP、SP 和其他寄存器沿调用链逐帧回溯
    - 对每个栈帧，如果没有对应 `personality` 就跳过；如果有，就传入 `_UA_CLEANUP_PHASE` 作为参数并调用它；而搜索阶段标记过的那个栈帧还会额外带上 `_UA_HANDLER_FRAME`
    - 如果 `personality` 返回 `_URC_CONTINUE_UNWIND`，表示没有 `landing pad`，即该栈帧不需要额外处理
    - 如果 `personality` 返回 `_URC_INSTALL_CONTEXT`，表示找到了 `landing pad`，需要跳转到 `landing pad` 继续执行
    - 对于那些没有在搜索阶段被标记的中间栈帧，`landing pad` 只负责清理工作（通常是析构已离开作用域的变量），然后调用 `_Unwind_Resume` 回到清理阶段
    - 对于搜索阶段标记的那个栈帧，`landing pad` 会调用 `__cxa_begin_catch`，随后执行 `catch` 块中的代码，最后调用 `__cxa_end_catch` 完成销毁异常对象

    > `landing pad` 在下面 Level 2 部分会介绍，它是一段编译器为函数生成的处理异常的代码。这里补充一点，具体跳转到 `landing pad` 的操作由 `unwinder` 完成，而跳转到哪里则是由 `personality` 决定的。
    >

关于 `personality` 我们在下一篇会详细介绍，此处只需要了解它连接了 Level 1 和 Level 2 API，其主要功能是：

- 在栈展开过程中检查每个栈帧是否有匹配的 `catch`
- 搜索阶段返回 `_URC_CONTINUE_UNWIND` 或 `_URC_HANDLER_FOUND`，以表示该栈帧能否处理该异常
- 清理阶段返回 `_URC_CONTINUE_UNWIND` 或 `_URC_INSTALL_CONTEXT`，以表示是否跳转到 `landing pad`

除此之外，还有几个常见的 API：

- `_Unwind_ForcedUnwind`: 强制栈展开，也就是跳过搜索阶段，直接进入清理阶段，典型场景是 `pthread_cancel`
- `_Unwind_Resume`: Level 1 中几乎唯一一个直接由编译器生成调用的 API。如果当前栈帧不能捕获异常、但需要先清理栈上对象，那么清理完成后就会调用 `_Unwind_Resume` 继续清理阶段
- `_Unwind_DeleteException:` 调用 `_Unwind_Exception` 中的 `exception_cleanup` 销毁给定的异常对象。
- `_Unwind_Backtrace`: 忽略 `personality`，而是执行一个回调。典型场景就是 gdb 里的 backtrace，大致原理是用当前指令寄存器 `%rip` 去查 `.eh_frame`，算出“上一帧在哪”，然后不断重复

完整 `_Unwind_RaiseException` 栈展开的代码如下：

```cpp
static _Unwind_Reason_Code unwind_phase1(unw_context_t *uc, _Unwind_Context *ctx,
                                         _Unwind_Exception *obj) {
  // Search phase: unwind and call personality with _UA_SEARCH_PHASE for each frame
  // until a handler (catch block) is found.
  unw_init_local(uc, ctx);
  for(;;) {
    if (ctx->fdeMissing) return _URC_END_OF_STACK;
    if (!step(ctx)) return _URC_FATAL_PHASE1_ERROR;
    ctx->getFdeAndCieFromIP();
    if (!ctx->personality) continue;
    switch (ctx->personality(1, _UA_SEARCH_PHASE, obj->exception_class, obj, ctx)) {
    case _URC_CONTINUE_UNWIND: break;
    case _URC_HANDLER_FOUND:
      unw_get_reg(ctx, UNW_REG_SP, &obj->private_2);
      return _URC_NO_REASON;
    default: return _URC_FATAL_PHASE1_ERROR; // e.g. stack corruption
    }
  }
  return _URC_NO_REASON;
}

static _Unwind_Reason_Code unwind_phase2(unw_context_t *uc, _Unwind_Context *ctx,
                                         _Unwind_Exception *obj) {
  // Cleanup phase: unwind and call personality with _UA_CLEANUP_PHASE for each frame
  // until reaching the handler. Restore the register state and transfer control.
  unw_init_local(uc, ctx);
  for(;;) {
    if (ctx->fdeMissing) return _URC_END_OF_STACK;
    if (!step(ctx)) return _URC_FATAL_PHASE2_ERROR;
    ctx->getFdeAndCieFromIP();
    if (!ctx->personality) continue;
    _Unwind_Action actions = _UA_CLEANUP_PHASE;
    size_t sp;
    unw_get_reg(ctx, UNW_REG_SP, &sp);
    if (sp == obj->private_2) actions |= _UA_HANDLER_FRAME;
    switch (ctx->personality(1, actions, obj->exception_class, obj, ctx)) {
    case _URC_CONTINUE_UNWIND:
      break;
    case _URC_INSTALL_CONTEXT:
      unw_resume(ctx); // Return if there is an error
      return _URC_FATAL_PHASE2_ERROR;
    default: return _URC_FATAL_PHASE2_ERROR; // Unknown result code
    }
  }
  return _URC_FATAL_PHASE2_ERROR;
}

_Unwind_Reason_Code _Unwind_RaiseException(_Unwind_Exception *obj) {
  unw_context_t uc;
  _Unwind_Context ctx;
  __unw_getcontext(&uc);
  _Unwind_Reason_Code phase1 = unwind_phase1(&uc, &ctx, obj);
  if (phase1 != _URC_NO_REASON) return phase1;
  return unwind_phase2(&uc, &ctx, obj);
}
```

显然这个过程是可以在一次遍历情况下完成的，之所以要遍历两次，主要是为了在没有任何 `catch` 能处理异常的情况下，避免过早做真正的栈展开。也就是说，在搜索阶段没有找到任何可以处理异常的栈帧时，运行时就能更早终止程序。

## Level 2: C++ ABI

在 Level 1 的基础上，定义了 `__cxa_*` API（例如 `__cxa_allocate_exception`、`__cxa_throw`、`__cxa_begin_catch`、`__cxa_end_catch` 等），以及如何通过这些 API，实现 C++ 的 `throw` / `catch` 语法。

### __cxa_exception

`__cxa_exception` 是在 `_Unwind_Exception` 的基础上，再补充一层 C++ 异常语义信息的结构。

```cpp
struct __cxa_exception {
  void *reserve; // here on 64-bit platforms
  size_t referenceCount; // here on 64-bit platforms
  std::type_info *exceptionType;
  void (*exceptionDestructor)(void *);
  unexpected_handler unexpectedHandler; // by default std::get_unexpected()
  terminate_handler terminateHandler; // by default std::get_terminate()
  __cxa_exception *nextException; // linked to the next exception on the thread stack
  int handlerCount; // incremented in __cxa_begin_catch, decremented in __cxa_end_catch, negated in __cxa_rethrow; last non-dependent performs the clean

  // The following fields cache information the catch handler found in phase 1.
  int handlerSwitchValue; // ttypeIndex in libc++abi
  const char *actionRecord;
  const char *languageSpecificData; // LSDA
  void *catchTemp; // landingPad
  void *adjustedPtr; // adjusted pointer of the exception object

  _Unwind_Exception unwindHeader;
};
```

每个线程都会维护一个当前被捕获异常的栈，`caughtExceptions` 指向栈顶，也就是最近一次被捕获的异常，`__cxa_exception::nextException` 则指向栈里的下一个异常。`uncaughtExceptions` 保存没有被捕获的异常数量，用于支持 `std::uncaught_exceptions()`。

```cpp
struct __cxa_eh_globals {
  __cxa_exception *caughtExceptions;
  unsigned uncaughtExceptions;
};
```

```cpp
int main() {
  try {
    throw 1;
  } catch (...) {
    try {
      throw 2;
    } catch (...) {
      // The global exception stack has two exceptions here.
    }
  }
}
```

而具体处理异常所需的信息，例如某个 IP 指令寄存器是否位于 `try-catch` 范围内、是否存在需要执行的离开作用域变量析构等，通常放在 `language-specific data area`（LSDA）里。这部分属于具体实现细节，不是 Level 2 ABI 直接规定的内容。

> LSDA 也就是 ELF 中的 `.gcc_except_table`，我们在下一篇再详细展开。
>

### **Landing pad**

`landing pad` 由编译器生成，是一段专门用于异常处理的代码，完整定义如下：

A section of user code intended to catch, or otherwise clean up after, an exception. It gains control from the exception runtime via the personality routine, and after doing the appropriate processing either merges into the normal user code or returns to the runtime by resuming or raising a new exception.

它通常会完成以下三种动作之一（注意每个栈帧只会执行其中一种）：

- 无法捕获对应异常，调用已离开作用域变量的析构函数，或者调用通过 `__attribute__((cleanup(...)))` 注册的回调，然后使用 `_Unwind_Resume` 回到清理阶段
- 能捕获对应异常，先析构已离开作用域的变量，再调用 `__cxa_begin_catch`，执行 `catch` 块里的代码，最后调用 `__cxa_end_catch`
- 如果在 `catch` 中有 `rethrow`，则会先析构 `catch` 子句里定义的局部变量，再调用 `__cxa_end_catch`，然后通过 `_Unwind_Resume` 继续清理阶段

如果一个 `try` 块后面跟着多个 `catch` 子句，那么 LSDA 中会有多条 `catch` 条目。不过在代码生成层面，它们通常会汇总到同一个 `landing pad` 中。`personality` 在把控制权转交给 `landing pad` 之前，会调用 `_Unwind_SetGP`，把 `handlerSwitchValue` 放进 `__builtin_eh_return_data_regno(1)` 对应的寄存器里（x86_64 下是 `%rdx`），用来告诉 `landing pad` 这次匹配到的是哪个类型异常，从而跳转到对应的 `catch` 块。

`rethrow` 则是在 `catch` 代码执行过程中通过 `__cxa_rethrow` 触发的。它需要先析构 `catch` 子句里定义的局部变量，再调用 `__cxa_end_catch`，抵消 `catch` 开始时那次 `__cxa_begin_catch`。

### API

- `__cxa_allocate_exception`：当代码里出现 `throw A();` 时，编译器生成的代码会调用这个构造函数，分配一块内存来存放 `__cxa_exception` 和 `A` 对象。其中 `__cxa_exception` 就紧挨在 `A` 对象的左侧。下面这个函数展示了程序可见的异常对象地址和 `__cxa_exception` 之间的关系：

    ```cpp
    static void *thrown_object_from_cxa_exception(__cxa_exception *exception_header) {
      return static_cast<void *>(exception_header + 1);  // address of A
    }
    ```

    > 注意 `__cxa_exception` 是在堆上创建的。运行时通常还会在启动时预留一小块内存，并预先构造一个 `std::bad_alloc`，以便在内存分配失败时仍然能够抛出异常。
    >
- `__cxa_throw`：先根据上面的关系找到 `__cxa_exception`，填好其中各个字段（`referenceCount`、`exception_class`、`unexpectedHandler`、`terminateHandler`、`exceptionType`、`exceptionDestructor`、`unwindHeader.exception_cleanup`），然后调用 `_Unwind_RaiseException` 开始栈展开
- `__cxa_begin_catch`：编译器会在 `catch` 块开头生成对它的调用。主要作用是更新 `__cxa_exception` 中的 `handlerCount`，更新当前线程的全局异常栈，返回被抛出对象的地址
- `__cxa_end_catch`：编译器会在 `catch` 块结束处，或者在 `rethrow` 前生成对它的调用。主要作用是更新 `__cxa_exception` 中的 `handlerCount`，如果为0，则从全局异常栈出栈。
- `__cxa_rethrow`：它会给异常对象打上“重新抛出”的标记。这样当 `__cxa_end_catch` 把 `handlerCount` 减到 0 时，这个异常对象不会被销毁，因为后续 `_Unwind_Resume` 恢复清理阶段时还要继续使用它

Level 2 的 API 主要都是为了提供 C++ 的各种语法底层支持，除了基础的 `throw` / `catch` 之外，还包括 `std::current_exception`、`std::rethrow_exception`、`std::get_terminate` 等。下面是一个简化版的 `__cxa_throw` 实现：

```cpp
void __cxa_throw(void *thrown, std::type_info *tinfo, void (*destructor)(void *)) {
  __cxa_exception *hdr = (__cxa_exception *)thrown - 1;
  hdr->exceptionType = tinfo; hdr->destructor = destructor;
  hdr->unexpectedHandler = std::get_unexpected();
  hdr->terminateHandler = std::get_terminate();
  hdr->unwindHeader.exception_class = ...;
  __cxa_get_globals()->uncaughtExceptions++;
  _Unwind_RaiseException(&hdr->unwindHeader);
  // Failed to unwind, e.g. the .eh_frame FDE is absent.
  __cxa_begin_catch(&hdr->unwindHeader);
  std::terminate();
}
```

## Example

下面结合一个例子，再梳理一遍整个异常处理流程。

```cpp
struct A {
    ~A() {}
};

void baz() {
    throw 1;
}

void bar() {
    A a;
    baz();
}

void foo() {
    try {
        bar();
    } catch (int x) {
        x++;
    }
}
```

对应的简化汇编代码如下：

```cpp
void baz() {
    __cxa_exception *thrown = __cxa_allocate_exception(sizeof(int));
    *thrown = 1;
    __cxa_throw(thrown, &typeid(int), nullptr/*destructor*/);
}

void bar() {
    A a;
    baz();
    return;
landing_pad:
    a.~A();
    _Unwind_Resume();
}

void foo() {
    bar();
    return;
landing_pad:
	  __cxa_begin_catch(obj);
	  x++;
	  __cxa_end_catch(obj);
}
```

控制流可以概括成下面几步：

- `foo` 调用 `bar`，`bar` 调用 `baz`，`baz` 抛出异常
- `baz` 动态分配一块内存，这块内存里依次保存一个 `__cxa_exception` 对象和被抛出的 `int`，然后执行 `__cxa_throw`
- `__cxa_throw` 会设置 `__cxa_exception` 中的字段，然后调用 `_Unwind_RaiseException`

`_Unwind_RaiseException` 开始执行栈展开。第一阶段要先搜索能够捕获 `int` 异常的栈帧：

- 对 `bar` 来说，传入 `_UA_SEARCH_PHASE` 调用 `personality`；返回值是 `_URC_CONTINUE_UNWIND`，表示这里不能捕获该异常
- 对 `foo` 来说，传入 `_UA_SEARCH_PHASE` 调用 `personality`；返回值是 `_URC_HANDLER_FOUND`，表示这里能捕获该异常
- `foo` 这个栈帧的栈指针会被记录下来，存入 `private_2`，然后搜索阶段结束

此时已经确定 `foo` 的栈帧可以接住这个异常，第二阶段开始做清理：

- `bar` 的栈帧没有被搜索阶段标记，传入 `_UA_CLEANUP_PHASE` 调用 `personality`，返回 `_URC_INSTALL_CONTEXT`，代表有 `landing pad`
- 跳转到 `bar` 栈帧对应的 `landing pad`，完成清理后，通过 `_Unwind_Resume` 回到清理阶段
- `foo` 的栈帧在搜索阶段已经被标记，传入 `_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME` 调用 `personality` 时，返回 `_URC_INSTALL_CONTEXT`，代表有 `landing pad`
- 跳转到 `foo` 栈帧对应的 `landing pad`，其中调用 `__cxa_begin_catch`，执行 `catch` 代码，最后调用 `__cxa_end_catch`

完整的[汇编](https://godbolt.org/z/TxvWfbh8j)如下，可以对照加深理解（重点关注 `bar` 和 `foo` 的 `landing pad`）：

```nasm
A::~A() [base object destructor]:
        pushq   %rbp
        movq    %rsp, %rbp
        movq    %rdi, -8(%rbp)
        nop
        popq    %rbp
        ret
        .set    A::~A() [complete object destructor],A::~A() [base object destructor]
baz():
        pushq   %rbp
        movq    %rsp, %rbp
        movl    $4, %edi
        call    __cxa_allocate_exception
        movl    $1, (%rax)
        movl    $0, %edx
        movl    $_ZTIi, %esi
        movq    %rax, %rdi
        call    __cxa_throw
bar():
        pushq   %rbp
        movq    %rsp, %rbp
        pushq   %rbx
        subq    $24, %rsp
        call    baz()
        leaq    -17(%rbp), %rax
        movq    %rax, %rdi
        call    A::~A() [complete object destructor]
        jmp     .L6

        ; landing pad of bar
        movq    %rax, %rbx
        leaq    -17(%rbp), %rax
        movq    %rax, %rdi
        call    A::~A() [complete object destructor]
        movq    %rbx, %rax
        movq    %rax, %rdi
        call    _Unwind_Resume
.L6:
        movq    -8(%rbp), %rbx
        leave
        ret
foo():
        pushq   %rbp
        movq    %rsp, %rbp
        subq    $16, %rsp
        call    bar()
        jmp     .L12

        ; landing pad of foo
        cmpq    $1, %rdx
        je      .L9
        movq    %rax, %rdi
        call    _Unwind_Resume
.L9:
        ; catch block in foo
        movq    %rax, %rdi
        call    __cxa_begin_catch
        movl    (%rax), %eax
        movl    %eax, -4(%rbp)
        addl    $1, -4(%rbp)
        call    __cxa_end_catch
.L12:
        nop
        leave
        ret
```

这里详细分析下 `bar` 和 `foo` 的 `landing pad`：

对于 `bar` ，在跳转到对应的 `landing pad` 之前，`_Unwind_RaiseException` 已经通过 `personality` 确定了 `bar` 不能处理这个异常，因此它的 `landing pad` 就是清理栈上的对象，然后调用 `_Unwind_Resume` 继续栈展开。

```nasm
        ; landing pad of bar
        movq    %rax, %rbx
        leaq    -17(%rbp), %rax
        movq    %rax, %rdi
        call    A::~A() [complete object destructor]
        movq    %rbx, %rax
        movq    %rax, %rdi
        call    _Unwind_Resume
```

对于 `foo`，在跳转到对应的 `landing pad` 之前，`_Unwind_RaiseException` 已经通过 `personality` 确定了 `foo` 能处理这个异常，并且知道是第几个 `catch` 块与之匹配。相关信息会通过下面两个寄存器传给 `landing pad`：

```nasm
; %rax -> exception object，后续会传给 __cxa_begin_catch
; %rdx -> 类型匹配结果
```

通过比对 `%rdx`，跳转到对应的 `catch` 块。`__cxa_begin_catch` 会返回被抛出对象的地址，也就是 `catch` 块里 `x` 对应的地址。执行 `x++` 之后，最后调用 `__cxa_end_catch` 完成这次异常捕获。

> 注意，`%rax` 里已经保存了抛出的异常对象。`__cxa_begin_catch` 之所以还要再返回一次对象地址，是因为这里可能需要做一次地址调整。
>

```nasm
        ; landing pad of foo
        ; 确定foo能处理当前异常 通过比较%rdx 跳转到对应的catch block进行处理
        cmpq    $1, %rdx
        je      .L9                ; go to catch(int)

        ; 不能catch当前异常 继续调用_Unwind_Resume
        movq    %rax, %rdi
        call    _Unwind_Resume
.L9:
        ; catch block in foo
        movq    %rax, %rdi
        call    __cxa_begin_catch
        movl    (%rax), %eax
        movl    %eax, -4(%rbp)
        addl    $1, -4(%rbp)       ; x++
        call    __cxa_end_catch
```

## Misc

最后再补充一些零碎的信息。

### Native exception vs Foreign exception

前面提到 `_Unwind_Exception` 中有个 `exception_class` 字段，

Level 1 API 不会处理该字段，而是将其原样传给 `personality`，`personality` 利用这个值来区分 `native exception` 和 `foreign exception`：

- `native exception`: 由相同C++ ABI运行时抛出的异常。即包含完整的C++类型信息（RTTI），可以被C++运行时正确地栈展开，从而进行捕获。
- `foreign exceptions`: 非C++代码产生，不遵循C++ ABI异常处理规范，只能被`catch (...)`捕获。

> 之所以要强调相同C++ ABI运行时的一个典型例子是：`libstdc++` 抛出的异常会被 `libc++abi` 视为 `foreign exception`。
>

`__cxa_begin_catch` 和 `__cxa_end_catch` 对于 `native exception` 以及 `foreign exception` 有不同的处理方式：

`void* __cxa_begin_catch(void *obj)` 编译器会在 `catch` 块开头生成对它的调用。对于：

- `native exception`
    - 增加 `handlerCount`
    - 将异常压入当前线程的全局异常栈，并减少 `uncaught_exception` 计数
    - 返回调整后的异常对象的地址指针
- `foreign exception`（不一定有 `__cxa_exception` 头部）
    - 若当前线程的全局异常栈为空则压栈，否则调用 `std::terminate` （在任意时刻，C++ ABI运行时只能处理一个 `foreign exception`）
    - 返回 `static_cast<_Unwind_Exception *>(obj) + 1`（假设 `_Unwind_Exception` 紧邻被抛出对象）

`void __cxa_end_catch()` 在 `catch` 块结束或 `rethrow` 时被调用。对于：

- `native exception`
    - 从当前线程的全局异常栈中取出异常，减少 `handlerCount`
    - 当 `handlerCount` 减至 0 时（引用计数为 0），将其从全局异常栈中出栈
    - 当 `handlerCount` 减至 0 时调用 `__cxa_free_exception`（若为 dependent exception，则减少 `referenceCount`，待其降至 0 时再调用 `__cxa_free_exception`）
- `foreign exception`
    - 调用 `_Unwind_DeleteException`
    - 执行 `__cxa_eh_globals::uncaughtExceptions = nullptr;`（和 `__cxa_begin_catch` 时对应，栈中只有一个异常）

> 注意，除 `__cxa_begin_catch` 和 `__cxa_end_catch` 之外，大多数 `__cxa_*` 函数都无法处理 `foreign exception`（因为它们没有 `__cxa_exception` 头部）。
>

这一篇到这就差不多了，主要以了解异常处理和栈展开的流程为主。下一篇将从 `personality` 开始，详细描述栈展开的原理。

## Reference

- [C++ exception handling ABI | MaskRay](https://maskray.me/blog/2020-12-12-c++-exception-handling-abi)
- [CppCon 2017: Dave Watson “C++ Exceptions and Stack Unwinding”](https://www.youtube.com/watch?v=_Ivd3qzgT7U)
- [C++ ABI for Itanium: Exception Handling](https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html)