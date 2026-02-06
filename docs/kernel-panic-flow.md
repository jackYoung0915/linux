# Linux 内核 Panic 流程分析

> 基于本仓库内核源码 (kernel/panic.c, kernel/crash_core.c 等) 整理

---

## 1. 总览

内核 panic 是 Linux 内核遇到不可恢复的致命错误时的最终处理手段。一旦进入 panic，系统将停止正常运行，
根据配置选择 kdump/kexec 转储、超时重启 或 永久挂起(halt)。

核心入口函数：
- `panic(fmt, ...)` — 可变参数版本 (`kernel/panic.c:622`)
- `vpanic(fmt, args)` — 实际实现 (`kernel/panic.c:429`)

---

## 2. Panic 触发来源

```
                         ┌─────────────────────┐
                         │  Panic 触发来源       │
                         └──────────┬──────────┘
           ┌────────────────────────┼──────────────────────────┐
           │                        │                          │
   ┌───────▼────────┐    ┌─────────▼──────────┐    ┌─────────▼──────────┐
   │  Oops → Panic   │    │   直接调用 panic()  │    │  NMI panic         │
   │ (panic_on_oops) │    │                    │    │  (nmi_panic())     │
   └───────┬────────┘    └─────────┬──────────┘    └─────────┬──────────┘
           │                        │                          │
           └────────────────────────┼──────────────────────────┘
                                    ▼
                             vpanic(fmt, args)
```

### 2.1 Oops → Panic 路径

当内核发生 oops（如空指针解引用、非法内存访问）时：

**x86 架构路径：**
```
异常/故障 → do_page_fault / do_trap / ...
  → die(str, regs, err)                   [arch/x86/kernel/dumpstack.c:453]
    → oops_begin()                         [arch/x86/kernel/dumpstack.c:349]
      → oops_enter()                       [kernel/panic.c:838]
        → nbcon_cpu_emergency_enter()      — 进入紧急打印模式
        → tracing_off()                    — 关闭 ftrace
        → debug_locks_off()                — 关闭 lockdep
        → do_oops_enter_exit()             — 处理 pause_on_oops
    → __die(str, regs, err)                — 打印 oops 信息、寄存器、调用栈
    → oops_end(flags, regs, sig)           [arch/x86/kernel/dumpstack.c:375]
      → crash_kexec(regs)                  — 如果 kexec_should_crash() 则执行 kdump
      → oops_exit()                        [kernel/panic.c:859]
        → print_oops_end_marker()
        → nbcon_cpu_emergency_exit()
        → kmsg_dump(KMSG_DUMP_OOPS)
      → if (in_interrupt())  panic("Fatal exception in interrupt")
      → if (panic_on_oops)   panic("Fatal exception")
      → 否则: make_task_dead(SIGSEGV)      — 仅杀死当前进程
```

**arm64 架构路径：**
```
异常 → die(str, regs, err)                [arch/arm64/kernel/traps.c:206]
  → oops_enter()
  → __die(str, err, regs)                 — 打印 oops 信息
  → crash_kexec(regs)                     — 如果满足条件
  → oops_exit()
  → if (in_interrupt())  panic(...)
  → if (panic_on_oops)   panic(...)
  → 否则: make_task_dead(SIGSEGV)
```

### 2.2 直接调用 panic() 的场景

| 场景 | 调用示例 |
|------|----------|
| 栈保护检测到破坏 | `__stack_chk_fail()` → `panic("stack-protector: ...")` |
| init 进程死亡 | `make_task_dead()` → `panic("Attempted to kill init!")` |
| 中断上下文异常退出 | `make_task_dead()` → `panic("Aiee, killing interrupt handler!")` |
| panic_on_warn 触发 | `check_panic_on_warn()` → `panic("...: panic_on_warn set")` |
| warn_limit 超限 | `check_panic_on_warn()` → `panic("...: system warned too often")` |
| panic_on_taint 触发 | `add_taint()` → `panic("panic_on_taint set ...")` |
| 各子系统致命错误 | 文件系统、内存管理、调度器等直接调用 panic() |

### 2.3 NMI Panic

