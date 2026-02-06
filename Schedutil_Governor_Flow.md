# Schedutil Governor 调频策略下发流程详解

> 基于 Linux 内核源码分析  
> 核心文件：`kernel/sched/cpufreq_schedutil.c`

---

## 目录

1. [Schedutil 概述](#1-schedutil-概述)
2. [整体架构图](#2-整体架构图)
3. [触发调频的入口点](#3-触发调频的入口点)
4. [Schedutil 核心处理流程](#4-schedutil-核心处理流程)
5. [频率下发的两条路径](#5-频率下发的两条路径)
6. [关键函数接口详解](#6-关键函数接口详解)
7. [数据结构](#7-数据结构)
8. [频率计算公式](#8-频率计算公式)
9. [时序图](#9-时序图)
10. [调试与追踪](#10-调试与追踪)

---

## 1. Schedutil 概述

**Schedutil** 是 Linux 内核中基于调度器提供的 CPU 利用率数据的 CPUFreq Governor。它与调度器紧密集成，能够快速响应负载变化。

### 1.1 核心特点

| 特性 | 说明 |
|------|------|
| **调度器集成** | 直接从 CFS/RT/DL 调度类获取利用率 |
| **PELT 驱动** | 使用 Per-Entity Load Tracking 数据 |
| **快速响应** | 支持 Fast Switch 模式，无需睡眠 |
| **频率不变性** | 支持 Frequency Invariant 计算 |
| **IO 等待加速** | 针对 IO 等待任务的频率提升 |

### 1.2 代码位置

```
kernel/sched/
├── cpufreq_schedutil.c    # Schedutil Governor 核心实现
├── cpufreq.c              # 调度器与 cpufreq 的接口
├── fair.c                 # CFS 调度类 (调用 cpufreq_update_util)
├── rt.c                   # RT 调度类
├── deadline.c             # DL 调度类
└── sched.h                # cpufreq_update_util() 定义
```

---

## 2. 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              调度器层 (Scheduler)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌───────────┐     ┌───────────┐     ┌───────────┐     ┌───────────┐      │
│   │   CFS     │     │    RT     │     │    DL     │     │   SCX     │      │
│   │ fair.c    │     │   rt.c    │     │deadline.c │     │  ext.c    │      │
│   └─────┬─────┘     └─────┬─────┘     └─────┬─────┘     └─────┬─────┘      │
│         │                 │                 │                 │            │
│         └────────────┬────┴─────────────────┴─────────────────┘            │
│                      │                                                      │
│                      ▼                                                      │
│         ┌────────────────────────────┐                                      │
│         │   cpufreq_update_util()    │  ← 调频触发入口                      │
│         │      (sched.h)             │                                      │
│         └────────────┬───────────────┘                                      │
│                      │                                                      │
└──────────────────────┼──────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Schedutil Governor 层                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│         ┌────────────────────────────┐                                      │
│         │   update_util callback     │                                      │
│         │  ┌─────────────────────┐   │                                      │
│         │  │sugov_update_single_ │   │  ← 单 CPU Policy                     │
│         │  │  freq/perf()        │   │                                      │
│         │  └─────────────────────┘   │                                      │
│         │  ┌─────────────────────┐   │                                      │
│         │  │sugov_update_shared()│   │  ← 多 CPU 共享 Policy                │
│         │  └─────────────────────┘   │                                      │
│         └────────────┬───────────────┘                                      │
│                      │                                                      │
│                      ▼                                                      │
│         ┌────────────────────────────┐                                      │
│         │     get_next_freq()        │  ← 计算目标频率                      │
│         │  - effective_cpu_util()    │                                      │
│         │  - map_util_freq()         │                                      │
│         └────────────┬───────────────┘                                      │
│                      │                                                      │
│          ┌───────────┴───────────┐                                          │
│          │                       │                                          │
│          ▼                       ▼                                          │
│   ┌─────────────────┐    ┌─────────────────┐                               │
│   │   Fast Path     │    │   Slow Path     │                               │
│   │  (fast_switch)  │    │  (kthread)      │                               │
│   └────────┬────────┘    └────────┬────────┘                               │
│            │                      │                                         │
└────────────┼──────────────────────┼─────────────────────────────────────────┘
             │                      │
             ▼                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CPUFreq Core 层                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌──────────────────────────┐    ┌────────────────────────────────┐       │
│   │cpufreq_driver_fast_      │    │  __cpufreq_driver_target()     │       │
│   │       switch()           │    │                                │       │
│   └────────────┬─────────────┘    └──────────────┬─────────────────┘       │
│                │                                  │                         │
└────────────────┼──────────────────────────────────┼─────────────────────────┘
                 │                                  │
                 ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                          CPUFreq Driver 层                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                  │
│   │ intel_pstate  │  │  amd_pstate   │  │ acpi-cpufreq  │  ...             │
│   │ .fast_switch  │  │ .fast_switch  │  │ .fast_switch  │                  │
│   │ .target       │  │ .target       │  │ .target_index │                  │
│   └───────┬───────┘  └───────┬───────┘  └───────┬───────┘                  │
│           │                  │                  │                           │
└───────────┼──────────────────┼──────────────────┼───────────────────────────┘
            │                  │                  │
            ▼                  ▼                  ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Hardware (CPU)                                    │
│                     P-State / OPP / DVFS Controller                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 触发调频的入口点

调度器在以下场景会调用 `cpufreq_update_util()` 触发调频：

### 3.1 触发场景

```
┌───────────────────────────────────────────────────────────────────────────┐
│                    cpufreq_update_util() 触发场景                          │
├───────────────┬───────────────────────────────────────────────────────────┤
│    调度类     │                    触发点                                  │
├───────────────┼───────────────────────────────────────────────────────────┤
│               │  • enqueue_task_fair() - 任务入队                          │
│     CFS       │  • dequeue_task_fair() - 任务出队                          │
│   (fair.c)    │  • update_load_avg()   - 负载更新 (PELT)                  │
│               │  • task wakeup with SCHED_CPUFREQ_IOWAIT                   │
├───────────────┼───────────────────────────────────────────────────────────┤
│      RT       │  • inc_rt_prio() / dec_rt_prio()                          │
│    (rt.c)     │  • RT 任务入队/出队时定期触发                              │
├───────────────┼───────────────────────────────────────────────────────────┤
│      DL       │  • __add_running_bw() / __sub_running_bw()                │
│ (deadline.c)  │  • DL 任务带宽变化时触发                                   │
├───────────────┼───────────────────────────────────────────────────────────┤
│     SCX       │  • scx_set_cpuperf()                                      │
│   (ext.c)     │  • BPF 调度器设置性能目标                                  │
└───────────────┴───────────────────────────────────────────────────────────┘
```

### 3.2 cpufreq_update_util() 实现

```c
/* kernel/sched/sched.h */
static inline void cpufreq_update_util(struct rq *rq, unsigned int flags)
{
    struct update_util_data *data;

    /* 获取当前 CPU 注册的回调数据 */
    data = rcu_dereference_sched(*per_cpu_ptr(&cpufreq_update_util_data,
                          cpu_of(rq)));
    if (data)
        /* 调用 schedutil 注册的回调函数 */
        data->func(data, rq_clock(rq), flags);
}
```

### 3.3 Flags 参数

```c
/* include/linux/sched/cpufreq.h */
#define SCHED_CPUFREQ_IOWAIT    (1U << 0)  /* IO 等待唤醒，需要 boost */
```

---

## 4. Schedutil 核心处理流程

### 4.1 Governor 初始化流程

```
sugov_init()                         # Governor 初始化
    │
    ├── cpufreq_enable_fast_switch() # 尝试启用快速切换
    │
    ├── sugov_policy_alloc()         # 分配 policy 数据结构
    │
    ├── sugov_kthread_create()       # 创建工作线程 (slow path)
    │       │
    │       └── kthread_create("sugov:%d")
    │           sched_setattr(SCHED_DEADLINE)  # 设为 DL 调度类
    │
    └── sugov_tunables_alloc()       # 分配可调参数
            │
            └── rate_limit_us        # 调频速率限制


sugov_start()                        # Governor 启动
    │
    ├── 选择 update 回调函数:
    │   ├── policy_is_shared()   → sugov_update_shared()
    │   ├── fast_switch + perf   → sugov_update_single_perf()
    │   └── 其他                 → sugov_update_single_freq()
    │
    └── cpufreq_add_update_util_hook()  # 注册回调到调度器
            │
            └── 设置 per_cpu(cpufreq_update_util_data, cpu)
```

### 4.2 主处理流程 (单 CPU Policy)

```
sugov_update_single_freq()           # 单 CPU Policy 的更新入口
    │
    ├── sugov_update_single_common()
    │       │
    │       ├── sugov_iowait_boost()     # 处理 IO 等待 boost
    │       │
    │       ├── sugov_should_update_freq()  # 检查是否需要更新
    │       │       │
    │       │       ├── 检查 rate_limit
    │       │       └── 检查 limits_changed
    │       │
    │       ├── sugov_iowait_apply()     # 应用 IO boost
    │       │
    │       └── sugov_get_util()         # 获取 CPU 利用率
    │               │
    │               ├── cpu_util_cfs_boost()   # CFS 利用率
    │               ├── effective_cpu_util()   # 综合利用率
    │               │       │
    │               │       ├── cpu_util_rt()  # RT 利用率
    │               │       ├── cpu_util_dl()  # DL 利用率
    │               │       └── cpu_util_irq() # IRQ 利用率
    │               │
    │               └── sugov_effective_cpu_perf()  # 添加 headroom
    │
    ├── get_next_freq()                  # 计算目标频率
    │       │
    │       ├── get_capacity_ref_freq()  # 获取参考频率
    │       ├── map_util_freq()          # 利用率映射到频率
    │       └── cpufreq_driver_resolve_freq()  # 解析到实际频率
    │
    ├── sugov_hold_freq()                # 检查是否保持频率
    │
    ├── sugov_update_next_freq()         # 更新下一个频率
    │
    └── 频率下发:
            │
            ├── [Fast Path] cpufreq_driver_fast_switch()
            │
            └── [Slow Path] sugov_deferred_update()
                    └── irq_work_queue()
```

---

## 5. 频率下发的两条路径

### 5.1 Fast Path (快速路径)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Fast Path 流程                                  │
│                   (适用于支持 fast_switch 的驱动)                         │
└─────────────────────────────────────────────────────────────────────────┘

  sugov_update_single_freq() / sugov_update_shared()
           │
           │  policy->fast_switch_enabled == true
           │
           ▼
  ┌─────────────────────────────────────┐
  │   cpufreq_driver_fast_switch()      │  ← CPUFreq Core
  │   (drivers/cpufreq/cpufreq.c)       │
  └──────────────┬──────────────────────┘
                 │
                 │  target_freq = clamp(freq, min, max)
                 │
                 ▼
  ┌─────────────────────────────────────┐
  │   cpufreq_driver->fast_switch()     │  ← Driver Callback
  │                                     │
  │   例如:                              │
  │   - intel_cpufreq_fast_switch()     │
  │   - amd_pstate_fast_switch()        │
  │   - acpi_cpufreq_fast_switch()      │
  └──────────────┬──────────────────────┘
                 │
                 │  直接写入硬件寄存器 (无睡眠)
                 │
                 ▼
  ┌─────────────────────────────────────┐
  │   arch_set_freq_scale()             │  ← 更新频率缩放因子
  │   cpufreq_stats_record_transition() │  ← 记录统计信息
  │   trace_cpu_frequency()             │  ← 触发 trace event
  └─────────────────────────────────────┘

  优点: 延迟低 (<1us), 在调度器上下文直接执行
  缺点: 需要硬件/驱动支持
```

### 5.2 Slow Path (慢速路径)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          Slow Path 流程                                  │
│                  (适用于不支持 fast_switch 的驱动)                        │
└─────────────────────────────────────────────────────────────────────────┘

  sugov_update_single_freq() / sugov_update_shared()
           │
           │  policy->fast_switch_enabled == false
           │
           ▼
  ┌─────────────────────────────────────┐
  │     sugov_deferred_update()         │  ← 延迟更新
  │                                     │
  │     if (!work_in_progress) {        │
  │         work_in_progress = true;    │
  │         irq_work_queue(&irq_work);  │
  │     }                               │
  └──────────────┬──────────────────────┘
                 │
                 │  IRQ Work 机制 (在中断上下文安全执行)
                 │
                 ▼
  ┌─────────────────────────────────────┐
  │      sugov_irq_work()               │  ← IRQ Work Handler
  │                                     │
  │      kthread_queue_work(&worker,    │
  │                         &work);     │
  └──────────────┬──────────────────────┘
                 │
                 │  唤醒 kthread (SCHED_DEADLINE)
                 │
                 ▼
  ┌─────────────────────────────────────┐
  │       sugov_work()                  │  ← Kthread Worker
  │                                     │
  │   mutex_lock(&work_lock);           │
  │   __cpufreq_driver_target(policy,   │
  │                           freq,     │
  │                           RELATION);│
  │   mutex_unlock(&work_lock);         │
  └──────────────┬──────────────────────┘
                 │
                 ▼
  ┌─────────────────────────────────────┐
  │   __cpufreq_driver_target()         │  ← CPUFreq Core
  │   (drivers/cpufreq/cpufreq.c)       │
  │                                     │
  │   → cpufreq_driver->target() 或     │
  │   → cpufreq_driver->target_index()  │
  └──────────────┬──────────────────────┘
                 │
                 ▼
  ┌─────────────────────────────────────┐
  │   Driver 具体实现                    │
  │   (可能涉及 ACPI/固件调用, 可睡眠)   │
  └─────────────────────────────────────┘

  优点: 兼容所有驱动
  缺点: 延迟较高 (需要上下文切换)
```

### 5.3 路径选择逻辑

```c
/* kernel/sched/cpufreq_schedutil.c: sugov_start() */
if (policy_is_shared(policy))
    uu = sugov_update_shared;           /* 多 CPU 共享 policy */
else if (policy->fast_switch_enabled && cpufreq_driver_has_adjust_perf())
    uu = sugov_update_single_perf;      /* 性能级别直接调整 */
else
    uu = sugov_update_single_freq;      /* 标准频率调整 */

/* 在 sugov_update_single_freq() 中选择路径 */
if (sg_policy->policy->fast_switch_enabled) {
    cpufreq_driver_fast_switch(policy, next_f);  /* Fast Path */
} else {
    sugov_deferred_update(sg_policy);            /* Slow Path */
}
```

---

## 6. 关键函数接口详解

### 6.1 调度器层接口

| 函数 | 文件 | 说明 |
|------|------|------|
| `cpufreq_update_util()` | sched.h | 调频触发入口，由调度器调用 |
| `cpufreq_add_update_util_hook()` | cpufreq.c | 注册 governor 回调 |
| `cpufreq_remove_update_util_hook()` | cpufreq.c | 移除 governor 回调 |
| `cpufreq_this_cpu_can_update()` | cpufreq.c | 检查当前 CPU 能否更新频率 |

### 6.2 Schedutil Governor 接口

| 函数 | 说明 |
|------|------|
| **回调函数** | |
| `sugov_update_single_freq()` | 单 CPU Policy，基于频率的更新 |
| `sugov_update_single_perf()` | 单 CPU Policy，基于性能级别的更新 |
| `sugov_update_shared()` | 多 CPU 共享 Policy 的更新 |
| **利用率计算** | |
| `sugov_get_util()` | 获取 CPU 综合利用率 |
| `effective_cpu_util()` | 计算有效 CPU 利用率 (CFS+RT+DL+IRQ) |
| `sugov_effective_cpu_perf()` | 添加 DVFS headroom |
| **频率计算** | |
| `get_next_freq()` | 根据利用率计算目标频率 |
| `get_capacity_ref_freq()` | 获取参考频率 |
| `map_util_freq()` | 利用率到频率的映射 |
| **更新控制** | |
| `sugov_should_update_freq()` | 检查是否应该更新频率 |
| `sugov_update_next_freq()` | 确认并更新下一个频率 |
| `sugov_deferred_update()` | 延迟更新 (slow path) |
| **IO Boost** | |
| `sugov_iowait_boost()` | 处理 IO 等待 boost 请求 |
| `sugov_iowait_apply()` | 应用 IO boost |
| `sugov_iowait_reset()` | 重置 IO boost 状态 |
| **Governor 生命周期** | |
| `sugov_init()` | Governor 初始化 |
| `sugov_exit()` | Governor 退出 |
| `sugov_start()` | Governor 启动 |
| `sugov_stop()` | Governor 停止 |
| `sugov_limits()` | 频率限制变更处理 |

### 6.3 CPUFreq Core 接口

| 函数 | 文件 | 说明 |
|------|------|------|
| `cpufreq_driver_fast_switch()` | cpufreq.c | 快速频率切换 |
| `cpufreq_driver_adjust_perf()` | cpufreq.c | 直接性能级别调整 |
| `__cpufreq_driver_target()` | cpufreq.c | 慢速频率切换 |
| `cpufreq_driver_resolve_freq()` | cpufreq.c | 解析到实际可用频率 |

### 6.4 Driver 层回调接口

```c
struct cpufreq_driver {
    /* Fast Path - 不可睡眠 */
    unsigned int (*fast_switch)(struct cpufreq_policy *policy,
                                unsigned int target_freq);
    
    /* 性能级别调整 - 不可睡眠 */
    void (*adjust_perf)(unsigned int cpu,
                        unsigned long min_perf,
                        unsigned long target_perf,
                        unsigned long capacity);
    
    /* Slow Path - 可睡眠 */
    int (*target)(struct cpufreq_policy *policy,
                  unsigned int target_freq,
                  unsigned int relation);
    
    int (*target_index)(struct cpufreq_policy *policy,
                        unsigned int index);
    
    /* ... */
};
```

---

## 7. 数据结构

### 7.1 sugov_policy

```c
/* kernel/sched/cpufreq_schedutil.c */
struct sugov_policy {
    struct cpufreq_policy   *policy;        /* 关联的 cpufreq policy */
    
    struct sugov_tunables   *tunables;      /* 可调参数 */
    struct list_head        tunables_hook;
    
    raw_spinlock_t          update_lock;    /* 更新锁 */
    u64     last_freq_update_time;          /* 上次更新时间 */
    s64     freq_update_delay_ns;           /* rate_limit 转换值 */
    unsigned int    next_freq;              /* 下一个目标频率 */
    unsigned int    cached_raw_freq;        /* 缓存的原始频率 */
    
    /* Slow Path 相关 */
    struct irq_work         irq_work;       /* IRQ work */
    struct kthread_work     work;           /* kthread work */
    struct mutex            work_lock;
    struct kthread_worker   worker;
    struct task_struct      *thread;        /* sugov kthread */
    bool    work_in_progress;               /* 工作进行中标志 */
    
    bool    limits_changed;                 /* 限制变更标志 */
    bool    need_freq_update;               /* 需要强制更新 */
};
```

### 7.2 sugov_cpu

```c
struct sugov_cpu {
    struct update_util_data update_util;    /* 回调数据 */
    struct sugov_policy     *sg_policy;     /* 所属 policy */
    unsigned int            cpu;            /* CPU 编号 */
    
    /* IO Boost */
    bool            iowait_boost_pending;   /* IO boost 待处理 */
    unsigned int    iowait_boost;           /* 当前 IO boost 值 */
    u64             last_update;            /* 上次更新时间戳 */
    
    /* 利用率 */
    unsigned long   util;                   /* 计算后的利用率 */
    unsigned long   bw_min;                 /* 最小带宽需求 (DL) */
    
#ifdef CONFIG_NO_HZ_COMMON
    unsigned long   saved_idle_calls;       /* 保存的 idle 调用计数 */
#endif
};
```

### 7.3 sugov_tunables

```c
struct sugov_tunables {
    struct gov_attr_set attr_set;
    unsigned int        rate_limit_us;      /* 调频速率限制 (微秒) */
};
```

### 7.4 update_util_data

```c
/* include/linux/sched/cpufreq.h */
struct update_util_data {
    void (*func)(struct update_util_data *data, u64 time, unsigned int flags);
};
```

---

## 8. 频率计算公式

### 8.1 利用率到频率的映射

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         频率计算公式                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   基本公式 (频率不变性):                                                  │
│                                                                         │
│                        util                                             │
│   next_freq = 1.25 × ────── × max_freq                                 │
│                        max                                              │
│                                                                         │
│   其中:                                                                  │
│     - util: 当前 CPU 利用率 (0 ~ SCHED_CAPACITY_SCALE)                  │
│     - max:  CPU 最大算力 (arch_scale_cpu_capacity)                       │
│     - max_freq: CPU 最大频率                                             │
│     - 1.25: headroom 系数，使 80% 利用率时就达到最高频                    │
│                                                                         │
│   代码实现:                                                              │
│   ```c                                                                  │
│   /* kernel/sched/cpufreq_schedutil.c */                               │
│   freq = get_capacity_ref_freq(policy);     /* max_freq */              │
│   freq = map_util_freq(util, freq, max);    /* 1.25 * util/max * freq */│
│   ```                                                                   │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   综合利用率计算:                                                        │
│                                                                         │
│   util_cfs = cpu_util_cfs() + util_est                                  │
│                                                                         │
│   util = clamp(util_cfs + util_rt, uclamp_min, uclamp_max) + irq + dl  │
│                                                                         │
│   effective_util = util * (1 + headroom)                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 map_util_freq 实现

```c
/* include/linux/sched/cpufreq.h */
static inline unsigned long map_util_freq(unsigned long util,
                      unsigned long freq,
                      unsigned long cap)
{
    return freq * util / cap;  /* 基本映射 */
}

/* kernel/sched/cpufreq_schedutil.c */
/* 在 sugov_effective_cpu_perf() 中添加 headroom */
actual = map_util_perf(actual);  /* actual + (actual >> 2) = 1.25 * actual */
```

### 8.3 IO Wait Boost

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        IO Wait Boost 机制                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   IO 等待唤醒时的频率提升策略:                                            │
│                                                                         │
│   ┌────────────────┐                                                    │
│   │ 首次 IO 唤醒   │  → iowait_boost = IOWAIT_BOOST_MIN (12.5%)         │
│   └───────┬────────┘                                                    │
│           │ 在 1 tick 内再次 IO 唤醒                                     │
│           ▼                                                             │
│   ┌────────────────┐                                                    │
│   │ boost 翻倍     │  → iowait_boost = min(boost * 2, 100%)             │
│   └───────┬────────┘                                                    │
│           │ 超过 1 tick 无 IO 唤醒                                       │
│           ▼                                                             │
│   ┌────────────────┐                                                    │
│   │ boost 衰减     │  → iowait_boost = boost / 2                        │
│   └───────┬────────┘                                                    │
│           │ boost < IOWAIT_BOOST_MIN                                    │
│           ▼                                                             │
│   ┌────────────────┐                                                    │
│   │ boost 清零     │  → iowait_boost = 0                                │
│   └────────────────┘                                                    │
│                                                                         │
│   IOWAIT_BOOST_MIN = SCHED_CAPACITY_SCALE / 8 = 128 (12.5%)            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 9. 时序图

### 9.1 完整调频时序 (Fast Path)

```
     Scheduler              Schedutil              CPUFreq Core           Driver
         │                      │                       │                    │
         │  cpufreq_update_util()                       │                    │
         │─────────────────────►│                       │                    │
         │                      │                       │                    │
         │                      │ sugov_update_single_freq()                 │
         │                      │───────┐               │                    │
         │                      │       │               │                    │
         │                      │◄──────┘               │                    │
         │                      │  get_next_freq()      │                    │
         │                      │───────┐               │                    │
         │                      │       │               │                    │
         │                      │◄──────┘               │                    │
         │                      │                       │                    │
         │                      │  cpufreq_driver_fast_switch()              │
         │                      │──────────────────────►│                    │
         │                      │                       │                    │
         │                      │                       │  ->fast_switch()   │
         │                      │                       │───────────────────►│
         │                      │                       │                    │
         │                      │                       │                    │ 写硬件寄存器
         │                      │                       │                    │───────┐
         │                      │                       │                    │       │
         │                      │                       │                    │◄──────┘
         │                      │                       │   new_freq         │
         │                      │                       │◄───────────────────│
         │                      │                       │                    │
         │                      │                       │ arch_set_freq_scale()
         │                      │                       │───────┐            │
         │                      │                       │       │            │
         │                      │                       │◄──────┘            │
         │                      │      freq             │                    │
         │◄─────────────────────│◄──────────────────────│                    │
         │                      │                       │                    │

    延迟: < 1 微秒 (典型值)
```

### 9.2 完整调频时序 (Slow Path)

```
     Scheduler        Schedutil           IRQ Context      Kthread         Driver
         │                │                    │              │               │
         │  cpufreq_      │                    │              │               │
         │  update_util() │                    │              │               │
         │───────────────►│                    │              │               │
         │                │                    │              │               │
         │                │ sugov_deferred_    │              │               │
         │                │    update()        │              │               │
         │                │────────┐           │              │               │
         │                │        │           │              │               │
         │                │◄───────┘           │              │               │
         │                │                    │              │               │
         │                │ irq_work_queue()   │              │               │
         │                │───────────────────►│              │               │
         │                │                    │              │               │
         │◄───────────────│                    │              │               │
         │  返回调度器     │                    │              │               │
         │                │                    │              │               │
         │                │                    │ sugov_irq_   │               │
         │                │                    │    work()    │               │
         │                │                    │─────────────►│               │
         │                │                    │              │               │
         │                │                    │              │ sugov_work()  │
         │                │                    │              │───────┐       │
         │                │                    │              │       │       │
         │                │                    │              │◄──────┘       │
         │                │                    │              │               │
         │                │                    │              │ __cpufreq_    │
         │                │                    │              │ driver_target │
         │                │                    │              │──────────────►│
         │                │                    │              │               │
         │                │                    │              │               │ 设置频率
         │                │                    │              │               │─────┐
         │                │                    │              │               │     │
         │                │                    │              │               │◄────┘
         │                │                    │              │◄──────────────│
         │                │                    │              │               │

    延迟: 数十到数百微秒 (取决于调度延迟)
```

---

## 10. 调试与追踪

### 10.1 sysfs 接口

```bash
# 查看/设置 governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo schedutil > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# schedutil 可调参数
cat /sys/devices/system/cpu/cpufreq/policy0/schedutil/rate_limit_us
echo 1000 > /sys/devices/system/cpu/cpufreq/policy0/schedutil/rate_limit_us

# 当前频率
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq

# 频率范围
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_min_freq
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_max_freq
```

### 10.2 Trace Events

```bash
# 启用 cpu_frequency trace
echo 1 > /sys/kernel/debug/tracing/events/power/cpu_frequency/enable

# 查看 trace
cat /sys/kernel/debug/tracing/trace

# 或使用 trace-cmd
trace-cmd record -e power:cpu_frequency -e sched:sched_switch sleep 1
trace-cmd report
```

### 10.3 关键 Trace Points

```c
/* 频率变更 trace */
trace_cpu_frequency(freq, cpu);

/* 可以添加自定义调试 */
pr_debug("schedutil: cpu %d, util %lu, freq %u\n", cpu, util, next_freq);
```

### 10.4 调试技巧

```bash
# 检查 fast_switch 是否启用
cat /sys/devices/system/cpu/cpufreq/policy0/fast_switch_enabled

# 查看 schedutil kthread
ps aux | grep sugov

# 查看 kthread 调度属性 (应该是 SCHED_DEADLINE)
chrt -p $(pgrep -f "sugov:0")

# 监控频率变化
watch -n 0.1 "cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_cur_freq"
```

---

## 11. CPPC 调频流程

### 11.1 CPPC 概述

**CPPC (Collaborative Processor Performance Control)** 是 ACPI 规范定义的 CPU 性能控制接口，主要用于 ARM64 服务器和部分 x86 平台。CPPC 调频 **完全复用 schedutil 的框架**，只是在驱动层有不同的实现。

### 11.2 CPPC 与 Schedutil 的关系

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CPPC 调频完整流程                                     │
└─────────────────────────────────────────────────────────────────────────┘

                          与其他驱动完全相同
                               │
    ┌──────────────────────────┼──────────────────────────┐
    │                          │                          │
    │        调度器层           │                          │
    │    cpufreq_update_util() │                          │
    │             │            │                          │
    │             ▼            │                          │
    │      Schedutil Governor  │   ← 相同的 Governor      │
    │  sugov_update_single_*() │                          │
    │             │            │                          │
    │             ▼            │                          │
    │     get_next_freq()      │   ← 相同的频率计算       │
    │             │            │                          │
    └─────────────┼────────────┘                          │
                  │                                        │
    ┌─────────────┼────────────┐                          │
    │             ▼            │                          │
    │      CPUFreq Core        │                          │
    │  cpufreq_driver_fast_    │                          │
    │       switch()           │                          │
    │             │            │                          │
    └─────────────┼────────────┘                          │
                  │                                        │
    ╔═════════════╪════════════╗                          │
    ║             ▼            ║   ← CPPC 特有部分        │
    ║    CPPC Driver 层        ║                          │
    ║  cppc_cpufreq_fast_      ║                          │
    ║       switch()           ║                          │
    ║             │            ║                          │
    ║             ▼            ║                          │
    ║      cppc_set_perf()     ║   ← ACPI CPPC 接口      │
    ║             │            ║                          │
    ║             ▼            ║                          │
    ║    写入 CPPC 寄存器       ║   ← 系统内存/PCC        │
    ╚══════════════════════════╝                          │
                                                          │
```

### 11.3 CPPC Driver 关键接口

```c
/* drivers/cpufreq/cppc_cpufreq.c */

static struct cpufreq_driver cppc_cpufreq_driver = {
    .flags      = CPUFREQ_CONST_LOOPS | CPUFREQ_NEED_UPDATE_LIMITS,
    .verify     = cppc_verify_policy,
    .target     = cppc_cpufreq_set_target,      /* Slow Path */
    .get        = cppc_cpufreq_get_rate,
    .fast_switch = cppc_cpufreq_fast_switch,    /* Fast Path */
    .init       = cppc_cpufreq_cpu_init,
    .exit       = cppc_cpufreq_cpu_exit,
    .set_boost  = cppc_cpufreq_set_boost,
    .name       = "cppc_cpufreq",
};
```

### 11.4 Fast Switch 条件

CPPC 是否支持 fast_switch 取决于 **DESIRED_PERF 寄存器的位置**：

```c
/* drivers/acpi/cppc_acpi.c */
bool cppc_allow_fast_switch(void)
{
    struct cpc_register_resource *desired_reg;
    struct cpc_desc *cpc_ptr;
    int cpu;

    for_each_online_cpu(cpu) {
        cpc_ptr = per_cpu(cpc_desc_ptr, cpu);
        desired_reg = &cpc_ptr->cpc_regs[DESIRED_PERF];
        
        /* 只有寄存器在系统内存或系统 I/O 时才支持 fast_switch */
        if (!CPC_IN_SYSTEM_MEMORY(desired_reg) &&
            !CPC_IN_SYSTEM_IO(desired_reg))
            return false;
    }
    return true;
}
```

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    CPPC 寄存器位置与路径选择                             │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   DESIRED_PERF 寄存器位置          →        调频路径                    │
│   ─────────────────────────────────────────────────────────             │
│   System Memory (MMIO)             →        Fast Path ✓                 │
│   System I/O (Port I/O)            →        Fast Path ✓                 │
│   PCC (Platform Communication      →        Slow Path ✗                 │
│        Channel)                             (需要固件交互)              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 11.5 CPPC Fast Path 实现

```c
/* drivers/cpufreq/cppc_cpufreq.c */
static unsigned int cppc_cpufreq_fast_switch(struct cpufreq_policy *policy,
                                              unsigned int target_freq)
{
    struct cppc_cpudata *cpu_data = policy->driver_data;
    unsigned int cpu = policy->cpu;
    u32 desired_perf;
    int ret;

    /* 1. 频率转换为 CPPC 性能级别 */
    desired_perf = cppc_khz_to_perf(&cpu_data->perf_caps, target_freq);
    cpu_data->perf_ctrls.desired_perf = desired_perf;
    
    /* 2. 调用 ACPI CPPC 接口设置性能 */
    ret = cppc_set_perf(cpu, &cpu_data->perf_ctrls);

    if (ret) {
        pr_debug("Failed to set target on CPU:%d. ret:%d\n", cpu, ret);
        return 0;
    }
    return target_freq;
}
```

### 11.6 CPPC Slow Path 实现

```c
/* drivers/cpufreq/cppc_cpufreq.c */
static int cppc_cpufreq_set_target(struct cpufreq_policy *policy,
                                    unsigned int target_freq,
                                    unsigned int relation)
{
    struct cppc_cpudata *cpu_data = policy->driver_data;
    unsigned int cpu = policy->cpu;
    struct cpufreq_freqs freqs;
    int ret = 0;

    /* 1. 频率转换为性能级别 */
    cpu_data->perf_ctrls.desired_perf =
        cppc_khz_to_perf(&cpu_data->perf_caps, target_freq);
    
    freqs.old = policy->cur;
    freqs.new = target_freq;

    /* 2. 频率转换通知 (开始) */
    cpufreq_freq_transition_begin(policy, &freqs);
    
    /* 3. 调用 ACPI CPPC 接口 */
    ret = cppc_set_perf(cpu, &cpu_data->perf_ctrls);
    
    /* 4. 频率转换通知 (结束) */
    cpufreq_freq_transition_end(policy, &freqs, ret != 0);

    return ret;
}
```

### 11.7 cppc_set_perf() 底层实现

```c
/* drivers/acpi/cppc_acpi.c */
int cppc_set_perf(int cpu, struct cppc_perf_ctrls *perf_ctrls)
{
    struct cpc_desc *cpc_desc = per_cpu(cpc_desc_ptr, cpu);
    struct cpc_register_resource *desired_reg, *min_perf_reg, *max_perf_reg;
    
    /* 获取 CPPC 寄存器描述 */
    desired_reg = &cpc_desc->cpc_regs[DESIRED_PERF];
    min_perf_reg = &cpc_desc->cpc_regs[MIN_PERF];
    max_perf_reg = &cpc_desc->cpc_regs[MAX_PERF];

    /* 如果寄存器在 PCC 中，需要获取锁 */
    if (CPC_IN_PCC(desired_reg) || CPC_IN_PCC(min_perf_reg) || 
        CPC_IN_PCC(max_perf_reg)) {
        down_read(&pcc_ss_data->pcc_lock);  /* Phase-I */
        /* ... 检查 PCC 通道归属 ... */
    }

    /* 写入性能寄存器 */
    cpc_write(cpu, desired_reg, perf_ctrls->desired_perf);
    if (perf_ctrls->min_perf)
        cpc_write(cpu, min_perf_reg, perf_ctrls->min_perf);
    if (perf_ctrls->max_perf)
        cpc_write(cpu, max_perf_reg, perf_ctrls->max_perf);

    /* 如果在 PCC 中，需要 doorbell 通知平台 */
    if (CPC_IN_PCC(desired_reg)) {
        /* Phase-II: 批量发送 PCC 命令 */
        send_pcc_cmd(pcc_ss_id, CMD_WRITE);
    }
    
    return ret;
}
```

### 11.8 CPPC 调频完整调用链

```
┌─────────────────────────────────────────────────────────────────────────┐
│              CPPC Fast Path 完整调用链                                   │
└─────────────────────────────────────────────────────────────────────────┘

[调度器上下文]
update_load_avg() / enqueue_task_fair() / ...
    └── cpufreq_update_util(rq, flags)
            └── sugov_update_single_freq()           ← Schedutil
                    ├── sugov_get_util()
                    │       └── effective_cpu_util()
                    ├── get_next_freq()
                    │       └── map_util_freq()
                    └── cpufreq_driver_fast_switch()  ← CPUFreq Core
                            └── cppc_cpufreq_fast_switch()  ← CPPC Driver
                                    ├── cppc_khz_to_perf()  # 频率→性能转换
                                    └── cppc_set_perf()     ← ACPI CPPC
                                            └── cpc_write() # 写入寄存器
                                                    └── [MMIO/PIO 写操作]


┌─────────────────────────────────────────────────────────────────────────┐
│              CPPC Slow Path (PCC) 完整调用链                             │
└─────────────────────────────────────────────────────────────────────────┘

[调度器上下文]
cpufreq_update_util()
    └── sugov_update_single_freq()
            └── sugov_deferred_update()
                    └── irq_work_queue()

[IRQ 上下文]
sugov_irq_work()
    └── kthread_queue_work()

[Kthread 上下文]
sugov_work()
    └── __cpufreq_driver_target()
            └── cppc_cpufreq_set_target()           ← CPPC Driver
                    ├── cppc_khz_to_perf()
                    ├── cpufreq_freq_transition_begin()
                    ├── cppc_set_perf()              ← ACPI CPPC
                    │       ├── down_read(&pcc_lock)  # PCC Phase-I
                    │       ├── cpc_write()           # 写入 PCC 缓冲区
                    │       ├── up_read(&pcc_lock)
                    │       ├── down_write(&pcc_lock) # PCC Phase-II  
                    │       ├── send_pcc_cmd(CMD_WRITE) # 发送 Doorbell
                    │       └── up_write(&pcc_lock)
                    └── cpufreq_freq_transition_end()
```

### 11.9 CPPC vs 其他驱动对比

| 特性 | CPPC | Intel P-State | ACPI CPUFreq |
|------|------|---------------|--------------|
| **标准** | ACPI CPPC | Intel HWP | ACPI P-States |
| **平台** | ARM64/x86 | Intel | x86 (AMD/Intel) |
| **fast_switch** | 条件支持 | 支持 | 支持 |
| **性能表示** | 性能级别 | 性能百分比 | 频率 |
| **硬件接口** | 系统内存/PCC | MSR | MSR/MMIO |
| **固件交互** | 可能 (PCC) | 无 | 无 |

### 11.10 调试 CPPC

```bash
# 检查 CPPC 驱动是否加载
lsmod | grep cppc

# 查看 CPPC 相关的 sysfs
ls /sys/devices/system/cpu/cpu0/cpufreq/

# 查看 CPPC 特有属性
cat /sys/devices/system/cpu/cpu0/cpufreq/freqdomain_cpus
cat /sys/devices/system/cpu/cpu0/cpufreq/auto_select
cat /sys/devices/system/cpu/cpu0/cpufreq/energy_performance_preference_val

# 检查 fast_switch 是否启用
cat /sys/devices/system/cpu/cpufreq/policy0/fast_switch_enabled

# 查看 ACPI CPPC 信息
cat /sys/firmware/acpi/tables/CPPC  # 如果存在

# 内核日志
dmesg | grep -i cppc
```

---

## 附录：调用链总结

### Fast Path 完整调用链

```
[调度器上下文]
update_load_avg() / enqueue_task_fair() / ...
    └── cpufreq_update_util(rq, flags)
            └── data->func()  [sugov_update_single_freq]
                    ├── sugov_update_single_common()
                    │       ├── sugov_iowait_boost()
                    │       ├── sugov_should_update_freq()
                    │       ├── sugov_iowait_apply()
                    │       └── sugov_get_util()
                    │               └── effective_cpu_util()
                    ├── get_next_freq()
                    │       ├── get_capacity_ref_freq()
                    │       ├── map_util_freq()
                    │       └── cpufreq_driver_resolve_freq()
                    ├── sugov_update_next_freq()
                    └── cpufreq_driver_fast_switch()
                            └── cpufreq_driver->fast_switch()
                                    └── [写入硬件寄存器]
```

### Slow Path 完整调用链

```
[调度器上下文]
cpufreq_update_util()
    └── sugov_update_single_freq()
            └── sugov_deferred_update()
                    └── irq_work_queue(&sg_policy->irq_work)

[IRQ 上下文]
sugov_irq_work()
    └── kthread_queue_work(&sg_policy->worker, &sg_policy->work)

[Kthread 上下文 - SCHED_DEADLINE]
sugov_work()
    └── __cpufreq_driver_target()
            ├── cpufreq_driver->target()
            └── cpufreq_driver->target_index()
                    └── [调用 ACPI/固件设置频率]
```

---

> **文档版本**: 1.0  
> **最后更新**: 2026年1月  
> **基于内核版本**: Linux 6.19-rc7
