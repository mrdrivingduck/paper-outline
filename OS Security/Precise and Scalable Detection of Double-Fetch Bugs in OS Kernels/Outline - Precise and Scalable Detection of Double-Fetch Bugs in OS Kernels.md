# Outline

## Precise and Scalable Detection of Double-Fetch Bugs in OS Kernels - S&P 2018

Created by : Mr Dk.

2019 / 07 / 04 12:19

Nanjing, Jiangsu, China

---

## Introduction

OS 内核中的 BUG 可能引发：

* privilege escalation
* information leaks
* denial of service

Double-fetch BUG 是一类特殊的竞争条件 BUG

* 内核读取同一特定的用户空间内存超过一次
* 假设每次访问内存时，内存中的数据都不被改变

然而，这种假设不一定成立

* 一个并发执行的用户线程可以在内核多次读取期间修改内存内容
* 从而导致数据不一致性
* 进一步导致绕过参数合法性检查、缓冲区溢出等

目前，已有研究发现 Window OS 上的 double-fetch 导致的特权提升

内核因为性能原因会多次读取用户空间内存 - _multi-read_

* 取一块可变长度的 message，最大长度为 4 KB
* 如果每次分配 4 KB 的缓冲区，并拷贝 4 KB，浪费内存空间和 CPU 资源

因此，通常的做法是：

* 读取 4 B 的 `size`
* 分配 `size` 大小的缓冲区，并读取 `size` 大小的 message

Linux 内核中存在超过 1000 次 multi-read

其中，多少次 multi-read 是真正的 double-fetch 呢？

为了解决这个问题，必须：

1. 定义和区分 double-fetch 和 multi-read
2. 自动查证 multi-read 是否是一个 double-fetch

相关工作由于基于不够准确的定义，导致了很多的 false positives & false negatives

在本文中提出了 DEADLINE

一种静态检测 multi-read 和 double-fetch 的自动化工具

并得到较高的准确率和覆盖率

DEADLINE 覆盖了所有可以在 x86 架构下编译的驱动、文件系统、外围模块

* Linux 和 FreeBSD 内核

为了使 DEADLINE 能够检测 double-fetch BUG

首先需要正式建模，从数学上对 multi-read 和 double-fetch 进行区分

本质上，当以下条件发生时，multi-read 会转变成 double-fetch：

1. 两次 fetch 来源于重叠的用户空间内存
2. 两次 fetch 中重叠的数值产生了一定联系
3. 这种联系可以在竞争条件中被破坏

DEADLINE 通过两个步骤检测 double-fetch：

1. DEADLINE 试图找到尽可能多的 multi-read
   * 对每一个 multi-read 构造执行路径
   * 通过将内核源代码编译为 LLVM IR，并进行静态代码分析
2. DEADLINE 跟随执行路径判断 multi-read 是否是一个 double-fetch bug
   * DEADLINE 将 LLVM IR 转换为符号表示 symbolic representation (SR)
     * 每个变量由符号表达式表示
   * 根据 double-fetch 的定义解决 SR 的符号约束
   * 正确的结果意味着 double-fetch 的存在

挑战：

DEADLINE 需要系统性地搜索路径，采集 multi-read

剪枝不相关的指令、线性化执行路径

对于 double-fetch 的判断

* 符号化内存读写
* 模仿常用库函数的行为

与相关工作中的词汇匹配不同

本文使用了程序分析的方法采集 multi-read

并通过后续的切片、循环展开，对执行路径进行剪枝

此外，还提出了如何利用和修复 double-fetch bug 的方法

本文的主要贡献：

* 对于 double-fetch 的正式、准确的定义，避免了人工检查 multi-read 的必要
* DEADLINE 的设计与实现 - 基于特别为 double-fetch 检测剪裁后的符号执行模型，自动检测内核代码
* 找到了 Linux 内核中的 23 个新 BUG 和 FreeBSD 内核中的 1 个新 BUG
* 提出四个通用的策略，避免 double-fetch 出现

---

## Background

### Address Space Separation

