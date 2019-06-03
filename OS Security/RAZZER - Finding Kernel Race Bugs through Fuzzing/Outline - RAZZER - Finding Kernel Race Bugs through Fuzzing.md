# Outline

## RAZZER: Finding Kernel Race Bugs through Fuzzing - S&P 2019

Created by : Mr Dk.

2019 / 06 / 03 22:09

Nanjing, Jiangsu, China

---

## Introduction

数据竞争是 OS 中有害行为的根源

* 数据竞争导致的循环加锁，会导致系统死锁而未响应
* Safety assertions 会导致内核自动重启，导致拒绝服务
* Memory corruptions - 特权升级攻击

现有工作的局限：数据竞争来源于内核中的不确定性行为

理解数据竞争需要：

* 准确的控制流和数据流信息
* 准确的并发执行信息（可能会被外部因素影响）

RAZZER 是一个基于 fuzzing 的竞争检测器

同时采用了静态分析和动态分析

* 通过静态分析，过度估计所有可能出现数据竞争的点
* 两个阶段的动态分析
  * 单线程 fuzzing - 找出可能引发竞争的单线程输入程序
  * 多线程 fuzzing - 构建一个多线程程序，利用定制的管理程序，故意阻塞触发竞争

避开了外部因素的影响，使竞争行为确定

通过 LLVM 的 pass 进行 points-to analysis，实现了 RAZZER 的静态分析

管理程序通过修改 QEMU 和 KVM 实现

两个阶段的 fuzzing 针对内核的系统调用接口

RAZZER 与 Syzkaller 与 SKI 相比

* 用时更短
* 更加有效率

RAZZER 能够指出特定的数据竞争发生在哪

以及竞争发生时的调用栈

本文的贡献：

* 面向竞争的 fuzzer - 专为检测竞争设计的 fuzzer
* robust implementation，利用了 LLVM 和 KVM/QEMU，不需要人为改动
* 实用性 - 找到了 Linux 内核中的 30 个竞争

---

## Problem Scope and Design Requirements

### Problem Scope and Terminology

数据竞争：输出依赖于两个不确定性事件发生的时间或顺序

数据竞争发生于满足以下三个条件的两条内存访问指令之间：

1. 访问相同的内存地址
2. 至少有一个是写指令
3. 它们被并发执行

如果满足这些条件，内存访问结果将是不确定的

计算结果依赖于执行顺序

本文用 `RacePair<cand>` 表示可能满足这些条件的两条指令

用 `RacePair<true>` 表示确认满足这三个条件的两条指令（子集）

数据竞争可被分为：良性的和有害的

良性的竞争是开发者希望或故意触发的，忍受一定程度的变差

* 比如维护 performance counter
* 避免缓慢的数据碰撞
* ？
* `RacePair<benign>`

有害的数据竞争消极地影响程序的运行时行为，用 `RacePair<harm>` 表示

由于计算结果的不确定性，有害数据竞争可能导致多种后果

* 死锁 - 导致系统无响应
* Safety assertions - 导致系统重启
* 内存安全性违反 - 优先级泄露
* 使用 `RacePair<harm>` 表示

#### Race Example: CVE-2017-2636

### Design Requirements

#### Design Requirements

##### R1

Find an input program which executes `RacePair<cand>`

* 寻找一个多线程的用户空间程序
* 每个线程执行 `RacePair<cand>` 中的每条指令

##### R2

Find a thread interleaving for the input program which executes `RacePair<cand>` concurrently

只靠 R1 不能保证 `RacePair<cand>` 并发执行，触发数据竞争

分析应当能够识别特定的线程交叉执行顺序，使 `RacePair<cand>` 被并发执行

已有工作只满足了 R1 和 R2 中的一个

本文分析了用于识别竞争 BUG 的两种常用技术

* fuzz testing
* random thread interleaving

#### Requirement Study: Traditional Fuzz Testing

传统的 fuzz testing 精力集中在 R1

由于 R2 没有被考虑

不能够有效发现数据竞争

Syzkaller 的核心是随机生成（或修改）的系统调用

每个系统调用被随机分配给用户线程 A 或 B

虽然使线程交替执行了可能产生竞争的代码

触发竞争的机会非常低

#### Requirement Study: Thread Interleaving Tools

随机线程交叉工具，专注于 R2

