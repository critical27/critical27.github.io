---
layout: single
title: Linkers, Loaders and Shared Libraries in Linux and C++
date: 2026-09-06 00:00:00 +0800
categories: 学习
tags: Linux C++
---

这篇文章整理自 CppCon 2023 的演讲《Linkers, Loaders and Shared Libraries in Windows, Linux, and C++》，它主要围绕动态库，介绍了 GOT、PLT、PIC 等概念，并对比了它们在 Windows 和 Linux 上的行为。这个演讲不仅能让你对这些概念有一个整体的认知，更重要的是它阐述了这些东西是从何而来，为了解决什么问题的，它们又会带来什么问题。鉴于我主要的工作经验都集中于 Linux，所以我对整体内容做了一些调整，只在最后会进行一些简单对比。

文章中涉及的术语，可能会根据中英文语境不同交替使用。

- 链接器：Linker
- 动态加载器：Loader, Dynamic Loader
- 动态库：Shared Library, Dynamic Shared Library, Shared Object, Dynamic Object, Dynamic Shared Object (DSO), Dynamic Load Library (DLL)
- Binary：即 ELF Binary，其中包括可执行文件和动态库
- 符号：Symbol，包括函数和全局变量
- 位置无关代码：Position-Independent Code (PIC)

## Introduction

构建可执行文件的过程的第一阶段从源文件开始。编译器将源文件处理成目标文件（object file），而目标文件本质上只是若干 section。最典型的两个 section 是包含指令的代码段 `.text` 和包含程序数据的数据段 `.data`，除此之外还有许多其他类型的 section。第二阶段中，linker 首先把来自不同目标文件的同名 section 合到一起，拼接成一个更大的 section；然后重新排列这些 section，并使运行时权限要求相近的 section 彼此相邻。最后，可执行文件在运行时，由 dynamic loader 将这些相邻的数据块，也就是 segment，按照 page 对齐的边界映射到内存，并相应地调整各个内存 page 的权限。例如，存放代码的 page 需要读取和执行权限，因此 loader 映射完代码 segment 后，会将其权限设置为可读、可执行。

![figure](<{{'/archive/SharedLibrary-1.png' | prepend: site.baseurl}}>)

代码中通常存在大量函数调用。最简单的情况是可执行文件调用自身实现的函数，但实际情况并不总是如此。通常，一个进程可以将动态库映射到自己的地址空间，并调用其中实现的函数，动态库本身也可以调用其他动态库所实现的函数。

![figure](<{{'/archive/SharedLibrary-2.png' | prepend: site.baseurl}}>)

调用其他动态库的函数，也是在 loader 的帮助下完成的。要理解整个过程，我们先从重定位说起。

### Relocation

假设有一段代码要调用函数 `foo`，而 `foo` 实现在另一个动态库中。代码段本身并不包含字符串 `foo`，而是包含一条调用指令，其目标地址在链接时仍然未知。因此，代码段中只能暂时放入一个占位符，例如一串零。

![figure](<{{'/archive/SharedLibrary-3.png' | prepend: site.baseurl}}>)

除此之外，linker 还会生成一个 section，用来描述这些占位符该如何被修正，即重定位（relocation）。重定位本质上是交给 loader 的一项小任务，相当于 linker 告诉 loader：“请在加载这个 binary 时，找到函数 `foo`，找到后用它的地址覆盖这个占位值。”

![figure](<{{'/archive/SharedLibrary-4.png' | prepend: site.baseurl}}>)

如果这个占位符位于代码段中，这种重定位在实践中的确存在，称为 text relocation，也称为 direct-access relocation。但通常不会使用这种重定位方法，原因有二。第一，修改代码段会使这个 binary 无法在多个进程之间共享。第二，这种重定位需要针对每个调用点分别执行，而不是每个函数只执行一次。如果同一个函数有成千上万个调用点，loader 加载时间会很久。

为了减少重定位的开销，常见的方法就是使得同一个函数的多个调用，都指向同一个位置。这样 loader 只需要进行一次解析，并将结果写入到该位置，就能使所有该函数的调用完成重定位。在这种方案中，loader 读取重定位信息并完成符号解析后，只需要将 `foo` 的地址写入这一个位置，而不必修改所有调用点。显然，这种设计牺牲了一部分运行时性能（和虚函数一样，多一次跳转），以期大幅节省加载时间。另一个优势是能够保证代码段只读和位置无关，从而使动态库可以被多个进程共享。

![figure](<{{'/archive/SharedLibrary-5.png' | prepend: site.baseurl}}>)

