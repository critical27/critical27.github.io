---
layout: single
title: dynamic_cast From Scratch
date: 2025-04-10 00:00:00 +0800
categories: 学习
tags: C++
---

这篇文章总结自Arthur O'Dwyer在CppCon上的演讲`dynamic_cast From Scratch`，主要是先介绍运行时多态的相关基础知识，以及如何自己实现一个`dynamic_cast`。

### dynamic_cast

一个正确实现的dynamic_cast应该能完成以下功能，为了简单，我们只用指针之间的转换为例，引用之间的转换类似。给定`a`类型为`A`：

1. 同类型指针的转换，比如`dynamic_cast<A*>(a)`应该返回`a`。
2. up-cast，即子类指针向基类指针的转换，这种场景可以用隐式转换或者`static_cast`也能完成。比如`dynamic_cast<B*>(a)`，如果`B`是`A`的Base subobject，那么返回`A`中的`B`对象的指针，否则返回`nullptr`。
3. down-cast，即基类指针向子类指针的转换。比如`dynamic_cast<B*>(a)`，如果`a`指向的对象类型中有且只有一个`B`对象，那就会返回这个`B`对象的指针，否则返回`nullptr`。
4. side-cast，多继承情况下，会向“兄弟类型“转换。比如`dynamic_cast<B*>(a)`，假设`C`同时public继承自`A`和`B`，此时`C`是A的most-derived类型，且因为`C`有一个`unambiguous`且`public`的基类B，此时会能完成side-cast，返回`a`中的`C`对象中的`B`对象指针。如果不能进行side-cast，会返回nullptr。
5. cast to most-derived object，`dynamic_cast<void*>(a)`会返回A的most-derived类型的指针

