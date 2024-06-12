---
layout: single
title: What is Core Analyzer
date: 2024-06-12 00:00:00 +0800
categories: 实践
tags: LDBC
---

一个有意思的工具：Core Analyzer。这一篇以官方的介绍为主，顺便介绍一下安装和简单使用。

## What is Core Analyzer

许多程序错误，特别是 C/C++ 中的错误，都与内存有关。当观察到程序故障时，无论是错误的行为还是彻底崩溃，解开谜团的关键往往归结为无效的数据对象或无法访问的内存地址。一个内存问题可能非常简单（比如解引用一个空指针），也有可能异常困难，比如多线程竞争而导致内存损坏等等。当错误隐藏在涉及大量数据对象的复杂执行上下文中时，尤其如此。任何有经验的人都会告诉你，将寻找错误比作大海捞针一点也不夸张。

内存分配和访问涉及内核、堆管理器、编译器和应用程序代码。因此，了解内存如何在宏观到微观的不同阶段被它们分配、拥有和访问是至关重要的。换句话说，相同的内存被不同的软件层以不同的方式查看和管理。例如，编译器会根据一个类的定义，构造及初始化一个对象，并根据其内存布局相应地访问数据，这些信息可以从调试符号看到；另一方面，内存管理器在进行对象分配内存时，可能会在这个内存对象中或者对象周围嵌入堆元数据以指示其大小和空闲/使用状态，其中还可能包括填充字节、安全签名等；一个对象可能位于进程的 text、data、堆或栈中，由内核虚拟内存管理器设置的某些权限位进行管理和保护。所有这些信息都很重要，它们可以帮你建立一套方法论，从而证明或排除程序故障原因的猜想。

一个程序有许多数据对象，对象的类型从简单的原始类型，如字符、整形、浮点数等，到具有多重继承的复杂聚合，如 C++ 对象。这些数据对象会直接或间接地相互引用，一个对象可能被多个其他对象共享或引用。应用程序代码通常通过多个间接引用来访问内存目标，这使得出现问题时很难找出根本原因。在典型的 Debug 过程中，我们可能手头有一个或多个可疑的数据对象。挑战在于找出还有哪些对象持有对可疑对象的引用，并且可能不正确地访问它们。如果程序有成千上万甚至数百万的变量，使用调试器检查所有变量的传统方式可能是一项令人望而却步的任务。这个过程十分繁琐易错，也或多或少类似于逆向工程，会非常困难。除此之外，堆数据对象与全局和局部变量不同，没有描述它们的类型或位置的调试符号，这使得 debugger 无能为力。然而，有不少对象是在堆上动态创建的，它们也常常会是怀疑的目标。以内存越界为例，追踪这类错误的关键是弄清楚什么是受害者前面的内存对象，谁拥有它，以及它是如何被读写的。

> 题外话，在我看来 Debug 涉及三个能力：对问题现象的认知程度，对代码的熟悉程度，以及对系统知识的掌握程度。三者侧重不同方向而又互补，缺一不可。而调查内存问题的难处在于：首先在于能否描述清楚问题的现象，什么情况下出现，进而找到怀疑对象，最终看能否顺藤摸瓜，检查怀疑有问题的代码和现象是否匹配。这个过程不仅仅是技术上的困难，更考验人的细心还耐心程度。
>

给定一个任意的内存对象的堆地址，它的数据类型与堆上的其他对象的关系并不明显。Core Analyzer 旨在帮助回答这些问题。尽管一些调试器提供了部分功能（例如，WinDbg 有一个扩展命令 `!heap` 来检查堆内存），但它们中没有一个可以使用堆信息来揭示众多数据对象之间的复杂关系。此外，许多程序出于各种原因使用自定义内存管理器（例如 tcmalloc 和 jemalloc），在这种情况下，调试器根本不知道内存管理器的存在。Core Analyzer 将进程的地址空间分类为 text、data、堆或栈区域，并内置了主流运行时内存管理器的堆数据结构，因此它能够扫描进程的整个堆，检查其一致性，找到并指出内存损坏的地址。通过搜索进程的地址空间中所有直接或间接引用可疑对象的引用，这个工具能以系统的方式帮助揭示对象的类型和用法。Core Analyzer 能够分析不同平台上的各种 core 文件格式，例如 Linux/Mac OS 上的 ELF core 和 Windows 上的 minidump。

