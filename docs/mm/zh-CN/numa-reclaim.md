# NUMA对内存回收的影响

> 在多槽位系统上，node级的watermarks，node级的kswapd线程，以及NUMA均衡是怎么相互影响形成了现在的内存回收机制。以及当远程节点承受压力时可能出现的问题

## 背景: NUMA 内存是节点本地的

在NUMA系统中，每个物理CPU槽位（**node**）拥有一个内存池。内核将每个节点表示为`pg_data_t`（也称为`pgdat`）。每个`pgdat`包含一个或多个区域（`struct zone`），每个区域保持自己的watermark阈值。

```
有两个NUMA-node的系统
---------------------------------------------------------
  Node 0 (pgdat 0)                Node 1 (pgdat 1)
  +--------------------+          +--------------------+
  | zone: ZONE_DMA     |          | zone: ZONE_DMA     |
  | zone: ZONE_DMA32   |          | zone: ZONE_DMA32   |
  | zone: ZONE_NORMAL  |          | zone: ZONE_NORMAL  |
  |                    |          |                    |
  |  kswapd0 thread    |          |  kswapd1 thread    |
  +--------------------+          +--------------------+
```

每个节点**有一个kswapd内核线程**。回收计数、watermark和LRU列表都是按节点计算的。这是驱动本文档其他内容的基本事实。

---

## Node级别的watermarks

每个 `struct zone` 存储了它们自己的 watermark 数组:

```c
/* include/linux/mmzone.h */
enum zone_watermarks {
    WMARK_MIN,
    WMARK_LOW,
    WMARK_HIGH,
    WMARK_PROMO,        /* used when memory-tiering NUMA balancing is on */
    NR_WMARK
};

struct zone {
    /* ... */
    unsigned long _watermark[NR_WMARK];
    unsigned long watermark_boost;   /* temporary boost after fragmentation events */
    /* ... */
};
```
额外还有一个 `watermark_boost` :

```c
static inline unsigned long wmark_pages(const struct zone *z,
                                         enum zone_watermarks w)
{
    return z->_watermark[w] + z->watermark_boost;
}

static inline unsigned long min_wmark_pages(const struct zone *z)  { ... }
static inline unsigned long low_wmark_pages(const struct zone *z)  { ... }
static inline unsigned long high_wmark_pages(const struct zone *z) { ... }
static inline unsigned long promo_wmark_pages(const struct zone *z){ ... }
```

watermarks控制两条不同的回收路径：

| 条件 | 行为 |
|---|---|
| free pages < `low_wmark_pages(zone)` | `wakeup_kswapd()` — 唤醒node的kswapd |
| free pages < `min_wmark_pages(zone)` | 直接回收 — 当前分配进程被阻塞 |

由于每个节点的每个区域都有自己的watermark，**Node 1可以位于 `WMARK_MIN`，而Node 0则舒适地位于`WMARK_HIGH`之上**。内核不对节点间的水印进行平均。

---

## Node级kswapd

### 每个pgdat都有一个内核线程

也就是每个node对应一个kswapd。`kswapd_run()` 在系统启动的时候创建这个线程:

```c
/* mm/vmscan.c */
void __meminit kswapd_run(int nid)
{
    pg_data_t *pgdat = NODE_DATA(nid);
    /* ... */
    pgdat->kswapd = kthread_create_on_node(kswapd, pgdat, nid,
                                            "kswapd%d", nid);
    /* ... */
    wake_up_process(pgdat->kswapd);
}
```

这个线程固定在它所属的node上(`kthread_create_on_node`) 因此，kswapd 本身所做的分配来自正确的本地池。存储在 `pg_data_t::kswapd` 中的`pgdat`指针是该节点的线程句柄。

### 怎么唤醒线程kswapd

每当(`mm/page_alloc.c`)里的分配器，尝试对空闲页面低于`WMARK_LOW`的区域进行分配内存时，就会调用`wakeup_kswapd()`：

