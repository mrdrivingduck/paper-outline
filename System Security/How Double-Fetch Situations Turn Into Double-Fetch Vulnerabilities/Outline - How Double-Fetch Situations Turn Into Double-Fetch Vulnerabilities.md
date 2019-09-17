# Outline

## How Double-Fetch Situations Turn Into Double-Fetch Vulnerabilities: A Study of Double Fetches in the Linux Kernel - Usenix Security 2017

Created by : Mr Dk.

2019 / 05 / 25 20:33

Nanjing, Jiangsu, China

---

## Abstract

第一个系统分析 Linux 内核中 double-fetch 威胁的方法

使用 pattern-based 分析

找到了 Linux 内核中 90 个 double-fetch

* 其中，57 个存在于驱动中
* 之前的动态分析无法检测到

对 90 个 double-fetch 进行人工分析

* 推断了 double-fetch 发生的三个典型场景

基于 Coccinelle 静态分析了 double-fetch

* 应用于 Linux、FreeBSD、Android 的内核
* 找到了 6 个之前没有被报告的漏洞
  * 其中 4 个存在于驱动中
  * 3 个是可被利用的安全威胁

所有的漏洞都被内核开发者确认并修复

论文中的方法也被 Coccinelle 接受，被集成到内核补丁审查中

论文也提出了自动修复 double-fetch 漏洞的方法

---

## Introduction

多核心的硬件使并发程序越来越普遍

尤其是在：

* 操作系统
* 实时操作系统
* 计算密集型系统

然而，并发程序会产生很多难以检测的并发漏洞

主要被归类为：

* Atomicity-violation 原子性违反
* Order-violation 顺序违反
* Deadlocks 死锁

Data race - 数据竞争 - 也是一个并发程序中经常出现的场景

会在两个线程访问一个共享的内存时，且至少一个访问是写操作时发生

它们的相对访问次序不会被任何同步机制保证

因此数据竞争经常会导致 atomicity-violation 或 order-violation

除了在线程之间发生外，数据竞争还可能发生在 内核 和 用户空间 之间

Double-fetch 首次出现时，被用于形容一种 Windows 内核威胁

* 内核将用户空间的同一数据取了两次

> Double-fetch bug 发生于内核 读取 并 使用 用户空间的同一数据两次
>
> （并希望这两次的数据是一样的）
>
> 而一个并发的用户线程在两次使用的时间窗口之间修改了该数据
>
> 导致了数据不一致，可能引发缓冲区溢出等威胁

第一个研究 double-fetch 的人

通过追踪内存访问进行 __动态分析__

* 代码覆盖率较低
* 无法分析需要在特定硬件上执行的代码
* 如果无法访问驱动或仿真，则无法分析设备驱动的代码

并且他们只分析了 Windows，没有分析 Linux、FreeBSD、OpenBSD 等

他们不仅提出了 _how to find_，还提出了 _how to exploit_

因此审计完整的内核代码，尤其是驱动，变得尤为重要

设备驱动是重要的 kernel-level 程序

* 在操作系统和设备间提供接口，成为软硬件的桥梁
* 占据内核代码的很大一部分 (Linux 4.5 的 `44%`)
* 内核 bug 的重灾区

`26%` 的 Linux 内核代码运行在 x86 平台以外

* 基于 x86 平台的动态分析无法覆盖

所以提出了能够覆盖所有驱动和平台的静态分析

本文作者发现：

* 大部分的 double-fetch 不是 double-fetch bug
  * 虽然 fetch 了两次，但实际只使用了一次 fetch 的结果

因此细化了分析方法，检测实际的 double-fetch bug

并分析了 Linux、Android、FreeBSD

经过分析，大部分的 double-fetch 和 double-fetch bug 存在于驱动程序中

* 因此动态分析无法找到这些漏洞

---

## Background

数据如何在内核、驱动、用户空间之间交换？

数据竞争和 double-fetch 是如何发生的？

### Kernel/User Space Protection

现代操作系统中，内存被划分为内核空间和用户空间

* 内核空间是内核代码执行和存储内部数据的地方
* 用户空间是用户进程运行的地方

每个用户进程运行在各自的地址空间中

只能访问该地址空间内的内存地址

这些虚拟地址被内核映射为实际的物理地址

保证了空间的隔离

操作系统提供了和用户空间进行数据交换的特殊机制

* 在 Windows 中

  * 使用 Input and Output Control (IOCTL) method
  * 或 shared memory object method - 共享内存对象？
  * 与共享内存区域类似

