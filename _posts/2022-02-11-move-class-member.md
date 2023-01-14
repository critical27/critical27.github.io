---
layout: single
title: "std::move(object.member) vs std::move(object).member"
date: 2022-02-08 00:00:00 +0800
categories: 学习
tags: C++
---

咬文嚼字...

### Question

给定`E1.E2`表达式，返回的E2类型是啥？

> `E1->E2`会被转换为`(*(E1)).E2`，所以我们都用`E1.E2`来讨论

我们参照<http://eel.is/c++draft/expr.ref#>里的描述来说明

* If E2 is a static data member and the type of E2 is T, then E1.E2 is an lvalue; the expression designates the named member of the class. The type of E1.E2 is T.
* If E2 is declared to have type “reference to T”, then E1.E2 is an lvalue; the type of E1.E2 is T.
* If E2 is a non-static data member and the type of E1 is “cq1 vq1 X”, and the type of E2 is “cq2 vq2 T”, the expression designates the corresponding member subobject of the object designated by the first expression.
  * If E1 is an lvalue, then E1.E2 is an lvalue;
  * otherwise E1.E2 is an xvalue.
* If E2 is a bit-field, E1.E2 is a bit-field.

简单来说，对于非static成员变量和非引用类型成员变量，`E2`都是`E1`的一部分，所以返回的`E1.E2`的类型会根据`E1`的类型来决定，如果`E1`是`lvalue`，`E1.E2`也是`lvalue`，否则`E1.E2`是`xvalue`。

而对于static成员变量或者是引用类型成员变量，`E2`所指向的对象或者static变量都不属于`E1`，因此返回的总是`lvalue`。

### `std::move(object.member)`和`std::move(object).member`有啥区别？

`std::move(object.member)`会无条件将`object.member`转换为一个右值，`move`之后原有的`member`状态会被修改。如果`member`是个共享的变量，那么可能会导致意外的问题。

而`std::move(object).member`则是把`object`转换为右值，然后获取一个成员变量，根据`member`成员变量类型和上面的规则，最后可能会取到一个左值或者右值。

我们看下面的例子

```c++
struct WidgetCfg {
    std::string config_name_;  // non-reference non-static data member
    ...
};

struct Window {
    std::string config_name_;

    Window(WidgetCfg && w) // Constructor accepting a temporary WidgetCfg so we are safe to move from it
    // Either
    : config_name_(std::move(w.config_name_)) // [1] move from temporary. No problem
    // or
    : config_name_(std::move(w).config_name_) // [2] move from temporary. No problem
    {}

    ...
};
```

由于`WidgetCfg`中的`config_name_`是个普通的成员变量，所以无论`Window`的成员初始化列表怎么写，`Window`中的`config_name_`都会用右值来初始化，没有问题。

如果我们把`WidgetCfg`中的`config_name_`改成引用类型，`config_name_`其指向一个shared string，这时会发生啥？

```c++
struct WidgetCfg {
    std::string & config_name_;  // Now it is a reference to a shared global string
    ...
};

struct Window {
    std::string config_name_;  // Oops, we forget to change this

    Window(WidgetCfg && w) // Constructor accepting a temporary WidgetCfg so we are safe to move from it
    // Either
    : config_name_(std::move(w.config_name_)) // [1] move from the global variable, leaves one in the
                                              //     (empty or even dangerous to use) 'moved from' state!
    // or
    : config_name_(std::move(w).config_name_) // [2] not a temporary any more, so makes a copy rather than move.
                                              //     Not so effective but still safe!
    {}

    ...
};
```

`: config_name_(std::move(w.config_name_))`这种写法会把`w.config_name_`所指向的string转为右值，也就是变为emtpy string。
`: config_name_(std::move(w).config_name_)`这种写法虽然是会走复制构造，但是并不会影响shared string。

鉴于以上的问题，文章里面觉得`std::move(object).member`是更好地写法。

### 那么在函数返回值中呢？

直接上例子

```c++
#include <iostream>
#include <type_traits>

struct Internal {
  Internal() { puts("Internal()"); }
  ~Internal() { puts("~Internal()"); }
  Internal(const Internal &) noexcept { puts("Internal(const Internal &)"); }
  Internal(Internal &&) noexcept { puts("Internal(Internal&&)"); }
  Internal& operator=(const Internal&) noexcept { puts("operator=(const Internal&)"); return *this; }
  Internal& operator=(Internal &&) noexcept { puts("operator(Internal &&)"); return *this; }

  std::string fStr{"fff"};
  std::string gStr{"ggg"};
};

struct S {
    std::string f() const & { return internal.fStr; }
    std::string f() && { return std::move(internal.fStr); }

    std::string g() const & { return internal.gStr; }
    std::string g() && { return std::move(internal).gStr; }

    void print() const { std::cout << "fStr: " << internal.fStr << ", gStr: " << internal.gStr << "\n"; }

    Internal internal;
};

int main() {
    std::cout << "---------------------\n";
    {
        S s;
        std::cout << std::move(s).f() << "\n";
        s.print();
    }
    std::cout << "---------------------\n";
    {
        S s;
        std::cout << std::move(s).g() << "\n";
        s.print();
    }
}
```

我们通过`ref-qualifiers`分别写了`f`和`g`的左右值版本，两种办法都可行。

```text
---------------------
Internal()
fff
fStr: , gStr: ggg
~Internal()
---------------------
Internal()
ggg
fStr: fff, gStr: 
~Internal()
```

### reference

* <http://eel.is/c++draft/expr.ref#>
* <https://oliora.github.io/2016/02/12/where-to-put-std-move.html>
