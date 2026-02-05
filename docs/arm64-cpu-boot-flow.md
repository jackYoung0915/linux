# ARM64 CPU 启动流程分析

## 概述

本文档详细分析 ARM64 架构下 CPU 启动的完整流程，包括 ACPI MADT/PPTT 表的处理、拓扑结构的建立等关键环节。

## 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            ARM64 CPU 启动流程                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │   固件层    │    │  早期启动   │    │  SMP 初始化 │    │  拓扑建立   │      │
│  │  (UEFI/ATF) │───▶│  (head.S)   │───▶│  (smp.c)    │───▶│ (topology)  │      │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘      │
│         │                  │                  │                  │              │
│         ▼                  ▼                  ▼                  ▼              │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐      │
│  │ ACPI Tables │    │ CPU0 启动   │    │ 解析 MADT   │    │ 解析 PPTT   │      │
│  │ MADT/PPTT   │    │ (BSP)       │    │ GICC 条目   │    │ 拓扑/缓存   │      │
│  └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## 详细启动流程

### 阶段 1: 固件层初始化

```
固件 (UEFI/ATF)
    │
    ├─ 初始化所有 CPU 核心
    ├─ 设置 PSCI (Power State Coordination Interface)
    ├─ 准备 ACPI 表 (MADT, PPTT, SRAT 等)
    └─ 将控制权交给内核 (通常在 EL2 或 EL1)
```

### 阶段 2: 内核早期启动 (head.S)

**文件**: `arch/arm64/kernel/head.S`

```
内核入口点
    │
    ├─ 设置初始页表
    ├─ 启用 MMU
    ├─ 设置异常向量
    ├─ 初始化 BSP (Boot CPU)
    └─ 跳转到 start_kernel()
```

### 阶段 3: SMP 初始化

#### 3.1 CPU 枚举 (`smp_init_cpus`)

**文件**: `arch/arm64/kernel/smp.c`

```c
void __init smp_init_cpus(void)
{
    if (acpi_disabled)
        of_parse_and_init_cpus();    // 设备树方式
    else
        acpi_parse_and_init_cpus();  // ACPI 方式
}
```

#### 3.2 ACPI 方式解析 CPU

```c
static void __init acpi_parse_and_init_cpus(void)
{
    // 1. 解析 MADT 表中的 GICC 条目
    acpi_table_parse_madt(ACPI_MADT_TYPE_GENERIC_INTERRUPT,
                          acpi_parse_gic_cpu_interface, 0);
    
    // 2. 将 CPU 映射到 NUMA 节点
    acpi_map_cpus_to_nodes();
    
    // 3. 建立早期 CPU 到节点的映射
    for (i = 0; i < nr_cpu_ids; i++)
        early_map_cpu_to_node(i, acpi_numa_get_nid(i));
}
```

#### 3.3 MADT GICC 条目处理

```c
static void __init acpi_map_gic_cpu_interface(
    struct acpi_madt_generic_interrupt *processor)
{
    u64 hwid = processor->arm_mpidr;  // 获取 MPIDR
    
    // 检查 CPU 是否启用
    if (!(processor->flags & 
          (ACPI_MADT_ENABLED | ACPI_MADT_GICC_ONLINE_CAPABLE)))
        return;
    
    // 验证 MPIDR 有效性
    if (hwid & ~MPIDR_HWID_BITMASK || hwid == INVALID_HWID)
        return;
    
    // 检查是否为 Boot CPU
    if (cpu_logical_map(0) == hwid) {
        bootcpu_valid = true;
        cpu_madt_gicc[0] = *processor;
        return;
    }
    
    // 设置 CPU 逻辑映射
    set_cpu_logical_map(cpu_count, hwid);
    cpu_madt_gicc[cpu_count] = *processor;
    
    // 设置 ACPI parking protocol
    acpi_set_mailbox_entry(cpu_count, processor);
    
    cpu_count++;
}
```

### 阶段 4: CPU 准备与启动

#### 4.1 准备 CPU (`smp_prepare_cpus`)

```c
void __init smp_prepare_cpus(unsigned int max_cpus)
{
    // 1. 初始化 CPU 拓扑
    init_cpu_topology();
    
    // 2. 存储 Boot CPU 拓扑信息
    store_cpu_topology(this_cpu);
    numa_store_cpu_info(this_cpu);
    numa_add_cpu(this_cpu);
    
    // 3. 准备并设置每个可能的 CPU 为 present
    for_each_possible_cpu(cpu) {
        ops = get_cpu_ops(cpu);
        if (ops->cpu_prepare(cpu) == 0)
            set_cpu_present(cpu, true);
    }
}
```

