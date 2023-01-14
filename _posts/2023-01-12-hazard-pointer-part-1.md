---
layout: single
title: Hazard pointer, part I
date: 2023-01-13 00:00:00 +0800
categories: 论文
tags: C++ Concurrency
---

> 断更太久，实在是最近太忙... 焦头烂额ing

## 核心思想

这篇文章会介绍hazard pointer，和EBR、RCU一样，也是一种安全内存回收机制(SMR，safe memory reclamation)。它可以让读线程就将正在使用的对象内存地址告知其他线程，并由写线程在合适的实际回收相应内存。其核心思想是：当一个读线程将一个对象的地址赋给它的hazard pointer时，即意味着它在向其它线程宣称：“我正在读该指针所指向的对象，如果你愿意，你可以将指针指向其他对象，但别修改我正在读的对象，当然更不要去delete这个指针。”

我们将结合实际使用来看hazard pointer的原理，假设有个读多写少的`WRRMMap(Write-Rarely-Read-Many Map)`，它内部有一个指向某个经典的单线程map对象（比如std::map）的指针，同时它还向外界提供了线程安全的读写接口。

> 不用太纠结于具体实现，下面会逐步解释为什么
>

```cpp
template <class K, class V>
class WRRMMap {
 public:
  map<K, V>* pMap_;

  void Update(const K& k, const V& v);

  V Lookup(const K& k);
};
```

在用hazard pointer时，每当`WRRMMap`需要更新的时候，想要对它进行更新的线程就会首先复制一个`pMap_`所指向的对象，在新版本上做修改，然而再将`pMap_`指向新版本，最后在合适的时机delete掉`pMap_`原先所指的旧版本。这里的copy-on-write有一定性能开销，但因为`WRRMMap`通常是读多写少，所以是可以接受的。然而，麻烦的问题是我们怎样才能妥善地将就副本销毁掉，因为任何时候都可能有其它线程正在对它们进行读取。

## 数据结构

```cpp
// Hazard pointer record
class HPRecType {
  HPRecType* pNext_;
  int active_;
  // Global header of the HP list
  static HPRecType* pHead_;
  // The length of the list
  static int listLen_;

 public:
  // Can be used by the thread that acquired it
  void* pHazard_;

  static HPRecType* Head() {
    return pHead_;
  }

  static HPRecType* Acquire();

  static void Release(HPRecType* p);
};

// Per-thread private variable
thread_local vector<Map<K, V>*> rlist;
```

要实现上面的功能，hazard pointer采取的也是延迟释放内存的方式。`pHead_`是一个全局链表，用来保存正在使用或者曾经被使用过的map指针，整个链表通过CAS来进行修改，`pNext_`指向链表的下一个节点。`pHazard_`就是所谓的hazard pointer，它用来给读线程保存当前读取对象的指针，如果`pHazard_`不为空，其他线程就无法回收`pHazard_`所指向对象占用的内存。此外每个线程都拥有一个退役链表`rlist`（在我们的实现中，这实际上就是一个`vector<Map<K,V>*>`），退役链表`rlist`负责存放该线程不再需要的，一旦其它线程也不再使用它们就可以被delete掉的指针。`rlist`的访问不需要被同步，它是个thread_local变量。

## 实现细节

### 读流程

```cpp
V Lookup(const K& k) {
  HPRecType* pRec = HPRecType::Acquire();
  Map<K, V>* ptr;
  do {
    ptr = pMap_;
    pRec->pHazard_ = ptr;
  } while (pMap_ != ptr);

  // Save Willy
  V result = (*ptr)[k];
  // pRec can be released now, because it's not used anymore
  HPRecType::Release(pRec);
  return result;
}
```

1. 每当需要从`pMap_`中读取数据时，先要调用`HPRecType::Acquire`来获取一个`HPRecType`并将其加入到全局链表中，此时我们获取的`pRec`就是当前线程所独自占用的。
2. 接下来在while循环中，读线程不断尝试将`HPRecType`的`pHazard_`置为`ptr`，通过这一步来宣布当前`ptr`指向的对象有读线程正在使用。这里会通过不断检查`pMap_`和`ptr`是否相等，保证这几步操作过程中， `ptr`始终和`pMap_`相等。

    > 后面还会详细解释这个while循环的作用
    >
