---
layout: single
title: Jemalloc 5.3.0 internals, part 2
date: 2024-08-27 00:00:00 +0800
categories: 学习
tags: Linux
---

上一篇我们介绍了Jemalloc的逻辑层，这一篇我们把目光投向物理层，首先看page allocator中的一种hpa，也就是huge page allocator。

## Introduction

上一篇我们提到过，arena在`arena_slab_alloc`函数中会调用`pa_alloc`获取一个edata，最终将其赋值给arena对应bin中的`slabcur`。后续arena中相同大小的内存分配，都会优先在这个slab中进行。

`pa_alloc`中的`pa`就是page allocator的缩写，在Jemalloc中，所有的数据都是通过page allocator向内核申请内存获取的，它作为物理层负责实际管理操作系统的内存page。它和逻辑层的接口就是`pa_alloc`，每次返回一个edata，也就是extent。

> 如无特殊说明，edata和extent二者可以互换，优先使用代码或者注释中使用的名词。
>

page allocator会分成多个shard，不同shard之间的内存不相交，arena和shard是多对多的关系。arena通过`arena_s`结构体中的`pa_shard`这个成员变量获取到arena对应的page allocator shard。page allocator是一个接口，目前有两种实现：

- pac (page allocator classic)
- hpa (huge page allocator)

我们可以看到`pa_alloc`中的实现，会优先检查能否使用hpa进行分配，如果分配失败，则回退到pac进行分配：

```cpp
edata_t *pa_alloc(tsdn_t *tsdn,
                  pa_shard_t *shard,
                  size_t size,
                  size_t alignment,
                  bool slab,
                  szind_t szind,
                  bool zero,
                  bool guarded,
                  bool *deferred_work_generated) {
    witness_assert_depth_to_rank(tsdn_witness_tsdp_get(tsdn), WITNESS_RANK_CORE, 0);
    assert(!guarded || alignment <= PAGE);

    edata_t *edata = NULL;
    if (!guarded && pa_shard_uses_hpa(shard)) {
        edata = pai_alloc(tsdn,
                          &shard->hpa_sec.pai,
                          size,
                          alignment,
                          zero,
                          /* guarded */ false,
                          slab,
                          deferred_work_generated);
    }
    /*
     * Fall back to the PAC if the HPA is off or couldn't serve the given
     * allocation request.
     */
    if (edata == NULL) {
        edata = pai_alloc(tsdn,
                          &shard->pac.pai,
                          size,
                          alignment,
                          zero,
                          guarded,
                          slab,
                          deferred_work_generated);
    }
    if (edata != NULL) {
        assert(edata_size_get(edata) == size);
        pa_nactive_add(shard, size >> LG_PAGE);
        emap_remap(tsdn, shard->emap, edata, szind, slab);
        edata_szind_set(edata, szind);
        edata_slab_set(edata, slab);
        if (slab && (size > 2 * PAGE)) {
            emap_register_interior(tsdn, shard->emap, edata, szind);
        }
        assert(edata_arena_ind_get(edata) == shard->ind);
    }
    return edata;
}
```

不论是small class, large class还是更大块的内存，都会调用`pa_alloc`，所以它的参数看起来会比较复杂。简单回顾下调用`pa_alloc`的不同情况：

- small class

```cpp
	edata_t *slab = pa_alloc(tsdn, &arena->pa_shard, bin_info->slab_size,
	    /* alignment */ PAGE, /* slab */ true, /* szind */ binind,
	     /* zero */ false, guarded, &deferred_work_generated);
```

- large class以及更大内存

```cpp
	edata_t *edata = pa_alloc(tsdn, &arena->pa_shard, esize, alignment,
	    /* slab */ false, szind, zero, guarded, &deferred_work_generated);
```

`pa_alloc`中关键参数：

- `tsdn`: thread specific data。
- `pa_shard_t`: arena对应的page allocator shard。
- `size`: 要分配多大内存。small class就一次分配slab的大小，large class和更大对象则根据要求malloc的大小，并且为了内存对齐，可能会额外增加一些padding。
- `alignment`: 内存对齐。smal class会按PAGE对齐，large class及更大则按CHELINE对齐。
- `slab`: 是否是slab，只有small class会设置为true。
- `szind`: 分配大小在size class表中的下标。

