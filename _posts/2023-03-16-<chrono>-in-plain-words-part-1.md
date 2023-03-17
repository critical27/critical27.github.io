---
layout: single
title: std::chrono in plain words, part 1
date: 2023-03-16 00:00:00 +0800
categories: 学习
tags: C++
---

之前断断续续使用过chrono中的方法, 对于其中的概念总是一知半解, 导致每次查cppreference总是抓不到重点. 这次就试着用直白的语言, 描述下chrono的概念和原理. 这一篇主要着重于介绍chrono的核心, duration.

### Chrono

为什么要引入chrono? 在chrono之前, 我们几乎都用一个数字来表示时间, 所有时间单位的转换也需要自己来进行. 尽管这些单位换算在数学上很简单, 但这些单位换算和单位修改会影响大量代码, 容易出错和遗漏. 比如某个函数的参数期望单位是秒, 而有一天其他人改成了毫秒, 所有这个函数的调用方都要进行相应修改, 否则可能会出现逻辑上的错误.

> 另外一块的复杂度在于, 一旦使用数字来表示时间, 可能会受到C++类型隐式转换的影响
>

chrono的出现就是为了避免用数字表现时间带来的各种隐式问题, 其本质是定义通过模版定义了不同单位的时间, 从而让编译器帮助程序员在编译期就能发现逻辑上的错误.

### Duration

要理解chrono, 其基础就在于理解duration. duration的定义是一段时间, 可以是任意单位:

- 3秒
- 3分钟
- 3小时

chrono中这些时间单位本质上都是一个duration:

```cpp
std::chrono::hours
std::chrono::minutes
std::chrono::seconds
std::chrono::milliseconds
std::chrono::microseconds
std::chrono::nanoseconds
```

所有duration有以下特点, 我们以seconds为例:

- seconds is an arithmetic-like type.
- sizeof(seconds) == 8.
- It is trivially destructible.
- It is trivially default constructible.
- It is trivially copy constructible.
- It is trivially copy assignable.
- It is trivially move constructible.
- It is trivially move assignable.

可以看到它本身是traival的, 本质上和一个long long或者int64_t没有任何区别. 比如, seconds本质上就等同于如下实现(实际实现下面会重新说明)

```cpp
class seconds {
  int64_t sec_;
 public:
  seconds() = default;
  // ...
};
```

所有duration的初始化可以有default initialization和zero initialization. 但不允许数字类型向duration类型的转换.

```cpp
seconds s;      // no initialization
seconds s{};    // zero initialization
seconds s{3};   // 3 seconds
```

所有整形或者算数类型向duration的隐式转换都被编译器禁止

```cpp
void f(seconds d) { cout << d.count() << "s\n"; }

seconds s = 3;  // error: Not implicitly constructible from int
f(3);           // error: Not implicitly constructible from int
f(seconds{3});  // ok, 3s
f(3s);          // ok, 3s, Requires C++14
seconds x{3};
f(x);           // ok, 3s
```

duration也支持逻辑运算符和一些算术运算符

```cpp
auto x = 3s;
x += 2s;
if (x < 10s) {
  cout << "orz" << "\n";
}
```

> 所有duration都有一个count方法, 用于输出其数字表示形式. 但这个方法一般只用于IO输入输出, 而不应该依赖于这个数字进行计算.
>

以上就是duration的基本用法, 它本质上就是一个整数的wrapper, 其对外表现和一个整数没有很大区别. (除了禁止整数向duration的隐式转换)

### Duration的作用

那么有人要问, chrono定义这么多duration的意义何在? 仅仅是为了禁止整形向时间类型的转换吗? duration的一个重要作用就是能够在自动完成不同时间单位的换算.

比如文章一开始提到的例子, 有一个函数接收一个单位为秒的函数如下所示:

```cpp
void f(seconds d) { cout << d.count() << "s\n"; }
```

当有一天修改单位为毫秒后, 代码要么能够正常无误的运行, 要么在编译期就报错. 其背后的原理和编译器处理隐式转换的逻辑是一样的(比如允许`float → double`, `int32_t → int64_t`, 而反之通常会提示warning或者报错), chrono只允许向更精细类型的隐式转换.

```cpp
void f(milliseconds d) { cout << d.count() << "ms\n"; }

f(seconds{3});  // ok, no change needed! 3000ms
f(3h);          // ok, no change needed! 3000ms
f(3456us);      // error: no conversion

auto x = 2s;
auto y = 3ms;
f(x + y);       // ok, 2003ms
f(y - x);       // ok, -1997ms
```

这样设计的原因在于, 不同时间单位的换算关系是已知的, 当执行不损失精度的单位换算时, 就不需要手动来完成这些单位换算. 当且仅当出现损失精度的类型转换时, 需要通过`duration_cast`手动完成.

```cpp
void f(milliseconds d) { cout << d.count() << "ms\n"; }

f(3456us);                                // error
f(duration_cast<milliseconds>(3456us));   // ok, same as 3ms

```

`duration_cast`后面的模版参数代表目标类型, `duration_cast`永远都是往0的方向进行取整, 所以上面的3456us转换为了3ms, 而-3456us则会被转换为-3ms. (C++17中又引入了floor, ceil, round三种取整方式)

当我们使用chrono时需要进行不同单位之间的换算时:

- 应当尽量使用隐式转换. 因为如果编译能够通过, 那就说明chrono能够正确处理
- 当且仅当编译无法通过时, 此时需要通过`duration_cast`来进行显式转换, 并且强制要求程序员思考取整方式

> 隐式转换的性能, 是不亚于手动通过数字进行单位换算的, 编译时如果开了-O2的优化, 隐式转换和手动转换生成的汇编代码是相同的.
>

### 深入理解Duration

之前简单提过seconds的可能实现, 为了支持在编译器能够检查类型转换是否合法, duration的实现实际是通过模版完成的, chrono中内置的duration类型实际就是一个模版的实例化结果.

```cpp
using std::chrono::nanoseconds  = std::chrono::duration<int64_t, std::nano>;
using std::chrono::microseconds = std::chrono::duration<int64_t, std::micro>;
using std::chrono::milliseconds = std::chrono::duration<int64_t, std::milli>;
using std::chrono::seconds      = std::chrono::duration<int64_t>;
using std::chrono::minutes      = std::chrono::duration<int64_t, std::ratio<60>>;
using std::chrono::hours        = std::chrono::duration<int64_t, std::ratio<3600>>;
```

模版原型如下:

```cpp
template <class Rep, class Period = ratio<1>>
class duration {
public:
    using rep = Rep;
    using period = Period;
    // ...
};
```

第一个模版参数`Rep`是这个duration用什么数字类型来表示, 可以看到内置的几个类型都是int64_t. 对于`seconds`来说, `int64_t`足以可以表示2924亿年长度的时间, 而`milliseconds`可以表示2.924亿年长度的时间. 如果我们不需要表示这么长的时间怎么办? 比如, 当只需要用32位整形来表示`seconds`时, 我们就可以如此定义:

```cpp
using seconds32 = std::chrono::duration<int32_t>;
```

我们甚至可以用浮点数类型来表示时间

```cpp
using fseconds = duration<float>;
void f(fseconds d) {
  cout << d.count() << "s\n";
}
f(45ms + 63us);  // 0.045063s

void g(seconds d) {
  cout << d.count() << "s\n";
}
g(45ms + 63us);  // error: no known conversion
```

由于`fseconds`使用了浮点数, 上面的代码是能够编译并且运行的, 如果使用默认的`seconds`作为的参数类型则会编译报错, 因为无论是毫秒还是微秒都不能隐式转换为秒.

而在chrono的实现中, 当我们使用浮点数类型来表示时间时, 所有转换都不会存在truncation error(比如前文提到的3456ms转换为3s), 取而代之的是由于浮点数不能精确表示一个数字时, 会发生rounding error.

而第二个模板参数`Period`, 在chrono中本质上代表的则是时间的单位, 但对外的表现形式时这个单位和秒的换算关系, 这也是为什么称为`ratio`的原因. 我们举几个例子:

> chrono中所有换算关系都是以秒为换算单位, 但ratio本身就是一个编译器运算的工作类, 没有任何实际含义.
>

```cpp
// 1微秒是1/1000000秒
using micro = ratio<1, 1000000>;
using std::chrono::microseconds = std::chrono::duration<int64_t, std::micro>;

// 1分钟则是60/1 = 60秒
using std::chrono::minutes = std::chrono::duration<int64_t, std::ratio<60, 1>>;

// 1小时则是3600/1 = 3600秒
using std::chrono::hours = std::chrono::duration<int64_t, std::ratio<3600, 1>>;
```

基于这个模版, 我们可以自定义时间类型, 比如一个每秒60帧的游戏中, 每一帧的时长就可以如下定义:

```cpp
using frames = duration<int32_t, ratio<1, 60>>;

void f(duration<float, milli> d) {
  // ...
}

f(frames{1});         // 16.6667ms
f(45ms + frames{5});  // 128.333ms
```

到这里, 我们总算能回答一个问题了: chrono是如何实现不同类型之间的运算? 我们就以上面的`f(45ms + frames{5})`为例, 它们的模版实际模版参数如下, `<>`中的模板参数都是编译器就确定好的:

```cpp
// 首先我们计算求和的部分
45 <int64_t, 1/1000> + 5 <int32_t, 1/60>
```

当我们要计算两个不同单位时间时, 首先要将它们都转换成相同单位(不妨想象成两个单位的最小公倍数), 然后再进行相加:

```cpp
// 1000和60的最小公倍数是3000, 上面的算式等同于
45 * 3 <int64_t, 1/3000> + 5 * 50 <int64_t, 1/3000>
// 单位相同, 就可以直接相加, 得到
385 <int64_t, 1/3000>
// 然后把上一步结果, 传到f这个参数时, 需要再向duration<float, milli>转换
// 由于(1/3000) / (1/1000) = (1/3), 最终得到
128.333 <float, 1/1000>
```

所以, chrono就是如此对两个不同类型的时间进行运算的. 给定任意类型, 在编译期就能够确定能否进行相应运算. 如果可以, 代码正常编译, 在运行期也不会产生算术运算之外的额外开销. 如果不能, 在编译期就会报错.

值得提示的另一点是, duration的模版如果没有自定义时间类型的需要, 是完全不需要关注的, 使用默认提供的duration类型即可.

这一篇到这就结束, 下一篇, 我们会基于duration, 介绍剩余的几个概念.