```c
/* mm/vmscan.c */
void wakeup_kswapd(struct zone *zone, gfp_t gfp_flags, int order,
                   enum zone_type highest_zoneidx)
{
    pg_data_t *pgdat = zone->zone_pgdat;   /* the zone's owning node */
    /* ... */
    if (READ_ONCE(pgdat->kswapd_order) < order)
        WRITE_ONCE(pgdat->kswapd_order, order);
    /* ... */
    wake_up_interruptible(&pgdat->kswapd_wait);
}
```

只有**该区node的kswapd**被唤醒。Node-1的短缺不会自动唤醒Node-0的kswapd。

### balance_pgdat会做什么？

一旦唤醒，内核kswapd线程会调用`balance_pgdat()`:

```c
/* mm/vmscan.c */
static int balance_pgdat(pg_data_t *pgdat, int order, int highest_zoneidx)
```

`balance_pgdat()` 会根据优先顺序循环node上的所有zones， 然后调用`shrink_node()` 直到所有被管理的zones处于平衡状态。`pgdat_balanced()`可以用来检测是否"Balanced":

```c
/* mm/vmscan.c */
static bool pgdat_balanced(pg_data_t *pgdat, int order, int highest_zoneidx)
{
    /* ... */
    for_each_managed_zone_pgdat(zone, pgdat, i, highest_zoneidx) {
        if (sysctl_numa_balancing_mode & NUMA_BALANCING_MEMORY_TIERING)
            mark = promo_wmark_pages(zone);
        else
            mark = high_wmark_pages(zone);
        /* check free pages >= mark ... */
    }
}
```

kswapd 会回收直到其**自身节点**上的每个区域都超过`WMARK_HIGH`。它没有义务从远程节点回收，也不会查看远程节点的watermark。

### kswapd vs 直接回收

| | kswapd (background) | Direct reclaim |
|---|---|---|
| Triggered by | free pages < `WMARK_LOW` | free pages < `WMARK_MIN` |
| Who reclaims | Dedicated kernel thread | The allocating process itself |
| Process latency | None (async) | Process blocks |
| Entry point | `balance_pgdat()` | `try_to_free_pages()` → `do_try_to_free_pages()` |
| Node scope | Own node's LRU | Follows the allocation's `zonelist` |

!!!注意
    `pg_data_t`中的`kswapd_failures`意思时说，有多少次没有回收到任何page。在`MAX_RECLAIM_RETRIES`次失败后，`kswapd_test_hopeless()`返回true，节点被视为"hopeless"。kswapd会回退，让直接回收处理该节点。上场的顺序还有些需要注意的地方。
---

## NUMA均衡以及和回收的交互

### NUMA怎样暗示/‘影响’faults的工作

当`CONFIG_NUMA_BALANCING`开启时，内核会定期扫描任务的VMA，使用`task_numa_work()`（在`kernel/sched/fair.c`中），在想要评估的页面上安装PROT_NONE PTE。下一次访问此类页面时，会被标记为page-fault，达到`mm/memory.c`中的`do_numa_page()`。

`do_numa_page()` 调用`numa_migrate_check()`，决定错误页面是否应迁移到本地节点。如果需要迁移，则通过`migrate_misplaced_folio_prepare()`和`migrate_misplaced_folio()`进行：

```c
/* mm/memory.c */
static vm_fault_t do_numa_page(struct vm_fault *vmf)
{
    /* ... */
    target_nid = numa_migrate_check(folio, vmf, vmf->address, &flags,
                                     writable, &last_cpupid);
    if (target_nid == NUMA_NO_NODE)
        goto out_map;               /* stay put */

    if (migrate_misplaced_folio_prepare(folio, vma, target_nid)) {
        flags |= TNF_MIGRATE_FAIL;
        goto out_map;
    }
    if (!migrate_misplaced_folio(folio, target_nid)) {
        nid = target_nid;
        flags |= TNF_MIGRATED;      /* moved to local node */
        task_numa_fault(last_cpupid, nid, nr_pages, flags);
        return 0;
    }
    flags |= TNF_MIGRATE_FAIL;
    /* ... */
}
```

同样的模式也适用于透明的巨页，处理方式为`mm/huge_memory.c`。

### 晋升（promotion）路径与回收路径