binary 中有一整个 section 用来存放这样的占位符。在 Windows 中，它称为 IAT，即导入地址表（Import Address Table）；在 Linux 中，与之对应的是 GOT，即全局偏移表（Global Offset Table）。

## Linux Shared Library

Linux ELF binary 通过两个 section 记录相关信息：

- `.dynamic` section 包含一份原始的动态库名称列表。
- `.dynsym` section 是更为人熟知的符号表，其中不仅包含需要导入的符号，还包含该 binary 中的参与动态链接的全局符号（包含全局变量和函数）。

![figure](<{{'/archive/SharedLibrary-6.png' | prepend: site.baseurl}}>)

需要导入的符号在符号表 `.dynsym` 中被标记为 undefined。链接时它们仍未定义，需要由 loader 找到相应的定义并完成正确绑定。

![figure](<{{'/archive/SharedLibrary-7.png' | prepend: site.baseurl}}>)

通过 `.dynamic` 和 `.dynsym`，一个 ELF binary 告知 loader：“这是我希望你加载到进程中的动态库，请从这些动态库中帮我找到以下符号。”

需要注意的是，**在 Linux 中，`.dynsym` 符号表中的符号可以由 `.dynamic` 中的动态库中的任意一个提供**。而 loader 会在符号解析时按照一定顺序来查找这些动态库，直到搜到第一个提供这个符号的动态库。

从积极的一面来看，这可能正是开发者有意需要的行为。例如，你引入了某个第三方动态库，但希望用自己的实现覆盖其中的某个函数。这种机制就是 Interposition。最常见的使用 Interposition 的例子就是通过链接各种 malloc 库，比如 jemalloc、tcmalloc 等，从而覆盖默认 malloc。

### Interposition

Interposition 是指从一个 binary 中覆盖另一个 binary 内符号的能力，它是 Linux 执行模型的基石之一。

我们前面提到，在符号解析时，loader 会按照一定顺序来查找这些动态库。具体来说，Linux 中动态库的搜索顺序是广度优先。假设有一个可执行程序加载了 `lib1` 和 `lib2`，而它们又分别加载了 `lib3`、`lib4` 和 `lib5`。现在假设 `lib5` 想要使用符号 `foo`。loader 搜索符号 `foo` 的顺序是：首先搜索可执行程序，然后依次搜索 `lib1`、`lib2`、`lib3` 和 `lib4`，最后才搜索 `lib5`。即使 `lib5` 自己实现了 `foo`，在 loader 搜索到 `lib5` 之前，其他所有 binary 都有机会 interpose `foo`。尤其需要注意的是，可执行程序的搜索优先级高于当前动态库。

![figure](<{{'/archive/SharedLibrary-8.png' | prepend: site.baseurl}}>)

这是默认行为，但可以通过链接的选项进行调整。最直接的方式是在链接 `lib5` 时使用 linker 选项 `-Bsymbolic`。`-Bsymbolic` 告诉 linker：解析 `lib5` 中的未定义符号时，应当先在 `lib5` 内部查找，然后再按照通常的广度优先顺序继续搜索。

这里还需要介绍一下 `LD_PRELOAD`。`LD_PRELOAD` 是一个环境变量；如果它存在并包含一组动态库名称，loader 会在加载可执行程序之后、加载任何依赖库之前加载这些动态库。它们也会在这个位置加入广度优先搜索顺序。

Interposition 是很多 profiling、tracing、hook 工具的基础。但它带来的影响也很明显：linker 不能轻易假设“我看到的这个函数实现就是最终会被调用的实现”，函数调用行为也更隐蔽。

Interposition 会在 loader 符号解析时发生，接下来我们看一下什么时候会进行符号解析。这里仍然使用上面的依赖关系树，假设 `lib5` 想要导入符号 `foo`，而可执行程序想要导入符号 `bar`。

![figure](<{{'/archive/SharedLibrary-9.png' | prepend: site.baseurl}}>)

可执行程序的符号解析会在链接时进行，如果 linker 无法解析 `bar`，就会拒绝链接该可执行程序。

但动态库并非如此：即使 linker 找不到 `foo`，仍然会正常链接 `lib5`。因为在 linker 看来，`foo` 的实现可能位于可执行程序中，而链接动态库时它没有机会发现这个实现。

默认行为由 linker 选项 `--allow-shlib-undefined` 控制。最简单的干预方式是在链接可执行程序时使用 `--no-allow-shlib-undefined`，此外也有其他可以应用于动态库的选项，比如 `-z defs` 以及 `--no-undefined`。

### Position-Independent Code