```c
// kernel/panic.c:363
void nmi_panic(struct pt_regs *regs, const char *msg)
{
    if (panic_try_start())
        panic("%s", msg);           // 第一个到达的 CPU 执行 panic
    else if (panic_on_other_cpu())
        nmi_panic_self_stop(regs);  // 其他 CPU 的 NMI 上下文自行停止
}
```

---

## 3. vpanic() 核心流程详解

以下是 `vpanic()` 函数 (`kernel/panic.c:429`) 的完整流程：

```
vpanic(fmt, args)
│
├── [Step 1] 清除 panic_on_warn (防止递归 WARN → panic)
│
├── [Step 2] local_irq_disable() + preempt_disable_notrace()
│   禁止本地中断和抢占，防止被打断
│
├── [Step 3] panic_try_start() — CPU 竞争
│   ├── 成功 (cmpxchg): 当前 CPU 成为 panic CPU，继续执行
│   └── 失败 (其他 CPU 已 panic): panic_smp_self_stop() 永久停止自己
│
├── [Step 4] 打印 panic 信息
│   ├── console_verbose()          — 设置 console 为最详细级别
│   ├── bust_spinlocks(1)          — 强制突破 spinlock 打印
│   ├── pr_emerg("Kernel panic - not syncing: %s\n", buf)
│   └── dump_stack()               — 打印当前 CPU 调用栈 (如果满足条件)
│
├── [Step 5] kgdb_panic(buf)
│   如果启用了 KGDB，给调试器一个机会介入
│
├── [Step 6] 第一次尝试 crash_kexec (默认路径)
│   if (!crash_kexec_post_notifiers)
│       __crash_kexec(NULL)        — 尝试跳转到 kdump 内核
│       ├── kexec_trylock()
│       ├── crash_setup_regs()     — 保存寄存器
│       ├── crash_save_vmcoreinfo()— 保存 vmcoreinfo
│       ├── machine_crash_shutdown()— 架构相关关机
│       │   └── crash_smp_send_stop() — 停止其他 CPU
│       ├── crash_cma_clear_pending_dma()
│       └── machine_kexec(kexec_crash_image) — 跳转!! (不会返回)
│   ※ 如果没有加载 crash kernel，此步骤无效，继续向下执行
│
├── [Step 7] panic_other_cpus_shutdown()
│   ├── if (panic_print & SYS_INFO_ALL_BT)
│   │   → panic_trigger_all_cpu_backtrace() — NMI 方式触发所有 CPU 打印调用栈
│   └── smp_send_stop() 或 crash_smp_send_stop()
│       停止所有其他 CPU
│
├── [Step 8] printk_legacy_allow_panic_sync()
│   允许 panic CPU 上的 legacy console 同步打印
│
├── [Step 9] atomic_notifier_call_chain(&panic_notifier_list, 0, buf)
│   遍历并调用所有注册的 panic notifier 回调
│   （各驱动/子系统可注册自己的 panic 处理函数）
│
├── [Step 10] sys_info(panic_print) — 打印系统诊断信息
│   根据 panic_print 位掩码打印：
│   ├── SYS_INFO_TASKS (0x01)              — 所有任务信息
│   ├── SYS_INFO_MEM (0x02)                — 内存信息
│   ├── SYS_INFO_TIMERS (0x04)             — 定时器信息
│   ├── SYS_INFO_LOCKS (0x08)              — 锁信息
│   ├── SYS_INFO_FTRACE (0x10)             — ftrace buffer
│   ├── SYS_INFO_ALL_BT (0x40)             — 所有 CPU backtrace
│   └── SYS_INFO_BLOCKED_TASKS (0x80)      — blocked tasks
│
├── [Step 11] kmsg_dump_desc(KMSG_DUMP_PANIC, buf)
│   调用所有注册的 kmsg dumper（如 pstore、mtdoops 等）
│   将内核日志转储到持久化存储
│
├── [Step 12] 第二次尝试 crash_kexec (post-notifiers 路径)
│   if (crash_kexec_post_notifiers)
│       __crash_kexec(NULL)        — 在 notifier 和 kmsg_dump 之后跳转到 kdump
│
├── [Step 13] 最终 console 刷新
│   ├── console_unblank()          — 恢复 console 显示
│   ├── debug_locks_off()
│   ├── console_flush_on_panic(CONSOLE_FLUSH_PENDING) — 刷新未打印的消息
│   └── if (panic_console_replay)
│       console_flush_on_panic(CONSOLE_REPLAY_ALL)    — 重放所有日志
│
├── [Step 14] 超时重启逻辑
│   ├── if (panic_timeout > 0)
│   │   等待 panic_timeout 秒后重启
│   │   for (i = 0; i < panic_timeout * 1000; ...)
│   │       touch_nmi_watchdog()   — 防止 watchdog 干扰
│   │       panic_blink()          — 键盘 LED 闪烁
│   │       mdelay(100ms)
│   │
│   └── if (panic_timeout != 0)    — 正数或负数都重启
│       emergency_restart()
│       ├── kmsg_dump(KMSG_DUMP_EMERG)
│       └── machine_emergency_restart()  — 架构相关紧急重启
│
└── [Step 15] 永久挂起 (panic_timeout == 0)
    ├── pr_emerg("---[ end Kernel panic - not syncing: %s ]---\n", buf)
    ├── suppress_printk = 1
    ├── console_flush_on_panic(CONSOLE_FLUSH_PENDING) — 最后一次刷新
    ├── nbcon_atomic_flush_unsafe()                    — 强制 nbcon 刷新
    ├── local_irq_enable()
    └── for (;;)                   — 无限循环
        touch_softlockup_watchdog()
        panic_blink()              — 持续闪烁键盘 LED
        mdelay(100ms)
```