page allocator shard数据结构如下：

```cpp
struct pa_shard_s {
    pa_central_t *central;
    atomic_zu_t nactive;
    atomic_b_t use_hpa;
    bool ever_used_hpa;
    pac_t pac;
    sec_t hpa_sec;
    hpa_shard_t hpa_shard;
    edata_cache_t edata_cache;
    unsigned ind;
    malloc_mutex_t *stats_mtx;
    pa_shard_stats_t *stats;
    emap_t *emap;
    base_t *base;
};
```

下面就开始介绍两种page allocator的实现。

## hpa

我们先需要理解hpa shard的数据结构，一个hpa shard负责管理HUGEPAGE整数倍大小的内存：

```cpp
struct hpa_shard_s {
    pai_t pai;
    hpa_central_t *central;
    malloc_mutex_t mtx;
    malloc_mutex_t grow_mtx;
    base_t *base;
    edata_cache_fast_t ecf;
    psset_t psset;
    uint64_t age_counter;
    unsigned ind;
    emap_t *emap;
    hpa_shard_opts_t opts;
    size_t npending_purge;
    hpa_shard_nonderived_stats_t stats;
    nstime_t last_purge;
};
```

hpa shard有以下关键成员变量：

- `central`: 负责管理从操作系统获得的huge page。
- `base`: 内部主要是一个edata/extent的链表，每个edata是由huge page中切分而得。另外base内部维护了若干内存使用情况的元信息，包括allocated, resident, mapped等。
- `ecf`: edata cache fast的缩写，hpa中又涉及若干层缓存edata cache fast, edata cache以及base，在后面的`edata_cache_fast_get`部分会介绍。
- `psset`: page slab set，其中维护了若干hpdata的堆，每当有allocation和deallocation请求来时，会挑选一个hpdata记录各个edata相关元信息。
- `emap`: 一个前缀树，可以在释放时找到是否相邻内存也是释放状态，进而将其合并。

> 在我的环境中hpa没有启用，需要加个环境变量`MALLOC_CONF="hpa:true”`
>

`pa_alloc`中首先会尝试从hpa进行内存分配，实际上是调用`hpa_alloc`。`hpa_alloc`只能进行对齐为PAGE的内存分配，最终调用到`hpa_alloc_batch_psset`，有几个关键函数：

- `hpa_try_alloc_batch_no_grow`：尝试从hpa shard中进行内存分配
- `hpa_central_extract`：如果从hpa shard中分配失败，尝试使用central进行分配。

相关调用路径如下，我们详细看下这几部分代码。

```cpp
hpa_alloc
  -> hpa_alloc_batch
    -> hpa_alloc_batch_psset
      -> hpa_try_alloc_batch_no_grow
        -> hpa_try_alloc_one_no_grow
          -> edata_cache_fast_get
          -> psset_pick_alloc
      -> hpa_central_extract
        -> pages_map
          -> os_pages_map
            -> mmap
        -> hpa_alloc_ps
          -> base_alloc
            -> base_alloc_impl
        -> hpdata_init
      -> psset_insert
```

## 从hpa shard中进行分配

首先我们看从hpa shard中进行分配的逻辑，入口在`hpa_try_alloc_batch_no_grow`，其中就是反复调用`hpa_try_alloc_one_no_grow`，每次成功分配一个edata，就将其加入到`result`链表中。

```cpp
static size_t hpa_try_alloc_batch_no_grow(tsdn_t *tsdn,
                                          hpa_shard_t *shard,
                                          size_t size,
                                          bool *oom,
                                          size_t nallocs,
                                          edata_list_active_t *results,
                                          bool *deferred_work_generated) {
    malloc_mutex_lock(tsdn, &shard->mtx);
    size_t nsuccess = 0;
    for (; nsuccess < nallocs; nsuccess++) {
        edata_t *edata = hpa_try_alloc_one_no_grow(tsdn, shard, size, oom);
        if (edata == NULL) {
            break;
        }
        edata_list_active_append(results, edata);
    }

    hpa_shard_maybe_do_deferred_work(tsdn, shard, /* forced */ false);
    *deferred_work_generated = hpa_shard_has_deferred_work(tsdn, shard);
    malloc_mutex_unlock(tsdn, &shard->mtx);
    return nsuccess;
}
```

