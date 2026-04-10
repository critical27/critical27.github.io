---
layout: single
title: C++ Exception Handling ABI, part 2
date: 2026-04-09 00:00:00 +0800
categories: 学习
tags: C++
---

话不多说，这一篇争取把上一篇不够详尽的部分补齐。

## personality

首先，我们从上一篇没有详细介绍的 `personality` 开始。在栈展开的过程中，`libgcc` 或者 `libunwind` 作为 `unwinder` 会逐帧调用 `personality`，它作为连接 Level 1 Base ABI 和 Level 2 C++ ABI 的桥梁，需要告知 `unwinder` 以下信息：

1. 在栈展开的搜索阶段，告知 `unwinder` 当前帧是否有匹配的 `catch` 块来处理该异常
2. 在栈展开的清理阶段，告知 `unwinder` 当前帧是否需要执行相应的清理，如果需要，对应的 `landing pad` 地址是什么，以便后续 `unwinder` 进行实际跳转。

每一帧的清理逻辑，也就是上一篇所说的 `landing pad`。根据具体函数逻辑，它会完成以下三项之一：

- 无法捕获对应异常，调用已离开作用域变量的析构函数，或者调用通过 `__attribute__((cleanup(...)))` 注册的回调，然后使用 `_Unwind_Resume` 回到清理阶段
- 能捕获对应异常，先析构已离开作用域的变量，再调用 `__cxa_begin_catch`，执行 `catch` 块里的代码，最后调用 `__cxa_end_catch`
- 如果在 `catch` 中有 `rethrow`，则会先析构 `catch` 子句里定义的局部变量，再调用 `__cxa_end_catch`，然后通过 `_Unwind_Resume` 继续清理阶段

不同的语言、实现或架构可能会使用不同的 `personality` 程序。对于 C++ 而言，在 ELF 中最常见的 `personality` 实现是 `__gxx_personality_v0`。

在进一步介绍 `__gxx_personality_v0` 之前，我们需要再补充一些背景知识。

### .gcc_except_table

ELF 中，将具体语言处理异常所需的信息，例如某个 IP 指令寄存器是否位于 `try-catch` 范围内、是否存在需要执行的离开作用域变量析构等，保存到 `.gcc_except_table` 数据段中。这个数据段是 ELF 中一整块连续的字节区域，里面存放了所有函数的异常处理数据，这些数据就是上一篇所提到的 LSDA (Language-specific Data Area)。整体上逻辑关系如下：

```cpp
.gcc_except_table
├── LSDA(func1)
├── LSDA(func2)
├── LSDA(func3)
└── ...
```

每一个 LSDA 中又包含以下部分：

- `header`：`landing pad` 的基地址，`type table` 的编码格式，`call site table` 的编码格式，`action table` 的起始位置
- `call site table`：表中每个条目都保存 `[start, length, landing_pad_offset, action_record_offset]` 四个字段。当地址在 `[start, start + length)` 这段代码中出现异常时，对应的 `landing pad` 入口地址偏移量，以及第一个 `action` 在 `action table` 中的偏移量（如果没有 `action` 则为 0）。
- `action table`：每个条目有两个字段 `[switch_value, next_action_offset]`，用于表明给定范围内抛出异常对应的 `action`，比如 `cleanup/catch/noexcept`。其中 `switch_value` 用来保存每个 `catch` 的具体类型在 `type table` 中的下标（0 代表是一个 `cleanup action`），`next_action_offset` 表示下一个 `action` 在 `action table` 中的偏移量（`0` 表示没有后续）。注意一段代码对应的所有 `action` 被组织成了一个单链表。
- `type table`：保存各个类型的 `RTTI` 指针，用于检查异常对象类型是否匹配。如果指针为空，代表匹配所有类型 `catch (...)`。

几张表的关联关系如下：

函数会按 `try` 语句分割成多个代码范围，`call site table` 保存的是给定代码地址范围内出现异常时的 `landing pad` 和相应的 `action`，而每个条目中 `landing_pad_offset` 和 `action_record_offset` 可能的组合有：

- `landing_pad_offset` 为 0，则 `action_record_offset` 也 0，代表没有 `landing pad`。
- `landing_pad_offset` 不为 0，代表有 `landing pad`，其中包含这段代码的所有可能的异常操作，即包括所有的 `catch` （无论是否能捕获），以及额外的 `cleanup` 逻辑。此时若：
    - `action_record_offset` 为 0，代表当前栈帧需要进行额外清理（比如局部变量的析构）
    - `action_record_offset` 不为 0，代表有对应的 `action`，此时 `action table` 中 `action_record_offset` 对应条目即为第一个 `action`。