当访问远程页面时：

```
Remote page accessed on Node 0 by CPU on Node 0
               |
               V
         NUMA hint fault fires
         (do_numa_pagev called)
               |
   +-----------+------------
   |                       |
   V                       V
Migration succeeds     Migration fails
(page moves to         (page stays on
 Node 0 — promotion)    Node 1 — remote)
               |                   |
               V                   V
     Page now on Node 0's    Page stays on Node 1's
     LRU — ages normally      LRU — aged and potentially
                               reclaimed by kswapd1
```
无法迁移的页面（例如Node-0内存不足，或页面被共享且迁移被阻止）会留在Node-1的LRU上。Node-1的kswapd最终会将其重新标记为冷页，无论处理page-fault的进程是否仍在Node-0上活跃使用。

这就是核心紧张的地方：**回收路径和晋升路径相互竞争**。为了缓解Node-1的压力，kswapd1正要重新回收的page，可能正是NUMA平衡准备提升到Node-0以获得更好局域化的页面。（冲突了）

### WMARK_PROMO和内存层级

当`sysctl_numa_balancing_mode` 被设置为 `NUMA_BALANCING_MEMORY_TIERING` (used for CXL/persistent memory tiers), `pgdat_balanced()` 使用 `promo_wmark_pages(zone)` 而不是 `high_wmark_pages(zone)` 作为它目标。
这为更快的层（tier）保留了更多空间，这样从较慢的内存中晋升就能顺利进行，而不会立即触发回收。

---

## zone_reclaim_mode

### 是什么

`vm.zone_reclaim_mode` （内核变量：`node_reclaim_mode`）是一个sysctl配置，当非零时，页面分配器会尝试在**之前**尝试本地回收，然后再回退到远端节点的内存。它旨在用于那些NUMA本地性比强制回收缓存溢出成本更重要的工作负载。

（bits）比特位被定义在`include/uapi/linux/mempolicy.h`:

```c
#define RECLAIM_ZONE    (1<<0)  /* enable zone/node reclaim */
#define RECLAIM_WRITE   (1<<1)  /* writeback dirty pages during reclaim */
#define RECLAIM_UNMAP   (1<<2)  /* unmap mapped pages during reclaim */
```

`node_reclaim_enabled()` 任何位被标记，都会返回true：

```c
/* mm/internal.h */
static inline bool node_reclaim_enabled(void)
{
    return node_reclaim_mode & (RECLAIM_ZONE|RECLAIM_WRITE|RECLAIM_UNMAP);
}
```

当被启用且分配未达到其首选区域的watermark时，`mm/page_alloc.c`中的`get_page_from_freelist()`会在本地节点调用`node_reclaim()`，然后再切换到远程区域：

```c
/* mm/page_alloc.c */
if (!node_reclaim_enabled() ||
    !zone_allows_reclaim(zonelist_zone(ac->preferred_zoneref), zone))
    continue;

ret = node_reclaim(zone->zone_pgdat, gfp_mask, order);
```

`zone_allows_reclaim()`  强制执行 NUMA 距离限制：只有当目标区域的节点距离在，首选区域节点`node_reclaim_distance`范围内时，才会尝试回收（默认情况下：`include/linux/topology.h` 中的`RECLAIM_DISTANCE = 30`）。这防止了分配者为避免实际靠近的远程节点而进行昂贵的本地回收。

### 为什么它会导致延迟问题

在远程分配前强制本地回收意味着：

- 有用的热页会从本地内存中被驱逐，以保留一个空的watermark缓冲区。
- 跨节点工作集或跨节点共享内存的工作负载，由于zone_reclaim从未查看远程节点的空闲空间，因此出现不成比例的驱逐现象。（相当于是说没有看到全局的情况，直接做回收）
- 写回路径（`RECLAIM_WRITE`）会为分配路径增加I/O延迟，反之通过远程DRAM会快速完成任务。

对于通用工作负载——包括大多数数据库和内存缓存——热缓存页的丢失超过了避免远程访问NUMA惩罚带来的任何好处。

### 当前的状态