有一些上下文可以补充一下，这样会更容易理解这个核心函数的逻辑：

- 一个edata负责处理具体的内存分配和释放。edata按相同大小被切分成了若干块（称为HUGEPAGE_PAGE），用来处理某个范围之内的内存分配（范围大小参照上一篇的size class）。
- 每一个edata需要维护若干元信息，比如edata对应的原始HUGEPAGE地址，以及edata中每一块的使用情况。这些元信息都保存在hpdata中。

`hpa_try_alloc_one_no_grow`的核心逻辑是：

1. 在`edata_cache_fast_get`中尝试获取一个edata，其中又涉及多层缓存：edata cache fast, edata cache以及base。
2. 获取edata后，需要在psset中获取并初始化一个hpdata，用来保存edata相关的元信息。
3. 将更新后的hpdata重新放入psset中。另外，为了高效处理内存分配和释放，还会将hpdata保存到一个radix tree中。

```cpp
static edata_t *hpa_try_alloc_one_no_grow(tsdn_t *tsdn,
                                          hpa_shard_t *shard,
                                          size_t size,
                                          bool *oom) {
    bool err;
    edata_t *edata = edata_cache_fast_get(tsdn, &shard->ecf);
    if (edata == NULL) {
        *oom = true;
        return NULL;
    }

    hpdata_t *ps = psset_pick_alloc(&shard->psset, size);
    if (ps == NULL) {
        edata_cache_fast_put(tsdn, &shard->ecf, edata);
        return NULL;
    }

    psset_update_begin(&shard->psset, ps);

    if (hpdata_empty(ps)) {
        /*
         * If the pageslab used to be empty, treat it as though it's
         * brand new for fragmentation-avoidance purposes; what we're
         * trying to approximate is the age of the allocations *in* that
         * pageslab, and the allocations in the new pageslab are
         * definitionally the youngest in this hpa shard.
         */
        hpdata_age_set(ps, shard->age_counter++);
    }

    void *addr = hpdata_reserve_alloc(ps, size);
    edata_init(edata,
               shard->ind,
               addr,
               size,
               /* slab */ false,
               SC_NSIZES,
               /* sn */ hpdata_age_get(ps),
               extent_state_active,
               /* zeroed */ false,
               /* committed */ true,
               EXTENT_PAI_HPA,
               EXTENT_NOT_HEAD);
    edata_ps_set(edata, ps);

    /*
     * This could theoretically be moved outside of the critical section,
     * but that introduces the potential for a race.  Without the lock, the
     * (initially nonempty, since this is the reuse pathway) pageslab we
     * allocated out of could become otherwise empty while the lock is
     * dropped.  This would force us to deal with a pageslab eviction down
     * the error pathway, which is a pain.
     */
    err = emap_register_boundary(tsdn, shard->emap, edata, SC_NSIZES, /* slab */ false);
    if (err) {
        hpdata_unreserve(ps, edata_addr_get(edata), edata_size_get(edata));
        /*
         * We should arguably reset dirty state here, but this would
         * require some sort of prepare + commit functionality that's a
         * little much to deal with for now.
         *
         * We don't have a do_deferred_work down this pathway, on the
         * principle that we didn't *really* affect shard state (we
         * tweaked the stats, but our tweaks weren't really accurate).
         */
        psset_update_end(&shard->psset, ps);
        edata_cache_fast_put(tsdn, &shard->ecf, edata);
        *oom = true;
        return NULL;
    }

    hpa_update_purge_hugify_eligibility(tsdn, shard, ps);
    psset_update_end(&shard->psset, ps);
    return edata;
}
```

我们详细看下这几个步骤。

### 获取edata

获取edata的逻辑是：

1. 首先尝试从hpa shard中的edata cache fast进行分配。
2. 如果分配失败，则调用`edata_cache_fast_try_fill_from_fallback`，内部实际上是从edata cache的尝试补充一个edata到edata cache fast。
3. 再次尝试从edata cache fast进行分配，如果分配失败，则调用`base_alloc_edata`尝试分配，详情参见下面的`base allocator`部分。