在现代 OS 中，虚拟地址空间被分为用户空间和内核空间

用户空间被每个进程分开，使用户进程有一种独占地址空间的假象

用户内存可以被在该地址空间中运行的所有线程访问，也可以被内核访问

而内核空间只能被内核访问

虽然内核可以访问用户空间内存

内核通常不会直接对用户进程提供的地址进行解引用

* 否则内存损坏将会使整个系统 crash

如果内核需要用户空间中的数据

会先将用户空间中的数据在内核中拷贝一份

并对内部的拷贝数据进行操作

* 通过 transfer function 实现
  * Linux - `copy_from_user` & `get_user`
  * FreeBSD - `copyin` & `fuword`
* 不仅进行数据传递，还具有用户空间合法性检查、处理非法地址或缺页等功能

大量 `__user` 标记用于保证用户空间内存只能通过 transfer function 被访问

### Multi-read as a Common Practice

由于系统调用可以传入的参数有限

通常传入内核的是一个指向用户空间结构体的指针

* 理论上来说，任何 multi-read 都可以被设计为 single-read
* 可以但是没有必要

内核通常会先 fetch 请求头部

再基于头部中的信息构建一次完整的请求

* size checking
* type selection
* shallow copy
* dependency lookup
  * 存在多个处理请求的 handler
  * 先基于 header 中的信息选择一个 handler
  * 再将整个请求拷贝进内核
* protocol/signature checking
  * 请求头部先被预定义的协议数据检查
  * 内核会提前拒绝与协议不符合的请求
* information guessing
  * 当特定信息缺失时，内核会基于多次 fetch 猜测部分信息
  * 并在之后 fetch 完整的数据
  * 可能导致数据不一致

### Reflection on Prior Works

之前的工作由于经验主义规则的不准确性

导致了大量 false positives 和 false negatives

没有办法保证人为定义的规则能够覆盖所有的 multi-reads

此外，假设在循环之间和函数调用之间没有 double-fetch 是很危险的

* 存在部分出现在循环中的 double-fetch bug
* 存在部分出现在过程间的 double-fetch bug

Coccinelle 不支持宏展开

* 当 `CONFIG_COMPAT` 被启用时，一些因兼容性原因而设计的函数会生效
* 其中可能包含 BUG

已有工作采用了动态方法

将 multi-read 定义为对同一用户虚拟地址空间进行至少两次读操作：

* 两次读操作都从内核代码中执行
* 发生在一次预定义的时间间隔中

动态分析的本质决定了这种方法很难应用于整个内核

* 代码覆盖率低

这种方法暗示了两种局限：

1. 这种方法局限于只能在可以被仿真的硬件设备中寻找 bug
2. 即使在被仿真的设备驱动中，实际能被测试的代码量取决于测试套件

最重要的是，没有工作能够从定义上给出区分 multi-read 和 double-fetch 的区别

---

## Double-Fetch Bugs: A Formal Definition

当以下四项条件满足时，DEADLINE 将一个执行路径标记为 double-fetch bug：

1. 至少有两次从用户空间内存的读取操作
   * 即，必须是一次 multi-read
   * 由 _transfer function_ 来识别
2. 两次 fetches 必须覆盖用户空间内存中的一段重合区域
   * 如果条件满足，将 multi-read 称为 overlapped-fetch
3. 在两次 fetch 期间，重叠的内存区域必须存在一定关联
   * 控制依赖和数据依赖
4. DEADLINE 无法证明在第二次 fetch 之后这种关联是否已经存在

形式化定义：

* Fetch - `(A, S)`
  * `A` 代表用户空间地址
  * `S` 代表拷贝内存的长度
* Overlapped-fetch
  * 两次 fetch - `(A0, S0)` 和 `(A1, S1)`
  * 重叠 - `A0 ≤ A1 < A0 + S0` 或 `A1 ≤ A0 < A1 + S1`
  * 对应地，用 `(A01, S01)` 来表示两次 fetch 重叠的内存区域
  * 用 `(A01, S01, i = [0, 1])` 分别表示两次 fetch 拷贝的内存