`zone_reclaim_mode` 是 **默认关掉** (默认值是0)。内核文档(`Documentation/admin-guide/sysctl/vm.rst`) 明确说明：

> *zone_reclaim_mode 默认被关闭。对于需要缓存的数据服务器或工作负载，应关闭zone_reclaim_mode，因为缓存效果可能比数据局部性更重要。*

当每个进程的工作集完全集中在单一节点内，且远程访问延迟是主要性能问题时，比如，高度分区的HPC或NUMA支持的实时工作负载，这时候就i应该关闭这个参数。

!!!警告
    在共享内存工作负载或工作集大于一个NUMA节点的工作负载上启用`zone_reclaim_mode`通常会影响吞吐量。自己需要针对自己的具体工作负载进行跑benchmark测试。

---

## 远程内存压力导致本地回收

考虑一个双节点系统，Node-1的分配正在耗尽本地内存：

```
Node 0: 20 GB free (comfortably above WMARK_HIGH)
Node 1:  1 GB free (below WMARK_LOW, kswapd1 running hard)
```

在`zone_reclaim_mode = 0`默认值的情况下，运行在Node-1上的进程会发生什么？

1. Node-1 的页分配器依次遍历Node-1 的各个内存域（zone），发现它们的内存水位均低于阈值，于是去检查**备用域链表（zonelist）**—— 该链表包含Node-0 的内存域，作为内存分配的后备来源。
2. 分配器**不调用`node_reclaim()`**，因为`node_reclaim_enabled()`为假。
3. 分配从Node-0立即成功。进程获得远程内存但不会停滞。
4. 与此同时，kswapd1 在后台继续从Node-1 的 LRU 中回收，试图将Node-1 推回`WMARK_HIGH`以上。

**关键是：kswapd1 无论Node-0 的闲置内存有多少，都会从 Node-1 的 LRU 中回收。** 它没有机制“借用” Node-0 的存储余量。结果是，Node-1 可以从Node-1 回收活跃使用的页面，而Node-0大部分处于空闲状态。

这不是bug——这是通用工为作负载的正确行。另一种做法（因为远程内存空闲而停止回收）会无限泄漏远程节点的内存，最终导致远程内存也耗尽。

!!!注意
    Node-1 上的进程**不会在Node-1内存压力升高时自动迁移至Node-0**。进程放置属于调度器范畴，而非内存回收范畴。可使用`numactl --cpunodebind`或`taskset`显式控制进程放置，或依赖调度器的负载均衡机制（该机制会考虑 NUMA 拓扑，但不保证将进程移出存在内存压力的节点）。

---

## 观察node级的回收统计数据

### /proc/vmstat

`/proc/vmstat` 输出系统级计数器。与内存回收最相关的有：

```bash
grep -E 'pgsteal|pgscan|kswapd' /proc/vmstat
```

关键的字段(命名在文件`mm/vmstat.c`):

| 计数 | 含义 |
|---|---|
| `pgsteal_kswapd` | Pages reclaimed by kswapd (all nodes combined) |
| `pgsteal_direct` | Pages reclaimed by direct reclaim |
| `pgscan_kswapd` | Pages scanned by kswapd |
| `pgscan_direct` | Pages scanned by direct reclaim |
| `pgscan_direct_throttle` | Times direct reclaim was throttled |
| `kswapd_low_wmark_hit_quickly` | Times kswapd restored the low watermark without a full scan |
| `kswapd_high_wmark_hit_quickly` | Times kswapd reached the high watermark quickly |
| `zone_reclaim_success` | `node_reclaim()` calls that reclaimed enough pages |
| `zone_reclaim_failed` | `node_reclaim()` calls that did not reclaim enough |

只有在`zone_reclaim_mode != 0`，的情况下`zone_reclaim_success` 和`zone_reclaim_failed`才不为0。

### Node级的vmstat

每个节点通过 sysfs 暴露自身的 vmstat 计数器：

```bash
# Per-node free pages
cat /sys/devices/system/node/node0/vmstat
cat /sys/devices/system/node/node1/vmstat

# Example fields visible here:
# nr_inactive_anon, nr_active_anon, nr_inactive_file, nr_active_file
# pgsteal_kswapd, pgscan_kswapd, pgsteal_direct, pgscan_direct
```

