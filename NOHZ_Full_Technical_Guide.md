# NOHZ_FULL 技术详解与应用指南

> 完全动态时钟 (Full Dynticks / Adaptive Ticks) 技术深度解析  
> 基于 Linux 内核源码分析

---

## 目录

1. [概述](#1-概述)
2. [Linux Tick 机制演进](#2-linux-tick-机制演进)
3. [NOHZ_FULL 工作原理](#3-nohz_full-工作原理)
4. [内核配置与启动参数](#4-内核配置与启动参数)
5. [Housekeeping CPU 机制](#5-housekeeping-cpu-机制)
6. [Tick 依赖管理](#6-tick-依赖管理)
7. [RCU 回调卸载](#7-rcu-回调卸载)
8. [OS Jitter 来源与消除](#8-os-jitter-来源与消除)
9. [应用场景](#9-应用场景)
10. [最佳实践配置](#10-最佳实践配置)
11. [调试与验证](#11-调试与验证)
12. [已知限制与注意事项](#12-已知限制与注意事项)

---

## 1. 概述

### 1.1 什么是 NOHZ_FULL？

**NOHZ_FULL**（也称为 Full Dynticks 或 Adaptive Ticks）是 Linux 内核的一项特性，允许 CPU 在只有一个可运行任务时完全停止调度时钟中断（tick）。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    传统 Tick vs NOHZ_FULL                                │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  传统周期性 Tick (HZ=1000，每毫秒一次中断):                               │
│                                                                         │
│  ──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──┬──►  时间              │
│    ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑  ↑                     │
│    │  │  │  │  │  │  │  │  │  │  │  │  │  │  │  │                      │
│   tick中断 (每1ms一次，无论CPU是否繁忙)                                   │
│                                                                         │
│  NOHZ_IDLE (空闲时停止 tick):                                            │
│                                                                         │
│  ──┬──┬──┬────────────────────┬──┬──┬──┬───────────►  时间              │
│    ↑  ↑  ↑                    ↑  ↑  ↑  ↑                               │
│    │  │  │    (空闲，无tick)   │  │  │  │  (空闲)                        │
│   busy   idle                busy      idle                             │
│                                                                         │
│  NOHZ_FULL (单任务运行时也停止 tick):                                     │
│                                                                         │
│  ──┬─────────────────────────────────────────────────►  时间            │
│    ↑                                                                    │
│    │    (只有1个任务运行，停止 tick，几乎零内核干扰)                      │
│   进入用户态                                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 为什么需要 NOHZ_FULL？

| 问题 | 影响 | NOHZ_FULL 解决方案 |
|------|------|-------------------|
| **Tick 中断抖动** | 每次 tick 中断导致 1-10μs 延迟 | 停止不必要的 tick |
| **OS Jitter** | 影响实时应用的确定性 | 最小化内核干扰 |
| **HPC 迭代同步** | 一个 CPU 延迟导致所有 CPU 等待 | 消除周期性中断 |
| **功耗** | 持续中断消耗电量 | 仅在需要时唤醒 |

### 1.3 核心术语

| 术语 | 英文 | 说明 |
|------|------|------|
| **Tick** | Scheduling-clock tick | 调度时钟中断，通常 1000Hz |
| **NOHZ** | No HZ / Dyntick | 动态时钟，按需产生 tick |
| **NOHZ_IDLE** | Dyntick-idle | 空闲时停止 tick |
| **NOHZ_FULL** | Full Dyntick | 单任务运行时也停止 tick |
| **Adaptive Ticks** | - | NOHZ_FULL 的另一种叫法 |
| **Housekeeping CPU** | - | 负责管家任务的 CPU |
| **Isolated CPU** | - | 被隔离的 CPU (运行关键任务) |
| **OS Jitter** | - | 操作系统引起的抖动/延迟 |

---

## 2. Linux Tick 机制演进

### 2.1 三代 Tick 机制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Linux Tick 机制演进                               │
├───────────────┬─────────────────┬─────────────────┬─────────────────────┤
│     版本      │  CONFIG 选项     │     行为        │       适用场景      │
├───────────────┼─────────────────┼─────────────────┼─────────────────────┤
│ 传统周期性    │ HZ_PERIODIC     │ 始终产生 tick   │ 重负载、短空闲周期   │
│ (Legacy)      │                 │                 │                     │
├───────────────┼─────────────────┼─────────────────┼─────────────────────┤
│ 空闲动态时钟  │ NO_HZ_IDLE      │ 空闲时停止 tick │ 大多数场景 (默认)    │
│ (2.6.21+)     │                 │                 │                     │
├───────────────┼─────────────────┼─────────────────┼─────────────────────┤
│ 完全动态时钟  │ NO_HZ_FULL      │ 单任务时停止    │ 实时/HPC 应用       │
│ (3.10+)       │                 │ tick            │                     │
└───────────────┴─────────────────┴─────────────────┴─────────────────────┘
```

### 2.2 Tick 的作用

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    调度时钟 Tick 的主要功能                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 时间维护 (Timekeeping)                                              │
│     - 更新 jiffies 计数器                                               │
│     - 维护系统时间 (wall clock)                                         │
│     - 更新 load average                                                 │
│                                                                         │
│  2. 调度器 (Scheduler)                                                  │
│     - 更新任务运行时间统计                                               │
│     - 检查时间片是否到期                                                 │
│     - 触发负载均衡                                                       │
│     - 更新 PELT 负载追踪                                                 │
│                                                                         │
│  3. 定时器 (Timers)                                                     │
│     - 检查并处理到期的定时器                                             │
│     - hrtimer 高精度定时器处理                                           │
│                                                                         │
│  4. RCU (Read-Copy-Update)                                              │
│     - 检测静止状态 (quiescent state)                                    │
│     - 处理 RCU 回调                                                      │
│                                                                         │
│  5. 其他                                                                 │
│     - POSIX CPU 定时器                                                   │
│     - perf 事件轮换                                                      │
│     - 虚拟内存统计                                                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 3. NOHZ_FULL 工作原理

### 3.1 核心条件

NOHZ_FULL CPU 停止 tick 的条件：

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    进入 Tickless 模式的条件                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   ┌─────────────────────────────────────────────────────────────┐       │
│   │              can_stop_full_tick() 检查                      │       │
│   └─────────────────────────────────────────────────────────────┘       │
│                              │                                          │
│         ┌────────────────────┼────────────────────┐                     │
│         │                    │                    │                     │
│         ▼                    ▼                    ▼                     │
│   ┌───────────┐       ┌───────────┐       ┌───────────┐                │
│   │ 只有1个   │  AND  │ 无 tick   │  AND  │ 无 pending│                │
│   │ 可运行任务│       │ 依赖      │       │ 回调/定时 │                │
│   └───────────┘       └───────────┘       └───────────┘                │
│         │                    │                    │                     │
│         │                    │                    │                     │
│         └────────────────────┼────────────────────┘                     │
│                              │                                          │
│                              ▼                                          │
│                    ┌─────────────────┐                                  │
│                    │  停止 tick      │                                  │
│                    │  进入 tickless  │                                  │
│                    └─────────────────┘                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 状态转换

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    NOHZ_FULL 状态转换图                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                         ┌───────────────────┐                           │
│                         │    有 Tick 状态   │                           │
│                         │  (Tick Running)   │                           │
│                         └─────────┬─────────┘                           │
│                                   │                                     │
│           ┌───────────────────────┼───────────────────────┐             │
│           │                       │                       │             │
│           ▼                       │                       ▼             │
│   ┌───────────────┐               │               ┌───────────────┐     │
│   │  用户态运行   │               │               │   Idle 状态   │     │
│   │ (单任务)      │◄──────────────┴──────────────►│               │     │
│   │               │                               │               │     │
│   └───────┬───────┘                               └───────┬───────┘     │
│           │                                               │             │
│           │ 满足条件                          满足条件     │             │
│           │                                               │             │
│           ▼                                               ▼             │
│   ┌───────────────┐                               ┌───────────────┐     │
│   │ Tickless 运行 │                               │ Dyntick-idle  │     │
│   │ (Full Dyntick)│                               │  (No tick)    │     │
│   └───────┬───────┘                               └───────────────┘     │
│           │                                                             │
│           │ 需要 tick 的事件发生                                         │
│           │ (如: 新任务唤醒、定时器到期、RCU 回调)                        │
│           │                                                             │
│           ▼                                                             │
│   ┌───────────────┐                                                     │
│   │ 恢复 Tick     │                                                     │
│   │ (tick_nohz_   │                                                     │
│   │  full_kick)   │                                                     │
│   └───────────────┘                                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.3 关键代码路径

```c
/* kernel/time/tick-sched.c */

/* 检查是否可以停止 tick */
static bool can_stop_full_tick(int cpu, struct tick_sched *ts)
{
    /* 必须只有一个可运行任务 */
    if (!sched_can_stop_tick(cpu))
        return false;
    
    /* 检查 tick 依赖 */
    if (check_tick_dependency(&ts->tick_dep_mask))
        return false;
    
    /* 检查 POSIX CPU 定时器 */
    if (posix_cpu_timers_can_stop_tick(current))
        return false;
    
    /* ... 其他检查 ... */
    
    return true;
}

/* 停止 tick */
static void tick_nohz_full_stop_tick(struct tick_sched *ts, int cpu)
{
    if (tick_nohz_next_event(ts, cpu))
        tick_nohz_stop_tick(ts, cpu);  /* 重编程下一次唤醒时间 */
    else
        tick_nohz_retain_tick(ts);     /* 保持 tick */
}

/* 恢复 tick (被依赖触发) */
void tick_nohz_full_kick(void)
{
    if (!tick_nohz_full_cpu(smp_processor_id()))
        return;
    irq_work_queue(this_cpu_ptr(&nohz_full_kick_work));
}
```

---

## 4. 内核配置与启动参数

### 4.1 内核配置选项

```kconfig
# 必须启用
CONFIG_NO_HZ_FULL=y       # 启用完全动态时钟
CONFIG_RCU_NOCB_CPU=y     # 启用 RCU 回调卸载
CONFIG_CPU_ISOLATION=y    # 启用 CPU 隔离

# 推荐启用
CONFIG_NO_HZ_FULL_ALL=n   # 不要将所有 CPU 设为 nohz_full
CONFIG_CPUSETS=y          # 用于 CPU 亲和性控制

# 可选优化
CONFIG_PREEMPT_NONE=y     # 非抢占内核 (某些场景)
CONFIG_IRQ_TIME_ACCOUNTING=y  # 精确 IRQ 时间统计
```

### 4.2 启动参数

```bash
# 基本配置
nohz_full=1-7             # CPU 1-7 为 adaptive-tick CPUs
                          # CPU 0 为 housekeeping CPU (必须保留至少一个)

# RCU 回调卸载
rcu_nocbs=1-7             # 卸载 CPU 1-7 的 RCU 回调到专用线程
                          # nohz_full= 会自动隐含此设置

# CPU 隔离 (与 nohz_full 配合使用)
isolcpus=nohz,domain,managed_irq,1-7
# - nohz: 等同于 nohz_full=
# - domain: 从调度域中移除这些 CPU
# - managed_irq: 不允许 managed IRQ 路由到这些 CPU

# 示例: 8 CPU 系统完整配置
nohz_full=1-7 rcu_nocbs=1-7 isolcpus=domain,managed_irq,1-7 \
irqaffinity=0 nohz=on
```

### 4.3 参数关系图

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    启动参数关系与功能                                    │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   nohz_full=1-7                                                         │
│       │                                                                 │
│       ├──► 停止 tick (当 CPU 只有1个任务时)                             │
│       │                                                                 │
│       ├──► 自动启用 rcu_nocbs=1-7                                       │
│       │        │                                                        │
│       │        └──► RCU 回调由 rcuo* kthread 处理                       │
│       │                                                                 │
│       └──► 设置 HK_TYPE_KERNEL_NOISE                                   │
│                │                                                        │
│                ├──► HK_TYPE_TICK: 定时器/workqueue 绑定到 housekeeping │
│                ├──► HK_TYPE_TIMER: 同上                                │
│                ├──► HK_TYPE_RCU: RCU 处理卸载                          │
│                ├──► HK_TYPE_WQ: 非绑定 workqueue 避开                  │
│                └──► HK_TYPE_KTHREAD: kthread 创建避开                  │
│                                                                         │
│   isolcpus=domain,1-7                                                   │
│       │                                                                 │
│       └──► 从调度器负载均衡域中移除                                     │
│            (任务不会被自动迁移到这些 CPU)                               │
│                                                                         │
│   isolcpus=managed_irq,1-7                                              │
│       │                                                                 │
│       └──► 阻止 managed IRQ 路由到这些 CPU                              │
│                                                                         │
│   irqaffinity=0                                                         │
│       │                                                                 │
│       └──► 将默认 IRQ 亲和性设为 CPU 0                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Housekeeping CPU 机制

### 5.1 Housekeeping 概念

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Housekeeping vs Isolated CPUs                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│                          8 CPU 系统示例                                  │
│                                                                         │
│   ┌─────────┐  ┌─────────┬─────────┬─────────┬─────────┬─────────┐     │
│   │  CPU 0  │  │  CPU 1  │  CPU 2  │  CPU 3  │  CPU 4  │  ...    │     │
│   ├─────────┤  ├─────────┴─────────┴─────────┴─────────┴─────────┤     │
│   │Housekee-│  │              Isolated / nohz_full CPUs          │     │
│   │ping CPU │  │                                                  │     │
│   ├─────────┤  ├──────────────────────────────────────────────────┤     │
│   │         │  │                                                  │     │
│   │ - 时间  │  │  - 运行关键应用任务                              │     │
│   │   维护  │  │  - 最小化内核干扰                                │     │
│   │ - RCU   │  │  - 停止调度时钟 tick                             │     │
│   │   处理  │  │  - 无 RCU 回调处理                               │     │
│   │ - 中断  │  │  - 无非绑定 workqueue                            │     │
│   │   处理  │  │  - 无定时器迁移                                  │     │
│   │ - WQ    │  │                                                  │     │
│   │ - 定时器│  │                                                  │     │
│   │         │  │                                                  │     │
│   └─────────┘  └──────────────────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Housekeeping 类型

```c
/* include/linux/sched/isolation.h */
enum hk_type {
    HK_TYPE_DOMAIN,       /* 调度域隔离 */
    HK_TYPE_MANAGED_IRQ,  /* Managed IRQ 隔离 */
    HK_TYPE_KERNEL_NOISE, /* 内核噪声隔离 (nohz_full 设置) */
    HK_TYPE_MAX,
    
    /* 以下类型共享 HK_TYPE_KERNEL_NOISE 的值 */
    HK_TYPE_TICK    = HK_TYPE_KERNEL_NOISE,  /* Tick 相关 */
    HK_TYPE_TIMER   = HK_TYPE_KERNEL_NOISE,  /* 定时器相关 */
    HK_TYPE_RCU     = HK_TYPE_KERNEL_NOISE,  /* RCU 相关 */
    HK_TYPE_MISC    = HK_TYPE_KERNEL_NOISE,  /* 其他杂项 */
    HK_TYPE_WQ      = HK_TYPE_KERNEL_NOISE,  /* Workqueue 相关 */
    HK_TYPE_KTHREAD = HK_TYPE_KERNEL_NOISE   /* Kthread 相关 */
};
```

### 5.3 Housekeeping API

```c
/* kernel/sched/isolation.c */

/* 检查 CPU 是否为 housekeeping CPU */
bool housekeeping_test_cpu(int cpu, enum hk_type type);

/* 获取 housekeeping CPU 掩码 */
const struct cpumask *housekeeping_cpumask(enum hk_type type);

/* 获取任意一个 housekeeping CPU */
int housekeeping_any_cpu(enum hk_type type);

/* 将任务亲和性设置为 housekeeping CPUs */
void housekeeping_affine(struct task_struct *t, enum hk_type type);
```

---

## 6. Tick 依赖管理

### 6.1 Tick 依赖类型

某些内核子系统需要 tick 才能正常工作，它们可以声明 tick 依赖：

```c
/* include/linux/tick.h */
enum tick_dep_bits {
    TICK_DEP_BIT_POSIX_TIMER = 0,  /* POSIX CPU 定时器 */
    TICK_DEP_BIT_PERF_EVENTS = 1,  /* perf 事件 */
    TICK_DEP_BIT_SCHED       = 2,  /* 调度器 */
    TICK_DEP_BIT_CLOCK_UNSTABLE = 3, /* 时钟不稳定 */
    TICK_DEP_BIT_RCU         = 4,  /* RCU 需要检测静止状态 */
    TICK_DEP_BIT_RCU_EXP     = 5,  /* RCU 加速宽限期 */
};
```

### 6.2 依赖触发示例

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Tick 依赖触发场景                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   依赖类型              触发条件                       影响              │
│   ─────────────────────────────────────────────────────────────────     │
│   POSIX_TIMER          进程设置了 CPU 定时器          恢复 tick         │
│                                                                         │
│   PERF_EVENTS          perf 事件需要轮换              恢复 tick         │
│                        (硬件 PMU 计数器不足)                            │
│                                                                         │
│   SCHED                多个可运行任务                 恢复 tick         │
│                        (需要时间片调度)                                 │
│                                                                         │
│   RCU                  有待处理的 RCU 回调            恢复 tick         │
│                        (未卸载时)                                       │
│                                                                         │
│   RCU_EXP              RCU 加速宽限期进行中           恢复 tick         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 6.3 依赖管理 API

```c
/* 设置全局 tick 依赖 */
void tick_nohz_dep_set(enum tick_dep_bits bit);
void tick_nohz_dep_clear(enum tick_dep_bits bit);

/* 设置特定 CPU 的 tick 依赖 */
void tick_nohz_dep_set_cpu(int cpu, enum tick_dep_bits bit);
void tick_nohz_dep_clear_cpu(int cpu, enum tick_dep_bits bit);

/* 设置任务的 tick 依赖 */
void tick_nohz_dep_set_task(struct task_struct *tsk, enum tick_dep_bits bit);
void tick_nohz_dep_clear_task(struct task_struct *tsk, enum tick_dep_bits bit);

/* 设置信号组 (进程) 的 tick 依赖 */
void tick_nohz_dep_set_signal(struct task_struct *tsk, enum tick_dep_bits bit);
void tick_nohz_dep_clear_signal(struct task_struct *tsk, enum tick_dep_bits bit);
```

---

## 7. RCU 回调卸载

### 7.1 为什么需要 RCU 回调卸载

RCU (Read-Copy-Update) 是内核中广泛使用的同步机制。当使用 `call_rcu()` 注册回调时，这些回调需要在安全时机执行。传统上这需要 tick 中断来触发。

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RCU 回调处理模式                                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   传统模式 (无 RCU_NOCB_CPU):                                           │
│   ─────────────────────────────                                         │
│                                                                         │
│   CPU N:  call_rcu() ──► 添加回调到本地队列 ──► RCU_SOFTIRQ 处理        │
│                                                    ↑                    │
│                                                    │                    │
│                                               需要 tick                 │
│                                                                         │
│   NOCB 模式 (RCU_NOCB_CPU=y + rcu_nocbs=N):                            │
│   ──────────────────────────────────────────                            │
│                                                                         │
│   CPU N:  call_rcu() ──► 添加回调到队列 ──┐                             │
│                                           │                             │
│                                           ▼                             │
│                                     rcuo* kthread                       │
│                                     (可运行在其他 CPU)                   │
│                                           │                             │
│                                           ▼                             │
│                                      处理回调                           │
│                                      (无需 tick)                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 7.2 NOCB Kthreads

```bash
# 查看 RCU 卸载线程
$ ps aux | grep rcuo
root         14  0.0  0.0      0     0 ?        S    10:00   0:00 [rcuog/0]
root         15  0.0  0.0      0     0 ?        S    10:00   0:00 [rcuop/0]
root         16  0.0  0.0      0     0 ?        S    10:00   0:00 [rcuos/0]

# rcuog: RCU grace-period kthread
# rcuop: RCU offload kthread (preemptible RCU)
# rcuos: RCU offload kthread (SRCU)
```

### 7.3 绑定 RCU 线程到 Housekeeping CPU

```bash
# 将所有 rcuo* 线程绑定到 CPU 0
for pid in $(pgrep -f "rcuo"); do
    taskset -p 0x1 $pid
done

# 或使用 cgroups
mkdir -p /sys/fs/cgroup/cpuset/housekeeping
echo 0 > /sys/fs/cgroup/cpuset/housekeeping/cpuset.cpus
echo 0 > /sys/fs/cgroup/cpuset/housekeeping/cpuset.mems
for pid in $(pgrep -f "rcuo"); do
    echo $pid > /sys/fs/cgroup/cpuset/housekeeping/cgroup.procs
done
```

---

## 8. OS Jitter 来源与消除

### 8.1 OS Jitter 主要来源

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         OS Jitter 来源分类                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                     硬件中断                                    │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │  - 定时器中断 (tick)                    → nohz_full 消除       │    │
│  │  - 外设中断 (NIC, 磁盘等)               → irqaffinity 绑定     │    │
│  │  - IPI (处理器间中断)                   → 避免跨 CPU 操作      │    │
│  │  - NMI                                  → 难以完全消除         │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                     软中断/Tasklet                              │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │  - TIMER_SOFTIRQ                        → nohz_full            │    │
│  │  - NET_TX/RX_SOFTIRQ                    → 中断亲和性           │    │
│  │  - SCHED_SOFTIRQ                        → isolcpus=domain      │    │
│  │  - RCU_SOFTIRQ                          → rcu_nocbs            │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                     内核线程                                    │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │  - ksoftirqd                            → 保持 CPU 在用户态    │    │
│  │  - kworker                              → WQ_SYSFS + 亲和性    │    │
│  │  - rcuop/rcuos                          → 绑定到 housekeeping  │    │
│  │  - migration                            → isolcpus=domain      │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                     其他来源                                    │    │
│  ├────────────────────────────────────────────────────────────────┤    │
│  │  - 页错误                               → mlock + 预分配       │    │
│  │  - TLB shootdown                        → 避免模块卸载等       │    │
│  │  - 系统调用                             → 最小化内核进入       │    │
│  │  - 调度器负载均衡                       → isolcpus=domain      │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 8.2 per-CPU Kthreads 处理

主要的 per-CPU kthreads 及其处理方式：

| Kthread | 功能 | 消除方法 |
|---------|------|----------|
| `ksoftirqd/%u` | 软中断处理 | 保持 CPU 在用户态 |
| `kworker/%u` | Workqueue 执行 | 绑定非绑定 WQ 到 housekeeping |
| `rcuop/%u` | RCU 回调卸载 | 绑定到 housekeeping CPU |
| `migration/%u` | 任务迁移 | `isolcpus=domain` |
| `irq/%d-%s` | 线程化中断 | IRQ 亲和性设置 |

---

## 9. 应用场景

### 9.1 实时应用 (Real-Time)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         实时应用场景                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景描述:                                                               │
│  - 工业控制系统、机器人控制、音视频处理                                  │
│  - 需要确定性响应时间 (通常 < 100μs)                                    │
│  - 任何意外延迟都可能导致系统故障                                        │
│                                                                         │
│  NOHZ_FULL 收益:                                                        │
│  - 消除 tick 中断带来的 1-10μs 延迟                                     │
│  - 最坏情况延迟可预测                                                    │
│  - 减少中断处理开销                                                      │
│                                                                         │
│  典型配置:                                                               │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │  # 8 CPU 系统                                                  │     │
│  │  nohz_full=1-7                                                 │     │
│  │  rcu_nocbs=1-7                                                 │     │
│  │  isolcpus=domain,managed_irq,1-7                               │     │
│  │  irqaffinity=0                                                 │     │
│  │  nowatchdog                   # 禁用 watchdog                  │     │
│  │  nosoftlockup                 # 禁用软锁检测                   │     │
│  │  nmi_watchdog=0               # 禁用 NMI watchdog              │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.2 高性能计算 (HPC)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         HPC 应用场景                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景描述:                                                               │
│  - 科学计算、气象模拟、基因分析                                          │
│  - 多 CPU 并行计算，需要同步屏障                                         │
│  - 一个 CPU 延迟会导致所有其他 CPU 等待                                  │
│                                                                         │
│  问题示例:                                                               │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │                                                               │     │
│  │  CPU 0: ████████████──────────────────────────────────────    │     │
│  │  CPU 1: ████████████──────────────────────────────────────    │     │
│  │  CPU 2: ████████████──────────────────────────────────────    │     │
│  │  CPU 3: ████████████▓▓▓▓(tick中断)────────────────────────    │     │
│  │                     ↑                                         │     │
│  │                     同步屏障                                   │     │
│  │                     所有 CPU 必须等待 CPU 3                    │     │
│  │                                                               │     │
│  │  延迟放大效应: N 个 CPU 的总等待时间 = (N-1) × tick延迟       │     │
│  │                                                               │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  NOHZ_FULL 收益:                                                        │
│  - 消除同步等待时的意外延迟                                              │
│  - 迭代时间更加一致                                                      │
│  - 整体运行时间减少                                                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.3 低延迟交易系统

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      低延迟交易系统场景                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景描述:                                                               │
│  - 高频交易 (HFT)、市场数据处理                                          │
│  - 延迟敏感，微秒级别差异影响收益                                        │
│  - 需要极低的尾延迟 (P99/P99.9)                                         │
│                                                                         │
│  关键指标:                                                               │
│  - 平均延迟: 通常 < 10μs                                                │
│  - P99 延迟: 目标 < 50μs                                                │
│  - P99.99 延迟: 尽可能低                                                │
│                                                                         │
│  NOHZ_FULL 收益:                                                        │
│  - 降低延迟抖动                                                          │
│  - 减少尾延迟                                                            │
│  - 更可预测的响应时间                                                    │
│                                                                         │
│  额外优化:                                                               │
│  - 使用 busy polling 代替中断驱动 I/O                                   │
│  - DPDK/SPDK 用户态驱动                                                 │
│  - 内核旁路 (kernel bypass)                                             │
│  - NUMA 优化                                                             │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 9.4 虚拟化/云计算

```
┌─────────────────────────────────────────────────────────────────────────┐
│                      虚拟化场景 (vCPU Pinning)                           │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  场景描述:                                                               │
│  - 虚拟机 vCPU 绑定到物理 CPU                                           │
│  - Host tick 干扰 Guest 执行                                            │
│  - 多租户环境需要隔离                                                    │
│                                                                         │
│  Host 配置:                                                              │
│  ┌───────────────────────────────────────────────────────────────┐     │
│  │  # 保留 CPU 0-1 给 Host，其余给 VM                            │     │
│  │  nohz_full=2-63                                                │     │
│  │  isolcpus=domain,managed_irq,2-63                              │     │
│  └───────────────────────────────────────────────────────────────┘     │
│                                                                         │
│  收益:                                                                   │
│  - 减少 VM Exit / VM Entry                                              │
│  - 提高 vCPU 执行效率                                                   │
│  - 更好的 Guest 性能隔离                                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 10. 最佳实践配置

### 10.1 完整配置示例

```bash
#!/bin/bash
# setup_nohz_full.sh - NOHZ_FULL 系统配置脚本

# === 假设: 8 CPU 系统，CPU 0 为 housekeeping，CPU 1-7 为 isolated ===

# 1. 内核启动参数 (添加到 GRUB)
CMDLINE="nohz_full=1-7 rcu_nocbs=1-7 \
         isolcpus=domain,managed_irq,1-7 \
         irqaffinity=0 \
         nohz=on \
         nowatchdog \
         nmi_watchdog=0 \
         nosoftlockup \
         skew_tick=1 \
         tsc=reliable \
         rcu_nocb_poll"

# 2. 运行时 IRQ 亲和性设置
set_irq_affinity() {
    for irq in /proc/irq/*/smp_affinity; do
        echo 1 > "$irq" 2>/dev/null || true
    done
}

# 3. 绑定 RCU 线程到 housekeeping CPU
bind_rcu_threads() {
    for pid in $(pgrep -f "rcu"); do
        taskset -p 0x1 "$pid" 2>/dev/null || true
    done
}

# 4. 绑定其他内核线程
bind_kthreads() {
    for pid in $(pgrep -f "ksoftirqd|kworker|migration"); do
        taskset -p 0x1 "$pid" 2>/dev/null || true
    done
}

# 5. 设置 workqueue cpumask
setup_workqueues() {
    for wq in /sys/devices/virtual/workqueue/*/cpumask; do
        echo 1 > "$wq" 2>/dev/null || true
    done
}

# 6. 写回脏页设置
setup_vm() {
    # 减少写回触发的抖动
    echo 1500 > /proc/sys/vm/dirty_writeback_centisecs
    echo 3000 > /proc/sys/vm/dirty_expire_centisecs
}

# 7. 验证设置
verify_setup() {
    echo "=== NOHZ_FULL CPUs ==="
    cat /sys/devices/system/cpu/nohz_full
    
    echo "=== Isolated CPUs ==="
    cat /sys/devices/system/cpu/isolated
    
    echo "=== Housekeeping CPUs ==="
    cat /sys/devices/system/cpu/online | \
        awk -F, '{for(i=1;i<=NF;i++) print $i}' | \
        while read cpu; do
            if ! grep -q "^$cpu$" /sys/devices/system/cpu/nohz_full; then
                echo $cpu
            fi
        done
}

# 执行配置
set_irq_affinity
bind_rcu_threads
bind_kthreads
setup_workqueues
setup_vm
verify_setup
```

### 10.2 应用程序配置

```bash
# 将关键应用绑定到 isolated CPU
taskset -c 1-7 ./my_realtime_app

# 设置实时优先级
chrt -f 99 taskset -c 1-7 ./my_realtime_app

# 使用 cgroups v2
mkdir -p /sys/fs/cgroup/isolated
echo "+cpuset" > /sys/fs/cgroup/cgroup.subtree_control
echo "1-7" > /sys/fs/cgroup/isolated/cpuset.cpus
echo "0" > /sys/fs/cgroup/isolated/cpuset.mems
echo $$ > /sys/fs/cgroup/isolated/cgroup.procs
exec ./my_realtime_app
```

### 10.3 应用程序最佳实践

```c
/* 应用程序优化建议 */

// 1. 锁定内存，避免页错误
mlockall(MCL_CURRENT | MCL_FUTURE);

// 2. 预分配资源
void *buffer = malloc(BUFFER_SIZE);
memset(buffer, 0, BUFFER_SIZE);  // 预触发页错误

// 3. 避免系统调用 (在热路径中)
// 使用用户态计时器而非 gettimeofday()
// 使用共享内存而非管道/套接字

// 4. 避免触发调度
// 不要使用会导致调度的同步原语
// 使用 busy waiting 代替 sleep

// 5. 避免内存分配
// 预分配所有需要的内存
// 使用对象池模式
```

---

## 11. 调试与验证

### 11.1 验证 NOHZ_FULL 状态

```bash
# 检查内核配置
zcat /proc/config.gz | grep -E "NO_HZ|RCU_NOCB|CPU_ISOLATION"

# 查看 nohz_full CPUs
cat /sys/devices/system/cpu/nohz_full

# 查看 isolated CPUs
cat /sys/devices/system/cpu/isolated

# 查看 RCU nocb CPUs
cat /sys/kernel/rcu_nocb_cpus  # 如果存在

# 检查特定 CPU 的 tick 状态
cat /proc/interrupts | grep -E "LOC|timer"

# 查看 tick 统计
cat /proc/timer_list | grep -A5 "Tick Device"
```

### 11.2 使用 Ftrace 追踪

```bash
# 设置 ftrace
cd /sys/kernel/tracing

# 追踪 tick 相关函数
echo 1 > max_graph_depth
echo function_graph > current_tracer
echo tick_nohz_* > set_ftrace_filter
echo 1 > tracing_on

# 运行一段时间后
cat trace

# 追踪特定 CPU 的中断
echo 1 > per_cpu/cpu1/trace

# 清理
echo 0 > tracing_on
echo nop > current_tracer
```

### 11.3 使用 perf 分析

```bash
# 查看中断频率
perf stat -e irq:irq_handler_entry -a -C 1-7 sleep 10

# 查看调度器活动
perf stat -e sched:sched_switch -a -C 1-7 sleep 10

# 记录详细事件
perf record -e irq:* -e sched:* -C 1-7 sleep 10
perf script

# 使用 osnoise tracer (内核 5.14+)
echo osnoise > /sys/kernel/tracing/current_tracer
cat /sys/kernel/tracing/trace
```

### 11.4 OS Jitter 测试工具

```bash
# 使用 rt-tests 工具包
# 安装: apt install rt-tests

# cyclictest - 测量调度延迟
cyclictest -t 1 -p 99 -a 1 -n -m -l 100000

# 输出示例:
# T: 0 ( 1234) P:99 I:1000 C: 100000 Min:      1 Act:    2 Avg:    2 Max:   15

# hwlatdetect - 检测硬件延迟
hwlatdetect --duration=60

# 自定义测试 (内核自带)
git clone git://git.kernel.org/pub/scm/linux/kernel/git/frederic/dynticks-testing.git
cd dynticks-testing
./run_test.sh
```

---

## 12. 已知限制与注意事项

### 12.1 技术限制

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         NOHZ_FULL 限制                                   │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  1. 至少需要一个 housekeeping CPU                                       │
│     - 不能将所有 CPU 都设为 nohz_full                                   │
│     - 需要有 CPU 负责时间维护和系统任务                                  │
│                                                                         │
│  2. 需要至少 2 个 CPU                                                    │
│     - 单 CPU 系统无法使用 NOHZ_FULL                                     │
│                                                                         │
│  3. 每秒至少一次 tick                                                    │
│     - 用于更新负载统计、vruntime 等                                      │
│     - 这是目前的实现限制，未来可能改进                                   │
│                                                                         │
│  4. 某些条件会恢复 tick                                                  │
│     - POSIX CPU 定时器                                                  │
│     - perf 事件轮换                                                      │
│     - 多任务运行                                                         │
│     - 某些 RCU 操作                                                      │
│                                                                         │
│  5. 用户/内核转换开销略增                                               │
│     - 需要通知 RCU 等子系统状态变化                                      │
│                                                                         │
│  6. 重配置需要重启                                                       │
│     - 运行时无法更改 nohz_full 和 rcu_nocbs 设置                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 12.2 注意事项

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         使用注意事项                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ✓ DO:                                                                  │
│    - 仔细规划 CPU 分配                                                  │
│    - 测试验证配置效果                                                    │
│    - 绑定所有可能的中断和线程到 housekeeping CPU                        │
│    - 使用 mlock() 锁定关键应用内存                                      │
│    - 预分配所有资源                                                      │
│    - 保持应用在用户态运行                                                │
│                                                                         │
│  ✗ DON'T:                                                               │
│    - 不要在 isolated CPU 上运行系统服务                                 │
│    - 不要使用 POSIX CPU 定时器 (在 isolated CPU 上)                     │
│    - 不要期望零 jitter (物理限制)                                       │
│    - 不要忽略 housekeeping CPU 的负载                                   │
│    - 不要在生产环境前跳过测试                                            │
│                                                                         │
│  ⚠ WATCH OUT:                                                           │
│    - 调试器 (如 gdb) 可能导致意外 tick                                  │
│    - 某些系统调用会触发 tick                                             │
│    - 网络/磁盘 I/O 可能引入中断                                         │
│    - 页错误会进入内核                                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### 12.3 常见问题排查

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| tick 未停止 | 多个可运行任务 | 确保只有一个任务运行 |
| 周期性抖动 | POSIX CPU 定时器 | 避免使用 `setitimer()` |
| 随机抖动 | 硬件中断 | 检查并重定向 IRQ |
| 无法启动 | 配置错误 | 确保保留 housekeeping CPU |
| 性能下降 | 过度隔离 | 保留足够的 housekeeping 资源 |

---

## 参考资料

1. **内核文档**
   - `Documentation/timers/no_hz.rst`
   - `Documentation/admin-guide/kernel-per-CPU-kthreads.rst`
   - `Documentation/admin-guide/kernel-parameters.txt`

2. **核心代码**
   - `kernel/time/tick-sched.c` - NOHZ 核心实现
   - `kernel/sched/isolation.c` - CPU 隔离管理
   - `kernel/rcu/tree.c` - RCU 实现

3. **相关工具**
   - `rt-tests` - 实时性能测试工具
   - `trace-cmd` - Ftrace 前端
   - `perf` - 性能分析工具

4. **社区资源**
   - LWN.net 相关文章
   - Linux Plumbers Conference 演讲

---

> **文档版本**: 1.0  
> **最后更新**: 2026年1月  
> **基于内核版本**: Linux 6.19
