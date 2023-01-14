---
layout: single
title: "What is type traits"
date: 2022-02-02 00:00:00 +0800
categories: 学习
tags: C++
---

什么是`Type traits`?

`Type traits` are a clever technique used in C++ template metaprogramming that gives you the ability to inspect and transform the properties of types.

1. `Type traits`通常被用在`conditional compliation`中(c++17引入)，编译器来根据不同的输入类型，生成不同的代码。（后面举例）

比如，给定一个类型`T`，它可以是`int`，`bool`，`std::vector`或者任意类型，借助`type traits`，我们可以对编译器问各种问题：

* T是整型吗
* T是函数吗
* T是指针吗
* T是一个类吗
* T有析构函数吗
* T可以复制吗
* T会抛异常吗
* T有一个名字叫reserve的成员函数吗
* ...

> 有了`type traits`，在mordern c++中，可以取代SFINAE

2. `Type traits`还可以对类型做转换

比如给定类型`T`，我们可以做如下转换：

* 增加或者删除`const`描述符
* 增加或者删除引用
* 增加或者删除指针
* 转换称为一个有符号数或者无符号数
* ...

这个通常只会在写一个library的时候才会见到，绝大多数应用程序是很难见到类似用法的。

`type traits`只发生在编译器，没有任何运行期消耗，就是一个模板元编程的技巧。

### What is a type trait?

本质上`type traits`就是一个**模板类**，这个模板类包含若干成员常量(`member constant`)，这些常量：

* 要么是之前向编译器询问的问题的答案
* 要么是类型转换之后的结果

我们分别举例：

```c++
template<typename T>
struct is_floating_point;
```

`std::is_floating_point`是`<type_traits>`里的一个`type traits`，作用就是告知`T`是否是一个浮点数。我们可以看到它其实就是一个`struct`，里面有一个bool常量`value`，根据传入的类型不同，`value`成员会被设置为`true`或者`false`。

另一个用法如下

```c++
template<typename T>
struct remove_reference;
```

`remove_reference`是一个类型转换，将`T&`转换为`T`，同样它也是一个`struct`，里面有一个常量`type`就是转换之后的类型。

### How do I use a type trait?

```c++
#include <iostream>
#include <type_traits>

class Class {};

int main() {
    std::cout << std::is_floating_point<Class>::value << '\n';
    std::cout << std::is_floating_point<float>::value << '\n';
    std::cout << std::is_floating_point<int>::value << '\n';
}
```

运行之后打印结果就是

```text
0
1
0
```

### How does it work exactly?

编译器实际会根据`type_traits`传入的不同类型，会生成多个`struct`，比如对于上面的例子，总共传入了`Class`，`float`，`int`三个类型，编译器就会实际产生类似如下代码（注意其类型是static）：

```c++
struct is_floating_point_Class {
    static const bool value = false;
};

struct is_floating_point_float {
    static const bool value = true;
};

struct is_floating_point_int {
    static const bool value = false;
};
```

所以上面的代码实际就会转换成如下代码

```c++
int main() {
    std::cout << std::is_floating_point_Class::value << '\n';
    std::cout << std::is_floating_point_float::value << '\n';
    std::cout << std::is_floating_point_int::value << '\n';
}
```

编译器根据传入的三个不同类，生成对应的结构体，每个结构体有一个成员变量`value`，我们只需要通过`::`获取其值就可以。再次说明这些都是发生在编译期，没有任何运行期开销。

### Under the hood

真实是否真的是像上面所说？

我们在`c++ insight`运行类上面的代码，可以看到实际运行的代码如下：

```c++
int main() {
  std::operator<<(std::cout.operator<<(std::integral_constant<bool, false>::value), '\n');
  std::operator<<(std::cout.operator<<(std::integral_constant<bool, true>::value), '\n');
  std::operator<<(std::cout.operator<<(std::integral_constant<bool, false>::value), '\n');
}
```

也就是`is_floating_point`实际被转换为了另一个`type_traits` (integral_constant)，`c++ reference`中给出的一种可能实现如下