而 `action table` 条目中的 `switch_value` 大于 0 代表指向 `type table` 中的一个条目，等于 0 代表当前栈帧需要进行局部变量清理（对应上面 `call site table` 中 `action_record_offset` 为 0 的情况），小于 0 则是 `exception specification`，已经在现代 C++ 中很少见。

比如，如果某个代码范围内中有两个 `catch`，但都无法捕获当前异常，且需要额外清理时，LSDA 中的相关数据示意图如下：

```cpp
call site table:
    start
    length
    landing_pad_offset: 指向入口地址 其中包含两个catch以及cleanup
    action_record_offset: 假设为x 指向第一个action

action table:
    ; 第x个条目 对应第一个catch (对应type table中第m个类型)
    [switch_value = m, next_action_offset = y]
    ...
    ; 第y个条目 对应第二个catch (对应type table中第n个类型)
    [switch_value = n, next_action_offset = z]
    ...
    ; 第z个条目 switch_value为0代表是cleanup next_action_offset为0代表没有后续action
		[switch_value = 0, next_action_offset = 0]

type table:
		; 第m个条目
		第一个catch类型的RTTI指针
		...
		; 第n个条目
    第二个catch类型的RTTI指针
```

本质上 LSDA 只是一段字节流，没有显式结构体。在栈展开过程中，需要由`__gxx_personality_v0` 解析 LSDA 中的内容（至于是哪个 LSDA 则是由 unwinder 来负责查找并传递），从而确定当前帧能否处理对应异常：

1. 根据 `throw` 异常时的指令寄存器 IP，去查 `call site table`，确定当前调用点对应 `call site table` 中的哪一个条目，以及第一个 `action` 是什么
2. 遍历对应的 `action` 链表，读取每一个 `catch` 对应的类型下标。通过比较当前异常的类型信息和 `type table` 中对应的类型信息，如果匹配则表示当前帧可以处理该异常。否则根据 `next_action_offset` 跳转到下一个 `action`。
3. 如果当前帧所有 `action` 遍历完后仍不能处理该异常（`next_action_offset` 为 `0`），则通过返回值告知 `unwinder` 当前栈帧无法处理，由 `unwinder` 在 `_Unwind_RaiseException` 中继续展开上一个栈帧，并重复上述过程。

换而言之，不同代码块对应的 `landing_pad_offset` 和 `action_record_offset` 如下：

- 没有局部变量析构的非 `try` 块：`landing_pad_offset==0 && action_record_offset==0`
- 有局部变量析构的非 `try` 块：`landing_pad_offset!=0 && action_record_offset==0`，栈展开的清理阶段需要先对当前栈帧进行清理，才能继续
- 有 `__attribute__((cleanup(...)))` 的非 `try` 块：`landing_pad_offset!=0 && action_record_offset==0`，同上
- `try` 块：`landing_pad_offset!=0 && action_record_offset!=0`。`landing_pad_offset` 指向由多个 `catch` 块拼接的一段代码。`action table` 对应的条目中 `switch_value > 0`，指向 `type table` 中一个非空类型的 RTTI 指针
- 有 `catch (...)` 的 `try` 块：同上。`action table` 对应的条目中 `switch_value > 0`， `type table` 对应条目中 RTTI 指针为空（表示 `catch (...)`）
- 在有 `noexcept` 说明符的函数中，异常可能向调用方传播：`landing_pad_offset!=0 && action_record_offset!=0`。`landing pad` 指向调用 `std::terminate` 的代码块，`action table` 对应的条目中 `switch_value > 0`，且 `type table` 对应条目中 RTTI 指针为空（表示 `catch (...)`）

### __gxx_personality_v0

到这我们就可以总结 `__gxx_personality_v0` 的具体功能了：

- 通过读取当前栈帧的 LSDA 在栈展开过程中检查每个栈帧是否有匹配的 `catch`
- 搜索阶段：
    - 返回 `_URC_CONTINUE_UNWIND`：当前栈帧无法处理该异常
    - 返回 `_URC_HANDLER_FOUND`：当前栈帧能处理该异常
- 清理阶段：
    - 返回 `_URC_CONTINUE_UNWIND`：没有对应 `landing pad`，不需要额外处理
    - 返回 `_URC_INSTALL_CONTEXT`：有对应 `landing pad`，由 `unwinder` 跳转到该地址继续执行

