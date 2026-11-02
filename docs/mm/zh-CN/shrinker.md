# 收缩器 API

> 一种回调机制，允许子系统参与内存回收——当内核需要内存时，它会要求已注册的收缩器释放其缓存。

## 什么是收缩器？

内核维护着许多内存中的缓存：目录项（dentry）缓存、inode 缓存、DMA-buf 堆、驱动私有对象池、zswap 池等。这些缓存很有价值——它们避免了昂贵的重复计算或 I/O，但在系统压力较大时，必须释放内存。

收缩器 API 是子系统向内存管理核心注册的方式，用来声明：“我有可以让你回收的对象。”当回收运行时，它会在每次扫描中调用每个已注册的收缩器两次：第一次询问有多少内存可释放（计数阶段），第二次实际释放部分内存（扫描阶段）。

树中收缩器的例子：

- `fs/super.c` — 覆盖 dentry LRU 和 inode LRU 的每个超级块收缩器（`super_cache_scan` / `super_cache_count`）
- `mm/huge_memory.c` — 延迟拆分 THP 收缩器
- `mm/zswap.c` — zswap 池收缩器
- `mm/workingset.c` — workingset 影子条目收缩器
- `fs/xfs/xfs_icache.c` — XFS inode GC 收缩器
- `mm/vmalloc.c` — vmap 节点收缩器

## `struct shrinker`

来自 `include/linux/shrinker.h`：

```c
struct shrinker {
    unsigned long (*count_objects)(struct shrinker *,
                                   struct shrink_control *sc);
    unsigned long (*scan_objects)(struct shrinker *,
                                  struct shrink_control *sc);

    long batch;     /* reclaim batch size, 0 = default */
    int seeks;      /* seeks to recreate an obj */
    unsigned flags;

    /*
     * The reference count of this shrinker. Registered shrinkers have an
     * initial refcount of 1, then lookup operations are allowed to use it
     * via shrinker_try_get(). On unregistration the initial refcount is
     * discarded, and the shrinker is freed asynchronously via RCU after
     * its refcount reaches 0.
     */
    refcount_t refcount;
    struct completion done;  /* used to wait for refcount to reach 0 */
    struct rcu_head rcu;

    void *private_data;

    /* Internal use */
    struct list_head list;
#ifdef CONFIG_MEMCG
    int id;          /* ID in shrinker_idr */
#endif
#ifdef CONFIG_SHRINKER_DEBUG
    int debugfs_id;
    const char *name;
    struct dentry *debugfs_entry;
#endif
    /* objs pending delete, per node */
    atomic_long_t *nr_deferred;
};

#define DEFAULT_SEEKS 2
```

### 字段说明

| 字段 | 描述 |
|-------|------|
| `count_objects` | 返回“可能”被释放的对象数量。调用时不会真的释放任何对象。 |
| `scan_objects` | 释放最多 `sc->nr_to_scan` 个对象。返回已释放的数量、`SHRINK_STOP` 或 `SHRINK_EMPTY`。 |
| `batch` | 每次 `scan_objects` 调用扫描多少对象。`0` 表示使用内部默认值 `128`。较大的批量可减少调用开销；较小的批量可改善延迟。 |
| `seeks` | 重新创建一个对象的相对成本（以磁盘寻道次数为单位）。用于优先级缩放公式。若不确定，使用 `DEFAULT_SEEKS`（2）。当对象不需要 I/O 即可重建时，将其设为 0，会导致它们被更激进地裁剪。 |
| `flags` | `SHRINKER_NUMA_AWARE`、`SHRINKER_MEMCG_AWARE`、`SHRINKER_NONSLAB`——见下文。 |
| `private_data` | 收缩器实现的不透明指针，例如 `fs/super.c` 中的 `super_block *`。 |
| `nr_deferred` | 每个节点上已计算但尚未完成的扫描工作计数（由于 `batch` 舍入或竞争造成）。会延续到下一次调用中。由内核核心管理。 |

## `struct shrink_control`

内核会在每次调用两个回调函数时向它们传递一个 `struct shrink_control`。

```c
struct shrink_control {
    gfp_t gfp_mask;

    /* current node being shrunk (for NUMA aware shrinkers) */
    int nid;

    /*
     * How many objects scan_objects should scan and try to reclaim.
     * Reset before every call.
     */
    unsigned long nr_to_scan;

    /*
     * How many objects did scan_objects process?
     * Defaults to nr_to_scan; callee should update to reflect actual progress.
     */
    unsigned long nr_scanned;

    /* current memcg being shrunk (for memcg aware shrinkers) */
    struct mem_cgroup *memcg;
};
```