#### 4.2 启动 Secondary CPU (`__cpu_up`)

```c
int __cpu_up(unsigned int cpu, struct task_struct *idle)
{
    // 1. 设置 secondary data
    secondary_data.task = idle;
    update_cpu_boot_status(CPU_MMU_OFF);
    
    // 2. 调用 CPU 操作方法启动 CPU
    ret = boot_secondary(cpu, idle);
    
    // 3. 等待 CPU 上线
    wait_for_completion_timeout(&cpu_running, msecs_to_jiffies(5000));
    
    return cpu_online(cpu) ? 0 : -EIO;
}

static int boot_secondary(unsigned int cpu, struct task_struct *idle)
{
    const struct cpu_operations *ops = get_cpu_ops(cpu);
    
    // 调用具体的启动方法 (PSCI/spin-table/parking-protocol)
    return ops->cpu_boot(cpu);
}
```

#### 4.3 Secondary CPU 入口 (`secondary_start_kernel`)

```c
asmlinkage notrace void secondary_start_kernel(void)
{
    unsigned int cpu = smp_processor_id();
    
    // 1. 设置 mm 上下文
    mmgrab(&init_mm);
    current->active_mm = &init_mm;
    
    // 2. 卸载 identity mapping
    cpu_uninstall_idmap();
    
    // 3. 检查 CPU 能力
    check_local_cpu_capabilities();
    
    // 4. 存储 CPU 信息和拓扑
    cpuinfo_store_cpu();
    store_cpu_topology(cpu);      // ★ 关键：存储拓扑信息
    
    // 5. 通知 CPU 正在启动
    notify_cpu_starting(cpu);
    
    // 6. 设置 IPI
    ipi_setup(cpu);
    
    // 7. 添加到 NUMA
    numa_add_cpu(cpu);
    
    // 8. 标记为在线
    set_cpu_online(cpu, true);
    complete(&cpu_running);
    
    // 9. 进入 idle 循环
    cpu_startup_entry(CPUHP_AP_ONLINE_IDLE);
}
```

## ACPI PPTT 表处理

### PPTT 表结构

PPTT (Processor Properties Topology Table) 描述处理器和缓存拓扑：

```
┌─────────────────────────────────────────────────────────────────┐
│                         PPTT Table                              │
├─────────────────────────────────────────────────────────────────┤
│  Header                                                         │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Processor Hierarchy Node (Package/Socket)               │   │
│  │   ├─ flags: PHYSICAL_PACKAGE                            │   │
│  │   ├─ parent: 0 (root)                                   │   │
│  │   └─ private_resources: [Cache L3]                      │   │
│  │       │                                                 │   │
│  │       ├─ Processor Hierarchy Node (Cluster)             │   │
│  │       │   ├─ flags: ACPI_PROCESSOR_ID_VALID             │   │
│  │       │   ├─ private_resources: [Cache L2]              │   │
│  │       │   │                                             │   │
│  │       │   ├─ Processor Hierarchy Node (Core 0)          │   │
│  │       │   │   ├─ flags: LEAF_NODE                       │   │
│  │       │   │   ├─ acpi_processor_id: 0                   │   │
│  │       │   │   └─ private_resources: [L1I, L1D]          │   │
│  │       │   │                                             │   │
│  │       │   └─ Processor Hierarchy Node (Core 1)          │   │
│  │       │       ├─ flags: LEAF_NODE                       │   │
│  │       │       ├─ acpi_processor_id: 1                   │   │
│  │       │       └─ private_resources: [L1I, L1D]          │   │
│  │       │                                                 │   │
│  │       └─ ...                                            │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ Cache Type Structure (L3)                               │   │
│  │   ├─ size, line_size, associativity                     │   │
│  │   ├─ attributes: UNIFIED                                │   │
│  │   └─ next_level_of_cache: 0                             │   │
│  └─────────────────────────────────────────────────────────┘   │
│  ...                                                           │
└─────────────────────────────────────────────────────────────────┘
```

### PPTT 处理函数

**文件**: `drivers/acpi/pptt.c`

#### 查找处理器节点