> 上述流程没有描述各种错误路径，文章最后会涉及到一些
>

如果有 `landing pad`，`__gxx_personality_v0` 会调用 `_Unwind_SetIP` 设置 IP 寄存器，跳转到 `landing pad`。在将控制权转移到 `landing pad` 之前，`personality` 会调用 `_Unwind_SetGR` 设置两个寄存器，分别存储异常对象指针 `_Unwind_Exception *` 和类型匹配结果。

> 这两个寄存器，与架构相关，实际上是通过 `__builtin_eh_return_data_regno(0)` 和 `__builtin_eh_return_data_regno(1)`设置，x86_64下是 `%rax` 和 `%rdx`，可以参照上一篇中的例子。
>

除此了上述寄存器之外，所有 `callee-saved` 寄存器均会被恢复，而所有 `caller-saved` 寄存器则不会被恢复，其内容在控制权转移后是未定义的。

对于 `native exception`，当 `personality` 在搜索阶段返回 `_URC_HANDLER_FOUND` 时，栈帧的 LSDA 相关信息会被缓存。当 `personality` 在清理阶段被再次调用，且参数为 `actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME)` 时，`personality` 会加载缓存，无需再解析 `.gcc_except_table`。

在其他三种情况下，`personality` 必须解析 `.gcc_except_table`：

- `actions & _UA_SEARCH_PHASE`
- `actions & _UA_CLEANUP_PHASE && actions & _UA_HANDLER_FRAME && !is_native`
- `actions & _UA_CLEANUP_PHASE && !(actions & _UA_HANDLER_FRAME)`

一个简化的 `__gxx_personality_v0` 实现如下：

```cpp
_Unwind_Reason_Code __gxx_personality_v0(int version, _Unwind_Action actions, uint64_t exceptionClass, _Unwind_Exception *exc, _Unwind_Context *ctx) {
  scan_results results;
  if (actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME) && is_native) {
    auto *hdr = (__cxa_exception *)(exc+1) - 1;
    // Load cached results from phase 1.
    results.switchValue = hdr->handlerSwitchValue;
    results.actionRecord = hdr->actionRecord;
    results.languageSpecificData = hdr->languageSpecificData;
    results.landingPad = reinterpret_cast<uintptr_t>(hdr->catchTemp);
    results.adjustedPtr = hdr->adjustedPtr;

    _Unwind_SetGR(...);
    _Unwind_SetGR(...);
    _Unwind_SetIP(ctx, results.landingPad);
    return _URC_INSTALL_CONTEXT;
  }
  scan_eh_tab(results, actions, native_exception, unwind_exception, context);
  if (results.reason == _URC_CONTINUE_UNWIND ||
      results.reason == _URC_FATAL_PHASE1_ERROR)
    return results.reason;
  if (actions & _UA_SEARCH_PHASE) {
    auto *hdr = (__cxa_exception *)(exc+1) - 1;
    // Cache LSDA results in hdr.
    hdr->handlerSwitchValue = results.switchValue;
    hdr->actionRecord = results.actionRecord;
    hdr->languageSpecificData = results.languageSpecificData;
    hdr->catchTemp = reinterpret_cast<void *>(results.landingPad);
    hdr->adjustedPtr = results.adjustedPtr;
    return _URC_HANDLER_FOUND;
  }
  // _UA_CLEANUP_PHASE
  _Unwind_SetGR(...);
  _Unwind_SetGR(...);
  _Unwind_SetIP(ctx, results.landingPad);
  return _URC_INSTALL_CONTEXT;
}
```

> 注意代码中的 `_Unwind_SetGR` 和 `_Unwind_SetIP`
>

`__gxx_personality_v0` 的完整实现可以参照：

