---
layout: single
title: SIMD Intrinsics整理
date: 2022-06-27 00:00:00 +0800
categories: C++ SIMD
tags: C++ SIMD
---

简单总结了下SIMD的常用Intrinsics。

## 命令命名组成

前缀 + 名字 + 后缀

### 前缀

主要用来表示命令适用的寄存器大小

256 bits: Only available for 256bit SSE registers (always use **`_mm256_`** prefix)

128 bits: Only available for 128bit SSE registers (always use **`_mm_`** prefix)

≤ 64 bits: Operation does not operate on SSE registers but usual 64bit registers (always use **`_mm_`** prefix)

### 名字

主要用来表示命令的作用，主要分为几大类

- Register I/O
- Comparsions
- Arithmetics
- Bit Operations
- Conversions
- Byte Manipulation

### 后缀

决定该intrinsics对什么样的数据类型操作，部分是没有后缀的

The suffix chosen determines the data type on which the intrinsic operates. It must be added as a suffix to the intrinsic name separated by an underscore, so a possible name for the data type `pd` in this example would be `_mm_cmpeq_pd`.

> 后缀参见左下这张表第一列，其中的X应该是代表寄存器位数，比如128/256。第二列中的Type映射到右下表格，代表操作数（也就是参数）的数据类型

![figure]({{'/archive/SIMD-suffix.png' | prepend: site.baseurl}})

- 名字中带`p`的代表`packed`，寄存器中可能会有多个`packed`的对象 (寄存器位数可能远大于packed大小)

`ph`: 16 bit float

`ps`: 32 bit float

`pd`: 64 bit float

`epi8/epi16/epi32/epi64`: 8/16/32/64位有符号整数

`epu8/epu16/epu32/epu64`: 8/16/32/64位无符号整数

有的操作会对寄存器中多个packed对象尽心操作，而有的操作又只会对单个packed对象进行操作，最好的例子就是`_mm_store_epi64`和`_mm_storel_epi64`，前者是会按64bit有符号整数的粒度执行两次写入到某个内存地址（实际存128bit数据），而storel代表只存低64位，所以会按64bit有符号整数写入到某个内存地址（实际只存64bit数据）。

又比如`_mm_cmpeq_epi8`是针对128bits寄存器进行操作，但这个操作的粒度是针对128bits中的每个8位有符号数进行比较。

> 由于带`p`的实际是对寄存器中多个`packed`对象进行操作，所以一般最大就到64bit的数据类型，极少的Intrinsics中有`epi128`

- 名字中带s的代表single，是针对单个数据类型进行操作(这种操作基本都是针对寄存器来进行的 所以位数基本上都是128或者256)

`si128/si256`: 按单个128/256有符号整数

`su128/su256`: 按单个128/256无符号整数

`ss`: 单个32bit float

`sd`: 单个64bit float

> ss和sd会比较特殊 因为寄存器中的剩余bit根据不同Intrinsics有不同的操作方式

- 带`m`的后缀比如`m128`和`m256`都是用于AVX，暂不涉及

> 前缀中出现的`_mm_`和`_mm256_`是另一回事

### Example

我们从命名规则上来稍微介绍，不会详细介绍对应参数，重点在于理解其命名规则

`_mm_loadu_si128`：u代表unaligned，对单个128进行读取

`_mm_load_sd`: 读取128中的低64bit进行读取(高64bit置为0)

`_mm_setzero_pd:` 将128bit中全部置0 (对应`128bit`的名字为`_mm_setzero_si128` 两个操作的执行效果完全相同，但一个是对`packed double`进行操作，一个是直接对`single 128 bits`进行操作)

`_mm_cmpeq_epi8`: 对两个128bit中的每8位进行比较

`_mm_testz_si128`: 对两个128bit中进行按位与

`_mm_cvtsi128_si64`: cvt代表类型转换，从单个128bit转为单个64bit有符号整数（也就是取低64位）

`_mm_storeu_si128`：把128位数据按有符号整数写入到某个内存地址

`_mm_storel_epi64`：把128位数据的低64位按有符号整数写入到某个内存地址

## Intrinsics分类

### Register I/O

主要作用就是从内存load数据到SSE寄存器，或者将SSE寄存器数据保存到另一个SSE寄存器或者内存，其中又分为几类

- Aligned Load: 从地址对齐的内存读数据到SSE寄存器中

![figure]({{'/archive/SIMD-aligned-load.png' | prepend: site.baseurl}})

- Unaligned Load

![figure]({{'/archive/SIMD-unaligned-load.png' | prepend: site.baseurl}})

- Aligned Store

![figure]({{'/archive/SIMD-aligned-store.png' | prepend: site.baseurl}})

- Unaligned Store

![figure]({{'/archive/SIMD-unaligned-store.png' | prepend: site.baseurl}})

### Comparisons

![figure]({{'/archive/SIMD-comparisons.png' | prepend: site.baseurl}})

### Arithmetics

![figure]({{'/archive/SIMD-arithmetics.png' | prepend: site.baseurl}})

- Bit Operations

![figure]({{'/archive/SIMD-bit-operations.png' | prepend: site.baseurl}})

- Conversions

![figure]({{'/archive/SIMD-conversions.png' | prepend: site.baseurl}})

- Byte Manipulation

![figure]({{'/archive/SIMD-byte-manipulation.png' | prepend: site.baseurl}})

## Reference

[intrinsics guide](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)

[cheatsheet](https://db.in.tum.de/~finis/x86%20intrinsics%20cheat%20sheet%20v1.0.pdf)