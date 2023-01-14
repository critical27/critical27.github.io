---
layout: single
title: "What is std::initializer_list"
date: 2022-02-02 00:00:00 +0800
categories: 学习
tags: C++
---

什么是`Initializer Lists`?

### 初始化列表(Initializer Lists)是啥？

List-initialization can be used

* as the initializer in a variable definition
* as the initializer in a new-expression
* in a return statement
* as a for-range-initializer
* as a function argument
* as a subscript

```c++
int a={1};
std::complex<double> z{1,2};
new std::vector<std::string>{"once","upon","a","time"};   // 4 string elements
f({"Nicholas","Annemarie"});                              // pass list of two elements to a funcion
return { "Norah" };                                       // return list of one element
int* e{};                                                 // initialization to zero / null pointer
x = double{1};                                            // explicitly construct a double
std::map<std::string,int> anim = { {"bear",4},{"cassowary",2},{"tiger",7} };
```

### Example1

```c++
// Creates a vector of 2 integers of value 2.
std::vector<int> vec(2,2);

// Creates a vector of 3 integers of value 3.
std::vector<int> vec(3,3);

// Creates a vector of 2 integers of value 3. By calling:
// std::vector<int>(std::initializer_list<int>)
std::vector<int> vec{3,3};
```

### Example2

注意unique_ptr的vector会无法通过编译，原因下面会解释。

```c++
// Vector of 2 shared_ptr objects, using C++17’s class template type deduction.
std::vector vec{std::make_shared<int>(1), std::make_shared<int>(2)};

// Failed to compile, because unique_ptr is not copyable
std::vector vec{std::make_unique<int>(1), std::make_unique<int>(2)};
```

### Initializer Lists到底在干啥？

The usage of Initializer Lists that results in an `std::initializer_list` object

实际上使用初始化列表就会生成一个`std::initializer_list`对象，比如：
一个`E`类型的初始化列表就会构造一个`std::initializer_list<E>`对象，这个对象实际上是一个`prvalue`类型的数组，数组中有N个`const E`，其中N是初始化列表中的元素个数。

**数组中的每个元素都是`copy-inialized`，`std::initializer_list<E>`就是那个数组的引用**

> An object of type `std::initializer_list<E>` is constructed from an initializer list as if the implementation generated and materialized a prvalue of type “array of N const E”, where N is the number of elements in the initializer list. Each element of that array is copy-initialized with the corresponding element of the initializer list, and the `std::initializer_list<E>` object is constructed to refer to that array.

```c++
struct X {
  X(std::initializer_list<double> v);
};
X x{ 1,2,3 };
```

上面的代码基本就会转成下面的实现
```c++
// 构造数组
const double __a[3] = {double{1}, double{2}, double{3}};
// std::initializer_list<double>对象指向那个数组，用这个std::initializer_list<double>来构造X
X x(std::initializer_list<double>(__a, __a + 3));
```

> assuming that the implementation can construct an initializer_list object with a pair of pointers.

### 再看Example2

我们回到之前说unique_ptr报错的例子

```c++
#include <memory>
#include <vector>
#include <iostream>

int main() {
    std::vector vec{std::make_unique<int>(1), std::make_unique<int>(2)};
    return 0;
}
```

等价代码如下

```c++
// 构造数组
const std::unique_ptr<int> __a[] = {std::make_unique<int>(1), std::make_unique<int>(2)};
// std::initializer_list<unique_ptr<int>>对象指向那个数组，用这个std::initializer_list<unique_ptr<int>>来构造vec
std::vector vec(std::initializer_list<std::unique_ptr<int>>(__a, __a + 2));
```

在clang编译报错如下(报错里的`__a`是编译器中的，而不是等价代码中的)，可以看到实际在`__a`上构造中的元素时，实际是调用了placement new

`::new ((void*)__p) _Up(_VSTD::forward<_Args>(__args)...);`

而传入的`__args`是`const std::unique_ptr<int> &`类型，构造`_Up`要调用复制构造，而`unique_ptr`没有复制构造，就报错了。

