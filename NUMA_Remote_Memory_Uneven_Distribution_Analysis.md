# 程序设置远端 NUMA 内存却不能均匀分配的原因分析

## 问题描述

程序通过 `set_mempolicy` 或 `mbind` 设置使用远端 NUMA 节点的内存，但实际运行发现内存分配并不均匀——部分内存仍然分配在本地节点或分布不符合预期。

---

## 一、首先确认你使用的内存策略

Linux 提供以下 NUMA 内存策略（`mempolicy`），不同策略的行为差异很大：

| 策略 | 行为 | 均匀分配？ |
|------|------|-----------|
| `MPOL_DEFAULT` | 在当前 CPU 所在节点分配 | 否，全在本地 |
| `MPOL_PREFERRED` | 优先指定节点，不足时回退 | 否，只是偏好 |
| `MPOL_BIND` | 严格绑定到指定节点集合 | 否，按 zonelist 顺序 |
| `MPOL_INTERLEAVE` | 按页交替在多节点间轮询分配 | **是**，按页轮询 |
| `MPOL_WEIGHTED_INTERLEAVE` | 按权重交替分配 | 按权重比例 |
| `MPOL_PREFERRED_MANY` | 优先多节点集合，有回退 | 否，只是偏好 |
| `MPOL_LOCAL` | 在当前 CPU 所在节点分配 | 否，全在本地 |

**关键结论**：只有 `MPOL_INTERLEAVE` 和 `MPOL_WEIGHTED_INTERLEAVE` 才能实现多节点间的均匀分配。

---

## 二、导致不均匀分配的六大原因

### 原因 1：使用了 MPOL_BIND 而非 MPOL_INTERLEAVE

这是最常见的误解。`MPOL_BIND` 并不意味着"均匀分配到指定节点"，它的含义是"**严格限制只能从指定节点集合分配，按 zonelist 顺序优先选择**"。

**内核代码路径**：

```c
// mm/mempolicy.c: policy_nodemask()
case MPOL_BIND:
    /* Restrict to nodemask (but not on lower zones) */
    if (apply_policy_zone(pol, gfp_zone(gfp)) &&
        cpuset_nodemask_valid_mems_allowed(&pol->nodes))
        nodemask = &pol->nodes;
    if (pol->home_node != NUMA_NO_NODE)
        *nid = pol->home_node;
    break;
```

`MPOL_BIND` 只设置了 `nodemask` 限制，**不改变 preferred nid**。Buddy allocator 仍然按 zonelist 的顺序（通常是距离最近的节点优先）分配内存。结果是：绑定到 {node1, node2} 时，node1 如果距离当前 CPU 更近，绝大部分内存会先分配到 node1，直到 node1 耗尽才回退到 node2。

**解决方案**：

```c
// 如果需要跨节点均匀分配，用 MPOL_INTERLEAVE
set_mempolicy(MPOL_INTERLEAVE, &remote_nodemask, maxnode);

// 或者用 numactl 命令
numactl --interleave=0,1,2,3 ./your_program
```

### 原因 2：NUMA Balancing 自动将页面迁回本地节点

即使你成功将内存分配到远端节点，**内核的 NUMA Balancing 机制会自动检测到跨节点访问并将页面迁移回本地节点**。

流程如下：

```
1. task_tick_numa() → task_numa_work()
   将 PTE 标记为 PROT_NONE

2. 进程访问远端内存 → do_numa_page()

3. mpol_misplaced() 检查：
   默认策略 preferred_node_policy 带有 MPOL_F_MORON 标志
   → polnid = 当前 CPU 所在节点（thisnid）
   → 发现页面不在 thisnid → 返回目标节点 = thisnid

4. migrate_misplaced_folio() 将页面从远端迁移回本地
```

**关键代码**：

