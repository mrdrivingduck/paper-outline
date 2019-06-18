# Outline

## Charm: Facilitating Dynamic Analysis of Device Drivers of Mobile Systems - Usenix Security 2018

Created by : Mr Dk.

2019 / 06 / 18 10:51

Nanjing, Jiangsu, China

---

## Introduction

如今的移动系统包含了一系列各种各样的 I/O 设备

* 相机
* 屏幕
* 传感器
* GPU 加速器
* 网络设备

截止 2015 年

目前市场上有超过 1000 家安卓设备制造商

和超过 24000 种独立的安卓设备

对于移动操作系统来说

大量的定制设备驱动需要被实现

为成千上万种设备提供支持

设备驱动运行于操作系统内核中

是很多种威胁的源头

然而，在移动操作系统上对设备驱动进行动态分析困难、低效，甚至不可能

比如，一些方法需要驱动运行在一个可控环境中，比如虚拟机

然而，对于运行在移动 OS 内核中的驱动是不可能的

一些内核 fuzzer 应用在移动系统上时会产生一定的问题

* 比如 _kAFL_ 需要驱动运行在基于 x86 的虚拟机中（对于移动设备不现实）
* 直接在移动系统上使用 _Syzkaller_ 具有挑战性
  * 缺乏 fuzzing 特征的支持
  * 缺乏对系统控制台的访问

本文中的 Charm 是一套促进移动系统设备驱动动态分析的系统

最核心的贡献在于

实现了在不同的物理机器的虚拟机中执行移动系统的 I/O 设备驱动

* 由于驱动执行于虚拟机中
* 分析者能够通过不同的动态分析技术分析其中的威胁

将移动系统的设备驱动在工作站的虚拟机中执行通常情况下是不可能的

* 驱动需要访问移动系统中的 I/O 设备硬件

本文通过 _remote device driver execution_ 的方法解决了这个问题

* 设备驱动访问 I/O 设备的请求被 hypervisor 截获
* 并通过一根低延时的 USB 通道路由到真实设备上
* 实际的移动系统用于执行不频繁的低层 I/O 操作
* 驱动程序完全运行在虚拟机中

远程的设备驱动执行引发了两个重要的挑战：

* 设备驱动和对应的 I/O 设备之间的交互是时间敏感的
  * 因此虚拟机和真实设备之间的高延时会导致超时问题
  * 本文中通过定制的 USB 通道解决
  * 不需要任何定制的硬件用于连接移动系统和虚拟机
  * 利用了平常可用的 USB 接口
* 设备驱动除了和 I/O 设备硬件交互以外，还与内核的其它模块交互
  * 总线驱动
  * 功耗管理模块
  * 时钟模块
  * _resident modules_
  * 由于与移动系统的 USB 通道的使用有关，因此不能放到虚拟机中执行
  * 通过 Remote Procedure Call (RPC) 接口解决这个问题
    * 远程驱动通过 RPC 接口与移动系统中的这些模块进行交互
    * 基于 Linux API 实现
    * 不同的设备驱动和不同的移动系统可以使用相同的 RPC 接口

Charm 原型运行于 Intel Xeon-based 工作站上

和三款智能手机中

实现了远程设备驱动执行：

* Nexus 5X - 相机和语音驱动
* Nexus 6P - GUP 设备驱动
* Samsung Galaxy S7 - Inertial Measurement Unit (IMU) 传感器驱动

只支持开放源代码的设备驱动

经过大量评估：

* 在 Charm 中加入对新的设备驱动的支持是可行的
* 虽然存在远程驱动执行的开销，Charm 与真实移动系统的性能相当
* fuzzer 可以在 Charm 中 fuzz 相同数量的 fuzzing 程序，并取得相似的代码覆盖率
* 能够记录并重现设备驱动的执行，轻松重现一个 BUG
* 使用 GDB 等调试工具分析驱动中的威胁是可行的

---

## Motivation

讨论了三种重要的动态分析技术

讨论了将这些技术应用在移动系统设备驱动上的现有挑战

简要介绍 Charm 如何克服这些挑战

### Manual Interactive Debugging

安全分析员通常使用 GDB 等调试器分析威胁

调试器使分析人员能够在代码中下断点

查看内存中的内容

并对重要的数据结构设置监视点

检测试图修改它们的行为

然而，对于运行在移动操作系统内核中的设备驱动来说不可行