```c
static struct acpi_pptt_processor *acpi_find_processor_node(
    struct acpi_table_header *table_hdr,
    u32 acpi_cpu_id)
{
    // 遍历 PPTT 表查找匹配的处理器节点
    while ((entry + proc_sz) <= table_end) {
        cpu_node = (struct acpi_pptt_processor *)entry;
        
        if (entry->type == ACPI_PPTT_TYPE_PROCESSOR &&
            acpi_cpu_id == cpu_node->acpi_processor_id &&
            acpi_pptt_leaf_node(table_hdr, cpu_node)) {
            return cpu_node;
        }
        entry = ACPI_ADD_PTR(...);
    }
    return NULL;
}
```

#### 获取拓扑信息

```c
// 查找 Package ID
int find_acpi_cpu_topology_package(unsigned int cpu)
{
    return find_acpi_cpu_topology_tag(cpu, PPTT_ABORT_PACKAGE,
                                      ACPI_PPTT_PHYSICAL_PACKAGE);
}

// 查找 Cluster ID
int find_acpi_cpu_topology_cluster(unsigned int cpu)
{
    // 找到 CPU 节点的父节点（对于 SMT 线程，需要再上一层）
    cpu_node = acpi_find_processor_node(table, acpi_cpu_id);
    is_thread = cpu_node->flags & ACPI_PPTT_ACPI_PROCESSOR_IS_THREAD;
    
    cluster_node = fetch_pptt_node(table, cpu_node->parent);
    if (is_thread)
        cluster_node = fetch_pptt_node(table, cluster_node->parent);
    
    return cluster_node->acpi_processor_id;
}

// 判断是否为 SMT 线程
int acpi_pptt_cpu_is_thread(unsigned int cpu)
{
    return check_acpi_cpu_flag(cpu, 2, ACPI_PPTT_ACPI_PROCESSOR_IS_THREAD);
}
```

## 拓扑结构建立

### 初始化拓扑 (`init_cpu_topology`)

**文件**: `drivers/base/arch_topology.c`

```c
void __init init_cpu_topology(void)
{
    // 重置所有 CPU 的拓扑信息
    reset_cpu_topology();
    
    // 根据启动方式选择解析方法
    if (acpi_disabled)
        parse_dt_topology();        // 设备树
    else
        parse_acpi_topology();      // ACPI
}
```

### ACPI 拓扑解析

```c
static int __init parse_acpi_topology(void)
{
    // 检查是否支持 SMT
    if (acpi_pptt_cpu_is_thread(0) > 0)
        cpu_smt_set_num_threads(2, 2);
    
    return 0;
}
```

### 存储 CPU 拓扑 (`store_cpu_topology`)

**文件**: `drivers/base/arch_topology.c`

```c
void store_cpu_topology(unsigned int cpuid)
{
    struct cpu_topology *cpuid_topo = &cpu_topology[cpuid];
    
    // 如果已经设置过，直接返回
    if (cpuid_topo->package_id != -1)
        goto topology_populated;
    
    // 从 ACPI PPTT 获取拓扑信息
    cpuid_topo->thread_id  = find_acpi_cpu_topology(cpuid, 0);  // 线程 ID
    cpuid_topo->core_id    = find_acpi_cpu_topology(cpuid, 1);  // 核心 ID
    cpuid_topo->cluster_id = find_acpi_cpu_topology_cluster(cpuid);
    cpuid_topo->package_id = find_acpi_cpu_topology_package(cpuid);
    
topology_populated:
    // 更新 sibling 掩码
    update_siblings_masks(cpuid);
}
```

### 更新 Sibling 掩码

```c
void update_siblings_masks(unsigned int cpuid)
{
    struct cpu_topology *cpu_topo = &cpu_topology[cpuid];
    
    for_each_online_cpu(cpu) {
        struct cpu_topology *cpu_topo_other = &cpu_topology[cpu];
        
        // 更新 core siblings (同一 package 内的所有核心)
        if (cpu_topo->package_id == cpu_topo_other->package_id) {
            cpumask_set_cpu(cpuid, &cpu_topo_other->core_sibling);
            cpumask_set_cpu(cpu, &cpu_topo->core_sibling);
        }
        
        // 更新 cluster siblings
        if (cpu_topo->cluster_id == cpu_topo_other->cluster_id &&
            cpu_topo->package_id == cpu_topo_other->package_id) {
            cpumask_set_cpu(cpuid, &cpu_topo_other->cluster_sibling);
            cpumask_set_cpu(cpu, &cpu_topo->cluster_sibling);
        }
        
        // 更新 thread siblings (同一核心内的所有线程)
        if (cpu_topo->core_id == cpu_topo_other->core_id &&
            cpu_topo->cluster_id == cpu_topo_other->cluster_id &&
            cpu_topo->package_id == cpu_topo_other->package_id) {
            cpumask_set_cpu(cpuid, &cpu_topo_other->thread_sibling);
            cpumask_set_cpu(cpu, &cpu_topo->thread_sibling);
        }
    }
}
```

