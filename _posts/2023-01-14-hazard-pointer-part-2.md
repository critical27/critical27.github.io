---
layout: single
title: Hazard pointer, part II
date: 2023-01-14 00:00:00 +0800
categories: 论文 实践
tags: C++ Concurrency
---

能否有足够工程能力将理论算法变现？

我们在[上一篇](https://critical27.github.io/c++/concurrency/hazard-pointer-part-1/)介绍了hazard pointers，让我们试试在实践中能否解决所有的并发问题。我可以保证，多数博客上比较简单的实现，都有内存泄露问题……

### hazard pointer

为了简单，我们将`HazptrList`实现为一个单例，retirelist作为thread_local变量保存后续需要回收的内存。同时在内存分配和释放的位置适当的增加了一些日志，其余部分和上一篇中的实现几乎完全一致，不做过多介绍。

需要注意的是，在原始论文中，没有将全局链表的内存释放，肯定是无法通过sanitizer检测的，所以我们在`HazptrList`析构函数中，将链表的内存也进行释放。

```cpp
#define CAS(val, oldval, newval) __sync_bool_compare_and_swap((val), (oldval), (newval))
#define MEMORY_FENCE() asm volatile("" ::: "memory")

using MapType = std::map<std::string, std::string>;
using MapPointer = std::map<std::string, std::string>*;

struct HazptrNode {
  ~HazptrNode() {
    VLOG(2) << "[Map] del map " << pHazard_;
    delete pHazard_;
  }

  std::atomic<MapPointer> pHazard_{nullptr};
  HazptrNode* next_{nullptr};
  bool active_{false};
};

class HazptrList {
 private:
  HazptrNode* head_{nullptr};

  ~HazptrList() {
    LOG(INFO) << "HazptrList dtor";
    auto cur = head_;
    while (cur != nullptr) {
      auto next = cur->next_;
      VLOG(2) << "[Node] del node " << cur;
      delete cur;
      cur = next;
    }
  }

 public:
  static HazptrList& instance() {
    static HazptrList instance;
    return instance;
  }

  HazptrNode* head() { return head_; }

  HazptrNode* acquire() {
    auto p = head_;
    while (p != nullptr) {
      if (p->active_ || !CAS(&(p->active_), false, true)) {
        p = p->next_;
        continue;
      }
      return p;
    }

    HazptrNode* node = new HazptrNode();
    VLOG(2) << "[Node] new node " << node;
    node->active_ = true;

    HazptrNode* oldHead;
    do {
      oldHead = head_;
      node->next_ = oldHead;
    } while (!CAS(&head_, oldHead, node));
    return node;
  }

  void release(HazptrNode* node) {
    (node->pHazard_).store(nullptr, std::memory_order_release);
    node->active_ = false;
  }
};

thread_local std::vector<MapPointer> retireList;
```

### LockFreeMap

`LockFreeMap`就是上文所说的`WRRMMap`，提供线程安全的读取和写入接口。主要修改了两个地方：

1. 当更新操作完成时候，会调用`retire`将旧指针放入`retirelist`中，当`retirelist`大小达到一定大小，开始尝试进行内存回收。为了后面调试方便，我们把大小修改为1。
2. 在回收内存的函数`scanAndRetire`中，原始论文使用两个vector来分别保存已经退役的指针和正在使用的指针，并对这两个vector求差集。这里为了简单明了，直接使用unorder_set来确定是否存在特定指针。

```cpp
class LockFreeMap {
 public:
  LockFreeMap() {
    pMap_ = new MapType();
    VLOG(2) << "[Map] new map " << pMap_;
  }

  std::string lookup(const std::string& key) {
    auto hazptr = HazptrList::instance().acquire();
    MapPointer pRead;
    do {
      pRead = pMap_;
      (hazptr->pHazard_).store(pRead, std::memory_order_release);
    } while (pMap_ != pRead);
    VLOG(1) << "lookup acquire " << pRead;
    auto value = (*pRead)[key];
    CHECK_EQ(hazptr->pHazard_, pRead);
    HazptrList::instance().release(hazptr);
    VLOG(1) << "lookup release " << pRead;
    return value;
  }

  void update(const std::string& key, const std::string& val) {
    MapPointer pNew = nullptr;
    MapPointer pOld = nullptr;
    do {
      pOld = pMap_;
      if (pNew != nullptr) {
        VLOG(2) << "[Map] del map " << pNew;
      }
      delete pNew;
      pNew = new MapType(*pMap_);
      VLOG(2) << "[Map] new map " << pNew;
      (*pNew)[key] = val;
    } while (!CAS(&pMap_, pOld, pNew));
    VLOG(1) << "update cas from " << pOld << " to " << pNew;
    retire(pOld);
  }

  void retire(MapPointer pOld) {
    retireList.emplace_back(pOld);
    if (retireList.size() > 1) {
      scanAndRetire();
    }
  }

  void scanAndRetire() {
    std::unordered_set<MapPointer> hazardPointers;
    auto cur = (HazptrList::instance()).head();
    while (cur != nullptr) {
      auto pHazard = (cur->pHazard_).load(std::memory_order_acquire);
      if (pHazard != nullptr) {
        hazardPointers.emplace(pHazard);
      }
      cur = cur->next_;
    }

    auto iter = retireList.begin();
    while (iter != retireList.end()) {
      auto pRetire = *iter;
      if (!hazardPointers.count(pRetire)) {
        VLOG(2) << "[Map] del map " << pRetire;
        delete pRetire;
        if (*iter != retireList.back()) {
          *iter = retireList.back();
        }
        retireList.pop_back();
      } else {
        iter++;
      }
    }
  }

 private:
  MapPointer pMap_{nullptr};
};
```

### 测试代码

然后是测试代码，我们分别启动10个线程，每个线程随机对`LockFreeMap`进行10次读写后退出，主线程则等待所有其他线程退出。

```cpp
TEST(HazptrTest, HazptrTest) {
  std::vector<std::thread> threads;
  int count = 2;
  LockFreeMap map;
  for (int i = 0; i < count; i++) {
    threads.emplace_back([&map, i] {
      for (int j = 0; j < 10; j++) {
        if (folly::Random::rand32(0, 100) < 50) {
          map.update("key", folly::stringPrintf("thread_%d_val_%d", i, j));
        } else {
          map.lookup("key");
        }
      }
    });
  }
  LOG(INFO) << "Wait all threads";
  for (auto& t : threads) {
    t.join();
  }
  LOG(INFO) << "All threads done";
}
```

如果你开启了sanitizer，就会看到类似下面的日志

```cpp
==2710631==ERROR: LeakSanitizer: detected memory leaks

// 省略若干
...

Indirect leak of 144 byte(s) in 3 object(s) allocated from:
    #0 0x13426ed in operator new(unsigned long) (build/bin/test/kv_test+0x13426ed)
    #1 0x135b637 in nebula::storage::LockFreeMap::update(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&, std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> > const&) src/storage/test/KVTest.cpp:128:14
    #2 0x1354f91 in nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0::operator()() const src/storage/test/KVTest.cpp:182:15
    #3 0x1354f91 in void std::__invoke_impl<void, nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0>(std::__invoke_other, nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0&&) /usr/bin/../lib/gcc/x86_64-linux-gnu/9/../../../../include/c++/9/bits/invoke.h:60:14
    #4 0x1354f91 in std::__invoke_result<nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0>::type std::__invoke<nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0>(nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0&&) /usr/bin/../lib/gcc/x86_64-linux-gnu/9/../../../../include/c++/9/bits/invoke.h:95:14
    #5 0x1354f91 in void std::thread::_Invoker<std::tuple<nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0> >::_M_invoke<0ul>(std::_Index_tuple<0ul>) /usr/bin/../lib/gcc/x86_64-linux-gnu/9/../../../../include/c++/9/thread:244:13
    #6 0x1354f91 in std::thread::_Invoker<std::tuple<nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0> >::operator()() /usr/bin/../lib/gcc/x86_64-linux-gnu/9/../../../../include/c++/9/thread:251:11
    #7 0x1354f91 in std::thread::_State_impl<std::thread::_Invoker<std::tuple<nebula::storage::HazptrTest_HazptrTest_Test::TestBody()::$_0> > >::_M_run() /usr/bin/../lib/gcc/x86_64-linux-gnu/9/../../../../include/c++/9/thread:195:13
    #8 0x5564a03 in execute_native_thread_routine (build/bin/test/kv_test+0x5564a03)
    #9 0x7f72eded1608 in start_thread /build/glibc-SzIz7B/glibc-2.31/nptl/pthread_create.c:477:8
    #10 0x7f72eddec132 in clone /build/glibc-SzIz7B/glibc-2.31/misc/../sysdeps/unix/sysv/linux/x86_64/clone.S:95

SUMMARY: AddressSanitizer: 432 byte(s) leaked in 6 allocation(s).
```

可以看到我们在`update`函数中申请的内存发生了泄露，为什么呢？我这里把线程数修改为2，我们分析下内存泄露的原因。下面是我调整日志级别并适当调整缩进的输出，第一列为线程号。2710705是主线程。2710706和2710707是两个读写线程，分别对应日志中左侧和右侧。

这里分别解释下几个日志输出：

1. `new node addr`和`del map addr`分别对应申请和释放`HazptrNode`内存
2. `new map addr`和`del map addr`分别对应申请和释放map内存，也是我们发生内存泄露的地方
3. `cas from addr1 to addr2`代表当前线程通过cas操作更新了数据

可以发现new map总共出现了21次，而del map总共出现了22次，去掉最后两次del空指针，只有20次。也就是说我们有一个map的内存的确没有释放。仔细检查以后我们可以发现是`0x604000020290`这个map没有释放，为什么？

> 通过看这个日志还可以发现，假设出现在线程1出现日志`cas from addr1 to addr2`，那么`addr1`只会在这个线程去释放，这也符合hazard pointer的算法，感兴趣的可以思考一下。通过分析内存申请和释放的时机，能够加深对hazard pointer的理解。
>

```cpp
2710705 new map 0x60400001dbd0
2710705 Wait all threads
------------------------------------

2710707                                             new node 0x60300001d920
2710706 new node 0x60300001c1b0
2710707                                             lookup acquire 0x60400001dbd0
2710706 lookup acquire 0x60400001dbd0
2710707                                             lookup release 0x60400001dbd0
2710706 lookup release 0x60400001dbd0
2710707                                             lookup acquire 0x60400001dbd0
2710707                                             lookup release 0x60400001dbd0
2710706 new map 0x60400001e150
2710706 cas from 0x60400001dbd0 to 0x60400001e150
2710707                                             new map 0x604000020010
2710706 new map 0x60400001e190
2710707                                             del map 0x604000020010
2710706 cas from 0x60400001e150 to 0x60400001e190
2710706 del map 0x60400001dbd0
2710707                                             new map 0x604000020050
2710707                                             del map 0x604000020050
2710706 del map 0x60400001e150
2710707                                             new map 0x604000020090
2710707                                             cas from 0x60400001e190 to 0x604000020090
2710706 new map 0x60400001e1d0
2710706 del map 0x60400001e1d0
2710707                                             new map 0x6040000200d0
2710706 new map 0x60400001e210
2710707                                             cas from 0x604000020090 to 0x6040000200d0
2710706 del map 0x60400001e210
2710707                                             del map 0x60400001e190
2710706 new map 0x60400001e250
2710707                                             del map 0x604000020090
2710706 cas from 0x6040000200d0 to 0x60400001e250
2710706 new map 0x60400001e290
2710707                                             new map 0x604000020110
2710706 cas from 0x60400001e250 to 0x60400001e290
2710707                                             del map 0x604000020110
2710706 del map 0x6040000200d0
2710706 del map 0x60400001e250
2710707                                             new map 0x604000020150
2710707                                             cas from 0x60400001e290 to 0x604000020150
2710706 lookup acquire 0x604000020150
2710707                                             lookup acquire 0x604000020150
2710706 lookup release 0x604000020150
2710707                                             lookup release 0x604000020150
2710706 lookup acquire 0x604000020150
2710707                                             lookup acquire 0x604000020150
2710706 lookup release 0x604000020150
2710707                                             lookup release 0x604000020150
2710706 lookup acquire 0x604000020150
2710706 lookup release 0x604000020150
2710707                                             new map 0x604000020190
2710707                                             cas from 0x604000020150 to 0x604000020190
2710706 new map 0x60400001e2d0
2710707                                             del map 0x60400001e290
2710706 del map 0x60400001e2d0
2710707                                             del map 0x604000020150
2710706 new map 0x60400001e310
2710706 cas from 0x604000020190 to 0x60400001e310
2710707                                             new map 0x6040000201d0
2710707                                             del map 0x6040000201d0
2710706 new map 0x60400001e350
2710706 cas from 0x60400001e310 to 0x60400001e350
2710706 del map 0x604000020190
2710707                                             new map 0x604000020210
2710706 del map 0x60400001e310
2710707                                             del map 0x604000020210
2710707                                             new map 0x604000020250
2710707                                             cas from 0x60400001e350 to 0x604000020250
2710707                                             new map 0x604000020290
2710707                                             cas from 0x604000020250 to 0x604000020290
2710707                                             del map 0x60400001e350
2710707                                             del map 0x604000020250

------------------------------------
2710705 All threads done
2710705 HazptrList dtor
2710705 del node 0x60300001c1b0
2710705 del map 0
2710705 del node 0x60300001d920
2710705 del map 0
```

可以发现这个内存地址是在接近结束时，由第二个线程分配的，此后就再也没有释放。那么问题当然就找到了，也就是`LockFreeMap`中最新的一个版本没有释放。

![figure]({{'/archive/hazard-pointers-part-2.png' | prepend: site.baseurl}})

我们该怎么修改才能防止类似的内存泄漏呢？考虑以下几种解决办法

1. `LockFreeMap`中析构当前的`pMap_`

    > 乍一看在这个例子中可以解决问题，但是问题的本质在于对于所有线程都有一个`retirelist`，thread_local的变量在线程退出时就会析构，但此时`retirelist`中可能还有若干指针，它们指向一些过期版本的对象。正是这些对象的内存没有释放导致内存泄漏。
    >
2. 在1的基础上，线程退出时释放`retirelist`中每个指针所指向的内存地址

    > 如果想要在线程退出时，在析构`retirelist`时做一些特殊处理，那我们不能简单的使用vector了，而需要在一个类的析构中进行内存释放。
    >

    我们把`retirelist`改造成一个类，并进行相应的修改再试一下。

    ```cpp
    struct RetireList {
      ~RetireList() {
        for (auto ptr : ptrs) {
          VLOG(2) << "[Map] del map " << ptr;
          delete ptr;
        }
      }

      std::vector<MapPointer> ptrs;
    };

    thread_local RetireList retireList;
    ```

    在我的环境运行了几次，并没有再报内存泄露，但这样真的没问题吗？

    考虑以下场景，线程A在退出之前想要将自身`retireList`中的p1释放，但与此同时，并没有保证其他线程没有在读p1。也就是说线程A在退出时破坏了hazard pointer的约束条件：当有线程正在读pMap_指向的对象时，可以修改pMap_指向其他对象，但不能修改当前对象，或者是释放其内存。

3. 所以，要解决2的问题，我们必须保证所有线程在进行完读写之前不能析构。可以使用任何的同步方法来同步这些线程，这里就不再展示具体代码。
4. 此外，还有一种办法。folly的`ThreadLocal`还提供了额外的接口，可以让我们在`LockFreeMap`的析构解决同样的问题。我们将`retireList`变为`LockFreeMap`的一个成员变量，也就保证了二者生命周期相同。在析构时，通过`accessAllThreads`可以通过内部获取锁的方式，获得所有线程的`std::vector<MapPointer>`，也就是过期指针。相关改动如下，其余部分省略。

    ```cpp
    class LockFreeMap {
    	// ...

      ~LockFreeMap() {
        VLOG(2) << "[Map] del map " << pMap_;
        delete pMap_;
        auto accessor = retireList.accessAllThreads();
        for (const auto& vec : accessor) {
          for (const auto* retired : vec) {
            VLOG(2) << "[Map] del map " << retired;
            delete retired;
          }
        }
      }

      // ...

    private:
      MapPointer pMap_{nullptr};
      folly::ThreadLocal<std::vector<MapPointer>, MapPointer> retireList;
    };
    ```

### 写在最后

所以，即便在我们了解hazard pointer原理之后，在多线程并发环境下，还是会遇到各种各样的问题。想要实现一个工业级别的hazard pointer或者类似内存回收机制，并不是一件容易的事。首先，你需要对原理有足够的透彻的理解。其次，实现的代码是否违背了算法的约束条件。最后，工程层面是否做的尽善尽美。说到这里，我突然意识到，貌似可以用TLA+来验证hazard pointer，有空时候可以试一试。