探索所有可能的线程交叉场景

通常来说不考虑 R1

* 所以只能运行已有的程序，不能在巨大的代码空间中有效搜索

线程交叉工具基于随机调度

* 在 R2 上的效率受到严重限制
* SKI 随机选择一个内核线程，执行直到遇到内存访问指令
* 随机选择另一个内核线程，重复这个过程

搜索了所有可能的线程交叉案例

但较大的搜索空间导致效率低下

---

## Design

混合方法，同时利用静态分析和动态分析

### Identifying Race Candidates

静态分析的目标是识别内核中所有的 `RacePair<cand>`

* 每个 `RacePair<cand>` 包含两条内存访问指令
* 在运行时有潜在威胁导致竞争

Points-to 分析是一个比较流行的方法

* 因为其表示了每个内存指令指向哪里

局限性：

* 准确率
  * control-flow / data-flow 信息只能在运行时被获得
* 性能
  * `O(n^3)`
  * `n` 表示被分析程序的大小

RAZZER's points-to analysis

* Context-insentitive
* Flow-insensitive
* Field-sensitive

过度估计了 `RacePairs<cand>`，同时排除了一定不会发生竞争的情况

* 访问结构体中的不同成员变量

为了解决性能问题

RAZZER 定制了内核的分割分析

* 通过模块组件将内核分割
* 对每个组件进行预分析

基于源代码目录结构拆分内核

进行每个模块的预分析时，RAZZER 也会提供核心模块

* `fs` / `net/core` 两个模块经常被其它子模块使用

静态分析没有考虑同步原语

* 如果考虑这些原语，能够降低误报率

### Per-Core Scheduler in Hypervisor

由于内核线程穿插的随机性，竞争条件是不确定性的

因此，RAZZER 将对应的内核运行于定制的虚拟化环境中

避免了外部时间的不确定性

* RAZZER 在访客用户空间中运行多线程程序
* 这些程序试图在访客内核中触发竞争

此外，为了能够完全控制线程穿插，RAZZER 修改了管理程序

管理程序为客户内核提供一下特性：

1. 为每个虚拟 CPU core 设置断点
2. 在客户内核命中断点后，继续内核线程的执行
3. 检测一个竞争是否真正发生

#### Setup Per-Core Breakpoint

RAZZER 提供新的 hypercall interface - `hcall_set_bp()`

从而使客户内核能够建立 per-core 断点

该函数由客户内核调用，包含两个参数：

1. `vCPU_ID` - 指定断点设置在哪个虚拟 CPU 上
2. `guest_addr`  - 指定了断点被安装的客户内核的地址空间的地址

一旦接收到 hypercall

* 管理程序在 `guest_addr` 地址上安装硬件断点
* 断点仅在 `vCPU_ID` 上有效

有两个特别重要的任务：

1. 准确控制每个 CPU 核心的执行行为
2. 在运行特定的系统调用时，决定一个断点是否被特定的内核线程触发

为了完成第一个任务，RAZZER 利用了硬件断点

* 由虚拟化的调试寄存器支持
* 保证了断点事件总是被最先传输到管理程序
  * 利用了 Intel VT-x 中的 Virtual Machine Control Structure (VMCS)
  * Interrupt bitmap
  * 每一位对应客户的中断号
  * 当 Interrupt bitmap 中的某一位被置位
  * 中断立刻触发 `VMEXIT` 事件，并立刻传输到管理程序

在 x86 平台上，RAZZER 将客户地址保存在虚拟化的调试寄存器中

* 调试寄存器是虚拟化的，因此被每个虚拟 CPU 维护

RAZZER 保障了只有在对应的虚拟 CPU 执行时才触发硬件断点

通过对应硬件断点中断置位 VMCS，RAZZER 能够率先检测到硬件断点

第二个任务：

管理程序可获知 CPU 的上下文，但不可获知客户内核的上下文

而客户内核可能运行了其它服务

如果简单地安装了硬件断点，断点可能被无关的程序触发

* 断点触发事件不管用户程序是什么都会发生

RAZZER 通过 virtual machine introspection (VMI) 决定内核线程上下文

管理程序通过 VMI 检索内核线程的 id

RAZZER 管理程序从而能够根据线程号决定断点是否被触发

#### Resume Per-Core Execution

