# PCIe 知识要点与 Linux 内核设备加载流程

> 分享人：Linux 内核开发工程师  
> 日期：2026年1月

---

## 目录

1. [PCIe 基础概念](#1-pcie-基础概念)
   - 1.1 [什么是 PCIe？](#11-什么是-pcie)
   - 1.2 [PCIe vs PCI 对比](#12-pcie-vs-pci-对比)
   - 1.3 [PCIe 带宽速率](#13-pcie-带宽速率)
   - 1.4 [Lane（通道）概念](#14-lane通道概念)
   - 1.5 [PCIe 发展历程](#15-pcie-发展历程)
2. [PCIe 拓扑结构](#2-pcie-拓扑结构)
   - 2.1 [整体架构](#21-整体架构)
   - 2.2 [核心组件说明](#22-核心组件说明)
   - 2.3 [PCIe Switch 内部结构](#23-pcie-switch-内部结构)
3. [PCIe 配置空间](#3-pcie-配置空间)
   - 3.1 [配置空间结构](#31-配置空间结构)
   - 3.2 [重要寄存器说明](#32-重要寄存器说明)
   - 3.3 [BAR (Base Address Register) 解析](#33-bar-base-address-register-解析)
4. [PCIe 事务层协议](#4-pcie-事务层协议)
   - 4.1 [PCIe 分层架构](#41-pcie-分层架构)
   - 4.2 [TLP (Transaction Layer Packet) 类型](#42-tlp-transaction-layer-packet-类型)
   - 4.3 [TLP 包结构](#43-tlp-包结构)
5. [Linux 内核 PCIe 子系统架构](#5-linux-内核-pcie-子系统架构)
   - 5.1 [内核 PCIe 目录结构](#51-内核-pcie-目录结构)
   - 5.2 [核心数据结构关系](#52-核心数据结构关系)
6. [PCIe 设备枚举流程](#6-pcie-设备枚举流程)
   - 6.1 [枚举总体流程](#61-枚举总体流程)
   - 6.2 [关键代码路径](#62-关键代码路径)
   - 6.3 [BDF 地址编码](#63-bdf-地址编码)
7. [PCIe 驱动加载流程](#7-pcie-驱动加载流程)
   - 7.1 [驱动注册与匹配流程](#71-驱动注册与匹配流程)
   - 7.2 [驱动结构定义](#72-驱动结构定义)
   - 7.3 [Probe 函数典型实现](#73-probe-函数典型实现)
   - 7.4 [设备初始化步骤图](#74-设备初始化步骤图)
8. [关键数据结构](#8-关键数据结构)
   - 8.1 [struct pci_dev](#81-struct-pci_dev)
   - 8.2 [struct pci_driver](#82-struct-pci_driver)
   - 8.3 [struct pci_bus](#83-struct-pci_bus)
9. [实践示例](#9-实践示例)
   - 9.1 [查看系统 PCIe 设备](#91-查看系统-pcie-设备)
   - 9.2 [内核配置空间访问](#92-内核配置空间访问)
   - 9.3 [sysfs 接口](#93-sysfs-接口)
10. [常见问题与调试](#10-常见问题与调试)
    - 10.1 [调试命令](#101-调试命令)
    - 10.2 [常见问题](#102-常见问题)
11. [pci-utils 基本用法](#11-pci-utils-基本用法)
    - 11.1 [安装](#111-安装)
    - 11.2 [lspci — 列出 PCI 设备](#112-lspci--列出-pci-设备)
    - 11.3 [setpci — 读写 PCI 配置空间](#113-setpci--读写-pci-配置空间)
    - 11.4 [update-pciids — 更新 PCI ID 数据库](#114-update-pciids--更新-pci-id-数据库)
    - 11.5 [实用调试场景](#115-实用调试场景)
    - 11.6 [pciutils 库 (libpci) 编程接口](#116-pciutils-库-libpci-编程接口)

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

### 1.5 PCIe 发展历程

PCIe 技术从传统 PCI 总线演化而来，经历了多次重大技术革新，以下是完整的发展脉络：

#### 1.5.1 前 PCIe 时代

| 时间 | 标准 | 说明 |
|------|------|------|
| **1990年** | PCI 草案 | Intel 主导制定 PCI (Peripheral Component Interconnect) 规范草案 |
| **1992年** | PCI 1.0 | 正式发布，32位/33MHz 并行总线，带宽 133MB/s |
| **1993年** | PCI 2.0 | 改进规范，完善电气特性和协议细节 |
| **1995年** | PCI 2.1 | 支持 66MHz，带宽提升至 266MB/s (32位) 或 533MB/s (64位) |
| **1998年** | PCI-X 1.0 | 面向服务器市场，64位/133MHz，带宽达 1066MB/s |
| **2002年** | PCI-X 2.0 | 支持 266MHz/533MHz，但共享总线架构的瓶颈愈发明显 |

```
    PCI 总线 (共享并行架构的局限)
    ┌──────────────────────────────────────────────────────────────┐
    │                                                              │
    │   CPU ◄══════════════════════════════════════════════►       │
    │          ║          ║          ║          ║                  │
    │        设备A       设备B       设备C       设备D              │
    │                                                              │
    │   问题：所有设备共享同一总线带宽，存在竞争和冲突               │
    │         并行信号高频下的串扰和时序问题严重                     │
    └──────────────────────────────────────────────────────────────┘
```

#### 1.5.2 PCIe 各代际发展

**PCIe 1.0 (2003年)**
- 由 Intel、Dell、HP、IBM 等联合推出
- 采用**点对点串行**架构，彻底取代 PCI 共享并行总线
- 每通道 (Lane) 传输速率 2.5 GT/s，采用 8b/10b 编码
- 单通道 (x1) 有效带宽 250 MB/s（双向 500 MB/s）
- 支持 x1, x2, x4, x8, x12, x16, x32 通道配置
- 定义了完整的分层协议架构（事务层、数据链路层、物理层）

**PCIe 1.1 (2005年)**
- 对 1.0 的小幅改进
- 完善了信号完整性要求和测试规范

**PCIe 2.0 (2007年)**
- 传输速率翻倍至 **5.0 GT/s**，仍使用 8b/10b 编码
- 单通道有效带宽 500 MB/s（双向 1 GB/s）
- 保持与 PCIe 1.x 完全向后兼容
- 引入链路速度自动协商机制

**PCIe 3.0 (2010年)**
- 传输速率提升至 **8.0 GT/s**
- 编码方式从 8b/10b 改为 **128b/130b**，编码效率从 80% 提升至 ~98.5%
- 单通道有效带宽约 984 MB/s（双向约 1.97 GB/s）
- 成为最长寿、应用最广泛的 PCIe 版本之一
- 此后很长一段时间，大量显卡、SSD 和网卡基于此标准

**PCIe 4.0 (2017年)**
- 传输速率再翻倍至 **16.0 GT/s**，编码仍为 128b/130b
- 单通道有效带宽约 1.97 GB/s（双向约 3.94 GB/s）
- AMD Zen 2 架构（如 Ryzen 3000 系列）率先支持
- NVMe SSD（如 Samsung 980 PRO）开始大量采用 PCIe 4.0 x4

**PCIe 5.0 (2019年)**
- 传输速率提升至 **32.0 GT/s**，编码仍为 128b/130b
- 单通道有效带宽约 3.94 GB/s（双向约 7.88 GB/s）
- Intel 第 12 代 Alder Lake 和 AMD Zen 4 架构开始支持
- 主要面向高端 SSD、AI 加速卡、高速网卡（100G/200G 以上）

**PCIe 6.0 (2022年)**
- 传输速率再次翻倍至 **64.0 GT/s**
- 编码方式革命性改变：采用 **PAM4 (4级脉冲幅度调制)** + **1b/1b FLIT 编码** + **FEC (前向纠错)**
- 引入 FLIT (Flow Control Unit) 固定大小包机制，取代传统的可变长 TLP
- 单通道有效带宽约 7.88 GB/s（双向约 15.75 GB/s）
- x16 配置下双向带宽高达 **252 GB/s**

**PCIe 7.0 (预计 2025年)**
- 目标传输速率 **128.0 GT/s**
- 继续使用 PAM4 调制
- x16 双向带宽预计将达 **512 GB/s**
- 主要面向数据中心、AI/ML 训练、高性能计算 (HPC)

```
    PCIe 各版本发展时间线与带宽演进
    ═══════════════════════════════════════════════════════════════

    2003       2007       2010       2017    2019  2022  2025(预计)
     │          │          │          │       │     │      │
     ▼          ▼          ▼          ▼       ▼     ▼      ▼
    1.0        2.0        3.0        4.0     5.0   6.0    7.0
   2.5GT/s    5GT/s      8GT/s     16GT/s  32GT/s 64GT/s 128GT/s
   8b/10b    8b/10b    128b/130b  128b/130b       PAM4   PAM4

    x16 双向带宽:
    8GB/s → 16GB/s → 32GB/s → 64GB/s → 128GB/s → 252GB/s → 512GB/s
              ×2       ×2       ×2        ×2        ×2         ×2

    关键技术节点:
    ├─ 2003: 串行取代并行，点对点取代共享总线
    ├─ 2010: 128b/130b 编码取代 8b/10b，编码效率质的飞跃
    ├─ 2022: PAM4 调制 + FLIT 机制，信号层面革命性变化
    └─ 2025: 继续倍增，面向 AI/HPC 超大带宽需求
```

#### 1.5.3 PCIe 标准管理组织

PCIe 规范由 **PCI-SIG (PCI Special Interest Group)** 制定和维护：

- **成立时间**: 1992 年
- **成员**: 超过 900 家企业（含 Intel、AMD、NVIDIA、Broadcom、Samsung 等）
- **职责**: 制定 PCI/PCIe 规范、合规性测试、互操作性认证
- **官网**: https://pcisig.com

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

## 11. pci-utils 基本用法

**pciutils** (也写作 pci-utils) 是 Linux 下用于查看和操作 PCI/PCIe 设备的标准用户态工具集，由 Martin Mares 维护，是每一位内核开发和系统管理员的必备工具。

### 11.1 安装

```bash
# Debian / Ubuntu
$ sudo apt install pciutils

# RHEL / CentOS / Fedora
$ sudo dnf install pciutils
# 或
$ sudo yum install pciutils

# Arch Linux
$ sudo pacman -S pciutils

# 源码编译
$ git clone https://github.com/pciutils/pciutils.git
$ cd pciutils
$ make && sudo make install
```

### 11.2 lspci — 列出 PCI 设备

`lspci` 是 pciutils 中最常用的命令，用于列出系统中所有 PCI/PCIe 设备信息。

#### 基本用法

```bash
# 列出所有 PCI 设备（简洁模式）
$ lspci
00:00.0 Host bridge: Intel Corporation 12th Gen Core Processor Host Bridge (rev 02)
00:02.0 VGA compatible controller: Intel Corporation Alder Lake-P GT2 (rev 0c)
00:1f.0 ISA bridge: Intel Corporation Alder Lake PCH eSPI Controller (rev 01)
01:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd NVMe SSD ...
02:00.0 Network controller: Intel Corporation Wi-Fi 6 AX201 (rev 20)
```

#### 常用选项

```bash
# -v / -vv / -vvv  逐步增加详细程度
$ lspci -v              # 简要详细信息
$ lspci -vv             # 更详细信息
$ lspci -vvv            # 最详细信息（包括所有 Capability 解析）

# -s  指定设备（BDF 地址过滤）
$ lspci -s 01:00.0              # 查看特定设备
$ lspci -s 01:                  # 查看总线 01 上的所有设备
$ lspci -s :00.0                # 查看所有总线上 device=00, function=0 的设备

# -d  按 Vendor:Device ID 过滤
$ lspci -d 8086:                # 列出所有 Intel (0x8086) 设备
$ lspci -d :1234                # 列出 Device ID 为 0x1234 的设备
$ lspci -d 10de:*               # 列出所有 NVIDIA (0x10de) 设备

# -t  树形拓扑显示
$ lspci -tv
-[0000:00]-+-00.0  Intel Corporation 12th Gen Core Processor Host Bridge
           +-02.0  Intel Corporation Alder Lake-P GT2
           +-1c.0-[01]----00.0  Samsung Electronics Co Ltd NVMe SSD
           +-1c.4-[02]----00.0  Intel Corporation Wi-Fi 6 AX201
           \-1f.0  Intel Corporation Alder Lake PCH eSPI Controller

# -n  显示数字格式的 Vendor/Device ID（不解析名称）
$ lspci -n
00:00.0 0600: 8086:4621 (rev 02)
01:00.0 0108: 144d:a80a

# -nn  同时显示名称和数字 ID（最实用）
$ lspci -nn
00:00.0 Host bridge [0600]: Intel Corporation 12th Gen ... [8086:4621] (rev 02)
01:00.0 NVMe controller [0108]: Samsung Electronics [144d:a80a]

# -k  显示设备使用的内核驱动和模块
$ lspci -k
01:00.0 Non-Volatile memory controller: Samsung Electronics Co Ltd ...
        Subsystem: Samsung Electronics Co Ltd ...
        Kernel driver in use: nvme
        Kernel modules: nvme

# -x / -xx / -xxx / -xxxx  以十六进制 dump 配置空间
$ lspci -x -s 01:00.0          # 标准配置空间 (前64字节)
$ lspci -xx -s 01:00.0         # 标准配置空间 (前256字节)
$ lspci -xxx -s 01:00.0        # 扩展配置空间 (需 root)
$ lspci -xxxx -s 01:00.0       # 完整 4KB 扩展配置空间 (需 root)

# -D  始终显示 Domain 号
$ lspci -D
0000:00:00.0 Host bridge: Intel Corporation ...
```

#### 组合使用示例

```bash
# 查看 NVMe SSD 完整信息 + PCIe 链路状态
$ sudo lspci -vvv -s 01:00.0

# 输出中重点关注:
#   LnkCap: Speed 8GT/s, Width x4        ← 设备支持的最大链路能力
#   LnkSta: Speed 8GT/s, Width x4        ← 当前实际链路状态
#   LnkCtl: ASPM L1 Enabled              ← 电源管理状态

# 查看所有网络设备及其驱动
$ lspci -nn -k -d ::0200
02:00.0 Ethernet controller [0200]: Intel Corporation I210 [8086:1533] (rev 03)
        Kernel driver in use: igb

# 以机器可读格式输出（适合脚本解析）
$ lspci -mm
$ lspci -vmm               # 每个字段独立一行
$ lspci -vmm -s 01:00.0
Slot:   01:00.0
Class:  Non-Volatile memory controller
Vendor: Samsung Electronics Co Ltd
Device: NVMe SSD Controller ...
```

### 11.3 setpci — 读写 PCI 配置空间

`setpci` 用于直接读取或写入 PCI 设备的配置空间寄存器，是底层调试和硬件配置的利器。

> **警告**: 错误使用 `setpci` 可能导致系统不稳定甚至硬件损坏，请谨慎操作。

#### 基本语法

```bash
setpci [选项] -s <BDF> <寄存器>[.<宽度>][=<值>]

# 宽度说明:
#   .B = Byte (1字节)
#   .W = Word (2字节)
#   .L = Long (4字节, 默认)
```

#### 常用示例

```bash
# 读取 Vendor ID (偏移 0x00, 2字节)
$ sudo setpci -s 01:00.0 0x00.W
144d

# 读取 Device ID (偏移 0x02, 2字节)
$ sudo setpci -s 01:00.0 0x02.W
a80a

# 读取 Command 寄存器 (偏移 0x04, 2字节)
$ sudo setpci -s 01:00.0 COMMAND.W
0407

# 读取 Status 寄存器
$ sudo setpci -s 01:00.0 STATUS.W
0010

# 读取 Class Code (偏移 0x08, 4字节，包含 revision)
$ sudo setpci -s 01:00.0 0x08.L
01080200

# 读取 BAR0
$ sudo setpci -s 01:00.0 BASE_ADDRESS_0.L
a1200004

# 使用寄存器名称（更易读）
$ sudo setpci -s 01:00.0 VENDOR_ID.W
$ sudo setpci -s 01:00.0 DEVICE_ID.W
$ sudo setpci -s 01:00.0 CLASS_DEVICE.W

# 写入 Command 寄存器（启用 Bus Master）
$ sudo setpci -s 01:00.0 COMMAND.W=0x0407

# 使用位掩码进行部分写入
# 格式: 寄存器.宽度=值:掩码
# 仅修改掩码为1的位
$ sudo setpci -s 01:00.0 COMMAND.W=0004:0004   # 仅设置 Bus Master 位

# 读取 PCIe Capability 中的 Link Status
# 首先找到 PCIe Cap 偏移 (通常在 Capability List 中)
$ sudo setpci -s 01:00.0 CAP_EXP+12.W    # Link Status Register
```

#### 寄存器名称速查

```bash
# setpci 支持的常用寄存器名称:
VENDOR_ID           # 0x00  厂商 ID
DEVICE_ID           # 0x02  设备 ID
COMMAND             # 0x04  命令寄存器
STATUS              # 0x06  状态寄存器
REVISION            # 0x08  修订号
CLASS_PROG          # 0x09  编程接口
CLASS_DEVICE        # 0x0a  设备类别
CACHE_LINE_SIZE     # 0x0c  缓存行大小
LATENCY_TIMER       # 0x0d  延迟计时器
HEADER_TYPE         # 0x0e  头类型
BASE_ADDRESS_0      # 0x10  BAR0
BASE_ADDRESS_1      # 0x14  BAR1
BASE_ADDRESS_2      # 0x18  BAR2
BASE_ADDRESS_3      # 0x1c  BAR3
BASE_ADDRESS_4      # 0x20  BAR4
BASE_ADDRESS_5      # 0x24  BAR5
INTERRUPT_LINE      # 0x3c  中断线
INTERRUPT_PIN       # 0x3d  中断引脚

# PCIe Capability 相关 (通过 CAP_EXP 前缀访问):
CAP_EXP+02.W       # PCIe Capabilities Register
CAP_EXP+08.L       # Device Capabilities
CAP_EXP+0c.W       # Link Capabilities (低16位)
CAP_EXP+10.W       # Link Control
CAP_EXP+12.W       # Link Status
```

### 11.4 update-pciids — 更新 PCI ID 数据库

`lspci` 依赖 PCI ID 数据库来将数字 ID 解析为可读的厂商和设备名称。

```bash
# 更新本地 PCI ID 数据库（需要网络）
$ sudo update-pciids
Downloaded daily snapshot dated 2026-02-05 ...

# 数据库文件位置
$ ls -la /usr/share/misc/pci.ids*
# 或
$ ls -la /usr/share/hwdata/pci.ids

# 数据库来源: https://pci-ids.ucw.cz/
```

### 11.5 实用调试场景

#### 场景一：排查 PCIe 链路降速

```bash
# 1. 查看设备的链路能力和当前状态
$ sudo lspci -vvv -s 01:00.0 | grep -E "LnkCap|LnkSta"
    LnkCap: Port #0, Speed 16GT/s, Width x4, ...
    LnkSta: Speed 8GT/s (downgraded), Width x4 (ok), ...

# 解读: 设备支持 PCIe 4.0 x4，但当前运行在 PCIe 3.0 x4 (降速)
# 可能原因: 主板限制、BIOS 设置、信号完整性问题

# 2. 同时检查上游端口 (Root Port 或 Switch Downstream Port)
$ sudo lspci -vvv -s 00:1c.0 | grep -E "LnkCap|LnkSta"
```

#### 场景二：检查 MSI/MSI-X 中断配置

```bash
# 查看设备的 MSI-X 信息
$ sudo lspci -vvv -s 01:00.0 | grep -A5 "MSI-X"
    Capabilities: [b0] MSI-X: Enable+ Count=128 Masked-
        Vector table: BAR=0 offset=00002000
        PBA: BAR=0 offset=00003000
```

#### 场景三：识别未知设备

```bash
# 1. 获取设备的 Vendor:Device ID
$ lspci -nn -s 03:00.0
03:00.0 Unclassified device [00ff]: Unknown device [1234:5678]

# 2. 在线查询
# 访问 https://pci-ids.ucw.cz/ 输入 Vendor ID 和 Device ID 查询

# 3. 查看 subsystem ID 获取更多线索
$ lspci -vvv -s 03:00.0 | grep -i subsystem
```

#### 场景四：导出设备信息用于 Bug 报告

```bash
# 完整导出所有 PCI 信息（适合附在 Bug 报告中）
$ sudo lspci -vvvnn > pci_info.txt

# 导出完整配置空间十六进制 dump
$ sudo lspci -xxxx > pci_config_dump.txt

# 导出机器可读格式
$ lspci -vmm > pci_machine_readable.txt
```

### 11.6 pciutils 库 (libpci) 编程接口

pciutils 还提供了 C 语言库 `libpci`，可以在用户态程序中直接使用：

```c
/* 使用 libpci 读取设备信息的示例 */
#include <pci/pci.h>
#include <stdio.h>

int main(void)
{
    struct pci_access *pacc;
    struct pci_dev *dev;
    char namebuf[1024];

    /* 初始化 libpci */
    pacc = pci_alloc();
    pci_init(pacc);
    pci_scan_bus(pacc);

    /* 遍历所有设备 */
    for (dev = pacc->devices; dev; dev = dev->next) {
        pci_fill_info(dev, PCI_FILL_IDENT | PCI_FILL_BASES |
                           PCI_FILL_CLASS | PCI_FILL_CAPS);

        printf("%04x:%02x:%02x.%d  vendor=%04x device=%04x class=%04x\n",
               dev->domain, dev->bus, dev->dev, dev->func,
               dev->vendor_id, dev->device_id, dev->device_class);

        /* 获取可读名称 */
        printf("  %s\n",
               pci_lookup_name(pacc, namebuf, sizeof(namebuf),
                               PCI_LOOKUP_DEVICE,
                               dev->vendor_id, dev->device_id));
    }

    pci_cleanup(pacc);
    return 0;
}
```

编译方式：

```bash
$ gcc -o pci_scan pci_scan.c -lpci
$ sudo ./pci_scan
```

---

## 参考资料

1. **PCI Express Base Specification** - PCI-SIG
2. **Linux Kernel Documentation** - `Documentation/PCI/`
3. **Linux Device Drivers, 3rd Edition** - O'Reilly
4. **Understanding Linux Network Internals** - O'Reilly
5. **pciutils 项目主页** - https://github.com/pciutils/pciutils
6. **PCI ID 数据库** - https://pci-ids.ucw.cz/

---

> **文档版本**: 1.1  
> **最后更新**: 2026年2月