```cpp
edata_t *edata_cache_fast_get(tsdn_t *tsdn, edata_cache_fast_t *ecs) {
    witness_assert_depth_to_rank(tsdn_witness_tsdp_get(tsdn), WITNESS_RANK_EDATA_CACHE, 0);

    if (ecs->disabled) {
        assert(edata_list_inactive_first(&ecs->list) == NULL);
        return edata_cache_get(tsdn, ecs->fallback);
    }

    edata_t *edata = edata_list_inactive_first(&ecs->list);
    if (edata != NULL) {
        edata_list_inactive_remove(&ecs->list, edata);
        return edata;
    }
    /* Slow path; requires synchronization. */
    edata_cache_fast_try_fill_from_fallback(tsdn, ecs);
    edata = edata_list_inactive_first(&ecs->list);
    if (edata != NULL) {
        edata_list_inactive_remove(&ecs->list, edata);
    } else {
        /*
         * Slowest path (fallback was also empty); allocate something
         * new.
         */
        edata = base_alloc_edata(tsdn, ecs->fallback->base);
    }
    return edata;
}
```

```cpp
static void edata_cache_fast_try_fill_from_fallback(tsdn_t *tsdn, edata_cache_fast_t *ecs) {
    edata_t *edata;
    malloc_mutex_lock(tsdn, &ecs->fallback->mtx);
    for (int i = 0; i < EDATA_CACHE_FAST_FILL; i++) {
        edata = edata_avail_remove_first(&ecs->fallback->avail);
        if (edata == NULL) {
            break;
        }
        edata_list_inactive_append(&ecs->list, edata);
        atomic_load_sub_store_zu(&ecs->fallback->count, 1);
    }
    malloc_mutex_unlock(tsdn, &ecs->fallback->mtx);
}
```

base allocator实际就是负责和操作系统进行交互，会调用诸如`mmap`, `munmap`, `madvise`等系统调用进行内存管理。比如mmap的调用路径如下：

```cpp
base_alloc_impl
  -> base_extent_alloc
    -> base_block_alloc
      -> base_map
        -> pages_map
          -> os_pages_map
            -> mmap
```

这里补充说明下base allocator，其数据结构如下，核心就是它保存了一个base block的链表`blocks`。而base block就是一个edata的封装。

```cpp
struct base_s {
	ehooks_t ehooks;
	ehooks_t ehooks_base;
	malloc_mutex_t mtx;
	bool auto_thp_switched;
	pszind_t pind_last;
	size_t extent_sn_next;
	base_block_t *blocks;
	edata_heap_t avail[SC_NSIZES];
	size_t allocated;
	size_t resident;
	size_t mapped;
	size_t n_thp;
};
```

核心函数是`base_alloc_impl`:

1. 遍历base中的各个能够分配给定size的堆，尝试找到一个非空的edata(也就是extent)。
2. 如果找不到，则调用`base_extent_alloc`，使用base来分配一个edata。
3. 调用`base_extent_bump_alloc`初始化这个edata，最终会调用`edata_binit`设置各种成员变量。

```cpp
static void *base_alloc_impl(
        tsdn_t *tsdn, base_t *base, size_t size, size_t alignment, size_t *esn) {
    alignment = QUANTUM_CEILING(alignment);
    size_t usize = ALIGNMENT_CEILING(size, alignment);
    size_t asize = usize + alignment - QUANTUM;

    edata_t *edata = NULL;
    malloc_mutex_lock(tsdn, &base->mtx);
    for (szind_t i = sz_size2index(asize); i < SC_NSIZES; i++) {
        edata = edata_heap_remove_first(&base->avail[i]);
        if (edata != NULL) {
            /* Use existing space. */
            break;
        }
    }
    if (edata == NULL) {
        /* Try to allocate more space. */
        edata = base_extent_alloc(tsdn, base, usize, alignment);
    }
    void *ret;
    if (edata == NULL) {
        ret = NULL;
        goto label_return;
    }

    ret = base_extent_bump_alloc(base, edata, usize, alignment);
    if (esn != NULL) {
        *esn = (size_t)edata_sn_get(edata);
    }
label_return:
    malloc_mutex_unlock(tsdn, &base->mtx);
    return ret;
}
```