3. 之后就可以对`ptr`所指向对象进行读取了。(注意不是使用`pMap_`，因为`pMap_`可能又被修改过，从这个角度看`ptr`获取的是`pMap_`的一个快照版本，而不一定是最新版本)
4. 使用完之后，需要调用`HPRecType::Release`。这一步会在全局链表中将之前通过`Acquire`申请的节点标记为过期，此后`ptr`指向的内存就可以被回收。

我们可以具体看下`Acquire`和`Release`的实现：

```cpp
class HPRecType {
  // ...

  // Acquires one hazard pointer
  static HPRecType* Acquire() {
    // Try to reuse a retired HP record
    HPRecType* p = pHead_;
    for (; p; p = p->pNext_) {
      if (p->active_ || !CAS(&p->active_, 0, 1)) {
        continue;
      }
      // Got one!
      return p;
    }

    // Increment the list length
    int oldLen;
    do {
      oldLen = listLen_;
    } while (!CAS(&listLen_, oldLen, oldLen + 1));

    // Allocate a new one
    HPRecType* p = new HPRecType;
    p->active_ = 1;
    p->pHazard_ = 0;

    // Push it to the front
    do {
      old = pHead_;
      p->pNext_ = old;
    } while (!CAS(&pHead_, old, p));

    return p;
  }

  // Releases a hazard pointer
  static void Release(HPRecType* p) {
    p->pHazard_ = 0;
    p->active_ = 0;
  }

  // ...
}
```

`Acquire`中就是在全局链表中寻找一个可用节点，如果没找到就新增一个节点。然后将节点标记为active，并将这个节点返回。`Release`就是将之前`Acquire`获取的节点清空，并标记为inactive。

### 写流程

```cpp
void Update(const K& k, const V& v) {
  Map<K, V>* pNew = 0;
  do {
    Map<K, V>* pOld = pMap_;
    delete pNew;
    pNew = new Map<K, V>(*pOld);
    (*pNew)[k] = v;
  } while (!CAS(&pMap_, pOld, pNew));
  Retire(pOld);
}
```

写流程是典型copy-on-write，`pNew = new Map<K, V>(*pOld)`，然后在新版本上进行修改，最后尝试将新版本替换进`pMap_`。在更新成功之后，就可以调用`Retire`将旧版本退役，表示如果其他线程也不再使用这个过期副本之后，就可以回收对应内存。

`Retire`逻辑如下，每当一个写线程替换`pMap_`时，它会将替换下来的旧map指针放入一个thread_local链表中。直到该链表中的旧map达到一定数量的时候，写线程就会扫描读线程的hazard pointer，看看旧map列表中有哪些是与这些hazard pointer匹配的。匹配就代表仍有读线程在使用，不能回收对应内存，写线程就会仍将其保留在链表中，直到下次扫描。如果某个旧map不与任何读线程的hazard pointer匹配，那么就将其内存回收。这里的终点在于写线程在delete任何被替换下来的旧map之前须得检查其他读线程的hazard pointer所指向的map是否仍在被使用。任何一个hazard pointer指向的对象，都需要读线程显式告知不再使用这些对象之后（即上面读流程所说的`Release`接口），才可能会被回收内存。

```cpp
template <class K, class V>
class WRRMMap {
  Map<K, V>* pMap_;

 private:
  static void Retire(Map<K, V>* pOld) {
    // put it in the retired list
    rlist.push_back(pOld);
    if (rlist.size() >= R) {
      Scan(HPRecType::Head());
    }
  }

  void Scan(HPRecType* head) {
    // Stage 1: Scan hazard pointers list collecting all non-null ptrs
    vector<void*> hp;
    while (head) {
      void* p = head->pHazard_;
      if (p) {
        hp.push_back(p);
      }
      head = head->pNext_;
    }

    // Stage 2: sort the hazard pointers
    sort(hp.begin(), hp.end(), less<void*>());

    // Stage 3: Search for'em!
    vector<Map<K, V>*>::iterator i = rlist.begin();
    while (i != rlist.end()) {
      if (!binary_search(hp.begin(), hp.end(), *i) {
        // Aha!
        delete *i;
        if (&*i != &rlist.back()) {
          *i = rlist.back();
        }
        rlist.pop_back();
      } else {
        ++i;
      }
    }
  }
};
```