---

## 4. 多 CPU 同步机制

### 4.1 panic_cpu 原子变量

```c
atomic_t panic_cpu = ATOMIC_INIT(PANIC_CPU_INVALID);  // 初始值 -1
```

多个 CPU 可能同时触发 panic（如同时收到 NMI）。使用 `atomic_cmpxchg` 保证只有一个 CPU
成为 "panic CPU" 并执行 panic 流程：

```
CPU 0: panic_try_start() → cmpxchg(-1, 0) → 成功! → 执行 panic 流程
CPU 1: panic_try_start() → cmpxchg(-1, 1) → 失败(已是0) → panic_smp_self_stop()
CPU 2: panic_try_start() → cmpxchg(-1, 2) → 失败(已是0) → panic_smp_self_stop()
```

### 4.2 辅助判断函数

| 函数 | 作用 |
|------|------|
| `panic_in_progress()` | 是否有任何 CPU 正在 panic |
| `panic_on_this_cpu()` | 当前 CPU 是否是 panic CPU |
| `panic_on_other_cpu()` | 是否有其他 CPU 正在 panic |

### 4.3 停止其他 CPU

```
panic_other_cpus_shutdown(crash_kexec)
├── if (!crash_kexec)
│   └── smp_send_stop()           — 普通停止(IPI)
└── if (crash_kexec)
    └── crash_smp_send_stop()     — crash 停止(可能用 NMI shootdown)
        [x86] → kdump_nmi_shootdown_cpus()
                → nmi_shootdown_cpus()  — 通过 NMI 强制停止所有 CPU
                → crash_save_cpu()      — 保存每个 CPU 的寄存器状态
```

---

## 5. Kdump/Kexec 路径

```
vpanic()
├── [默认路径] crash_kexec_post_notifiers = false
│   → __crash_kexec(NULL)         — 在 notifier 之前尝试 kdump
│
└── [延迟路径] crash_kexec_post_notifiers = true
    → 先执行 panic notifiers + kmsg_dump
    → __crash_kexec(NULL)         — 之后再尝试 kdump
```

`__crash_kexec()` 内部流程 (`kernel/crash_core.c:120`)：

```
__crash_kexec(regs)
│
├── kexec_trylock()                — 获取 kexec 锁
│
├── if (kexec_crash_image)         — 检查是否加载了 crash kernel
│   ├── crash_setup_regs(&fixed_regs, regs) — 保存/修复寄存器
│   ├── crash_save_vmcoreinfo()    — 保存 vmcoreinfo
│   ├── machine_crash_shutdown(&fixed_regs) — 架构相关的崩溃关机
│   │   [x86: native_machine_crash_shutdown()]
│   │   ├── local_irq_disable()
│   │   ├── crash_smp_send_stop()  — 停止其他 CPU (NMI shootdown)
│   │   ├── cpu_emergency_disable_virtualization()
│   │   ├── cpu_emergency_stop_pt()— 停止 Intel PT
│   │   ├── clear_IO_APIC()        — 清除 IO-APIC
│   │   └── lapic_shutdown()       — 关闭 Local APIC
│   │
│   ├── crash_cma_clear_pending_dma() — 等待 CMA 区域 DMA 完成
│   └── machine_kexec(kexec_crash_image) — ★ 跳转到 crash kernel ★
│       （此后不再返回，由 kdump 内核接管）
│
└── kexec_unlock()
```

