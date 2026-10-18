# Zone 回收策略

> 内核如何按 zone 回收内存、`vm.zone_reclaim_mode` 的作用，以及为什么默认关闭它

## 关键源文件

| 文件 | 相关内容 |
|------|-----------|
| `mm/vmscan.c` | `node_reclaim()`、`balance_pgdat()`、`node_reclaim_mode` |
| `mm/page_alloc.c` | `get_page_from_freelist()`、`zone_watermark_fast()`、`zone_allows_reclaim()`、`boost_watermark()` |
| `include/linux/mmzone.h` | `struct zone`、`enum zone_watermarks`、`enum zone_type`、水位线访问函数 |
| `include/uapi/linux/mempolicy.h` | `RECLAIM_ZONE`、`RECLAIM_WRITE`、`RECLAIM_UNMAP` 位定义 |
| `include/linux/topology.h` | `RECLAIM_DISTANCE` 默认值 |

---

## 内存 zone

Linux 将物理内存划分为不同的 zone，以便满足具有不同寻址限制的分配请求，避免浪费。zone 类型定义在 `include/linux/mmzone.h` 的 `enum zone_type` 中：

```c
enum zone_type {
#ifdef CONFIG_ZONE_DMA
    ZONE_DMA,       /* first ~16MB: ISA/legacy DMA devices */
#endif
#ifdef CONFIG_ZONE_DMA32
    ZONE_DMA32,     /* first 4GB: 32-bit DMA devices */
#endif
    ZONE_NORMAL,    /* remaining directly-mapped physical memory */
#ifdef CONFIG_HIGHMEM
    ZONE_HIGHMEM,   /* above ZONE_NORMAL on 32-bit arches */
#endif
    ZONE_MOVABLE,   /* movable pages only; aids hotplug/THP */
#ifdef CONFIG_ZONE_DEVICE
    ZONE_DEVICE,    /* persistent/device memory */
#endif
    __MAX_NR_ZONES
};
```

各 zone 构成一个层级。需要 `ZONE_DMA` 内存的分配器只能使用 `ZONE_DMA`。请求 `ZONE_NORMAL` 内存的分配器在必要时可以回退到 `ZONE_DMA32` 或 `ZONE_DMA`，反向则不行。每个 zone 都彼此独立：它有自己的空闲链表、水位线和回收压力。

```
物理地址空间（典型的 64 位 x86 系统）：

0         16MB          4GB                          RAM 顶部
|─ZONE_DMA─|───ZONE_DMA32───|────────ZONE_NORMAL────────────|
  GFP_DMA     GFP_DMA32          GFP_KERNEL（默认）
```

---

## Zone 水位线

每个 zone 都会跟踪三个水位线，用来控制何时开始回收。这些值保存在 `struct zone._watermark[]` 中，并通过内联辅助函数访问——不要直接读取该数组。

### `enum zone_watermarks` 枚举

```c
/* include/linux/mmzone.h */
enum zone_watermarks {
    WMARK_MIN,
    WMARK_LOW,
    WMARK_HIGH,
    WMARK_PROMO,   /* used with memory tiering / NUMA balancing */
    NR_WMARK
};
```

三个主要级别如下：

| 水位线 | 含义 |
|-----------|---------|
| `WMARK_MIN` | 最后防线。直接回收会阻塞分配进程，直到空闲页面数升至此级别以上。 |
| `WMARK_LOW` | 后台阈值。低于此级别时会唤醒 `kswapd`。 |
| `WMARK_HIGH` | 目标值。`kswapd` 持续回收，直到空闲页面数达到此级别，然后进入睡眠。 |

### 水位线访问宏

