# Linux 内核 NUMA Balancing 流程代码深度解析

## 目录

- [1. 概述](#1-概述)
- [2. 整体架构与数据流](#2-整体架构与数据流)
- [3. 核心数据结构](#3-核心数据结构)
- [4. 阶段一：周期性扫描触发 (task_tick_numa)](#4-阶段一周期性扫描触发-task_tick_numa)
- [5. 阶段二：VMA 扫描与 PTE 标记 (task_numa_work)](#5-阶段二vma-扫描与-pte-标记-task_numa_work)
- [6. 阶段三：NUMA Hint Fault 处理 (do_numa_page)](#6-阶段三numa-hint-fault-处理-do_numa_page)
- [7. 阶段四：NUMA 故障统计与放置决策 (task_numa_fault / task_numa_placement)](#7-阶段四numa-故障统计与放置决策-task_numa_fault--task_numa_placement)
- [8. 阶段五：任务迁移执行 (task_numa_migrate)](#8-阶段五任务迁移执行-task_numa_migrate)
- [9. NUMA Group 机制](#9-numa-group-机制)
- [10. 扫描周期自适应调整](#10-扫描周期自适应调整)
- [11. 内存迁移决策 (should_numa_migrate_memory)](#11-内存迁移决策-should_numa_migrate_memory)
- [12. Memory Tiering 支持](#12-memory-tiering-支持)
- [13. 关键 sysctl 参数](#13-关键-sysctl-参数)
- [14. 完整流程图](#14-完整流程图)

---

## 1. 概述

NUMA Balancing（也称 Automatic NUMA Balancing）是 Linux 内核中的一种自动化 NUMA 优化机制。其核心思想是：**将任务（task）迁移到其频繁访问的内存所在的 NUMA 节点，或将内存页迁移到任务正在运行的 NUMA 节点**，从而减少跨节点内存访问（remote access）带来的延迟开销。

该机制通过 `CONFIG_NUMA_BALANCING` 编译选项控制，运行时通过 `sysctl_numa_balancing_mode` 控制启停：

```c
// include/linux/sched/sysctl.h
#define NUMA_BALANCING_DISABLED         0x0
#define NUMA_BALANCING_NORMAL           0x1
#define NUMA_BALANCING_MEMORY_TIERING   0x2
```

NUMA Balancing 的工作分为两个维度：
- **Task Placement（任务放置）**：将任务迁移到其内存访问最频繁的 NUMA 节点
- **Page Migration（页面迁移）**：将页面迁移到正在访问它的任务所在的 NUMA 节点

---

## 2. 整体架构与数据流

```
                    ┌─────────────────────────────┐
                    │     Scheduler Tick           │
                    │  (task_tick_fair → entity_   │
                    │   tick → task_tick_numa)     │
                    └──────────────┬──────────────┘
                                   │ 检查扫描周期
                                   ▼
                    ┌─────────────────────────────┐
                    │   task_work_add()            │
                    │   (注册 task_numa_work 回调) │
                    └──────────────┬──────────────┘
                                   │ 返回用户态前执行
                                   ▼
                    ┌─────────────────────────────┐
                    │   task_numa_work()           │
                    │   遍历 VMA, 调用             │
                    │   change_prot_numa()         │
                    │   将 PTE 标记为 PROT_NONE    │
                    └──────────────┬──────────────┘
                                   │ 进程再次访问这些页面
                                   ▼
                    ┌─────────────────────────────┐
                    │   Page Fault (NUMA Hint)     │
                    │   handle_pte_fault →         │
                    │   do_numa_page()             │
                    └──────────┬───────┬──────────┘
                               │       │
              ┌────────────────┘       └────────────────┐
              ▼                                         ▼
  ┌───────────────────────┐              ┌───────────────────────────┐
  │  Page Migration       │              │  task_numa_fault()        │
  │  (migrate_misplaced_  │              │  记录 NUMA fault 统计    │
  │   folio)              │              │  更新 per-node 计数器    │
  └───────────────────────┘              └────────────┬────────────┘
                                                      │ 周期性触发
                                                      ▼
                                         ┌───────────────────────────┐
                                         │  task_numa_placement()    │
                                         │  计算最优 preferred node  │
                                         │  更新 scan period         │
                                         └────────────┬────────────┘
                                                      │
                                                      ▼
                                         ┌───────────────────────────┐
                                         │  numa_migrate_preferred() │
                                         │  → task_numa_migrate()    │
                                         │  尝试将任务迁移到         │
                                         │  preferred node           │
                                         └───────────────────────────┘
```

---

## 3. 核心数据结构

### 3.1 task_struct 中的 NUMA 相关字段

```c
struct task_struct {
    /* NUMA Balancing 相关 */
    int                          numa_scan_seq;          // 当前扫描序列号
    unsigned int                 numa_scan_period;       // 当前扫描周期(ms)
    unsigned int                 numa_scan_period_max;   // 最大扫描周期
    int                          numa_preferred_nid;     // 首选 NUMA 节点
    unsigned long                numa_migrate_retry;     // 下次重试迁移的时间
    u64                          node_stamp;             // 上次 NUMA 扫描的运行时间戳
    u64                          last_task_numa_placement;// 上次放置计算的时间
    u64                          last_sum_exec_runtime;  // 上次放置时的累计运行时间
    struct callback_head         numa_work;              // task_work 回调
    unsigned long                *numa_faults;           // per-node fault 统计数组
    unsigned long                total_numa_faults;      // 总 NUMA fault 数
    unsigned long                numa_faults_locality[3];// [0]=remote, [1]=local, [2]=failed_migration
    unsigned long                numa_pages_migrated;    // 已迁移的页面数
    struct numa_group __rcu      *numa_group;            // 所属 NUMA 组
};
```

### 3.2 mm_struct 中的 NUMA 相关字段

```c
struct mm_struct {
    unsigned long numa_next_scan;    // 下次允许扫描的 jiffies 时间
    unsigned long numa_scan_offset;  // 当前 VMA 扫描偏移地址
    int           numa_scan_seq;     // 全局扫描序列号
};
```

### 3.3 vma_numab_state（per-VMA NUMA 状态）

```c
struct vma_numab_state {
    unsigned long next_scan;           // 该 VMA 下次允许扫描的时间
    unsigned long pids_active_reset;   // PID active bitmap 重置时间
    unsigned long pids_active[2];      // 活跃 PID bitmap (双缓冲)
    int           start_scan_seq;      // 该 VMA 首次扫描时的序列号
    int           prev_scan_seq;       // 上次完成扫描的序列号
};
```

### 3.4 numa_faults 数组的组织方式

`task_struct->numa_faults` 是一个一维数组，按以下方式索引：

```c
// 4 种 fault bucket，每种包含 (nr_node_ids * 2) 个元素
// priv = 0(shared) 或 1(private)
enum numa_faults_stats {
    NUMA_MEM = 0,      // 内存 fault 统计（衰减后的稳定值）
    NUMA_CPU,           // CPU fault 统计（衰减后的稳定值，按运行时间加权）
    NUMA_MEMBUF,        // 内存 fault 缓冲（当前扫描周期累积值）
    NUMA_CPUBUF,        // CPU fault 缓冲（当前扫描周期累积值）
};

// 索引计算
static inline int task_faults_idx(enum numa_faults_stats s, int nid, int priv)
{
    return NR_NUMA_HINT_FAULT_TYPES * (s * nr_node_ids + nid) + priv;
}
```

每次 `task_numa_placement()` 时，MEMBUF/CPUBUF 的值会衰减合并到 MEM/CPU 中。

### 3.5 numa_group 结构

```c
struct numa_group {
    refcount_t       refcount;
    spinlock_t       lock;
    int              nr_tasks;         // 组内任务数
    pid_t            gid;              // 组 ID (创建者 pid)
    int              active_nodes;     // 活跃节点数
    struct rcu_head  rcu;
    unsigned long    total_faults;
    unsigned long    max_faults_cpu;
    unsigned long    faults[];         // per-node fault 统计
};
```

---

## 4. 阶段一：周期性扫描触发 (task_tick_numa)

**源文件**: `kernel/sched/fair.c`

`task_tick_numa()` 在每个调度器 tick 中被调用（通过 `task_tick_fair()` → `entity_tick()`），负责判断是否需要启动 NUMA 扫描。

```c
static void task_tick_numa(struct rq *rq, struct task_struct *curr)
{
    struct callback_head *work = &curr->numa_work;
    u64 period, now;

    /* 跳过无内存映射/正在退出/内核线程/已排队的任务 */
    if (!curr->mm || (curr->flags & (PF_EXITING | PF_KTHREAD)) || work->next != work)
        return;

    /* 使用任务运行时间（而非墙钟时间）驱动扫描，
     * 确保只有真正运行的任务才触发扫描 */
    now = curr->se.sum_exec_runtime;
    period = (u64)curr->numa_scan_period * NSEC_PER_MSEC;

    if (now > curr->node_stamp + period) {
        if (!curr->node_stamp)
            curr->numa_scan_period = task_scan_start(curr);
        curr->node_stamp += period;

        /* 检查 mm 级别的扫描频率限制 */
        if (!time_before(jiffies, curr->mm->numa_next_scan))
            task_work_add(curr, work, TWA_RESUME);
    }
}
```

**关键设计要点**：

1. **基于运行时间而非墙钟时间**：使用 `sum_exec_runtime` 而非 `jiffies`，确保只有 CPU 密集型任务才会频繁扫描，空闲任务不会浪费资源
2. **双重频率控制**：
   - `node_stamp + period`：per-task 的运行时间门槛
   - `mm->numa_next_scan`：per-mm 的墙钟时间门槛，避免同一地址空间的多线程重复扫描
3. **延迟执行**：通过 `task_work_add(TWA_RESUME)` 注册回调，实际扫描在返回用户态前执行，避免在调度器关键路径中做耗时操作
4. **防重入**：`work->next != work` 检查防止同一任务多次注册扫描回调

---

## 5. 阶段二：VMA 扫描与 PTE 标记 (task_numa_work)

**源文件**: `kernel/sched/fair.c`

这是 NUMA balancing 中最核心的扫描函数，负责遍历进程的 VMA 并将页表项标记为 PROT_NONE，制造 NUMA hint fault。

### 5.1 函数入口与频率控制

```c
static void task_numa_work(struct callback_head *work)
{
    unsigned long migrate, next_scan, now = jiffies;
    struct task_struct *p = current;
    struct mm_struct *mm = p->mm;
    ...

    /* 任务正在退出则跳过 */
    if (p->flags & PF_EXITING)
        return;

    /* cpuset 仅绑定了一个 NUMA 节点的内存，无需迁移 */
    if (cpusets_enabled() && nodes_weight(cpuset_current_mems_allowed) == 1)
        return;

    /* 初始化首次扫描延迟 */
    if (!mm->numa_next_scan) {
        mm->numa_next_scan = now +
            msecs_to_jiffies(sysctl_numa_balancing_scan_delay);
    }

    /* 频率限制：确保不超过最大扫描频率 */
    migrate = mm->numa_next_scan;
    if (time_before(now, migrate))
        return;

    /* 初始化扫描周期 */
    if (p->numa_scan_period == 0) {
        p->numa_scan_period_max = task_scan_max(p);
        p->numa_scan_period = task_scan_start(p);
    }

    /* 原子更新下次扫描时间，防止多线程竞争 */
    next_scan = now + msecs_to_jiffies(p->numa_scan_period);
    if (!try_cmpxchg(&mm->numa_next_scan, &migrate, next_scan))
        return;

    /* 延迟此任务的下次扫描，让同进程其他线程有机会扫描 */
    p->node_stamp += 2 * TICK_NSEC;
```

### 5.2 扫描量控制

```c
    pages = sysctl_numa_balancing_scan_size;
    pages <<= 20 - PAGE_SHIFT;  /* MB → pages */
    virtpages = pages * 8;       /* 允许扫描的虚拟页面数(用于跳过空区域) */
```

**双配额机制**：
- `pages`：实际需要标记的 PTE 数量（有物理页面映射的）
- `virtpages`：允许扫描的最大虚拟地址范围，是 pages 的 8 倍，用于快速跳过未映射或已标记的区域

### 5.3 VMA 遍历与过滤

```c
retry_pids:
    start = mm->numa_scan_offset;
    vma_iter_init(&vmi, mm, start);
    vma = vma_next(&vmi);
    if (!vma) {
        reset_ptenuma_scan(p);  /* 扫描完一轮，重置偏移 */
        start = 0;
        vma_iter_set(&vmi, start);
        vma = vma_next(&vmi);
    }

    for (; vma; vma = vma_next(&vmi)) {
        /* 过滤不可迁移的 VMA */
        if (!vma_migratable(vma) || !vma_policy_mof(vma) ||
            is_vm_hugetlb_page(vma) || (vma->vm_flags & VM_MIXEDMAP))
            continue;

        /* 跳过只读文件映射(共享库等可能有缓存副本) */
        if (!vma->vm_mm ||
            (vma->vm_file && (vma->vm_flags & (VM_READ|VM_WRITE)) == (VM_READ)))
            continue;

        /* 跳过不可访问的 VMA (避免与 PROT_NONE 混淆) */
        if (!vma_is_accessible(vma))
            continue;
```

**VMA 跳过条件总结**：

| 条件 | 原因 |
|------|------|
| `!vma_migratable` | VMA 标记为不可迁移 |
| `!vma_policy_mof` | 内存策略不允许 NUMA 优化 |
| `is_vm_hugetlb_page` | 大页不参与自动 NUMA 迁移 |
| `VM_MIXEDMAP` | 混合映射页面无法安全迁移 |
| 只读文件映射 | 可能是共享库，缓存副本足够 |
| `!vma_is_accessible` | 避免与用户态 PROT_NONE 冲突 |

### 5.4 Per-VMA NUMAB 状态管理

```c
        /* 首次访问该 VMA，初始化 numab_state */
        if (!vma->numab_state) {
            struct vma_numab_state *ptr = kzalloc(sizeof(*ptr), GFP_KERNEL);
            if (!ptr) continue;
            if (cmpxchg(&vma->numab_state, NULL, ptr)) {
                kfree(ptr);
                continue;
            }
            vma->numab_state->start_scan_seq = mm->numa_scan_seq;
            vma->numab_state->next_scan = now +
                msecs_to_jiffies(sysctl_numa_balancing_scan_delay);
            vma->numab_state->pids_active_reset = vma->numab_state->next_scan +
                msecs_to_jiffies(VMA_PID_RESET_PERIOD);
            vma->numab_state->prev_scan_seq = mm->numa_scan_seq - 1;
        }

        /* 新 VMA 延迟扫描，避免短期任务的无谓开销 */
        if (mm->numa_scan_seq && time_before(jiffies, vma->numab_state->next_scan))
            continue;

        /* 周期性重置活跃 PID bitmap */
        if (mm->numa_scan_seq &&
            time_after(jiffies, vma->numab_state->pids_active_reset)) {
            vma->numab_state->pids_active_reset += msecs_to_jiffies(VMA_PID_RESET_PERIOD);
            vma->numab_state->pids_active[0] = READ_ONCE(vma->numab_state->pids_active[1]);
            vma->numab_state->pids_active[1] = 0;
        }

        /* 同一扫描周期内不重复扫描 */
        if (vma->numab_state->prev_scan_seq == mm->numa_scan_seq)
            continue;

        /* 跳过未被当前 PID 访问的 VMA */
        if (!vma_pids_forced && !vma_is_accessed(mm, vma)) {
            vma_pids_skipped = true;
            continue;
        }
```

**PID 访问跟踪机制**：
- `pids_active[2]` 使用双缓冲 bitmap 记录哪些 PID 最近访问了该 VMA
- 当 NUMA hint fault 发生时，`vma_set_access_pid_bit()` 记录访问 PID
- `vma_is_accessed()` 检查当前 PID 是否在 bitmap 中
- 前两次扫描无条件允许（建立初始数据）
- 周期性重置 bitmap 防止过期数据

### 5.5 PTE 标记核心操作

```c
        do {
            start = max(start, vma->vm_start);
            end = ALIGN(start + (pages << PAGE_SHIFT), HPAGE_SIZE);
            end = min(end, vma->vm_end);

            /* 核心：将 PTE 权限改为 PROT_NONE (numa hint) */
            nr_pte_updates = change_prot_numa(vma, start, end);

            if (nr_pte_updates)
                pages -= (end - start) >> PAGE_SHIFT;
            virtpages -= (end - start) >> PAGE_SHIFT;

            start = end;
            if (pages <= 0 || virtpages <= 0)
                goto out;

            cond_resched();
        } while (end != vma->vm_end);
```

`change_prot_numa()` 的实现（`mm/mempolicy.c`）：

```c
unsigned long change_prot_numa(struct vm_area_struct *vma,
                               unsigned long addr, unsigned long end)
{
    struct mmu_gather tlb;
    long nr_updated;

    tlb_gather_mmu(&tlb, vma->vm_mm);
    nr_updated = change_protection(&tlb, vma, addr, end, MM_CP_PROT_NUMA);
    if (nr_updated > 0) {
        count_vm_numa_events(NUMA_PTE_UPDATES, nr_updated);
        count_memcg_events_mm(vma->vm_mm, NUMA_PTE_UPDATES, nr_updated);
    }
    tlb_finish_mmu(&tlb);
    return nr_updated;
}
```

**操作本质**：调用 `change_protection()` 将 PTE 的 `_PAGE_PRESENT` 位清除、设置 `_PAGE_PROTNONE` 位。这使得后续对这些页面的访问会触发 page fault，但 VMA 本身的权限不变（仍然可访问），内核可以区分这是 NUMA hint fault 还是真正的权限异常。

### 5.6 前进保证与开销控制

```c
    /* 如果没有可扫描的 VMA 但有被跳过的，强制扫描一个 */
    if (!vma && !vma_pids_forced && vma_pids_skipped) {
        vma_pids_forced = true;
        goto retry_pids;
    }

out:
    if (vma)
        mm->numa_scan_offset = start;
    else
        reset_ptenuma_scan(p);
    mmap_read_unlock(mm);

    /* 限制扫描开销不超过 3%：每花 1 单位时间扫描，
     * 需运行 32 单位时间的其他代码 */
    if (unlikely(p->se.sum_exec_runtime != runtime)) {
        u64 diff = p->se.sum_exec_runtime - runtime;
        p->node_stamp += 32 * diff;
    }
}
```

---

## 6. 阶段三：NUMA Hint Fault 处理 (do_numa_page)

**源文件**: `mm/memory.c`

当进程访问被 `change_prot_numa()` 标记过的页面时，由于 PTE 为 PROT_NONE，会触发 page fault。内核在 `handle_pte_fault()` 中检测到：

```c
// mm/memory.c: handle_pte_fault()
if (pte_protnone(vmf->orig_pte) && vma_is_accessible(vmf->vma))
    return do_numa_page(vmf);
```

`pte_protnone() && vma_is_accessible()` 的组合精确识别出这是 NUMA hint fault（而非真正的 PROT_NONE 页面）。

### 6.1 do_numa_page 完整流程

```c
static vm_fault_t do_numa_page(struct vm_fault *vmf)
{
    struct vm_area_struct *vma = vmf->vma;
    struct folio *folio = NULL;
    int nid = NUMA_NO_NODE;
    bool writable = false;
    int last_cpupid;
    int target_nid;
    int flags = 0, nr_pages;

    spin_lock(vmf->ptl);
    old_pte = ptep_get(vmf->pte);

    /* 验证 PTE 未被并发修改 */
    if (unlikely(!pte_same(old_pte, vmf->orig_pte))) {
        pte_unmap_unlock(vmf->pte, vmf->ptl);
        return 0;
    }

    /* 恢复原始页面保护属性 */
    pte = pte_modify(old_pte, vma->vm_page_prot);
    writable = pte_write(pte);

    folio = vm_normal_folio(vma, vmf->address, pte);
    if (!folio || folio_is_zone_device(folio))
        goto out_map;

    nid = folio_nid(folio);
    nr_pages = folio_nr_pages(folio);

    /* 检查页面是否需要迁移 */
    target_nid = numa_migrate_check(folio, vmf, vmf->address, &flags,
                                    writable, &last_cpupid);
    if (target_nid == NUMA_NO_NODE)
        goto out_map;

    /* 尝试隔离页面准备迁移 */
    if (migrate_misplaced_folio_prepare(folio, vma, target_nid)) {
        flags |= TNF_MIGRATE_FAIL;
        goto out_map;
    }

    pte_unmap_unlock(vmf->pte, vmf->ptl);

    /* 执行页面迁移 */
    if (!migrate_misplaced_folio(folio, target_nid)) {
        nid = target_nid;
        flags |= TNF_MIGRATED;
        task_numa_fault(last_cpupid, nid, nr_pages, flags);
        return 0;  /* 迁移成功，迁移代码已设置新 PTE */
    }

    flags |= TNF_MIGRATE_FAIL;
    /* 迁移失败，fall through 恢复 PTE */

out_map:
    /* 恢复 PTE 映射（去除 PROT_NONE 标记） */
    if (folio && folio_test_large(folio))
        numa_rebuild_large_mapping(vmf, vma, folio, pte, ...);
    else
        numa_rebuild_single_mapping(vmf, vma, vmf->address, vmf->pte, writable);
    pte_unmap_unlock(vmf->pte, vmf->ptl);

    /* 记录 NUMA fault 统计 */
    if (nid != NUMA_NO_NODE)
        task_numa_fault(last_cpupid, nid, nr_pages, flags);
    return 0;
}
```

### 6.2 numa_migrate_check：迁移检查

```c
int numa_migrate_check(struct folio *folio, struct vm_fault *vmf,
                       unsigned long addr, int *flags,
                       bool writable, int *last_cpupid)
{
    struct vm_area_struct *vma = vmf->vma;

    /* 只读访问不参与 NUMA 分组 */
    if (!writable)
        *flags |= TNF_NO_GROUP;

    /* 标记共享页面 */
    if (folio_maybe_mapped_shared(folio) && (vma->vm_flags & VM_SHARED))
        *flags |= TNF_SHARED;

    /* 获取页面的 last_cpupid 记录 */
    if (folio_use_access_time(folio))
        *last_cpupid = (-1 & LAST_CPUPID_MASK);  /* memory tiering 模式 */
    else
        *last_cpupid = folio_last_cpupid(folio);

    /* 记录当前 PID 的 VMA 访问 */
    vma_set_access_pid_bit(vma);

    /* 统计计数 */
    count_vm_numa_event(NUMA_HINT_FAULTS);
    if (folio_nid(folio) == numa_node_id()) {
        count_vm_numa_event(NUMA_HINT_FAULTS_LOCAL);
        *flags |= TNF_FAULT_LOCAL;
    }

    /* 根据内存策略判断页面是否放错位置 */
    return mpol_misplaced(folio, vmf, addr);
}
```

### 6.3 mpol_misplaced：基于内存策略的迁移判断

```c
int mpol_misplaced(struct folio *folio, struct vm_fault *vmf, unsigned long addr)
{
    struct mempolicy *pol;
    int curnid = folio_nid(folio);
    int thisnid = numa_node_id();
    int polnid = NUMA_NO_NODE;

    pol = get_vma_policy(vma, addr, folio_order(folio), &ilx);
    if (!(pol->flags & MPOL_F_MOF))
        goto out;  /* 策略不允许 NUMA 迁移 */

    switch (pol->mode) {
    case MPOL_INTERLEAVE:       polnid = interleave_nid(pol, ilx); break;
    case MPOL_PREFERRED:        polnid = first_node(pol->nodes); break;
    case MPOL_LOCAL:            polnid = thisnid; break;
    case MPOL_BIND:
    case MPOL_PREFERRED_MANY:   /* 检查是否在允许节点集中 */ break;
    }

    /* 对于 MPOL_F_MORON（Migrate On Reference），
     * 将目标设为当前 CPU 的节点 */
    if (pol->flags & MPOL_F_MORON) {
        polnid = thisnid;
        if (!should_numa_migrate_memory(current, folio, curnid, thiscpu))
            goto out;
    }

    if (curnid != polnid)
        ret = polnid;  /* 页面不在策略指定的节点上，返回目标节点 */
    return ret;
}
```

### 6.4 页面迁移执行

```c
// mm/migrate.c
int migrate_misplaced_folio_prepare(struct folio *folio,
                                     struct vm_area_struct *vma, int node)
{
    /* 跳过共享可执行文件映射（共享库） */
    if ((vma->vm_flags & VM_EXEC) && folio_maybe_mapped_shared(folio))
        return -EACCES;

    /* 跳过脏文件页（异步迁移不支持） */
    if (folio_test_dirty(folio))
        return -EAGAIN;

    /* 目标节点内存不足时放弃 */
    if (!migrate_balanced_pgdat(pgdat, nr_pages))
        return -EAGAIN;

    /* 从 LRU 隔离页面 */
    if (!folio_isolate_lru(folio))
        return -EAGAIN;

    return 0;
}

int migrate_misplaced_folio(struct folio *folio, int node)
{
    LIST_HEAD(migratepages);
    list_add(&folio->lru, &migratepages);

    /* 异步迁移 */
    nr_remaining = migrate_pages(&migratepages, alloc_misplaced_dst_folio,
                                 NULL, node, MIGRATE_ASYNC,
                                 MR_NUMA_MISPLACED, &nr_succeeded);
    ...
    return nr_remaining ? -EAGAIN : 0;
}
```

---

## 7. 阶段四：NUMA 故障统计与放置决策 (task_numa_fault / task_numa_placement)

### 7.1 task_numa_fault：记录 NUMA Fault

每次 NUMA hint fault 处理后，无论页面是否成功迁移，都会调用此函数记录统计信息。

```c
void task_numa_fault(int last_cpupid, int mem_node, int pages, int flags)
{
    struct task_struct *p = current;
    bool migrated = flags & TNF_MIGRATED;
    int cpu_node = task_node(current);
    int local = !!(flags & TNF_FAULT_LOCAL);
    int priv;

    /* Memory Tiering 模式下跳过慢速内存节点的统计 */
    if (!node_is_toptier(mem_node) &&
        (sysctl_numa_balancing_mode & NUMA_BALANCING_MEMORY_TIERING ||
         !cpupid_valid(last_cpupid)))
        return;

    /* 首次分配 per-node fault 数组 */
    if (unlikely(!p->numa_faults)) {
        int size = sizeof(*p->numa_faults) *
                   NR_NUMA_HINT_FAULT_BUCKETS * nr_node_ids;
        p->numa_faults = kzalloc(size, GFP_KERNEL|__GFP_NOWARN);
        if (!p->numa_faults) return;
    }

    /* 判断 private vs shared 访问 */
    if (unlikely(last_cpupid == (-1 & LAST_CPUPID_MASK))) {
        priv = 1;  /* 首次访问视为 private */
    } else {
        priv = cpupid_match_pid(p, last_cpupid);
        if (!priv && !(flags & TNF_NO_GROUP))
            task_numa_group(p, last_cpupid, flags, &priv);
    }

    /* 同一 NUMA Group 内多节点的 shared fault 视为 local */
    ng = deref_curr_numa_group(p);
    if (!priv && !local && ng && ng->active_nodes > 1 &&
        numa_is_active_node(cpu_node, ng) &&
        numa_is_active_node(mem_node, ng))
        local = 1;

    /* 周期性触发放置决策和迁移 */
    if (time_after(jiffies, p->numa_migrate_retry)) {
        task_numa_placement(p);
        numa_migrate_preferred(p);
    }

    /* 更新统计计数器 */
    if (migrated)
        p->numa_pages_migrated += pages;
    if (flags & TNF_MIGRATE_FAIL)
        p->numa_faults_locality[2] += pages;

    /* 记录到 per-node buffer 中 */
    p->numa_faults[task_faults_idx(NUMA_MEMBUF, mem_node, priv)] += pages;
    p->numa_faults[task_faults_idx(NUMA_CPUBUF, cpu_node, priv)] += pages;
    p->numa_faults_locality[local] += pages;
}
```

**Private vs Shared 判断**：
- 页面的 `last_cpupid` 记录了上次访问该页面的 CPU 和 PID
- 如果当前 PID 与 `last_cpupid` 记录的 PID 相同 → **private** 访问
- 否则 → **shared** 访问，可能触发 NUMA Group 合并

### 7.2 task_numa_placement：放置决策

当扫描序列号更新后，此函数重新评估任务的最优 NUMA 节点。

```c
static void task_numa_placement(struct task_struct *p)
{
    int seq, nid, max_nid = NUMA_NO_NODE;
    unsigned long max_faults = 0;
    unsigned long fault_types[2] = { 0, 0 };  /* [shared, private] */
    unsigned long total_faults;
    u64 runtime, period;
    struct numa_group *ng;

    seq = READ_ONCE(p->mm->numa_scan_seq);
    if (p->numa_scan_seq == seq)
        return;  /* 同一周期内不重复计算 */
    p->numa_scan_seq = seq;

    total_faults = p->numa_faults_locality[0] + p->numa_faults_locality[1];
    runtime = numa_get_avg_runtime(p, &period);

    ng = deref_curr_numa_group(p);
    if (ng) {
        spin_lock_irq(&ng->lock);
    }

    /* 遍历所有在线节点，找到 fault 最多的节点 */
    for_each_online_node(nid) {
        unsigned long faults = 0, group_faults = 0;
        int priv;

        for (priv = 0; priv < NR_NUMA_HINT_FAULT_TYPES; priv++) {
            /* 衰减合并：new_value = buf - old_value/2 + old_value
             *          即 new_value = buf + old_value/2 (指数移动平均) */
            diff = p->numa_faults[membuf_idx] - p->numa_faults[mem_idx] / 2;
            fault_types[priv] += p->numa_faults[membuf_idx];
            p->numa_faults[membuf_idx] = 0;

            /* CPU fault 按运行时间加权归一化 */
            f_weight = div64_u64(runtime << 16, period + 1);
            f_weight = (f_weight * p->numa_faults[cpubuf_idx]) /
                       (total_faults + 1);
            f_diff = f_weight - p->numa_faults[cpu_idx] / 2;
            p->numa_faults[cpubuf_idx] = 0;

            /* 更新稳定值 */
            p->numa_faults[mem_idx] += diff;
            p->numa_faults[cpu_idx] += f_diff;
            faults += p->numa_faults[mem_idx];
            p->total_numa_faults += diff;

            /* 更新 group 统计 */
            if (ng) {
                ng->faults[mem_idx] += diff;
                ng->faults[cpu_idx] += f_diff;
                ng->total_faults += diff;
                group_faults += ng->faults[mem_idx];
            }
        }

        /* 选择 fault 最多的节点作为 preferred node */
        if (!ng) {
            if (faults > max_faults) { max_faults = faults; max_nid = nid; }
        } else if (group_faults > max_faults) {
            max_faults = group_faults; max_nid = nid;
        }
    }

    /* 确保目标节点有 CPU */
    max_nid = numa_nearest_node(max_nid, N_CPU);

    if (ng) {
        numa_group_count_active_nodes(ng);
        spin_unlock_irq(&ng->lock);
        /* 对于 NUMA Group，使用更复杂的拓扑感知选择 */
        max_nid = preferred_group_nid(p, max_nid);
    }

    /* 设置新的 preferred node */
    if (max_faults && max_nid != p->numa_preferred_nid)
        sched_setnuma(p, max_nid);

    /* 更新扫描周期 */
    update_task_scan_period(p, fault_types[0], fault_types[1]);
}
```

**衰减机制的数学含义**：

```
stable[n] = stable[n-1] + (buf - stable[n-1]/2)
          = stable[n-1]/2 + buf
```

这本质上是一个指数移动平均，每个周期旧值衰减 50%，并加上新周期的观测值。这确保了：
- 近期数据权重更大
- 不会被历史噪声数据长期影响
- 渐进收敛到稳定的访问模式

---

## 8. 阶段五：任务迁移执行 (task_numa_migrate)

### 8.1 numa_migrate_preferred：触发迁移

```c
static void numa_migrate_preferred(struct task_struct *p)
{
    unsigned long interval = HZ;

    if (unlikely(p->numa_preferred_nid == NUMA_NO_NODE || !p->numa_faults))
        return;

    /* 设置周期性重试间隔 */
    interval = min(interval, msecs_to_jiffies(p->numa_scan_period) / 16);
    p->numa_migrate_retry = jiffies + interval;

    /* 已在 preferred node 上运行则无需迁移 */
    if (task_node(p) == p->numa_preferred_nid)
        return;

    task_numa_migrate(p);
}
```

### 8.2 task_numa_migrate：核心迁移逻辑

```c
static int task_numa_migrate(struct task_struct *p)
{
    struct task_numa_env env = {
        .p = p,
        .src_cpu = task_cpu(p),
        .src_nid = task_node(p),
        .imbalance_pct = 112,    /* 初始允许 12% 的负载不均衡 */
        .best_task = NULL,
        .best_imp = 0,
        .best_cpu = -1,
    };

    /* 使用最低级 SD_NUMA 域的 imbalance 阈值 */
    sd = rcu_dereference(per_cpu(sd_numa, env.src_cpu));
    if (sd)
        env.imbalance_pct = 100 + (sd->imbalance_pct - 100) / 2;

    /* 首先尝试 preferred node */
    env.dst_nid = p->numa_preferred_nid;
    dist = node_distance(env.src_nid, env.dst_nid);

    /* 计算迁移收益 */
    taskweight = task_weight(p, env.src_nid, dist);
    groupweight = group_weight(p, env.src_nid, dist);
    taskimp = task_weight(p, env.dst_nid, dist) - taskweight;
    groupimp = group_weight(p, env.dst_nid, dist) - groupweight;

    update_numa_stats(&env, &env.src_stats, env.src_nid, false);
    update_numa_stats(&env, &env.dst_stats, env.dst_nid, true);

    /* 在 preferred node 上寻找合适的 CPU */
    task_numa_find_cpu(&env, taskimp, groupimp);

    /* 如果 preferred node 没有合适位置，或 NUMA Group 跨多节点，
     * 搜索其他节点 */
    ng = deref_curr_numa_group(p);
    if (env.best_cpu == -1 || (ng && ng->active_nodes > 1)) {
        for_each_node_state(nid, N_CPU) {
            if (nid == env.src_nid || nid == p->numa_preferred_nid)
                continue;

            taskimp = task_weight(p, nid, dist) - taskweight;
            groupimp = group_weight(p, nid, dist) - groupweight;
            if (taskimp < 0 && groupimp < 0)
                continue;

            env.dst_nid = nid;
            update_numa_stats(&env, &env.dst_stats, env.dst_nid, true);
            task_numa_find_cpu(&env, taskimp, groupimp);
        }
    }

    /* 执行迁移 */
    if (env.best_cpu == -1)
        return -EAGAIN;

    if (env.best_task == NULL) {
        /* 直接迁移到空闲 CPU */
        ret = migrate_task_to(p, env.best_cpu);
    } else {
        /* 与目标 CPU 上的任务交换 */
        ret = migrate_swap(p, env.best_task, env.best_cpu, env.src_cpu);
    }
    return ret;
}
```

### 8.3 task_numa_find_cpu：在节点内搜索目标 CPU

```c
static void task_numa_find_cpu(struct task_numa_env *env,
                                long taskimp, long groupimp)
{
    bool maymove = false;
    int cpu;

    if (env->dst_stats.node_type == node_has_spare) {
        /* 目标节点有空余容量 */
        src_running = env->src_stats.nr_running - 1;
        dst_running = env->dst_stats.nr_running + 1;
        imbalance = max(0, dst_running - src_running);
        imbalance = adjust_numa_imbalance(imbalance, dst_running, env->imb_numa_nr);

        if (!imbalance) {
            maymove = true;
            if (env->dst_stats.idle_cpu >= 0) {
                /* 有空闲 CPU，直接选择 */
                env->dst_cpu = env->dst_stats.idle_cpu;
                task_numa_assign(env, NULL, 0);
                return;
            }
        }
    } else {
        /* 目标节点繁忙，检查负载均衡是否允许迁移 */
        load = task_h_load(env->p);
        dst_load = env->dst_stats.load + load;
        src_load = env->src_stats.load - load;
        maymove = !load_too_imbalanced(src_load, dst_load, env);
    }

    /* 遍历目标节点所有 CPU，逐一评估 */
    for_each_cpu(cpu, cpumask_of_node(env->dst_nid)) {
        if (!cpumask_test_cpu(cpu, env->p->cpus_ptr))
            continue;

        env->dst_cpu = cpu;
        if (task_numa_compare(env, taskimp, groupimp, maymove))
            break;  /* 找到足够好的候选 */
    }
}
```

### 8.4 task_numa_compare：评估交换候选

`task_numa_compare()` 是最复杂的评估函数，它比较将当前任务迁移到目标 CPU 的收益（可能涉及与目标 CPU 上的任务交换）。

**核心评估逻辑**：

```
imp（improvement）= 迁移后的 NUMA locality 改善值

对于目标 CPU 无任务（空闲）的情况：
  imp = taskimp 或 groupimp (取决于是否有 NUMA Group)

对于需要交换的情况：
  imp = p 在 dst 的权重 - p 在 src 的权重
      + cur 在 src 的权重 - cur 在 dst 的权重

即：交换后双方的 NUMA locality 总改善值
```

**选择策略**：
1. 优先选择空闲 CPU 直接迁移
2. 如果没有空闲 CPU，考虑与目标 CPU 上的任务交换
3. 偏好交换到 preferred node 的任务
4. `SMALLIMP` 阈值防止小改善导致的 ping-pong
5. 交换不能导致过大的负载不均衡

---

## 9. NUMA Group 机制

NUMA Group 用于识别**共享内存的多个进程/线程**，使它们的 NUMA 放置决策协调一致。

### 9.1 Group 创建与合并 (task_numa_group)

```c
static void task_numa_group(struct task_struct *p, int cpupid, int flags, int *priv)
{
    /* 如果任务没有 group，创建一个 */
    if (unlikely(!deref_curr_numa_group(p))) {
        grp = kzalloc(sizeof(*grp) + fault_stats_size, GFP_KERNEL);
        grp->gid = p->pid;
        grp->nr_tasks = 1;
        /* 复制任务的 fault 数据到 group */
        for (i = 0; i < NR_NUMA_HINT_FAULT_STATS * nr_node_ids; i++)
            grp->faults[i] = p->numa_faults[i];
        rcu_assign_pointer(p->numa_group, grp);
    }

    /* 检查是否应该合并到另一个 group */
    tsk = cpu_rq(cpu)->curr;  /* cpu = 上次访问该页面的 CPU */

    grp = rcu_dereference(tsk->numa_group);
    my_grp = deref_curr_numa_group(p);

    /* 合并条件：
     * 1. 对方 group 更大（或相等时地址更小）
     * 2. 同一进程的线程总是合并
     * 3. TNF_SHARED 标志表示真正的共享内存 */
    if (my_grp->nr_tasks > grp->nr_tasks)
        goto no_join;
    if (tsk->mm == current->mm)
        join = true;
    if (flags & TNF_SHARED)
        join = true;

    /* 执行合并：将当前任务的统计从旧 group 转移到新 group */
    if (join) {
        for (i = 0; i < NR_NUMA_HINT_FAULT_STATS * nr_node_ids; i++) {
            my_grp->faults[i] -= p->numa_faults[i];
            grp->faults[i] += p->numa_faults[i];
        }
        my_grp->nr_tasks--;
        grp->nr_tasks++;
        rcu_assign_pointer(p->numa_group, grp);
    }
}
```

### 9.2 Group 感知的放置决策

当任务属于 NUMA Group 时，`task_numa_placement()` 使用 **group fault 总和** 而非单个任务的 fault 来确定 preferred node。这使得同 group 的所有任务倾向于汇聚到相同的节点集合。

### 9.3 preferred_group_nid：拓扑感知的节点选择

对于不同的 NUMA 拓扑类型，使用不同的策略：

| 拓扑类型 | 策略 |
|----------|------|
| `NUMA_DIRECT` | 所有节点直连，直接选择 fault 最多的节点 |
| `NUMA_GLUELESS_MESH` | 网状拓扑，选择 `group_weight()` 最高的节点 |
| `NUMA_BACKPLANE` | 层级拓扑，递归搜索最高得分的节点子组 |

---

## 10. 扫描周期自适应调整

### 10.1 调整策略 (update_task_scan_period)

```c
#define NUMA_PERIOD_SLOTS     10
#define NUMA_PERIOD_THRESHOLD 7   /* 70% 阈值 */

static void update_task_scan_period(struct task_struct *p,
                                     unsigned long shared, unsigned long private)
{
    unsigned long remote = p->numa_faults_locality[0];
    unsigned long local  = p->numa_faults_locality[1];

    /* 无 fault 或有迁移失败 → 扫描减慢（翻倍） */
    if (local + shared == 0 || p->numa_faults_locality[2]) {
        p->numa_scan_period = min(p->numa_scan_period_max,
                                   p->numa_scan_period << 1);
        return;
    }

    period_slot = DIV_ROUND_UP(p->numa_scan_period, NUMA_PERIOD_SLOTS);
    lr_ratio = (local * NUMA_PERIOD_SLOTS) / (local + remote);
    ps_ratio = (private * NUMA_PERIOD_SLOTS) / (private + shared);

    if (ps_ratio >= NUMA_PERIOD_THRESHOLD) {
        /* 大部分访问是 local private → 已优化好，扫描减慢 */
        diff = (ps_ratio - NUMA_PERIOD_THRESHOLD) * period_slot;
    } else if (lr_ratio >= NUMA_PERIOD_THRESHOLD) {
        /* 大部分访问是 local shared → 共享内存为主，扫描减慢 */
        diff = (lr_ratio - NUMA_PERIOD_THRESHOLD) * period_slot;
    } else {
        /* private 访问多但不在 local node → 需要迁移，扫描加速 */
        diff = -(NUMA_PERIOD_THRESHOLD - max(lr_ratio, ps_ratio)) * period_slot;
    }

    p->numa_scan_period = clamp(p->numa_scan_period + diff,
                                 task_scan_min(p), task_scan_max(p));
    memset(p->numa_faults_locality, 0, sizeof(p->numa_faults_locality));
}
```

**自适应逻辑总结**：

```
                        local ratio 高
                            │
              ┌─────────────┼──────────────┐
              │             │              │
    private ratio 高    mixed         shared ratio 高
              │             │              │
         扫描减慢      扫描减慢        扫描减慢
    (已经本地化)    (由其他因素    (共享内存迁移
                     主导)         无意义)

                        local ratio 低
                            │
              ┌─────────────┼──────────────┐
              │             │              │
    private ratio 高    mixed         shared ratio 高
              │             │              │
         扫描加速      适度加速        扫描减慢
    (需要迁移)      (部分需要     (无法通过迁移
                     迁移)         解决)
```

### 10.2 扫描周期范围

```c
unsigned int sysctl_numa_balancing_scan_period_min = 1000;  // 1 秒
unsigned int sysctl_numa_balancing_scan_period_max = 60000; // 60 秒

// 实际范围受 task_nr_scan_windows 影响：
// task_scan_min = max(floor, scan_period_min / nr_windows)
// task_scan_max = scan_period_max / nr_windows
//
// nr_windows = RSS / scan_size，即扫描整个 RSS 所需的窗口数
```

---

## 11. 内存迁移决策 (should_numa_migrate_memory)

此函数在 `mpol_misplaced()` 中被调用，用于决定是否实际迁移一个页面。

```c
bool should_numa_migrate_memory(struct task_struct *p, struct folio *folio,
                                 int src_nid, int dst_cpu)
{
    int dst_nid = cpu_to_node(dst_cpu);
    int last_cpupid, this_cpupid;

    /* 不能迁移到无内存的节点 */
    if (!node_state(dst_nid, N_MEMORY))
        return false;

    /* Memory Tiering 模式：基于访问热度决策 */
    if (folio_use_access_time(folio)) {
        /* 目标节点有足够空间 → 允许迁移 */
        if (pgdat_free_space_enough(pgdat))
            return true;
        /* 否则基于热度阈值和提升速率限制 */
        latency = numa_hint_fault_latency(folio);
        if (latency >= th)
            return false;
        return !numa_promotion_rate_limit(pgdat, rate_limit, nr);
    }

    /* 常规模式：Multi-stage node selection */
    this_cpupid = cpu_pid_to_cpupid(dst_cpu, current->pid);
    last_cpupid = folio_xchg_last_cpupid(folio, this_cpupid);

    /* 早期快速路径：首次 fault 或同 PID 的 private 访问 */
    if ((p->numa_preferred_nid == NUMA_NO_NODE || p->numa_scan_seq <= 4) &&
        (cpupid_pid_unset(last_cpupid) || cpupid_match_pid(p, last_cpupid)))
        return true;

    /* 两阶段过滤器：
     * 1. 上次访问来自不同节点 → 不迁移（可能是临时访问）
     * 2. 上次访问来自同一节点且同一 PID → 迁移（确认是稳定的访问模式）
     *
     * 数学原理：P(migrate) = P(same_nid)^2
     * 连续两次从同一节点访问才允许迁移，过滤掉偶尔的远程访问 */
    if (!cpupid_pid_unset(last_cpupid) &&
        cpupid_to_nid(last_cpupid) != dst_nid)
        return false;

    /* NUMA Group 感知：如果 group 的 CPU fault 主要在 src_nid，
     * 不应该把内存迁走 */
    if (ng) {
        unsigned long src_faults, dst_faults;
        src_faults = group_faults_cpu(ng, src_nid);
        dst_faults = group_faults_cpu(ng, dst_nid);

        if (group_faults_cpu(ng, dst_nid) * ng->active_nodes
            < group_faults_cpu(ng, src_nid))
            return false;
    }

    /* 如果 src 是 preferred node 的一部分，保守处理 */
    if (folio_nid(folio) == p->numa_preferred_nid) {
        /* 需要 dst 有更高的 NUMA fault 比例才允许迁移 */
        ...
    }

    return true;
}
```

**两阶段过滤器的原理**：

每次 NUMA hint fault 时记录 `(cpu, pid)` 到页面的 `last_cpupid`。下次 fault 时，比较：
- 如果 `last_cpupid` 的节点 ≠ 当前节点 → 说明页面被不同节点的任务交替访问，迁移无意义
- 如果 `last_cpupid` 的节点 = 当前节点 → 连续两次从同一节点访问，确认是稳定的本地访问需求

这等效于 `P(migrate) = P(same_nid)^2`，显著降低了误判率。

---

## 12. Memory Tiering 支持

NUMA Balancing 自 Linux 5.18 起支持 Memory Tiering（内存分层）模式，用于管理异构内存系统（如 DRAM + CXL 内存 / PMEM）。

### 12.1 模式控制

```c
#define NUMA_BALANCING_MEMORY_TIERING 0x2

// 可与普通模式组合使用：
sysctl_numa_balancing_mode = NUMA_BALANCING_NORMAL | NUMA_BALANCING_MEMORY_TIERING;
```

### 12.2 Memory Tiering 特殊行为

1. **基于访问热度的迁移决策**：
   - 使用页面的访问时间（而非 cpupid）判断热度
   - 热度高于阈值的页面从慢速内存提升（promote）到快速内存
   - 阈值通过 `sysctl_numa_balancing_hot_threshold` 控制

2. **速率限制**：
   - `sysctl_numa_balancing_promote_rate_limit` 限制提升速率
   - `numa_promotion_adjust_threshold()` 根据目标节点可用空间动态调整热度阈值

3. **慢速内存节点的 fault 统计跳过**：
   ```c
   // task_numa_fault() 中：
   if (!node_is_toptier(mem_node) &&
       (sysctl_numa_balancing_mode & NUMA_BALANCING_MEMORY_TIERING))
       return;  // 慢速内存的 fault 不计入 NUMA 放置统计
   ```

4. **目标节点空间不足时唤醒 kswapd**：
   ```c
   // migrate_misplaced_folio_prepare() 中：
   if (!migrate_balanced_pgdat(pgdat, nr_pages)) {
       if (sysctl_numa_balancing_mode & NUMA_BALANCING_MEMORY_TIERING)
           wakeup_kswapd(...);  // 尝试回收空间以支持提升
       return -EAGAIN;
   }
   ```

---

## 13. 关键 sysctl 参数

所有参数可通过 `/sys/kernel/debug/sched/numa_balancing/` 或 `/proc/sys/kernel/numa_balancing*` 访问。

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `scan_delay_ms` | 1000 | 新任务/VMA 开始扫描前的延迟(ms) |
| `scan_period_min_ms` | 1000 | 最小扫描周期(ms) |
| `scan_period_max_ms` | 60000 | 最大扫描周期(ms) |
| `scan_size_mb` | 256 | 每次扫描的目标大小(MB) |
| `hot_threshold_ms` | - | Memory Tiering 热度阈值(ms) |
| `promote_rate_limit` | - | Memory Tiering 提升速率限制(MB/s) |
| `mode` | 1 | 0=禁用, 1=普通, 2=Memory Tiering, 3=两者都启用 |

---

## 14. 完整流程图

```
┌────────────────────────────────────────────────────────────────────┐
│                    NUMA Balancing 完整流程                          │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  init_numa_balancing(clone_flags, p)                               │
│    │ 初始化 task 的 NUMA balancing 状态                             │
│    │ 设置 numa_work 回调 = task_numa_work                          │
│    │ 初始 scan_period = scan_delay                                 │
│    ▼                                                               │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ 调度器运行循环                                            │      │
│  │                                                          │      │
│  │  task_tick_fair()                                        │      │
│  │    → entity_tick()                                       │      │
│  │      → task_tick_numa(rq, curr)                          │      │
│  │          │                                               │      │
│  │          ├─ 检查: mm 存在? 非内核线程? 未排队?            │      │
│  │          ├─ 检查: sum_exec_runtime > node_stamp+period?  │      │
│  │          ├─ 检查: jiffies >= mm->numa_next_scan?         │      │
│  │          └─ task_work_add(curr, work, TWA_RESUME)        │      │
│  └──────────────────────────────────┬───────────────────────┘      │
│                                     │                              │
│                                     ▼                              │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ task_numa_work() — 返回用户态前执行                        │      │
│  │                                                          │      │
│  │  1. 频率控制: cmpxchg(mm->numa_next_scan)                │      │
│  │  2. 计算扫描配额: pages, virtpages                        │      │
│  │  3. mmap_read_trylock(mm)                                │      │
│  │  4. 从 mm->numa_scan_offset 开始遍历 VMA:               │      │
│  │     ├─ 过滤: migratable? accessible? hugetlb?            │      │
│  │     ├─ 初始化/检查 vma->numab_state                      │      │
│  │     ├─ 检查 PID 是否访问过该 VMA                          │      │
│  │     └─ change_prot_numa(vma, start, end)                 │      │
│  │        → change_protection(MM_CP_PROT_NUMA)              │      │
│  │        → 将 PTE 标记为 PROT_NONE (保留 _PAGE_PROTNONE)   │      │
│  │  5. 更新 mm->numa_scan_offset                            │      │
│  │  6. 开销控制: node_stamp += 32 * scan_time               │      │
│  └──────────────────────────────────┬───────────────────────┘      │
│                                     │                              │
│               用户态访问标记过的页面                                  │
│                                     │                              │
│                                     ▼                              │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ Page Fault Handler                                       │      │
│  │                                                          │      │
│  │  handle_pte_fault()                                      │      │
│  │    → pte_protnone() && vma_is_accessible()               │      │
│  │    → do_numa_page(vmf)                                   │      │
│  │        │                                                 │      │
│  │        ├─ numa_migrate_check()                           │      │
│  │        │   ├─ 标记 TNF_NO_GROUP / TNF_SHARED / TNF_LOCAL │      │
│  │        │   ├─ vma_set_access_pid_bit() 记录 PID          │      │
│  │        │   └─ mpol_misplaced() → 判断目标节点              │      │
│  │        │       └─ should_numa_migrate_memory()           │      │
│  │        │           ├─ Memory Tiering: 热度检查            │      │
│  │        │           └─ 普通模式: 两阶段 cpupid 过滤        │      │
│  │        │                                                 │      │
│  │        ├─ target_nid != NUMA_NO_NODE?                    │      │
│  │        │   ├─ YES: migrate_misplaced_folio_prepare()     │      │
│  │        │   │       migrate_misplaced_folio()             │      │
│  │        │   │       → migrate_pages(MIGRATE_ASYNC)        │      │
│  │        │   └─ NO: 仅恢复 PTE                             │      │
│  │        │                                                 │      │
│  │        ├─ 恢复 PTE 映射 (去除 PROT_NONE)                 │      │
│  │        │   ├─ numa_rebuild_single_mapping()              │      │
│  │        │   └─ numa_rebuild_large_mapping()               │      │
│  │        │                                                 │      │
│  │        └─ task_numa_fault(cpupid, nid, pages, flags)     │      │
│  └──────────────────────────────────┬───────────────────────┘      │
│                                     │                              │
│                                     ▼                              │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ task_numa_fault() — 统计记录                              │      │
│  │                                                          │      │
│  │  1. 分配 p->numa_faults[] 数组(首次)                      │      │
│  │  2. 判断 private vs shared (cpupid_match_pid)            │      │
│  │  3. shared 访问 → task_numa_group() 合并 NUMA Group       │      │
│  │  4. 更新 MEMBUF / CPUBUF 计数器                          │      │
│  │  5. 更新 locality[local/remote/failed]                   │      │
│  │  6. 周期性触发:                                           │      │
│  │     ├─ task_numa_placement()                             │      │
│  │     └─ numa_migrate_preferred()                          │      │
│  └──────────────────────────────────┬───────────────────────┘      │
│                                     │                              │
│                                     ▼                              │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ task_numa_placement() — 放置决策                          │      │
│  │                                                          │      │
│  │  1. 衰减合并: stable = stable/2 + buffer                 │      │
│  │  2. CPU fault 按运行时间加权归一化                         │      │
│  │  3. 找到 max_faults 的节点 → max_nid                     │      │
│  │  4. NUMA Group: 使用 group_faults 而非 task_faults       │      │
│  │  5. preferred_group_nid() 拓扑感知选择                    │      │
│  │  6. sched_setnuma(p, max_nid) 设置 preferred node        │      │
│  │  7. update_task_scan_period() 自适应调整扫描周期           │      │
│  └──────────────────────────────────┬───────────────────────┘      │
│                                     │                              │
│                                     ▼                              │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ numa_migrate_preferred() / task_numa_migrate()            │      │
│  │                                                          │      │
│  │  1. 已在 preferred node → 跳过                            │      │
│  │  2. task_numa_migrate():                                 │      │
│  │     ├─ 计算 taskimp/groupimp (NUMA 收益)                 │      │
│  │     ├─ task_numa_find_cpu() 搜索目标 CPU                  │      │
│  │     │   ├─ node_has_spare → 检查 idle CPU                │      │
│  │     │   ├─ node_overloaded → 检查负载均衡                 │      │
│  │     │   └─ task_numa_compare() 评估交换候选               │      │
│  │     ├─ 搜索 preferred node 以外的节点 (if needed)         │      │
│  │     └─ 执行迁移:                                         │      │
│  │         ├─ migrate_task_to() 直接迁移到空闲 CPU           │      │
│  │         └─ migrate_swap() 与目标 CPU 任务交换              │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │ update_scan_period() — 跨节点迁移时重置扫描周期            │      │
│  │                                                          │      │
│  │  当 CFS 负载均衡将任务迁移到不同 NUMA 节点时调用           │      │
│  │  重置 scan_period = task_scan_start() 以加速重新收集数据   │      │
│  └──────────────────────────────────────────────────────────┘      │
└────────────────────────────────────────────────────────────────────┘
```

---

## 附录：关键设计思想总结

### A. 核心权衡

| 维度 | 保守策略 | 激进策略 |
|------|----------|----------|
| 扫描频率 | 低开销、低响应性 | 高开销、高响应性 |
| 迁移决策 | 低 false positive、可能错过机会 | 高 locality、可能 ping-pong |
| task vs memory | 迁移任务（快速但影响缓存） | 迁移内存（慢但精确） |

### B. 避免 Ping-Pong 的机制

1. **两阶段 cpupid 过滤**：连续两次同节点访问才迁移页面
2. **SMALLIMP 阈值**：迁移改善值必须超过门槛
3. **hysteresis（滞后）**：Group 内交换减去 1/16 的改善值
4. **扫描周期自适应**：已优化的工作负载自动降低扫描频率
5. **imbalance_pct**：迁移不能导致过大的负载不均衡

### C. 多线程/多进程协调

1. **mm->numa_next_scan + cmpxchg**：同一地址空间的多线程不重复扫描
2. **NUMA Group**：共享内存的任务统一决策
3. **node_stamp 偏移**：错开同进程线程的扫描时机

### D. 开销控制

1. 扫描开销限制在 3% 以内（`node_stamp += 32 * diff`）
2. 基于运行时间驱动（空闲任务不扫描）
3. 新 VMA / 新任务有延迟（`scan_delay`）
4. PID 跟踪跳过未访问的 VMA
5. 自适应周期可增长到 60 秒