### 字段语义

**`gfp_mask`** — 触发回收的分配上下文。收缩器必须尊重它。如果 `__GFP_FS` 未设置，收缩器不得进入文件系统（例如，`super_cache_scan` 会在这种情况下立即返回 `SHRINK_STOP`）。如果 `__GFP_IO` 未设置，则应避免启动 I/O。

**`nid`** — 当前正在回收的 NUMA 节点。只有在设置了 `SHRINKER_NUMA_AWARE` 时才有意义；其他收缩器始终看到节点 0。

**`nr_to_scan`** — 在调用 `scan_objects` 前由核心设置。收缩器应尝试释放这个数量的对象。它可以被修改（内核在每次调用前会重置它）。

**`nr_scanned`** — 由核心预先填充为 `nr_to_scan`。收缩器应更新它以反映实际检查了多少对象（而不仅仅是释放了多少）。内核用它来计算更新 `nr_deferred` 时已完成的工作量。

**`memcg`** — 目标内存 cgroup。仅在遍历按 memcg 处理的收缩器时由核心设置；对于全局（root memcg）路径则为 `NULL`。只有设置了 `SHRINKER_MEMCG_AWARE` 时才有意义。

## `count_objects` 与 `scan_objects`

这两个回调实现了两阶段契约：

### `count_objects` — 估算可回收容量

```c
unsigned long (*count_objects)(struct shrinker *, struct shrink_control *sc);
```

返回缓存中“可能”被释放的对象数量。返回值决定了是否会调用 `scan_objects`：

| 返回值 | 含义 |
|---|---|
| `SHRINK_EMPTY` (`~0UL - 1`) | 缓存已完全为空。核心会清除该收缩器在 per-memcg 中的位，跳过后续扫描，直到有新对象被添加。 |
| `0` | 目前没有可回收对象（例如，所有对象都被锁定，或计数低于阈值）。此回合中不会调用 `scan_objects`。 |
| `N > 0` | 大约有 `N` 个可回收对象。核心会据此计算通过 `nr_to_scan` 请求扫描多少对象。 |

**没有锁要求。** `count_objects` 不能获取可能与回收发生死锁的锁。计数可能过期——核心容忍 `count_objects` 和 `scan_objects` 实际释放量之间的差异。这是设计使然：计数用于按比例缩放，而不是硬性限制。

### `scan_objects` — 实际释放对象

```c
unsigned long (*scan_objects)(struct shrinker *, struct shrink_control *sc);
```

遍历缓存，释放最多 `sc->nr_to_scan` 个对象，更新 `sc->nr_scanned`，并返回已释放对象的数量。

| 返回值 | 含义 |
|---|---|
| `N >= 0` | 本次调用中释放的对象数量。 |
| `SHRINK_STOP` (`~0UL`) | 目前无法继续推进——需要的锁可能在当前回收上下文中造成死锁。核心不会在当前回收调用中再次调用该收缩器的 `scan_objects`。 |
| `SHRINK_EMPTY` (`~0UL - 1`) | 缓存中已无对象。 |

```c
/* Defined in include/linux/shrinker.h */
#define SHRINK_STOP  (~0UL)
#define SHRINK_EMPTY (~0UL - 1)
```

## 收缩器标志

```c
#define SHRINKER_NUMA_AWARE   BIT(2)
#define SHRINKER_MEMCG_AWARE  BIT(3)
#define SHRINKER_NONSLAB      BIT(4)
```

**`SHRINKER_NUMA_AWARE`** — 该收缩器会按 NUMA 节点调用一次。`sc->nid` 会设置为当前正在回收的节点。当你的缓存按节点分区（例如按节点的 LRU）时，使用这个标志。没有这个标志时，收缩器只会调用一次，且无论哪个节点压力较大，`nid` 都为 0。

**`SHRINKER_MEMCG_AWARE`** — 该收缩器参与按 memcg 的回收。当某个 memcg 超过其限制时，核心会调用设置了 `MEMCG_AWARE` 的收缩器，并将 `sc->memcg` 设为目标 cgroup。随后，收缩器必须只释放归属于该 memcg 的对象。这要求你的 LRU 在初始化时使用 `list_lru_init_memcg()`。没有这个标志时，收缩器只会在全局（root cgroup）回收期间被调用。

