# Outline

## What You Corrupt Is Not What You Crash: Challenges in Fuzzing Embedded Devices - NDSS 2018

Created by : Mr Dk.

2019 / 06 / 11 15:47

Nanjing, Jiangsu, China

---

## Introduction

联网的嵌入式系统成为现代生活的核心组件

与它们相关的在线服务是物联网的核心

嵌入式设备上的大部分威胁很容易修复

* 比如认证或不安全的管理接口的威胁 - 可以通过加强厂商和用户的意识而修复
* 编程错误导致的内存损坏 - fuzz-testing

所谓的 fuzzing

指自动生成恶意输入，并向设备发送，并监控异常行为

程序异常属于出错的一种，体现为 crashes

不幸的是

桌面系统有各种各样的机制检测出错状态：

* 段错误
* heap dardening
* sanitizers

便于分析：

* 命令行返回值
* core dumps

嵌入式设备通常缺少类似的机制

* 受限的 I/O 能力
* 成本受限
* 首先的计算能力

因此，silent memory corruptions 在嵌入式系统中经常发生

为嵌入式系统的 fuzzing 带来许多挑战

Fuzzing 可以在有或没有源代码的前提下进行

* 有源代码可以使 fuzzing 更有效率 - 编译时 corruption 检测
* 但嵌入式设备的源码很少可以获得

因此唯一可行的办法是监控设备，检测错误行为发生的标志

* liveness check - 不足以检测所有的危险
* 很难在黑盒测试中检测嵌入式设备的内存是否已经损坏

本文对不同类型的嵌入式设备上内存损坏的影响进行了综合分析

作者提出了一种仿真（局部与完整）和运行时分析相结合的方式

实验表明本文采取的分析技术检测出了 100% 的内存损坏状态

局部仿真显著减少了 fuzzing 的吞吐量

使用完全仿真可以达到 100% 的检测率，同时大幅提高 fuzzing 效率

本文的主要贡献：

* 分析了对嵌入式设备进行 fuzzing 的挑战
* 根据检测嵌入式设备软件中内存损坏的难度，对嵌入式系统进行分类
* 评估了现实中不同内存损坏的效果
* 提出了可被用于提升嵌入式设备 fuzzing 效率的方法
* 提出了六种启发，被用于检测内存损坏引发的错误；嵌入式系统的固件可运行于局部或完全的仿真环境中
* 通过 Avatar 和 PANDA 框架的结合，实现了启发

---

## Fuzzing Embedded System

Fuzzing - 自动化的程序测试技术 - 向被测试系统输入随机数据

现代的 fuzzer 根据生成输入的策略可被分为：

* Mutation-based fuzzers
* Generation-based fuzzers
* Guided fuzzers

大部分 fuzzing 背后的原理及其工具都是用于测试桌面 PC 的

测试嵌入式设备存在许多挑战

### Classes of Embedded Devices

嵌入式设备和通用计算机的两个主要区别：

* 嵌入式系统被设计为满足专用需求
* 嵌入式系统通过一些连接传感器、执行器的外设与世界交互

本文中，根据它们使用的操作系统进行进一步分类

* OS 是安全特征的来源
* 负责处理从错误状态的恢复过程
* 更复杂的安全原语的基本要素

#### Type-I: General Purpose OS-based Devices

与传统的桌面或服务器版本相比

嵌入式版本的通用 OS 通常经过了最小化处理

比如 Linux kernel 被广泛用于嵌入式世界

结合了轻量级的用户空间环境

* busybox
* uClibc

#### Type-II: Embedded OS-based Devices

为嵌入式设备定制的操作系统

这些系统适用于低计算功耗的设备

通常不含 MMU 之类的加强版处理器特征

但内核代码和用户代码的逻辑分离依旧存在

* uClinux
* ZephyrOS
* VxWorks

运行于单一用途的用户设备上

#### Type-III: Devices without an OS-Abstraction

所谓的整体固件

操作基于一个单一的控制循环

外围设备触发中断

在这些设备上运行的代码可能是完全定制的

或基于操作系统库

* Contiki
* TinyOS
* mbed OS

