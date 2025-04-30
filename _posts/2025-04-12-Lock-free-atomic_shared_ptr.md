---
layout: single
title: Lock-free atomic_shared_ptr
date: 2025-04-10 00:00:00 +0800
categories: 学习
tags: C++
---

How to implement a lock-free `atomic_shared_ptr`?

## shared_ptr

我们都知道`shared_ptr`不是线程安全的。准确来说，这句话可以从几个方面来理解：

首先，`shared_ptr`中的引用计数（准确说是control block）这部分是线程安全的，即如果一个`shared_ptr`是最后一个指向某个对象的`shared_ptr`，且这个`shared_ptr`要析构时，引用计数中的`use_count`归零，此时会将对象进行析构。

> 注意此时引用计数自身并不一定析构，因为`weak_count`不一定为0，即可能存在`weak_ptr`还指向它。只有当所有指向这个引用计数的`weak_ptr`都析构时，它才会析构。
>

其次，`shared_ptr`中维护的对象指针那一部分就不是线程安全的了。比如多个线程同时修改`shared_ptr`指向的对象时，如果多个线程间没有进行同步，就会出现竞争。

最后，多个线程同时读写`shared_ptr`指向的对象也不是线程安全的。比如有一个名为`global`的`shared_ptr`在多个线程中使用，线程1要修改`global`指向另外一个对象，而线程2此时正要读取`global`。

```cpp
// Thread 1
global = other_thing;

// Thread 2
auto p = global;
```

线程1实际执行的代码会类似如下，首先构造一个临时`shared_ptr`对象，此时`Right`的引用计数会加一，然后通过一次`swap`交换`*this`和临时对象，最后返回`*this`。

```cpp
shared_ptr& operator=(const shared_ptr& _Right) noexcept {
    shared_ptr(_Right).swap(*this);
    return *this;
}

void swap(_Ptr_base& _Right) noexcept {
    swap(_Ptr, _Right._Ptr);   // 更新对象指针
    swap(_Rep, _Right._Rep);   // 更新引用计数
}
```

问题就出在`swap`中，如果线程1只更新了`p1`中的对象指针，还没来得及更新引用计数就因为context switch等被挂起。之后线程2将`global`指向了另一个对象，此时`global`的引用计数部分归零，触发了`global`原来指向对象的析构。当线程1继续执行时，它并不知道挂起期间发生了析构。当它完成赋值后，后续使用这个野指针时就会出现use-after-free。

## atomic_shared_ptr

C++20提供了`atomic_shared_ptr`，本质上就是`atomic<shared_ptr>`。为了解决`shared_ptr`中无法原子的操作对象指针和引用计数的问题，`atomic_shared_ptr`将对象指针和引用计数都保存到`ControlBlock`中，从而可以原子的修改`ControlBlock`。实际的实现有类似如下的接口：

```cpp
template <typename T>
class atomic_shared_ptr {
public:
    shared_ptr<T> load();
    void store(shared_ptr<T> p);
    // ...
private:
    atomic<ControlBlock*> ctrl;
}

// ControlBlock中有一个atomic<long>作为引用计数和自旋锁
template <typename T>
class ControlBlock {
    T* ptr = nullptr;
    atomic<long> ref_count;
};
```

我们可以看下gcc的实现，在`ControlBlock`的`load`和`store`中都会通过自旋锁的形式，更新引用计数和对象，最后再返回。

```cpp

value_type load(memory_order __o) const noexcept {
    __glibcxx_assert(__o != memory_order_release && __o != memory_order_acq_rel);
    // Ensure that the correct value of _M_ptr is visible after locking.,
    // by upgrading relaxed or consume to acquire.
    if (__o != memory_order_seq_cst) __o = memory_order_acquire;

    value_type __ret;
    auto __pi = _M_refcount.lock(__o);
    __ret._M_ptr = _M_ptr;
    __ret._M_refcount._M_pi = _S_add_ref(__pi);
    _M_refcount.unlock(memory_order_relaxed);
    return __ret;
}

// 会在store中调用
void swap(value_type& __r, memory_order __o) noexcept {
    _M_refcount.lock(memory_order_acquire);
    std::swap(_M_ptr, __r._M_ptr);
    _M_refcount._M_swap_unlock(__r._M_refcount, __o);
}
```