内核调试器 KGDB，通过提供交互式的调试来解决问题

然而，用 KGDB 来吊事移动系统的内核要么难以使用，要么需要特定的适配器

KGDB 需要访问控制台，可以通过 UART 硬件实现

而移动系统通常没有 UART 硬件，因此不支持 KGDB

即使有 UART 硬件，也需要打开系统，找到 UART 引脚并焊接...

* 又困难又容易出错

Charm 解决了这个问题

因为驱动运行在虚拟机中，所以可以轻易被分析

### Record-and-Replay

是一种分析程序行为的工具

它使得分析人员能够记录设备驱动的执行，并在需要时重放

一个 crash 可能依赖于特定驱动执行的穿插顺序和 I/O 设备引发的中断

如果执行过程被记录，那么 crash 能够被轻易重现

特别地，在 Charm 中

驱动执行过程的重现甚至不需要访问真实的移动系统

任何能够访问虚拟机的用户都可以重现设备驱动的执行过程并分析

特别地，Charm 在 hypervisor 中记录了所有驱动和远程 I/O 设备的交互

并在需要时可以被查阅

### Fuzzing

Fuzzing 是一种动态分析技术

通过向被测试模块提供不同的输入

找出软件模块中的 bug

对于设备驱动程序

输入是通过系统调用进入驱动的

* `ioctl`
* `read`

如果输入是随机选择的，那么 fuzzing 会产生低代码覆盖率的问题

为了提升代码覆盖率

基于反馈的 fuzzing 通过采集执行信息，用于指引输入生成的过程

比如 _kAFL_ 就是这样的一种工具

由 hypervisor 利用 Inter Processor Tracer (PT) 硬件采集虚拟机的执行信息

* 使用该技术对移动系统进行 fuzzing 在目前不可行
* 移动设备大多使用 ARM 处理器，不提供 Intel 的 PT 硬件
* 此外，对于 ARM 的 hypervisor 支持暂时没有

然而，Charm 通过将虚拟机运行在 x86 的机器上，使 kAFL 可以被使用

另一款可以用于对内核设备驱动进行 fuzzing 的工具是 _Syzkaller (by Google)_

使用了基于编译器的信息采集器，并用它来指引输入的生成

使用编译器将信息采集器插入到内核中

Syzkaller 可以直接 fuzz 运行在移动系统中的设备驱动

在 Charm 中使用 Syzkaller 的好处：

1. Syzkaller 能够借助其它只能在虚拟机中使用的动态分析技术，加强 bug 的分析
2. 在虚拟机中利用新的内核 sanitizer 更加简单
   * 内核 sanitizers 在编译时注解内核，使 Syzkaller 能够通过监视内核执行找出 non-crash bug
   * KASAN - use after free
   * KTSAN - data races
   * KMSAN - the use of uninitialized memory
   * KUBSAN - undefined behavior
   * 移动系统的内核通常不支持这些 sanitizer

而在 Charm 中，可以选择 sanitizer 支持的虚拟机内核

Syzkaller 通过 OS 的 console 读取内核 log

需要内核 log 在 crash 的瞬间记录 dump stack

虚拟机的 console 由 hypervisor 管理确保可用

而在移动系统上获取 crash 时的 console message 具有挑战性

* 需要一个特定的 adapter

对于内核开发者来说，应该很熟悉使用串口来获取崩溃前信息的难度

当硬件不可获得时，调试移动系统内核只能通过 USB 从 Android Debug Bridge (ADB)

然而，ADB 接口无法传递内核的 crash logs

因为 ADB 守护进程也崩溃了

在移动系统重启后，crash logs 可能不一定可用

因为一次 crash 会损坏内核，阻止其将控制台信息刷新到存储介质的能力

在 Charm 原型中，选用了 Syzkaller 作为分析工具之一

比较其在虚拟机上和在移动系统上的 fuzzing 性能

---

## Overview

目标：将现有的动态分析技术应用到移动设备的 I/O 驱动上

### Strew-man Approaches

两种假想的试图将设备驱动运行在虚拟机中的方式

1. 将设备运行在一个已有的虚拟机内
   * 驱动需要访问移动系统中的 I/O 设备硬件
   * 在 boot 阶段，驱动不会被内核初始化，因为内核没有找到 I/O 设备
   * 如果强制调用初始化函数，驱动程序将会立刻报错
   * 在虚拟机中对 I/O 设备硬件进行仿真
     * I/O 设备的种类过多，工程量过大