在两个客户内核线程停在了对应的断点地址上（`RacePair<cand>`）

RAZZER 继续了两个虚拟 CPU 的执行 - 两个线程并发执行 `RacePair<cand>`

RAZZER 需要做一个重要的决定：哪个内核线程应当先被执行？

* 因为一些 BUG 只会在特定的执行顺序下显现

管理程序提供 `hcall_set_order()` 接口以控制执行顺序

* 在函数中，特定虚拟 CPU 的 id 先被执行
* 只执行一条指令，保证特定的虚拟 CPU 率先被执行
* 之后，RAZZER 恢复所有虚拟 CPU 的执行

#### Check Race Results

当两个断点并发命中时

管理程序检测 `RacePair<cand>` 是否会导致竞争

检测访问的目标地址是否相同

* 如果相同，则给定的 `RacePair<cand>` 被提升为 `RacePair<true>`

从技术上来说，通过对每个 `RacePair<cand>` 的指令进行反汇编

并获取每个虚拟 CPU 寄存器中的具体数值

计算出目标地址是否相同

### Two Phased Fuzzing to Discover Races

RAZZER 的 fuzzing 包含两个阶段：

* 单线程 fuzzing 阶段
* 多线程 fuzzing 阶段

每个 fuzzing 阶段包含两个组件：

* Generator - create a user program
* Executor - run the program

#### Single-Thread Fuzzing

单线程 generator 生成 `Pst`，带有一系列随机系统调用的单线程程序

接下来，单线程 executor 运行每个 `Pst`

测试每次 `Pst` 的执行是否覆盖 `RacePair<cand>`

如果覆盖，将 `Pst` 注解信息后，传递至下一阶段

##### Single-Thread Generator

单线程 generator 产生一个单线程的用户空间程序

产生一个随机系统调用的序列

测试内核的行为

RAZZER 通过以下两个策略构造 `Pst`：

* generation
* mutation

RAZZER 根据预定义的系统调用语法，随机生成 `Pst`

* 包括所有可用的系统调用
* 以及每个系统调用的合理参数数值

通过这些语法，RAZZER 试图构造一个合理的用户程序

* 随机选择一个序列的系统调用
* 随机输入每个系统调用的参数
* 系统调用的返回值被随机传入下一个系统调用中

利用了 Syzkaller 中的预定义的系统调用语法

##### Single-Thread Executor

在运行每个 `Pst` 的同时，完成以下两个任务：

任务一：

* 利用底层内核的支持，采集 `Pst` 执行时的覆盖
* 如果 RAZZER 检测到执行覆盖了 `RacePair<cand>` 中的指令
  * 一个系统调用导致内核执行了 `RacePair<cand>` 中的一条指令
  * 另一个系统调用导致内核执行了 `RacePair<cand>` 中的另一条指令
* RAZZER 注解详细信息：
  * 两个产生竞争的系统调用
  * `RacePair<cand>` 的地址

在一个 `Pst` 中可能会有多个 `RacePair<cand>` 被匹配

* RAZZER 为每个 `RacePair<cand>` 拷贝一份 `Pst`，并分别注解

任务二：

如果 `Pst` 的运行结果产生了没有被执行过的覆盖

RAZZER 将该 `Pst` 保存，并重新输入单线程 generotor

（构造 fuzzing 语料库）

##### Example: CVE-2017-2636

#### Multi-Thread Fuzzing

对于每一个 `RacePair<cand>`，generator 将 `Pst` 转换为 `Pmt` （多线程版本）

多线程 executor 运行每个 `Pmt`

如果 `Pmt` 被管理程序确认会触发竞争，提升为 `RacePair<true>`

并通过重新输入 generator 继续变换 `Pmt`

如果 `Pmt` 使内核 crash，RAZZER 会产生详细报告

##### Multi-Thread Generator

将注解后的 `Pst` 作为输入，输出 `Pmt`

* 多线程版本
* 利用注解信息确定性地触发竞争

输入参数：

* `Pst`
* `Pst` 中产生竞争的系统调用的索引 `i` & `j`
* 产生竞争的对应指令 `RP_i` & `RP_j`

算法产生两个工作线程，每个线程绑定在独立的虚拟 CPU 上

RAZZER 利用了内核已有的特性绑定线程

提取 `Pst` 中所有的系统调用，并随后分割到两个不同的线程中