---

## 6. Panic Notifier 机制

panic notifier 是一个 atomic notifier chain，允许内核子系统和驱动注册回调，
在 panic 时执行最后的清理或数据保存操作。

```c
ATOMIC_NOTIFIER_HEAD(panic_notifier_list);  // kernel/panic.c:77

// 注册示例
static struct notifier_block my_panic_nb = {
    .notifier_call = my_panic_handler,
    .priority = 0,
};
atomic_notifier_chain_register(&panic_notifier_list, &my_panic_nb);
```

在 `vpanic()` 中调用：
```c
atomic_notifier_call_chain(&panic_notifier_list, 0, buf);
```

常见注册 panic notifier 的子系统：
- **pstore** — 持久化存储日志
- **各硬件看门狗驱动** — 确保 watchdog 状态正确
- **hypervisor 接口** — 通知 host 端
- **网络子系统** (netconsole) — 通过网络发送 panic 信息
- **RAS/EDAC** — 记录硬件错误信息

---

## 7. 关键配置参数

### 7.1 启动参数 (Boot Parameters)

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `panic=N` | panic 后 N 秒重启；0=不重启；负数=立即重启 | `CONFIG_PANIC_TIMEOUT` |
| `panic_on_oops` | oops 时是否触发 panic | `CONFIG_PANIC_ON_OOPS` |
| `panic_on_warn` | WARN 时是否触发 panic | 0 |
| `oops=panic` | 等同于 `panic_on_oops=1` | - |
| `crash_kexec_post_notifiers` | kdump 是否在 notifier 之后执行 | false |
| `panic_print=N` | 废弃，使用 `panic_sys_info` | - |
| `panic_sys_info=tasks,mem,...` | panic 时打印的系统信息 | 0 |
| `panic_console_replay` | panic 时是否重放全部 console 日志 | false |
| `pause_on_oops=N` | oops 后暂停 N 秒（防止滚屏） | 0 |
| `panic_on_taint=0xHEX[,nousertaint]` | 某些 taint 标志触发 panic | 0 |

### 7.2 sysctl 接口 (/proc/sys/kernel/)

| 路径 | 说明 |
|------|------|
| `/proc/sys/kernel/panic` | panic_timeout |
| `/proc/sys/kernel/panic_on_oops` | oops 触发 panic |
| `/proc/sys/kernel/panic_on_warn` | warn 触发 panic |
| `/proc/sys/kernel/panic_print` | (废弃) 打印掩码 |
| `/proc/sys/kernel/panic_sys_info` | panic 时系统信息掩码 |
| `/proc/sys/kernel/oops_all_cpu_backtrace` | oops 时所有 CPU backtrace |
| `/proc/sys/kernel/warn_limit` | WARN 次数限制 |

### 7.3 panic_timeout 行为

```
panic_timeout > 0  → 等待 N 秒后调用 emergency_restart() 重启
panic_timeout < 0  → 立即调用 emergency_restart() 重启 (不等待)
panic_timeout == 0 → 永久挂起，LED 闪烁
```

---

## 8. 完整时序图