如果不上锁会，仍然会出现use-after-free等问题。其本质问题在于即便可以原子的修改`ControlBlock`，也不能保证同时原子读取`ControlBlock`中的对象指针和引用计数。假设线程1正在`load`，而线程2正在`store`。线程1在`load`时刚读取了对象，在更新引用计数之前，线程2开始执行`store`，如果此时引用计数归零，触发了对象的析构，之后线程1再使用这个对象时就会出现use-after-free。

```cpp
atomic<shared_ptr<T>> a(make_shared<T>(...));

// Thread 1
auto s = a.load();

// Thread 2
a.store(make_shared<T>(...));
```

那么有没有可能实现一个lock free的`atomic_shared_ptr`呢？在回答之前，我们先总结一下目前的场景：在并发场景下，**当一个线程正在读取某个`atomic_shared_ptr`时，另一个线程想要析构这个`atomic_shared_ptr`指向的对象**。换而言之，如果我们在读某个对象的时候，如果能够**原子的获取到`atomic_shared_ptr`中的`ControlBlock`，并且更新引用计数**，那么写线程就不会错误的析构对象了。

### split reference count

一开始我直觉上的第一反应是使用[DCAS](https://en.wikipedia.org/wiki/Double_compare-and-swap)，但仔细一想是不可行的：引用计数是在`ControlBlock`中的，只有当先读取到`ControlBlock`，才能更新引用计数。那既然我们没有办法原子的更新这个全局引用计数，有没有办法在读的时候我们更新一个本地引用计数，然后让全局引用计数感知到本地引用计数呢？这就是通过split reference count来解决这个问题的核心思路：

- `atomic_shared_ptr`中的`ControlBlock`中的引用计数是全局的，用来记录有多少`shared_ptr`正在指向当前这个对象。
- 新增一个本地引用计数，用它来作为全局引用计数的引用计数，即只要读的线程正在使用全局引用计数，我们就不会触发全局引用计数归零。所以本质上它是引入了两层引用计数。

具体做法如下，在`atomic_shared_ptr`中，我们把原始的`ControlBlock`替换为`CountedControlBlock`。在其中增加一个本地引用计数`local_ref_count`，它的核心作用就是**记录有多少个正在执行的`load`操作**。只要本地引用计数不为0，写的线程就不可以把相应对象析构。由于`CountedControlBlock`把`ControlBlock`和本地引用计数打包在了一起，那么我们就可以通过CAS操作完成获取`ControlBlock`并更新(本地)引用计数的需求。

```cpp
struct CountedControlBlock {
    ControlBlock* ctrl;
    long local_ref_count;
};
std::atomic<CountedControlBlock> cctrl;
```

> 上面代码中`CountedControlBlock`是16个字节，我们可以通过DCAS的形式来进行更新，下面还有一种办法可以不依赖于DCAS就能实现更新。
>

`load`时需要依次进行以下操作：

- 通过CAS增加本地引用计数，防止写线程析构正在读取的对象。
- 增加全局引用计数。
- 读取对象。
- 通过CAS减少本地引用计数，表明这次读取已经完成。

注意在`decrement_local_ref_count`中，如果发现`old_ccb.ctrl != prev_ccb.ctrl`，我们需要检查是否在读取的过程中，是否存在其他并发的`store`操作修改了其指向的对象。如果发现`CountedControlBlock`在读取过程中被修改过，那么需要将之前的引用计数减一，这里可以对应后面的`store`操作来理解。

```cpp
shared_ptr<T> load() {
    auto current_ccb = increment_local_ref_count();
    current_ccb.ctrl->increment_ref_count();
    auto result = shared_ptr<T>(current_ccb.ctrl);
    decrement_local_ref_count(current_ccb);
    return result;
}

CountedControlBlock increment_local_ref_count() {
    CountedControlBlock old_ccb = cctrl.load(), new_ccb;
    do {
        new_ccb = old_ccb;
        new_ccb.local_ref_count++;
    } while (!cctrl.compare_exchange(old_ccb, new_ccb));
    return new_ccb;
}

void decrement_local_ref_count(CountedControlBlock prev_ccb) {
    CountedControlBlock old_ccb = prev_ccb, new_ccb;
    do {
        new_ccb = old_ccb;
        new_ccb.local_ref_count--;
    } while (old_ccb.ctrl == prev_ccb.ctrl && !cctrl.compare_exchange(old_ccb, new_ccb));
    if (old_ccb.ctrl != prev_ccb.ctrl) {
        // A store must have moved my local count to the global count
        prev_ccb.ctrl->decrement_ref_count();
    }
}
```

`store`时需要依次做一下操作：

- 构造一个新的`CountedControlBlock`用来保存数据。
- 通过exchange更新自身`CountedControlBlock`。
- 将之前的本地引用计数加到全局引用计数中。
- 更新之前的全局引用计数减一。

注意在`old_ccb.ctrl->add_ref_count(old_ccb.local_ref_count);`这一步中，如果本地引用计数不为0，那么就代表存在并发的读操作。因此通过将本地引用计数增加到全局引用计数中，由于全局引用计数的上升，此时`store`操作一定能保证对象在这些并发读操作完成之前不会发生析构。当然，这样做有一个副作用就是，对象的析构既有可能发生在`store`中也有可能发生在`load`中，取决于谁最后调用`decrement_ref_count`。

```cpp
void store(shared_ptr<T> desired) {
    auto new_ccb = CountedControlBlock{desired.ctrl, 0};
    auto old_ccb = ctrl.exchange(new_ccb);
    // The store operation helps an in-flight load by
    // moving its local_ref_count onto the global ref_count
    old_ccb.ctrl->add_ref_count(old_ccb.local_ref_count);
    old_ccb.ctrl->decrement_ref_count();
    desired.clear();
}
```

到这，我们就完成了一个lock-free的`atomic_shared_ptr`了。

### Folly’s atomic_shared_ptr

当然，这里还遗留了一个问题，我们刚才使用的`CountedControlBlock`大小为16，需要使用DCAS。如果我的CPU不支持这个指令，或者是DCAS的性能太差怎么办。Folly的`atomic_shared_ptr`利用了x86下虚拟地址的高16位都是0的这个特点，因此我们可以只使用`ControlBlock`指针的低48位，而把本地引用计数保存在高16位。通过这样的形式，我们只需要使用普通的CAS即可。

```cpp
struct PackedPtr {
  signed long long ptr : 48;
  unsigned long long local_ref_cnt : 16;
};
```

### Deferred Reclamation

split reference count通过本地引用计数保证了当一个reader正在读取某个对象时，不让并发的writer将其修改。这其实是一个经典的延迟回收问题，有很多标准的解决办法，比如EBR、Hazard Pointers、RCU等。我们可以参考[这里](https://github.com/DanielLiamAnderson/atomic_shared_ptr)的实现，主要改动几种在两个地方

```cpp
[[nodiscard]] shared_ptr_type load() const {
    folly::hazptr_holder hp = folly::make_hazard_pointer();
    control_block_type* current_control_block = nullptr;

    do {
        current_control_block = hp.protect(control_block);
    } while (current_control_block != nullptr &&
             !current_control_block->increment_if_nonzero());

    return shared_ptr<T>(current_control_block);
}

// ...

void store(shared_ptr_type desired) {
    auto new_control_block = std::exchange(desired.control_block, nullptr);
    auto old_control_block = control_block.exchange(new_control_block);
    if (old_control_block) {
        old_control_block->decrement_count();
    }
}

// Release a reference to the object.
void decrement_count() noexcept {
    if (ref_count.fetch_sub(1) == 1) {
        delete ptr;
        this->retire();
    }
}
```

1. 在`load`操作中，首先通过一个HazardPointer来保护引用计数部分，并且需要进行一次CAS操作，即检查当前引用计数是否为0，如果不为0则加一。这里检查是否为0的原因是为了防止并发写线程触发析构，导致引用计数归零。从另一方面理解，可以把`increment_if_nonzero`理解为`weak_ptr`的`lock`操作，它本质上同样是检查相应对象是否存在，如果存在则增加引用计数。
2. 在`store`操作中，当老对象引用计数减一时，我们不能直接释放释放对应的ControlBlock(`load`中可能还在读取它)，而是调用HazardPointer的`retire`接口进行延迟析构。

## Reference

[https://github.com/DanielLiamAnderson/atomic_shared_ptr](https://github.com/DanielLiamAnderson/atomic_shared_ptr)

[https://github.com/facebook/folly/blob/main/folly/concurrency/AtomicSharedPtr.h](https://github.com/facebook/folly/blob/main/folly/concurrency/AtomicSharedPtr.h)