* 第一个线程放置第 `i` 个系统调用之前的系统调用
* 后一个线程放置第 `i+1` 到第 `j-1` 个系统调用

为了引导 `RacePair<cand>` 的执行顺序，插入 `hcall_set_order()`

随后，为了确定性地触发竞争

RAZZER 在插入引发竞争的系统调用之前，插入 `hcall_set_bp()` 设置断点

随后，RAZZER 利用 `hcall_check_race()` 检测 `RacePair<cand>` 是否引发竞争

为了引发有害竞争后的行为，RAZZER 加入了最近生成的系统调用

##### Multi-Thread Executor

运行 `Pmt`，检测 `RacePair<cand>` 是否真的触发竞争

运行时，在插入竞争指令之前

通过 hypercalls 设置了每个 core 的断点

之后调用 `hcall_check_race()` 确定竞争是否真的被触发 - 触发条件：

1. 管理程序是否捕获到了两个断点
2. `RacePair<cand>` 访问的具体内存地址是否相同

由于导致竞争不一定会导致危害

RAZZER 调用竞争后的系统调用，区分出有害的竞争条件

##### Example: CVE-2017-2636

---

## Implementation

RAZZER 的静态分析基于 LLVM 和 SVF

提供了 points-to analysis 的框架

由于 SVF 的使用对象是用户空间程序

本文修改了 SVF，使其能够处理内核中的内存分配和释放

RAZZER 的静态分析生成了源代码中的两条指令的集合

* 其中分别表明源文件名和行号
* `RacePair<cand>` 在构建内核期间被翻译为机器地址

RAZZER 的管理程序实现在 QEMU 上，使用了 KVM 进行硬件加速

当断点命中，使用 _Capstone_ 进行反汇编，以检测 `RacePair<cand>`

Fuzzer 通过 Syzkaller 实现

---

## Evaluation

### Experimental Setup

创建了 32 个虚拟机

* 16 个用于单线程 fuzzing
* 16 个用于多线程 fuzzing

### Target Kernel Preparation

不需要目标内核的任何改动

首先使用 LLVM 套件，产生 bitcode，进行静态分析

随后，使用 GCC 构建内核，运行在 RAZZER 的管理程序上

### Newly Found Race Bugs

测试了多个版本的 Linux 内核

运行了大约 7 星期

找到了 30 个有危害的竞争

#### Efficiency in Discovering New Harmful Races

non-race bugs - 由单线程 fuzzing 发现

race bugs - 由多线程 fuzzing 发现

#### Reliability and Security Impacts

导致不可预测的、不确定性的 crash

可能被攻击者滥用，发动优先级泄露攻击

拒绝服务攻击

覆盖内核的证书结构体

#### Very Old Races in the Kernel

发生该竞争的可能性很低

在发生竞争的指令之间

只有三条非内存访问的指令（时间窗口极小）

由于 RAZZER 强制了确定性的线程穿插，因此能够检测出来

#### Root Cause Information

RAZZER 产生的详细报告，有助于内核开发者快速修复

因为竞争 Bug 难以重现

### Effectiveness of Static Analysis

#### Performance Overhead of Partitioned Analysis

#### Correctness of `RacePairs<cand>`

由于静态分析拆分了内核

可能导致一些漏报

#### Effectiveness of `RacePair<cand>`

RAZZER 产生的 `RacePair<cand>` 只占所有内存访问指令对的 `0.05%`

### Hypervisor Overhead

RAZZER 由于利用了 hypercall 保证了虚拟 CPU 上确定性的程序行为

与管理程序进行通信需要额外开销

### Comparison Study of the Fuzzing Efficiency

---

## Related Work

### Dynamic Fuzz Testing

### Dynamic Thread Scheduler

### Dynamic Race Detector

### Static Analysis

---

## Discussion

### False Negatives in Static Analysis

RAZZER 依赖静态分析的结果

如果静态分析漏报，会导致 RAZZER 漏报

漏报主要由于拆分内核的分析 - 模块间竞争

### Applying RAZZER for Other Systems

---

## Summary

一遇到这种和 LLVM 有关的文章就看不懂了。。。

实现时用了很多虚拟化的工具

没用过 所以也没有概念

这一块实在是很薄弱 有空还是得多折腾折腾呢

---