`base_extent_alloc`是base allocator分配extent的核心函数：

1. 调用`base_block_alloc`获取一个base block。
2. base allocator中有一个base block链表，获取到一个base block之后，将其加入链表头。
3. 然后返回这个base block中的edata。

```cpp
static edata_t *base_extent_alloc(tsdn_t *tsdn, base_t *base, size_t size, size_t alignment) {
    malloc_mutex_assert_owner(tsdn, &base->mtx);

    ehooks_t *ehooks = base_ehooks_get_for_metadata(base);
    /*
     * Drop mutex during base_block_alloc(), because an extent hook will be
     * called.
     */
    malloc_mutex_unlock(tsdn, &base->mtx);
    base_block_t *block = base_block_alloc(tsdn,
                                           base,
                                           ehooks,
                                           base_ind_get(base),
                                           &base->pind_last,
                                           &base->extent_sn_next,
                                           size,
                                           alignment);
    malloc_mutex_lock(tsdn, &base->mtx);
    if (block == NULL) {
        return NULL;
    }
    block->next = base->blocks;
    base->blocks = block;
    if (config_stats) {
        base->allocated += sizeof(base_block_t);
        base->resident += PAGE_CEILING(sizeof(base_block_t));
        base->mapped += block->size;
        if (metadata_thp_madvise() &&
            !(opt_metadata_thp == metadata_thp_auto && !base->auto_thp_switched)) {
            assert(base->n_thp > 0);
            base->n_thp += HUGEPAGE_CEILING(sizeof(base_block_t)) >> LG_HUGEPAGE;
        }
        assert(base->allocated <= base->resident);
        assert(base->resident <= base->mapped);
        assert(base->n_thp << LG_HUGEPAGE <= base->mapped);
    }
    return &block->edata;
}
```

### 初始化hpdata

无论通过edata cache fast, edata cache还是base allocator获取到一个edata/extent之后，此时还无法直接进行内存分配。这是因为一个edata/extent内部是被划分为相同大小的多块内存区域，还需要在extent中找到一个具体的可用地址。一个extent中，它的实际地址指向哪块内存，内存来源是不是HUGEPAGE，是否允许分配内存等等元信息，都会保存在hpdata这个类中。每个edata都有一个hpdata指针指向对应的hpdata，hpdata实际则是保存在hpa shard的psset中。

因此在获取edata后，接下来需要在hpa shard中的psset中挑选一个合适的`hpdata`，用来保存这个edata的metadata，具体步骤如下：

1. `psset_pick_alloc`: 找到了一个足够分配特定大小的hpdata。
2. `hpdata_reserve_alloc`: 需要进一步处理在extent找到对应可用的内存范围，对应修改上一步获取的hpdata，最终返回这个edata中这次内存分配的地址。
3. `edata_init`: 根据首地址和大小初始化edata。
4. `edata_ps_set`: 将edata中的`e_ps`指针指向这个hpdata，以供后续edata使用。

psset的数据结构如下：

- `pageslab`是一个堆数组， 多个堆对应不同大小size class，每个堆中保存相同大小的hpdata。
- `pageslab_bitmap`是一个bitmap，用来快速检查这些page slab是否为空。

```cpp
struct psset_s {
    hpdata_age_heap_t pageslabs[PSSET_NPSIZES];
    fb_group_t pageslab_bitmap[FB_NGROUPS(PSSET_NPSIZES)];
    psset_bin_stats_t merged_stats;
    psset_stats_t stats;
    hpdata_empty_list_t empty;
    hpdata_purge_list_t to_purge[PSSET_NPURGE_LISTS];
    fb_group_t purge_bitmap[FB_NGROUPS(PSSET_NPURGE_LISTS)];
    hpdata_hugify_list_t to_hugify;
};
```

hpdata本质上就是用来保存extent的metadata，代码中有一段注释是这么描述的：