`struct zone` 在 `_watermark[]` 中保存原始水位线值。读取时会叠加 `watermark_boost` 字段，以实现临时提升（参见[水位线提升](#watermark-boosting)）。始终使用以下访问函数：

```c
/* include/linux/mmzone.h */

struct zone {
    unsigned long _watermark[NR_WMARK];
    unsigned long watermark_boost;
    /* ... */
};

static inline unsigned long wmark_pages(const struct zone *z,
                                        enum zone_watermarks w)
{
    return z->_watermark[w] + z->watermark_boost;
}

static inline unsigned long min_wmark_pages(const struct zone *z)  { return wmark_pages(z, WMARK_MIN); }
static inline unsigned long low_wmark_pages(const struct zone *z)  { return wmark_pages(z, WMARK_LOW); }
static inline unsigned long high_wmark_pages(const struct zone *z) { return wmark_pages(z, WMARK_HIGH); }
```

水位线值根据 `vm.min_free_kbytes` 计算。调高该 sysctl 会按比例提高所有 zone 中的这三个级别。

### 检查水位线：`zone_watermark_fast()` 与 `zone_watermark_ok()`

分配器在热路径中调用 `zone_watermark_fast()`；若需要较慢的完整检查，则回退到 `zone_watermark_ok()`。这两个函数都位于 `mm/page_alloc.c`：

```c
bool zone_watermark_ok(struct zone *z, unsigned int order,
                       unsigned long mark,
                       int highest_zoneidx, unsigned int alloc_flags);

static inline bool zone_watermark_fast(struct zone *z, unsigned int order,
                                       unsigned long mark, int highest_zoneidx,
                                       unsigned int alloc_flags, gfp_t gfp_mask);
```

`zone_watermark_fast()` 会尝试快速路径：如果空闲页面数超过 `mark + lowmem_reserve[highest_zoneidx]`，且不考虑 order-N 分配需求，就立即返回 `true`。否则会调用 `__zone_watermark_ok()`，完整统计所有空闲链表阶数上的页面。

---

## 按 zone 运行的 kswapd：`balance_pgdat()`

`kswapd` 是每个节点（每个 `pgdat`）对应一个的内核线程，但它按 zone 逐个回收。核心循环是 `mm/vmscan.c` 中的 `balance_pgdat()`：

```c
/*
 * kswapd scans the zones in the highmem->normal->dma direction. It skips
 * zones which have free_pages > high_wmark_pages(zone), but once a zone is
 * found to have free_pages <= high_wmark_pages(zone), any page in that zone
 * or lower is eligible for reclaim until at least one usable zone is
 * balanced.
 */
static int balance_pgdat(pg_data_t *pgdat, int order, int highest_zoneidx)
```

扫描方向很重要：kswapd **从最高 zone 索引向下扫描**（例如 `ZONE_MOVABLE` → `ZONE_NORMAL` → `ZONE_DMA32` → `ZONE_DMA`）。一旦发现某个 zone 低于 `WMARK_HIGH`，该 zone 及所有更低的 zone 都会在同一轮中成为回收候选对象。kswapd 会继续工作，直到至少一个可满足触发分配请求的 zone 恢复平衡，空闲页面数重新高于 `WMARK_HIGH`。

```
kswapd 的 zone 扫描顺序（从高到低）：

  ZONE_MOVABLE  ← 首先扫描
  ZONE_NORMAL
  ZONE_DMA32
  ZONE_DMA      ← 最后扫描

如果 ZONE_NORMAL 低于 WMARK_HIGH，则同一轮中会从
ZONE_NORMAL、ZONE_DMA32 和 ZONE_DMA 回收页面。
```

`kswapd` 也会处理水位线提升：每次调用 `balance_pgdat()` 时，它都会在开始阶段将各 zone 的 `zone->watermark_boost` 保存到局部数组 `zone_boosts[]` 中。每次 `shrink_node` 迭代后，它都会按快照中的值减少 `zone->watermark_boost`，随着回收取得进展逐渐消除提升值。

---

## `node_reclaim()` 与 `vm.zone_reclaim_mode`

### sysctl 参数

`vm.zone_reclaim_mode` 是内核变量 `node_reclaim_mode`（在 `mm/vmscan.c` 中声明）对用户可见的名称：

```c
/* mm/vmscan.c */
int node_reclaim_mode __read_mostly;  /* default: 0 (disabled) */
```

它的各个位定义在 `include/uapi/linux/mempolicy.h` 中：

```c
#define RECLAIM_ZONE  (1<<0)  /* Enable zone reclaim */
#define RECLAIM_WRITE (1<<1)  /* Writeout pages during reclaim */
#define RECLAIM_UNMAP (1<<2)  /* Unmap pages during reclaim */
```

这些位可以组合使用：

| 值 | 效果 |
|-------|--------|
| `0` | 禁用 zone 回收（默认） |
| `1` | 从本地节点回收干净的文件页 |
| `2` | + 允许在回收期间回写脏页 |
| `4` | + 允许取消文件页的映射 |
| `7` | 启用所有回收类型 |

只要设置了任意一位，`node_reclaim_enabled()` 就会返回 true：

```c
/* mm/internal.h */
static inline bool node_reclaim_enabled(void)
{
    return node_reclaim_mode & (RECLAIM_ZONE|RECLAIM_WRITE|RECLAIM_UNMAP);
}
```

### `get_page_from_freelist()` 中的调用位置

该钩子位于 `mm/page_alloc.c` 的 `get_page_from_freelist()` 中，这是主要的分配路径。某个 zone 未通过水位线检查后，分配器会判断是否值得尝试节点回收：

```c
/* mm/page_alloc.c — inside get_page_from_freelist() */
if (!node_reclaim_enabled() ||
    !zone_allows_reclaim(zonelist_zone(ac->preferred_zoneref), zone))
    continue;

ret = node_reclaim(zone->zone_pgdat, gfp_mask, order);
switch (ret) {
case NODE_RECLAIM_NOSCAN:
    /* did not scan */
    continue;
case NODE_RECLAIM_FULL:
    /* scanned but unreclaimable */
    continue;
default:
    /* did we reclaim enough? */
    if (zone_watermark_ok(zone, order, mark,
            ac->highest_zoneidx, alloc_flags))
        goto try_this_zone;
    continue;
}
```

如果回收成功且该 zone 现在高于其水位线，分配就会从该 zone 继续进行。否则，分配器会转到 zonelist 中的下一个 zone。

### `node_reclaim()` 内部

`mm/vmscan.c` 中的 `node_reclaim()` 是同步的逐节点回收路径。它由分配器直接调用，并会阻塞正在分配内存的进程：

```c
int node_reclaim(struct pglist_data *pgdat, gfp_t gfp_mask, unsigned int order)
{
    const unsigned long nr_pages = 1 << order;
    struct scan_control sc = {
        .nr_to_reclaim = max(nr_pages, SWAP_CLUSTER_MAX),
        .gfp_mask      = current_gfp_context(gfp_mask),
        .order         = order,
        .priority      = NODE_RECLAIM_PRIORITY,  /* = 4: scan 1/16 of zone */
        .may_writepage = !!(node_reclaim_mode & RECLAIM_WRITE),
        .may_unmap     = !!(node_reclaim_mode & RECLAIM_UNMAP),
        .may_swap      = 1,
        .reclaim_idx   = gfp_zone(gfp_mask),
    };
    /* ... */
}
```

开始扫描前，它会检查两个条件；满足其中任一条件时都会提前结束调用：

1. **回收阈值**：如果 `node_pagecache_reclaimable(pgdat) <= pgdat->min_unmapped_pages` *并且* 可回收 slab 低于 `pgdat->min_slab_pages`，则立即返回 `NODE_RECLAIM_FULL`（可回收内存太少，不值得尝试）。
2. **CPU 局部性**：如果该节点关联了 CPU，且它不是调用任务所在 CPU 的节点，则返回 `NODE_RECLAIM_NOSCAN`。这可确保 node_reclaim 只在本地节点或没有 CPU 的节点（仅有内存的节点）上执行。

成功时会增加 `PGSCAN_ZONE_RECLAIM_SUCCESS`；失败时会增加 `PGSCAN_ZONE_RECLAIM_FAILED`。这两个计数器在 `/proc/vmstat` 中分别显示为 `zone_reclaim_success` 和 `zone_reclaim_failed`。

---

## 为什么 `zone_reclaim_mode` 默认关闭

当 `zone_reclaim_mode=1` 时，内核会优先从本地内存回收页面，然后才回退到远程 NUMA 节点分配。这样做原本是为了保持 NUMA 局部性，但实际使用中会造成一个显著问题：

**`node_reclaim()` 会让分配进程停滞。** 因为它在分配热路径中同步运行，进程会在整个扫描和回收周期内阻塞。在存在明显内存压力的系统上，这会导致应用延迟突增。通过 zonelist 回退从远程节点分配，通常比等待本地回收快得多。

具体取舍如下：

| 场景 | 建议行为 |
|----------|--------------------|
| 访问远程节点带来的延迟开销不到 2 倍 | `zone_reclaim_mode=0`——从远程节点分配 |
| 应用严格要求 NUMA 局部性，且工作集较大但能放入单个节点 | `zone_reclaim_mode=1` 可能有帮助 |
| 流式或 I/O 密集型负载 | `zone_reclaim_mode=0`——页缓存不断周转会使本地回收适得其反 |
| 每个节点固定有大型缓冲池的数据库 | `zone_reclaim_mode=1` 可以减少跨节点流量 |

`mm/vmscan.c` 中的内核注释对此有明确说明：

> 如果该值非零，则在空闲页面数低于水位线时调用 node_reclaim。

“如果该值非零”也暗示了将其设为“零”是更安全的默认选择。几乎所有现代服务器负载都会将它保持为 `0`。

---

## `RECLAIM_DISTANCE` 与 `zone_allows_reclaim()`

即使 `zone_reclaim_mode` 非零，内核也不会对拓扑上靠近请求节点的 zone 调用 `node_reclaim()`。阈值为 `RECLAIM_DISTANCE`，定义在 `include/linux/topology.h` 中：

```c
#ifndef RECLAIM_DISTANCE
/*
 * If the distance between nodes in a system is larger than RECLAIM_DISTANCE
 * (in whatever arch specific measurement units returned by node_distance())
 * and node_reclaim_mode is enabled then the VM will only call node_reclaim()
 * on nodes within this distance.
 */
#define RECLAIM_DISTANCE 30
#endif

extern int __read_mostly node_reclaim_distance;
```

运行时变量 `node_reclaim_distance` 的初始值为 `RECLAIM_DISTANCE`（30），平台代码可以覆盖该值。例如，AMD EPYC 机器会提高此值，因为在权衡回收开销后，访问相距两跳的远程节点内存（ACPI SLIT 距离为 32）仍能提升性能。

检查函数是 `mm/page_alloc.c` 中的 `zone_allows_reclaim()`：

```c
#ifdef CONFIG_NUMA
int __read_mostly node_reclaim_distance = RECLAIM_DISTANCE;

static bool zone_allows_reclaim(struct zone *local_zone, struct zone *zone)
{
    return node_distance(zone_to_nid(local_zone), zone_to_nid(zone)) <=
                node_reclaim_distance;
}
```

ACPI SLIT 距离值：`LOCAL_DISTANCE=10`（同一节点），`REMOTE_DISTANCE=20`（一跳）。由于 `zone_allows_reclaim()` 检查的是距离是否小于等于阈值，值为 `30` 时允许在本地及一跳远的节点上回收；距离大于 `30` 的 zone 会跳过回收，改为直接从远程节点分配。

```
node_distance(local, candidate) <= node_reclaim_distance?
  是 → 可以运行 node_reclaim()
  否 → 跳过回收，直接从远程 zone 分配
```

---

## DMA zone 压力

与系统总内存相比，`ZONE_DMA`（最初约 16 MB）和 `ZONE_DMA32`（最初 4 GB）都很小。它们会被设备专用分配（`GFP_DMA`、`GFP_DMA32`）占满，而且不容易回收。

### DMA zone 为什么会耗尽

- 设备驱动会使用 `GFP_DMA` 或 `GFP_DMA32` 分配 bounce buffer、描述符环和固件区域。
- 这些分配通常会长期存在（持续到设备生命周期结束）。
- 内核不会自动将 DMA 分配迁移到其他 zone。

### 回退与 OOM 行为

- 如果 `ZONE_NORMAL` 耗尽，`GFP_KERNEL` 分配（普通内存）可以回退到 `ZONE_DMA32` 或 `ZONE_DMA`，但反向不成立。
- 如果 `ZONE_DMA` 中没有空闲页面，`GFP_DMA` 分配会触发 OOM killer，即使 `ZONE_NORMAL` 还有数 GB 空闲内存也是如此，因为设备无法访问 `ZONE_NORMAL` 内存。

```
GFP_KERNEL 分配路径（zonelist 回退顺序）：
  ZONE_NORMAL → ZONE_DMA32 → ZONE_DMA
  （只能向低 zone 回退，不能向高 zone 回退）

GFP_DMA 分配路径：
  仅限 ZONE_DMA——耗尽时触发 OOM
```

### 检测 DMA zone 压力

```bash
# 检查各 zone 的空闲页面数与水位线
grep -A 10 "zone" /proc/zoneinfo | grep -E "Node|zone|free|min|low|high"

# 健康 DMA zone 的示例：
# Node 0, zone      DMA
#   pages free     2652
#         min      33
#         low      41
#         high     49
```

异常迹象：`DMA` 或 `DMA32` 中的 `free` 页面数接近 `min`，并且 `dmesg` 中记录了 `DMA-zone allocation failure` 分配失败。

---

## 水位线提升 {#watermark-boosting}

### 要解决的问题

高阶分配（例如 THP）回退到较低的 migratetype（例如 `MIGRATE_UNMOVABLE` 从 `MIGRATE_MOVABLE` 中借用页面）时，会污染低阶空闲链表。如果不采取措施，后续的高阶分配会持续失败；而 kswapd 也不会被唤醒，因为尽管 zone 内部已经碎片化，其空闲页面数可能仍高于常规水位线。

### 工作原理

当 `try_to_claim_block()` 检测到低于 `pageblock_order` 的阶数发生 migratetype 回退时，会调用 `mm/page_alloc.c` 中的 `boost_watermark()`。该函数会临时提高 `zone->watermark_boost`：

```c
static inline bool boost_watermark(struct zone *zone)
{
    unsigned long max_boost;

    if (!watermark_boost_factor)
        return false;
    /* skip tiny zones where boosting would cause OOM */
    if ((pageblock_nr_pages * 4) > zone_managed_pages(zone))
        return false;

    max_boost = mult_frac(zone->_watermark[WMARK_HIGH],
                          watermark_boost_factor, 10000);
    if (!max_boost)
        return false;

    max_boost = max(pageblock_nr_pages, max_boost);

    zone->watermark_boost = min(zone->watermark_boost + pageblock_nr_pages,
                                max_boost);
    return true;
}
```

`watermark_boost_factor` 的默认值为 `15000`（即 `WMARK_HIGH` 的 150%）。每次调用 `wmark_pages()` 时都会加上该提升值，使 zone 看起来比实际更缺内存——即使 zone 名义上仍高于 `WMARK_HIGH`，也会因此触发 kswapd。

提升水位线后，会设置 zone 标志 `ZONE_BOOSTED_WATERMARK`。下次经过 `rmqueue()` 的返回路径时，会检查该标志并唤醒 kswapd：

```c
/* mm/page_alloc.c */
if ((alloc_flags & ALLOC_KSWAPD) &&
    unlikely(test_bit(ZONE_BOOSTED_WATERMARK, &zone->flags))) {
    clear_bit(ZONE_BOOSTED_WATERMARK, &zone->flags);
    wakeup_kswapd(zone, 0, 0, zone_idx(zone));
}
```

随后，`balance_pgdat()` 会逐步消除提升值：每次迭代都会从 `zone->watermark_boost` 中减去初始快照 `zone_boosts[i]`，直到提升值归零且 zone 真正恢复平衡。

### 控制水位线提升行为

```bash
# 默认值：15000（WMARK_HIGH 的 150%）
cat /proc/sys/vm/watermark_boost_factor

# 完全禁用水位线提升
echo 0 > /proc/sys/vm/watermark_boost_factor
```

如果系统频繁发生 THP 回退，且由此引起的 kswapd 唤醒造成了不必要的回收压力，可以将其设为 `0`。

---

## 观察 zone 状态

### `/proc/zoneinfo`

这是诊断各 zone 状态的权威信息源。关键字段如下：

```
Node 0, zone   Normal
  pages free     524288
        boost    0         ← 当前 watermark_boost
        min      8192
        low      10240
        high     12288
        spanned  2097152
        present  2097152
        managed  1998765
```

检查是否有 zone 的 `free` 低于 `low` 或接近 `min`。

### `/proc/vmstat`——zone 回收计数器

只有在 `CONFIG_NUMA=y` 时才会提供这些计数器：

```bash
grep zone_reclaim /proc/vmstat
# zone_reclaim_success  42      ← node_reclaim() 成功释放了足够的页面
# zone_reclaim_failed   1831    ← node_reclaim() 已运行，但未能满足分配
```

启用 `zone_reclaim_mode` 时，如果 `zone_reclaim_failed` 计数很高，就强烈表明该负载的热页面无法在本地回收，应禁用 `zone_reclaim_mode`。

### 快速诊断流程

```bash
# 1. 检查 zone 水位线和空闲页面数
grep -A 15 "^Node" /proc/zoneinfo | grep -E "Node|zone|free|boost|min|low|high"

# 2. 检查是否启用了 zone 回收
cat /proc/sys/vm/zone_reclaim_mode

# 3. 检查回收成功/失败比率（仅 NUMA）
grep -E "zone_reclaim|pgscan" /proc/vmstat

# 4. 检查 DMA zone 是否耗尽
grep -A 10 "zone.*DMA" /proc/zoneinfo | grep free

# 5. 检查是否有水位线提升引起的 kswapd 活动
grep -E "pageoutrun|kswapd" /proc/vmstat
```

### 调优建议

| 现象 | 可能原因 | 处理措施 |
|---------|-------------|--------|
| `zone_reclaim_failed` >> `zone_reclaim_success` | 工作集无法放入本地节点 | `echo 0 > /proc/sys/vm/zone_reclaim_mode` |
| NUMA 系统出现延迟突增 | `zone_reclaim_mode` 使分配器停滞 | 禁用 zone 回收模式 |
| 其他 zone 仍有空闲内存却发生 OOM | DMA zone 已耗尽 | 减少使用 `GFP_DMA`/`GFP_DMA32` 的分配；检查驱动的内存用量 |
| 内存压力不大但 kswapd 仍在运行 | 水位线提升处于活动状态 | 检查 `/proc/zoneinfo` 中的 `boost` 字段；考虑降低 `watermark_boost_factor` |
| 高阶分配持续失败 | Zone 碎片化 | 检查 `/proc/buddyinfo`；考虑调整 `vm.compaction_proactiveness` |

## 延伸阅读

### 内核源代码

- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — `node_reclaim()`、`balance_pgdat()`、`kswapd()` 和 `node_reclaim_mode` 变量
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `get_page_from_freelist()`、`zone_watermark_fast()`、`zone_watermark_ok()`、`zone_allows_reclaim()`、`boost_watermark()`
- [include/linux/mmzone.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) — `struct zone` 水位线字段、`enum zone_watermarks`、`enum zone_type` 和水位线访问内联函数
- [include/uapi/linux/mempolicy.h](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/uapi/linux/mempolicy.h) — `RECLAIM_ZONE`、`RECLAIM_WRITE`、`RECLAIM_UNMAP` 位定义

### 内核文档

- `Documentation/admin-guide/sysctl/vm.rst` — `zone_reclaim_mode`、`min_free_kbytes`、`watermark_boost_factor` 和 `watermark_scale_factor`（[在线版本](https://docs.kernel.org/admin-guide/sysctl/vm.html)）
- `/proc/sys/vm/zone_reclaim_mode` — 运行时开关；`0` 表示禁用，`1`/`2`/`4` 表示逐步启用更积极的回收

### LWN 文章

- [Zone reclaim mode considered harmful](https://lwn.net/Articles/432224/) — 分析 `zone_reclaim_mode=1` 为什么会降低大多数负载的吞吐量，以及默认关闭的原因
- [Watermark boosting](https://lwn.net/Articles/750500/) — 介绍 `watermark_boost_factor` 的设计动机，以及它如何缓解碎片化导致的 kswapd 活动不足
- [DMA zone management](https://lwn.net/Articles/258803/) — 介绍 DMA zone 为什么较小、如何耗尽，以及阻止跨 zone 边界回退的限制

### 相关文档

- [NUMA Effects on Memory Reclaim](numa-reclaim.md) — 每节点的 kswapd 线程、NUMA 平衡与回收的交互，以及 NUMA 场景下的 `zone_reclaim_mode`
- [NUMA Zonelist Construction and Fallback Ordering](numa-zonelist.md) — 分配器的回退列表如何决定调用 `node_reclaim()` 之前先尝试哪个 zone
- [Reclaim](reclaim.md) — 完整的回收机制：LRU 链表、`shrink_node()`、直接回收和 `try_to_free_pages()`
- [Compaction](compaction.md) — `kcompactd` 和主动内存紧缩如何处理高阶分配失败；水位线提升旨在暴露这类失败