```text
/Library/Developer/CommandLineTools/SDKs/MacOSX12.1.sdk/usr/include/c++/v1/memory:916:28: error: call to implicitly-deleted copy constructor of 'std::unique_ptr<int>'
        ::new ((void*)__p) _Up(_VSTD::forward<_Args>(__args)...);
                           ^   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
/Library/Developer/CommandLineTools/SDKs/MacOSX12.1.sdk/usr/include/c++/v1/__memory/allocator_traits.h:288:13: note: in instantiation of function template specialization 'std::allocator<std::unique_ptr<int>>::construct<std::unique_ptr<int>, const std::unique_ptr<int> &>' requested here
        __a.construct(__p, _VSTD::forward<_Args>(__args)...);
            ^
/Library/Developer/CommandLineTools/SDKs/MacOSX12.1.sdk/usr/include/c++/v1/memory:1047:18: note: in instantiation of function template specialization 'std::allocator_traits<std::allocator<std::unique_ptr<int>>>::construct<std::unique_ptr<int>, const std::unique_ptr<int> &, void>' requested here
        _Traits::construct(__a, _VSTD::__to_address(__begin2), *__begin1);
                 ^
/Library/Developer/CommandLineTools/SDKs/MacOSX12.1.sdk/usr/include/c++/v1/vector:1077:12: note: in instantiation of function template specialization 'std::__construct_range_forward<std::allocator<std::unique_ptr<int>>, const std::unique_ptr<int> *, std::unique_ptr<int> *>' requested here
    _VSTD::__construct_range_forward(this->__alloc(), __first, __last, __tx.__pos_);
           ^
/Library/Developer/CommandLineTools/SDKs/MacOSX12.1.sdk/usr/include/c++/v1/vector:1335:9: note: in instantiation of function template specialization 'std::vector<std::unique_ptr<int>>::__construct_at_end<const std::unique_ptr<int> *>' requested here
        __construct_at_end(__il.begin(), __il.end(), __il.size());
        ^
test.cpp:5:17: note: in instantiation of member function 'std::vector<std::unique_ptr<int>>::vector' requested here
    std::vector vec{std::make_unique<int>(1), std::make_unique<int>(2)};
                ^
/Library/Developer/CommandLineTools/SDKs/MacOSX12.1.sdk/usr/include/c++/v1/memory:1584:3: note: copy constructor is implicitly deleted because 'unique_ptr<int>' has a user-declared move constructor
  unique_ptr(unique_ptr&& __u) _NOEXCEPT
  ^
```

gcc的报错类似，报错的栈中直接出现了`std::uninitialized_copy`

```
In file included from /home/insights/insights.cpp:2:
In file included from /usr/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/vector:65:
/usr/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/stl_construct.h:109:38: error: call to deleted constructor of 'std::unique_ptr<int>'
    { ::new(static_cast<void*>(__p)) _Tp(std::forward<_Args>(__args)...); }
                                     ^   ~~~~~~~~~~~~~~~~~~~~~~~~~~~
/usr/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/stl_uninitialized.h:92:8: note: in instantiation of function template specialization 'std::_Construct<std::unique_ptr<int>, const std::unique_ptr<int> &>' requested here
                std::_Construct(std::__addressof(*__cur), *__first);
                     ^
/usr/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/stl_uninitialized.h:151:2: note: in instantiation of function template specialization 'std::__uninitialized_copy<false>::__uninit_copy<const std::unique_ptr<int> *, std::unique_ptr<int> *>' requested here
        __uninit_copy(__first, __last, __result);
        ^
/usr/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/stl_uninitialized.h:333:19: note: in instantiation of function template specialization 'std::uninitialized_copy<const std::unique_ptr<int> *, std::unique_ptr<int> *>' requested here
    { return std::uninitialized_copy(__first, __last, __result); }
                  ^
/usr/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/stl_vector.h:1585:11: note: in instantiation of function template specialization 'std::__uninitialized_copy_a<const std::unique_ptr<int> *, std::unique_ptr<int> *, std::unique_ptr<int>>' requested here
            std::__uninitialized_copy_a(__first, __last,
                 ^
/usr/bin/../lib/gcc/x86_64-linux-gnu/11/../../../../include/c++/11/bits/stl_vector.h:629:2: note: in instantiation of function template specialization 'std::vector<std::unique_ptr<int>>::_M_range_initialize<const std::unique_ptr<int> *>' requested here
        _M_range_initialize(__l.begin(), __l.end(),
        ^
/home/insights/insights.cpp:6:36: note: in instantiation of member function 'std::vector<std::unique_ptr<int>>::vector' requested here
        std::vector<std::unique_ptr<int>> vec{ std::make_unique<int>(1), std::make_unique<int>(2)};
```