## CPU 启动方法

ARM64 支持多种 CPU 启动方法：

### 1. PSCI (Power State Coordination Interface)

**文件**: `arch/arm64/kernel/psci.c`

```c
static int cpu_psci_cpu_boot(unsigned int cpu)
{
    phys_addr_t pa_secondary_entry = __pa_symbol(secondary_entry);
    int err;
    
    // 调用 PSCI CPU_ON 命令
    err = psci_ops.cpu_on(cpu_logical_map(cpu), pa_secondary_entry);
    
    return err;
}

static const struct cpu_operations cpu_psci_ops = {
    .name       = "psci",
    .cpu_init   = cpu_psci_cpu_init,
    .cpu_prepare = cpu_psci_cpu_prepare,
    .cpu_boot   = cpu_psci_cpu_boot,
    .cpu_die    = cpu_psci_cpu_die,
    .cpu_kill   = cpu_psci_cpu_kill,
};
```

### 2. Spin Table

**文件**: `arch/arm64/kernel/smp_spin_table.c`

```c
static int smp_spin_table_cpu_boot(unsigned int cpu)
{
    // 写入 release 地址
    u64 *release_addr = phys_to_virt(cpu_release_addr[cpu]);
    
    // 写入 secondary entry 地址
    writeq_relaxed(__pa_symbol(secondary_holding_pen), release_addr);
    
    // 发送 SEV 唤醒 CPU
    sev();
    
    return 0;
}
```

### 3. ACPI Parking Protocol

**文件**: `arch/arm64/kernel/acpi_parking_protocol.c`

```c
static int acpi_parking_protocol_cpu_boot(unsigned int cpu)
{
    struct parking_protocol_mailbox *mailbox;
    
    // 写入 entry point 到 mailbox
    mailbox->entry_point = __pa_symbol(secondary_entry);
    mailbox->cpu_id = cpu_logical_map(cpu);
    
    // 发送 wakeup IPI
    arch_send_wakeup_ipi(cpu);
    
    return 0;
}
```

## 完整调用流程图

```
start_kernel()
    │
    ├─ setup_arch()
    │     └─ smp_init_cpus()                    // 枚举 CPU
    │           ├─ acpi_parse_and_init_cpus()   // 解析 MADT
    │           │     ├─ acpi_parse_gic_cpu_interface()
    │           │     └─ acpi_map_cpus_to_nodes()
    │           └─ smp_cpu_setup()              // 初始化每个 CPU
    │                 └─ init_cpu_ops()          // 设置启动方法
    │
    ├─ smp_prepare_boot_cpu()
    │     ├─ cpuinfo_store_boot_cpu()
    │     └─ setup_boot_cpu_features()
    │
    └─ smp_init()
          └─ smp_prepare_cpus()                  // 准备所有 CPU
          │     ├─ init_cpu_topology()           // ★ 初始化拓扑
          │     │     └─ parse_acpi_topology()
          │     │           └─ acpi_pptt_cpu_is_thread()
          │     ├─ store_cpu_topology(0)         // Boot CPU 拓扑
          │     └─ ops->cpu_prepare()            // 准备每个 CPU
          │
          └─ cpu_up(cpu) [for each secondary CPU]
                └─ __cpu_up()
                      └─ boot_secondary()
                            └─ ops->cpu_boot()   // PSCI/spin-table
                                  │
                                  ▼
                            [Secondary CPU]
                            secondary_start_kernel()
                                  ├─ cpuinfo_store_cpu()
                                  ├─ store_cpu_topology() // ★ 存储拓扑
                                  │     ├─ find_acpi_cpu_topology()
                                  │     ├─ find_acpi_cpu_topology_cluster()
                                  │     ├─ find_acpi_cpu_topology_package()
                                  │     └─ update_siblings_masks()
                                  ├─ notify_cpu_starting()
                                  ├─ numa_add_cpu()
                                  └─ set_cpu_online()
```