```cpp
/*
 * The metadata representation we use for extents in hugepages.  While the PAC
 * uses the edata_t to represent both active and inactive extents, the HP only
 * uses the edata_t for active ones; instead, inactive extent state is tracked
 * within hpdata associated with the enclosing hugepage-sized, hugepage-aligned
 * region of virtual address space.
 *
 * An hpdata need not be "truly" backed by a hugepage (which is not necessarily
 * an observable property of any given region of address space).  It's just
 * hugepage-sized and hugepage-aligned; it's *potentially* huge.
 */
```

`psset_pick_alloc`的逻辑如下：

1. 首先通过`fb_ffs`中，根据bitmap计算给定size的pageslab堆所在下标
2. 通过下标，检查pageslab堆是否有可用的hpdata，并返回

```cpp
hpdata_t *psset_pick_alloc(psset_t *psset, size_t size) {
    assert((size & PAGE_MASK) == 0);
    assert(size <= HUGEPAGE);

    pszind_t min_pind = sz_psz2ind(sz_psz_quantize_ceil(size));
    pszind_t pind = (pszind_t)fb_ffs(psset->pageslab_bitmap, PSSET_NPSIZES, (size_t)min_pind);
    if (pind == PSSET_NPSIZES) {
        return hpdata_empty_list_first(&psset->empty);
    }
    hpdata_t *ps = hpdata_age_heap_first(&psset->pageslabs[pind]);
    if (ps == NULL) {
        return NULL;
    }

    hpdata_assert_consistent(ps);

    return ps;
}
```

### emap

在edata和hpdata都设置好之后，会调用`emap_register_boundary`将edata保存到hpa shard的emap中，emap其本质就是个就是个radix tree。这个radix tree的主要作用是根据edata指针找到与它相关的内存分配信息，进而在分配和释放时候进行可用区域的合并或者切分。

```cpp
struct emap_s {
    rtree_t rtree;
};
```

这个radix tree有两层，所有数据都保存在叶子节点中，数据结构如下：

```cpp
struct rtree_leaf_elm_s {
    /*
     * Single pointer-width field containing all three leaf element fields.
     * For example, on a 64-bit x64 system with 48 significant virtual
     * memory address bits, the index, edata, and slab fields are packed as
     * such:
     *
     * x: index
     * e: edata
     * s: state
     * h: is_head
     * b: slab
     *
     *   00000000 xxxxxxxx eeeeeeee [...] eeeeeeee e00ssshb
     */
    atomic_p_t le_bits;
};
```

其内部就是一个64位指针，所有这些信息都是通过编码`rtree_contents_t`而得，相关数据结构如下：

```cpp
enum extent_state_e {
    extent_state_active = 0,
    extent_state_dirty = 1,
    extent_state_muzzy = 2,
    extent_state_retained = 3,
    extent_state_transition = 4, /* States below are intermediate. */
    extent_state_merging = 5,
    extent_state_max = 5 /* Sanity checking only. */
};

struct rtree_metadata_s {
    szind_t szind;
    extent_state_t state; /* Mirrors edata->state. */
    bool is_head;         /* Mirrors edata->is_head. */
    bool slab;
};

struct rtree_contents_s {
    edata_t *edata;
    rtree_metadata_t metadata;
};
```

具体64位中每一位的作用如下：

- 0-8：空闲。
- 9-16：这次内存分配大小在size class中的下标。
- 17-57：extent指针，由于extent的地址在分配时是128位对齐过的，低７位一定是0，不会与其它位冲突。又由于它不会大于Linux下虚拟地址长度（低48位），因而17-57位不会和其他字段冲突。
- 58~59：空闲。
- 60~62：extent状态，参照上面`extent_state_e`。
- 63：extent是否是链表头，即`rtree_metadata_s`中的`is_head`这个布尔类型成员变量。（作用未知）
- 64：extent是否是一个slab，即`rtree_metadata_s`中的slab这个布尔类型成员变量。（实际是在调用`pa_alloc`时指定的，small class为true，large class及更大为false）

编码这个64位指针代码如下：