2. 在移动系统中的虚拟机上运行设备驱动
   * 使虚拟机能够访问底层的 I/O 设备
     * 这种技术主要支持基于 x86 工作站的的 PCI 设备，不支持移动系统的 I/O 设备
     * 在移动系统上运行基于硬件的虚拟机是不可能的
       * ARM 处理器提供硬件虚拟化支持
       * 但 hypervisor 模式被禁止，以防止 rootkits
     * 基于软件的虚拟化 - 低性能

### Charm's Approach

Charm 将设备驱动程序的执行从移动系统硬件上解耦

设备驱动需要访问 I/O 设备以正确执行

Charm 通过 _remote device driver execution_ 重用了物理 I/O 设备

* 将物理移动系统与工作站通过 USB 线连接
* 设备驱动程序完全在工作站中执行
* 只有不频繁的低层 I/O 操作被转发并在物理移动系统上被执行

在 Charm 中，远程 I/O 操作的延时相当重要

高延时将导致各种超时问题：

* 设备驱动程序通常会等待一个给定的时间，等待 I/O 设备的回应
  * 如果设备回应超时，则驱动程序触发超时错误
* I/O 设备可能需要及时地读写寄存器
  * 比如当设备触发中断时，需要驱动及时清除寄存器中的某一位
  * 否则将重复触发中断

在 x86 虚拟机中执行设备驱动

由于移动系统通常使用 ARM 处理器

为什么不使用 ARM 虚拟机呢？

* 使用了 ARM 虚拟机和 ARM-to-x86 的指令解释
* 指令解释的开销使执行速度减慢
* 设备驱动触发了各种各样的超时错误

为了达到设备驱动的延迟要求，需要本地执行指令

使用了硬件虚拟化的 x86 虚拟机实现，在 KVM 中重新实现了 Charm

（ARM 工作站太少见）

### Potential Concerns

1. 较差的性能
   * 远程 I/O 操作可能显著拖慢设备驱动的执行速度
   * 甚至可能导致设备行为出错
   * 利用 x86 处理器和低延时的 USB 通道
     * 解决了超时问题
     * 性能与直接在移动系统上运行相当
2. ARM ISA 和 x86 ISA 的差异问题
   * 虚拟机使用的 x86 ISA 会导致设备驱动行为不正确 → ❌
   * 设备驱动由 C 实现，BUG 出现在源代码中，而与目标平台无关
   * 编译器错误 - 一个 BUG 可能出现在 C x86 编译器中，而没有出现在 C ARM 编译器中

---

## Remote Device Driver Execution

Charm 的核心技术是移动系统设备驱动的远程执行

* 设备驱动运行在工作站的虚拟机上

截获驱动访问 I/O 设备的低层请求

通过 USB 通道路由到真实的移动系统硬件上

类似地，I/O 设备触发的中断被路由到虚拟机中的设备驱动上

### Device and Device Driver Interactions

远程驱动执行需要在与 I/O 设备不同的物理机器上执行设备驱动

为了实现，在工作站的 hypervisor 和移动系统上分别加入一个 stub 模块实现通信

* 访问 I/O 设备的寄存器 - ✔️
* 中断 - ✔️
* DMA - ❌

前两种已经足够用于移植和执行很多设备驱动了

#### Register Accesses

通过 hypervisor，截获设备驱动访问设备寄存器的请求

* 对于 write，将待写入的值转发到移动系统的 stub 中
* 对于 read，向 stub 模块发送一个 read 请求，接收回复，并返回设备驱动的虚拟机中

#### Interrupts

移动系统中的 stub 模块代替远程驱动注册了一个中断 handler

当 I/O 设备触发中断时

移动系统的 stub 将中断转发到工作站中的 stub

然后将中断注入到虚拟机中

由设备驱动来处理中断

### Device Driver Initialization

为了使设备驱动程序能够被初始化

内核必须在系统中检测到对应的 I/O 设备

因此必须使虚拟机内核检测到对应的 I/O 外设

ARM 和 x86 机器使用了不同的 I/O 设备检测方法

* ARM
  * device tree
  * 包含一系列硬件组件的软件清单
  * 在 boot 阶段，内核将 device tree 进行转化，并初始化对应的设备驱动
* x86
  * Advanced Configuration and Power Interface (ACPI)
  * 在 x86 虚拟机中，ACPI 接口由 hypervisor 仿真