* Control dependence
  * `V` 被视为控制依赖，当且仅当 `V ∈ (A01, S01, 0)`
  * 且 `V` 必须服从于使第二次 fetch 发生的条件集合 `[Vc]`
  * 为了证明 double-fetch 不会出现，必须证明由 `(A01, S01, 1)` 构造的 `V'` 也服从于 `[Vc]`
* Data dependence
  * `V` 被视为数据依赖，当且仅当 `V ∈ (A01, S01, 0)` 且 `V` 被使用
    * 比如赋值给了其它变量，参与运算，或传入子函数
  * 为了证明 double-fetch 不会出现，必须证明由 `A01, S01, 1` 构造的 `V'` 必须满足 `V' == V`

---

## DEADLINE Overview

Double-fetch 的正式建模激发了作者使用符号检查进行 double-fetch 的检测

* symbolic checking

与动态分析不同，需要给定一个具体的输入来执行代码

符号执行提供了更好的通用性

* 解决一个能够满足条件的特定场景
* 证明一个条件永远无法被满足

使得 DEADLINE 的检测精确度上升，也减少了人工的工作

此外，符号检查作为一种静态分析方法

不受硬件可用性和机器配置的限制

在理论上可应用于所有的内核驱动、文件系统、外围模块等

它还使 DEADLINE 能够从任何点开始进行路径搜索

而不是只能从固定的入口处开始

* 比如系统调用入口，或内核启动

然而，在进行符号检查之前

DEADLINE 需要采集尽可能多的 multi-reads

对于每一次 multi-read，DEADLINE 为其构造执行路径

* 首先将内核源码编译为 LLVM IR
* 静态分析 IR，识别 multi-reads，剪枝相关联的执行路径

使用 LLVM IR 的原因：

1. LLVM IR 保留了大部分 DEADLINE 需要的信息
   * 类型信息、函数名、参数
   * 只有 `__user` 标记丢失，但可以通过在 LLVM 元数据中标记进行恢复
     * 用于识别用户空间的内存对象
2. LLVM IR 是一种 single static assignment (SSA) 形式
   * 近似模仿了符号执行的逻辑
3. 使用 LLVM IR 也可以利用一些 LLVM 的分析 passes

总体过程：

1. 扫描内核，采集尽可能多的 multi-read 及其执行路径
2. 在每个执行路径上进行检查，查看 multi-read 是否变为了 double-fetch bug

算法 1：

1. 扫描内核，找到由 transfer function 标记的所有 fetches
2. 对找到的每一个 fetch，在 control flow graph 中扫描其前向和后向，寻找其它 fetch
3. 如果找到其它 fetch，将其标记为 fetch pair，组成 multi-read
4. 为每对 fetch pair 构建所有可能执行到这两次 fetch 的执行路径
5. 对于每一条构建的执行路径，通过符号执行引擎，基于正式定义，检测是否存在 double-fetch bug

---

## Finding Multi-Reads

### Fetch Pairs Collection

静态列举所有可能出现的 multi-reads

* 在 CFG 中至少可以静态到达的路径

方法：

1. 识别所有的 fetches (通过 transfer function)
2. 为整个内核构建一个完整的过程间 CFG
3. 对每一对 fetches 进行可到达性检查

由于内核代码的庞大性和复杂性，2 和 3 几乎不可能

DEADLINE 使用了一种自底向上的方法

从每一次 fetch 出发，在其被显式调用的函数中

扫描所有此次 fetch 可以到达的指令

在这些指令中：

* 要么标记为找到了一个 fetch pair
* 要么将包含 fetch 的子函数内联，重新执行搜索

除了两次 fetch 以外，其封装函数 `Fn` 也与该 pair 相关联

这使 DEADLINE 不需要从固定的入口和结束点构造执行路径

除去了大量不相关指令

#### Indirect Calls

在内核中使用，模仿多态行为

DEADLINE 不试图解决这类问题

* 事实上，这种问题只能在运行时解决

DEADLINE 保守地识别非直接调用的所有潜在调用目标

### Execution Path Construction