最终的固件将会包含系统代码和用户代码（编译到一起）

从而组成一个整体固件

### Past Experiments

已有工作覆盖了：

* 不同方法
* 不同输入生成策略
* 不同种类的嵌入式设备

### Main Challenges of Fuzzing Embedded Devices

#### Fault Detection

大部分的 fuzzing 依赖于可见的 crashes 作为错误发生的标志

桌面系统提供保护机制，在遇到错误时触发 crashes

这种机制在嵌入式设备中很少出现

就算可以触发 crashes，监控这些 crashes 也十分困难

* 桌面系统的 crashes 通常伴随着 error messages
* 嵌入式系统缺乏等价的 I/O 功能

因此，liveness check 需要被用于检测测试的有效性

* active probing
  * 软件的状态被周期性地检测
  * 提供合法的输入，并评估对应的回应
* passive probing
  * 在不改变设备状态的条件下获取信息
  * 观察设备对于测试输入的回应
  * 观察可见的 crashes

#### Performance and Scalability

在桌面系统中，同个软件的多个实例可被并行 fuzzing

而对于资源受限的嵌入式设备来说不可行

此外，fuzzing 经常需要重启目标设备以重新建立一个干净的状态用于下一个测试

* 对于桌面系统、虚拟机来说，很容易
* 重置一个嵌入式设备可能需要几分钟

导致整个测试过程的缓慢

#### Instrumentation

> In the context of computer programming, __instrumentation__ refers to an ability to monitor or measure the level of a product's performance, to diagnose errors, and to write trace information. Programmers implement instrumentation in the form of code instructions that monitor specific components in a system (for example, instructions may output logging information to appear on the screen). When an application contains instrumentation code, it can be managed by using a management tool. Instrumentation is necessary to review the performance of the application. Instrumentation approaches can be of two types: source instrumentation and binary instrumentation.

Fuzzing 是黑盒测试

生成更加聪明的输入数据需要知道一些系统的状态信息

对于桌面系统来说

* compile-time instrumentation
* run-time instrumentation

的结合能够检测细微的内存损坏

然而，固件源代码镜像通常无法获得

在很多情况下很难重新编译软件

嵌入式系统可能包含一系列设备

* 每个设备有各自的操作系统、外设、处理器
* 一个综合的工具链很难获得

只能依赖二进制动态 instrumentation 框架

以上静态或动态 instrumentation 工具与目标 OS 和 CPU 体系结构紧密联系

* 因此无法支持 Type-II 和 Type-III 的嵌入式设备

---

## Memory Corruption in Embedded Systems

一系列的 BUG 将会导致桌面系统的 crashes

### Bugs, Faults, Corruptions & Crashes

固件和软件都用于完成特定的任务

因此它们可能存在 bugs - 将程序带入了一个计划外的状态

而这种状态可以被攻击者利用，从而被视为安全威胁

其中，一类 bug 被称为内存损坏

* 空间 - 越界访问内存
* 时间 - 访问一块已经不存在的内存

内存损坏自身可以导致可视的 crash

* terminated
* recovery - exception handlers

目前，以下机制能够触发这些 crashes

* stack canaries
* heap consistency checks

但在特定场合下，部分内存损坏不会导致可观测的或立刻的 crash

即所谓的 _silent memory corruptions_

* 程序继续执行，进入非计划的错误状态

这种错误只有在一定时间后才会被发现

* 当一个特定功能被请求
* 当接收到特定的事件序列

只要程序继续运行，就不会被视为是问题

* 但只要程序进入了错误状态，操作的完整性就无法被保证了

桌面系统也有可能发生 silent memory corruption

* 但频率极低
* 因为桌面 OS 有很多道保护
* 加固程序，使错误更有可能引发 crash
  * memory isolation
  * protection mechanisms
  * memory structures integrity checks

而嵌入式系统缺乏类似的机制

### Experimental Setup

实验目标：在不同的系统上触发相同的内存损坏条件，并分析它们是否：

* 引发可观测到的 crashes？
* 导致 silent 的内存损坏？

每种设备类型中选择一款设备，并使用完整的 GNU-Linux 桌面 OS 作为 baseline system