所以在通过`initializer_list<std::unique_ptr<int>>`构造vector的时候需要调用`unique_ptr`的复制构造，也就导致了编译报错。

### Example3

我们可以再看一个例子:

```c++
#include <initializer_list>
#include <iostream>

auto f(int i, int j, int k) {
    return std::initializer_list<int>{i, j, k};
}

int main(int argc, const char *[]) {
    for (int i : f(argc + 1, argc + 2, argc + 3)) {
        std::cout << i << ',';
    }
}
```

这个函数的结果是啥？我们假设`argc`为1

> 我的环境中打印的是 2,42037796,738197505,%

所以这到底是啥？

根据上面的总结，在`f`函数中返回了一个`std::initializer_list<int>`，我们知道`std::initializer_list<int>`指向一个数组，数组长度为3，其中的元素为`i, j, k`。问题关键来了，这个数组在出`f`的作用域时，已经释放了，而此时`std::initializer_list<int>`还指向那个数组，所以打印啥都不足为奇了。

事实上，我的环境中clang是打印了warning的

```
test.cpp:5:38: warning: returning address of local temporary object [-Wreturn-stack-address]
    return std::initializer_list<int>{i, j, k};
                                     ^~~~~~~~~
1 warning generated.
```

### 总结

到这里我们已经明白了使用初始化列表来进行构造时候会发生的事，我们以`std::vector vec{std::make_shared<int>(1), std::make_shared<int>(2)};`为例

1. 创建一个const T类型的数组，并使用初始化列表中的元素来初始化这个数组

`const std::shared_ptr<int> __a[] = {std::make_shared<int>(1), std::make_shared<int>(2)};`

2. 创建一个`std::initializer_list`对象，这个对象指向上面的数组

`std::initializer_list<std::shared_ptr<int>> __temp(__a, __a + 2);`

3. 使用这个`std::initializer_list`对象通过vector的复制构造，完成真正的构造

`std::vector vec(__temp);`

> An `initializer_list<T>` is passed by value. That is required by the overload resolution rules and does not impose overhead because an `initializer_list<T>` object is just a small handle (typically two words) to an array of `T`.

第2步中的`initializer_list`的说明(cpp-reference)

> The underlying array is a temporary array of type `const T[N]`, in which each element is `copy-initialized` (except that narrowing conversions are invalid) from the corresponding element of the original initializer list. **The lifetime of the underlying array is the same as any other temporary object, except that initializing an initializer_list object from the array extends the lifetime of the array exactly like binding a reference to a temporary** (with the same exceptions, such as for initializing a non-static class member). The underlying array may be allocated in read-only memory.

再看下最后一步中，vector的构造的说明，原型如下:

`vector(std::initializer_list<T> init, const Allocator& alloc = Allocator());`

### 到底构造了几个shared_ptr

有了上面的总结，我们应该可以回答下面的代码中构造了几个`shared_ptr`

```c++
int main() {
  std::vector<std::shared_ptr<int>> vec{std::make_shared<int>(1), std::make_shared<int>(2)};

  // 等价于如下代码
  // const std::shared_ptr<int> __a[] = {std::make_shared<int>(1), std::make_shared<int>(2)};
  // std::vector vec(std::initializer_list<std::shared_ptr<int>>(__a, __a + 2));
}
```

在<https://cppinsights.io/>中我们可以看到上面的代码实际被转换成如下代码 vector中实际构造了4个shared_ptr