- [gcc/libstdc++-v3/libsupc++/eh_personality.cc at master · gcc-mirror/gcc](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/eh_personality.cc)
- [llvm-project/libcxxabi/src/cxa_personality.cpp at main · llvm/llvm-project](https://github.com/llvm/llvm-project/blob/main/libcxxabi/src/cxa_personality.cpp)

## .eh_frame

了解了 `personality` 后，我们再完善上一篇没有说清楚的另一个细节。即 `unwinder` 通过 `personality` 发现当前帧不能处理该异常时，该如何从当前栈帧获取到上一个栈帧，过程中相关的寄存器又该如何恢复。这部分栈展开的相关信息都保存在 ELF 的 `.eh_frame` 和 `.eh_frame_hdr` 中。

`.eh_frame` 里保存的是如何“从当前栈帧恢复到上一个栈帧”的规则（称为 CFI 指令），并不会直接保存上一个栈帧的相关寄存器是多少。换而言之，可以理解为，给定当前寄存器状态，通过这些规则，就能算出上一个栈帧的相关寄存器值。`.eh_frame` 由若干条记录组成，分为两类：

- CIE（Common Information Entry）：描述一类函数通用的规则
- FDE（Frame Description Entry）：描述某个具体函数（或代码区间）的信息

结构关系如下：

```cpp
.eh_frame:
  [CIE]
  [FDE -> 指向某个 CIE]
  [FDE -> 指向某个 CIE]
  ...
```

> `.eh_frame_hdr` 是一个二分索引加速结构，用于给定 IP 快速找到对应的 FDE，这里不展开介绍。
>

一个 CIE 包含以下字段：

- `length`
- `CIE_id`：对于 CIE 而言总是 0，用于区分 CIE 和 FDE
- `version`
- `augmentation string`
- `code_alignment_factor`：指令地址对齐单位
- `data_alignment_factor`：栈对齐的单位
- `return_address_register`：哪个寄存器代表返回地址（`%rip`）
- `augmentation data`
- `initial instructions`：CFI 指令，定义函数刚进入时的“初始栈布局规则”

每个 FDE 都有一个关联的 CIE，FDE 包含以下字段：

- `length`
- `CIE_pointer`：从当前位置减去 `CIE_pointer` 得到关联的 CIE
- `initial_location`：FDE 描述的起始代码地址
- `address_range`：FDE 描述的范围为 `[initial_location, address_range)`
- `augmentation data`
- `CFI instructions`：CFI 指令

每个 FDE 中可能会有一个关联的 LSDA 指针， 指向 `.gcc_except_table` 中对应的 LSDA。当异常发生时，`unwinder` 在栈展开过程中会通过 `.eh_frame` 找到当前 IP 对应的 FDE，然后从 FDE 中取出 LSDA 指针传给 `__gxx_personality_v0`，由 `personality` 去解析 LSDA。

> 之所以说可能是与 CIE 中的 `augmentation string` 有关，略过
>

### CFI

CIE 和 FDE 其中很多字段跟我们的问题并没有太大关系，对于栈展开，我们最关心的部分就是 FDE 中的 `instructions` 字段，即 CFI 指令（Call Frame Information instructions）。CFI 指令用来描述“在函数执行到不同位置时，如何从当前栈帧恢复出上一层栈帧的寄存器值（尤其是返回地址）”，也就是 `unwinder` 在栈展开过程中进行栈帧回溯所需的信息。汇编器会利用这些指令，组装出 `.eh_frame` 中的 CIE 和 FDE，以供 `unwinder` 使用。

首先我们理解一个核心概念 CFA（Canonical Frame Address），其定义是**调用当前函数前，调用方 caller 的 `%rsp`**。而 `.eh_frame` 的核心任务就是：不管执行到了函数的哪条指令，如何通过当前栈帧的各个寄存器，以及 `.eh_frame` 中的 CFI 指令计算出 CFA，最终计算出上一个栈帧的相关寄存器值（这里主要关心 `%rip`，`%rbp` 和 `%rsp`）。

我们用一个最简单的例子来理解下上述的流程。

```cpp
void bar() {
    throw 1;
}

void foo() {
    bar();
}
```

通过 `g++ -S -O0 test.cpp`，可以获取到对应汇编代码。其中 `bar` 的汇编如下：

```nasm
_Z3barv:
.LFB0:
    .cfi_startproc
    endbr64

    # prologue
    pushq %rbp
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq %rsp, %rbp
    .cfi_def_cfa_register 6

    # function body
    movl $4, %edi
    call __cxa_allocate_exception@PLT
    movl $1, (%rax)
    movl $0, %edx
    leaq _ZTIi(%rip), %rcx
    movq %rcx, %rsi
    movq %rax, %rdi
    call __cxa_throw@PLT

    .cfi_endproc
```

有一点基础知识需要提前说明：在 DWARF 规范（也就是 `.cfi_*` 使用的规范）中，寄存器是通过编号来表示的。在 x86-64 下：

- 寄存器 6 代表 `%rbp`
- 寄存器 7 代表 `%rsp`

`.cfi_startproc` 会标记函数开始。汇编器会在 `.eh_frame` 中新建一个 FDE。注意调用方在调用 `bar` 时，会额外将调用方 `foo` 的返回地址压栈。此时 `CFA = %rsp + 8`。（再次强调，CFA 是调用方的 `%rsp`）

之后进入 prologue。`pushq %rbp` 将上一层函数的 `%rbp` 压入栈，此时 `%rsp` 减 8，此时 `CFA = %rsp + 16`。在执行完 `pushq %rbp` 后，需要告知 `unwinder` CFA 的计算方式发生了改变，对应 CFI 指令为 `.cfi_def_cfa_offset 16`，代表更新偏移量为 16。

另外也需要告知 `unwinder` 原先的 `%rbp` 被压栈（对应 DWARF 规范中的寄存器 6），即 `%rbp` 被保存在 `CFA - 16` 处（`CFA - 8` 是返回地址），对应 CFI 指令为 `.cfi_offset 6, -16`。这样当栈展开时，依靠这个信息就可以把上一个栈帧（我们例子 `foo` 的 `%rbp`）恢复出来。

在 `movq %rsp, %rbp` 更新当前栈帧的 `%rsp` 后，需要告诉 `unwinder`，计算 CFA 的基址寄存器由调用方的 `%rsp` 换成了寄存器 6（也就是当前栈帧的 `%rbp`）。对应 CFI 指令是 `.cfi_def_cfa_register 6`，偏移量保持上一次设置的 16 不变。之后不管 `%rsp` 怎么变化（例如压入临时变量等），寻找 CFA 只需要 `CFA = %rbp + 16` 即可得到。

后续具体抛异常的代码略过。最终 `.cfi_endproc` 会标记函数结束，对应 FDE 也就完成了。

`foo` 的情况也类似，我们只补充一下 epilogue 部分：在 `popq %rbp` 之后，`%rsp` 加 8，此时 `CFA = %rsp + 8`（由于 `%rbp` 出栈，`CFA = %rbp + 16` 不再成立了）。`.cfi_def_cfa 7, 8` 指令能告知 `unwinder`，`CFA = %rsp + 8`，寄存器 7 代表 `%rsp`。

```nasm

_Z3foov:
.LFB1:
    .cfi_startproc
    endbr64

    # prologue
    pushq %rbp
    .cfi_def_cfa_offset 16
    .cfi_offset 6, -16
    movq %rsp, %rbp
    .cfi_def_cfa_register 6

    call _Z3barv
    nop

    # epilogue
    popq %rbp
    .cfi_def_cfa 7, 8
    ret
    .cfi_endproc
```

当 `bar` 函数 `throw` 时，会调用到 `__cxa_throw` 函数。`unwinder` 会获取当前 CPU 的 IP 寄存器 `%rip`，根据 `%rip` 找到对应的 FDE 记录。接下来 `unwinder` 会回放一遍从该函数开头（`.cfi_startproc`）一直到当前抛出异常所在的 IP 地址为止所有的 `.cfi_*` 指令。通过回放，`unwinder` 可以算出当前的 CFA 是多少。知道 CFA 之后，如何获取调用方 `foo` 在调用该函数时的相关寄存器状态呢？

答案是 `unwinder` 通过 `.cfi_offset 6, -16` 这条指令就能算出 `%rbp` 和 `%rip`：

- `%rbp`：调用者的 `%rbp`（寄存器 6）被保存在内存中 `CFA - 16` 的位置。
- `%rip`：而对于调用者的 `%rip`，也就是返回地址是在进入到被调用函数之前就已经被压栈，因此调用者的返回地址保存在 `CFA - 8` 的位置。

`unwinder` 将相关寄存器恢复到调用方调用当前函数前的状态，也就是 `foo` 调用 `bar` 之前的状态：

- `%rbp = CFA - 16`
- `%rip = CFA - 8`
- `%rsp = CFA`

此时 `unwinder` 已经从 `bar` 回溯到了 `foo`。之后可以继续进行栈展开，即使用更新过的 `%rip`，去恢复 `foo` 的调用方的相关寄存器状态。

到这我们就理解了 `unwinder` 如何进行栈帧回溯了。这里我们再进一步思考这样一个问题：为什么栈展开过程中必须恢复 `callee-saved` 相关寄存器？又为什么无需恢复 `caller-saved` 相关寄存器？

当 `foo` 调用 `bar` 时，会保存 `caller-saved` 相关寄存器，例如 `%rax`、`%rcx`、`%r8`-`%r11` 等。当越过 `call` 这条边界后，这些寄存器里的数据都变成垃圾了。而当从 `bar` 栈展开回到 `foo` 时，对于 `foo` 而言只不过相当于又跨回了这条边界，这些寄存器对于调用方 `foo` 无关紧要，`unwinder` 也不需要去恢复它们。

而 `callee-saved` 相关寄存器就不一样了。调用方 `foo` 期望无论何时，无论被调用方 `bar` 正常返回还是异常发生时，`callee-saved` 寄存器都能保持不变。然而当异常发生时，`bar` 的正常流程被打断了，`bar` 并没有机会去执行 `epilogue` 来恢复这些寄存器。这也就是 `unwinder` 需要通过 `.eh_frame` 中的 CFI 指令，代替被调用方 `bar` 来完成它没有完成的义务，即将这些调用方所期望的 `callee-saved` 寄存器一一恢复。

### 如何查看.eh_frame

可以通过如下方式比对查看`.eh_frame` 数据段：

```bash
$ readelf -wF a.out

000000a8 000000000000001c 00000024 FDE cie=00000088 pc=0000000000001169..0000000000001198
   LOC           CFA      rbp   ra
0000000000001169 rsp+8    u     c-8
000000000000116e rsp+16   c-16  c-8
0000000000001171 rbp+16   c-16  c-8

000000c8 000000000000001c 000000cc FDE cie=00000000 pc=0000000000001198..00000000000011a8
   LOC           CFA      rbp   ra
0000000000001198 rsp+8    u     c-8
000000000000119d rsp+16   c-16  c-8
00000000000011a0 rbp+16   c-16  c-8
00000000000011a7 rsp+8    c-16  c-8
```

上面两段就是 `bar` 和 `foo` 解析之后的 FDE，可以对照最终二进制文件的地址加深理解（汇编代码都是一致的，只不过起始地址有所不同）。

```bash
$ objdump -d a.out

0000000000001169 <_Z3barv>:
    1169:       f3 0f 1e fa             endbr64
    116d:       55                      push   %rbp
    116e:       48 89 e5                mov    %rsp,%rbp
    1171:       bf 04 00 00 00          mov    $0x4,%edi
    1176:       e8 e5 fe ff ff          call   1060 <__cxa_allocate_exception@plt>
    117b:       c7 00 01 00 00 00       movl   $0x1,(%rax)
    1181:       ba 00 00 00 00          mov    $0x0,%edx
    1186:       48 8d 0d 13 2c 00 00    lea    0x2c13(%rip),%rcx        # 3da0 <_ZTIi@CXXABI_1.3>
    118d:       48 89 ce                mov    %rcx,%rsi
    1190:       48 89 c7                mov    %rax,%rdi
    1193:       e8 d8 fe ff ff          call   1070 <__cxa_throw@plt>

0000000000001198 <_Z3foov>:
    1198:       f3 0f 1e fa             endbr64
    119c:       55                      push   %rbp
    119d:       48 89 e5                mov    %rsp,%rbp
    11a0:       e8 c4 ff ff ff          call   1169 <_Z3barv>
    11a5:       90                      nop
    11a6:       5d                      pop    %rbp
    11a7:       c3                      ret
```

## Misc

最后再补充一些零碎的信息。

### exception propagation

一些会影响异常传播的编译器参数：

- `fno-exceptions -fno-asynchronous-unwind-tables`: `.eh_frame` 和 `.gcc_except_table` 都不存在
- `fno-exceptions -fasynchronous-unwind-tables`: `.eh_frame` 存在，`.gcc_except_table` 不存在
- `fexceptions`: `.eh_frame` 和 `.gcc_except_table` 都存在（默认情况）

当一个异常从当前函数向调用方传播时，如果：

- 没有 `.eh_frame`： `_Unwind_RaiseException` 返回 `_URC_END_OF_STACK`。`__cxa_throw` 调用 `std::terminate`
- 有 `.eh_frame` 但当前栈帧没有对应 LSDA：透传，不调用局部变量析构函数
- 有 `.eh_frame` 且当前栈帧有对应 LSDA，但 `call site table` 中找不到抛异常处 IP 对应条目：`__gxx_personality_v0` 调用 `__cxa_call_terminate` 或者 `std::terminate`。这表明当前 IP 不在可抛出异常的范围内，找不到对应的 `landing pad`，只能退出
- 有 `.eh_frame` 且当前栈帧有对应 LSDA，`call site table` 中找到了抛异常处 IP 对应条目：执行可能的清理并展开到父帧。此时 `landing pad` 为 0 表明当前栈帧无需额外处理，继续栈展开。而 `landing pad` 非 0 则表示有清理或者 `catch`，如果无法捕获异常则会调用 `_Unwind_Resume` 继续栈展开。

而当一个异常从当前 `noexcept` 函数向调用方传播时：

- `fno-exceptions -fno-asynchronous-unwind-tables`：调用 `std::terminate`
- `fno-exceptions -fasynchronous-unwind-tables`：透传，不会调用局部变量析构函数
- `fexceptions`： 调用 `std::terminate`

当 `std::terminate` 被调用时，会有一个诊断信息，形如 `terminate called after throwing an instance of 'int'`。此时没有 `stack trace`，如果进程会处理 `SIGABRT` 信号，`signal handler` 可能会获得 `stack trace`。

### noexcept

最后再看一下 `noexcept` 的一些示例。我们仍然用刚才示例代码：

```cpp
void bar() {
    throw 1;
}

void foo() {
    bar();
}

int main() {
    foo();
}
```

`bar` 抛出的异常，最终调用到 `__cxa_throw`，调用 `_Unwind_RaiseException` 开始栈展开，过程中会调用 `__gxx_personality_v0` 查看是否有栈帧能处理这个异常。由于我们代码中压根没有 `try/catch` 语句，也不需要额外清理，因此编译器并不会生成 LSDA。搜索阶段一路回溯到栈底也找不到能捕获异常的 `catch handler`，`_Unwind_RaiseException` 返回 `_URC_END_OF_STACK`，由 `__cxa_throw` 调用 `std::terminate`。core dump 如下：

```cpp
>>> bt
#0  0x00007c2fa169eb2c in pthread_kill () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007c2fa164527e in raise () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007c2fa16288ff in abort () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007c2fa1aa5ff5 in ?? () from /lib/x86_64-linux-gnu/libstdc++.so.6
#4  0x00007c2fa1abb0da in ?? () from /lib/x86_64-linux-gnu/libstdc++.so.6
#5  0x00007c2fa1aa5a55 in std::terminate() () from /lib/x86_64-linux-gnu/libstdc++.so.6
#6  0x00007c2fa1abb391 in __cxa_throw () from /lib/x86_64-linux-gnu/libstdc++.so.6
#7  0x000060248ae80198 in bar() ()
#8  0x000060248ae801a5 in foo() ()
#9  0x000060248ae801b5 in main ()
```

而如果我们把 `bar` 函数添加上 `noexcept` 关键字，可以发现 core dump 有所不同。

```cpp
void bar() noexcept {
    throw 1;
}

void foo() {
    bar();
}

int main() {
    foo();
}
```

在栈展开过程中，`unwinder` 调用 `__gxx_personality_v0` 处理这个栈帧时，它发现一个 `noexcept` 函数中抛出了异常，会直接调用 `__cxa_call_terminate`。

```cpp
>>> bt
#0  0x00007f164189eb2c in pthread_kill () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00007f164184527e in raise () from /lib/x86_64-linux-gnu/libc.so.6
#2  0x00007f16418288ff in abort () from /lib/x86_64-linux-gnu/libc.so.6
#3  0x00007f1641ca5ff5 in ?? () from /lib/x86_64-linux-gnu/libstdc++.so.6
#4  0x00007f1641cbb0da in ?? () from /lib/x86_64-linux-gnu/libstdc++.so.6
#5  0x00007f1641ca58e6 in __cxa_call_terminate () from /lib/x86_64-linux-gnu/libstdc++.so.6
#6  0x00007f1641cba8ba in __gxx_personality_v0 () from /lib/x86_64-linux-gnu/libstdc++.so.6
#7  0x00007f1641bf4b06 in ?? () from /lib/x86_64-linux-gnu/libgcc_s.so.1
#8  0x00007f1641bf51f1 in _Unwind_RaiseException () from /lib/x86_64-linux-gnu/libgcc_s.so.1
#9  0x00007f1641cbb384 in __cxa_throw () from /lib/x86_64-linux-gnu/libstdc++.so.6
#10 0x0000579e7a320198 in bar() ()
#11 0x0000579e7a3201a5 in foo() ()
#12 0x0000579e7a3201b5 in main ()
```

要理解这个路径，我们需要先看看对应程序的 `.gcc_except_table`。

```bash
$ readelf -x .gcc_except_table a.out

Hex dump of section '.gcc_except_table':
  0x00002154 ffff0100                            ....
```

可以看到只有四个字节，实际是 LSDA 的 header：

1. `ff`：`landing pad` 的基地址 —— 表示没有特定的 landing pad 基址
2. `ff`：`type table` 的编码格式 —— 表示没有类型信息表（没有 `catch`，不需要做 RTTI 类型匹配）。
3. `01`：`call site table` 编码，`01` 表示采用 `uleb128` 编码。
4. `00`：`call site table` 长度为 0

> • In GCC, for a `noexcept` function, a possibly-throwing call site unhandled by a try block does not get an entry in the `.gcc_except_table` call site table. If the function has no try block, it gets a header-only `.gcc_except_table` (4 bytes)
• In Clang, there is a call site entry calling `__clang_call_terminate`. The size overhead is larger than GCC's scheme. Improving this requires LLVM IR work
>

由于 `call site table` 中没有任何有效条目，在两阶段栈展开过程中，`__gxx_personality_v0` 都会将该帧的搜索结果设置为 `found_terminate`。

```cpp
while (p < info.action_table) {
  _Unwind_Ptr cs_start, cs_len, cs_lp;
  _uleb128_t cs_action;

  // Note that all call-site encodings are "absolute" displacements.
  p = read_encoded_value(0, info.call_site_encoding, p, &cs_start);
  p = read_encoded_value(0, info.call_site_encoding, p, &cs_len);
  p = read_encoded_value(0, info.call_site_encoding, p, &cs_lp);
  p = read_uleb128(p, &cs_action);

  // The table is sorted, so if we've passed the ip, stop.
  if (ip < info.Start + cs_start)
    p = info.action_table;
  else if (ip < info.Start + cs_start + cs_len) {
    if (cs_lp)
      landing_pad = info.LPStart + cs_lp;
    if (cs_action)
      action_record = info.action_table + cs_action - 1;
    goto found_something;
  }
}

// If ip is not present in the table, call terminate.  This is for
// a destructor inside a cleanup, or a library routine the compiler
// was not expecting to throw.
found_type = found_terminate;
goto do_something;
```

完整流程是：

1. 在搜索阶段，设置为 `found_terminate`，此时 `landing_pad` 为 0，将当前结果缓存，并返回 `_URC_HANDLER_FOUND`，代表找到了catch handler（尽管实际的 `landing pad` 是 terminate）。

    ```cpp
    if (actions & _UA_SEARCH_PHASE) {
      if (found_type == found_cleanup)
        CONTINUE_UNWINDING;

      // For domestic exceptions, we cache data from phase 1 for phase 2.
      if (!foreign_exception) {
        save_caught_exception(ue_header, context, thrown_ptr, handler_switch_value,
                              language_specific_data, landing_pad, action_record);
      }
      return _URC_HANDLER_FOUND;
    }
    ```

2. 在清理阶段，通过读取缓存结果，再次设置为 `found_terminate`，最终也就调用了 [__cxa_call_terminate](https://github.com/gcc-mirror/gcc/blob/master/libstdc%2B%2B-v3/libsupc%2B%2B/eh_personality.cc#L691)。

    ```cpp
    // Shortcut for phase 2 found handler for domestic exception.
    if (actions == (_UA_CLEANUP_PHASE | _UA_HANDLER_FRAME) && !foreign_exception) {
      restore_caught_exception(ue_header, handler_switch_value,
                               language_specific_data, landing_pad);
      found_type = (landing_pad == 0 ? found_terminate : found_handler);
      goto install_context;
    }
    ```


最后，我们可以从 core dump 看到整个 Itanium C++ ABI 异常处理的各个关键组件：

- `__cxa_throw` 是 Itanium C++ ABI 定义的接口，`libstdc++` 提供了具体实现。
- `_Unwind_RaiseException` 是 Itanium Base ABI 定义的栈展开接口，`libgcc_s` 提供了具体实现，基于 DWARF 展开信息。
- `__gxx_personality_v0` 负责：
    - 在栈展开过程中检查每个栈帧是否有匹配的 catch
    - 决定是否执行 landing pad
- `__cxa_call_terminate`
- 最终 `abort()`，则是在 `glibc` 中

相关内容整理得差不多了，大多数内容都是通过阅读 MaskRay 的博客重新消化输出的，不免会有不少疏漏错误。但整个过程还是学到了不少，有点意思。

## Reference

- [C++ exception handling ABI | MaskRay](https://maskray.me/blog/2020-12-12-c++-exception-handling-abi)
- [Stack unwinding | MaskRay](https://maskray.me/blog/2020-11-08-stack-unwinding)
- [CppCon 2017: Dave Watson “C++ Exceptions and Stack Unwinding”](https://www.youtube.com/watch?v=_Ivd3qzgT7U)