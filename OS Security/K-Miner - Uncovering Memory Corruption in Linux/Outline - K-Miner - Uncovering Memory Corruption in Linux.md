# Outline

## K-Miner: Uncovering Memory Corruption in Linux - NDSS 2018

Created by : Mr Dk.

2019 / 05 / 17 21:20

Nanjing, Jiangsu, China

---

## Abstract

OS 内核正成为攻击目标，通过攻陷内核：

* 使攻击者绕开 OS 的安全机制
* 使攻击者能够控制整个系统

Linux 内核由低层语言实现

提供有限的存储器安全保护

使攻击者能够利用 memory-corruption（内存损坏？）

发动复杂的运行时攻击

很多防御手段在运行时保护 OS 的安全（run-time monitor）：

* control-flow integrity, CFI
* 目标是防止内存损坏的征兆出现，不从根源上消除问题（OS code 中的 BUG）

寻找 BUG 可利用静态分析而被自动化

* 现有方法局限于程序局部的检查
* 由于内核代码量过大，在可扩展性上存在挑战
* 目前没有工具可被用于 OS 内核的全局静态分析

K-Miner 是用于分析 Linux 等大型商用 OS 内核的新框架

* 利用内核高度标准化的接口进行
  * 可扩展的指针分析
  * 全局、上下文敏感的分析

通过程序内分析，K-Miner 有组织且可信地揭露了几种不同等级的内存损坏威胁

* dangling pointers

  > 悬空指针：一个指针所指的内存被释放后，这个指针就被悬空了。
  >
  > 访问悬空指针，结果随机。可能导致程序功能不正常，也可能导致程序崩溃。
  >
  > 如果受到影响的是其它功能，问题通常很难定位。

* use-after-free

  > 悬空（dangling）的指针被创建，并指向一个内存中被释放的对象。
  >
  > - 一块内存区域被分配并且有一个指针指向它
  > - 内存区域被释放但是指针是可用的
  > - 指针被使用并访问之前释放掉的内存

* double-free

  > 同一个指针被 `free` 两次。

* double-lock

  > 上锁两次的意思？？

K-Miner 利用了 LLVM 编译器套件

---

## Introduction

OS 内核是所有现代软件平台的基石

* 提供服务
* 为用户应用提供接口

通常通过硬件机制与用户空间隔离

* 内存保护
* 处理器优先级

内核代码的内存损坏为无特权的用户敞开了大门：

* 颠覆控制过程和数据结构
* 控制整个系统

之前的研究工作主要是 OS 内核的运行时保护

* 大部分是运行时监控器（run-time monitor）
* 提供对策、防止攻击者利用内存损坏

### Run-time Monitors vs. Compile-time Verification

Monitors 根据攻击者的能力对其进行建模

并被设计为能够防御一些特定种类的攻击

* Control-flow integrity (CFI) 被用于检测 control-flow hijacking attacks
* 然而无法防御只针对数据的攻击者
* 因此，虽然有 monitor 的存在，无法防御不同种类的攻击

不同防御的组合能够使内核抵御多种类型的攻击

因此，只要内核中有内存损坏的代码，OS 就可能受到新的类型的攻击

一种替代方案：在系统部署前分析整个系统

* 对于少于 10000 行代码的微内核来说可行
* 构造形式模型（formal model），根据模型来检测实现的正确性
* 对于大型内核来说不可行
  * 代码量大
  * 大量使用没有安全保证的机器码
* 即使对于微内核来说
  * 构造模型并检验需要 10 个人多年的工作才行

虽然动态分析的检测已经很成功了

静态分析有着不同的好处：

* 彻底的静态分析涵盖了超出程序执行的代码
* 能够对检测出的异常结果给出强有力的解释
* 如果静态检测没有报告异常，那么可以声称不存在内存损坏威胁

### Static Analysis of Commodity Kernels

静态分析面临很严重的可扩展性挑战

因此，所有的分析框架都局限于程序内分析：

* 每个函数的本地检测
* 没有一个框架支持程序内的 __数据流分析__ :
  * 保守地估计程序行为
  * 可信地揭露全局指针关系导致的内存损坏