```c
// mm/mempolicy.c: mpol_misplaced()
/* Migrate the folio towards the node whose CPU is referencing it */
if (pol->flags & MPOL_F_MORON) {
    polnid = thisnid;  // 目标设为当前 CPU 的节点
    if (!should_numa_migrate_memory(current, folio, curnid, thiscpu))
        goto out;
}
```

而默认策略恰好带有 `MPOL_F_MORON`：

```c
// mm/mempolicy.c: numa_policy_init()
preferred_node_policy[nid] = (struct mempolicy) {
    .mode = MPOL_PREFERRED,
    .flags = MPOL_F_MOF | MPOL_F_MORON,   // <— 允许 NUMA balancing 迁移
    .nodes = nodemask_of_node(nid),
};
```

**解决方案**：

```bash
# 方案 A：全局关闭 NUMA Balancing
echo 0 > /proc/sys/kernel/numa_balancing

# 方案 B：对特定内存区域使用 MPOL_BIND（不带 MPOL_F_NUMA_BALANCING）
# MPOL_BIND 的 vma_policy_mof() 检查：
# 如果 VMA 有 MPOL_BIND 策略且没设 MPOL_F_MOF 标志，
# change_prot_numa() 的 VMA 会被 task_numa_work() 跳过
```

```c
// mm/mempolicy.c
// MPOL_BIND 默认不带 MPOL_F_MOF 标志（除非用户显式加 MPOL_F_NUMA_BALANCING）
// 这意味着 NUMA Balancing 不会扫描 MPOL_BIND 的 VMA

// kernel/sched/fair.c: task_numa_work()
if (!vma_migratable(vma) || !vma_policy_mof(vma) || ...)
    continue;  // ← 跳过没有 MPOL_F_MOF 标志的 VMA
```

因此 **`mbind` 使用 `MPOL_BIND` 绑定的区域默认不会被 NUMA Balancing 扫描和迁移**，是安全的。但如果你用的是 `set_mempolicy(MPOL_PREFERRED, ...)` 或默认策略 + `numactl --preferred`，NUMA Balancing **会**将页面迁回来。

### 原因 3：THP（透明大页）分配偏向本地节点

内核对 THP（2MB 大页）的分配有特殊优化，即使设置了远端节点的策略，THP 也倾向本地分配：

```c
// mm/mempolicy.c: alloc_pages_mpol()
if (IS_ENABLED(CONFIG_TRANSPARENT_HUGEPAGE) &&
    order == HPAGE_PMD_ORDER && ilx != NO_INTERLEAVE_INDEX) {
    /*
     * For hugepage allocation and non-interleave policy which
     * allows the current node, we only try to allocate from
     * the current/preferred node and don't fall back to other nodes,
     * as the cost of remote accesses would likely offset THP benefits.
     */
    if (pol->mode != MPOL_INTERLEAVE &&
        pol->mode != MPOL_WEIGHTED_INTERLEAVE &&
        (!nodemask || node_isset(nid, *nodemask))) {
        // 先尝试只在本地节点分配 THP
        page = __alloc_frozen_pages_noprof(
            gfp | __GFP_THISNODE | __GFP_NORETRY, order, nid, NULL);
        if (page || !(gfp & __GFP_DIRECT_RECLAIM))
            return page;
    }
}
```

**结论**：除了 `MPOL_INTERLEAVE` / `MPOL_WEIGHTED_INTERLEAVE`，THP 的分配优先在本地节点尝试，只有失败后才考虑远端。

**解决方案**：

```bash
# 禁用 THP
echo never > /sys/kernel/mm/transparent_hugepage/enabled

# 或对特定区域使用 madvise(addr, len, MADV_NOHUGEPAGE)
```

### 原因 4：MPOL_BIND 的 zonelist 回退顺序

即使指定了 `MPOL_BIND` 到远端节点，buddy allocator 的 zonelist 按距离排序：

```c
// mm/page_alloc.c: build_zonelists()
while ((node = find_next_best_node(local_node, &used_mask)) >= 0) {
    if (node_distance(local_node, node) !=
        node_distance(local_node, prev_node))
        node_load[node] += 1;  // 同距离节点间负载均衡
    node_order[nr_nodes++] = node;
    prev_node = node;
}
```