* 在 Linux 和 FreeBSD 中

  * 提供了在内核空间和用户空间进行安全数据交换的函数
  * _transfer functions_
  * Linux
    * `copy_from_user()`
    * `copy_to_user()`
    * `get_user()`
    * `put_user()`
  * 不仅可以交换数据，还提供保护机制，防止非法内存访问

因此，Linux 中的 double-fetch 会多次调用 _transfer function_

### Memory Access in Drivers

设备驱动是内核的组件

负责使内核对硬件设备进行使用、信息交换

驱动的典型特征：

* 对同步和异步操作的支持
* 被多次打开的能力
* 驱动中的 bug 可能导致整个系统控制权限的泄露
* 驱动需要经常从用户空间中拷贝信息

在 Linux 中，所有的设备都由文件表示

* 以便从用户空间可以直接访问设备

内核在 `/dev` 目录下为每个驱动创建文件

* 用户空间进程可以通过文件 IO 的系统调用访问设备

驱动提供所有与文件相关的系统调用的具体实现

* `read()` - 驱动需要将数据拷贝到用户空间
* `write()` - 驱动需要从用户空间取得数据

__驱动通过 transfer function 完成这些功能__

同样，double-fetch 会发生在多次调用 transfer function 的时候

### Double Fetch

Double-fetch 是一种特殊的内存竞争条件

发生在内核和用户空间之间

从技术上讲

* Double-fetch 发生在内核函数中，比如系统调用
* 内核函数从用户空间的同一个内存地址上取了两次数据
  * 第一次用于检查或确认
  * 第二次用于使用
* 同时，在两次取数据的时间窗口内
* 一个并发的用户线程修改了该内存的数据
* 当内核函数第二次取数据使用时
  * 可能导致 buffer overflow / null-pointer crash 或其它情况等

Double-fetch 可以被分为以下四类：

#### Benign Double Fetch

不会产生危害

* 可能存在额外的保护机制
* Double-fetch 的数据没有被使用两次

#### Harmful Double Fetch

Double-fetch bug

在特定场合下，可能导致内核崩溃

* 需要用户空间进程触发数据竞争条件

#### Double-fetch Vulnerability

一旦数据竞争条件可被利用

则 double-fetch bug 转变为 double-fetch vulnerability

* 通过缓冲区溢出，导致特权提升、信息泄露或内核崩溃

在论文中，对良性和恶意的 double-fetch 都进行了分析

* 可能目前代码是良性的
* 可能在代码发生更新时转变为恶性（比如 double-fetch 的数据被重用）
* 当其中一次 fetch 是多余时，有一些良性的 double-fetch 导致了性能下降

很多方法试图分析数据竞争

* 静态分析
  * 不用运行程序就可以进行分析
  * 缺乏上下文，产生大量的 false reports
* 动态分析
  * 通过执行程序检测数据竞争
  * 通常控制线程调度器，使 bug 出现的可能性提高
  * 运行时开销大
  * 驱动代码需要有特定的硬件才能被执行

目前没有方法能够分析 double-fetch -

1. Double-fetch 与普通的数据竞争不同，因为竞争条件分隔在内核与用户空间之间
   1. 普通的数据竞争，`read()` 和 `write()` 都在相同的地址空间中
   2. 识别对相同内存位置的读写操作即可
   3. Double-fetch 中，内核只有两次 `read`，而 `write` 位于用户空间中
2. 在 Linux 中，内核空间和用户空间通过 transfer functions 进行交互
   1. 意味着访问内存不是简单的解引用
   2. 基于解引用的分析不再有用
3. Linux 中，double-fetch 更为复杂
   1. first fetch - 拷贝数据
   2. first check or use - 第一次使用
   3. second fetch - 再一次拷贝相同数据
   4. second use
   5. 通过匹配 fetch 操作可以找到所有的 fetch
   6. 但是使用数据的方式有很多种，在文中，use 被定义为：
      1. condition check - 读取数据用于比较
      2. assignment to other variables
      3. a function call argument pass
      4. a computation using the fetched data

### Coccinelle

Coccinelle 是一个程序匹配转换引擎

由一种专用的 SmPL (Semantic Patch Language) 语言对 C 代码进行匹配、转换

一开始目标是与 Linux 驱动并行开发

现在已被广泛用于寻找、修复系统中的代码