OS 内核的精确数据流分析存在巨大的挑战：

* 代码量过于庞大，过于复杂
* Linux - `2400` 万行代码，编译内核在顶配机器上也需要几个小时

随着代码量的增大，可能的执行路径数量会以指数方式增长

同时，为了保证分析的彻底性

分析时应该考虑所有的路径和状态

* 剪枝或跳过部分代码会导致不可信的结果
* 进行分析所需要的资源很快超出现实中的阈值

因此全局分析和过程间分析在已有的方法中无法实现

### Goals and Contribution

本文作者利用了内核软件的一个明显、独特的性质：

内核提供给用户空间的接口是高度标准化的

因此作者利用系统调用接口作为起点

根据各自独立的执行路径将内核代码分开

* 显著地减少了相关路径的数量
* 使作者能够进行复杂的、过程间的数据流分析
* K-Miner - 第一个能够进行复杂数据流分析的静态分析框架

拆分内核代码存在一些挑战：

* 全局数据结构的频繁重用
* 每个系统调用和全局内存状态（context，上下文）的同步
* 复杂的、深度嵌套的指针别名关系

本文的贡献：

* 内核代码的全局静态分析
* 框架原型实现
  * 提供了多种已知威胁的分析方式
  * 可扩展
  * 基于 Web 的 UI
  * K-Miner - 在 LLVM 的基础上实现
* 广泛评估
  * 应用到多个 Linux 发布版的所有系统调用上
  * 自动化、可扩展
  * 报告了两个 use-after-return 威胁

---

## Background

### Data-Flow Analysis

静态分析的大致思想：

* 将一个程序和一个预编译的内容作为输入
* 找到使给定内容为 `true` 的所有路径

举例：

* Liveness analysis
* Dead-code analysis
* Type-state analysis
* Null-ness analysis

比如，对于一个程序的 null-ness 分析

可通过查看 pointer-assignment graph (PAG) 进行

另一个常用的数据结构是 inter-procedural control-flow graph (ICFG)

* 可被用于进行 path-sensitive 的分析

taint 分析和 source-sink 分析

* 通过相关的 value-flow graph (VFG) 追踪独立的内存对象

静态分析中追踪某个个体数值被称为 data-flow analysis

* 最简单的方法：枚举所有的程序执行路径
* 用给定的属性测试每一张图
* Meet Over all Paths (MOP)

在一般场景中，MOP 是不可判定的

* By reduction to the post correspondence problem

但 MOP 可被所谓的 Monotone Framework 估计

* 一个数学对象的集合
* 分析程序行为的规则

### Memory-corruption Vulnerabilities

内存损坏由操作系统代码中的安全性 BUG 引起

运行时攻击利用这些 BUG：

* 注入恶意代码
* 重用已有代码，使用恶意输入
* 损坏完整的数据结构，获取更高的特权

内存损坏威胁通常根据根源来进行分类：

* Integer overflows (IO)
* Use-after-free (UAF)
* Dangling pointers (DP)
* Double free (DF)
* Buffer overflow (BO)
* Missing pointer checks (MPC)
* Uninitialized data (UD)
* Type errors (TE)
* Synchronization errors (SE)

每一种内存损坏都违反了定义好的程序行为

可能导致运行时任意的程序行为

攻击者可以通过故意触发这些错误

完成无意识的、恶意的行为

对于 OS，将内核代码暴露给用户空间的主要接口是系统调用

在本文的方法中

作者结合了上述类型的不同数据流分析

报告内核代码中可在用户空间程序中通过系统调用获取到潜在的 BUG

在本文的概念验证实现中，作者专注于实现：

* Dangling pointers (DP)
* Use-after-free (UAF)
* Double free (DF)

比如：

> 悬挂指针发生于：一个内存地址被分配给一个指针
>
> 这块内存随后变得不可用了
>
> 可能发生在 __heap allocation__ 上
>
> * 这块内存被 `free` 了，但指针依旧可用
>
> 可能发生在 __stack allocation__ 上
>
> * 包含分配对象的 stack frame 变得不合法了
> * 由于分配过程中的嵌套的 `return` 导致

