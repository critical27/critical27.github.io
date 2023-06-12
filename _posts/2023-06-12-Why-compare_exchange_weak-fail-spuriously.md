---
layout: single
title: Why compare_exchange_weak fail spuriously?
date: 2023-06-12 00:00:00 +0800
categories: 学习
tags: C++
---

重新理解compare_exchange操作。

### 背景

C++11中提供了两种CAS（compare-and-swap）操作

- `compare_exchange_weak`
- `compare_exchange_strong`

它们的声明如下

```cpp
bool compare_exchange_weak(T& expected,
                           T desired,
                           std::memory_order success,
                           std::memory_order failure) noexcept;

bool compare_exchange_strong(T& expected,
                             T desired,
                             std::memory_order success,
                             std::memory_order failure) noexcept;
```

CAS操作能够对某个原子变量在满足某种条件时，修改这个原子变量的值。比如这样一个操作就等同于以下两个逻辑

`bool success = x.compare_exchange_strong(y, z);`

- *If x==y, make x=z and return true*
- *Otherwise, set y=x and return false*

也就是x的当前值和期望值y相等时，执行read-modify-write操作，把x修改为z。而x的当前值和期望值y不想等时，会把y赋值为x的当前值，也就是执行了`atomic::load`这个load operation。本质上CAS操作提供的就是有条件替换的能力，也就是conditional exchange，与之相对应的是`atomic::exchange`提供的是无条件替换。另一方面，由于atomic只提供了有限的原子运算能力，比如`atomic<int>`只能原子的进行加减运算，却没有原子的乘除，通过CAS操作，就能够完成所有的运算，甚至可以说CAS操作是很多lock-free programming的最核心要素。

那为什么有两种CAS操作的函数呢？在cppreference中针对`compare_exchange_weak`有这样一句话：

*The weak forms of the functions are allowed to fail spuriously, that is, act as if `*this != expected` even if they are equal.*

也就是说，`compare_exchange_weak`可能会fail spuriously，即`x.compare_exchange_weak(y, z);`可能会在`x == y`时仍然返回false，且没有发生read-modify-write。所以通常在使用`compare_exchange_weak`时，都需要一个循环，比如下面这样：

```cpp
while (!x.compare_exchange_strong(y, z)) {}
```

为什么`compare_exchange_weak`会`spuriously fail`？到底其中发生了什么？

### 正题

第一个问题是，当`spuriously fail`时，`y`在`compare_exchange_weak`后是什么值？

根据上面的描述，当CAS失败时，会设置y为x的当前值。如果这个CAS操作后没有再执行其他atomic operation，那么`y == x`。换个角度说，当`spuriously fail`时，它也必须让y等于x的当前值。因为只有这样，才能在下一次while循环，重新尝试CAS操作。

```cpp
while (!x.compare_exchange_strong(y, z)) {}
```

第二个问题是，为什么明明`x == y`，`compare_exchange_weak`却不执行修改，而要在下一次循环中重试？

要解释这个问题可能得从atomic operation是如何实现的说起。首先，atomic operation很大程度上是依赖于硬件完成的，比如CPU可能要对某个内存地址获取shared/exclusive ownership，然后再进行相应的load/store operation，也就是读/写操作。

而对于CAS会更特殊一些，它要先进行读操作，根据读操作的结果和期望值进行比对，可能会进行写操作。

> 这个流程和upgradable lock是非常相似的，可以类比理解
> 

`compare_exchange_strong`和`compare_exchange_weak`在上述流程中对内存的ownership是不同的，我们从`compare_exchange_strong`看起。下面是`compare_exchange_strong`在硬件上的一种可能的伪代码实现，注意不是实际代码。

```cpp
bool compare_exchange_strong(T& old_v, T new_v) {
  Lock L;         // Get exclusive access
  T tmp = value;  // Current value of the atomic
  if (tmp != old_v) {
    old_v = tmp;
    return false;
  }
  value = new_v;
  return true;
}
```

伪代码逻辑足够清晰简单，唯一需要注意的是它通过硬件提供的接口（不是我们常用的`std::mutex`），获取了一把某个内存地址的独占锁。

而通常来说，load operation会比store operation更快，因为load对应shared ownership，而store对应exclusive ownership。所以我们可以在真正上锁之前，增加一个fast path，在那里检查是否和期望值相等。如果满足，再获取独占锁，再次比对是否和期望值相同，然后再继续执行。

之所以要在获取独占锁时再检查一次，是因为在获取锁期间，当前值可能会被修改。所以下面的伪代码实际和所谓的double-lock check非常相似，只不过是在硬件层面实现的。

```cpp
bool compare_exchange_strong(T& old_v, T new_v) {
  T tmp = value;  // Current value of the atomic, shared access
  if (tmp != old_v) {
    old_v = tmp;
    return false;
  }
  Lock L;              // Get exclusive access
  tmp = value;         // value could have changed!
  if (tmp != olv_v) {  // Double check
    old_v = tmp;
    return false;
  }
  value = new_v;
  return true;
}
```

那么`compare_exchange_weak`呢？首先需要说明的是，`spuriously fail`不是在所有平台上都会发生的，比如在x86就不会。在这些平台`compare_exchange_weak`的实现可能和`compare_exchange_strong`完全相同。那些可能发生`spuriously fail`的平台呢？在这些平台上，可能由于对某个地址获取独占锁的开销太大，或者是硬件提供了TimedLock的能力（类比`std::timed_mutex`），`compare_exchange_weak`采用了类似如下的实现：

```cpp
bool compare_exchange_weak(T& old_v, T new_v) {
  T tmp = value;  // Current value of the atomic, shared access
  if (tmp != old_v) {
    old_v = tmp;
    return false;
  }
  TimedLock L;                    // Get exclusive access or fail
  if (!L.locked()) return false;  // old_v is correct
  tmp = value;                    // value could have changed!
  if (tmp != olv_v) {             // Double check
    old_v = tmp;
    return false;
  }
  value = new_v;
  return true;
}
```

唯一的区别点就是从独占锁换成了`TimedLock`，并且上锁失败之后就返回false了。

```cpp
  TimedLock L;                    // Get exclusive access or fail
  if (!L.locked()) return false;  // old_v is correct
```

这样就能完美解释`spuriously fail`出现的现象了。如果上锁失败，此时期望值可能和当前值相等，也可能不相等。而相等时，也不进行任何操作，直接返回false，也就是所谓的`spuriously fail`。

最后一个问题是，什么情况可能会出现上锁失败，最终导致`spuriously fail`呢？一种可能的解释就是，当需要上锁时，在某些平台上，这个CPU需要不断询问其他CPU能否获取exclusive ownership，而这个过程在一段时间内其他CPU无法及时响应，最终导致超时。

### Reference

C++ atomics: from basic to advanced