```c++
int main() {
  std::vector<std::shared_ptr<int>> vec =
    std::vector<std::shared_ptr<int>, std::allocator<std::shared_ptr<int>>>
      {
        std::initializer_list<std::shared_ptr<int>>{
            std::make_shared<int>(1), std::make_shared<int>(2)
        },
        std::allocator<std::shared_ptr<int>>()
      };
}
```

> 两个是在构造initializer_list时构造的
> 另外两个则是在调用vector的复制构造时构造的

在这个main函数中发生的事如下：

* 1 vector allocation
* 2 constructions
* 2 copy constructors
* 1 vector deallocation
* 4 destructions

---

换一个写法 又有多少次呢？

```c++
int main() {
  std::vector<std::shared_ptr<int>> vec;
  vec.emplace_back(std::make_shared<int>(1));
  vec.emplace_back(std::make_shared<int>(2));
}
```

在`vec.emplace_back(std::make_shared<int>(1));`时

* 1 vector allocation
* 1 construction
* 1 move
* 1 destruction

在`vec.emplace_back(std::make_shared<int>(2));`时

* 1 vector reallocation
* 1 construction
* 2 moves
* 2 destructions (moved from objects)

在main结束时

* 1 vector deallocation
* 2 destructions

两种写法对比

| initialzer_list       | emplace_back          |
| --------------------- | --------------------- |
| 1 vector allocation   | 1 vector allocation   |
| 2 constructions       | 2 constructions       |
| 2 copies              | 3 moves               |
| 4 destructions        | 5 destructions        |
| 1 vector deallocation | 1 vector deallocation |
|                       | 1 vector reallocation |

### 那么这个呢？

前面都是以`vector`为例，如果我们使用的是array呢？

```c++
#include <cstdio>
#include <vector>
#include <memory>

int main() {
    std::array<std::shared_ptr<int>, 2> data{std::make_shared<int>(1), std::make_shared<int>(2)};
}
```

首先，`std::array`其实就是C语言的数组，但`array`知道数组大小，也就可以防止数组越界，本质上是下面的实现

```c++
template<typename T, std::size_t Size>
struct array
{
  T data[Size];
};
```

我们先看下这个[List initialization](https://en.cppreference.com/w/cpp/language/list_initialization)，它的所有语法规则如下

```
Direct-list-initialization
  T object { arg1, arg2, ... };                 (1)
  T { arg1, arg2, ... }                         (2)
  new T { arg1, arg2, ... }                     (3)
  Class { T member { arg1, arg2, ... }; };      (4)
  Class::Class() : member{arg1, arg2, ...} {...	(5)

Copy-list-initialization
  T object = {arg1, arg2, ...};	                (6)
  function( { arg1, arg2, ... } )	            (7)
  return { arg1, arg2, ... };                   (8)
  object[ { arg1, arg2, ... } ]               	(9)
  object = { arg1, arg2, ... }                  (10)
  U( { arg1, arg2, ... } )                      (11)
  Class { T member = { arg1, arg2, ... }; };    (12)
```

仔细阅读下面的解释后，array实际是走的其中这一条

`Otherwise, if T is an aggregate type, aggregate initialization is performed.`

再看一眼`Aggregate initialization`的说明

Aggregate initialization initializes aggregates. It is a form of **list-initialization** (since C++11) or **direct initialization** (since C++20)
An aggregate is one of the following types:
* array type
* class type (typically, struct or union), that has
  * no private or protected direct (since C++17)non-static data members
  * no user-declared constructors (until C++11)
  * no user-provided constructors (explicitly defaulted or deleted constructors are allowed) (since C++11) (until C++17)
  * no user-provided, inherited, or explicit constructors (explicitly defaulted or deleted constructors are allowed) (since C++17) (until C++20)
  * no user-declared or inherited constructors (since C++20)
  * no virtual, private, or protected (since C++17) base classes
  * no virtual member functions
  * no default member initializers (since C++11) (until C++14)

所以`std::array<std::shared_ptr<int>, 2> data{std::make_shared<int>(1), std::make_shared<int>(2)};`中，由于`std::array`实际是个aggregate类型，所以走的是`Aggregate initialization`

### reference

* <https://www.youtube.com/watch?v=sSlmmZMFsXQ>
* <https://cppinsights.io/>