所有系统都基于 ARM 架构

作者重新编译了每个固件镜像

* _mbed TLS_ - 一个为嵌入式系统和桌面系统设计的 SSL 库
* _expat_ - 一款流行的 XML 转换器

内存损坏的问题都曾经被公开发布过

* Type-I : router
* Type-II : IP camera

编译了受威胁的应用，并 load 到设备上

对于 Type-III 类型的商用设备，获得可定制的固件很难

因此使用了一块开发板，将代码进入到固件中，并将固件 load 到设备上

### Artificial Vulnerabilities

作者的目标不是发现新 BUG，而是分析嵌入式设备上内存损坏带来的影响

作者在测试程序中引入了几个导致内存损坏的威胁

* stack-based buffer overflows
* heap-based buffer overflows
* null pointer dereferences
* double free vulnerabilities
* format string vulnerability

保证每个威胁都有其独立的触发条件

### Observed Behavior

实验目标：在发送恶意输入并触发其中的一个威胁后，观测每个设备的行为

Linux 桌面系统在每一个测试点都发生了 crash

而嵌入式系统不总是检测出错误

* 继续执行，没有可见的效果
* 尽管底层系统内存已经损坏

观测结果分类：

* R1 - Observable Crash (✔️) - 设备的执行中止，出错信息能够被轻易观测到
* R2 - Reboot (✔️) - 设备立刻重启
* R3 - Hang (❗) - 目标挂起，停止对新的请求进行回应，可能卡死在一个死循环中
* R4 - Late Crash (❗) - 目标系统继续执行一定时间，并在之后 crash
* R5 - Malfunctioning (❌) - 程序继续执行，但返回错误的数据和不正确的结果
* R6 - No Effect (❌) - 尽管内存损坏，目标继续执行，没有产生可见的副作用

R1 和 R2 由于迅速反应，可以使 fuzzer 立刻识别对应的恶意输入

R3 和 R4 对于 fuzzer 来说很难处理，尤其是 crash 被延迟了一定时间之后

* fuzzer 可能已经又发送多个输入了，识别哪一个输入出了问题很难
* 但错误依旧可以被观测到

R5 的情况更复杂，因为不会产生 crashes

* fuzzer 需要知道对于每个输入的正确输出是什么
* 在安全测试中，这种情况很少见
* 一种解决方式：在 fuzzer 两次连续的输入之间，插入一些已知输出的功能测试
* 但这种方法导致了相当大的 delay

R6 的情况最糟糕 - 由于设备内存的一小部分损坏，可能会在未来出现副作用或不确定的行为

实验表明，嵌入式平台提供的机制越少，系统越难检测出内存损坏

而且 Type-II 和 Type-III 设备很少触发 crash

尤其的，对于 format string vulnerability

* 嵌入式 Linux (Type-I) 和桌面 Linux 都报告了段错误 - 因为试图访问未映射的内存
* 但 Type-II 和 Type-III 设备继续执行 - 因为缺少 MMU
* MMU 在检测内存损坏中扮演了重要角色

类似的，对于 null pointer dereference

* 嵌入式 Linux 和桌面 Linux 在程序试图写入空地址时，立刻触发 crashes
* 对于 Type-III 的设备，程序继续执行，对应数据被写入了对应地址
  * 但由于固件的执行不依赖于内存，因此内存损坏不影响系统功能
* 对于 Type-II 的设备，设备未响应了一段时间后重启
  * _uClinux_ 的内核被映射在内存低地址端
  * 所以向 `0x0` 写入数据后损坏了内核空间
  * 重启可能是因为 watchdog 检测到了设备未响应，自动重启了设备

总体上讲，silent 的内存损坏在桌面系统中很少发生

而在嵌入式系统中发生是一种常态

而 fuzzing 通常依赖于可观测到的 crashes 来检测 bug

那么嵌入式系统中受限的错误检测导致 fuzzing 嵌入式设备具有挑战性

---

## Mitigations

从前面的实验可以看出

Fuzzing Type-I 的设备不存在太多挑战

发生在 Type-III 系统中的内存损坏很少引发立刻的 crash