```jsx
uintptr_t rtree_leaf_elm_bits_encode(rtree_contents_t contents) {
    assert((uintptr_t)contents.edata % (uintptr_t)EDATA_ALIGNMENT == 0);
    uintptr_t edata_bits = (uintptr_t)contents.edata & (((uintptr_t)1 << LG_VADDR) - 1);

    uintptr_t szind_bits = (uintptr_t)contents.metadata.szind << LG_VADDR;
    uintptr_t slab_bits = (uintptr_t)contents.metadata.slab;
    uintptr_t is_head_bits = (uintptr_t)contents.metadata.is_head << 1;
    uintptr_t state_bits = (uintptr_t)contents.metadata.state << RTREE_LEAF_STATE_SHIFT;
    uintptr_t metadata_bits = szind_bits | state_bits | is_head_bits | slab_bits;
    assert((edata_bits & metadata_bits) == 0);

    return edata_bits | metadata_bits;
}
```

> edata指针与emap的叶子节点不是一一对应的，当分配一个extent ，它可能跨越多页。这时这些页的起始位置都需要作为key插入到radix tree中，它们对应了相同的value。相关实现在`emap_register_boundary`和`emap_register_interior`中，之所以分成两个是因为boundary查找需要一层层找，直到叶子节点，而interior 可以在叶子结点一个个遍历插入）。通过将指针align到页大小来找到它对应的value。
>

此外，为了加速查找，emap中还有两层cache：

```cpp
struct rtree_ctx_s {
    /* Direct mapped cache. */
    rtree_ctx_cache_elm_t cache[RTREE_CTX_NCACHE];
    /* L2 LRU cache. */
    rtree_ctx_cache_elm_t l2_cache[RTREE_CTX_NCACHE_L2];
};
```

L1 cache 有16个位置，将最后一个非叶子层的key模上16得到L1 cache。如果不命中，则去L2 cache 找，它是一个８个位置的数组，需要一个个遍历。如果L2也不命中，才按照key去radix tree中查找。

```cpp
bool emap_register_boundary(
        tsdn_t *tsdn, emap_t *emap, edata_t *edata, szind_t szind, bool slab) {
    assert(edata_state_get(edata) == extent_state_active);
    EMAP_DECLARE_RTREE_CTX;

    rtree_leaf_elm_t *elm_a, *elm_b;
    bool err = emap_rtree_leaf_elms_lookup(
            tsdn, emap, rtree_ctx, edata, false, true, &elm_a, &elm_b);
    if (err) {
        return true;
    }
    assert(rtree_leaf_elm_read(tsdn,
                               &emap->rtree,
                               elm_a,
                               /* dependent */ false)
                   .edata == NULL);
    assert(rtree_leaf_elm_read(tsdn,
                               &emap->rtree,
                               elm_b,
                               /* dependent */ false)
                   .edata == NULL);
    emap_rtree_write_acquired(tsdn, emap, elm_a, elm_b, edata, szind, slab);
    return false;
}
```

有关这个radix tree的实现有机会再继续深入研究。

到这里hpa shard中的核心逻辑已经介绍结束了，简单总结一下：

1. 获取edata。每个edata就对应一片物理内存，获取edata的过程中先尝试从hpa shard的基层缓存中获取（包括edata cache fast和edata cache），如果缓存命中失败，则需要和操作系统交互申请一片内存。
2. 初始化edata对应的元信息，保存在hpdata中，然后将这个hpdata保存在hpa shard的psset中。
3. 将edata加入到hpa shard的前缀树emap中，方便后续释放时进行可用内存的合并。

## 从central进行分配

如果上一步从hpa shard分配失败，则会调用`hpa_central_extract`尝试从central进行分配，主要逻辑如下：

1. 检查每个shard的central中的地址是否为空：如果为空，在`pages_map`中通过mmap获取128个HUGEPAGE大小的内存，保存在central中。
2. `hpa_alloc_ps`: 通过central中的base，获取一块能保存`hpdata_t`的内存
3. `hpdata_init`: 初始化hpdata
4. `psset_insert`: 将这块内存保存到hpa shard的psset中。

## 写在最后

本来是希望这一篇把整个物理层都讲完，没想到hpa涉及的细节这么多，下一篇我们介绍物理层的另一种page allocator: pac (page allocator classic)。