第一种思路是在 hypervisor 的 ACPI 层加入远程 I/O 设备

* 将 device tree entries 翻译为 ACPI 设备需要很大的工程量

因此，使内核先完成 ACPI 设备检测

再通过转换 device tree 检测远程 I/O 设备

* 减少了工程量
* 只需要将移动系统中对应 I/O 设备的 entry 拷贝到虚拟机中的 device tree 上即可

### Low-Latency USB Channel

USB 提供了足够的带宽

在原型实现中，一开始使用了在 ADB 接口上的基于 TCP的 socket 传输

产生了 1-2 ms 的环回延时

这一延时由虚拟机和移动系统的若干次用户/内核空间转换有关

为了解决这一问题，本文实现了一种低层、定制的 USB 通道

作者创建了 USB gadget interface，包含五个端点

* 两个端点用于寄存器访问的双向通信
* 两个端点用于 RPC 调用的双向通信
* 一个端点用于中断的单向通信

在移动系统和工作站上，stub 模块直接在内核中读写这些端点

避免了多次耗时的用户/内核状态切换

显著地降低了延迟

为了进一步降低延时

优化 - write batching

成批处理连续的寄存器写操作

* 在 USB 通道上发送写请求
* 异步接收 ACK

防止同步等待 ACK 带来的延迟

### Dependencies

设备驱动不仅仅和 I/O 设备硬件进行交互

还经常和移动系统中的其它内核模块进行交互

使用了两种解决方法解决这种依赖：

1. 如果内核模块在移动系统中不需要，那么将该模块移动到虚拟机中
   * 虚拟机中的模块越多，分析越到位
   * 比如 bus driver
2. 如果模块在移动系统中需要，那么在移动系统中保留这些模块
   * 为虚拟机实现 RPC 接口，以对这些模块进行远程访问

作者已经研究出了不能被移至虚拟机的内核模块最小集

* _resident modules_
  * power & clock management
  * pin controller
  * GPIO

这些模块负责 boot 移动系统的硬件组件和配置 USB 接口

通过通用的内核 API 实现 Charm 的 RPC 接口，以保证可移植性

* 基于 Android 的移动系统内核使用了几乎相同的 API

### Porting a Device Driver to Charm

在 Charm 中支持新的驱动

需要将驱动移植到 Charm 中

* 将驱动移植到不同 Linux 内核版本或使用不同平台的内核中

设备驱动开发者很熟悉这一过程

* 向 Charm 中移植驱动对于驱动开发者是日常操作...

移植设备驱动需要以下步骤：

1. 将设备驱动加入到虚拟机内核中 - （复制源代码到 kernel source tree）
   * 如果驱动有依赖模块，也需要被移动到虚拟机内核中
   * 虚拟机内核和移动系统内核的版本不同导致核心 API 不同
     * 所以尽可能使用相近的内核版本
     * 小改动
   * x86 与 ARM 的不兼容
     * 可能由于在驱动中使用了与平台相关的 API
     * 最好在 x86 内核中实现 ARM 的常量和 API，而不要改动驱动
2. 将驱动配置为能够在虚拟机中运行
   * 必须将移动系统中对应的 device tree 条目移动到虚拟机中
   * 依赖的 device tree 条目，如 bus entry，也需要被移动
3. 使驱动的 I/O 请求能够发送到远程的移动系统上
   * 对应 I/O 设备的寄存器页的物理地址
   * RPC 接口
4. 使移动系统能够处理远程操作
   1. stub 需要被移植到移动系统内核上
      * 加入内核模块
      * 使 USB 接口与该内核模块一起工作
   2. 虚拟机上运行的设备驱动需要在移动系统上被禁用
      * 在内核构建阶段禁用设备驱动
      * 在 device tree 中移除对应 I/O 设备的条目

---

## Implementation & Prototype

本文作者将 4 个设备移植到了 Charm 中：

* Camera 设备驱动 - Nexus 5X
* Audio 设备驱动 - Nexus 5X
* GPU 设备驱动 - Huawei Nexus 6P
* IMU 传感器驱动 - Samsung Galaxy S7

Charm 中暂未实现 DMA

而 DMA 在 CPU 与 I/O 外设之间传递数据时经常被用到

QEMU/KVM hypervisor

在虚拟机中，基于 x86 的中断控制器最多只支持 24 根中断线