---

## K-Miner

### Goals and Assumptions

识别并报告内核的用户空间接口中潜在的内存损坏 BUG

使开发者能够在部署 OS 前修复 BUG

对于运行时可能存在的恶意进程，作如下假设：

* 攻击者能够控制一个用户空间进程，并能够通过系统调用攻击内核
* OS 与用户进程隔离（通过虚存或不同的特权级）
* 攻击者无法通过模块向内核注入恶意代码（现代操作系统要求模块被密码学签名）
* K-Miner 应当能够报告可能被恶意进程触发的内存损坏威胁

这一假设迫使攻击者只能通过纯软件手段获取内核特权

但一个完整的、被认证的内核依旧可能被硬件攻击破坏完整性

### Overview

这一框架建立与已经存在的 LLVM 编译套件的基础上

#### Step 1

编译器接收两个输入：

* 一个配置文件 - 包含一些选中的内核特性
  * 用户可以自定义特性的 enable 或 disable
  * 如果特性被 disable，在实现中这部分代码将不会被包含进来
  * 分析结果只对特定的 <内核代码，配置文件> 合法
* 内核代码

编译套件根据配置文件转换内核代码

* 进行语法检查，构造 abstract syntax tree (AST)
* 编译器内部将 AST 转换为所谓的 IR (intermediate representation)

#### Step 2

编译套件将 IR 输入 K-Miner

K-Miner 将每一个 system call 函数入口作为起点，生成：

* Call graph (CG)
* Value-flow graph (VFG)
* Pointer-analysis graph (PAG)
* 其它内部数据结构

除此之外，还计算了一系列全局分配的内核对象

* 任意一个系统调用都能够到达这些对象

这些数据结构生成完毕后，K-Miner 就可以开始进行实际的静态分析了

对每一种类型的威胁，都分别有独立的分析

多种威胁的分析每次只对一个系统调用进行

* 利用之前生成的数据结构

这些分析由 context-sensitive 的 value-flow analysis 实现：

* 考虑给定系统调用的 control flow，并表示在 call graph 中
* 追踪过程间的上下文信息

#### Step 3

如果潜在的内存损坏 BUG 被检测出来了

K-Miner 将生成报告，包含所有的相关信息

* Affected kernel version
* Configuration file
* System call
* Program path
* Object

### Uncovering Memory Corruption

K-Miner 能够分析多种不同类型的威胁

每种分析 pass 都会利用与该类威胁相关的数据结构

检测特定条件是否为 `true`

推断内存和指针对于分析内核行为来说至关重要

因此，global context 和 pointer-analysis graph 是很多分析的基础

独立的内存对象在分配点处被分配，分配点遍及整个内核

潜在指向内存对象的变量在每个系统调用中被 PAG 追踪

* Forward analysis 推断存储单元个体的过去行为
* Backward analysis 决定了未来的行为

也可以通过结合特定的独立分析 pass，寻找 double-free 威胁：

1. 找到内存对象的分配点和回收点

   正向处理 VFG 的每个分配点，决定可到达的回收点

2. 通过反向跟随回收点，重建 <source-sink> 对

3. 重新分析前向路径，检查额外的回收点

任何存在超过一个回收点的路径将会被报告为 double-free 威胁

* 产生大量 false positives
* 判断第一次重新分配是否 dominates 第二次重新分配（？）

类似地，提供了以下几种威胁的检测条件：

* Dangling pointers
* Use-after-free
* Double-lock

### Challenges

#### Global State

大部分内存损坏威胁涉及指针

以及其指向的内存中的对象的状态或类型

进行过程间的指针分析存在效率问题

由于过程间分析允许全局变量

局部指针可能只是全局变量的一个别名，而不会产生局部影响

本文的分析是过程敏感的

这些别名关系不总是静态的

但可以在遍历 control-flow graph 时被更新

为了可以进行复杂的全局分析

* 只考虑与现在分析的 call graph 和上下文信息有关的 value flows