还有一个问题是，作为一个动态库，我们自然希望能被不同的可执行程序加载。而同一个动态库被多次加载时，其基地址是不同的。那么该如何保证，所有加载这个动态库的进程都能正确调用动态库的相关方法呢？

答案就是 Position-Independent Code (PIC)，即位置无关代码。如果一个动态库是由没有指定 `-fPIC` 的 object 文件构造出来的，那么在链接这个动态库的时候就会见到这个报错。

```text
error: relocation R_X86_64_PC32 against symbol `global' can not be
used when making a shared object; recompile with -fPIC
```

也就是说，动态库的所有代码必须是位置无关的。接下来我们会逐步解释位置无关是什么。

通常一个 binary 中可能看到如下三种调用形式：

![figure](<{{'/archive/SharedLibrary-10.png' | prepend: site.baseurl}}>)

1. 调用一个固定的硬编码地址：这种调用形式显然不具备位置无关性。如果整个 binary 被加载到另一个地址，写死在代码中的调用目标就会失效。

2. PC-relative call：在这种调用形式中，目标地址相对于指令寄存器计算，因此它是位置无关的；但这种方式不支持 interposition，因为 loader 没有可以介入并替换函数实现的位置。因此，它只用于 hidden symbol，后文还会进一步讨论。

3. Position-Independent Code：通过 GOT 达成位置无关。即所有函数调用，都指向 GOT 中的一个条目。每个 GOT 中的条目，根据不同情况，会在不同时机被 loader 修改，最终会指向实际函数的地址。

我们分类讨论一下动态库的所有代码是如何做到位置无关的：

1. 调用当前动态库定义的函数，且该函数是 hidden visibility，对该函数的调用会使用 PC-relative call，从而达成位置无关。
2. 调用当前动态库定义的函数，且该函数是 default visibility，对该函数的调用会使用 Position-Independent Code，即有对应的 GOT 条目。记作 Case 2。
3. 调用当前动态库未定义的函数，对该函数的调用会使用 Position-Independent Code，即有对应的 GOT 条目。记作 Case 3。

> 为了方便理解，关于 visibility 会在后面再介绍，现在只需要知道动态库默认使用 default visibility。

第一种情况比较简单，由于函数地址直接可以相对于指令寄存器计算，因此不需要符号解析或者重定位，也不需要修改 GOT。而后两种情况都需要 loader 介入，由 loader 修改 GOT 中对应的条目，但二者修改时机不同。此外，后两种情况下，同一个符号的多个调用点，都会指向同一个 GOT 条目。

接下来，为了更好地理解后两种情况是如何使用 GOT 的，我们会先描述一个简化场景，即没有发生 Interposition，**注意该场景和真实工作原理不一样**：

- 对于 Case 2，在链接之后，加载之前，GOT 中的条目指向该函数在当前动态库的一个偏移地址。加载动态库时，loader 可以根据该动态库最终加载的基地址，加上 GOT 条目中的偏移地址，得到最终的函数地址并修改 GOT 条目。
- 对于 Case 3，在链接之后，加载之前，GOT 中的条目只能填写未知。linker 通过重定位信息告知 loader 在加载这个动态库时，进行符号解析并获取对应定义的地址，最终修改对应 GOT 条目。

虽然真实的工作原理并不像上面说的那么简单，但背后的核心原理是一致的：PIC 会把后两种情况的函数调用变成间接调用，且 PIC 把需要修改的绝对地址集中放到 GOT 这类可写数据区，使代码段本身不必被修改，从而达成位置无关（主要指代码段）。如果动态库被加载到不同地址，GOT 中会被 loader 修改。因此，整个 binary 并不是位置无关的，但代码段始终是位置无关的。

> 真实情况是，对于可能被 interposition 的函数或者未定义的函数，函数调用会通过 PLT + GOT 共同完成间接调用，不是只依赖 GOT。具体原理参照下面延迟绑定部分。

### Summary of first half

到这我们总结一下前半部分，dynamic loader 在加载动态库的时候，会完成以下工作：
1. 根据 `.dynamic` 得到其依赖的其他动态库。
2. 根据 `.dynsym` 符号表得到需要导入的符号，按照广度优先的顺序进行查找这些动态库，找到第一个提供该符号的动态库。
3. 对于当前动态库定义的方法（default visibility），根据加载动态库的基地址，以及 GOT 条目中的偏移地址，修改 GOT 条目，这样可执行文件或者动态库就能调用当前动态库定义的方法了。
4. 对于当前动态库需要调用的其他未定义的方法，按广度优先查找相关可执行文件以及动态库，获取到最终地址后修改 GOT 条目。

这不代表 loader 就完美无缺了。想象一下，如果一个动态库需要导入成千上万个符号，那么在加载这个动态库的时候，其开销不可忽视。但当我们去使用一些比较大的可执行文件时，比如执行 `gcc --version`，却发现它运行得很快，这就涉及到下面要讨论的内容：延迟绑定。

### Lazy Binding

延迟绑定将符号解析推迟到函数第一次被调用时，这样做的原因是：一个 binary 可能包含许多符号，但某次运行实际只会用到其中少数几个。如果预先解析全部符号，大量解析工作都会被浪费，并可能造成明显的加载延迟。因此，ELF 的设计者提出了延迟绑定 Lazy Binding：只在某个符号第一次真正被使用时解析它。

Linux 默认使用延迟绑定。可以通过编译器选项、作用于单个函数的 attribute，或者环境变量改变这一行为。

这部分在之前介绍 [动态库加载](https://critical27.github.io/%E5%AD%A6%E4%B9%A0/explore-x86_64-main()-part-2/#%E5%BB%B6%E8%BF%9F%E7%BB%91%E5%AE%9A) 时有过详细分析，这里描述关键步骤。大致原理是在加载动态库之后，GOT 条目会指向符号解析的代码。当第一次函数调用时，会通过 PLT + GOT 跳转到符号解析流程，完成符号解析后，将解析结果写回 GOT 条目。

---
第一次调用

首先，函数调用不再像前面那样直接指向 GOT 条目。而是又增加了一层间接调用，调用函数 `foo` 首先会跳转到 PLT (Procedure Linkage Table) 中的一个条目，比如图中的第 n 个 PLT 条目 `PLTn`。每个条目都是 linker 精心设计好的一个桩函数，其中第一条指令会跳转到对应的 GOT 条目保存的地址，神奇的是，该地址就是对应桩函数的第二条指令。

![figure](<{{'/archive/SharedLibrary-11.png' | prepend: site.baseurl}}>)

而桩函数从第二条指令 `pushq $n` 会告知 loader 要解析的是 `.rela.plt` 的第几个重定位项，为符号解析准备参数。之后跳转到 `PLT0`，最终调用 `_dl_runtime_resolve` 进行符号解析。注意该函数是 loader 中的一个函数，且 loader 也被加载到了当前进程的地址空间中。在进行符号解析后，loader 会把解析结果覆盖对应的 GOT 条目，并把控制流转交给最开始调用的函数 `foo`。此时 PLT 和 GOT 的状态如下。

![figure](<{{'/archive/SharedLibrary-12.png' | prepend: site.baseurl}}>)

---
第二次调用

第一次调用之后，后续调用流程就简单得多。桩函数的第一个跳转指令仍然会跳转到对应的 GOT 条目，此时 GOT 条目中保存的已经是函数的实际地址，因此函数调用变成通过 PLT 的一次间接调用。

---

整个流程中，linker 和 loader 通过精心设计的 PLT 进行了多次跳转，最终完成符号解析。其精妙之处在于：
- 和只使用 GOT 时一样，每个函数都只需要一个入口
- GOT 条目中保存的地址区分了是否完成了符号解析，指向 PLT 条目时代表还未解析的 slow path，而指向实际地址代表已经解析
- 只需要修改 GOT，不需要修改代码段，保证代码段位置无关

另外，延迟绑定引入之后，无论是 Case 2 (default visibility) 还是 Case 3 (undefined function)，其流程都是一模一样的，唯一的区别只在于符号解析时，Case 2 中可能会找到当前动态库的定义，而 Case 3 只能找到其他动态库的定义。

> 延迟绑定可以被以下因素影响：指定 linker 参数 `-fno-plt` 或者 `-z now`，每个函数可以指定 attribute `noplt`，或者使用 `LD_BIND_NOW` 环境变量。

### Comparing Func Ptrs

理解了延迟绑定和 PLT 之后，我们再看一个具体的例子。C++ 标准 [expr.eq] §3.2 关于比较函数指针是否相等有这样一句话：“… if the pointers are both null, both point to the same function, or both represent the same address (6.8.2), they compare equal.”

但在动态库中，Case 2 和 Case 3 的函数调用都指向 PLT 条目，而每个 binary 都有各自的 PLT，那么不同 binary 中的函数指针又如何保证比较结果相等呢？为了实现这一点，Linux 做了大量额外工作。

我们分成两种情况来看：

1. 可执行文件获取其他动态库定义的函数的指针
2. 动态库获取可执行文件定义的函数的指针

第一种：可执行文件获取其他动态库定义的函数的指针。

假设一个可执行程序和动态库 `lib1` 都引用了 `foo`，且二者都获取其地址。
注意到可执行程序的符号表 `.dynsym` 中，符号 `foo` 虽然是 undefined symbol，但 `st_value` 值指向其对应的 PLT 条目 `foo@plt`。当 loader 需要解析 `foo` 的地址时，会从可执行程序的符号表中读取这个条目。通过这种方式，无论可执行程序还是 `lib1`，得到的 `foo` 地址都是可执行程序中的 `foo@plt`，且该地址是一个有效的函数指针，调用 `foo@plt` 最终也会执行 `foo`。

![figure](<{{'/archive/SharedLibrary-13.png' | prepend: site.baseurl}}>)

> Loader 使用可执行程序的 `.dynsym` 条目解析 `lib1` 中的 `&foo`。即使 `foo` 尚未被调用，`p1` 和 `p2` 也都会变成 `foo@plt`。而 Lazy Binding 仍然可以在之后发生。

第二种：动态库获取可执行文件定义的函数的指针。

可执行程序中获取的 `&foo` 就是 `foo` 的真实地址，而动态库 `lib1.so` 被加载时，dynamic loader 会解析其中的未定义符号，并按照前面介绍的符号解析顺序找到可执行程序中的 `foo`。

![figure](<{{'/archive/SharedLibrary-14.png' | prepend: site.baseurl}}>)

### Symbol Visibility

到这我们已经对动态库有了比较全面的了解，这里会补充一下前面提到的符号可见性 visibility。

ELF binary 中有一张包含全局符号的符号表 `.dynsym`，这些符号都有可能被其他 binary 获取到。其中的符号也可以被标记为 undefined，从而要求 loader 导入它们。

除此之外，还有另一个控制符号可见性的维度是 visibility。每个符号的 visibility 可以是 default/protected/hidden。比如，前面有提到 hidden visibility 的符号才会使用 PC-relative call，而具有 default visibility 的符号需要经过更长的 PLT/GOT 路径。

![figure](<{{'/archive/SharedLibrary-15.png' | prepend: site.baseurl}}>)

具有 default visibility 的符号可以被其他 binary 使用，会被 Interposition，并且会出现在 GOT 和符号表 `.dynsym` 中。hidden visibility 则不具备上述任何特性。protected visibility 与 default visibility 类似，区别在于它不会被 Interposition。

![figure](<{{'/archive/SharedLibrary-16.png' | prepend: site.baseurl}}>)

由于默认情况下，所有符号都会经过较长的 PLT/GOT 路径，但实际上很大一部分符号并没有必要保留 default visibility。可以看到 GCC 的 [wiki](https://gcc.gnu.org/wiki/Visibility) 中至少列出来把符号列为 hidden 的好处：

- It very substantially improves load times of your DSO (Dynamic Shared Object).
- It lets the optimiser produce better code.
- It reduces the size of your DSO by 5-20%.
- Much lower chance of symbol collision.

## Difference between Linux and Windows

最后，我们再简单对比下两种系统关于动态库的一些行为差异。

![figure](<{{'/archive/SharedLibrary-17.png' | prepend: site.baseurl}}>)

可以从可执行程序中覆盖动态库的符号吗？

- Linux：可以。
- Windows：不可以。

> Linux 默认支持 Interposition，而 Windows 默认更偏向显式导入导出，所以这类行为少很多。

如何创建进程级的 singleton？

- Windows：常规的 singleton 设计模式会为每个动态库分别创建一个 singleton。正确做法是从一个动态库中导出 singleton 变量，并将所有 binary 链接到这个动态库。
- Linux：直接将它放在可执行程序中即可。

动态库之间可以存在循环依赖吗？

- Linux：可以。
- Windows：不可以。

默认使用 Lazy binding 吗？

- Linux：是。
- Windows：否。

## Summary

这篇文章围绕动态库，介绍了动态库如何做到位置无关，如何通过 GOT 实现了 Interposition，以及如何通过 PLT + GOT 完成了延迟绑定。整理下来收获颇丰，推荐去看下链接里的演讲！

## Reference

[Linkers, Loaders and Shared Libraries in Windows, Linux, and C++ - Ofek Shilon - CppCon 2023](https://www.youtube.com/watch?v=_enXuIxuNV4)

[explore x86_64 main(), part 2 - Braid](https://critical27.github.io/%E5%AD%A6%E4%B9%A0/explore-x86_64-main()-part-2/)