如何在没有可信回应的条件下 fuzz 设备呢？

作者提出了六个选项

* 可能对测试者有用
* 分析了优势和局限

### Static Instrumentation

前提条件：源代码、编译工具链

大部分嵌入式设备的软件以 binary-only 的形式发放

因此获得源代码被认为是例外情况

而在获得源代码的前提下，拥有适合的编译工具链又是一种例外情况

* 用于重新编译固件

当测试者满足这些条件（比如测试者来自设备制造商）

* 他们可以标注源代码，以提供更佳的运行时信息
* 或引入 crash 处理功能

具体的 instrumentation 可包含多种技术的组合：

* 采集程序执行的返回值
* 测量代码覆盖率
* 修改 fuzzing 输入
* 为内存分配加入检测机制

大部分类似的工具对于嵌入式设备不可用

### Binary Rewriting

前提条件：二进制固件镜像，设备

当只能获得二进制固件时

二进制重写能够为代码进行标注

首先，固件需要被完全反汇编

其次，内存边界、数据结构在编译过程中完全丢失，需要被恢复以加入标注

* 局部反编译，非常具有挑战性

最终，嵌入式设备的内存使用通常经过了优化，以减少开销

* 为加入复杂的注解提供了很少的空间

### Physical Re-Hosting

前提条件：源代码、编译工具链、不同设备

分析者需要能够为不同的目标设备重新编译代码

* 对于 Type-II 的设备，编译一个 Type-I 的应用
* 对于 Type-I 的设备，编译一个桌面系统的应用

但是，依旧存在问题：

* 在新设备上找到的 bug 很难在源设备上重现
  * 不同的体系结构
  * 重新编译导致的一些变化
* 在源设备上可能会出现的问题，在新设备上不一定会出现

### Full Emulation

前提条件：固件映象、外围设备仿真器

分析者通常能够获得二进制的固件映像

* 直接从设备中提取
* 从固件的更新包中获得

已有论文指出，从 Type-I 固件中提取的应用可被虚拟运行

当完整的硬件文档可获得

并经过大量努力实现硬件仿真器

完全仿真 Type-III 的固件映像是有可能的

这个解决方法显著加强了 fuzzing：

* 测试可以在没有物理设备的条件下进行，加强了并行性
* 动态注解技术可以轻易被应用，仿真器可以采集大量运行时固件信息

缺点：

* 目标设备访问的所有外设都已经，且能被成功仿真
  * 太苛刻了...

总体来看，在仿真器中运行任意固件依旧是个问题

### Partial Emulation

前提条件：固件映像、设备

如果完全仿真在实际中无法进行

局部仿真依旧能够在测试时提供一些好处

总体思想是，使用仿真器转发与外围设备的交互

然而，这个觉得方案灵活性的提升带来了性能上的牺牲

* 由于与真实设备的额外交互

和可扩展性的牺牲

* 将每个仿真实例与物理设备配对

### Hardware-Supported Instrumentation

前提条件：设备、先进的调试功能支持

如果测试者能够使用先进的硬件标注机制访问物理设备

就可以在设备运行期间采集到足够的信息，提升错误检测的能力

* ARM's Embedded Trace Macrocell (ETM) & Coresight Debug & Trace
* Intel's Processor Trace (PT)

不同的追踪机制：

* branches
* all instructions
* all memory accesses

不同的技术依赖于不同的调试端口 - 专用追踪端口

* single wire debug (SWD) ports
* single wire output (SWO) ports

这些追踪硬件不一定可用

在低端设备上，尤其是 Type-III 设备上，这些功能因为成本问题被省略

开发设备可能会有类似的机制 - 比如微处理器生产前在 FPGA 上进行测试

有时，调试功能可能会因为防止第三方分析而被关闭

在测试真是设备的时候

找到可用且实用的硬件追踪支持的概率很小

### Summary

前三种解决方案需要测试者修改固件映像

* 静态 - 重新编译
* 动态 - 运行时标注

当进行第三方的安全测试时很难进行

后三种解决方案不需要修改固件

但需要额外的技术支持：

* 软件仿真器
* 硬件追踪支持