对比各节点间的`pgsteal_kswapd`可以看出哪个节点承担内存回收压力。数值严重不均衡通常代表内存放置存在问题。

### NUMA分配统计数据

```bash
numastat          # per-node hit/miss counters (reads /sys/devices/system/node/nodeN/numastat)
numastat -p <pid> # per-process NUMA mapping
```

相关的node级的计数在`/sys/devices/system/node/nodeN/numastat`:

| sysfs name | `/proc/vmstat` name | Meaning |
|---|---|---|
| `numa_hit` | `numa_hit` | Allocations that landed on the intended node |
| `numa_miss` | `numa_miss` | Allocations that landed on a different node |
| `numa_foreign` | `numa_foreign` | Allocations intended for this node that landed elsewhere |
| `local_node` | `numa_local` | Allocations by a CPU local to this node |
| `other_node` | `numa_other` | Allocations by a CPU on a remote node |
| `interleave_hit` | `numa_interleave` | Interleave policy landed on the intended node |

节点 1 的`numa_miss`持续升高，同时节点 1 上 kswapd 正在执行内存回收，这是典型信号：节点 1 内存压力大，正在降级使用远端内存分配。

### NUMA平衡计数

```bash
grep numa /proc/vmstat
```

| Counter | Meaning |
|---|---|
| `numa_hint_faults` | PROT_NONE fault fires (NUMA hint page scanned and accessed) |
| `numa_hint_faults_local` | Hint faults where the page was already local |
| `numa_pages_migrated` | Pages successfully promoted/migrated via NUMA balancing |

较低的`numa_pages_migrated`与`numa_hint_faults`比值意味着大部分提示缺页未能完成迁移，通常是因为目标节点内存已满。

---

## 微调

### 设置MPOL_INTERLEAVE来减小per-node hotspots

当一份大型共享内存数据集（例如数据库缓冲池、共享队列）全部分配在单个节点上时，该节点一旦出现内存压力就会回收该数据集的部分内容。内存交错分配会将内存分配行为以及内存压力分散到所有节点：

```c
/* Application code */
#include <numaif.h>

unsigned long nodemask = 0x3;  /* nodes 0 and 1 */
mbind(addr, length, MPOL_INTERLEAVE, &nodemask, 2, 0);
```

```bash
# At process start
numactl --interleave=all ./my_application
```
其取舍在于：单次内存访问有时会变为远端访问，拉高平均延迟。当数据集过大无法容纳在单个节点，或是多个 NUMA 本地进程都需要访问同一份数据时，这种方式通常是值得的。

### Transparent huge pages and NUMA

透明大页分配总是优先尝试本地节点。若本地节点无法满足 2MB 连续内存分配，但远端节点可以，内核会降级为远端分配或是基础页分配 —— 不会将透明大页请求拆分到多个节点。本地内存压力下，kswapd 的内存整理机制（`kcompactd`）会运行以创建连续空闲内存区域，但该内存整理同样是以节点为单位执行。

For applications that mix NUMA pinning with THP:

- `MADV_HUGEPAGE` on a region that spans both nodes will attempt THPs on whichever node each VMA range maps to.
- `numactl --membind=0 --huge-pages` restricts THP allocation to Node 0; if Node 0 lacks contiguous memory, the allocation falls back to base pages or blocks.

!!!小技巧
    如果某个节点上的 kswapd 持续比其他节点更活跃，而整机仍存在空闲内存，首先需要排查是否有单个进程或共享段将内存集中分配在该节点。`numastat -p <pid>`可以展示进程的虚拟内存映射在各节点上的分布情况。

### 调整node_reclaim_distance