DEADLINE 被给定一个三元组 `<F0, F1, Fn>`，代表 multi-read

DEADLINE 的目标：

1. 在封装函数 `Fn` 中找到所有连接 fetch `F1` 和 `F2` 的执行路径
2. 为每一条路径剪枝与 fetches 不相关或没有影响的指令

两个目标都能够通过标准的程序分析技术解决

* 第一个目标能够通过遍历 `Fn` 的 CFG 解决
* 第二个目标能够通过以下条件对 CFG 进行切分解决：
  * 一条指令对 fetch 有影响当且仅当：
    * 内存地址或大小要么直接通过该指令获得，要么与该指令有约束
  * 一跳指令被 fetch 影响当且仅当：
    * 由 fetch-in 的值获得或受 fetch-in 的值约束

#### Linearize an Execution Path

将路径线性化为一系列 IR 指令

对于没有循环的路径，线性化简单表述为基本块的拼接

对于存在循环的路径，需要将循环展开

DEADLINE 只将循环展开一次，暴露了它的局限：

* 无法找到在多次循环中 fetch 重叠内存的 double-fetch bug

但事实上，在内核中，这种情况几乎永远不会出现

---

## From Multi-Reads to Double-Fetch Bugs

### A Running Example

符号执行，证明 `size == attr->size`

由于无法证明，说明这是一个 double-fetch bug

* `$X` 代表变量的符号值
* `@X` 代表 `$X` 指向的内存对象
* 如果 `$X` 不是指针，那么 `@X` 为 `nil`

一个内存对象可以被表示为 - `<i, j, L>`

* `i` 和 `j` 分别表示为内存访问起始和结束字节
* `L` 可以为 `K` 或 `U`，分别代表内核空间和用户空间
* 对于用户空间，`U` 可以为 `U0` 或 `U1`，分别表示是第一次还是第二次 fetch

### Transforming IR to SR

将 LLVM IR 转换为 SR 与在指令路径上符号执行 LLVM 指令相同

每个变量都有一个 SR

指令和函数调用定义了如何传递这些 SR 和约束条件

所有的 SR 来源于一些列 root SR

* root SR 可能来自函数参数、全局变量，或内核、用户空间的内存数据块
  * 函数参数和全局变量被视为 root SR，因为它们的值不由执行路径上的指令定义
  * 内核或用户空间的内存数据块的初始内容位置，所以也视为 root SR

```assembly
%3 = add i32 %2 16
```

```assembly
$3 = $2 + 16
```

三种 LLVM 指令需要被特殊处理：

#### Branch Instructions

在符号执行的过程中

不产生新的路径搜索过程

* 只跟随符号执行之前准备好的特定路径

在遇到条件分支时，检查之前的路径条件，决定走哪一条分支

并使用这些信息来推断走这一条分支必须满足的条件

这样一来，DEADLINE 相当于在约束条件集合中加入了约束

并可以用于之后的判断

这与传统的符号执行中，复制状态并试图覆盖两个分支的做法不同

#### Library Functions and Inline Assemblies

内核没有标准库的概念

一些常用功能位于内核代码树的 `/lib` 下

这些库函数的功能可被分类为五种：

1. 内存分配 - `kmalloc`
2. 内存操作 - `memcpy`
3. 字符串操作 - `strnlen`
4. 同步操作 - `mutex_lock`
5. 调试或错误报告函数 - `printk`

作者使 DEADLINE 不会符号执行到这些函数中

而是人工地为每一个函数写了符号规则

通过符号的方式捕获了函数参数以及返回值之间的关系

在 Linux 内核中，只有 `45` 个函数

在 FreeBSD 内核中，只有 `12` 个函数

对于内联汇编，其中很大一部分和 double-fetch 无关

* 因此会被提早过滤，不会出现在执行路径中

对于经常出现在指令路径中的，本文作者也人为编写了符号执行规则

对于其余的，忽略 (假设它们没有影响)

### Memory Model

传统的符号执行将内存建模为一个线性数组

依赖 `select-store` 来表示内存读写

