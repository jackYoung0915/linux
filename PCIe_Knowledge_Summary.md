# PCIe 知识要点与 Linux 内核设备加载流程

> 分享人：Linux 内核开发工程师  
> 日期：2026年1月

---

## 目录

1. [PCIe 基础概念](#1-pcie-基础概念)
2. [PCIe 拓扑结构](#2-pcie-拓扑结构)
3. [PCIe 配置空间](#3-pcie-配置空间)
4. [PCIe 事务层协议](#4-pcie-事务层协议)
5. [Linux 内核 PCIe 子系统架构](#5-linux-内核-pcie-子系统架构)
6. [PCIe 设备枚举流程](#6-pcie-设备枚举流程)
7. [PCIe 驱动加载流程](#7-pcie-驱动加载流程)
8. [关键数据结构](#8-关键数据结构)
9. [实践示例](#9-实践示例)

---

## 1. PCIe 基础概念

### 1.1 什么是 PCIe？

**PCIe (PCI Express)** 是一种高速串行计算机扩展总线标准，用于连接主板与各种外设（显卡、网卡、SSD等）。

```
┌─────────────────────────────────────────────────────────────────┐
│                    PCIe 发展历程                                  │
├─────────────────────────────────────────────────────────────────┤
│  PCI (1992)  →  PCI-X (1998)  →  PCIe 1.0 (2003)               │
│                                        ↓                        │
│  PCIe 6.0 (2022) ← PCIe 5.0 (2019) ← ... ← PCIe 2.0 (2007)    │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 PCIe vs PCI 对比

| 特性 | PCI | PCIe |
|------|-----|------|
| **连接方式** | 并行总线（共享） | 点对点串行链路 |
| **传输方向** | 半双工 | 全双工 |
| **带宽** | 共享 133MB/s (32位) | 独享，可扩展 (x1~x32) |
| **热插拔** | 有限支持 | 原生支持 |
| **电源管理** | 基础 | 高级 ASPM |

### 1.3 PCIe 带宽速率

```
┌────────────────────────────────────────────────────────────────┐
│                    PCIe 各版本带宽对比                           │
├──────────┬────────────┬───────────┬───────────┬───────────────┤
│  版本    │ 传输速率    │  编码     │  x1 带宽   │  x16 带宽     │
├──────────┼────────────┼───────────┼───────────┼───────────────┤
│ PCIe 1.0 │ 2.5 GT/s   │ 8b/10b    │ 250 MB/s  │ 4 GB/s        │
│ PCIe 2.0 │ 5.0 GT/s   │ 8b/10b    │ 500 MB/s  │ 8 GB/s        │
│ PCIe 3.0 │ 8.0 GT/s   │ 128b/130b │ 984 MB/s  │ 15.75 GB/s    │
│ PCIe 4.0 │ 16.0 GT/s  │ 128b/130b │ 1.97 GB/s │ 31.5 GB/s     │
│ PCIe 5.0 │ 32.0 GT/s  │ 128b/130b │ 3.94 GB/s │ 63 GB/s       │
│ PCIe 6.0 │ 64.0 GT/s  │ 1b/1b+FEC │ 7.88 GB/s │ 126 GB/s      │
└──────────┴────────────┴───────────┴───────────┴───────────────┘

注：GT/s = Giga Transfers per second (每秒十亿次传输)
```

### 1.4 Lane（通道）概念

```
                        PCIe Link 结构
    ┌─────────────────────────────────────────────────────┐
    │                                                     │
    │   Device A                           Device B       │
    │   ┌──────┐                           ┌──────┐      │
    │   │      │  ════════ TX ══════════►  │      │      │
    │   │      │                           │      │      │  x1 Link
    │   │      │  ◄════════ RX ══════════  │      │      │  (1 Lane)
    │   └──────┘                           └──────┘      │
    │                                                     │
    └─────────────────────────────────────────────────────┘

    ┌─────────────────────────────────────────────────────┐
    │   Device A                           Device B       │
    │   ┌──────┐  ════════ Lane 0 TX ═══►  ┌──────┐      │
    │   │      │  ◄═══════ Lane 0 RX ════  │      │      │
    │   │      │  ════════ Lane 1 TX ═══►  │      │      │  x4 Link
    │   │      │  ◄═══════ Lane 1 RX ════  │      │      │  (4 Lanes)
    │   │      │  ════════ Lane 2 TX ═══►  │      │      │
    │   │      │  ◄═══════ Lane 2 RX ════  │      │      │
    │   │      │  ════════ Lane 3 TX ═══►  │      │      │
    │   │      │  ◄═══════ Lane 3 RX ════  │      │      │
    │   └──────┘                           └──────┘      │
    └─────────────────────────────────────────────────────┘
```

---

## 2. PCIe 拓扑结构

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           CPU / Memory                                   │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │     Root Complex (RC)    │
                    │   (PCIe 根复合体)         │
                    └──┬─────────┬─────────┬──┘
                       │         │         │
           ┌───────────▼─┐  ┌────▼────┐  ┌─▼───────────┐
           │ Root Port 0 │  │ Root    │  │ Root Port 2 │
           │   (RP)      │  │ Port 1  │  │    (RP)     │
           └──────┬──────┘  └────┬────┘  └──────┬──────┘
                  │              │              │
           ┌──────▼──────┐      │       ┌──────▼──────┐
           │  Endpoint   │      │       │   Switch    │
           │  (显卡)     │      │       │   (交换机)   │
           └─────────────┘      │       └──┬─────┬────┘
                                │          │     │
                         ┌──────▼────┐ ┌───▼─┐ ┌─▼───┐
                         │  Endpoint │ │ EP  │ │ EP  │
                         │  (NVMe)   │ │     │ │     │
                         └───────────┘ └─────┘ └─────┘
```

### 2.2 核心组件说明

| 组件 | 英文名称 | 功能描述 |
|------|----------|----------|
| **Root Complex** | RC | 连接 CPU/Memory 与 PCIe 总线的桥梁 |
| **Root Port** | RP | RC 上的 PCIe 端口，连接下游设备 |
| **Switch** | Switch | PCIe 交换机，扩展端口数量 |
| **Endpoint** | EP | 终端设备（显卡、网卡、NVMe等） |
| **Bridge** | Bridge | PCI-PCIe 桥接器 |

### 2.3 PCIe Switch 内部结构

```
                    ┌─────────────────────────────────┐
                    │         PCIe Switch             │
                    │  ┌──────────────────────────┐   │
                    │  │    Internal Bus (虚拟)    │   │
    ┌───────────────┤  └──────────────────────────┘   │
    │               │     │           │           │   │
    │  Upstream     │  ┌──▼──┐    ┌───▼──┐    ┌──▼──┐│
    │  Port         │  │ DP0 │    │ DP1  │    │ DP2 ││
    │               │  └──┬──┘    └───┬──┘    └──┬──┘│
    └───────────────┤     │           │           │   │
                    └─────┼───────────┼───────────┼───┘
                          │           │           │
                    ┌─────▼───┐ ┌─────▼───┐ ┌─────▼───┐
                    │   EP    │ │   EP    │ │   EP    │
                    └─────────┘ └─────────┘ └─────────┘
```

---

## 3. PCIe 配置空间

### 3.1 配置空间结构

PCIe 设备有 4KB 配置空间（PCI 只有 256B）：

```
    地址偏移
    ┌────────────────────────────────────────┐  0x000
    │                                        │
    │         PCI 兼容配置空间                │
    │           (256 Bytes)                  │
    │                                        │
    │  ┌──────────────────────────────────┐  │  0x000
    │  │  Vendor ID (2B) │ Device ID (2B) │  │
    │  ├──────────────────────────────────┤  │  0x004
    │  │  Command (2B)   │ Status (2B)    │  │
    │  ├──────────────────────────────────┤  │  0x008
    │  │  Class Code (3B)│ Revision (1B)  │  │
    │  ├──────────────────────────────────┤  │  0x00C
    │  │  BIST │ Header │ Latency │ Cache │  │
    │  ├──────────────────────────────────┤  │  0x010
    │  │         BAR0 (Base Address)      │  │
    │  ├──────────────────────────────────┤  │  0x014
    │  │         BAR1                     │  │
    │  ├──────────────────────────────────┤  │  ...
    │  │         BAR2 ~ BAR5              │  │
    │  ├──────────────────────────────────┤  │  0x034
    │  │    Capabilities Pointer          │  │
    │  ├──────────────────────────────────┤  │  0x03C
    │  │  IRQ Line │ IRQ Pin │ ...        │  │
    │  └──────────────────────────────────┘  │
    │                                        │
    ├────────────────────────────────────────┤  0x100
    │                                        │
    │      PCIe 扩展配置空间                  │
    │      (Extended Configuration)          │
    │           (3840 Bytes)                 │
    │                                        │
    │   - PCIe Capability                    │
    │   - MSI/MSI-X Capability               │
    │   - Power Management                   │
    │   - AER (Advanced Error Reporting)     │
    │   - SR-IOV                             │
    │   - ...                                │
    │                                        │
    └────────────────────────────────────────┘  0xFFF
```

### 3.2 重要寄存器说明

```c
/* 配置空间偏移定义 (include/uapi/linux/pci_regs.h) */

#define PCI_VENDOR_ID           0x00    /* 厂商ID */
#define PCI_DEVICE_ID           0x02    /* 设备ID */
#define PCI_COMMAND             0x04    /* 命令寄存器 */
#define PCI_STATUS              0x06    /* 状态寄存器 */
#define PCI_CLASS_REVISION      0x08    /* 类代码和版本 */
#define PCI_BASE_ADDRESS_0      0x10    /* BAR0 */
#define PCI_INTERRUPT_LINE      0x3c    /* 中断线 */
#define PCI_INTERRUPT_PIN       0x3d    /* 中断引脚 */
```

### 3.3 BAR (Base Address Register) 解析

```
    BAR 寄存器格式 (Memory BAR)
    ┌─────────────────────────────────────────────────────────┐
    │ 31                              4  3   2   1   0        │
    │ ├──────────────────────────────────┼───┼───┼───┼───┤   │
    │ │        Base Address              │Pre│Type │ 0 │     │
    │ │        (对齐后的基地址)           │fch│    │   │     │
    │ └──────────────────────────────────┴───┴───┴───┴───┘   │
    │                                                         │
    │  Bit 0: 0 = Memory Space, 1 = I/O Space                │
    │  Bit 2-1: Type (00=32位, 10=64位)                       │
    │  Bit 3: Prefetchable                                    │
    └─────────────────────────────────────────────────────────┘
```

---

## 4. PCIe 事务层协议

### 4.1 PCIe 分层架构

```
    ┌─────────────────────────────────────────────────────────┐
    │                   Software Layer                        │
    │               (驱动程序 / 应用程序)                       │
    └─────────────────────────────────────────────────────────┘
                              ↕
    ╔═════════════════════════════════════════════════════════╗
    ║               Transaction Layer (事务层)                 ║
    ║  - TLP (Transaction Layer Packet) 生成/解析             ║
    ║  - 流控制 (Flow Control)                                ║
    ║  - 虚拟通道 (Virtual Channels)                          ║
    ╚═════════════════════════════════════════════════════════╝
                              ↕
    ╔═════════════════════════════════════════════════════════╗
    ║                Data Link Layer (数据链路层)              ║
    ║  - DLLP (Data Link Layer Packet)                        ║
    ║  - ACK/NAK 协议                                         ║
    ║  - CRC 校验                                             ║
    ║  - 重传机制                                              ║
    ╚═════════════════════════════════════════════════════════╝
                              ↕
    ╔═════════════════════════════════════════════════════════╗
    ║                Physical Layer (物理层)                   ║
    ║  - 编码 (8b/10b, 128b/130b)                             ║
    ║  - 串行化/解串行化                                       ║
    ║  - 链路训练 (Link Training)                             ║
    ╚═════════════════════════════════════════════════════════╝
```

### 4.2 TLP (Transaction Layer Packet) 类型

```
┌─────────────────────────────────────────────────────────────────┐
│                        TLP 类型分类                              │
├─────────────────┬───────────────────────────────────────────────┤
│     类型        │                  描述                         │
├─────────────────┼───────────────────────────────────────────────┤
│ Memory Read     │ 内存读请求                                    │
│ Memory Write    │ 内存写请求                                    │
│ I/O Read        │ I/O 端口读                                    │
│ I/O Write       │ I/O 端口写                                    │
│ Config Read     │ 配置空间读 (Type 0/1)                         │
│ Config Write    │ 配置空间写 (Type 0/1)                         │
│ Message         │ 消息传递 (中断、电源管理等)                    │
│ Completion      │ 完成包 (对读请求的响应)                        │
└─────────────────┴───────────────────────────────────────────────┘
```

### 4.3 TLP 包结构

```
    ┌────────────────────────────────────────────────────────────┐
    │                    TLP Packet Structure                    │
    ├────────────────────────────────────────────────────────────┤
    │  ┌──────────────────────────────────────────────────────┐  │
    │  │              Header (3-4 DW)                         │  │
    │  │  ┌────────┬────────┬────────┬────────┐               │  │
    │  │  │ Fmt/   │ TC │   │ Length │ Req ID │ ...           │  │
    │  │  │ Type   │    │   │        │        │               │  │
    │  │  └────────┴────────┴────────┴────────┘               │  │
    │  └──────────────────────────────────────────────────────┘  │
    │  ┌──────────────────────────────────────────────────────┐  │
    │  │              Data Payload (0-1024 DW)                │  │
    │  │                    (可选)                             │  │
    │  └──────────────────────────────────────────────────────┘  │
    │  ┌──────────────────────────────────────────────────────┐  │
    │  │              ECRC (可选, 1 DW)                        │  │
    │  └──────────────────────────────────────────────────────┘  │
    └────────────────────────────────────────────────────────────┘

    注: DW = Double Word = 4 Bytes
```

---

## 5. Linux 内核 PCIe 子系统架构

### 5.1 内核 PCIe 目录结构

```
drivers/pci/
├── access.c          # 配置空间访问
├── bus.c             # 总线管理
├── probe.c           # 设备枚举探测  ← 核心文件
├── pci-driver.c      # PCI 驱动框架  ← 核心文件
├── pci.c             # PCI 核心功能
├── quirks.c          # 硬件兼容处理
├── msi/              # MSI/MSI-X 中断
│   ├── api.c
│   ├── irqdomain.c
│   └── msi.c
├── pcie/             # PCIe 特定功能
│   ├── portdrv.c     # PCIe Port 驱动
│   ├── aer.c         # 高级错误报告
│   ├── aspm.c        # 活动状态电源管理
│   └── dpc.c         # 下游端口遏制
├── hotplug/          # 热插拔支持
└── controller/       # Host Controller 驱动
```

### 5.2 核心数据结构关系

```
                    ┌───────────────────┐
                    │  pci_host_bridge  │
                    │   (Host 控制器)    │
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
         ┌─────────│     pci_bus       │─────────┐
         │         │   (PCI 总线)       │         │
         │         └─────────┬─────────┘         │
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│    pci_dev      │  │    pci_dev      │  │    pci_dev      │
│   (PCI 设备)     │  │   (Bridge)      │  │   (PCI 设备)     │
└────────┬────────┘  └────────┬────────┘  └─────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────┐  ┌─────────────────┐
│   pci_driver    │  │    pci_bus      │
│  (设备驱动)      │  │   (子总线)       │
└─────────────────┘  └────────┬────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │    pci_dev      │
                     │  (下游设备)      │
                     └─────────────────┘
```

---

## 6. PCIe 设备枚举流程

### 6.1 枚举总体流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                     PCIe 设备枚举流程图                               │
└──────────────────────────────────────────────────────────────────────┘

    内核启动
        │
        ▼
┌───────────────────┐
│ pci_host_probe()  │  ← Host Controller 驱动调用
└─────────┬─────────┘
          │
          ▼
┌─────────────────────────┐
│ pci_scan_root_bus_      │
│      bridge()           │  ← 扫描根总线
└─────────┬───────────────┘
          │
          ▼
┌─────────────────────────┐
│  pci_scan_child_bus()   │  ← 扫描子总线
└─────────┬───────────────┘
          │
          ├──────────────────────────────────────┐
          │                                      │
          ▼                                      ▼
┌──────────────────────┐              ┌──────────────────────┐
│ pci_scan_slot()      │              │ (递归扫描)            │
│  对每个 slot 扫描     │              │  如果是 Bridge,       │
└─────────┬────────────┘              │  继续扫描下游总线     │
          │                           └──────────────────────┘
          ▼
┌──────────────────────┐
│ pci_scan_single_     │
│     device()         │
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐     ┌────────────────────────────────┐
│ pci_scan_device()    │────►│  读取 Vendor ID / Device ID    │
└─────────┬────────────┘     │  判断设备是否存在               │
          │                  └────────────────────────────────┘
          ▼
┌──────────────────────┐
│ pci_setup_device()   │  ← 解析配置空间，填充 pci_dev 结构
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│ pci_device_add()     │  ← 添加到设备模型
└─────────┬────────────┘
          │
          ▼
┌──────────────────────┐
│ pci_bus_add_devices()│  ← 触发驱动匹配
└──────────────────────┘
```

### 6.2 关键代码路径

```c
/* drivers/pci/probe.c - 设备扫描核心函数 */

static struct pci_dev *pci_scan_device(struct pci_bus *bus, int devfn)
{
    struct pci_dev *dev;
    u32 l;

    /* 1. 读取 Vendor ID 和 Device ID */
    if (!pci_bus_read_dev_vendor_id(bus, devfn, &l, 60*1000))
        return NULL;   /* 设备不存在 */

    /* 2. 分配 pci_dev 结构 */
    dev = pci_alloc_dev(bus);
    if (!dev)
        return NULL;

    dev->devfn = devfn;
    dev->vendor = l & 0xffff;
    dev->device = (l >> 16) & 0xffff;

    /* 3. 读取完整配置空间，初始化设备 */
    if (pci_setup_device(dev)) {
        kfree(dev);
        return NULL;
    }

    return dev;
}
```

### 6.3 BDF 地址编码

```
    ┌─────────────────────────────────────────────────────────┐
    │              BDF (Bus:Device.Function) 地址              │
    ├─────────────────────────────────────────────────────────┤
    │                                                         │
    │   Example: 0000:03:00.0                                 │
    │                                                         │
    │   ┌────────┬────────┬────────┬──────────┐              │
    │   │ Domain │  Bus   │ Device │ Function │              │
    │   │ (16位) │ (8位)  │  (5位) │  (3位)   │              │
    │   ├────────┼────────┼────────┼──────────┤              │
    │   │  0000  │   03   │   00   │    0     │              │
    │   └────────┴────────┴────────┴──────────┘              │
    │                                                         │
    │   配置空间地址计算:                                      │
    │   Address = Base + (Bus << 20) + (Dev << 15) +         │
    │             (Func << 12) + Register                     │
    │                                                         │
    └─────────────────────────────────────────────────────────┘
```

---

## 7. PCIe 驱动加载流程

### 7.1 驱动注册与匹配流程

```
┌──────────────────────────────────────────────────────────────────────┐
│                     PCI 驱动加载流程                                  │
└──────────────────────────────────────────────────────────────────────┘

      驱动模块                                    设备枚举
          │                                          │
          ▼                                          ▼
┌───────────────────┐                    ┌───────────────────┐
│ module_init()     │                    │ pci_device_add()  │
│ 模块初始化         │                    │                   │
└─────────┬─────────┘                    └─────────┬─────────┘
          │                                        │
          ▼                                        ▼
┌───────────────────┐                    ┌───────────────────┐
│pci_register_      │                    │ device_add()      │
│   driver()        │                    │                   │
└─────────┬─────────┘                    └─────────┬─────────┘
          │                                        │
          │         ┌─────────────────┐            │
          └────────►│   PCI Core      │◄───────────┘
                    │                 │
                    │ pci_bus_match() │
                    │ 匹配 device_id  │
                    └────────┬────────┘
                             │
                    匹配成功? │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
               匹配成功           匹配失败
                    │                 │
                    ▼                 ▼
          ┌─────────────────┐    设备等待
          │ pci_device_     │    驱动加载
          │    probe()      │
          └────────┬────────┘
                   │
                   ▼
          ┌─────────────────┐
          │ drv->probe()    │  ← 调用驱动的 probe 函数
          │ 驱动初始化设备   │
          └─────────────────┘
```

### 7.2 驱动结构定义

```c
/* 典型的 PCI 驱动结构 */

/* 1. 设备 ID 表 - 声明驱动支持的设备 */
static const struct pci_device_id my_pci_ids[] = {
    { PCI_DEVICE(0x8086, 0x1234) },  /* Intel 某设备 */
    { PCI_DEVICE(0x10de, 0x5678) },  /* NVIDIA 某设备 */
    { PCI_DEVICE_CLASS(PCI_CLASS_NETWORK_ETHERNET << 8, ~0) },
    { 0, }  /* 结束标志 */
};
MODULE_DEVICE_TABLE(pci, my_pci_ids);

/* 2. 驱动结构体 */
static struct pci_driver my_pci_driver = {
    .name       = "my_pci_driver",
    .id_table   = my_pci_ids,
    .probe      = my_probe,      /* 设备探测 */
    .remove     = my_remove,     /* 设备移除 */
    .suspend    = my_suspend,    /* 电源管理 */
    .resume     = my_resume,
    .shutdown   = my_shutdown,
    .err_handler = &my_err_handler,  /* 错误处理 */
};

/* 3. 注册驱动 */
module_pci_driver(my_pci_driver);
```

### 7.3 Probe 函数典型实现

```c
static int my_probe(struct pci_dev *pdev, const struct pci_device_id *id)
{
    int ret;
    void __iomem *mmio_base;

    /* Step 1: 启用 PCI 设备 */
    ret = pci_enable_device(pdev);
    if (ret)
        return ret;

    /* Step 2: 请求 MMIO 区域 */
    ret = pci_request_regions(pdev, "my_driver");
    if (ret)
        goto err_disable;

    /* Step 3: 映射 BAR 空间 */
    mmio_base = pci_iomap(pdev, 0, 0);  /* 映射 BAR0 */
    if (!mmio_base) {
        ret = -ENOMEM;
        goto err_release;
    }

    /* Step 4: 设置 DMA 掩码 */
    ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(64));
    if (ret) {
        ret = dma_set_mask_and_coherent(&pdev->dev, DMA_BIT_MASK(32));
        if (ret)
            goto err_unmap;
    }

    /* Step 5: 启用 Bus Master */
    pci_set_master(pdev);

    /* Step 6: 请求中断 (MSI/MSI-X) */
    ret = pci_alloc_irq_vectors(pdev, 1, 16, PCI_IRQ_MSIX | PCI_IRQ_MSI);
    if (ret < 0)
        goto err_unmap;

    /* Step 7: 注册中断处理函数 */
    ret = request_irq(pci_irq_vector(pdev, 0), my_irq_handler,
                      0, "my_driver", pdev);
    if (ret)
        goto err_free_irq_vectors;

    /* Step 8: 设备特定初始化 */
    /* ... */

    dev_info(&pdev->dev, "Device initialized successfully\n");
    return 0;

err_free_irq_vectors:
    pci_free_irq_vectors(pdev);
err_unmap:
    pci_iounmap(pdev, mmio_base);
err_release:
    pci_release_regions(pdev);
err_disable:
    pci_disable_device(pdev);
    return ret;
}
```

### 7.4 设备初始化步骤图

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PCI 设备初始化步骤                                │
└─────────────────────────────────────────────────────────────────────┘

    ┌─────────────────┐
    │ 1. Enable Device │  pci_enable_device()
    │    唤醒设备      │  - 将设备从 D3 唤醒到 D0
    └────────┬────────┘  - 分配 I/O 和 Memory 资源
             │
             ▼
    ┌─────────────────┐
    │ 2. Request      │  pci_request_regions()
    │    Regions      │  - 独占访问 BAR 区域
    └────────┬────────┘  - 防止资源冲突
             │
             ▼
    ┌─────────────────┐
    │ 3. Map BAR      │  pci_iomap() / ioremap()
    │    映射 BAR     │  - 将物理地址映射到虚拟地址
    └────────┬────────┘  - 获得 void __iomem * 指针
             │
             ▼
    ┌─────────────────┐
    │ 4. Set DMA Mask │  dma_set_mask_and_coherent()
    │    设置DMA掩码   │  - 指定设备 DMA 地址能力
    └────────┬────────┘  - 32位 or 64位
             │
             ▼
    ┌─────────────────┐
    │ 5. Enable       │  pci_set_master()
    │    Bus Master   │  - 允许设备发起 DMA 传输
    └────────┬────────┘
             │
             ▼
    ┌─────────────────┐
    │ 6. Setup IRQ    │  pci_alloc_irq_vectors()
    │    配置中断      │  request_irq()
    └────────┬────────┘  - 配置 MSI/MSI-X 或 Legacy INTx
             │
             ▼
    ┌─────────────────┐
    │ 7. Device       │  设备特定初始化
    │    Specific     │  - 初始化硬件寄存器
    └─────────────────┘  - 注册网络/块设备等
```

---

## 8. 关键数据结构

### 8.1 struct pci_dev

```c
/* include/linux/pci.h */
struct pci_dev {
    struct list_head bus_list;      /* 总线设备链表节点 */
    struct pci_bus *bus;            /* 所属总线 */
    struct pci_bus *subordinate;    /* 桥接的下级总线 */

    unsigned int devfn;             /* 编码的 device & function */
    unsigned short vendor;          /* 厂商 ID */
    unsigned short device;          /* 设备 ID */
    unsigned short subsystem_vendor;
    unsigned short subsystem_device;
    unsigned int class;             /* 设备类别 */
    u8 revision;                    /* 修订版本 */
    u8 hdr_type;                    /* 头类型 */

    u8 pcie_cap;                    /* PCIe capability 偏移 */
    u8 msi_cap;                     /* MSI capability 偏移 */
    u8 msix_cap;                    /* MSI-X capability 偏移 */

    struct pci_driver *driver;      /* 绑定的驱动 */
    u64 dma_mask;                   /* DMA 地址掩码 */

    pci_power_t current_state;      /* 当前电源状态 D0-D3 */

    struct resource resource[DEVICE_COUNT_RESOURCE];  /* BAR 资源 */

    /* ... 更多字段 ... */
};
```

### 8.2 struct pci_driver

```c
/* include/linux/pci.h */
struct pci_driver {
    const char *name;                       /* 驱动名称 */
    const struct pci_device_id *id_table;   /* 支持的设备 ID 表 */

    int (*probe)(struct pci_dev *dev,
                 const struct pci_device_id *id);   /* 探测函数 */
    void (*remove)(struct pci_dev *dev);            /* 移除函数 */
    int (*suspend)(struct pci_dev *dev, pm_message_t state);  /* 挂起 */
    int (*resume)(struct pci_dev *dev);             /* 恢复 */
    void (*shutdown)(struct pci_dev *dev);          /* 关机 */

    struct pci_error_handlers *err_handler;         /* 错误处理 */

    struct device_driver driver;                    /* 通用驱动结构 */
    struct pci_dynids dynids;                       /* 动态 ID 列表 */
};
```

### 8.3 struct pci_bus

```c
/* include/linux/pci.h */
struct pci_bus {
    struct list_head node;          /* 总线链表节点 */
    struct pci_bus *parent;         /* 父总线 */
    struct list_head children;      /* 子总线列表 */
    struct list_head devices;       /* 设备列表 */

    struct pci_dev *self;           /* 代表此总线的桥设备 */
    struct list_head slots;         /* 插槽列表 */
    struct resource *resource[PCI_BRIDGE_RESOURCE_NUM];

    struct pci_ops *ops;            /* 配置空间访问操作 */
    void *sysdata;                  /* 系统特定数据 */

    unsigned char number;           /* 总线号 */
    unsigned char primary;          /* 主总线号 */
    unsigned char max_bus_speed;    /* 最大总线速度 */
    unsigned char cur_bus_speed;    /* 当前总线速度 */

    /* ... */
};
```

---

## 9. 实践示例

### 9.1 查看系统 PCIe 设备

```bash
# 列出所有 PCI 设备
$ lspci
00:00.0 Host bridge: Intel Corporation Device 9a14 (rev 01)
00:02.0 VGA compatible controller: Intel Corporation Device 9a49 (rev 03)
00:1f.0 ISA bridge: Intel Corporation Device a082 (rev 20)
01:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd ...

# 详细信息
$ lspci -vvv -s 01:00.0

# 树形显示
$ lspci -tv
-[0000:00]-+-00.0  Intel Corporation Device 9a14
           +-02.0  Intel Corporation Device 9a49
           +-1c.0-[01]----00.0  Samsung Electronics Co Ltd ...
           \-1f.0  Intel Corporation Device a082
```

### 9.2 内核配置空间访问

```c
/* 读取配置空间 */
u16 vendor, device;
u32 class;

pci_read_config_word(pdev, PCI_VENDOR_ID, &vendor);
pci_read_config_word(pdev, PCI_DEVICE_ID, &device);
pci_read_config_dword(pdev, PCI_CLASS_REVISION, &class);

/* 写入配置空间 */
u16 cmd;
pci_read_config_word(pdev, PCI_COMMAND, &cmd);
cmd |= PCI_COMMAND_MEMORY | PCI_COMMAND_MASTER;
pci_write_config_word(pdev, PCI_COMMAND, cmd);

/* 访问 PCIe 扩展能力 */
u16 devctl;
pcie_capability_read_word(pdev, PCI_EXP_DEVCTL, &devctl);
```

### 9.3 sysfs 接口

```bash
# PCI 设备 sysfs 路径
/sys/bus/pci/devices/0000:01:00.0/
├── class           # 设备类别
├── config          # 配置空间 (二进制)
├── device          # 设备 ID
├── vendor          # 厂商 ID
├── subsystem_device
├── subsystem_vendor
├── resource        # 资源描述
├── resource0       # BAR0 映射
├── resource1       # BAR1 映射
├── irq             # IRQ 号
├── enable          # 设备使能
├── driver -> ../../../bus/pci/drivers/nvme
├── msi_irqs/       # MSI 中断
└── power/          # 电源管理
```

---

## 10. 常见问题与调试

### 10.1 调试命令

```bash
# 查看内核 PCI 日志
$ dmesg | grep -i pci

# 查看设备详情
$ lspci -vvvxxx -s 01:00.0

# 查看 BAR 资源
$ cat /proc/iomem | grep -i pci

# 查看中断分配
$ cat /proc/interrupts | grep -i pci

# PCIe 链路状态
$ lspci -vvv -s 01:00.0 | grep -i "lnk"
```

### 10.2 常见问题

| 问题 | 可能原因 | 解决方案 |
|------|----------|----------|
| 设备未识别 | 硬件问题/BIOS设置 | 检查物理连接，更新BIOS |
| 驱动不加载 | ID 不匹配 | 检查 device_id 表 |
| DMA 失败 | DMA mask 设置错误 | 正确设置 dma_set_mask |
| 中断不工作 | MSI 配置问题 | 检查 MSI/MSI-X 设置 |
| 性能差 | PCIe 降速 | 检查 link speed/width |

---

## 参考资料

1. **PCI Express Base Specification** - PCI-SIG
2. **Linux Kernel Documentation** - `Documentation/PCI/`
3. **Linux Device Drivers, 3rd Edition** - O'Reilly
4. **Understanding Linux Network Internals** - O'Reilly

---

> **文档版本**: 1.0  
> **最后更新**: 2026年1月