下表列出了 Core Analyzer 的主要功能：

- Heap
    - 扫描堆并报告内存损坏和内存使用统计信息。
    - 显示给定地址周围的内存块布局。
    - 显示包含给定地址的内存块状态。
    - 显示最大的堆内存块（潜在的内存占用大户）。
- 内存引用
    - 查找与给定地址相关联的内存对象的大小、类型和符号。
    - 搜索并报告所有对给定对象的所有间接级别的引用。
- 其他
    - 查找给定 C++ 类的所有对象实例。
    - 显示被选定或所有线程共享的对象。
    - 显示带有数据对象上下文注释的反汇编指令。
    - 内存区域内的数据模式。
    - 包括所有段及其属性的详细进程映射

## 安装

Core Analyzer 开源，代码在这里，[yanqi27/core_analyzer: A power tool to debug memory-related issues (github.com)](https://github.com/yanqi27/core_analyzer)，里面有个脚本`build_gdb.sh`，本质上就是替换了一些 gdb 源文件，然后编译 gdb。（可能需要在这个脚本中修改 gdb 相关编译参数）

途中遇到了报错报错如下：

```cpp
./doodle/bin/gdb: /lib/x86_64-linux-gnu/libstdc++.so.6: version `CXXABI_1.3.13' not found (required by /home/doodle.wang/doodle/bin/gdb)
./doodle/bin/gdb: /lib/x86_64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.29' not found (required by /home/doodle.wang/doodle/bin/gdb)
./doodle/bin/gdb: /lib/x86_64-linux-gnu/libstdc++.so.6: version `GLIBCXX_3.4.30' not found (required by /home/doodle.wang/doodle/bin/gdb)
```

增加`--disable-source-highlight LDFLAGS="-static-libstdc++ -static-libgcc"`

```bash
# 不太记得下面的参数遇到什么问题了
# ../configure --prefix=/home/doodle.wang/doodle --with-python --disable-source-highlight LDFLAGS="-static-libstdc++ -static-libgcc"

# 最终使用的如下编译参数
../configure --prefix=/home/doodle.wang/doodle --disable-binutils --with-python --disable-ld --disable-gold --disable-gas --disable-sim --disable-gprof --disable-source-highlight CXXFLAGS='-g' CFLAGS='-g' LDFLAGS="-static-libstdc++ -static-libgcc"
```

最终安装成功之后，表面上就是一个普通的 gdb，其内部拓展了若干 gdb 命令。

> 官网[这里](https://core-analyzer.sourceforge.net/index_files/Page1510.html)提到的 stand-alone 工具 core_analyzer 没找到。
>

## 使用

```cpp
gdb a.out core.1234
```

这里的 gdb 就是在 core_analyzer 中编译的特殊 gdb。我们把 core_analyzer 拓展的 gdb 命令分成几类：

- 打印堆相关信息
- 检查特定内存地址的相关信息（包括在哪个 segment、block，直接引用和间接引用，以及内存 layout 等）
- 其他辅助命令

### 打印堆相关信息

1. 打印整个堆的概览，可以看到我此处程序使用的是 jemalloc 5.3.0

    ```bash
    (gdb) heap
    jemalloc 5.3.0
    arena[0]: inuse_blks=8981 inuse_bytes=1825672 free_blks=1788 free_bytes=2835576
    arena[1]: inuse_blks=1 inuse_bytes=16 free_blks=256 free_bytes=2097136
    arena[2]: inuse_blks=124 inuse_bytes=16392 free_blks=2341 free_bytes=2080760

    There are 3 arenas Total 8MB
    Total 9106 blocks in-use of 1MB
    Total 4385 blocks free of 6MB
    ```

2. 根据堆上对象大小，输出内存使用的统计信息

    ```cpp
    (gdb) heap /v
    jemalloc 5.3.0
    arena[0]: inuse_blks=8981 inuse_bytes=1825672 free_blks=1788 free_bytes=2835576
    	bin[0]: curslabs=1 nonfull_slabs=0
    	bin[1]: curslabs=2 nonfull_slabs=0
    	bin[2]: curslabs=38 nonfull_slabs=0
    	bin[3]: curslabs=2 nonfull_slabs=0
    	bin[4]: curslabs=38 nonfull_slabs=0
    	bin[5]: curslabs=1 nonfull_slabs=0
    	bin[6]: curslabs=2 nonfull_slabs=1
    	bin[7]: curslabs=1 nonfull_slabs=0
    	bin[8]: curslabs=1 nonfull_slabs=0
    	bin[9]: curslabs=1 nonfull_slabs=0
    	bin[10]: curslabs=1 nonfull_slabs=0
    	bin[11]: curslabs=1 nonfull_slabs=0
    	bin[12]: curslabs=6 nonfull_slabs=5
    	bin[13]: curslabs=1 nonfull_slabs=0
    	bin[14]: curslabs=1 nonfull_slabs=0
    	bin[15]: curslabs=1 nonfull_slabs=0
    	bin[16]: curslabs=6 nonfull_slabs=1
    	bin[17]: curslabs=1 nonfull_slabs=0
    	bin[18]: curslabs=1 nonfull_slabs=0
    	bin[19]: curslabs=3 nonfull_slabs=0
    	bin[20]: curslabs=5 nonfull_slabs=0
    	bin[21]: curslabs=3 nonfull_slabs=0
    	bin[22]: curslabs=1 nonfull_slabs=0
    	bin[23]: curslabs=8 nonfull_slabs=0
    	bin[24]: curslabs=2 nonfull_slabs=0
    	bin[25]: curslabs=2 nonfull_slabs=0
    	bin[26]: curslabs=1 nonfull_slabs=0
    	bin[27]: curslabs=10 nonfull_slabs=0
    	bin[28]: curslabs=4 nonfull_slabs=0
    	bin[30]: curslabs=2 nonfull_slabs=0
    	bin[31]: curslabs=1 nonfull_slabs=0
    	bin[32]: curslabs=4 nonfull_slabs=0
    	bin[35]: curslabs=5 nonfull_slabs=0
    arena[1]: inuse_blks=1 inuse_bytes=16 free_blks=256 free_bytes=2097136
    	bin[1]: curslabs=1 nonfull_slabs=0
    arena[2]: inuse_blks=124 inuse_bytes=16392 free_blks=2341 free_bytes=2080760
    	bin[0]: curslabs=1 nonfull_slabs=0
    	bin[1]: curslabs=1 nonfull_slabs=0
    	bin[2]: curslabs=1 nonfull_slabs=0
    	bin[3]: curslabs=1 nonfull_slabs=0
    	bin[4]: curslabs=2 nonfull_slabs=0
    	bin[5]: curslabs=1 nonfull_slabs=0
    	bin[6]: curslabs=1 nonfull_slabs=0
    	bin[7]: curslabs=1 nonfull_slabs=0
    	bin[8]: curslabs=1 nonfull_slabs=0
    	bin[9]: curslabs=1 nonfull_slabs=0
    	bin[10]: curslabs=2 nonfull_slabs=0
    	bin[12]: curslabs=2 nonfull_slabs=0
    	bin[13]: curslabs=2 nonfull_slabs=0
    	bin[14]: curslabs=1 nonfull_slabs=0
    	bin[17]: curslabs=1 nonfull_slabs=0
    	bin[21]: curslabs=1 nonfull_slabs=0
    	bin[23]: curslabs=1 nonfull_slabs=0

    There are 3 arenas Total 8MB
    Total 9106 blocks in-use of 1MB
    Total 4385 blocks free of 6MB

    ========== In-use Memory Histogram ==========
    Size-Range     Count       Total-Bytes
    0 - 16         806(8%)     9KB(0%)
    16 - 32        4747(52%)   148KB(8%)
    32 - 64        2788(30%)   168KB(9%)
    64 - 128       99(1%)      9KB(0%)
    128 - 256      168(1%)     36KB(2%)
    256 - 512      113(1%)     45KB(2%)
    512 - 1024     114(1%)     100KB(5%)
    1024 - 2KB     163(1%)     264KB(14%)
    2KB - 4KB      93(1%)      315KB(17%)
    4KB - 8KB      2(0%)       14KB(0%)
    8KB - 16KB     5(0%)       76KB(4%)
    16KB - 32KB    4(0%)       108KB(6%)
    --Type <RET> for more, q to quit, c to continue without paging--c
    32KB - 64KB    2(0%)       104KB(5%)
    64KB - 128KB   1(0%)       80KB(4%)
    256KB - 512KB  1(0%)       320KB(17%)
    Total          9106        1MB
    ========== Free Memory Histogram ==========
    Size-Range     Count       Total-Bytes
    0 - 16         1242(28%)   14KB(0%)
    16 - 32        245(5%)     7KB(0%)
    32 - 64        540(12%)    27KB(0%)
    64 - 128       1373(31%)   130KB(1%)
    128 - 256      536(12%)    99KB(1%)
    256 - 512      255(5%)     90KB(1%)
    512 - 1024     82(1%)      56KB(0%)
    1024 - 2KB     57(1%)      88KB(1%)
    2KB - 4KB      20(0%)      69KB(1%)
    4KB - 8KB      11(0%)      78KB(1%)
    8KB - 16KB     14(0%)      184KB(2%)
    16KB - 32KB    6(0%)       144KB(2%)
    256KB - 512KB  1(0%)       484KB(7%)
    512KB -        3(0%)       5MB(78%)
    Total          4385        6MB
    ```

3. 扫描整个堆，根据对象的直接引用或者间接引用输出可能的内存泄漏对象：

    ```bash
    (gdb) heap /leak
    Potentially leaked heap memory blocks:
    [1] addr=0x7ffff001e040 size=32
    Total 1 (32) leak candidates out of 9106 (1MB) in-use memory blocks
    ```

4. 打印内存使用最高的 n 个 block，这里所说的内存使用包括被全局变量、局部变量和其他 heap 对象所引用。

    ```bash
    (gdb) heap /tb 8
    Top 8 biggest in-use heap memory blocks:
    	addr=0x7fffee801000  size=2093056 (1MB)
    	addr=0x7fffeda43000  size=1822720 (1MB)
    	addr=0x7fffed87c000  size=1589248 (1MB)
    	addr=0x7fffed7b2000  size=495616 (484KB)
    	addr=0x7fffed82b200  size=327680 (320KB)
    	addr=0x7ffff0008e00  size=81920 (80KB)
    	addr=0x7ffff01c4b00  size=65536 (64KB)
    	addr=0x7ffff00aa300  size=40960 (40KB)
    ```

5. 打印引用堆上内存最多的全局变量或者局部变量

    ```bash
    (gdb) heap /tu 8
    [1] [heap block] 0x7ffff01bd7c0--0x7ffff01c37c0 size=24576
        |--> 674KB (842 blocks)
    [2] [heap block] 0x7ffff0025320--0x7ffff00253c0 size=160
        |--> 360KB (5 blocks)
    [3] [heap block] 0x7fffed82b200--0x7fffed87b200 size=327680
        |--> 320KB (1 blocks)
    [4] [heap block] 0x7ffff0000000--0x7ffff0007000 size=28672
        |--> 298KB (732 blocks)
    [5] [heap block] 0x7ffff0024860--0x7ffff0024870 size=16
        |--> 80KB (2 blocks)
    [6] [.data/.bss] /home/doodle.wang/Git/nebula-ng/build/bin/test/../3rd/libstdc++.so.6 Unknown global 0x7ffff5a5c268: 0x7ffff0008e00
        |--> 80KB (1 blocks)
    [7] [heap block] 0x7ffff0008e00--0x7ffff001ce00 size=81920
        |--> 80KB (1 blocks)
    [8] [heap block] 0x7ffff00740e0--0x7ffff0074140 size=96
        |--> 64KB (2 blocks)
    ```


### 检查特定内存地址的相关信息

1. 打印特定内存地址所在 segment

    > 打印`0x7ffff001e040`这个地址所在 segment
    >

    ```bash
    (gdb) heap /c 0x7ffff001e040
    jemalloc 5.3.0
    arena[0]: inuse_blks=8981 inuse_bytes=1825672 free_blks=1788 free_bytes=2835576
    arena[1]: inuse_blks=1 inuse_bytes=16 free_blks=256 free_bytes=2097136
    arena[2]: inuse_blks=124 inuse_bytes=16392 free_blks=2341 free_bytes=2080760

    There are 3 arenas Total 8MB
    Total 9106 blocks in-use of 1MB
    Total 4385 blocks free of 6MB
    ```

2. 打印特定内存地址所在 block

    ```bash
    (gdb) heap /b 0x7ffff001e040
    	[In-use]
    	[Address] 0x7ffff001e040
    	[Size]    32
    	[Offset]  0
    ```

3. 打印特定全局变量、局部变量或者堆上对象的统计信息，包括：
    1. 从该对象开始，经过若干层直接引用或间接引用，最终能触及的所有变量使用的内存。
    2. 该对象能直接应用的其他对象所使用的内存

    > 打印`executionPlan`这个对象的统计信息，它直接引用448个子节，能触及731个block，总计179KB大小的内存地址
    >

    ```bash
    (gdb) heap /u executionPlan
    Heap memory consumed by [stack] thread 1 frame 0 executionPlan @0x7fffffffdd30
    All reachable:
        |--> 179KB (731 blocks)
    Directly referenced:
        |--> 448 (1 blocks)
    ```

4. 检查特定内存地址被哪些对象所引用

    > 有一个`shared_ptr<PlanNode>`, 其对象地址在`0x7ffff0025d30`，检查`0x7ffff0025d30`这个地址被谁所引用
    >

    ```bash
    (gdb) p plan
    $4 = {<std::__shared_ptr<nebula::PlanNode, __gnu_cxx::_S_atomic>> = {<std::__shared_ptr_access<nebula::PlanNode, __gnu_cxx::_S_atomic, false, false>> = {<No data fields>}, _M_ptr = 0x7ffff0025d30,
        _M_refcount = {_M_pi = 0x7ffff0025d20}}, <No data fields>}

    (gdb) ref 0x7ffff0025d30
    Search for object type associated with 0x7ffff0025d30
    Address 0x7ffff0025d30 belongs to heap block [0x7ffff0025d20, 0x7ffff0025dc0] size=160
    ------------------------- 1 -------------------------
    [heap block] 0x7ffff0025d20--0x7ffff0025dc0 size=160 (type="std::_Sp_counted_ptr_inplace<nebula::InsertNode, std::allocator<nebula::InsertNode>, (__gnu_cxx::_Lock_policy)2>") @+16
    ```

5. 检查特定范围的内存地址被哪些对象直接和间接引用

    > 进而检查`0x7ffff0025d30`这个地址如何被直接和间接引用
    >

    ```bash
    (gdb) ref 0x7ffff0025d30 8 2
    Search for references to 0x7ffff0025d30 size 8 up to 2 levels of indirection
    ------------------------- Level 2 -------------------------
    [stack] thread 4 frame 7 rsp+1304 @0x7fffee7ed068: 0x7ffff00861c4
    [stack] thread 4 frame 9 guard.executionPlan_ @0x7fffee7ee520: 0x7ffff00860d0
    [heap block] 0x7ffff007cc00--0x7ffff007d200 size=1536 (type="std::_Sp_counted_ptr_inplace<nebula::exec::ExecutionContext, std::allocator<nebula::exec::ExecutionContext>, (__gnu_cxx::_Lock_policy)2>") @+104: 0x7ffff00860d0
    [heap block] 0x7ffff008bd80--0x7ffff008be60 size=224 (type="std::_Sp_counted_ptr_inplace<nebula::exec::Pipeline, std::allocator<nebula::exec::Pipeline>, (__gnu_cxx::_Lock_policy)2>") @+72: 0x7ffff00860c0
    [stack] thread 1 frame 0 executionPlan::"std::__shared_ptr<nebula::exec::ExecutionPlan, __gnu_cxx::_S_atomic>"._M_ptr @0x7fffffffdd30: 0x7ffff00860d0
    [stack] thread 1 frame 0 executionPlan::"std::__shared_ptr<nebula::exec::ExecutionPlan, __gnu_cxx::_S_atomic>"._M_refcount._M_pi @0x7fffffffdd38: 0x7ffff00860c0
    |--> [heap block] 0x7ffff00860c0--0x7ffff0086280 size=448

    [stack] thread 4 frame 6 rsp+40 @0x7fffee7eca58: 0x7ffff00a16c0
    [stack] thread 4 frame 6 this @0x7fffee7ecb38: 0x7ffff00a16c0
    [stack] thread 4 frame 7 rsp+296 @0x7fffee7ecc78: 0x7ffff00a16c0
    [stack] thread 4 frame 7 rsp+344 @0x7fffee7ecca8: 0x7ffff00a16c0
    [stack] thread 4 frame 7 rsp+352 @0x7fffee7eccb0: 0x7ffff00a16c0
    [stack] thread 4 frame 7 memGraphInsertNodes.this @0x7fffee7ecd28: 0x7ffff00a16c0
    [stack] thread 4 frame 7 this @0x7fffee7ed0d0: 0x7ffff00a16c0
    [stack] thread 4 frame 8 rsp+8 @0x7fffee7ed0e8: 0x7ffff00a16c0
    [stack] thread 4 frame 8 this @0x7fffee7ed0f8: 0x7ffff00a16c0
    [stack] thread 4 frame 9 op @0x7fffee7ee500: 0x7ffff00a16c0
    [heap block] 0x7ffff0000000--0x7ffff0007000 size=28672 @+17992: 0x7ffff00a16c0
    [heap block] 0x7ffff0034c98--0x7ffff0034ca0 size=8 @+0: 0x7ffff00a16c0
    |--> [heap block] 0x7ffff00a16c0--0x7ffff00a1800 size=320

    ------------------------- Level 1 -------------------------
        [heap block] 0x7ffff00860c0--0x7ffff0086280 size=448 (type="struct nebula::exec::ExecutionPlan").logicPlan_::"std::__shared_ptr<nebula::PlanNode, __gnu_cxx::_S_atomic>"._M_ptr @+40: 0x7ffff0025d30
        [heap block] 0x7ffff00a16c0--0x7ffff00a1800 size=320 (type="struct nebula::exec::InsertNode").planNode_::"std::__shared_ptr<nebula::InsertNode const, __gnu_cxx::_S_atomic>"._M_ptr @+272: 0x7ffff0025d30
        [stack] thread 1 frame 0 rsp+3128 @0x7fffffffdd48: 0x7ffff0025d30
        [stack] thread 1 frame 0 plan::"std::__shared_ptr<nebula::PlanNode, __gnu_cxx::_S_atomic>"._M_ptr @0x7fffffffdd58: 0x7ffff0025d30
        |--> searched target [0x7ffff0025d30, 0x7ffff0025d38)
    ```

6. 检查特定内存地址范围内的内存使用情况，如果被使用，显示其类型

    ```bash
    (gdb) pattern <start> <end>
    ```

    ```bash
    (gdb) pattern 0x7ffff0025d20 0x7ffff0025d40
    memory pattern [0x7ffff0025d20, 0x7ffff0025d40]:
    0x7ffff0025d20: 0x7ffff6ff4350 => [.text/.rodata] /home/doodle.wang/Git/nebula-ng/build/bin/test/../lib/libnb-plan.so vtable for std::_Sp_counted_ptr_inplace<nebula::InsertNode, std::allocator<nebula::InsertNode>, (__gnu_cxx::_Lock_policy)2> (0x7ffff6ff4340--0x7ffff6ff4378)
    0x7ffff0025d28: 0x100000004
    0x7ffff0025d30: 0x7ffff6ff85c0 => [.text/.rodata] /home/doodle.wang/Git/nebula-ng/build/bin/test/../lib/libnb-plan.so vtable for nebula::InsertNode (0x7ffff6ff85b0--0x7ffff6ff8648)
    0x7ffff0025d38: 0
    ```


### 其他辅助命令

1. 输出一个特定类型的所有对象（这个方法是通过虚函数表实现的，因此只能作用于有虚函数表的类）

    ```bash
    (gdb) obj <expression>
    ```

    ```bash
    (gdb) obj folly::CPUThreadPoolExecutor
    Searching objects of type="folly::CPUThreadPoolExecutor" size=2624 (vtable 0x7ffff60949a0--0x7ffff6094a60)
        [heap block] 0x7ffff00c6400--0x7ffff00c7000 size=3072
    Total objects found: 1
    ```

2. 打印一个类型的内存布局

    ```bash
    (gdb) dt <expression>
    ```

    ```bash
    (gdb) dt ExecutionPlan
    type=nebula::exec::ExecutionPlan  size=376
    {
      +0   (base) struct std::enable_shared_from_this<nebula::exec::ExecutionPlan>  size=16
      {
        +0   struct std::weak_ptr<nebula::exec::ExecutionPlan> _M_weak_this  size=16
      }
      +16  struct nebula::gql::RequestContext* rctx_  size=8
      +24  struct std::shared_ptr<nebula::PlanNode> logicPlan_  size=16
      +40  struct nebula::exec::ConsumerSupplier consumerSupplier_  size=32
      +72  struct folly::Executor* executor_  size=8
      +80  struct std::shared_ptr<nebula::exec::ExecutionContext> executionContext_  size=16
      +96  struct std::vector<std::shared_ptr<nebula::exec::Pipeline>, std::allocator<std::shared_ptr<nebula::exec::Pipeline> > > pipelines_  size=24
      +120 struct size_t numTotalPipelines_  size=8
      +128 struct size_t numRunningPipelines_  size=8
      +136 struct size_t numFinishPipelines_  size=8
      +144 struct std::unordered_map<long, nebula::exec::Operator*, std::hash<long>, std::equal_to<long>, std::allocator<std::pair<long const, nebula::exec::Operator*> > > consumerOperators_  size=56
      +200 struct std::mutex mutex_  size=40
      +240 struct std::atomic<bool> terminateRequested_  size=1
      +241 struct std::atomic<bool> pauseRequested_  size=1
      +244 struct std::atomic<int> toYield_  size=4
      +248 struct int32_t numRunningThreads_  size=4
      +252 struct nebula::exec::ExecutionState state_  size=4
      +256 struct std::vector<nebula::exec::NPromise<folly::Unit>, std::allocator<nebula::exec::NPromise<folly::Unit> > > threadFinishPromises_  size=24
      +280 struct std::vector<folly::SemiFuture<folly::Unit>, std::allocator<folly::SemiFuture<folly::Unit> > > completeFutures_  size=24
      +304 struct nebula::Status result_  size=40
      +344 struct std::function<folly::Future<folly::Unit> ()> killQuery_  size=32
    };
    ```

## 结论

到这基本的命令都介绍完了，感兴趣的可以移步[这里](https://core-analyzer.sourceforge.net/index_files/Page600.html)查看所有命令。

经过我这几天使用 core_analyzer，这个工具的确很强大。一般简单的内存问题，我们可以通过 ASAN 就能修复。复杂一些的并发场景内存问题，如果能稳定复现，是可以使用 rr 抓取问题现场进而解决的。如果不能稳定复现，使用 core_analyzer 是一个极其有用的武器。之后有时间会结合一个内存泄漏，详细介绍如何使用 core_analyzer 检查内存问题。

## Reference

[Core Analyzer Home (sourceforge.net)](https://core-analyzer.sourceforge.net/index.html)