* `select(a, i)` 返回了数组 `a` 位置 `i` 的值，模仿了内存读
* `store(a, i)` 返回了一个与 `a` 相同的新数组，但在位置 `i` 上包含一个值 `v`，模仿了内存写

在这种内存模型中，如果在两次 fetch 之间没有任何 `store`，那么两次 fetch 的结果将总是返回相同的值

* 对于单线程程序，或交叉执行的多线程程序来说，是对的
* 但是对于内核代码访问用户空间来说不成立
  * 因为用户进程可能在两次 fetch 之间修改值，但在 trace 中不会有 `store`

DEADLINE 扩展了这种模型，对每次读取加入了一个单调递增的值

来表示不同 fetch 的值可能不一样

此外，DEADLINE 用一个字节数组代表每个内存对象

并将每一个指针映射到一个内存单元上

使用了一些启发式的方法创建这种映射：

1. 不同的函数参数或全局变量被假设为指向不同的内存对象
2. 新分配的指针被假设为指向新的内存对象
3. 当赋值发生时，内存对象也被转移
4. 对于任何其它指针，如果无法证明其值位于已经存在的对象中，假设其指向新的对象

在检测 double-fetch 时，只考虑两次 fetch 的地址指针指向相同用户空间内存对象的情况

### Checking Against the Definitions

完成从 IR 到 SR 的翻译后

调用 SMT solver 检查所有的条件是否满足

1. 检查 `F0 = <A0, S0>`，`F1 = <A1, S1>` 是否共享重叠的内存区域
   * 在路径约束条件中加入 `(A1 ≤ A0 < A1 + S1) || (A0 ≤ A1 < (A0 + S0))`
   * 重叠被表示为 `<N, i, j>`，被表示为用户空间内存对象 `N` 的 `i` 至 `j` 字节重叠
   * 如果没有重叠发生，则说明 multi-read 是安全的
2. 对于每一个识别到的重叠，进一步检测其控制依赖和数据依赖
   * 控制依赖
     * 采集 `C0 = @N(i, j, U0)` 和 `C1 = @N(i, j, U1)`，证明 C1 与 C0 相同，甚至比 C0 更严格
   * 数据依赖
     * 证明 `C0 == C1`
   * 如果没有依赖关系，说明这是一次多余的 fetch

---

## Implementation

DEADLINE 实现为 LLVM pass 的形式

### Maximize Code Coverage

### Compiling Source Code to LLVM IR

---

## Findings

---

## Exploitation

### Leaking Information

当内核空间的数据被拷贝回用户空间时

被修改后的 size 能够将大量内核空间的内存拷贝到用户空间中

造成内核信息泄露

### Bypassing Restrictions

绕开了内核提早拒绝用户空间请求的代码

### Denial-of-Service

读取不合法的内核内存区域

DEADLINE 不会产生 double-fetch 的产生原因

1. double-fetch bug 不会产生明显的信号，需要人为定义规则来衡量 bug 是否被利用成功
2. 即使定义了 bug 的利用规则，也很难构造它们，因为 bug 被利用的位置可能距离 bug 发生的位置很远

---

## Mitigation

修复 double-fetch 的四种方法：

### Override with Values from the First Fetch

忽略第二次 fetch 中获得的值

用第一次 fetch 的值来覆盖

### Abort if Changes are Detected

在第二次 fetch 之后加入合法性检查

如果两次 fetch 的值存在出入，则退出

### Incremental Fetch

跳过第一次 fetch 中拷贝过的内存

只拷贝第一次 fetch 中没有拷贝过的内存

### Refactor into a Single-Fetch

可以将 double-fetch 重构为 single-fetch

会导致更多行的代码

#### Preventing Exploits with Transactional Memory

Double-fetch 的根源在于用户空间内存访问缺少原子性和一致性

在第一次 fetch 之前标记事务开始

在第二次 fetch 之后标记事务结束

---

## Summary

顶不住了 😫

实在是看不懂

以后静态分析的文章不想看了

编译原理学得真的不好 看不懂

太 tm 抽象了艹

---