```
时间 ─────────────────────────────────────────────────────────────────────►

[触发]         [初始化]              [转储/停止]             [善后]              [终结]
  │               │                     │                    │                   │
  │  local_irq_disable()                │                    │                   │
  │  preempt_disable()                  │                    │                   │
  │  panic_try_start()                  │                    │                   │
  │  console_verbose()                  │                    │                   │
  │  pr_emerg("Kernel panic...")        │                    │                   │
  │  dump_stack()                       │                    │                   │
  │  kgdb_panic()                       │                    │                   │
  │               │                     │                    │                   │
  │               │  __crash_kexec() ───┤ (若成功则不返回)   │                   │
  │               │  ↓(若无crash kernel)│                    │                   │
  │               │  NMI all-CPU BT     │                    │                   │
  │               │  smp_send_stop()    │                    │                   │
  │               │  printk sync        │                    │                   │
  │               │                     │                    │                   │
  │               │                     │  panic_notifiers   │                   │
  │               │                     │  sys_info()        │                   │
  │               │                     │  kmsg_dump()       │                   │
  │               │                     │                    │                   │
  │               │                     │  __crash_kexec() ──┤ (post_notifiers) │
  │               │                     │  ↓(若无crash kernel)                  │
  │               │                     │                    │  console_unblank  │
  │               │                     │                    │  console_flush    │
  │               │                     │                    │                   │
  │               │                     │                    │  wait timeout     │
  │               │                     │                    │  emergency_restart│
  │               │                     │                    │  或 永久 halt     │
```

---

## 9. Oops vs Panic 区别

| 特性 | Oops | Panic |
|------|------|-------|
| 严重程度 | 可恢复（通常仅杀死进程） | 不可恢复（系统停止） |
| 系统状态 | 继续运行（但已 tainted） | 完全停止 |
| 其他 CPU | 不影响 | 全部停止 |
| 转换关系 | 可升级为 panic (panic_on_oops) | - |
| 日志标记 | `Oops:` + `---[ end trace ]---` | `Kernel panic - not syncing:` |

Oops 升级为 Panic 的条件（任一满足）：
1. `panic_on_oops = 1`
2. 在中断上下文中发生 oops
3. idle 进程 (pid=0) 发生 oops
4. init 进程 (pid=1) 发生 oops

---

## 10. WARN → Panic 路径

```
WARN() / WARN_ON() / WARN_ONCE() / ...
  → __warn(file, line, caller, taint, regs, args)  [kernel/panic.c:872]
    → disable_trace_on_warning()
    → 打印警告信息、模块列表、寄存器、调用栈
    → check_panic_on_warn("kernel")                 [kernel/panic.c:372]
      ├── if (panic_on_warn)
      │   → panic("kernel: panic_on_warn set ...")
      └── if (warn_count >= warn_limit && warn_limit > 0)
          → panic("kernel: system warned too often")
    → add_taint(taint, LOCKDEP_STILL_OK)
```

---

## 11. 关键数据结构

```c
// panic CPU 同步
atomic_t panic_cpu = ATOMIC_INIT(PANIC_CPU_INVALID);  // -1

// panic notifier chain
ATOMIC_NOTIFIER_HEAD(panic_notifier_list);

// panic 相关全局变量
int panic_timeout;                 // 超时秒数
int panic_on_oops;                 // oops → panic
int panic_on_warn;                 // warn → panic
unsigned long panic_print;         // 系统信息打印掩码
bool crash_kexec_post_notifiers;   // kdump 时机控制
unsigned long panic_on_taint;      // taint → panic 掩码
bool panic_on_taint_nousertaint;   // 排除用户空间 taint

// 键盘 LED 闪烁回调
long (*panic_blink)(int state);
```

---

## 12. 源码文件索引

| 文件 | 内容 |
|------|------|
| `kernel/panic.c` | panic/vpanic 主逻辑, oops_enter/exit, WARN 处理 |
| `kernel/crash_core.c` | __crash_kexec, crash_kexec, vmcoreinfo |
| `kernel/exit.c` | make_task_dead — oops 后进程退出 |
| `kernel/reboot.c` | emergency_restart |
| `kernel/printk/printk.c` | console_flush_on_panic, printk_legacy_allow_panic_sync |
| `kernel/printk/nbcon.c` | nbcon_atomic_flush_unsafe |
| `include/linux/panic.h` | panic 相关声明、taint 定义 |
| `include/linux/panic_notifier.h` | panic_notifier_list 声明 |
| `include/linux/kmsg_dump.h` | kmsg dumper 接口 |
| `include/linux/sys_info.h` | SYS_INFO_* 掩码定义 |
| `arch/x86/kernel/dumpstack.c` | x86 die/oops_begin/oops_end |
| `arch/x86/kernel/crash.c` | x86 crash_smp_send_stop, machine_crash_shutdown |
| `arch/arm64/kernel/traps.c` | arm64 die() |
| `arch/arm64/kernel/smp.c` | arm64 panic_smp_self_stop, crash_smp_send_stop |