#### Huge Codebase

在不省略任何代码的条件下

* 减少一次分析中的代码量
* 允许 IR 的重用

通过根据系统调用接口拆分内核

* 显著减少了分析路径的数量
* 考虑到了所有的代码
* 重用了重要数据结构（比如内核上下文）

#### False Positives

误报由对程序行为的粗粒度的过度估计产生

导致产生了用户无法处理的过多报告

本文作者很仔细地设计分析

利用尽可能多的信息来减少运行时不可能发生的

或过粗粒度的估计

此外，还采取了去重、过滤等措施

#### Multiple Analysis

K-Miner 组合了多个不同分析 pass 的结果

此外，独立分析互相依赖对方的中间结果

框架需要根据目前已检查的代码同步中间结果

* 利用 LLVM 的 pass 基础设施
* 导出 IR，过一段时间后部分重新导入

---

## Implementation

K-Miner 框架基于 LLVM 编译套件和 SVF 分析框架

* LLVM 提供基本的底层数据结构、简单的指针分析、pass 基础设施、bitcode 文件格式
* SVF 提供额外的指针分析、value-flow 依赖图的稀疏表示法

修改内核的构建过程并链接

产生 bitcode 版本的内核镜像

* 该内核镜像可被作为 K-Miner 的输入

包含四个分析步骤：

1. LLVM-IR 被传递到 K-Miner 中进行预分析，初始化并填充内核全局上下文
2. 上下文信息被用于分析独立的系统调用
3. BUG 报告经过过滤，减少误报
4. 经过排序的报告由威胁报告引擎进行渲染

### Global Analysis Context

K-Miner 存储的全局上下文代表了基于源代码构造的内存对象模型

有效管理全局上下文信息是进行高度复杂的代码分析的先决条件

此外，需要保证上下文足够准确

* 以支持接下来的分析中的准确报告

这也是为什么在预分析阶段

K-Miner 模仿内核的执行，以建立并追踪内核全局上下文信息

#### Initializing the Kernel Context

内核通常在引导初期运行时通过填充全局数据结构初始化内存

* 任务清单
* 虚存位置

通过调用一系列特定函数完成 - `Initcalls`

* 只会被调用一次
* 由宏指令注解

宏使编译器将这些函数放置在专用代码段中

这个段只会在引导或驱动装载期间被执行

因此，这个代码段占据的大部分内存可在机器完成引导后被释放

为了初始化内核全局上下文

作者通过模仿 `Initcalls` 的执行，填充全局数据结构

上下文信息有几百 MB - 导出到磁盘上的文件中

在之后的 data-flow 分析阶段重新导入

#### Tracking Heap Allocations

通常，用户空间中的程序通过各种 `malloc()` 的变种在运行时动态分配内存

内核中有很多不同的动态分配内存的方法：

* slab allocator
* low-level page-based allocator
* various object caches

为了能够动态追踪内存对象

必须编译一个用于堆分配分配函数的列表

K-Miner 借助这个列表

通过将这些函数的调用处标记为堆分配的起点

这样，内核内存分配可在接下来的 data-flow 分析中被追踪

#### Establishing a Syscall Context

由于每个系统调用都会运行随后的分析

作者为每个系统调用建立专用的内存上下文

通过每一个系统调用的 call graph

采集全局变量和全局函数的使用

通过将该上下文信息与全局上下文进行反复比较

作者建立了一个准确的内存上下文描述

### Analyzing Kernel Code Per System Call

虽然分析独立的系统调用已经显著减少了相关代码量

资源需求还是过大

> 运行一个简单的指针分析使服务器立刻 out of memory

主要原因在于 - 函数指针

初级方法假定所有的全局变量和函数是被任意系统调用访问

这个估计相对保守，但低效

之后用了几个技术来改进初级方法

#### Improving Call Graph Accuracy

首先从分析 call-graph 开始

通过分析 call-graph 中所有函数的 IR

决定一个函数指针是否可达

（是否可以被局部变量访问到）

这样，作者能够采集可能的目标函数

提升初始 call-graph 的准确率

