---
layout: single
title: Jemalloc 5.3.0 internals, part 3
date: 2024-08-27 00:00:00 +0800
categories: 学习
tags: Linux
---

上一篇介绍了hpa (huge page allocator)，这一篇继续介绍物理层的另一种page allocator: pac  (page allocator classic)。

### Introduction

 和hpa不同，pac没有shard的概念，数据结构如下所示。pac中有三个缓存：`ecache_dirty`、`ecache_muzzy`和`ecache_retained`。pac会优先尝试在`ecache_dirty`中分配内存，如果失败则尝试在`ecache_muzzy`中分配内存，如果仍然失败，则会在`ecache_retained`中进行分配。当`ecache_retained`都无法进行内存分配时，此时才会通过base进行内存分配。

> 和上一篇一样，如无特殊说明，edata和extent二者可以互换，优先使用代码或者注释中使用的名词。
>

```cpp
struct pac_s {
    pai_t pai;
    ecache_t ecache_dirty;
    ecache_t ecache_muzzy;
    ecache_t ecache_retained;
    base_t *base;
    emap_t *emap;
    edata_cache_t *edata_cache;
    exp_grow_t exp_grow;
    malloc_mutex_t grow_mtx;
    san_bump_alloc_t sba;
    atomic_zu_t oversize_threshold;
    decay_t decay_dirty;
    decay_t decay_muzzy;
    malloc_mutex_t *stats_mtx;
    pac_stats_t *stats;
    atomic_zu_t extent_sn_next;
};
```

- `ecache_dirty` / `ecache_muzzy` / `ecache_retained` 是前面提到的三个缓存
- 和hpa类似，pac中也有base和emap。base用来维护edata链表及其内存使用的元信息，而emap是一个前缀树，方便进行edata的切分和合并。
- `decay_dirty` 负责记录从`ecache_dirty`进行回收的相关信息。
- `decay_muzzy` 负责记录从`ecache_muzzy`进行回收的相关信息。

通过pac进行内存分配的调用路径如下，接下来我们会逐步分解这个过程。

```cpp
pa_alloc
  -> pai_alloc
    -> pac_alloc_impl
      -> pac_alloc_real
        -> ecache_alloc
        -> ecache_alloc_grow
```

我们可以看到在`pac_alloc_real`中调用了两次`ecache_alloc`（最终是调用到`extent_recycle`）。第一次`extent_recycle`会传入`pac->ecache_dirty`，第二次`extent_recycle`会传入`pac->ecache_muzzy`。只有这两次都失败之后，会调用`ecache_alloc_grow`，其内部会第三次调用`extent_recycle`，此时会传入`pac->ecache_retained`。

```cpp
static edata_t *pac_alloc_real(tsdn_t *tsdn,
                               pac_t *pac,
                               ehooks_t *ehooks,
                               size_t size,
                               size_t alignment,
                               bool zero,
                               bool guarded) {
    assert(!guarded || alignment <= PAGE);

    edata_t *edata = ecache_alloc(
            tsdn, pac, ehooks, &pac->ecache_dirty, NULL, size, alignment, zero, guarded);

    if (edata == NULL && pac_may_have_muzzy(pac)) {
        edata = ecache_alloc(
                tsdn, pac, ehooks, &pac->ecache_muzzy, NULL, size, alignment, zero, guarded);
    }
    if (edata == NULL) {
        edata = ecache_alloc_grow(
                tsdn, pac, ehooks, &pac->ecache_retained, NULL, size, alignment, zero, guarded);
        if (config_stats && edata != NULL) {
            atomic_fetch_add_zu(&pac->stats->pac_mapped, size, ATOMIC_RELAXED);
        }
    }

    return edata;
}
```

## 第一次尝试和第二次尝试

根据上面描述的流程，我们首先看看前两次尝试从`ecache_dirty`和`ecache_muzzy`进行分配内存的逻辑：

```cpp
ecache_alloc
  -> extent_recycle
    -> extent_recycle_extract
    -> extent_recycle_split
```

不论`extent_recycle`中传入的是哪个ecache，它都是通过传入的不同ecache中的eset(extent set的缩写)来进行内存分配，其本质就是若干extent的集合。数据结构如下：