---

## Fault Detection Heuristics

提出了集中启发式的想法

独立实现

* 不仅可以进行实时分析，还可以进行事后分析
* 分析只依赖于从二进制代码执行过程中提取的信息

### Segment Tracking

最简单的方法 - 检测不合法的内存访问

核心思想 - 观察所有的内存读写，判断其是否发生在合法的位置

模仿 MMU 检测段错误的功能

需要内存读写和内存映射的信息

* 在仿真器中很容易获得

### Format Specifier Tracking

一种简单的方法，用于发现不安全的类 `printf()` 调用

检测进入 `printf()` 函数时，format string specifier 是否指向一个合法的位置

在最简单的例子中

如果没有动态产生的 format string specifier，那些合法的位置只会位于只读段

需要信息：

* 格式化处理函数的位置
* 进入这些函数时的寄存器状态
* 参数顺序

可通过逆向工程或对固件的自动化静态分析获得

### Heap Object Tracking

检测时间上和空间上与 heap 相关的 bug

评估 _allocation_ 和 _deallocation_ 函数的参数和返回值

记录堆对象的位置和大小

可以轻易检测出内存越界访问，或访问一个已经被释放的对象

但是这种思想需要依赖于：

* 执行的指令
* 寄存器状态
* 内存访问
* 内存分配和释放函数

需要通过逆向工程或其它先进技术获得这些信息

### Call Stack Tracking

目标是检测不返回调用者的函数

用于识别基于 stack 的内存损坏

覆盖了函数的返回地址

监控所有直接或间接的函数调用和返回指令

由于嵌入式设备通常是中断驱动的

这种思想可能会带来漏报

但是这种思想需要最少的信息 - 只需要执行指令的信息即可

### Call Frame Tracking

Call stack tracking 的先进版本

检测粗粒度的 stack 缓冲区溢出

栈帧通过追踪函数调用来定位

内存访问被检查，不能超过栈帧

需要信息：

* 识别执行指令
* 寄存器状态 
* 内存访问需要被监视

### Stack Object Tracking

细粒度检测栈变量的越界访问

内存的读写被监视，并记录对应的位置和大小

需要追踪执行指令和内存访问

以及栈变量的详细信息

可以从二进制代码中自动化提取

---

## Implementation

* _PANDA_ - 一个动态分析平台
* _Avatar_ - 嵌入式设备的动态分析工具

 本文的实现需要修改这两个框架

### Live Analysis System Description

实验系统的三个核心组件：

* Avatar
* PANDA
* PANDA 分析插件

Avatar 通过启用 Type-III 设备的局部仿真进行动态二进制分析

* 本质是提供设备和仿真器之间的自动转换，转发 I/O 访问
* 后台仿真器 S2E 被替换为 PANDA
* 不仅可以在执行期间进行分析，还可以产生轻量级的记录
* 用于识别 crash 的根源

PANDA 基于 QEMU

它的插件系统允许 hook 多个事件

* 访问物理内存
* 翻译
* 执行翻译后的块

且 PANDA 中已经实现了几个分析插件

* `callstack_instr`
  * 对函数调用和返回注册 callbacks
  * 提供目前调用栈的信息

总体上，使用 PANDA 来模拟固件

依赖其插件系统获取实时反馈

Avatar 负责将执行和内存访问重定向到物理设备

### State Caching

Avatar 保存并重放设备初始化之后的状态

重启所有外设的性能开销较大

---

## Experiments

在局部仿真和完全仿真的场景中进行实验

目标：将这些思路集成到实时 fuzzing 实验中

提供错误检测机制

缓和嵌入式操作系统中缺乏对应机制的问题

然而，这样方式将不可避免带来一些性能上的开销

### Target Setup

为一款 Type-III 的设备编译了 _expat_ 应用

使用了四种配置进行 fuzzing

* NAT: Native
  * Fuzzing 直接在实际设备上进行
  * 没有任何错误检测机制
  * 作为 baseline
* PE/MF: Partial Emulation with Memory Forwarding
  * 固件被仿真
  * 访问外围设备通过转发 I/O 内存访问到实际设备实现