在AMD EPYC或类似多芯片平台上，2跳的NUMA距离报告为32（高于默认`RECLAIM_DISTANCE` 30），内核可能拒绝在槽位内结构中进行本地回收，尽管这些访问相对较快。距离阈值可以调整： `mm/page_alloc.c`中的`node_reclaim_distance`变量默认为 RECLAIM_DISTANCE （30）。它不会作为 sysctl 暴露给用户——更改它需要修改内核源代码或使用自定义内核模块。要绕过这个问题，可以配置 `vm.zone_reclaim_mode = 0` 以完全禁用本地回收，或调整 NUMA 内存策略，使其优先选择这些平台上的远程分配。

---

## 关键源文件

| File | What it contains |
|---|---|
| `mm/vmscan.c` | `kswapd()`, `kswapd_run()`, `balance_pgdat()`, `pgdat_balanced()`, `wakeup_kswapd()`, `node_reclaim()`, `node_reclaim_mode` variable |
| `mm/page_alloc.c` | `get_page_from_freelist()`, `zone_watermark_ok()`, `__zone_watermark_ok()`, `zone_allows_reclaim()`, `node_reclaim_distance` |
| `include/linux/mmzone.h` | `struct zone` (watermark fields), `struct pglist_data` (kswapd fields), `enum zone_watermarks`, watermark accessor inlines |
| `mm/migrate.c` | `migrate_misplaced_folio()`, `migrate_misplaced_folio_prepare()`, `migrate_pages()` |
| `mm/memory.c` | `do_numa_page()`, `numa_migrate_check()` |
| `mm/internal.h` | `node_reclaim_enabled()` |
| `include/uapi/linux/mempolicy.h` | `RECLAIM_ZONE`, `RECLAIM_WRITE`, `RECLAIM_UNMAP`, `MPOL_INTERLEAVE` |
| `include/linux/topology.h` | `RECLAIM_DISTANCE` default (30), `node_reclaim_distance` declaration |
| `include/linux/vm_event_item.h` | `PGSTEAL_KSWAPD`, `PGSCAN_KSWAPD`, `NUMA_HINT_FAULTS`, `NUMA_PAGE_MIGRATE` enum values |
| `mm/vmstat.c` | Counter string names (`pgsteal_kswapd`, `numa_hit`, `numa_miss`, `zone_reclaim_success`, etc.) |
| `kernel/sched/fair.c` | `task_numa_work()` — NUMA hint PTE scanner |

## Further reading

### Kernel source

- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — `kswapd()`, `kswapd_run()`, `balance_pgdat()`, `pgdat_balanced()`, `wakeup_kswapd()`, `node_reclaim()`, `node_reclaim_mode`
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `get_page_from_freelist()`, `zone_allows_reclaim()`, `node_reclaim_distance`, and fallback zonelist traversal
- [mm/memory.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memory.c) — `do_numa_page()` and `numa_migrate_check()`: the NUMA hint fault path that drives page migration
- [mm/migrate.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/migrate.c) — `migrate_misplaced_folio()` and `migrate_misplaced_folio_prepare()`

### Kernel documentation

- `Documentation/admin-guide/sysctl/vm.rst` — documents `zone_reclaim_mode`, `min_free_kbytes`, and related reclaim knobs ([rendered](https://docs.kernel.org/admin-guide/sysctl/vm.html))

### LWN articles

- [Per-node reclaim and kswapd](https://lwn.net/Articles/461294/) — analysis of per-node kswapd wakeup logic and the interaction between nodes under memory pressure
- [NUMA balancing and page migration](https://lwn.net/Articles/568870/) — how hint faults, migration decisions, and the promotion path work in automatic NUMA balancing
- [Zone reclaim mode considered harmful](https://lwn.net/Articles/432224/) — the case for disabling `zone_reclaim_mode` on general-purpose workloads

### Related docs

- [NUMA Memory Management](numa.md) — NUMA overview, memory policies, automatic balancing, and monitoring
- [Zone Reclaim Policy](zone-reclaim.md) — `vm.zone_reclaim_mode`, watermark boosting, and DMA zone pressure in detail
- [NUMA Zonelist Construction and Fallback Ordering](numa-zonelist.md) — how fallback to remote nodes is ordered and how `MPOL_BIND` prevents it
- [Reclaim](reclaim.md) — the full reclaim machinery: LRU lists, `shrink_node()`, and direct reclaim