```cpp
struct eset_s {
    fb_group_t bitmap[FB_NGROUPS(SC_NPSIZES + 1)];
    eset_bin_t bins[SC_NPSIZES + 1];
    eset_bin_stats_t bin_stats[SC_NPSIZES + 1];
    edata_list_inactive_t lru;
    atomic_zu_t npages;
    extent_state_t state;
};
```

eset内部有若干个edata的堆`bins`，通过一个bitmap用于加速查找。此外还有一个edata链表作为LRU，用于决定哪些edata可以优先回收。具体使用ecache分配内存的逻辑如下：

1. 首先根据要分配内存的大小以及bitmap，确定`bins`堆数组中第一个满足大小要求的bin，将其下标记为`ind`，查找`bins[ind, ind + lg_max_fit)`范围内的这些edata堆。之所以只找这个范围内的edata堆是因为这些edata都满足大小，且不会造成太多的内存碎片。确定范围之后，遍历这些edata堆，挑选地址最小的edata返回。如果这些堆都没有可用的edata，则返回空（这部分逻辑都在`extent_recycle_extract`函数中）。
2. 如果`extent_recycle_extract`分配失败，则会在更大一点的bin中找一个可用的extent进行切分，并且将切分之后剩余的内存保存到ecache中（相关逻辑在`extent_recycle_split`函数中）。

下面我们再具体展开看看内部实现，首先是不需要切分时的逻辑：

```cpp
extent_recycle_extract
	-> emap_try_acquire_edata_neighbor_expand
    -> emap_try_acquire_edata_neighbor_impl
  -> eset_fit
    -> eset_first_fit
    -> eset_fit_alignment
```

- 如果这次内存分配是一次expand（实际是从`pa_expand`调用而来，正常内存分配时不会进入这个路径），则调用`emap_try_acquire_edata_neighbor_expand`，尝试根据前缀树emap中寻找一个edata看是否能扩大一块内存。如果分配成功，则直接返回这个edata。
- 如果这次内存分配不是一次expand，则调用`eset_fit`，尝试从pac中的ecache尝试进行分配。也就是上面提到的遍历bins中的edata堆找到地址最小的edata返回。
- 如果查找范围内的edata堆中没有可用的edata，且这次内存分配指定的对齐比PAGE大时，会调用`eset_fit_alignment`。即根据对齐需要的内存大小以及bitmap来确定bins堆数组中需要检查的范围，看每个堆中的第一个extent是否作为结果返回。

如果以上步骤都没有找到可用的edata，此时代表需要将ecache中更大块的内存进行切分。这一步需要借助前缀树来完成这个过程，相关逻辑在`extent_recycle_split`中：

```cpp
extent_recycle_split
  -> extent_split_interior
    -> extent_split_impl
      -> emap_split_prepare
      -> ehooks_split
      -> emap_split_commit
```

`extent_split_interior`中会根据前缀树对edata进行切分的处理。

## 第三次尝试

如果从dirty/muzzy都分配失败之后，会在`ecache_alloc_grow`中按顺序进行如下尝试

1. 首先对retained调用`extent_recycle`，尝试从`ecache_retained`进行内存分配，这部分逻辑和dirty/muzzy一样，这里不做展开。
2. 如果失败，则调用`extent_grow_retained`。
3. 如果仍然失败，则尝试从base进行内存分配。

```cpp
ecache_alloc_grow
  -> extent_alloc_retained
    -> extent_recycle
    -> extent_grow_retained
  -> extent_alloc_wrapper
```

### extent_grow_retained

主要逻辑如下：

1. `exp_grow_size_prepare`: 根据当前要分配内存的大小，确定从retained cache中能最小能满足的extent大小。每次调用`extent_grow_retained`后，下一次寻找的extent大小都会指数级上升，相关记录保存在pac的`exp_grow`成员变量中。
2. `edata_cache_get`: 从edata cache中获取一个edata。如果从edata cache中获取失败，则尝试从base中尝试获取一个edata。（注意这个edata cache不是上面的`ecache_dirty`、`ecache_muzzy`或者`ecache_retained`的类型ecache，吐槽一下jemalloc命名重合度极高，很容易分不清）
3. `edata_init`: 根据首地址和大小初始化edata。
4. `extent_split_interior`: 此时已经获取了一个 edata，开始对这个edata进行切分。
5. 如果切分成功，将剩余的部分保存到pac的`ecache_retained`中

### 从base中进行分配