当 `MPOL_BIND` 指定了多个远端节点（如 node 1 和 node 2），分配器仍然按照距离从近到远的 zonelist 顺序分配。如果 node 1 距离更近，所有内存先分配到 node 1 直到其内存不足。

### 原因 5：Kernel 内部分配（slab/kmalloc）不受用户态 mempolicy 控制

页面缓存、内核数据结构（page table, VMA, 文件系统元数据等）的分配走内核路径，**不遵守用户态的 mempolicy**。这些分配默认在当前 CPU 的本地节点进行。

```c
// mm/mempolicy.c: alloc_frozen_pages_noprof()
if (!in_interrupt() && !(gfp & __GFP_THISNODE))
    pol = get_task_policy(current);
// 注意：中断上下文中的分配使用 default_policy
// GFP_KERNEL 等内核分配通常绑定到当前节点
```

如果你的程序有大量文件 I/O，page cache 会占用本地节点内存，导致看起来分配不均匀。

### 原因 6：首次触发（First-Touch）已固定在本地节点

如果内存在设置 mempolicy **之前**已经被触发分配（first-touch），那么它已经在本地节点了。之后设置的 mempolicy 只影响**新分配**的页面。

```c
// 错误示例
char *buf = malloc(1GB);   // 此时还没有物理页面
memset(buf, 0, 1GB);       // First-touch → 分配到本地节点

// 现在才设置策略——太晚了！
set_mempolicy(MPOL_BIND, &remote_nodemask, maxnode);
// 之后 buf 的内存已经在本地节点了
```

**解决方案**：

```c
// 正确做法：先设置策略，再触发分配
set_mempolicy(MPOL_INTERLEAVE, &nodemask, maxnode);
char *buf = malloc(1GB);
memset(buf, 0, 1GB);  // 现在 first-touch 按 interleave 分配

// 或者用 mbind + MPOL_MF_MOVE 迁移已有页面
mbind(buf, 1GB, MPOL_BIND, &remote_nodemask, maxnode,
      MPOL_MF_MOVE | MPOL_MF_STRICT);
```

---

## 三、诊断方法

### 3.1 查看进程的 NUMA 内存分布

```bash
# 查看进程在每个 NUMA 节点上的内存使用
numastat -p <pid>

# 查看 /proc/<pid>/numa_maps（最详细）
cat /proc/<pid>/numa_maps
# 输出示例：
# 7f1234000000 interleave:0-3 anon=1024 dirty=1024 N0=256 N1=256 N2=256 N3=256
# 7f1234400000 bind:1 anon=512 dirty=512 N0=400 N1=112  ← 不均匀！
```

### 3.2 查看 NUMA Balancing 是否在迁移页面

```bash
# 检查 NUMA Balancing 状态
cat /proc/sys/kernel/numa_balancing

# 查看 NUMA 迁移统计
grep -E "numa_" /proc/vmstat
# numa_hit          - 在请求节点成功分配
# numa_miss         - 在请求节点分配失败，回退到其他节点
# numa_foreign      - 其他节点分配了本应在此节点分配的页面
# numa_interleave   - interleave 分配成功命中目标节点
# numa_local        - 在本地节点分配
# numa_other        - 在非本地节点分配
# numa_pte_updates  - NUMA Balancing 标记的 PTE 数
# numa_hint_faults  - NUMA hint fault 次数
# numa_hint_faults_local - 本地 NUMA hint fault
# numa_pages_migrated - NUMA Balancing 迁移的页面数  ← 如果持续增长说明在迁移

# 查看 NUMA balancing 迁移活动的 trace
echo 1 > /sys/kernel/debug/tracing/events/sched/sched_move_numa/enable
echo 1 > /sys/kernel/debug/tracing/events/migrate/mm_migrate_pages/enable
cat /sys/kernel/debug/tracing/trace_pipe
```

