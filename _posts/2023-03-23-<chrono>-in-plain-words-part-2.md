---
layout: single
title: std::chrono in plain words, part 2
date: 2023-03-23 00:00:00 +0800
categories: 学习
tags: C++
---

[上一篇](https://critical27.github.io/%E5%AD%A6%E4%B9%A0/chrono-in-plain-words-part-1/)主要介绍了chrono中的核心duration, 这篇继续介绍基于duration延伸出的若干概念:

- time_point
- clock

## time_point

duration和time_point都能指代时间, 区别在于:

- duration可以代表任意一段时间, 比如3600秒可以指代这个小时, 或者下一个小时
- time_point只能代表一个时间点, 比如`time_point<system_clock, seconds> tp{10'000s}`实际指代的是`1970-01-01 02:46:40 UTC`

time_point本质上就是基于某个时钟的某个时间点, 或者说是一个epoch加上或者减掉一段duration. 我们看下time_point的实现:

```cpp
template <class Clock, class Duration = typename Clock::duration>
class time_point {
  Duration d_;

 public:
  using clock = Clock;
  using duration = Duration;
  // ...
};
```

可以看到, 每个time_point里面都有一个duration, 所以time_point和duration在内存中一模一样(通过gdb也可以验证), 只不过含义不一样而已.

time_point和duration一样同样支持运算, 两个time_point相减可以得到一个duration (比如明天10点 - 今天10点 = 24小时长的duration), 但两个time_point不能相加, 没有任何物理含义. time_point和duration可以进行加减运算, 代表时间点的移动.

```cpp
// clock后面会提, 每个clock都有一个方法now, 返回当前time_point
auto tp1 = std::chrono::system_clock::now();
// ...
auto tp2 = std::chrono::system_clock::now();

auto d = tp2 - tp1;  // 两次时间点之间的耗时
d += 3s;
```

从其他方面上, time_point由于内部是由duration表示, 所以和duration的行为都是一致的

- time_point允许不损失精度的隐式转换
- 损失精度的转换需要显式通过`time_point_cast`完成
- duration向time_point转换需要显式转换
- time_point向duration抓换可以通过time_point的`time_since_epoch`方法完成, 表示从起始时间到time_point表示的时间的范围

```cpp
template <class D>
using sys_time = time_point<system_clock, D>;

sys_time<seconds> tp1{5s};            // 5s
sys_time<milliseconds> tp2 = tp1;     // 5000ms

sys_time<milliseconds> tp3{1234ms};                     // 1234ms
sys_time<seconds> tp4 = time_point_cast<seconds>(tp3);  // 1s

```

## clocks

### prototype

我们先看下clock的原型实现

```cpp
struct some_clock {
  using duration = chrono::duration<int64_t, microseconds>;
  using rep = duration::rep;
  using period = duration::period;
  using time_point = chrono::time_point<some_clock>;
  static constexpr bool is_steady = false;

  static time_point now() noexcept;
  static time_t to_time_t(const time_point& t) noexcept;
  static time_point from_time_t(time_t t) noexcept;
};
```

clock本身提供了若干方法: `now`是核心方法, 返回基于这个clock的当前时间. 而`to_time_t`和`from_time_t`只是为了兼容C语言而提供的time_point和time_t的转换方法. `is_steady`则用来表示这个时钟是否会跳变或者回退 (比如收到NTP影响).

如果说clock能够返回一个time_point, 那么每个time_point是否要关联一个clock呢? 答案是肯定的. 想象一个普通的手表, 它的精度就是秒, 手表这个时钟能够能给出的time_point集合就是若干”秒”. 而一个日历的精度则是天, 作为一个时钟, 日历能够给出的time_point集合则是若干”天”.

所以, 从chrono中获取的任何time_point都是由关联的clock给出的, 来自相同clock的time_point之间, 由于参考系相同, 所以可以互相比较, 而来自不同clock的time_point则不可以进行比较或者转换.

```cpp
system_clock::time_point tp1 = system_clock::now();
steady_clock::time_point tp2 = tp1;  // error: no viable conversion
```

### 不同类型的clock

在chrono中引入了一些类型的clock

```cpp
system_clock
steady_clock
high_resolution_clock
utc_clock
tai_lock
gps_clock
file_clock
```

后四种都是在C++20中才引入的, 这篇文章主要还是讨论前几种clock.

- system_clock: 如果需要返回和真实时间相关的time_point (比如现在是几点几分), 这就是唯一的选择. system_clock的返回的time_point都是基于`1970-01-01 00:00:00 UTC`(time since epoch)
- steady_clock: 想象成一个秒表即可, 主要作用就是用来计时, 但不能返回任何真实时间.
- high_resolution_clock: 本质上就是个alias template, 在不同编译器中的alias甚至不同, msvc中是steady_clock, gcc中是system_clock, clang中则是可以通过选项配置, 所以我想不到任何场景需要用high_resolution_clock.

实际上, clock的代码非常简单, 我们不妨以system_clock为例, 看下几个编译器所使用的实现

- clang

```cpp
class _LIBCPP_TYPE_VIS system_clock
{
public:
    typedef microseconds                     duration;
    typedef duration::rep                    rep;
    typedef duration::period                 period;
    typedef chrono::time_point<system_clock> time_point;
    static _LIBCPP_CONSTEXPR_AFTER_CXX11 const bool is_steady = false;

    static time_point now() _NOEXCEPT;
    static time_t     to_time_t  (const time_point& __t) _NOEXCEPT;
    static time_point from_time_t(time_t __t) _NOEXCEPT;
};
```

- gcc

```cpp
struct system_clock {
  typedef chrono::nanoseconds duration;
  typedef duration::rep rep;
  typedef duration::period period;
  typedef chrono::time_point<system_clock, duration> time_point;

  static_assert(system_clock::duration::min() < system_clock::duration::zero(),
                "a clock's minimum duration cannot be less than its epoch");

  static constexpr bool is_steady = false;

  static time_point now() noexcept;

  // Map to C API
  static std::time_t to_time_t(const time_point& __t) noexcept {
    return std::time_t(duration_cast<chrono::seconds>(__t.time_since_epoch()).count());
  }

  static time_point from_time_t(std::time_t __t) noexcept {
    typedef chrono::time_point<system_clock, seconds> __from;
    return time_point_cast<system_clock::duration>(__from(chrono::seconds(__t)));
  }

```

- MSVC

```cpp
_EXPORT_STD struct system_clock {  // wraps GetSystemTimePreciseAsFileTime/GetSystemTimeAsFileTime
  using rep = long long;
  using period = ratio<1, 10'000'000>;  // 100 nanoseconds
  using duration = _CHRONO duration<rep, period>;
  using time_point = _CHRONO time_point<system_clock>;
  static constexpr bool is_steady = false;

  _NODISCARD static time_point now() noexcept {  // get current time
    return time_point(duration(_Xtime_get_ticks()));
  }

  _NODISCARD static __time64_t to_time_t(
      const time_point& _Time) noexcept {  // convert to __time64_t
    return duration_cast<seconds>(_Time.time_since_epoch()).count();
  }

  _NODISCARD static time_point from_time_t(__time64_t _Tm) noexcept {  // convert from __time64_t
    return time_point{seconds{_Tm}};
  }
};
```

可以看到, 即便对于同样的system_clock, 各个编译器所使用的精度都不同, clang是1us, gcc是1ns, 而MSVC是10ns.

## example

到这里chrono的基础概念我们都介绍完了, 下面会举一些使用的例子.

- 计时

```cpp
auto t0 = steady_clock::now();
f();
auto t1 = steady_clock::now();
cout << nanoseconds{t1-t0}.count() << "ns\n";                   // 135169457ns
cout << duration<double>{t1-t0}.count() << "s\n";               // 0.135169s
cout << duration_cast<milliseconds>(t1-t0).count() << "ms\n";   // 135ms

```

- 结合timed_mutex控制加锁时间

```cpp
std::timed_mutex mut;
if (mut.try_lock_for(500ms)) {
  // got the lock
  // ...
}
if (mut.try_lock_until(steady_clock::now() + 500ms)) {
    // got the lock
    // ...
}
```

- 自定义duration

```cpp
using days = duration<int, ratio<86400>>;
using frames = duration<int, ratio<1, 60>>;

std::this_thread::sleep_for(days{1});
std::this_thread::sleep_for(frames{1});
```