如果从`ecache_retained`中进行内存分配都失败，则fallback到使用base进行内存分配。这里是通过一个钩子函数执行的，调用路径如下：

```cpp
extent_alloc_wrapper
  -> ehooks_alloc
    -> ehooks_default_alloc_impl (默认的钩子函数实现, 也可以通过用户指定钩子函数)
      -> ehooks_default_alloc
        -> ehooks_default_alloc_impl
          -> extent_alloc_core
            -> extent_alloc_mmap
              -> pages_map
                -> os_pages_map
                  -> mmap
```

核心函数在`extent_alloc_core`中，这里会根据一些配置，决定最终是调用sbrk还是mmap（上面的调用路径只列出来了mmap）。

```cpp
static void *extent_alloc_core(tsdn_t *tsdn,
                               arena_t *arena,
                               void *new_addr,
                               size_t size,
                               size_t alignment,
                               bool *zero,
                               bool *commit,
                               dss_prec_t dss_prec) {
    void *ret;

    assert(size != 0);
    assert(alignment != 0);

    /* "primary" dss. */
    if (have_dss && dss_prec == dss_prec_primary &&
        (ret = extent_alloc_dss(tsdn, arena, new_addr, size, alignment, zero, commit)) !=
                NULL) {
        return ret;
    }
    /* mmap. */
    if ((ret = extent_alloc_mmap(new_addr, size, alignment, zero, commit)) != NULL) {
        return ret;
    }
    /* "secondary" dss. */
    if (have_dss && dss_prec == dss_prec_secondary &&
        (ret = extent_alloc_dss(tsdn, arena, new_addr, size, alignment, zero, commit)) !=
                NULL) {
        return ret;
    }

    /* All strategies for allocation failed. */
    return NULL;
}
```

可以看到最终通过base就能和操作系统进行交互，进行具体的内存分配。

## 回收

和hpa不同的是，pac因为有三个缓存，这三个缓存之前是需要对其中的edata进行转换和回收，也就是所谓的decay。decay的时机可能有以下几种：

- arena在分配slab还是在释放slab时，都有可能会进行decay
- 手动通过mallctl工具触发decay
- jemalloc有两个参数控制多久回收缓存中空闲的内存：`dirty_decay_ms`和`muzzy_decay_ms`

decay的入口函数如下，其中有几个关键参数：

- `arena`: arena指针
- `ecache`: 从哪里进行decay，实际上是pac中的ecache指针，即`ecache_dirty`或者`ecache_muzzy`
- `decay`: 要decay到哪里去，实际上是pac中的decay cache指针，即`decay_dirty`或者`decay_muzzy`
- `all`: 是否进行全量的decay

```cpp
static bool arena_decay_impl(tsdn_t *tsdn,
                             arena_t *arena,
                             decay_t *decay,
                             pac_decay_stats_t *decay_stats,
                             ecache_t *ecache,
                             bool is_background_thread,
                             bool all) {
    if (all) {
        malloc_mutex_lock(tsdn, &decay->mtx);
        pac_decay_all(
                tsdn, &arena->pa_shard.pac, decay, decay_stats, ecache, /* fully_decay */ all);
        malloc_mutex_unlock(tsdn, &decay->mtx);
        return false;
    }

    if (malloc_mutex_trylock(tsdn, &decay->mtx)) {
        /* No need to wait if another thread is in progress. */
        return true;
    }
    pac_purge_eagerness_t eagerness =
            arena_decide_unforced_purge_eagerness(is_background_thread);
    bool epoch_advanced = pac_maybe_decay_purge(
            tsdn, &arena->pa_shard.pac, decay, decay_stats, ecache, eagerness);
    size_t npages_new;
    if (epoch_advanced) {
        /* Backlog is updated on epoch advance. */
        npages_new = decay_epoch_npages_delta(decay);
    }
    malloc_mutex_unlock(tsdn, &decay->mtx);

    if (have_background_thread && background_thread_enabled() && epoch_advanced &&
        !is_background_thread) {
        arena_maybe_do_deferred_work(tsdn, arena, decay, npages_new);
    }

    return false;
}
```

具体的调用路径如下：

```cpp
arena_decay_impl
  -> pac_decay_all              // full decay
    **->** pac_decay_to_limit
  -> pac_maybe_decay_purge      // not-full decay
    -> pac_decay_try_purge
      -> pac_decay_to_limit
```