回收内存的`Scan`函数会进行一个差集计算的算法，即当前线程的退役链表`rlist`作为一个集合，其它所有线程的hazard pointer作为一个集合，求它们两个的差集。那么，这个差集的意义又是什么呢？前一个集合是当前线程所认为没有用的旧map指针，而后一个集合则代表那些正被其他线程使用着的旧map指针（即那些线程的hazard pointer所指的map），这些指针在读线程使用完之后，就会将指针进行退役。如果一个指针退役了并且没有被任何一个线程的hazard pointer所指向（也就是使用）的话，该指针所指的旧map就是可以被安全回收的的。换言之，第一个集合与第二个集合的差集正是那些可被安全delete的旧map的指针。

在进行`Scan`时，我们需要对`rlist`跟`pHead`这个hazard pointer链表所代表的两个集合作差集运算。换而言之：“对于`rlist`中的每个指针，检查它是否同位于hazard pointer链表中，如果不是，则它属于差集，即可被安全delete。”此外，为了对该算法加以优化，我们可以在行查找之前先对hazard指针链表进行排序，然后对每个退役指针在其中进行二分查找。

> 其中运用了一点小小的技巧来避免`rlist`这个vector的重排：在delete了`rlist`中的一个可被删除的指针之后，我们将那个指针的位置用rlist的最后一个元素来填充，然后再将实际的最后一个元素pop掉。这样做是允许的，因为rlist中的元素并不要求有序。
>

### “细枝末节”

```cpp
V Lookup(const K&k){
   HPRecType * pRec = HPRecType::Acquire();
   Map<K, V> * ptr;
   do {
      ptr = pMap_;
      pRec->pHazard_ = ptr;
   } while (pMap_ != ptr);

   // Save Willy
   V result = (*ptr)[k];
   // pRec can be released now, because it's not used anymore
   HPRecType::Release(pRec);
   return result;
}
```

在上面的代码中，为什么读线程需要重新检查`pMap_`是否等于`ptr`呢？考虑下面这种情况：`pMap_`原先指向一个map `m`，读线程将`ptr`赋为`pMap_`，也就是`&m`，接着读线程就进入了休眠（此时它还未来得及将hazard pointer指向`m`）。而就在它的休眠期间，一个写线程更新了`pMap_`，然后将`pMap_`原先所指的map，即`m`放到了退役链表中，并紧接着检查各线程的hazard pointer，由于刚才的读线程在进入休眠时还未来得及设置hazard pointer，故`m`被认为是可以销毁的旧map，于是被写线程销毁。而这时读线程被唤醒，将Hazard pointer赋为`ptr`(即`&m`)。如果是这样的话，一旦读线程接下来对`ptr`进行解引用，它就会读取到被破坏的数据或访问了未被映射的内存。如果结果发现`pMap_`并不等于`&m`的话，读线程就无法确定将`m`退役掉的写线程是否看到了它的`hazard`指针被置为`&m`（换句话说，是否已将`m`销毁）。所以此时读线程继续对`m`进行读取就是不安全的，所以读线程需要从头来过。

如果读线程而后发现`pMap_`的确指向`m`，那么此时从`m`读取就是安全的。那么，这是否意味着在两次读取`pMap_`之间，`pMap_`没有被改动过呢？不一定，在这期间`pMap_`可能被修改指向`n`，然后又指回`m`，但这并不要紧。要紧的是在第二次读取的时候`pMap_`指向`m`，而此时对应的hazard pointer `pHazard_`也已经指向它了。

```cpp
void Update(const K&k, const V&v) {
   Map<K, V> * pNew = 0;
   do {
      Map<K, V> * pOld = pMap_;
      delete pNew;
      pNew = new Map<K, V>(*pOld);
      (*pNew)[k] = v;
   } while (!CAS(&pMap_, pOld, pNew));
   Retire(pOld);
}
```

同样，可能有多个线程会进行更新，所以更新`pMap_`需要使用CAS操作。如果CAS失败，需要将上一次更新的副本删除。

## 写在最后

但是，真的有这么简单吗？按照原始论文尝试去实现了一版hazard pointer，有不少内存泄露。想要通过sanitizer，还需要大量的改动，我们下一篇再说。

## Reference

[Lock-Free Data Structures with Hazard Pointers](https://erdani.org/publications/cuj-2004-12.pdf)