* PE/PM: Partial Emulation with Peripheral Modeling
  * 固件被仿真
  * 使用 Avatar 内的专用脚本模仿外围设备
  * 不需要物理外围设备
* FE: Full Emulation
  * 固件和外围设备在 PANDA 中完全仿真

使用了设备初始化结束后的快照

输入受威胁软件的数据通过一个简单的文本协议经过串口连接输入

### Fuzzer Setup

_boofuzz_ - 一款流行的 Python fuzzing 框架

* 生成并发送恶意输入
* 定义目标监控
* hooks

Fuzz over user-defined communication channels

实现了串口通信和 TCP 通信

显然，触发错误的输入肯定会占用更大的开销

强制使 fuzzer 在一定的概率下产生会触发错误的输入

* 0 / 0.01 / 0.05 / 0.1

增加了一个简单的 liveness check

* 在每次 fuzz 输入后，fuzzer 接收到设备的回应，并判断是否与期待的输出匹配

当接收到的回应与期待的值不同 或 连接超时

fuzzer 报告一个 crash，并重启设备

* fuzzer 通过 _boofuzz_ 来重启物理设备或仿真器

#### False Positives and False Negatives

目标是展示嵌入式设备在错误检测中的局限

和使用启发式的方法克服这个问题的可行性

而不是评估某个特定实现的效果

#### Fault Detection

在没有任何保护机制的条件下 fuzzing 是低效的

单纯使用 liveness check 只能够检测：

* stack-based buffer overflow
* format string vulnerability

因为这两个 bug 会导致设备未响应，从而导致超时

其它类型的 bug，虽然成功触发了内存损坏，但无法被 fuzzer 探测到

实验表明，六个启发的组合能够检测出所有的内存损坏

#### Performance

在不同的触发概率下

在一小时的 fuzz 会话中

目标可以处理的输入数量

PE/MF 的速度慢了一个量级 - 这是由于固件和外围设备的通信造成的

使用了启发对 PE/MF 的开销几乎没有影响

* 因为瓶颈位于转发 MMIO 的请求

然而，对于 PE/PM 和 FE 来说，启发带来的开销显而易见

完全仿真的速度比真实设备快，只要触发内存损坏的概率较低

主要原因有三个：

* TCP 的吞吐率比串行接口高
* 虽然固件被仿真，但仿真的时钟速率比低端专用设备高
* 检测到内存损坏伴随着强制重启，更高的触发比例意味着大量的重启时间

最重要的结果是，执行在 PANDA 中的固件，执行组合启发式算法后

在低于实际触发概率下，比真实的嵌入式设备运行快

* 前者能检测出所有的威胁
* 后者只能检测出两种

---

## Discussion

本文的测试强调了以下三个方面：

1. Relying only on liveness tests is a poor strategy
2. While full emulation is the best strategy, emulators are rarely available
   * 第三方测试者通常缺乏有关硬件的细节信息
   * 实现一个完整的仿真器需要大量的人力劳动
3. Partial emulation can lead to accurate vulnerability detection, with a significant performance impact
   * 当完全仿真无法进行时
   * 局部仿真可以在准确率上获得相同效果
   * 代价是在效率上降低一个量级
   * 仿真还可以记录执行过程，用于之后的问题溯源

---

## Conclusion

本文提出了 fuzzing 嵌入式设备的挑战

分析了嵌入式设备的类型、内存损坏的类型

Silent 的内存损坏不止会在嵌入式系统中出现

但桌面系统会有各种各样的保护措施

将错误转换为可被观测到的 crashes

而嵌入式设备上基本没有这类措施

本文研究了 silent 的内存损坏在不同类型的嵌入式设备上的效果

并评估了一系列措施来缓和这个问题

基于 Avatar 和 PANDA 实现了六个不同的实时分析思路

这个工作证明了好的仿真器的重要性

---

## Summary

这篇文章的关注点比较偏门

而且篇幅大部分集中在前半部分

也就是问题的发现和分类

在最后的具体实现中

借助了已有的工具做了一个比较片面的实验

只给了一些解决问题的思路吧

可能算是开辟了一个新的研究方向

---

