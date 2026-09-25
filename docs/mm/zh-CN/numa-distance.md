# NUMA距离与CPU Socket间延迟

> 内核怎样量化跨node的内存访问代价，以及为什么不用纳秒来展示这个代价？

## 主要的源文件

| 文件 | 描述 |
|------|-------------|
| [`include/linux/topology.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/include/linux/topology.h) | `LOCAL_DISTANCE`, `REMOTE_DISTANCE`, `RECLAIM_DISTANCE`, 默认宏`node_distance()` |
| [`arch/x86/include/asm/topology.h`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/include/asm/topology.h) | x86重写了: `node_distance(a, b)` → `__node_distance(a, b)` |
| [`mm/numa_memblks.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/numa_memblks.c) | `numa_distance[]` 扁平矩阵, `numa_set_distance()`, `__node_distance()` |
| [`drivers/base/node.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/node.c) | `node_read_distance()` — 每个node关联的属性 sysfs `distance` |
| [`mm/page_alloc.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) | `build_zonelists()`, `find_next_best_node()`, `node_reclaim_distance` |
| [`kernel/sched/fair.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) | `task_numa_migrate()`, `should_numa_migrate_memory()` |
| [`arch/x86/mm/numa.c`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) | x86 NUMA初始化, ACPI/AMD/OF初始化路径 |

---

## NUMA距离的定义

内核将跨NUMA的代价展现为一个**无量纲整数**，而不是用纳秒数。
在文件：`include/linux/topology.h`中，定义了两个常量来设置这个代价量。

```c
/* include/linux/topology.h — conforms to ACPI 2.0 SLIT distance definitions */
#define LOCAL_DISTANCE    10
#define REMOTE_DISTANCE   20
#define DISTANCE_BITS      8   /* distances are u8, range 0–255 */
```

在没有硬件提供数据表的平台上，会使用一般化的备用宏：

```c
/* include/linux/topology.h */
#ifndef node_distance
#define node_distance(from, to) \
    ((from) == (to) ? LOCAL_DISTANCE : REMOTE_DISTANCE)
#endif
```

而在x86平台，这个一般化的宏会被真实的查询函数替代：

```c
/* arch/x86/include/asm/topology.h */
extern int __node_distance(int, int);
#define node_distance(a, b) __node_distance(a, b)
```

`__node_distance()`这个函数根据传入的两个参数得到`u8`数组`numa_distance`里的一个元素, 而这个数组是在系统启动的时候根据固件读取的数据生成的:

```c
/* mm/numa_memblks.c */
int numa_distance_cnt;
static u8 *numa_distance;

int __node_distance(int from, int to)
{
    //如果传入的数据超出了numa_distance_cnt范围，会做特殊处理，以免出现访问异常
    if (from >= numa_distance_cnt || to >= numa_distance_cnt)
        return from == to ? LOCAL_DISTANCE : REMOTE_DISTANCE;
    return numa_distance[from * numa_distance_cnt + to];
}
EXPORT_SYMBOL(__node_distance);
```

### 距离是一个比例关系，而不是实际的延迟纳秒数

`LOCAL_DISTANCE = 10` **并不是**意味着10纳秒。这些值只是比例关系数值：如果距离是20，意思事说“访问次node是访问本地node的两倍”。ACPI SLIT说明文档定义这些数值是相对的性能比例。参见[6.2.15 _SLI (System Locality Information](https://uefi.org/sites/default/files/resources/ACPI_Spec_6_5_Aug29.pdf)。实际一个两槽位Intel系统上，也许本地DRAM的访问延迟是 ~80ns，远端访问DRAM延迟 ~130ns，但是实际从固件读取到的距离值分别是10和20 - 对于内核比较重要的是比例 1:2，而不是绝对的值。 

针对内存分层框架（该框架将硬件性能坐标转换为分层放置策略）所使用的抽象距离模型，see [CXL Memory and Kernel Memory Tiering](cxl-memory-tiering.md).

---

## 从哪里获取这些距离

### ACPI SLIT表

如果一个系统完全支持ACPI，是**System Locality Information Table (SLIT)**提供了这些距离信息。每个`SLIT[i][j]`，给出了从node-i到node-j的相对延迟值。多槽位x86服务器，ARM服务器，还有其他大多数槽位间的拓扑比较复杂的平台都是通过SLIT来提供距离值。
SLIT和SRAT(System Resource Affinity Table)一样是ACPI固件设施的一部分。SLIT映射了CPU和node内存范围的关系。
想要获取到详细的信息，比如内核是怎么解析SRAT和SLIT，参见即将到来的`numa-acpi-srat.md`.

在`arch/x86/mm/numa.c`里的`x86_numa_init()`函数尝试下面一个有顺序的初始化路径，直到初始化成功为止：

```
x86_acpi_numa_init()    ← SRAT + SLIT
amd_numa_init()         ← AMD-specific NUMA (老的平台)
of_numa_init()          ← Device Tree (非ACPI ARM/RISC-V通过DT)
dummy_numa_init()       ← 回落:单一node，所有内存都在node-0
```

### 没有SLIT的系统

如果没有SLIT，或者ACPI被disabled，`numa_alloc_distance()`函数会使用默认值来填充矩阵：

```c
/* mm/numa_memblks.c */
for (i = 0; i < cnt; i++)
    for (j = 0; j < cnt; j++)
        numa_distance[i * cnt + j] = i == j ?
            LOCAL_DISTANCE : REMOTE_DISTANCE;
```

每个node对，会得到一个值：如果是自身node就是10，如果和其他node就是20；这种默认值的设置在2-node的UMA-等价系统是正确的，但是在多node上就不能真实反应multi-hop的拓扑结构。

### 模拟的和VM NUMA

在虚拟化实现里，主机经常暴漏给客户主机一个扁平的拓扑数据，所有的距离都是10，因为客户主机实际并没有硬件间（node间）交互。这就导致（客户主机）内核的距离驱动策略行为上是，看似所有的内存的访问延迟都是一样的。参加[Common pitfalls](#common-pitfalls) 。

---

## 距离矩阵

### 读/sys/devices/system/node/nodeN/distance

The `node_read_distance()` function in `drivers/base/node.c` iterates over all online nodes and emits `node_distance(nid, i)` for each:

内核通过sysfs文件系统来暴漏距离矩阵的一行数据。`drivers/base/node.c`文件里的`node_read_distance()`调用函数`node_distance(nid, i)`：

```bash
# 2-socket system — 2 nodes
$ cat /sys/devices/system/node/node0/distance
10 21
$ cat /sys/devices/system/node/node1/distance
21 10
```
Node-0这一行，到自己的距离是10，到node-1的距离是21。Node-这一行，是Node-0的镜像。而21（非准确的20）这个数字是经常被用到的。固件经常使用21来代表“一跳，比REMOTE_DISTANCE基准稍稍大一”。
`numactl --hardware`这个命令可以显示更易读的数据格式：

```
node distances:
node   0   1
  0:  10  21
  1:  21  10
```

### 4槽位系统例子

在一个4槽位的服务器，node-0和node-1，共享一个NUMA域，node-2和node-3共享另一个，跨域流量需要经过两跳互联链路，可能显示如下的矩阵：

```
node distances:
node   0   1   2   3
  0:  10  11  21  22
  1:  11  10  22  21
  2:  21  22  10  11
  3:  22  21  11  10
```
距离10-11，是本地槽位（在同一个芯片上有一跳，或者是在相邻die之间有一跳）
距离21-22，是远端槽位（跨插槽互联结构为两跳）

怎么读这张表？行是*源*node，列是*目的*node。`[0][2] = 21`代表的意思是，node-0上关联的CPU访问node-2上的内存时的相对成本是2。

### 非对称的距离

SLIT规范里允许非对称的数据，也就是说：`SLIT[i][j]`不一定非要等于`SLIT[j][i]`。这中情况可能发生在特定的NUMA-over-fabric配置下(e.g. 一致性加速器互连（架构/链路）) ，在这种配置下，一个方向上读的代价可能与另一个方向有不同。而Linux在存储和使用这个全矩阵的时候，没有严格要求对称，所以sysfs和`node_distance()`函数调用可以如实的反应固件内部的数据。

### 内部存储

在系统启动时，会比较早的通过`memblock`申请内存，将这个矩阵存放在类型是u8的数组里，数组的索引公式：`numa_distance[from * numa_distance_cnt + to]`。`numa_set_distance()`函数生成了这个矩阵数组，同时如前所述这个函数，会拒绝超出u8类型的数据，并强制对角线的数据值总是`LOCAL_DISTANCE`.

---

## 内核怎么使用这些距离数据

### Zonelist排序(memory allocator fallback)

`mm/page_alloc.c`文件里的`build_zonelists()`函数，构造了**fallback zonelist**。对于这个列表来说，当首选节点无法满足内存分配请求时，分配器尝试使用的内存区域有序序列接着往下找。节点按照与本地节点之间的距离排序:

```c
/* mm/page_alloc.c — build_zonelists() */
while ((node = find_next_best_node(local_node, &used_mask)) >= 0) {
    if (node_distance(local_node, node) !=
        node_distance(local_node, prev_node))
        node_load[node] += 1;    /* new distance tier: reset round-robin */
    node_order[nr_nodes++] = node;
    prev_node = node;
}
build_zonelists_in_node_order(pgdat, node_order, nr_nodes);
```

`find_next_best_node()`利用函数`node_distance()`作为主要的排序键，所以首先从距离10开始选择，接着是距离21，距离30,按照顺序持续找。分配器在没有消耗完近距离的node内存之前不能跨到下一个远距离的node上。

### Node回收的阈值

`RECLAIM_DISTANCE` (默认值是30)用于控制`node_reclaim()`函数何时仅在邻近节点范围内执行内存回收:

```c
/* include/linux/topology.h */
#ifndef RECLAIM_DISTANCE
#define RECLAIM_DISTANCE 30
#endif

extern int __read_mostly node_reclaim_distance;

/* mm/vmscan.c */
int node_reclaim(struct pglist_data *pgdat, gfp_t gfp_mask, unsigned int order)
{
	int ret;
```

如果使能了`node_reclaim_mode`，同时一个候选node的距离超过了`node_reclaim_distance`，这个时候，内核不会尝试去在那个node上收割内存。因为，直接在这个远距离node分配内存，相较于在在高延迟链接的上做“收割再迁移”，代价更低.

AMD EPYC系统覆盖了`node_reclaim_distance`，因为他们的两跳距离（32）仍然比跨node收割性能更好。在内核的这个`topology.h`文件里的注释也有关于这一点的说明。

### 调度器：NUMA域的构造

调度器利用NUMA的距离，创建了`sched_domain`层级。彼此NUMA距离相同的节点构成一个NUMA调度域(`SD_NUMA`)。`kernel/sched/fair.c`文件里的函数`task_numa_migrate()`，每当需要决定是否要做迁移的时候，都会查询当前这个task所属的node与其首选节点之间的NUMA距离，依次距离决定是否要做迁移:

```c
/* kernel/sched/fair.c — task_numa_migrate() */
env.dst_nid = p->numa_preferred_nid;
dist = env.dist = node_distance(env.src_nid, env.dst_nid);
taskweight = task_weight(p, env.src_nid, dist);
...
taskimp = task_weight(p, env.dst_nid, dist) - taskweight;
```

`dist`会传入权重计算过程，这样迁移收益会根据任务迁移的距离做缩放。跨高距离链路的迁移，需要存在更大的负载不均衡才有执行的价值。

### NUMA均衡，迁移决定

`kernel/sched/fair.c`中的`should_numa_migrate_memory()`函数用于判断：远端内存页发生缺页异常时，是否应当触发内存迁移。对于慢速内存层级中的页面，该决策依据访问频率（页面最近发生缺页的时间），而非仅依靠NUMA距离。对于传统NUMA平衡机制，`mm/mempolicy.c` 中内存策略层的`numa_nearest_node()`会调用`node_distance()`，查找距离触发缺页的 CPU 更近的节点；而`mm/memory.c`里的`numa_migrate_check()`负责为迁移做前置检查：

```c
/* mm/mempolicy.c — nearest-node search */
dist = node_distance(node, n);
```

页面会被调度至距离触发缺页的CPU最近的节点，但受内存可用量与cpuset约束限制。

---

## 测量实际的延迟

!!! 警告，距离值不能真实翻译硬件延迟！！
    固件SLIT表项由BIOS/固件开发人员填充。在部分平台上，这些数值仅为近似值、占位值，甚至本身就是错误的。在得出性能相关结论前，务必通过直接延迟测量来核验距离值。

### Intel MLC (Memory Latency Checker) 内存延迟检测器

Intel的MLC是一个工业标准工具，用来测量“带负载”与无负载环境下的NUMA内存的延迟。它使用可配置的访问模式探测每一对节点：

```bash
# Measure idle latency matrix across all node pairs
./mlc --latency_matrix

# Measure loaded latency (with traffic injectors)
./mlc --loaded_latency

# Bandwidth matrix
./mlc --bandwidth_matrix
```

MLC以纳秒为单位输出延迟数据。将远端/本地内存延迟比值与内核的距离比值进行对比，以此评估SLIT是否校准正确。

### numactl + stream

在没有MLC的情况下，如何快速检测带宽：

```bash
# Node 0 CPUs reading from node 1 memory
numactl --cpunodebind=0 --membind=1 ./stream

# Node 0 CPUs reading from local memory
numactl --cpunodebind=0 --membind=0 ./stream
```

带宽比值体现远端访问会使吞吐性能下降多少。在双路服务器系统中，若SLIT上报的距离比值为 2:1，远端访问的带宽性能通常会下降30%~50%，而非刚好 50%。原因在于互连链路带宽往往是非对称的，瓶颈会转移到QPI/UPI链路，而不再是DRAM内存带宽。（QPI/UPI链路，会称为新的瓶颈）

### perf mem

`perf mem`工具可借助硬件性能监控单元，将缓存缺失与DRAM缺失定位到具体NUMA节点:

```bash
# Requires /proc/sys/kernel/perf_event_paranoid <= 1 (or CAP_PERFMON)
perf mem record -a -- sleep 5
perf mem report --sort=mem,sym
```

报告显示每个内存访问样本的源节点（本地/远程），方便你在不修改工作负载的情况下识别局部性较差的热点。是一个实时探测的功能。

### 带有负载与空闲时的延迟

两种测量在不同场景下都很重要：
- **空闲延迟**（无竞争流量）：表示硬件的原始往返时间。用于验证SLIT比率。
- **负载延迟**（带带宽注入器）：表示在真实工作负载条件下的延迟。负载下的互联争用可能显著推动远程延迟，远超空闲测量的延迟。

对于带宽受限的工作负载（数据库、高性能计算），加载延迟和带宽饱和是主要约束条件。对于延迟敏感的工作负载（内存缓存、交易系统），空闲延迟和轻负载下的尾延迟更为重要。每个场景所需的性能不一样，需要区别对待。

---

## 多跳NUMA

对于多个槽位的系统，距离矩阵编码跨槽位结构的**跳数**。一个4槽位的系统以环形或全连接网格排列，最多有两个不同的远距离：

```
距离20-22: 一跳 (directly connected socket pair)
距离 30-32: 两跳 (socket pair connected only through a third)
```

### 识别多跳拓扑

阅读距离矩阵，寻找非对角线元素中超过两个不同的值：

```bash
numactl --hardware | grep -A 10 "node distances"
```
如果你看到三个或以上不同的距离值，系统就有多跳路径。最长的路径表示最坏情况下的远程访问成本。

### 调度器与分配器的影响

'build_zonelists（）' 将节点分组为**距离层级**：所有距离本地节点相同距离的节点组成一层，分配器在进入下一层前，要耗尽这一层。在距离为 {10， 21， 32} 的四槽位的系统中，节点0的退回顺序为：

1. Node 0 (distance 10 — local)
2. Nodes directly connected (distance 21) — round-robin within tier
3. Nodes two hops away (distance 32) — only if tiers 1 and 2 are exhausted

调度器的“SD_NUMA”域同样对第一跳与第二跳不同：近层内的迁移比跨越两跳边界的迁移受到的惩罚更少。

### AMD EPYC多槽位拓扑

AMD EPYC处理器每个槽位暴露多个NUMA节点（每个芯片或每个象限一个）。一个双插槽EPYC系统可能有4或8个NUMA节点，插槽内距离为12，插槽间距离为 32。EPYC系统中BIOS提供的SLIT通常准确——AMD出厂时有经过验证的ACPI表。EPYC的“node_reclaim_distance”覆盖（如上所述）解释了32节点仍足够快，可以超过回收的收益。

---

## 常见陷阱

!!! 警告：“如果所有距离均为10：纳秒距离数值可能错误”
    如果“numactl --hardware”显示每个节点距离为10，说明固件没有提供真实的SLIT，或者SLIT已被全本地值合成。这种情况发生在两种常见场景中：

    1. **虚拟化NUMA**——虚拟机监控器会暴露NUMA节点（例如用于分散vCPU），但所有访客内存都位于主机的同一NUMA域。所有访客距离均为10，因为没有真正的远程成本。

    2. **损坏的BIOS SLIT**——有些固件即使在真实多插槽硬件上也会错误地报告所有距离为10。内核会将所有节点视为等距，不会优先分配本地内存。NUMA工作负载的性能可能不佳，且没有明显原因。

    要检测坏掉的BIOS情况：如果“lscpu”显示多个插槽，而“numactl -H”显示所有距离为10，SLIT几乎肯定是错误的。检查BIOS更新或使用 'acpidump |acpixtract -sLIT' 来检查原始表。

!!! 警告：“VM NUMA拓扑，但没有真实的NUMA效应”
    配置为“numaNodes=2”的虚拟机会有一个距离矩阵，但如果虚拟机管理程序从单一主机NUMA节点分配访客内存，访问任一访客节点的访问都会返回同一个物理内存。内核的NUMA平衡和区域列表决策由访客距离表驱动，会尝试无性能益处的迁移，且可能增加开销。考虑在不映射到真实硬件拓扑的虚拟机中禁用NUMA平衡（'echo 0 > /proc/sys/kernel/numa_balancing'）。

!!! 警告：“距离值为u8 — 最大255”
    “DISTANCE_BITS 8”表示距离以无符号的8位值存储。“numa_set_distance（）”拒绝任何不符合“u8”的值。提供超过255的距离的固件（在深度层级结构拓扑中可能存在）将被夹紧或拒绝，“__node_distance（）”会退回到“REMOTE_DISTANCE”。

!!! 警告：“numactl可能无法显示仅存储节点”
    没有CPU的节点（如CXL内存节点、HMAT通用启动节点）可能不会出现在numactl版本和内核配置的“numactl --hardware”输出中。使用'ls /sys/devices/system/node/'枚举所有节点，包括仅内存节点，并直接读取每个节点的“距离”文件。

## 额外阅读资料

### 内核源代码

- [arch/x86/mm/numa.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/arch/x86/mm/numa.c) — `x86_numa_init()` and the ACPI/AMD/OF/dummy init chain that populates the distance matrix
- [mm/numa_memblks.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/numa_memblks.c) — `__node_distance()`, `numa_set_distance()`, and the `numa_distance[]` flat array storage
- [drivers/base/node.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/drivers/base/node.c) — `node_read_distance()`: the sysfs handler that exposes `/sys/devices/system/node/nodeN/distance`
- [mm/page_alloc.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/mm/page_alloc.c) — `build_zonelists()` and `find_next_best_node()`: how distances drive fallback zonelist ordering
- [kernel/sched/fair.c](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/kernel/sched/fair.c) — `task_numa_migrate()` and `should_numa_migrate_memory()`: scheduler use of `node_distance()`

### LWN文章

- [NUMA distance and scheduler topology](https://lwn.net/Articles/392116/) — how the scheduler groups nodes by distance into scheduling domains
- [Heterogeneous memory management and NUMA](https://lwn.net/Articles/756022/) — how memory tiers with different distance characteristics are handled in modern kernels

### 相关文档

- [NUMA Topology Discovery: ACPI SRAT and SLIT](numa-acpi-srat.md) — how firmware SRAT and SLIT tables are parsed to populate `numa_distance[]`
- [NUMA Zonelist Construction and Fallback Ordering](numa-zonelist.md) — how `node_distance()` drives the page allocator's fallback node order
- [NUMA Effects on Memory Reclaim](numa-reclaim.md) — how `RECLAIM_DISTANCE` uses distance values to gate local reclaim
- [CXL Memory and Kernel Memory Tiering](cxl-memory-tiering.md) — how memory tiers beyond DRAM extend the distance model