**`SHRINKER_NONSLAB`** — 仅在与 `SHRINKER_MEMCG_AWARE` 组合时才有意义。

## 内核如何调用收缩器

### 调用路径

```
内存压力（分配失败或水位线被越过）
    │
    ├── kswapd（后台） ──────────────────────────────┐
    │                                                      │
    └── 直接回收（分配进程阻塞） ───────────────────────┤
                                                           │
                                                           v
                                              shrink_node() in mm/vmscan.c
                                                           │
                                                           v
                                              shrink_slab(gfp_mask, nid,
                                                          memcg, priority)
                                              in mm/shrinker.c
                                                           │
                                     ┌─────────────────────┴──────────────────────┐
                                     │ (非 root memcg)                           │ (root)
                                     v                                            v
                          shrink_slab_memcg()                    list_for_each_entry_rcu
                          遍历每个 memcg 位图                  遍历 shrinker_list
                                     │                                            │
                                     └──────────────┬─────────────────────────────┘
                                                    v
                                          do_shrink_slab(sc, shrinker, priority)
                                                    │
                                          1. 调用 count_objects()
                                          2. 通过 priority 计算 nr_to_scan
                                          3. 循环：按批次调用 scan_objects()
                                             直到 total_scan 用尽
                                             或返回 SHRINK_STOP
```

### `do_shrink_slab` 中的优先级缩放

`priority` 参数（来自 `mm/vmscan.c` 中的 `sc->priority`）控制收缩器被调用的激进程度。优先级越高，表示内存压力越大。

```c
/* From mm/shrinker.c: do_shrink_slab() */
if (shrinker->seeks) {
    delta = freeable >> priority;  /* at priority 12: very small delta */
    delta *= 4;
    do_div(delta, shrinker->seeks);
} else {
    delta = freeable / 2;          /* seeks=0: trim half regardless */
}

total_scan = nr >> priority;   /* deferred work, scaled down at low priority */
total_scan += delta;
total_scan = min(total_scan, (2 * freeable));
```

在 `priority = DEF_PRIORITY`（12，压力较低）时：`freeable >> 12` 非常小——收缩器几乎不会被触发。随着优先级接近 0（极端压力）：会应用完整的 deferred 计数和更大的 `delta`，每次调用会扫描更多对象。`total_scan` 被限制为 `2 * freeable`，以防止过度扫描。

`do_shrink_slab` 随后会循环调用 `scan_objects`，每次批量处理 `batch_size` 个对象，直到 `total_scan` 耗尽，或者返回 `SHRINK_STOP`。任何已经计算但尚未完成的工作（例如由于批次舍入造成）会保存在 `nr_deferred` 中，并在下一次调用时追加。

### 并发与锁

收缩器列表通过 `shrinker_mutex` 保护其修改（注册/注销），并通过 RCU 进行遍历。个别收缩器的查找使用 `shrinker_try_get()`（`refcount_inc_not_zero`）在 `do_shrink_slab` 运行期间持有引用。`shrinker_free()` 调用 `wait_for_completion(&shrinker->done)`，以等待所有正在进行中的引用都被释放，然后再释放内存。

这意味着 `scan_objects` 可以在不同 CPU 上并发执行（例如不同节点或不同 memcg）。收缩器实现必须在并发调用下安全可用。

## 注册与生命周期

### 三阶段生命周期

```c
/* 1. Allocate and configure */
struct shrinker *shrinker;

shrinker = shrinker_alloc(SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE,
                           "mydriver-%s", device_name);
if (!shrinker)
    return -ENOMEM;

shrinker->count_objects = mydriver_count_objects;
shrinker->scan_objects  = mydriver_scan_objects;
shrinker->seeks         = DEFAULT_SEEKS;
shrinker->private_data  = my_private_ctx;

/* 2. Register (shrinker becomes visible to reclaim) */
shrinker_register(shrinker);

/* ... device/module in use ... */

/* 3. Free (unregisters, drains references, then frees asynchronously via RCU) */
shrinker_free(shrinker);
```

### `shrinker_alloc(flags, fmt, ...)`