```c++
template< class T >
struct is_floating_point
     : std::integral_constant<
         bool,
         std::is_same<float, typename std::remove_cv<T>::type>::value  ||
         std::is_same<double, typename std::remove_cv<T>::type>::value  ||
         std::is_same<long double, typename std::remove_cv<T>::type>::value
     > {};
```

### How to use type traits

* Writing algorithms
* Restricting templates using enable_if
* Provide optimized versions of generic code for some types

#### conditional compilation

假设再写一个算法，这个算法对于有符号数和无符号数有不同处理，那么就可以利用`type_traits`

```c++
void algorithm_signed  (int i)      { /*...*/ }
void algorithm_unsigned(unsigned u) { /*...*/ }


template <typename T>
void algorithm(T t)
{
    if constexpr(std::is_signed<T>::value)
        algorithm_signed(t);
    else
    if constexpr (std::is_unsigned<T>::value)
        algorithm_unsigned(t);
    else
        static_assert(std::is_signed<T>::value || std::is_unsigned<T>::value, "Must be signed or unsigned!");
}
```

这个模板函数`algorithm`就像一个分发器，当传入有符号数的时候调用的是`algorithm_signed`，当传入无符号数时则会调用`algorithm_unsigned`，而当既不是有符号数也不是无符号数时，则会直接在编译器报错。

```c++
algorithm(3);       // T is int, include algorithm_signed()

unsigned x = 3;
algorithm(x);       // T is unsigned int, include algorithm_unsigned()

algorithm("hello"); // T is string, build error!
```

#### altering types

类型转换其实在标准库中会非常常见，比如`std::move`本质上就是把一个类型`T`转换为右值引用`T&&`，实际上`std::move`本质上就是调用了`std::remove_reference`这个`type traits`。`std::move`实际的操作如下：

1. 如果传入的类型`T`带引用`&`则把它去掉(没有则不做任何操作)，此时的类型为`T'`
2. `T'`再转换为右值引用`T'&&`

一种可能实现如下：

```c++
template <typename T>
typename remove_reference<T>::type&& move(T&& arg)
{
  return static_cast<typename remove_reference<T>::type&&>(arg);
}
```

实际上`std::move(t)`返回的类型就是`static_cast<typename std::remove_reference<T>::type&&>(t)`

### syntax sugar

在C++14中为了避免`type traits`中的各种`::value`或者`::type`，引入了两个语法糖`_v`和`_t`，如下所示

```c++
std::is_signed<T>::value;     /* ---> */   std::is_signed_v<T>;
std::remove_const<T>::type;   /* ---> */   std::remove_const_t<T>;
```

也就是要使用问编译器类型是否满足特定条件时使用`_v`，而要使用类型转换结果时使用`_t`，绝大多数`<type_traits>`中的`type traits`都有对应的语法糖。

### Template argument deduction

这部分实际还和模板推导有关，等下次再回顾下。

<https://en.cppreference.com/w/cpp/language/template_argument_deduction>

```c++
#include <type_traits>
#include <iostream>

int func(...) { return 0; }

int func(float f) { return 1; }

template <typename T>
typename std::enable_if<std::is_integral<T>::value, int>::type
func(T val) { return 2; }


int main() {
    // 0 2 1 1
    std::cout << func(nullptr) << " ";
    std::cout << func(2) << " ";
    std::cout << func(2.f) << " ";
    std::cout << func(2.0) << std::endl;
}
```

```c++
#include <type_traits>
#include <iostream>

int func(...) { return 0; }

int func(float f) { return 1; }

template <typename T>
typename std::enable_if<std::is_integral<T>::value, int>::type
func(T val) { return 2; }

template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, int>::type
func(T val) { return 3; }

int main() {
    // 0 2 1 3
    std::cout << func(nullptr) << " ";
    std::cout << func(2) << " ";
    std::cout << func(2.f) << " ";
    std::cout << func(2.0) << std::endl;
}
```

### reference

* <https://www.youtube.com/watch?v=VvbTP_k_Df4>
* <https://www.internalpointers.com/post/quick-primer-type-traits-modern-cpp>