其遍历 control-flow graph 的策略是 temporal logic CTL (Computational Tree Logic)

且 Coccinelle 实现的模式匹配是 path-sensitive 的，有着更好的 code coverage

被高度优化，以提升遍历所有执行路径时的性能

此外，由于 pattern-based 分析直接应用于源代码

一些被定义为宏的操作 - `get_user()` / `__get_user()` 不会被展开

从而有利于 _transfer function_ 的识别

因此，Coccinelle 是一个适合的工具

---

## Double Fetches in the Linux Kernel

分为两个阶段：

* 第一阶段
  * 利用基本的 double-fetch 匹配规则，识别多次调用 _transfer function_ 的函数
  * 人工分析
  * 对场景进行分类
* 第二阶段
  * 基于人工分析得到的细粒度结果
  * 使用更为精细的规则进行匹配
  * 找到所有的 double-fetch bug 和 double-fetch vulnerabilities

### Basic Pattern Matching Analysis

一个内核函数使用 _transfer function_ 从用户空间相同地址取数据至少两次

在 Linux 中，匹配 `get_user()` 和 `copy_from_user()` 及其所有变种

Pattern 允许拷贝的目标地址和拷贝数据的大小不同

但拷贝的源地址（用户空间地址）必须相同

在研究中，发现存在 false positives：

* _transfer function_ 从不同的地址取数据
  * 比如，内核可能依次取结构体中的数据，而不是整个结构体
* _transfer function_ 从相同的地址但不同的 offset 取数据
  * 或者对一个 source pointer 使用固定的 offset，比如用循环和 `++` 处理消息

由于每一部分的消息只会被 fetch 一次，因此不属于 double-fetch 问题，被自动排除

最终得到了 `90` 处候选，需要被人工检验

### Double Fetch Categorization

Double-fetch 倾向于发生的三个常见场景

大部分场合下，从用户空间向内核拷贝数据是直截了当的

直接调用 _transfer function_ 就可以

但是当数据带有数据类型或长度时，会变得复杂

这类数据经常会被驱动程序用于在硬件和用户空间之间传递消息

这样的数据通常包含 header 和 body

* header 包含信息的元数据
  * 信息类型的标识
  * 信息 body 的长度

因此内核需要先拷贝 header

* 决定需要的缓冲区类型
* 要为完整的信息分配缓冲区的空间大小

第二次拷贝，将消息拷贝到特定类型的缓冲区中

* 不仅拷贝了 body，还拷贝了完整的消息，包括 header

由此，header 被拷贝了两次

* 如果用户改变了 header 中的 type 或 size，就可能引发威胁

这种情况可以避免

* 只拷贝 body，并将 body 和 第一次拷贝的 header 连接起来

然而，拷贝整个消息更加方便

* 因此这种 bug 在 Linux 中经常出现
* 大部分 Linux 内核已经很老了，是在 double-fetch 被发现之前开发的

#### Type Selection

消息 header 被用于类型选择

* 消息 header 第一次被 fetch，用于识别消息类型
* 第二次 fetch 拷贝完整的消息

在 Linux 的驱动程序中，这种情况很常见：

* 用 `switch` 来处理多种类型的消息
* 第一次 fetch 的结果用于 `switch` 的条件判断
* 第二次 fetch 的结果用于对于特定 type 的消息的处理

总共发现了 `11` 次该类型的 double-fetch，`9` 个在驱动中

由于没有一处使用了第二次 fetch 的 header

因此它们不会导致 double-fetch bug

#### Size Checking

消息的实际长度可变

消息 header 被用于识别完整消息的长度

* 第一次 fetch，获得 header 中的 message size，验证合法性
* 按照 size 分配本地 buffer
* 第二次 fetch，拷贝整条消息到 buffer，包括 header

只要第二次拷贝的 header 中的 size 没有被使用，就不会产生 double-fetch bug

因为用户空间的其它线程可能修改 header 中的 size

总共发现了 `30` 次该类型的 double-fetch，`22` 个在驱动中，`4` 个可能产生威胁（都在驱动中）

#### Shallow Copy

当一个用户空间的 buffer 被拷贝到内存空间时

buffer 中存在一个指针，指向另一个用户空间的 buffer

因此，内核程序必须再调用一次 _transfer function_ 以完成深拷贝

因此会调用多次 _transfer function_，但这种情况不一定是 double-fetch

* 因为两次拷贝的 buffer 不同

总共发现了 `31` 次该类型的 double-fetch，`19` 个在驱动中

