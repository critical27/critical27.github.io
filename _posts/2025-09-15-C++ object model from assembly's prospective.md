---
layout: single
title: C++ object model from assembly's prospective
date: 2025-09-15 00:00:00 +0800
categories: 学习
tags: C++
---

结合CppCon2015的演讲Intro to the C++ Object Model，从汇编的视角再理解一下C++对象模型。我的环境是Linux和gcc 13.3。

### Layout of a class

首先我们看下一个只有成员变量的类的layout

```cpp
#include <stdio.h>

struct Complex {
  float real;
  float imag;
};

int main() {
  Complex c;
  printf("sizeof(Complex): %ld\n", sizeof(Complex));
  printf("address of c: %p\n", &c);
  printf("address of c.real: %p\n", &c.real);
  printf("address of c.imag: %p\n", &c.imag);
}

```

```cpp
sizeof(Complex): 8
address of c: 0x7ffc1d73d930
address of c.real: 0x7ffc1d73d930
address of c.imag: 0x7ffc1d73d934
```

关于类中的成员变量，标准中是这么说的：

> From Working Draft N4431, Clause 9.2, Note 13:

Nonstatic data members of a (non-union) class with the same access control (Clause [[class.access]](https://timsong-cpp.github.io/cppwp/std11/class.access)) are allocated so that later members have higher addresses within a class object. The order of allocation of non-static data members with different access control is unspecified ([[class.access]](https://timsong-cpp.github.io/cppwp/std11/class.access)). Implementation alignment requirements might cause two adjacent members not to be allocated immediately after each other; so might requirements for space for managing virtual functions ([[class.virtual]](https://timsong-cpp.github.io/cppwp/std11/class.virtual)) and virtual base classes ([[class.mi]](https://timsong-cpp.github.io/cppwp/std11/class.mi)).
>

在gcc 13.3 -O0下汇编代码如下

```nasm
    endbr64
    push   %rbp
    mov    %rsp,%rbp
    sub    $0x10,%rsp

    ; 保存canary到栈上
    mov    %fs:0x28,%rax
    mov    %rax,-0x8(%rbp)
    xor    %eax,%eax

    ; print sizeof(Complex)
    mov    $0x8,%esi
    lea    0xe74(%rip),%rax
    mov    %rax,%rdi
    mov    $0x0,%eax
    call   0x555555555070 <printf@plt>

    ; print address of c, which is -0x10(%rbp)
    lea    -0x10(%rbp),%rax
    mov    %rax,%rsi
    lea    0xe6f(%rip),%rax
    mov    %rax,%rdi
    mov    $0x0,%eax
    call   0x555555555070 <printf@plt>

    ; print address of c.real, which is -0x10(%rbp)
    lea    -0x10(%rbp),%rax
    mov    %rax,%rsi
    lea    0xe66(%rip),%rax
    mov    %rax,%rdi
    mov    $0x0,%eax
    call   0x555555555070 <printf@plt>

    ; print address of c.imag, which is -0x10(%rbp) + 4
    lea    -0x10(%rbp),%rax
    add    $0x4,%rax
    mov    %rax,%rsi
    lea    0xe5e(%rip),%rax
    mov    %rax,%rdi
    mov    $0x0,%eax
    call   0x555555555070 <printf@plt>

    ; 检查canary是否被修改
    mov    $0x0,%eax
    mov    -0x8(%rbp),%rdx
    sub    %fs:0x28,%rdx
    je     0x55555555520b <main+162>
    call   0x555555555060 <__stack_chk_fail@plt>

    leave
    ret
```

对应的内存layout如下所示

```nasm
+---------------------------------+
| return address of main's caller |
+---------------------------------+
| main's caller %rbp              |  <- %rbp
+---------------------------------+
| canary                          |  <- %rbp - 8
+---------------------------------+
| c.imag                          |  <- %rbp - 12
+---------------------------------+
| c.real                          |  <- %rbp - 16 = %rsp
+---------------------------------+
```

### x86-64 System V ABI calling convention

这里回顾下x86-64中调用约定中关于%rsp的部分。在x86-64 System V ABI中，在函数调用时，栈指针`%rsp`必须保持16字节对齐：

- caller负责保证在`call`指令执行时，`%rsp`是16字节对齐的

    ```nasm
    caller:
        ; ...
        call callee          ; 调用call时 caller能确保 %rsp % 16 = 0
    ```

- 调用call时，caller能确保`%rsp % 16 = 0`
- call指令执行后，会将caller的下一条指令地址压榨，此时`%rsp % 16 = 8`
- 一般callee最开始都有prologue，最开始会执行`pushq %rbp`，这时候就能保证`%rsp % 16 = 0`

    ```nasm
    callee:
        pushq %rbp
        movq %rsp, %rbp
        ; ...
    ```

- callee最后会有epilogue，调用完之后同样能保证`%rsp % 16 = 0`

    ```nasm
    leave
    ret
    ```


如果在没有prologue的情况下，也需要保证`%rsp`必须保持16字节对齐。比如在Compiler explorer gcc 13.3 -Og生成如下汇编代码：

```nasm
.LC0:
        .string "sizeof(Complex): %ld\n"
.LC1:
        .string "address of c: %p\n"
.LC2:
        .string "address of c.real: %p\n"
.LC3:
        .string "address of c.imag: %p\n"
main:
        subq    $24, %rsp

        ; print sizeof(Complex)
        movl    $8, %esi
        movl    $.LC0, %edi
        movl    $0, %eax
        call    printf

        ; print address of c
        leaq    8(%rsp), %rsi
        movl    $.LC1, %edi
        movl    $0, %eax
        call    printf

        ; print address of c.real
        leaq    8(%rsp), %rsi
        movl    $.LC2, %edi
        movl    $0, %eax
        call    printf

        ; print address of c.image
        leaq    12(%rsp), %rsi
        movl    $.LC3, %edi
        movl    $0, %eax
        call    printf
        movl    $0, %eax
        addq    $24, %rsp
        ret
```

注意到`main`中并没有压栈`%rbp`，而是通过调整`%rsp`保证16字节对齐。进入`main`时，`%rsp % 16 = 8`。由于要给Complex分配8字节空间，理论上只需要`subq  $8, %rsp`就能同时满足`%rsp`16字节对齐以及局部变量内存分配。但gcc 13.3 -Og编译下是直接分配了24字节，也能满足以上要求，只不过有16个字节是没有使用的。对应内存layout如下所示：

```
+---------------------------------+
| return address of main's caller |  <- %rsp + 24
+---------------------------------+
| unused                          |  <- %rsp + 16
+---------------------------------+
| c.imag                          |  <- %rsp + 12
+---------------------------------+
| c.real                          |  <- %rsp + 8
+---------------------------------+
| unused                          |  <- %rsp
+---------------------------------+
```

总结一下，x86-64 System V ABI的调用约定规定了在`call`指令执行时，`%rsp`必须是16的倍数：

- caller执行`call`指令会将8字节的返回地址压栈
- callee会在prologue中再将`%rbp`压栈
- 两次压栈后仍然能保证`%rsp`是16的倍数
- callee在epilogue中会将之前caller的`%rbp`和返回地址`%rip`弹出栈

### Layout of a derived class

接下来，我们看一下没有虚函数的派生类layout

```cpp
#include <cstdio>

struct Complex {
    float real;
    float imag;
};

struct Derived : public Complex {
    float angle;
};

int main() {
    Derived d;
    printf("address of d: %p\n", &d);
    printf("address of d.real: %p\n", &d.real);
    printf("address of d.imag: %p\n", &d.imag);
    printf("address of d.angle: %p\n", &d.angle);
}
```

gcc 13.3 -O0生成如下汇编代码

```nasm
    endbr64
    push   %rbp
    mov    %rsp,%rbp
    sub    $0x20,%rsp

    mov    %fs:0x28,%rax
    mov    %rax,-0x8(%rbp)
    xor    %eax,%eax

    ; print address of d
    lea    -0x14(%rbp),%rax
    mov    %rax,%rsi
    lea    0xe72(%rip),%rax
    mov    %rax,%rdi
    mov    $0x0,%eax
    call   0x555555555070 <printf@plt>

    ; print address of d.real
    lea    -0x14(%rbp),%rax
    mov    %rax,%rsi
    lea    0xe69(%rip),%rax
    mov    %rax,%rdi
    mov    $0x0,%eax
    call   0x555555555070 <printf@plt>

    ; print address of d.imag
    lea    -0x14(%rbp),%rax
    add    $0x4,%rax
    mov    %rax,%rsi
    lea    0xe61(%rip),%rax
    mov    %rax,%rdi
    mov    $0x0,%eax
    call   0x555555555070 <printf@plt>

    ; print address of d.angle
    lea    -0x14(%rbp),%rax
    add    $0x8,%rax
    mov    %rax,%rsi
    lea    0xe59(%rip),%rax
    mov    %rax,%rdi
    mov    $0x0,%eax
    call   0x555555555070 <printf@plt>

    mov    $0x0,%eax
    mov    -0x8(%rbp),%rdx
    sub    %fs:0x28,%rdx
    je     0x555555555211 <main()+168>
    call   0x555555555060 <__stack_chk_fail@plt>

    leave
    ret
```

进入到main执行完prologue之后的内存layout如下：

```cpp
+---------------------------------+
| return address of main's caller |
+---------------------------------+
| main's caller %rbp              |  <- %rbp
+---------------------------------+
| canary                          |  <- %rbp - 8
+---------------------------------+
| d.angle                         |  <- %rbp - 12
+---------------------------------+
| d.imag                          |  <- %rbp - 16
+---------------------------------+
| d.real                          |  <- %rbp - 20
+---------------------------------+
| unused                          |  <- %rbp - 32 = %rsp
+---------------------------------+
```

Inheritance works by “extending” the object。 Think of inheritance as essentially “stacking” subclasses on top of base classes.

本质上，派生类的layout就是在基类基础上继续叠加成员变量。两种写法本质上等价：

```cpp
struct Complex {
  float real;
  float imag;
};

struct Derived : public Complex {
  float angle;
};
```

```cpp
struct Derived {
  struct {
    float real;
    float imag;
  };
  float angle;
};
```

### Member function

接下来看下非需成员函数是如何实现的：

```cpp
#include <cmath>
#include <cstdio>

struct Complex {
    float func() const { return real + imag; }
    float real;
    float imag;
};

void print(const Complex &c) {
    printf("Abs: %f\n", c.func());
}
```

编译后查看下符号表，可以看到被name mangled之后的成员函数，通过`c++filt`可以看到代码中的函数并。

```bash
$ gcc -o test.o -c test.cpp

$ nm test.o                                                                                  doodle.wang@k8s-worker5 10:58:22
                 U printf
0000000000000000 T _Z5printRK7Complex
0000000000000000 W _ZNK7Complex4funcEv

$ c++filt _ZNK7Complex4funcEv                                                                doodle.wang@k8s-worker5 10:59:27
Complex::func() const
```

实际上，所有的成员函数实际会被处理为一个被name mangled的free function，比如`func`被转换成`_ZNK7Complex4funcEv`。而编译器还会把所有成员函数转换出来的free function的第一个参数设置为this指针。

```cpp
float _ZNK7Complex4funcEv(const Complex* this);
```

这么做的原因是，通过传递`this`指针，不同对象的成员函数就可以使用同一份代码，成员函数也不需要保存在对象中。`func`本质上就等同于如下实现：

```cpp
float Complex::func(Complex const * this) {
    return this->real + this->imag;
}
```

对同一个类的不同对象的成员函数时，只需要将`this`指针传递到这个free function进行调用即可。

```cpp
float _ZNK7Complex4funcEv(Complex const * this) {
    return this->real + this->imag;
}
```

我们看下成员函数`func`的汇编代码，`%rdi`作为第一个参数，保存了`this`。之后就可以通过`this`指针加上固定的偏移量，读取对应的成员变量。

```bash
$ objdump -d --no-show-raw-insn  test.o                                                      doodle.wang@k8s-worker5 11:01:35

0000000000000000 <_ZNK7Complex4funcEv>:
   0:   endbr64
   4:   push   %rbp
   5:   mov    %rsp,%rbp
   8:   mov    %rdi,-0x8(%rbp)      ; 保存this
   c:   mov    -0x8(%rbp),%rax
  10:   movss  (%rax),%xmm1.        ; 加载this->real到xmm1
  14:   mov    -0x8(%rbp),%rax
  18:   movss  0x4(%rax),%xmm0      ; 加载this->imag到xmm0
  1d:   addss  %xmm1,%xmm0.         ; xmm0 = xmm0 + xmm1
  21:   pop    %rbp
  22:   ret
```

### Virtual function

接下来我们来理解一下虚函数的调用机制。不妨想想下面的代码会输出什么：

```cpp
#include <iostream>
using std::cout;

struct Erdos {
  void whoAmI() { cout << "I am Erdos\n"; }
  virtual void whoAmIReally() { cout << "I really am Erdos\n"; }
};

struct Fermat : public Erdos {
  void whoAmI() { cout << "I am Fermat\n"; }
  virtual void whoAmIReally() { cout << "I really am Fermat\n"; }
};

int main() {
  Fermat f;
  f.whoAmI();
  f.whoAmIReally();

  Erdos &e = f;
  e.whoAmI();
  e.whoAmIReally();
}
```

结果为：

```cpp
I am Fermat
I really am Fermat
I am Erdos
I really am Fermat
```

这里有两点需要理解：

- 非虚函数调用是在编译期确定
- 虚函数代用是在运行期确定

> Non-virtual member functions bind statically. Function resolution occurs at compile time. Virtual member functions bind dynamically. Function resolution occurs when the object is created.
>

对于非虚函数`whoAmI`来说，编译器已经知道`f`是一个`Fermat`对象。而`Fermat`中有一个`Erdos`对象，所以可以通过`f`获取其基类`Erdos`对象。因此在处理`e.whoAmI()`时，编译器也知道有一个`Erdos`对象。

对于虚函数`whoAmIReally`来说，到底调用的是基类方法还是派生类的方法，在这个对象创建时就已经决定了。在构造`f`这个`Fermat`对象时，已经确定`f`使用的是`Fermat`的虚函数表。因此调用`f.whoAmIReally`时就会输出`I really am Fermat`。同理，即便可以通过`f`获取到其基类对象`e`，但最开始创建`f`时就已经确定了`f`这个对象的`vptr`指向`Fermet`的虚函数表，因此调用`e.whoAmIReally`时也是输出`I really am Fermat`。

我们可以结合汇编代码看看是如何完成上述工作的。为了便于理解，这里用的是compiler explorer中gcc 13.3 -O0的代码：

```nasm
main:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp

    movl    $vtable for Fermat+16, %eax
    movq    %rax, -16(%rbp)         ; 在栈上创建Fermat对象 实际上只有vptr

    leaq    -16(%rbp), %rax         ; 获取f的地址
    movq    %rax, %rdi
    call    Fermat::whoAmI()        ; 调用非虚方法Fermat::whoAmI

    ; 这里虽然调用的是虚函数 但编译器已经确定f是Fermat对象 不需要通过虚函数表进行调用
    leaq    -16(%rbp), %rax         ; 获取f的地址
    movq    %rax, %rdi
    call    Fermat::whoAmIReally()

    leaq    -16(%rbp), %rax         ; 获取f的地址
    movq    %rax, -8(%rbp)          ; 把f的地址也保存到-8(%rbp)
    movq    -8(%rbp), %rax
    movq    %rax, %rdi
    call    Erdos::whoAmI()         ; 调用非虚方法

    movq    -8(%rbp), %rax          ; 获取f的地址 vptr
    movq    (%rax), %rax            ; rax = *rax = *vptr = vtable地址
    movq    (%rax), %rdx            ; rdx = *(vtable第一个虚函数的地址)
    movq    -8(%rbp), %rax          ; 获取f的地址
    movq    %rax, %rdi
    call    *%rdx                   ; 调用虚函数

    movl    $0, %eax
    leave
    ret
```

注意到，编译器在处理`f.whoAmIReally()`时，虽然`whoAmIReally`是一个虚函数，但由于这里知道`f`的类型就是`Fermat`，因此不需要通过虚函数表进行虚函数调用。

而在处理`e.whoAmIReally()`时，首先是获取了对象`f`的地址，由于`Fermat`和基类`Erdos`中没有任何成员变量，所以这个地址保存的就是`f`的`vptr`地址。`f`是一个`Fermat`对象，所以`vptr`指向`Fermat`的`vtable`地址。`Fermat`只有一个虚函数，所以`Fermat`的`vtable`的地址也就是`Fermat`的`whoAmIReally`函数地址。相关地址的关系如下：

```cpp
*(&f) = vptr
*vptr = address of Fermat's vtable
*(vtable) = &Fermat::whoAmIReally
```

### Layout of derived class with virtual function

既然带有虚函数的派生类和基类中有虚函数表，那就继续看看他们的layout。

```cpp
#include <cmath>
#include <iostream>
using std::cout;

struct Complex {
  virtual ~Complex() = default;
  virtual float Abs() { return std::hypot(real, imag); }
  float real;
  float imag;
};

struct Derived : public Complex {
  virtual ~Derived() = default;
  virtual float Abs() { return std::hypot(std::hypot(real, imag), angle); }
  float angle;
};

int main() {
  cout << "sizeof(float): " << sizeof(float) << "\n";
  cout << "sizeof(void*): " << sizeof(void *) << "\n";
  cout << "sizeof(Complex): " << sizeof(Complex) << "\n";
  cout << "sizeof(Derived): " << sizeof(Derived) << "\n";
}
```

```cpp
sizeof(float): 4
sizeof(void*): 8
sizeof(Complex): 16
sizeof(Derived): 24
```

`Complex`的大小为16，因为现在有虚函数，所以需要额外8字节保存`vptr`，即：

`sizeof(Complex) = sizeof(vptr) + sizeof(real) + sizeof(imag) = 8 + 4 + 4 = 16`

`Derived`的大小为24，在`Complex`的基础上，额外需要4字节保存`angle`，由于不满足8字节对齐，所以增加了4字节padding，总大小为24。注意派生类的`vptr`也是保存在对象的首地址中。

`sizeof(Derived) = sizeof(Complex) + sizeof(angle) + 4 byte padding = 16 + 4 + 4 = 24`

### vtable review

`Complex`和`Derived`类各有一个虚函数表，每个对象在构造时，会将`vptr`指针指向相应的虚函数表，之后再调用虚函数时，根据`vptr`找到虚函数表，然后调用相应虚函数。

![figure]({{'/archive/vtable-1.png' | prepend: site.baseurl}})

> `Complex`对象`c`的`vptr`指向`Complex`类的虚函数表
>

![figure]({{'/archive/vtable-2.png' | prepend: site.baseurl}})

> `Derived`对象`c`的`vptr`指向`Derived`类的虚函数表
>

### Constructor of derived class

那么一个对象的`vptr`又是在什么时候被复制的呢？答案就是构造函数时，因为构造一个对象的时候，知道这个对象具体是哪个类，也就能在构造对象时，将`vptr`初始化为对应类的虚函数表地址了。

```cpp
#include <cmath>
#include <iostream>
using std::cout;

struct Complex {
  Complex() : real(0), imag(0) {}
  virtual ~Complex() = default;
  virtual float Abs() { return std::hypot(real, imag); }
  float real;
  float imag;
};

struct Derived : public Complex {
  Derived() : angle(0) {}
  virtual ~Derived() = default;
  float angle;
  virtual float Abs() { return std::hypot(std::hypot(real, imag), angle); }
};

int main() {
  Derived d;
}
```

基类`Complex`的构造函数如下，注意到构造函数也会通过第一个参数`%rdi`传递`this`指针，可以理解为在给定地址构造对象。相关逻辑比较简单，本质上就是初始化`*%rdi`为`Complex`类的虚函数表地址，保存`*(%rdi + 8)`为0，`*(%rdi + 12)`为0，分别对应`Complex`中的`real`和`imag`。

```nasm
Complex::Complex() [base object constructor]:
    pushq   %rbp
    movq    %rsp, %rbp

    movq    %rdi, -8(%rbp)                 ; 保存this到-8(%rbp)
    movl    $vtable for Complex+16, %edx   ; 保存Complex的vtable地址到%edx
    movq    -8(%rbp), %rax
    movq    %rdx, (%rax)                   ; 把vtable地址保存到*rdi

    movq    -8(%rbp), %rax
    pxor    %xmm0, %xmm0
    movss   %xmm0, 8(%rax)                 ; 初始化*(%rdi + 8)为0 即this->real为0

    movq    -8(%rbp), %rax
    pxor    %xmm0, %xmm0
    movss   %xmm0, 12(%rax)                ; 初始化*(%rdi + 12)为0 即this->imag为0
    nop

    popq    %rbp
    ret
```

派生类`Derived`的构造函数如下，注意是先调用了基类的构造，然后再更新`vptr`值。

```nasm
Derived::Derived() [base object constructor]:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp                       ; 预留24字节用于构造对象

    movq    %rdi, -8(%rbp)
    movq    -8(%rbp), %rax
    movq    %rax, %rdi
    call    Complex::Complex() [base object constructor]

    movl    $vtable for Derived+16, %edx    ; 保存Complex的vtable地址到%edx
    movq    -8(%rbp), %rax
    movq    %rdx, (%rax)                    ; 把vtable地址保存到*rdi

    movq    -8(%rbp), %rax
    pxor    %xmm0, %xmm0
    movss   %xmm0, 16(%rax)                 ; 初始化*(%rdi + 16为0 即this->angle为0
    nop

    leave
    ret
```

从上面的代码中可以看到，构造一个有虚函数的派生类对象时，依次进行了以下操作：

1. 调用基类构造
2. 将对象首地址设置为**基类**的虚函数表地址（注意不是派生类）
3. 按初始化列表初始化基类的成员变量
4. 执行基类构造函数体中的剩余操作
5. 更新对象首地址为**派生类**的虚函数地址
6. 按初始化列表初始化派生类的成员变量
7. 执行派生类构造函数体中的剩余操作

研究清楚了调用派生类构造函数的执行顺序后，可以思考下面代码会输出什么结果。

```cpp
#include <iostream>
using std::cout;

struct Erdos {
  Erdos() { whoAmIReally(); }
  virtual void whoAmIReally() { cout << "I really am Erdos\n"; }
};

struct Fermat : public Erdos {
  virtual void whoAmIReally() { cout << "I really am Fermat\n"; }
};

int main() {
  Fermat f;
}
```

结果是，稍微有点反直觉：

```cpp
I really am Erdos
```

按照之前分析的结果，在构造`Fermat`中的`Erdos`基类对象时，`vptr`会被首先设置为`Erdos`的虚函数表地址，之后调用`whoAmIReally`时，调用的自然是`Erdos::whoAmIReally`。而当构造完基类对象后，`vptr`才被更新为正确的子类虚函数表。

> Effective C++中有一条就是Never call virtual functions during construction or destruction.
>

构造函数的相关汇编如下，可以再结合加深理解。

```cpp
Erdos::Erdos() [base object constructor]:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp
    movq    %rdi, -8(%rbp)
    movl    $vtable for Erdos+16, %edx
    movq    -8(%rbp), %rax
    movq    %rdx, (%rax)
    movq    -8(%rbp), %rax
    movq    %rax, %rdi
    call    Erdos::whoAmIReally()
    nop
    leave
    ret

Fermat::Fermat() [base object constructor]:
    pushq   %rbp
    movq    %rsp, %rbp
    subq    $16, %rsp
    movq    %rdi, -8(%rbp)
    movq    -8(%rbp), %rax
    movq    %rax, %rdi
    call    Erdos::Erdos() [base object constructor]
    movl    $vtable for Fermat+16, %edx
    movq    -8(%rbp), %rax
    movq    %rdx, (%rax)
    nop
    leave
    ret
```

## Refernce

[CppCon 2015: Richard Powell “Intro to the C++ Object Model" - YouTube](https://www.youtube.com/watch?v=iLiDezv_Frk)