分配并初始化一个 `struct shrinker`。`fmt` 参数是一个类似 `printf` 的名字，用于 `/sys/kernel/debug/shrinker/`（在 `CONFIG_SHRINKER_DEBUG` 开启时）。如果请求了 `SHRINKER_MEMCG_AWARE`，但运行时禁用了 memcg，则该标志会被静默清除，收缩器会回退到非 memcg 模式。

分配失败时返回 `NULL`。

### `shrinker_register(shrinker)`

将收缩器加入 `shrinker_list`，设置 `SHRINKER_REGISTERED`，并将 `refcount` 初始化为 1（使其对回收可见）。只能在由 `shrinker_alloc()` 返回的收缩器上调用。

### `shrinker_free(shrinker)`

通过一次调用处理解除注册和释放：

1. 若已注册：释放初始引用计数，并等待（`wait_for_completion`）所有正在进行中的 `do_shrink_slab` 调用结束。
2. 在 `shrinker_mutex` 下从 `shrinker_list` 中移除。
3. 若启用了 `SHRINKER_MEMCG_AWARE`，则从 memcg IDR 中移除。
4. 通过 `call_rcu` 安排该结构本身执行 `kfree`。

在 `shrinker_free()` 返回后，可以安全地释放持有该收缩器的结构（例如 `super_block`、驱动的设备结构）。

### 何时调用各步骤

| 情况 | 何时注册 | 何时释放 |
|------|----------|----------|
| 模块全局缓存 | `module_init` / `subsys_initcall` | `module_exit` |
| 每设备缓存 | `probe()`，在缓存初始化后 | `remove()` / `probe()` 的错误路径 |
| 每超级块缓存 | `sget()` / `alloc_super()`，在挂载完成前 | `kill_sb()` / `deactivate_super()` |

## 真实示例：超级块收缩器

每个已挂载的文件系统都有一个按超级块管理的收缩器，覆盖 dentry LRU 和 inode LRU。它在 `alloc_super()` 中设置（`fs/super.c`）：

```c
/* fs/super.c: alloc_super() */
s->s_shrink = shrinker_alloc(SHRINKER_NUMA_AWARE | SHRINKER_MEMCG_AWARE,
                              "sb-%s", type->name);

s->s_shrink->scan_objects  = super_cache_scan;
s->s_shrink->count_objects = super_cache_count;
s->s_shrink->batch         = 1024;
s->s_shrink->private_data  = s;
```

收缩器会在超级块完全建立并标记为 `SB_BORN` 后再注册：

```c
/* fs/super.c: vfs_get_tree() / mount_fs() path */
shrinker_register(s->s_shrink);
```

**`super_cache_count`** 会查询 dentry 和 inode LRU 的 `list_lru_shrink_count()`，再加上来自 `s_op->nr_cached_objects()` 的文件系统私有对象数量；如果总数为 0，则返回 `SHRINK_EMPTY`。

**`super_cache_scan`** 会按比例将 `sc->nr_to_scan` 分配给 dentry、inode 和 FS 私有对象。它会先调用 `prune_dcache_sb()`（dentry 会锁住 inode），然后是 `prune_icache_sb()`，最后调用 `s_op->free_cached_objects()`。如果 `sc->gfp_mask` 中没有设置 `__GFP_FS`，它会立即返回 `SHRINK_STOP`，以避免重新进入可能持有锁的文件系统。

## 常见错误

!!! warning "在 `scan_objects` 中阻塞或睡眠，却没有检查 `gfp_mask`"
    `scan_objects` 可能在任何上下文下由直接回收调用。如果调用方持有你的收缩器也需要的锁，或者回收发生在文件系统锁之下，再获取该锁就会造成死锁。始终先检查 `sc->gfp_mask` 里的 `__GFP_FS` 和 `__GFP_IO`，再进入文件系统或启动 I/O。若无法安全推进，请返回 `SHRINK_STOP`。

!!! warning "在缓存实际上为空时返回 0"
    `0` 表示“现在没有可回收对象”（例如对象被锁定）。`SHRINK_EMPTY` 表示“缓存完全为空”。这两者不能混淆。对于全局回收路径而言，在缓存为空时返回 `0` 没有明显代价，但在 per-memcg 路径中，它会阻止内核清除该收缩器在 memcg 位图中的位，从而导致未来不必要的唤醒。