### Refined Double-Fetch Bug Detection

第二阶段，采用更为精细的 double-fetch 检测规则

识别 double-fetch bug 和 double-fetch vulnerability

#### Rule 0 - Basic Rule

从相同的来源 fetch 数据两次

#### Rule 1 - No Pointer Change

保证用户指针在两次 fetch 之间没有改变

* 消除了 `++`、加 offset 和向其它变量赋值的 false positives

#### Rule 2 - Pointer Aliasing

用户指针被赋值给了另一个指针，因为指针可能会改变

使用两个指针会更方便

* 一个用于检验数据，一个用于使用数据

如果忽略这种情况，可能导致 false negetives

#### Rule 3 - Explicit Type Conversion

一个消息指针被强制转换为 header 指针，被用于第一次 fetch

之后作为消息指针被用于第二次 fetch

忽略指针的强制转换，会导致 false negetives

此外，指针强制转换通常会和上一条规则一起

造成两个不同类型的指针指向同一个内存区域

#### Rule 4 - Combination of Element Fetch and Pointer Fetch

用户指针可能被同时用于：

* fetch 整个结构体
* 解引用获取结构体中的某个元素

虽然访问的地址不同，但是访问的内存空间有重叠

如果忽略这种情况，也会导致 false negetives

#### Rule 5 - Loop Involvement

Coccinelle 是 path-sensitive 的

当代码中出现循环

循环中的 _transfer function_ 会被报告为调用两次

可能导致 false positives

此外，如果循环中有两次 fetch

前一次循环的后一次 fetch 会与后一次循环的前一次 fetch 被匹配为 double-fetch

这也是 false-positives，因为地址应当在一次迭代结束后已经被修改了

---

## Evaluation

### Statistics and Analysis

对于分析结果得出的数据：

* 大部分 double-fetch 不会导致 double-fetch bug
* double-fetch 更可能出现在驱动程序中

### Analysis of Three Open Source Kernels

#### Linux 4.5

`10` min, `53` candidate files, `5` true double-fetch bugs

#### Android 6.0.1 based on Linux 3.18

包含对 Android 设备的额外驱动，以及附加功能：

* 加强的功耗管理
* 更快的图形支持

`9` min, `48` candidate files, `3` true double-fetch bugs

##### FreeBSD master branch

`2` min, `16` candidate files, no bug

存在 double-fetch，但被额外的检测机制保护

---

## Discussion

### Detected Bugs and Vulnerabilities

目前存在 double-fetch 但没有威胁的代码

在未来的代码更新中可能成为存在威胁的代码

* CVE-2016-5728

### Comparison

其它方法：分析不出来 / 无法覆盖所有代码

### Double-Fetch Bug Prevention

#### Don't Copy the Header Twice

第二次 fetch 只拷贝消息的 body

#### Use the Same Value

只使用某一次 fetch 中的相同数据

大部分 double-fetch 都是良性的

* 因为它们只使用了第一次 fetch 的值

#### Overwrite Data

有些场合中，数据必须被 fetch 两次并被使用两次

* 比如完整的信息要被传入其它函数中进行处理

将第二次 fetch 的 header 用第一次 fetch 的 header 覆盖

攻击者修改了第二次的 header 也没用

这种方法广泛应用于 FreeBSD 中

#### Compare Data

将第一次 fetch 与第二次 fetch 的数据进行比较

如果不一样，则终止操作

#### Synchronize Fetches

使用同步的方法保证原子性

* 保证数据在两次 fetch 之间不会被改变

性能代价

由于 compare data 方法不需要修改很多源代码

大部分漏洞修补采用的是这种方法

* 如果两次 fetch 中重叠的数据不同，内核会返回错误
* Advantages
  * 可以检测恶意用户的攻击
  * 也可以防止非恶意的数据变化（用户空间代码中的 bug 造成）

将 compare data 作为自动修补，注入代码，实现在 Coccinelle 中

### Interpretation of Results

设备驱动是重灾区

### Limitations

代码分析无法检测发生在低层的 double-fetch

* 预处理后或编译后的代码中

也可能发生在编译器优化中

* 发生在编译后的二进制代码中，而不在源代码中
* 指向共享内存的指针没有被标记为 `volatile`
* 编译器在二进制代码层面将单次内存访问拆分为多次

---

## Summary

这篇文章很对我胃口...

让我觉得读起来很顺

感觉这才是真正的计算机 ​😂​

---