以上情况中，只有后三个case涉及到RTTI([run-time type identification](https://en.wikipedia.org/wiki/Run-time_type_information))。想要实现一个`dynamic_cast`，我们需要回顾一些运行时多态的知识。

### 单继承多态

```cpp
class Animal {
    int legs;
    virtual void speak() {
        puts("hi");
    }
    virtual ~Animal();
};

class Cat : public Animal {
    int tails;
    void speak() override {
        printf("Ouch, my %d tails!", tails);
    }
};
```

我们首先简要回顾下多态的原理。基类和子类都有自己的虚函数表，每个对象的都有一个隐藏的数据成员`vptr`，它指向自己类的虚函数表。通过虚函数表，`Animal`和`Cat`都会有各自不同的虚函数调用，比如`speak`方法。

![figure]({{'/archive/dynamic-cast-0.png' | prepend: site.baseurl}})

`Animal`和`Cat`的layout如下所示，本质上Cat is-a Animal。通过一个基类指针`Animal*`，我们就可以实现运行时多态。

![figure]({{'/archive/dynamic-cast-1.png' | prepend: site.baseurl}})

### 多继承多态

引入多继承之后，就会出现所谓的菱形继承问题：

```cpp
class Animal {
    virtual ~Animal();
};
class Cat : public Animal {};
class Dog : public Animal {};
class CatDog : public Cat, public Dog {};
```

各个类的layout如下所示。`Cat`和`Dog`本身都是单继承，其layout都很容易理解。对于`CataDog`，它会拥有多继承中每一个父类对象，最后是`CatDog`自身的成员。

![figure]({{'/archive/dynamic-cast-2.png' | prepend: site.baseurl}})

菱形继承的问题在于，对于一个`CatDog`对象，我们无法直接使用其中的`Animal`，或者说无法区分用的是`Cat`中的`Animal`还是`Dog`中的`Animal`。本质上`CatDog`是两个`Animal`，而不仅是一个`Animal`。

### 虚继承

要解决菱形继承的问题，C++引入了虚继承。通过`Cat`和`Dog`虚继承自`Animal`，`CatDog`中的`Animal`就被去重了。但需要注意的是，去重后的`Animal`在`CatDog`的layout中位置变了，它不再是`Cat`或者`Dog`中的一部分，而是在`CatDog`的最后。

![figure]({{'/archive/dynamic-cast-3.png' | prepend: site.baseurl}})

所以，一但继承关系中出现虚继承之后，在运行时给定一个指针，我们就无法确认其基类在layout中的位置。比如下面的三种情况：

- `Cat`虚继承自`Animal`，此时layout中`Cat`和`Animal`相邻
- `Cat`虚继承自`Animal`，而`CatDog`继承自`Cat`，此时layout中依次为`Cat`，`CatDog`，`Animal`
- 如果引入更多继承关系，那么`Cat`和`Animal`之间的间隔大小在运行时就不可知了。

> 注意我们这里强调的是运行时
>

在静态时，给定一个类型，其layout是根据ABI和编译器等在编译期就确定了的。但引入虚继承后，在运行时，给定一个指针，这个指针指向的对象layout可能完全不同（参照下图）。如果此时我们想通过`dynamic_cast`获取这个对象的`虚基类`，如果无法确定其layout，也就无法确定虚基类指针位置，那么该如何正确实现`dynamic_cast`呢？

![figure]({{'/archive/dynamic-cast-4.png' | prepend: site.baseurl}})

我们现在已知：

1. most-derived类型的layout是确定的
2. 在运行时，对于任何不是most-drived类型的对象，我们无法确定其中虚基类在layout中偏移量

那如果要通过`dynamic_cast`获取任意对象中的虚基类，我们先要进行向下转型，先转成most-derived类型。由于其layout是确定的，也就能正确获取虚基类。所以问题就转换成，我们该如何确定most-derived类型？

### 虚函数表

根据上面的描述，我们可以得到以下结论：在虚继承关系下，想获取一个对象的虚基类对象，需要先获取到这个对象的most-derived类型，而确定most-derived类型的方法就是虚函数表。

假定给定一个`Cat`对象，它虚继承自`Animal`。它实际有两个虚函数表`vtpr`，一个是`Cat`的`vtpr`，一个是`Anmical`的`vptr`。可以看到两个虚函数表中都有`Cat::speak`这个方法，只不过`Animal`虚函数表中的`Cat::speak`方法接受的是一个`Animal*`的调用。

![figure]({{'/archive/dynamic-cast-5.png' | prepend: site.baseurl}})

可以看到两个虚函数表中都有`Cat::speak`这个方法。当一个实际指向`Cat`的`Animal`指针`a`调用speak方法时，实际会发生如下事情：

1. 通过Animal的虚函数表，找到`Cat::speak`这个方法（图中带红色*的那个）
2. 由于`Cat::speak`方法实际需要接受一个`Cat*`作为参数，所以需要从`Animal*`转换成一个`Cat*`（转换的方式我们一会再说）
3. 最终通过获取到的这个`Cat*`，调用`Cat::speak`方法

如何在这个对象中进行`Animal`和`Cat`之间的转换呢？给定一个`Cat`对象c，它其中有几个变量：

- 指向`Cat`的`vptr`，`Cat`的成员变量`tails`
- 指向`Animal`的`vptr`，`Animal`的成员变量`legs`

如果我们想进行转换，肯定不能直接在`c`这个变量中加减一个偏移量来获取类型。这是因为`Cat`这个类型可能还有其他子类，所以`c`的`layout`中的`Cat`和`Animal`这两部分之间还有其他成员。

![figure]({{'/archive/dynamic-cast-6.png' | prepend: site.baseurl}})

而在虚函数表中我们额外保存了一些信息：

- `Animal-offset`则是`Cat`到`Animal`的偏移量
- `md-offset`是当前`vtpr`到most-derived类，也就是`Cat`的偏移量

通过`c`找到`Cat`的虚函数表，发现`Cat`到`Animal`之前的偏移量`Animal-offset`为16，所以只需要将`Cat`的`vptr`加16就可以得到`Animal`的`vptr`。进而也就完成了`Cat`到`Animal`的向上转型，即获取虚基类。

同理，如果给定一个`Animal`指针`a`，可以通过`a`找到`Animal`的虚函数表，如果想获取其most-derived类型，可以根据`md-offset`为-16，通过`Animal`的`vptr`就可以得到`Cat`的`vptr`。进而也就完成了`Animal`到`Cat`的向下转型，即获取most-derived类型。

对于上图中的代码，现在想获取`c->legs`的汇编代码解释如下：

- `movq (%rdi), %rax` 将`c`指针保存到rax中
- `movq -24(%rax), %rdx` 将`%rax - 24`内存地址中的值，也就是`Animal-offset`，保存到rdx中

    > `c`指针指向的首地址就是`Cat`的`vptr`地址，也就是`(%rax)`，它指向`Cat`的第一个方法`Cat::speak`，而`-24(%rax`)指向`Animal-offset`。
    >
- `movl 8(%rdx, %rdi), %eax` 将`%rdx + %rdi + 8`中的值保存到eax中，也就是`legs`的值

### TLDR

最后我们总结一下虚函数表中的内容：

- 虚基类的偏移量（只在most-derived type的虚函数表中存在）
- most-derived type的偏移量
- type_info，用于RTTI
- 各种虚函数

![figure]({{'/archive/dynamic-cast-7.png' | prepend: site.baseurl}})

以及几种常见的真正的dynamic_cast：

- `dynamic_cast<void*>` to the most-derived class
- `dynamic_cast` across the hierarchy, to a sibling base
- `dynamic_cast` from base to derived

![figure]({{'/archive/dynamic-cast-8.png' | prepend: site.baseurl}})

### Reference

[CppCon 2017: Arthur O'Dwyer “dynamic_cast From Scratch”](https://www.youtube.com/watch?v=QzJL-8WbpuU)