!!! warning "误处理 `nr_scanned`"
    `sc->nr_scanned` 由核心预先填充为 `sc->nr_to_scan`。如果你的 `scan_objects` 检查到的对象数少于请求值（例如碰到 trylock 并跳过了一些对象），应更新 `sc->nr_scanned` 以反映实际检查数。核心会据此计算 `next_deferred`。如果在实际扫描更少时仍保留为 `nr_to_scan`，就会错误地认为这些工作已完成，并低估 deferred 工作量。

!!! warning "在错误路径中忘记调用 `shrinker_free`"
    如果 `shrinker_register` 已被调用，而你的设备或模块初始化随后失败，那么在返回错误前必须调用 `shrinker_free`。一个已注册的收缩器若指向已释放或未完全初始化的数据，在下一次回收周期中就会被调用，导致 use-after-free 或 NULL 解引用。`shrinker_free` 能同时处理未注册情况（若尚未注册则视为空操作）和已注册情况，因此在 `shrinker_alloc` 成功后，任何清理路径都可以无条件调用它。

!!! warning "`scan_objects` 的并发调用"
    内核可以为不同节点（若启用 `SHRINKER_NUMA_AWARE`）或不同 memcgs（若启用 `SHRINKER_MEMCG_AWARE`）并发调用 `scan_objects`。你的 LRU 遍历和计数器必须在并发访问下安全。应优先使用 `list_lru`（它内置了按节点、按 memcg 的锁），而不是自行实现列表。

!!! warning "无意中设置 `seeks = 0`"
    将 `seeks` 设为 0 会触发 `do_shrink_slab` 中的激进分支：`delta = freeable / 2`。这表示在任何优先级下，收缩器都会在一次扫描中请求处理一半的 freeable 对象。这对于可廉价重建的缓存是有意设计的（不需要 I/O），但如果误用于昂贵缓存，会在压力下造成过度驱逐。

## 关键源文件

| 文件 | 描述 |
|------|------|
| [`include/linux/shrinker.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/shrinker.h) | `struct shrinker`、`struct shrink_control`、`SHRINK_STOP`、`SHRINK_EMPTY`、标志以及 `shrinker_alloc` / `shrinker_register` / `shrinker_free` 声明 |
| [`mm/shrinker.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/shrinker.c) | `shrinker_alloc`、`shrinker_register`、`shrinker_free`、`do_shrink_slab`、`shrink_slab`、`shrink_slab_memcg` 以及 memcg 位图管理 |
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | `shrink_node`、`shrink_lruvec`——回收过程中调用 `shrink_slab()` 的位置 |
| [`fs/super.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/super.c) | `super_cache_scan`、`super_cache_count`，以及 `alloc_super` 中的每超级块收缩器设置 |
| [`fs/dcache.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/dcache.c) | `prune_dcache_sb`——由 `super_cache_scan` 调用，用于收缩 dentry LRU |
| [`fs/inode.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/fs/inode.c) | `prune_icache_sb`——由 `super_cache_scan` 调用，用于收缩 inode LRU |
| [`include/linux/list_lru.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/list_lru.h) | `list_lru_init_memcg`、`list_lru_shrink_count`、`list_lru_shrink_walk`——与收缩器配合使用的 NUMA/memcg 感知 LRU |

## 进一步阅读

- [mm/shrinker.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/shrinker.c) — `shrinker_alloc()`、`shrinker_register()`、`shrinker_free()`、`do_shrink_slab()`（优先级缩放和批量循环）、`shrink_slab()` 和 `shrink_slab_memcg()`，以及 per-memcg 位图管理
- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — `shrink_node()` 和 `shrink_lruvec()`，回收入口；在扫描 LRU 列表之后调用 `shrink_slab()`
- [LWN: Smarter shrinkers](https://lwn.net/Articles/550463/) — 两阶段 `count_objects` / `scan_objects` 分离设计的动机与背景，取代旧的单回调模型
- [LWN: Shrinker reorganization](https://lwn.net/Articles/966451/) — 将收缩器代码从 `mm/vmscan.c` 中抽离到专门的 `mm/shrinker.c` 文件，并引入了基于 RCU 的引用计数生命周期
- [reclaim](reclaim.md) — 更广泛的回收场景，展示 `shrink_slab()` 与 LRU 扫描、kswapd、直接回收的关系
- [memcg](memcg.md) — per-cgroup 回收上下文：`sc->memcg` 设置方式，以及 `SHRINKER_MEMCG_AWARE` 收缩器与 cgroup 内存限制的交互
