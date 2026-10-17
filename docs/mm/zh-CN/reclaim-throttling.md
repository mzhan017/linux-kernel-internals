# 回收节流与平衡

> 当内存回收跟不上时，内核如何减慢内存分配器

内存压力较高时，必须避免两种情况：单个分配内存的进程不能为了回收内存而耗尽 CPU；分配器持续高速运行时，也不能让回收过程不堪重负。内核通过节流解决这两个问题：暂时暂停分配进程，让回收取得进展，并协调 kswapd 与直接回收进程来平衡负载。

## 关键源文件

| 文件 | 相关函数 |
|------|--------------------|
| [`mm/vmscan.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) | `reclaim_throttle()`——核心节流睡眠；`throttle_direct_reclaim()`——pfmemalloc 储备节流；`consider_reclaim_throttle()`——NOPROGRESS 路径；`balance_pgdat()`——kswapd 节点平衡；`shrink_node()`——逐节点回收；`allow_direct_reclaim()`——唤醒条件；`pgdat_balanced()`——水位线检查；`writeback_throttling_sane()`——脏页节流可用性 |
| [`mm/memcontrol.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) | `__mem_cgroup_handle_over_high()`——执行 memory.high 限制；`reclaim_high()`——cgroup 回收循环；`calculate_high_delay()`——按比例惩罚；`mem_find_max_overage()` / `swap_find_max_overage()`——扫描超限祖先 |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `__alloc_pages_slowpath()`——慢速分配路径；`__alloc_pages_direct_reclaim()` / `__perform_reclaim()`——直接回收入口；`try_to_free_pages()` 调用位置 |
| [`include/linux/mmzone.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/mmzone.h) | `enum vmscan_throttle_state`、`enum pgdat_flags`、`enum lruvec_flags`、`reclaim_wait[]` 等待队列、`pfmemalloc_wait` 等待队列 |

---

## 为什么需要节流

如果没有节流，快速分配内存的应用可能会不断进入直接回收，耗尽整个 CPU 核心来尝试释放页面。其他进程会因此得不到足够资源，实际工作也会停滞。更隐蔽的问题是，回收本身可能陷入无法恢复的状态：如果找到的页面都正在回写，更积极地扫描只会浪费 CPU。正确的做法是暂停并等待。

节流有两个目标：

1. **公平性**——避免单个进程独占回收路径。
2. **反压**——当回收跟不上时，减慢分配器，而不是让它们在无效工作上空转。

内核会区分多种节流原因，每种原因都有自己的等待队列和超时时间：

```
enum vmscan_throttle_state {
    VMSCAN_THROTTLE_WRITEBACK,   /* pages cycling through LRU faster than writeback */
    VMSCAN_THROTTLE_ISOLATED,    /* too many parallel reclaimers isolating pages */
    VMSCAN_THROTTLE_NOPROGRESS,  /* reclaim making no headway at high priority */
    VMSCAN_THROTTLE_CONGESTED,   /* dirty pages backed by a congested device */
    NR_VMSCAN_THROTTLE,
};
```

`pg_data_t` 结构（每个 NUMA 节点一个）为每种节流原因保存一个等待队列，并为直接回收的 pfmemalloc 节流单独保存一个队列：

```c
/* include/linux/mmzone.h */
wait_queue_head_t reclaim_wait[NR_VMSCAN_THROTTLE];
wait_queue_head_t pfmemalloc_wait;
```

---

## 直接回收与 kswapd

内核中有两种回收执行者：

```
分配路径                            后台
────────                            ────
alloc_pages()                       kswapd (每个节点一个内核线程)
    │                                   │
    └─ __alloc_pages_slowpath()         └─ balance_pgdat()
           │                                    │
           └─ __alloc_pages_direct_reclaim()    └─ kswapd_shrink_node()
                  │                                     │
                  └─ __perform_reclaim()                └─ shrink_node()
                         │
                         └─ try_to_free_pages()
                                │
                                └─ shrink_zones()
                                       │
                                       └─ shrink_node()
```

**kswapd** 在后台运行并回收页面，直到各 zone 的空闲页面数高于 `high` 水位线。它不会阻塞任何分配进程。

当 `__alloc_pages_slowpath()` 无法利用现有空闲页面满足分配请求时，**直接回收**会在分配进程自身的上下文中运行。调用 `alloc_pages()` 的进程会进入 `try_to_free_pages()`，并可能在那里被节流。

!!! note "优先级递进"
    两条路径使用的回收优先级都从 `DEF_PRIORITY`（12）开始，逐渐降至 0。在优先级 *p* 下，每个 LRU 链表会扫描 `list_size >> p` 个页面；优先级越低，扫描的内存比例越大，直到优先级为 0 时检查整个链表。

---

## `reclaim_throttle()`——核心睡眠函数

`mm/vmscan.c` 中的 `reclaim_throttle()` 是所有非 pfmemalloc 节流路径共同经过的函数：

```c
void reclaim_throttle(pg_data_t *pgdat, enum vmscan_throttle_state reason)
```

它从 `pgdat->reclaim_wait[reason]` 中选择相应的等待队列，并根据节流原因以 `TASK_UNINTERRUPTIBLE` 状态让调用任务睡眠一段时间：

| 节流原因 | 超时时间 | 触发条件 |
|-----------------|---------|----------------|
| `VMSCAN_THROTTLE_WRITEBACK` | `HZ/10`（100 毫秒） | 页面在 LRU 中循环的速度超过回写速度（`sc->nr.immediate` 非零） |
| `VMSCAN_THROTTLE_ISOLATED` | `HZ/50`（20 毫秒） | 太多并行回收进程同时从 LRU 隔离页面 |
| `VMSCAN_THROTTLE_NOPROGRESS` | 1 个 jiffy | 直接回收处于优先级 1，且没有回收到页面 |
| `VMSCAN_THROTTLE_CONGESTED` | 1 个 jiffy | 所有脏页都由拥塞设备支持（设置了 `LRUVEC_CGROUP_CONGESTED` 或 `LRUVEC_NODE_CONGESTED`） |

!!! info "内核线程不受节流"
    对于设置了 `PF_USER_WORKER` 或 `PF_KTHREAD` 的任务（kswapd 自身除外），`reclaim_throttle()` 会跳过节流。日志线程、回写工作线程和其他内核线程必须能够继续运行，因为回收依赖它们取得进展。

对于 `VMSCAN_THROTTLE_WRITEBACK`，代码还会跟踪节流开始后完成回写的页面数（`pgdat->nr_reclaim_start`），这样在足够多的页面完成清理后，`__acct_reclaim_writeback()` 就可以提前唤醒受节流的任务，而不必总是等待整个超时时间。

**谁会唤醒受节流的任务？** 当 `allow_direct_reclaim()` 返回 true 时，kswapd 中的 `balance_pgdat()` 会唤醒 `pfmemalloc_wait` 队列。当回收效率超过 12%（扫描与回收页面数之比 `> 1/8`）时，`consider_reclaim_throttle()` 会唤醒 `reclaim_wait[]` 队列；kswapd 清除拥塞状态时也会唤醒这些队列。

---

## `throttle_direct_reclaim()`——pfmemalloc 储备节流

这是用途不同的另一条节流路径。内核会为通过网络进行交换所需的网络协议栈分配，预留一部分低 zone 页面（`pfmemalloc` 储备）。如果这些储备耗尽，继续进行直接回收会适得其反；此时应让 kswapd 不受阻碍地处理回收。

```c
static bool throttle_direct_reclaim(gfp_t gfp_mask,
                                    struct zonelist *zonelist,
                                    nodemask_t *nodemask)
```

该函数会在 `try_to_free_pages()` 的开头、任何回收工作开始之前调用。满足以下任一条件时，它会立即返回（不进行节流）：

- 调用任务设置了 `PF_KTHREAD`（内核线程）。
- 有致命信号待处理（让进程退出并自然释放其内存）。
- `allow_direct_reclaim()` 返回 true——从最低 zone 到 `ZONE_NORMAL`（含）所有 zone 的空闲页面总数，超过这些 zone 中 pfmemalloc 储备总量的一半。

实施节流时，进程会在 `pgdat->pfmemalloc_wait` 上睡眠：

```c
/* Caller cannot enter filesystem — might be holding a journal lock */
if (!(gfp_mask & __GFP_FS))
    wait_event_interruptible_timeout(pgdat->pfmemalloc_wait,
                                     allow_direct_reclaim(pgdat), HZ);
else
    /* Block until kswapd makes progress */
    wait_event_killable(pgdat->pfmemalloc_wait,
                        allow_direct_reclaim(pgdat));
```

每次发生节流都会增加 vmstat 计数器 `PGSCAN_DIRECT_THROTTLE`（在 `/proc/vmstat` 中显示为 `pgscan_direct_throttle`）。

`allow_direct_reclaim()` 会遍历从最低 zone 到 `ZONE_NORMAL` 的所有 zone，分别累计空闲页面数和 pfmemalloc 储备，然后检查 `free_pages > pfmemalloc_reserve / 2` 是否成立。如果节点处于无望状态（`kswapd_test_hopeless()`），则始终允许直接回收。

!!! warning "致命信号处理"
    如果 `throttle_direct_reclaim()` 返回 `true`，说明任务在睡眠期间收到了致命信号。此时 `try_to_free_pages()` 会返回 1，页面分配器会跳过 OOM killer——该进程已经处于退出过程中。

---

## `consider_reclaim_throttle()`——无进展节流

直接回收每次执行完 `shrink_zones()` 后，`consider_reclaim_throttle()` 都会判断是否需要节流：

```c
static void consider_reclaim_throttle(pg_data_t *pgdat,
                                      struct scan_control *sc)
```

- 如果回收效率高于 12.5%（`sc->nr_reclaimed > sc->nr_scanned >> 3`），则唤醒已在 `VMSCAN_THROTTLE_NOPROGRESS` 上睡眠的任务。
- 如果优先级为 1（再降低一级就会扫描整个 LRU），且 `sc->nr_reclaimed` 仍为零，则调用 `reclaim_throttle(pgdat, VMSCAN_THROTTLE_NOPROGRESS)`。
- kswapd 和 cgroup 回收不受此节流机制约束——它们在 I/O 停滞时改用 `VMSCAN_THROTTLE_WRITEBACK`。

---

## 内存 cgroup 节流——`memory.high` 路径

### 按比例延迟

cgroup v2 的 `memory.high` 是软限制。超过该限制不会立即杀死 cgroup 或将其强制停止，而是根据该 cgroup（或其任一祖先）超出限制的程度，按比例减慢超限进程的运行速度。

当一次内存分配导致 cgroup 超过 `memory.high` 时，记账路径会设置 `current->memcg_nr_pages_over_high`。返回用户空间时（或者在计数超过 `MEMCG_CHARGE_BATCH` 时提前），会调用 `__mem_cgroup_handle_over_high()`：

```
分配量超过 memory.high
           │
           v
    memcg_nr_pages_over_high += batch
           │
           v (返回用户空间时，或计数 > MEMCG_CHARGE_BATCH 时)
    __mem_cgroup_handle_over_high()
           │
           ├── reclaim_high()              ← try_to_free_mem_cgroup_pages()
           │       沿 cgroup 树向上遍历
           │
           ├── calculate_high_delay()      ← 以 jiffies 计的指数惩罚
           │
           └── schedule_timeout_killable(penalty_jiffies)
```

`reclaim_high()` 会沿 cgroup 层级向上遍历，并从每个超过自身 `memory.high` 限制的祖先回收内存。最多尝试回收 `MAX_RECLAIM_RETRIES`（16）次后，如果用量仍然超过限制，`calculate_high_delay()` 就会计算睡眠时长。

### 延迟计算公式

惩罚延迟由 `calculate_high_delay()` 计算：

```c
penalty_jiffies = max_overage * max_overage * HZ;
penalty_jiffies >>= MEMCG_DELAY_PRECISION_SHIFT;   /* 20 bits */
penalty_jiffies >>= MEMCG_DELAY_SCALING_SHIFT;     /* 14 bits */

/* Scale by nr_pages relative to MEMCG_CHARGE_BATCH */
return penalty_jiffies * nr_pages / MEMCG_CHARGE_BATCH;
```

`max_overage` 是一个定点数比率，表示该 cgroup（或超限最严重的祖先）超过 `memory.high` 的程度，计算方式如下：

```c
overage = (usage - high) << MEMCG_DELAY_PRECISION_SHIFT;
overage = overage / high;   /* fraction of the limit */
```

由于包含平方项，超限越多，惩罚增长越快：超出限制两倍的 cgroup 会比刚刚超限的 cgroup 受到大得多的惩罚。每次返回用户空间的批次，其最终延迟会限制在 `MEMCG_MAX_HIGH_DELAY_JIFFIES`（2 秒）以内，因此应用不会永久挂起，只会运行得非常缓慢。

!!! note "内存和交换空间分别计算惩罚"
    `calculate_high_delay()` 会调用两次：一次使用 `mem_find_max_overage()` 的结果（内存超限），另一次使用 `swap_find_max_overage()` 的结果（交换空间超限）。两次计算会分别遍历层级中的祖先，最终惩罚为两者之和。

!!! note "轻微超限不会产生延迟"
    如果 `penalty_jiffies <= HZ/100`（10 毫秒），任务不会进入睡眠。这样可以避免惩罚仅轻微超限、且可能已成功回收内存的 cgroup。

---

## 写入节流与回收停滞

### `PGDAT_WRITEBACK` 与回写节流

当 `shrink_node()` 发现检查过的每个页面都处于回写状态（`sc->nr.writeback == sc->nr.taken`）时，就会在该节点上设置 `PGDAT_WRITEBACK` 标志。这会通知 kswapd：脏页回写正在跟上 LRU 扫描的速度，页面在链表中的循环速度快于存储设备写出页面的速度。

如果 `sc->nr.immediate` 不为零，即有页面既处于回写状态，*又*已重新放回非活跃 LRU 链表的末尾，`shrink_node()` 就会调用：

```c
reclaim_throttle(pgdat, VMSCAN_THROTTLE_WRITEBACK);
```

该调用会让任务最多睡眠 100 毫秒；如果有足够多的页面完成回写，`__acct_reclaim_writeback()` 会唤醒等待队列，任务也会提前恢复运行。

### `LRUVEC_NODE_CONGESTED` 与 `VMSCAN_THROTTLE_CONGESTED`

如果扫描到的所有脏页都处于拥塞状态（`sc->nr.dirty == sc->nr.congested`），节点会被标记为拥塞：kswapd 设置 `LRUVEC_NODE_CONGESTED`，cgroup 回收设置 `LRUVEC_CGROUP_CONGESTED`。检测到任一标志的直接回收进程随后会调用：

```c
reclaim_throttle(pgdat, VMSCAN_THROTTLE_CONGESTED);
```

这里不会对 kswapd 本身进行节流——它可以继续扫描，因为让 kswapd 停滞会使所有回收工作都无法推进。只有用户上下文中的直接回收进程会睡眠。

### `balance_dirty_pages()`——上游写入节流机制

`mm/page-writeback.c` 中的写入节流路径，是减慢脏页生成速度超过写回速度的进程的主要机制。文件系统写入路径会调用 `balance_dirty_pages()`，回收路径不会直接调用它。不过两者会相互影响：如果 `balance_dirty_pages()` 不可用（例如旧式 cgroup v1 内存控制器中，回写与 memcg 相互独立），`writeback_throttling_sane()` 就会返回 `false`，而 `shrink_folio_list()` 会退回到直接等待回写 I/O 的方式；这种方式不具备常规脏页节流机制的公平性和按带宽比例调节的能力。

---

## 内存压力下的公平回收

### 回收优先级与 cgroup 层级

多个 cgroup 同时承受内存压力时，内核不会简单地只从内存用量最大的 cgroup 回收。`shrink_node_memcgs()` 会遍历节点 lruvec 树中的所有 memcg，并对每个 memcg 调用 `shrink_lruvec()`。从每个 cgroup 扫描的页面数量与其在该节点可回收页面中的占比成正比，并受其配置的 `swappiness` 影响。

回收优先级从 `DEF_PRIORITY`（12）开始，随着压力增大逐渐递减至 0。在每个优先级下，扫描范围是各 LRU 链表的 `1 >> priority`。优先级越低，扫描的比例越大；优先级为 0 时会扫描整个链表。

### `reclaim_high()` 中的 cgroup 层级遍历

执行 `memory.high` 限制时，`reclaim_high()` 不会只从直接超限的 cgroup 回收。它会通过 `parent_mem_cgroup()` 一直向上遍历到根节点，并从每个同样超过自身 `memory.high` 限制的祖先回收。这样可以避免深层嵌套的 cgroup 仅仅因为直接父级没有超限，就逃避祖先的限制。

```c
do {
    if (page_counter_read(&memcg->memory) > READ_ONCE(memcg->memory.high))
        nr_reclaimed += try_to_free_mem_cgroup_pages(...);
} while ((memcg = parent_mem_cgroup(memcg)) &&
         !mem_cgroup_is_root(memcg));
```

计算惩罚延迟时，`mem_find_max_overage()` 函数也会执行类似的祖先遍历，并选取超限最严重的祖先来确定惩罚程度。

---

## `balance_pgdat()`——kswapd 节点平衡

kswapd 的主循环会调用 `balance_pgdat()` 回收节点上的页面，直到至少有一个 zone 达到平衡。`pgdat_balanced()` 会检查 zone 的空闲页面是否超过 `high` 水位线；满足条件时，该 zone 即为平衡：

```c
static bool pgdat_balanced(pg_data_t *pgdat, int order, int highest_zoneidx)
```

它会从最低 zone 到最高 zone 依次检查；只要有一个 zone 在 `high_wmark_pages()` 阈值下满足 `__zone_watermark_ok()`，就会立即返回 `true`。

当 `pgdat_balanced()` 返回 true 后，kswapd 会调用 `prepare_kswapd_sleep()`，该函数会：

1. 唤醒所有在 `pgdat->pfmemalloc_wait` 上睡眠的进程（它们正在等待补充 pfmemalloc 储备）。
2. 调用 `clear_pgdat_congested()` 清除 `PGDAT_WRITEBACK`、`LRUVEC_NODE_CONGESTED` 和 `LRUVEC_CGROUP_CONGESTED`。

每当 `allow_direct_reclaim()` 变为 true 时，`balance_pgdat()` 也会在循环过程中唤醒 `pfmemalloc_wait`，因此受节流的分配进程不必一直等到 kswapd 完全平衡该节点才恢复运行。

---

## 观察节流情况

### PSI——最直接的信号

压力停滞信息（PSI）准确记录了节流所针对的现象：任务因等待内存而停滞。完整说明请参阅 [PSI](psi.md)。回收节流期间，在 `reclaim_throttle()` 或 `throttle_direct_reclaim()` 中等待的任务会由 `psi_memstall_enter()` / `psi_memstall_leave()` 计入内存停滞。

```bash
# 快照——"full" 表示所有可运行任务都处于停滞状态
cat /proc/pressure/memory
# some avg10=8.42 avg60=3.11 avg300=1.05 total=12450183
# full avg10=2.17 avg60=0.84 avg300=0.27 total=3241885
```

`full` 内存压力持续升高，是节流正在生效且系统回收速度跟不上的可靠信号。

### `/proc/vmstat` 计数器

```bash
grep -E 'pgscan|pgsteal|allocstall|throttle' /proc/vmstat
```

| 计数器 | 含义 |
|---------|---------|
| `pgscan_kswapd` | kswapd 扫描的页面数 |
| `pgsteal_kswapd` | kswapd 回收的页面数 |
| `pgscan_direct` | 直接回收期间扫描的页面数 |
| `pgsteal_direct` | 直接回收期间回收的页面数 |
| `pgscan_direct_throttle` | `throttle_direct_reclaim()` 阻塞进程的次数 |
| `allocstall_*` | 各 zone 类别进入 `try_to_free_pages()` 的次数 |

**如何解读这些比率：**

- 健康的系统中，`pgscan_kswapd` 远高于 `pgscan_direct`。这表示 kswapd 在后台处理内存压力，直接回收很少启动。
- 如果 `pgscan_direct` 相对于 `pgscan_kswapd` 较高，说明 kswapd 已经跟不上，分配进程正在自行回收。这表明系统正承受持续的内存压力。
- `pgscan_direct_throttle` 持续增加，表示 pfmemalloc 储备反复耗尽——系统可能已接近 kswapd 能够处理的极限。
- `allocstall_*` 计数较高，表示 `try_to_free_pages()` 被频繁调用。每次增加都对应一个进程进入直接回收（这本身不是节流事件，而是进入回收的次数）。

!!! tip "关注 steal/scan 比率"
    `pgsteal / pgscan` 表示回收效率。比率低于约 50% 表示回收扫描了许多无法释放的页面，通常是因为页面仍被引用、页面被固定，或写入压力过大。低效率会触发 `VMSCAN_THROTTLE_NOPROGRESS`。

---

## 调优

### `vm.swappiness`

该参数控制内核为回收匿名页（由交换空间支持）和文件页分配的相对代价。对应的内部变量是 `vm_swappiness`，默认值为 60。

- **0**——强烈优先回收文件缓存；只有别无选择时才交换匿名页（注意：自 Linux 3.5 起，这不会完全禁用交换空间）。
- **60**——默认平衡值；匿名页和文件页都可回收，但略微优先回收文件页。
- **100**——将匿名页和文件页视为回收代价相同。
- **200**——即使文件缓存仍可回收，也会积极交换匿名页（Linux 5.8 引入，用于主动回收场景）。

较低的 `swappiness` 会减少交换操作产生的 I/O，但代价是丢弃更多文件缓存。如果系统交换空间充足但磁盘较慢，且文件写入占主导，降低该值可能会增加 `VMSCAN_THROTTLE_WRITEBACK` 事件。

```bash
# 读取当前值
sysctl vm.swappiness

# 临时设置
sysctl -w vm.swappiness=10

# 设置为重启后仍然生效
echo "vm.swappiness = 10" >> /etc/sysctl.d/99-memory.conf
```

### `vm.vfs_cache_pressure`

该参数控制内核相对于页缓存回收 dentry 和 inode（VFS slab 缓存）的积极程度。`fs/dcache.c` 中的内部变量为 `sysctl_vfs_cache_pressure`，默认值是 100。

- **< 100**——内核回收 slab 缓存的积极程度低于页缓存。元数据密集型负载（大量小文件）可从较低的值中受益。
- **100**——默认值；slab 缓存和页缓存的权重相同。
- **> 100**——更积极地回收 slab 缓存。这样可以减少 dentry/inode 占用的内存，但元数据被回收后，VFS 查找开销会增加。

```bash
sysctl vm.vfs_cache_pressure
sysctl -w vm.vfs_cache_pressure=50
```

### `vm.dirty_ratio` 与 `vm.dirty_background_ratio`

这两个参数控制主要写入节流机制 `balance_dirty_pages()` 何时启动。它们会直接影响回收过程中遇到回写页面的情况，从而影响 `VMSCAN_THROTTLE_WRITEBACK` 的触发频率。

| sysctl | 内部变量 | 默认值 | 含义 |
|--------|------------------|---------|---------|
| `vm.dirty_background_ratio` | `dirty_background_ratio` | 10% | 脏页可用内存达到此比例时，`flusher` 线程开始后台回写 |
| `vm.dirty_ratio` | `vm_dirty_ratio` | 20% | 达到此比例时，`balance_dirty_pages()` 开始对写入进程进行节流 |

- 降低 `dirty_background_ratio` 会让后台回写更早启动，使脏页池保持较小，从而减少回写导致的回收停滞。
- 降低 `dirty_ratio` 会更早对写入进程进行节流，但应用的写入延迟会增加。
- 在内存容量很大的系统（数百 GB）上，按百分比计算的比例可能会形成极大的脏页池。可以考虑改用 `vm.dirty_bytes` 和 `vm.dirty_background_bytes` 设置绝对上限。

```bash
# 在 256 GB 系统上限制脏数据量
sysctl -w vm.dirty_background_bytes=1073741824   # 1 GB
sysctl -w vm.dirty_bytes=4294967296              # 4 GB
```

---

## 综合示例：压力逐步升级的过程

```
1. 分配需求上升；kswapd 被唤醒并运行 balance_pgdat()。
   └── pgscan_kswapd 增加；系统状态正常。

2. kswapd 开始落后；空闲页面降至 min 水位线以下。
   └── __alloc_pages_slowpath() → __alloc_pages_direct_reclaim()
   └── 进入 try_to_free_pages()；allocstall_* 增加。

3. 直接回收遇到正在回写的页面。
   └── sc->nr.immediate > 0
   └── reclaim_throttle(VMSCAN_THROTTLE_WRITEBACK)——睡眠 100 毫秒。
   └── 开始由 PSI 统计内存停滞。

4. 回收在优先级 1 下没有取得进展。
   └── consider_reclaim_throttle() → VMSCAN_THROTTLE_NOPROGRESS.

5. pfmemalloc 储备耗尽。
   └── throttle_direct_reclaim() → 在 pfmemalloc_wait 上睡眠。
   └── pgscan_direct_throttle 增加。
   └── kswapd 被唤醒（allow_direct_reclaim → wake_up kswapd_wait）。

6. kswapd 恢复该节点的内存；pgdat_balanced() 返回 true。
   └── wake_up_all(&pgdat->pfmemalloc_wait).
   └── clear_pgdat_congested().
   └── 直接回收进程被唤醒并重试分配。
```

从第 3 步开始，PSI 的 `memory.full` 压力会升高。通过 `/proc/vmstat` 计数器的变化量和 PSI 监控这一过程，可以准确了解系统进入了哪个节流阶段，以及在该阶段停留了多久。

## 延伸阅读

- [mm/vmscan.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/vmscan.c) — `reclaim_throttle()`、`throttle_direct_reclaim()`、`consider_reclaim_throttle()`、`balance_pgdat()`、`allow_direct_reclaim()` 和 `pgdat_balanced()`：完整的节流与 kswapd 平衡实现
- [mm/memcontrol.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/memcontrol.c) — `__mem_cgroup_handle_over_high()`、`reclaim_high()`、`calculate_high_delay()`、`mem_find_max_overage()` 和 `swap_find_max_overage()`：`memory.high` 按比例惩罚路径
- `Documentation/admin-guide/cgroup-v2.rst` — `memory.high` 的语义、超过软限制时的节流行为，以及它与 `memory.max` 的关系
- [LWN：限制内存占用较高的 cgroup](https://lwn.net/Articles/758781/) — `memory.high` 按比例延迟机制的早期设计讨论，说明了二次惩罚公式，以及为何没有采用简单的硬性停止方式
- [reclaim](reclaim.md) — 更全面的内存回收概览：kswapd、直接回收、水位线、LRU 链表和 swappiness
- [memcg](memcg.md) — 从运维角度介绍 `memory.high` 和 `memory.max`：如何设置限制及解读 `memory.events`