通过分析 call-graph，得到了潜在的目标函数列表

基于列表，接下来进行两个阶段的指针分析

#### Flow-sensitive Pointer-Analysis

首先进行 inclusion-based 指针分析

解决早先采集的函数指针的依赖问题

为了进一步提高准确率

进行了第二次指针分析，将 control-flow 考虑进来

再一次地最小化相关符号的数量

并为每个系统调用产生了准确的上下文环境

将这些作为中间结果，以每个系统调用为单位保存

可被用于接下来的 data-flow 分析

#### Intersecting Global Kernel State

将一个系统调用的上下文信息和全局内核上下文结合

### Minimizing False Positives

当过度保守估计程序行为时，经常发生

比如，如果不考虑 control flow

分析会假设指针之间的别名关系在运行时不会共存

#### Precise Data-Flow Analysis

......

#### Sanitizing Potential Reports

Data-flow 分析完成后

再次确认结果：

* 不可能的条件
* 运行时不可能的路径

并消除不同上下文中的相同结点，将它们合并至同一个 report 中

### Efficiently Combining Multiple Analysis

为了有效进行多个 data-flow 分析

框架利用了多种优化和分析技巧

#### Using Sparse Analysis

Value-flow graph 是 data-flow 分析中的重要数据结构

* 是一个直接的过程间图，追踪指针变量的任何操作

VFG 在内核代码中捕获指针的 def-use chains 来进行追踪

VFG 通过四个步骤构造：

1. 指针分析决定了每个变量的指向信息
2. 地址变量的非直接定义和使用对应的指令被变量注解
3. 函数被转换为 Static Single Assignment 形式
4. 将每一个 SSA 变量的 def-use 连接起来

#### Revisiting Functions

对一个函数，用所有可能的上下文环境，仅处理一次

并储存中间结果

分析时，检查该函数是否已被访问过

并重用中间结果

#### Parallelizing Execution

分析过程互相独立，因此可以并行处理

---

## Evaluation

5 个不同的 Linux 内核版本

平均需要 25 min 检测一个系统调用

### Security

#### Coverage

作者的目标揭露所有通过系统调用可以访问到的内存损坏

因此，需要确保分析过程能够获得所有可用于安全估计内核运行时行为的相关信息

* DP
* DF
* UAF
* 可扩展

分析中最重要的因素是它们的底层分析结构：

* PAG
* VFG
* 上下文信息

由于 VFG 和上下文信息由 PAG 获得

它们的准确率直接依赖于 PAG 的构造

本文的指针分析有如下两点假设：

1. 根据系统调用拆分内核是彻底的
2. 从 `Initcalls` 获得的内核上下文是完整的

满足这两点假设需要形式证明，但本文认为这两点假设是合理的

对于第一点：

系统调用由独立的进程触发，以同步的方式执行

调用进程暂停，直到系统调用执行结束

而中断和进程间通信可能使其它进程异步查询内核

这符合拆分原则，因此这些操作在不同的上下文中

特别地，K-Miner 允许考虑多个内存上下文

对于第二点：

反正就是 ojbk

#### Real-world Impact

* CVE-2014-3153 - dangling pointer
* CVE-2015-8962 - double-free

### Performance

#### Analysis Run Time

#### Memory Utilization

### Usability

K-Miner 可被集成到标准的 Linux kernel 构建进程中

---

## Discussion

### Future Work

分析 module API 也可以使用类似的方法

* 虽然 module API 没有系统调用那样高度标准化
* 但很多驱动的高层函数以符号的形式导出到内核镜像中

---

## Related Work

Data-flow 分析需要一个顶层函数和初始程序状态

这些要求被用户空间程序的 `main()` 函数和预定义的全局变量自然地满足了

然而，OS 是事件驱动的，而且用户空间可以通过系统调用影响内核执行

所以对于 data-flow 分析来说，没有单一入口

---

## Summary

这是读过的第一篇操作系统安全相关的论文

里面很多知识点都还不熟悉

要学的东西不少

但是挺贴近代码层的

我应该还算是有兴趣...

---