最终都是调用到`pac_decay_to_limit`，其主要逻辑就是在不破坏约束的前提下最多将`npages_decay_max`个pages进行decay。这里的约束是`ecache_npages_get(ecache) >= npages_limit`，即ecache中最少要有`npages_limit`个pages。

```cpp
static void pac_decay_to_limit(tsdn_t *tsdn,
                               pac_t *pac,
                               decay_t *decay,
                               pac_decay_stats_t *decay_stats,
                               ecache_t *ecache,
                               bool fully_decay,
                               size_t npages_limit,
                               size_t npages_decay_max) {
    witness_assert_depth_to_rank(tsdn_witness_tsdp_get(tsdn), WITNESS_RANK_CORE, 1);

    if (decay->purging || npages_decay_max == 0) {
        return;
    }
    decay->purging = true;
    malloc_mutex_unlock(tsdn, &decay->mtx);

    edata_list_inactive_t decay_extents;
    edata_list_inactive_init(&decay_extents);
    size_t npurge = pac_stash_decayed(
            tsdn, pac, ecache, npages_limit, npages_decay_max, &decay_extents);
    if (npurge != 0) {
        size_t npurged = pac_decay_stashed(
                tsdn, pac, decay, decay_stats, ecache, fully_decay, &decay_extents);
        assert(npurged == npurge);
    }

    malloc_mutex_lock(tsdn, &decay->mtx);
    decay->purging = false;
}
```

具体的decay操作是在这两个函数中完成的：

1. `pac_stash_decayed`: 负责确定哪些edata可以进行回收，将它们收集到一个链表中。其主要逻辑是不断调用`ecache_evict`，根据ecache中的LRU，找出不活跃的edata，然后将其从eset中删除。`pac_stash_decayed`中还会尝试将这些想要回收的edata和extent中的其他edata进行合并，如果合并成功就一并加入结果链表中。
2. `pac_decay_stashed`: 负责进行实际的回收。本质上是通过钩子函数，最终调用`madvise`进行的具体回收。

    ```cpp
    pac_decay_stashed
      -> extent_purge_lazy_wrapper
        -> extent_purge_lazy_impl
          -> ehooks_purge_lazy
            -> ehooks_default_purge_lazy_impl
              -> pages_purge_lazy
                -> madvise
    ```


## hpa vs pac

我们简单对比一下hpa和pac：

- hpa是有分片的，每次从hpa分配一个edata之后，还需要初始化对应的hpdata，用来保存 edata相关的元信息。而pac没有分片。
- hpa和pac都有多层缓存结构：hpa是edata_cache_fast和edata_cache，pac则是ecache和edata_cache。二者都是只有缓存都没有命中时，才会通过base去进行内存分配。
- hpa是通过psset管理edata（内部是若干hpdata的堆），pac则是通过eset管理edata。
- 二者都是通过emap这个前缀树进行edata的分配和合并的管理。

|               | hpa                               | pac                     |
| ------------- | --------------------------------- | ----------------------- |
| 是否有分片    | 有                                | 没有                    |
| 缓存结构      | edata_cache_fast_s和edata_cache_s | ecache_s和edata_cache_s |
| edata set     | psset_s                           | eset_s                  |
| edata合并分裂 | emap_s                            | emap_s                  |
| 是否有decay   | 没有                              | 有                      |

这里详细分析一下psset和eset。首先二者在hpa和pac中的作用是一样的，都通过一个bitmap维护了内存的使用状态，并进行内存的分配和释放。区别在于psset不直接控制edata，而是维护hpdata。而eset则直接控制edata。

## 写在最后

到这里jemalloc大体介绍就完了，第一篇我们介绍了逻辑层，第二三篇我们分别介绍了物理层的两种实现hpa和pac。这几篇拖的时间特别长，中途一路也想弃坑。主要原因是jemalloc相关文档和注释很少，需要不断翻代码，最终只会陷入到各种细节中，也就导致很难从一个全局视角自顶向下的描述清楚。另外5.3.0版本的高质量资料一方面很少，加上这个新版本进行了大量重构和概念上的重组，导致理解起来异常困难，其中一些概念和命名及其容易混淆，甚至写完之后也有一些不是那么清晰的地方。后面有机会，会从另外一个视角更简明扼要的把整体思想描述清楚。