### 3.3 确认 mempolicy 是否生效

```bash
# 查看进程的 mempolicy
cat /proc/<pid>/numa_maps | head -20
# 每行开头会显示生效的策略，例如：
# default  → 默认策略
# bind:1   → 绑定到 node 1
# interleave:0-3 → 在 node 0-3 间交替
# prefer:2 → 优先 node 2
```

---

## 四、完整的正确用法示例

### 场景：将内存均匀分配到远端 NUMA 节点

```c
#define _GNU_SOURCE
#include <numaif.h>
#include <numa.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

int main() {
    /* 假设当前 CPU 在 node 0，要均匀使用 node 1 和 node 2 */
    unsigned long nodemask = (1UL << 1) | (1UL << 2);  // node 1 + node 2

    /*
     * 方案 A：MPOL_INTERLEAVE — 按页轮询分配到各节点
     * 这是实现"均匀分配"的正确方式
     */
    if (set_mempolicy(MPOL_INTERLEAVE, &nodemask, sizeof(nodemask) * 8 + 1)) {
        perror("set_mempolicy");
        return 1;
    }

    /* 现在分配并触发的内存会在 node 1 和 node 2 之间均匀交替 */
    size_t size = 1UL << 30;  // 1GB
    char *buf = malloc(size);
    memset(buf, 0, size);  // first-touch 触发实际分配

    /* 分配完成后恢复默认策略 */
    set_mempolicy(MPOL_DEFAULT, NULL, 0);

    /*
     * 方案 B：mbind — 对特定区域设置策略
     * 更精细的控制，且 MPOL_BIND 的 VMA 默认不被 NUMA Balancing 迁移
     */
    char *buf2 = mmap(NULL, size, PROT_READ | PROT_WRITE,
                      MAP_ANONYMOUS | MAP_PRIVATE, -1, 0);

    if (mbind(buf2, size, MPOL_INTERLEAVE, &nodemask,
              sizeof(nodemask) * 8 + 1, 0)) {
        perror("mbind");
    }
    memset(buf2, 0, size);  // first-touch

    /* 验证分布 */
    printf("Check: cat /proc/%d/numa_maps\n", getpid());
    sleep(60);

    return 0;
}
```

### 用 numactl 命令

```bash
# 均匀交替分配到 node 1 和 node 2
numactl --interleave=1,2 ./your_program

# 严格绑定到 node 1（不均匀，但不会回退到本地）
numactl --membind=1 ./your_program

# 优先 node 1（可能被 NUMA Balancing 迁回）
numactl --preferred=1 ./your_program
```

---

## 五、总结决策树

```
你的目标是什么？
│
├─ 均匀分配到多个远端节点
│   └─ 使用 MPOL_INTERLEAVE（numactl --interleave）
│      └─ 不用担心 NUMA Balancing 迁移（INTERLEAVE 有 MPOL_F_MOF 但
│         mpol_misplaced 会正确处理 interleave 策略的 polnid）
│
├─ 全部分配到指定的一个远端节点
│   ├─ mbind(MPOL_BIND, {nodeX}) — 严格绑定，不会被 NUMA Balancing 迁移
│   └─ 注意 THP 可能仍在本地 → 考虑禁用 THP 或用 MADV_NOHUGEPAGE
│
├─ 优先远端节点，允许回退
│   ├─ set_mempolicy(MPOL_PREFERRED, {nodeX})
│   └─ ⚠️ NUMA Balancing 可能将页面迁回本地！
│       └─ 需要关闭 NUMA Balancing 或接受这个行为
│
└─ 保持远端分配不被迁移
    ├─ mbind(MPOL_BIND, ...) — VMA 级策略，NUMA Balancing 跳过
    ├─ 或关闭 NUMA Balancing: echo 0 > /proc/sys/kernel/numa_balancing
    └─ 或使用 mlock() 锁定页面
```