## 关键数据结构

### cpu_topology 结构体

```c
struct cpu_topology {
    int thread_id;          // SMT 线程 ID
    int core_id;            // 核心 ID
    int cluster_id;         // Cluster ID
    int package_id;         // Package/Socket ID
    
    cpumask_t thread_sibling;   // 同核心的线程
    cpumask_t core_sibling;     // 同 Package 的核心
    cpumask_t cluster_sibling;  // 同 Cluster 的核心
    cpumask_t llc_sibling;      // 共享 LLC 的核心
};

DEFINE_PER_CPU(struct cpu_topology, cpu_topology);
```

### MADT GICC 结构体

```c
struct acpi_madt_generic_interrupt {
    struct acpi_subtable_header header;
    u16 reserved;
    u32 cpu_interface_number;
    u32 acpi_processor_id;      // ACPI Processor UID
    u32 flags;                   // ENABLED, ONLINE_CAPABLE
    u32 parking_protocol_version;
    u32 performance_interrupt;
    u64 parked_address;
    u64 base_address;
    u64 gicv_base_address;
    u64 gich_base_address;
    u32 vgic_interrupt;
    u64 gicr_base_address;
    u64 arm_mpidr;              // ★ MPIDR 寄存器值
    u8 efficiency_class;
    u8 reserved2[1];
    u16 spe_interrupt;
    u16 trbe_interrupt;
};
```

### PPTT 处理器节点结构体

```c
struct acpi_pptt_processor {
    struct acpi_subtable_header header;
    u16 reserved;
    u32 flags;                    // PHYSICAL_PACKAGE, LEAF_NODE, etc.
    u32 parent;                   // 父节点偏移
    u32 acpi_processor_id;        // ACPI Processor UID
    u32 number_of_priv_resources; // 私有资源数量
    // 后面跟着 private_resources[] 数组
};
```

## 调试与验证

### 查看 CPU 拓扑信息

```bash
# 查看 CPU 拓扑
lscpu -e

# 查看详细拓扑
cat /sys/devices/system/cpu/cpu*/topology/thread_siblings_list
cat /sys/devices/system/cpu/cpu*/topology/core_siblings_list
cat /sys/devices/system/cpu/cpu*/topology/cluster_id
cat /sys/devices/system/cpu/cpu*/topology/physical_package_id
```

### 查看 ACPI 表

```bash
# 导出 ACPI 表
acpidump > acpi.dat
acpixtract -a acpi.dat

# 反汇编 PPTT 表
iasl -d pptt.dat
```

### 内核启动日志

```bash
dmesg | grep -E "(ACPI|PPTT|topology|CPU|smp)"
```

预期输出：
```
ACPI: PPTT 0x00000000XXXXXXXX 000XXX (v02 ...)
ACPI PPTT: CPU 0 package 0 cluster 0 core 0 thread 0
ACPI PPTT: CPU 1 package 0 cluster 0 core 0 thread 1
ACPI PPTT: CPU 2 package 0 cluster 0 core 1 thread 0
...
CPU: All CPU(s) started at EL2
SMP: Total of 8 processors activated.
```

## 总结

ARM64 CPU 启动流程的关键步骤：

| 阶段 | 关键函数 | 作用 |
|------|----------|------|
| 1. CPU 枚举 | `smp_init_cpus()` | 解析 MADT，建立 cpu_logical_map |
| 2. 拓扑初始化 | `init_cpu_topology()` | 初始化拓扑数据结构 |
| 3. CPU 准备 | `smp_prepare_cpus()` | 准备启动所有 Secondary CPU |
| 4. CPU 启动 | `__cpu_up()` → `boot_secondary()` | 通过 PSCI 等方式启动 CPU |
| 5. 拓扑存储 | `store_cpu_topology()` | 从 PPTT 获取并存储拓扑信息 |
| 6. CPU 上线 | `set_cpu_online()` | 标记 CPU 为在线状态 |

ACPI 表的作用：
- **MADT**: 提供 CPU 的硬件信息（MPIDR、GIC 配置等）
- **PPTT**: 提供处理器层次结构和缓存拓扑信息
- **SRAT**: 提供 NUMA 亲和性信息