ARM 中断控制器支持 987 根中断线

需要在 hypervisor 中实现中断线的翻译

将支持中断线扩展到 128 根

---

## Evaluation

### Feasibility

Charm 支持不同移动系统中的不同驱动

本文评估了将新的驱动移植到 Charm 上需要多久

* Less than one week

作者很熟悉内核编程和设备驱动编程

安全分析员也应当如此

### Performance

#### Experiment 1

Charm 在每一次的远程操作中产生了可观测的延时

使用 Syzkaller 发送大量系统调用对驱动进行 fuzzing

* 在 Charm 中
* 在移动系统中

Syzkaller 会自动生成带有大量系统调用的程序

作者运行 Syzkaller 1 小时

并记录了执行程序的数量和代码覆盖情况

配置：

* LVM - 低配虚拟机
* MVM - 中配虚拟机
* HVM - 高配虚拟机
* Phone - 移动系统

MVM 的效果最好

* 在资源上的富裕使得性能超过了 LVM
* HVM 上的高并发对性能产生了消极的影响
* MVM 和 HVM 比直接在移动系统上进行 fuzzing 效果好
  * 说明 Charm 的远程设备驱动执行不会对性能有消极影响

Charm 也取得了与在实际移动系统上进行 fuzzing 的代码覆盖率

#### Experiment 2

选取了一个性能测试程序

Nexus 5X 的相机驱动程序的初始化

* 从 EEPROM 芯片中读取大量数据，从而引发大量远程 I/O 操作

I/O 密集的性能测试显著影响了 Charm 的性能

### Record-and-Replay

Record-and-Replay 在 Charm 中的可行性

只记录设备驱动和 I/O 设备之间的交互

重放寄存器操作很简单：

* 写操作被忽略
* 读操作返回日志中记录的值

重放中断：

* 在所有之前的寄存器操作完成后，注入中断

在不需要移动系统的情况下，成功重放了设备驱动的执行

实验还评估了 Record-and-Replay 带来的开销：

* recording 不会给 Charm 的执行带来显著开销
* replay 比正常执行快得多

### Bug Finding

### Analyzing Vulnerabilities with GDB

### Building a Driver Exploit using GDB

---

## Related Work

### Remote I/O Access

_Avatar_ 和 _SURROGATES_ 用于在嵌入式设备中动态分析二进制固件

* 在仿真器中执行固件
* 向嵌入式设备转发低层的内存访问，包括 I/O 操作

嵌入式设备通常使用低带宽的 UART 或 JTAG 接口连接

甚至可能需要定制的 FPGA bridge

相比之下，Charm 使用了高带宽的 USB，且不需要任何定制硬件

且 Charm 不转发内存访问，只转发 I/O 请求，性能更好

与已有工作的区别体现在：I/O 操作的界限

* 在一些工作中，设备驱动需要在 I/O 设备的宿主机上运行
* 而 Charm 实现了设备驱动与 I/O 设备在不同的宿主机上运行

### Analysis of System Software

* Taint Tracking
* Symbolic and Concolic Execution
* Unpacking and Reverse Engineering
* Malware Sandboxing
* Fuzzing

很多分析基于虚拟化技术进行

支持全系统的分析

* 内核
* 设备驱动程序

在软件中模拟硬件

已有工作只支持运行在虚拟机中的系统软件

不支持移动系统的设备驱动

### Mobile Testing

---

## Limitations and Future Work

### DMA

### Closed source (binary) drivers

不支持闭源设备驱动

因此无法通过源代码移植到 x86 虚拟机上

可以直接使用 ARM 虚拟机

* 直接运行在 ARM 平台上
* 或者运行在 x86 平台上，运行 ARM-to-x86 解释器
  * 但需要提高解释器的性能

### Automatic device driver porting

能够自动移植代码

用户只需要提供驱动源代码和 resident module 的清单

框架自动实现所有 RPC 并移植驱动

---

## Summary

工作实在是太底层了

此外 有一些工具已经在几篇论文中反复提到

可以试着去玩一玩了

目前基于虚拟机的动态分析好像很流行

因为在 hypervisor 层面可以监控虚拟机的所有操作

内存操作也好、函数调用也好

并对感兴趣的行为进行分析

因此 如果要做这类工作的话

首先熟悉这些虚拟化工具是前提

然后只要有了任何的 idea

只需要在虚拟机上对 idea 进行验证分析即可

---

