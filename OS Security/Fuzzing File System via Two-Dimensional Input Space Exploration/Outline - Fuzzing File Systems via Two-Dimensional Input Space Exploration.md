# Outline

## Fuzzing File Systems via Two-Dimensional Input Space Exploration - S&P 2019

Created by : Mr Dk.

2019 / 07 / 21 13:40

Nanjing, Jiangsu, China

---

## Introduction

文件系统是最基础的系统服务之一

目前，大部分的文件系统运行在 OS 内核中

因此，文件系统中的 BUG 会导致内核问题：

* 系统重启
* OS 死锁
* 文件系统映像不可恢复的错误

此外，还会暴露很多安全隐患：

* 挂载一个精巧制作的硬盘映像
* 调用有害的文件系统操作，进行任意代码执行或特权提升

手工寻找 BUG 的过程挑战性过大

* _ext4_ - 50K LoC
* _Btrfs_ - 130K LoC

此外，许多广泛使用的文件系统还在被开发者进行不断优化

加入新特性，从而带来了新的 BUG

之前的测试工作，需要对文件系统和 OS 状态有非常深刻的理解

随着 OS 和文件系统的规模越来越大，变得不再实际

另一种方法 - fuzzing，不需要了解目标软件的很多知识

然而，对文件系统进行 fuzzing 依赖于两个输入：

* 一个挂载的磁盘映像
* 一系列的文件操作系统调用

现有的 fuzzer：

* 要么针对于对映像产生变化
* 要么产生文件系统调用的随机集合

对于 fuzzing 文件系统的有效性和包含性较低 - 三大挑战：

1. 磁盘映像是一个很大的结构化二进制对象
   * 它的最小尺寸可能已经比 fuzzer 的最大输出尺寸大 100 倍
   * 因此在对映像产生突变时，大量 I/O 导致了 fuzzing 吞吐量下降
   * 现有的 fuzzer 只突变映像中的非 0 块
     * 没有利用文件系统的布局
     * 对 meta block 进行突变比对 data block 进行突变更有效
     * 在突变后无法修复 metadata 的 checksum
2. 文件操作是一个上下文相关的工作流
   * 映像和在映像上执行的文件操作之间存在依赖
   * 文件系统的实时状态决定了系统调用能够操作哪个文件对象
   * 系统调用会给映像上的文件对象带来变化
   * 现有 fuzzer 通过硬编码的文件路径独立产生随机的系统调用
     * 无法产生有意义的文件操作序列
     * 无法覆盖较深代码层次
3. 对于 bug 的重现
   * 大部分 fuzzer 在测试输入时，没有重新装载新的 OS 实例或 FS 映像
   * 原因是它们使用 VM，需要一定的时间重新状态一个映像
   * 为了解决这个问题，它们重用了实例，导致了 OS 的脏状态
   * 从而导致了不稳定的执行和不可重现的 bug

JANUS - 一个反馈驱动的 fuzzer，有效利用了硬盘文件系统的二维输入

1. 对于第一个问题，JANUS 利用了文件系统的结构化数据
   * 只对 metadata block 进行突变
   * 减少了输入的搜索空间
2. 提出了面向映像的系统调用 fuzzing 技术
   * 不但会存储产生的系统调用
   * 还会在系统调用结束后推测每个文件对象的运行时状态
   * JANUS 使用推测出的状态作为反馈，产生新的一系列系统调用
   * 从而产生了上下文相关的工作流
3. JANUS 还解决了重现的问题
   * 在 _Linux Kernel Library_ 的帮助下，总是装载全新的 OS 状态进行 fuzzing

使用 JANUS 对最新版内核的 8 个流行的文件系统实现进行了长达四个月的 fuzzing

* 比 _Syzkaller_ 有着 `4.19×` 的代码覆盖
* 能够重现 `88-100%` 的 crashes

本文的主要贡献：

1. 现有的文件系统 fuzzer 中存在的三个问题：
   * 对整个映像进行 fuzzing 是低效的
   * Fuzzer 没有利用文件系统映像和文件操作的依赖性
   * Fuzzer 使用了老化的 OS 和文件系统，导致不可重现的 bug
2. JANUS
   * 有效地对映像中的 metadata block 进行突变
   * 利用 LKL 在很短的时间内提供一个干净状态的 OS 映像
3. 代码覆盖、重现、CVE bugs

威胁模型：

假设攻击者有能力挂载一个精心制作的硬盘映像

并操作映像上的文件，从而利用内核中的安全漏洞

攻击者能够不需要 root 特权就能实现攻击：

1. 现代 OS 会自动挂载一个不受信任的插入驱动 (如果文件系统支持)
2. macOS 允许非 root 用户挂载苹果磁盘映像上的 HFS、HFS+、APFS 等文件系统
   * 这些文件系统中存在 bug，会导致绕过内存读取的限制，或内核代码执行
   * Linux 也在用户命名空间中使用 `FS_USERNS_MOUNT` 支持非特权用户挂载任何 FS

---

## Background and Motivation

商用 OS 通常将磁盘文件系统实现为一个内核模块

用户负责对映像进行挂载

并通过文件操作管理数据

### A. A Primer on Fuzzing

Fuzzing 是一种很流行的软件测试方法

重复产生新的输入，并注入到目标程序中，触发 bug

为了有效探索目标程序

最近的 fuzzer 利用过去的代码覆盖来指引将来的输入产生

OS 是最为重要的软件

为了对 OS 进行 fuzzing

通过调用随机产生的系统调用

已有的一些框架使用了反馈驱动的方法触发内核 bug

### B. File System Fuzzing

一个磁盘映像包含二维输入：

1. 结构化的 FS 映像格式
2. 文件操作

几个文件系统 fuzzing 工具要么针对映像，要么针对文件操作

#### Disk Image Fuzzer

磁盘映像是一个大型的结构化二进制块，包含：

* 用户数据
* 管理结构 - metadata

其中，metadata 占据映像总体大小的 1%

另一方面，一个合法映像的最小尺寸也有 100MB 左右

* 在开启一些特性后，可能更大

将映像作为 fuzzing 输入时，可能产生三个问题：

1. 输入尺寸较大，导致了搜索空间的指数增加；同时，重要的 meta 很少被突变
2. Fuzzer 会进行频繁的读写操作，磁盘映像的尺寸拖慢了文件操作，产生了性能开销
3. 为了检测 metadata 的损坏，一些文件系统引入了校验和；内核会拒绝挂载没有正确校验和的 metadata

磁盘映像 fuzzer 需要挂载一个文件系统，并执行一系列的文件操作，触发文件系统中的 bug

早期的 fuzzer 通过随机将字节移动一定的位移来产生新的映像

或者只修改 metadata blocks

这些方法由于需要载入和保存整个映像，引入了大量磁盘 I/O

此外，这些方法没有使用之前执行的代码覆盖，从而产生较差质量的映像

为了克服这个问题，最近的 fuzzer 由代码覆盖驱动

提取映像中所有的非 0 块进行突变

* 这种方法接触到了大部分的 metadata block
* 减少了输入大小，提升了 fuzzing 的性能

虽然如此，这些非 0 块不仅包含了非 0 的 data 块，还丢失了被初始化后的 metadata 块

从而导致了次优的文件系统 fuzzing 效果

此外，由于 metadata block 没有被准确定位，无法修复它们的 checksum

#### File Operation Fuzzer

由于文件系统是 OS 的一部分

通用的 fuzzing 方法 - 调用一系列系统调用

虽说将这些 fuzzer 移植到对应的系统上很直截了当

但是它们通常无法有效 fuzzing 文件系统：

1. 文件操作只会对在映像上存在的文件对象进行修改
   * 一个完成的文件操作只会影响特定的对象
   * 然而，已有的 fuzzer 没有考虑到映像和文件操作的动态依赖
   * 🦐🐔8️⃣ fuzzing 导致肤浅的文件系统搜索
   * 比如 syzkaller - 基于静态的语法规则，描述系统调用每个参数和返回值的数据类型
     * 可以产生语义上正确的系统调用
     * 难以探测几次系统调用的共同行为，和修改后的文件系统映像
     * 比如，可能对某一个旧的路径 `open()` 多次，而该路径已经被重命名 `remove()` 或移除 `unlink()`
2. 已有的 OS fuzzer 主要依赖虚拟机
   * 对于每一个测试输入，为了保证性能，不会重新装载 OS 的状态副本
   * 对老化的文件系统进行 fuzzing 包含两个问题：
     1. 在执行完一些系统调用之后，执行状态变得不确定
        * 比如 `kmalloc()` 依赖于之前的分配，并在运行过程中行为不同
        * 有时一些内核组件会默默出错并从 OS 中分离，而不会触发任何文件系统错误
     2. Fuzzer 发现的 bug 积累了成千上万个系统调用的影响
        * 妨碍了 bug 的溯源和重现
3. 文件系统 fuzzer
   * 已有的 fuzzer，要么 fuzz 二进制输入，要么 fuzz 系统调用
   * 然而，为了 fuzz FS，需要突变两种输入：
     * 二进制映像
     * 对应的工作流 (FS 系统调用)
   * 两者结合

### C. Challenges of Fuzzing a File System

#### Handling Large Disk Images as Input

一个映像 fuzzer 应当能够有效 fuzz 一个复杂的、庞大的磁盘映像：

1. 对分散的 metadata 进行突变，并计算校验和
2. 在突变输入时，减少频繁的磁盘 I/O

理想化的 fuzzer 应当只针对 metadata，并在突变后修复校验和

#### Missing Context-aware Workloads

文件系统的工作流直接影响着映像

在运行时，只有合法的文件操作才会修改映像上的文件对象

然而，已有的 fuzzer 依赖于预定义的映像信息来产生系统调用

* 合法的文件或目录路径

无法完整地检测目标文件系统上所有可访问的文件对象

因此，在进行文件操作后，在产生新的文件操作之前

最好能够维护每个文件对象的运行时状态

#### Exploring Input Space in Two Dimensions

* 磁盘映像
* 文件操作

两种输入的格式完全不同

但存在一定的隐含联系

应当同时对它们进行突变

采用混合的方法，同时 fuzzing 两种输入

#### Reproducing Found Crashes

传统的 fuzzer 为了避免重启 VM 或从 snapshot 恢复

在多次输入的过程中重用了 OS 或文件系统实例

导致了不稳定的内核执行和不可重现的 bug

这个问题可以通过利用 library OS 来解决

* 它能够提供内核的行为
* 并能够在几毫秒内重新初始化 OS 状态

---

## Design

### A. Overview

1. 保存从 seed 映像中提取出的 metadata，作为突变目标
   * 在突变后，重新计算每一个 metadata 的 checksum
   * 由于 metadata 只占映像大小的 1%
   * 因此显著提高了 fuzzing 吞吐量
2. 不依赖于人为指定的映像上的文件信息，因为这个信息会随 fuzzing 而发生变化
   * JANUS 基于完成上一次工作后的映像推断状态来产生新的文件操作
3. 通过明智地调度 image fuzzing 和 file operation fuzzing 来探索二维的搜索空间
   * 由于原始映像决定了文件系统的初始状态，JANUS 总是先试图突变映像
4. 依赖于 library OS 实例运行与用户空间中
   * 以微不足道的开销重新加载文件系统
   * 从而提升重现 bug 的概率

JANUS 的二进制输入包含三部分：

1. 包含 seed 映像 metadata 的二进制对象
2. 序列化的程序
3. 程序对映像进行操作后推测的映像状态

首先，JANUS 依赖于一个特定文件系统的转换器，从 seed 映像中提取 metadata

同时，JANUS 也会检查 seed 映像以获得映像初始状态，并产生起始程序

Metadata + 程序 + 映像状态 被打包存入语料库中

JANUS 在一个死循环中，不停从语料库中选择一个测试用例，并开始 fuzzing

* 首先，fuzzing 引擎调用 image mutator 修改 metadata 中的字节
* 与此同时，程序不变
* Syscall fuzzer 要么程序中已有的参数值，要么在程序中加入新的系统调用，产生新的程序
* Syscall fuzzer 根据新的程序，推测出操作后的映像状态
* 此时，metadata 不变
* Metadata 与其它不变的部分共同组成新的 full-size 映像，并重新计算所有的 checksum
* 新的程序被序列化，并保存到磁盘上
* 用户空间系统 - executor - 产生一个新的实例，挂载 full-size 的映像，并执行保存到磁盘上的新程序中的文件操作
* 运行时的代码覆盖在一个与 JANUS 共享的 bitmap 中被描绘
* Fuzzing 引擎检测 bitmap，如果覆盖到了新的路径，JANUS 就将该 image、程序、预测状态保存到语料库中，用于未来的突变

对于每一次此时，JANUS 总是先执行 image mutator

如果没有有效的用例发现，再执行 syscall fuzzer

### B. Building Corpus

JANUS 依赖于 image parser 和 syscall fuzzer

基于一个 seed 映像，产生初始的语料库

语料库中的第一部分是 seed 映像的 metadata block

* JANUS 首先将整个映像映射到了一个内存 buffer 中
* 一个特定文件系统的 image parser 扫描映像，定位所有的 metadata
* JANUS 将所有的 metadata 组装为一个二进制对象，用于之后的 mutation；同时记录它们的大小，和在映像中的位移
* 对于由 checksum 保护的 metadata，JANUS 记录 checksum 在映像中的位移

语料库中的第二部分是映像上每个文件和路径的信息

* JANUS 使用这些信息生成上下文相关的对应程序
* 需要探测 seed 映像上每个文件的路径、类型、扩展属性

语料库中的第三部分是起始程序

* 包含了 syscall fuzzer 产生程序，用于突变
* 为了扩大总体的覆盖率，每个随机产生的系统调用操作一个唯一的文件对象

Metadata + 文件状态 + 起始程序 共同构成了一个输入用例，并被保存到语料库中

### C. Fuzzing Images

JANUS 使用 image mutator 对映像进行 fuzzing

* Image mutator 载入输入用例中的 metadata blocks，并应用几个常见的 fuzzing 策略
  * bit flipping
  * arithmetic operator 
  * random bytes
  * ...
* 产生了新的 metadata
* 除了随机数以外，还比较关注特殊值，比如 `0`，`-1`，`INT_MAX` 等
  * 可以产生更 corner 的 case，引发更多的 crashes
* 在修改 metadata 之后，JANUS 将这些块拷贝回 memory buffer 的对应位置，组成新的映像
* 为了保证映像合法性，JANUS 还需要重新计算每个 metadata 的校验和，并填充到映像对应位置

### D. Fuzzing File Operations

JANUS 能够产生面向映像的程序

从而有效探索文件系统如何处理各式各样的文件操作

一个程序包含：

* 一系列有序的系统调用，操作突变后的映像
* 维护一个变量库，记录系统调用使用到的变量
* 一系列被打开而未被关闭的文件描述符

JANUS 将每个系统调用描述为一个元组，包含：

* 系统调用编号
* 参数值
* 一个返回值

如果参数值或返回值不是一个常量，而是一个变量

JANUS 用该变量在变量库中的 index 表示

与 Syzkaller 类似，syscall fuzzer 有两种方式产生新程序：

1. Syscall mutation
   * 随机选择程序中的一个系统调用
   * 产生一系列新值，代替旧值
2. Syscall generation
   * 在原程序后追加新的系统调用，并随机产生其参数

#### Syscall Mutation

与 Syzkaller 的策略类似，对于一些不重要的参数，会与运行时状态独立

对于给定可用范围定义的参数，从可用集合中随机选择

* `int whence()` for `lseek()`

此外，对于整数类型，随机产生特定范围中的数

* `size_t count` for `write()`

一些文件操作需要一个指针类型的参数

* 通常被用于指向用户数据或内核输出 - `void *buf` for `write()` & `read()`
  * 对于前者，syscall fuzzer 声明一个填满随机数据的数组
  * 对于后者，使用一个固定的数组
    * 因为 JANUS 不由内核输出的实际值驱动

对于那些不重要的参数，但是其值依赖于文件系统运行时的上下文

JANUS 不仅基于它们的数据类型，还基于 JANUS 维护的文件系统状态

1. 如果需要文件描述符，syscall fuzzer 随机选择一个特定类型的打开的文件描述符
   * `write()` 需要文件描述符，`getdents()` 需要目录描述符
2. 如果需要一个路径
   * syscall fuzzer 随机挑选一个已有的文件或目录
   * 或挑选一个被最近几次操作删掉的过时文件
   * `rename()` / `rmdir()`
   * 如果路径被用于产生新的文件或目录，JANUS 会随机选择一个已经存在的目录随机产生新的路径
3. 如果系统调用操作一个特定文件的已有扩展名
   * `getxattr()` / `setxattr()`
   * Syscall fuzzer 会随机挑选一个被记录的文件的扩展名

这些规则是 JANUS 能够对 fresh 的文件对象产生上下文相关的程序

* 没有运行时错误
* 较高的 code coverage

#### Syscall Generation

对于一个新产生的系统调用

JANUS 要将其追加在程序后面

还需要总结该系统调用会对文件系统映像带来的映像

并更新对应的预测映像状态

JANUS 只维护程序执行完成后的预测映像状态

### E. Exploring Two-Dimensional Input Space

JANUS 按顺序调度两个核心 fuzzer

对于一个输入：

* JANUS 首先启动 image mutator 修改映像的 metadata
* 使用原程序进行 fuzzing
* 如果没有新的 code path，JANUS 启动 syscall fuzzer 修改原程序中的参数值，进行一定轮数
* 如果依旧没有新的 code path，JANUS 最终试着在程序最后添加系统调用

每一个 fuzzing 阶段的轮数都是用户定义的

以这样的顺序调度较为高效：

1. 提取的 metadata 代表了映像的初始状态
   * 当映像被几个系统调用操作后，其对于系统调用执行的影响越来越小
2. 引入新的文件操作指数增加了 mutation 空间
   * 因此 JANUS 更倾向于突变已有的系统调用，而不是增加新的

### F. Library OS based Kernel Execution Environment

为了避免不稳定执行和不可重现的 bug

JANUS 依赖于基于 library OS 的应用 - executor 来进行 fuzzing

JANUS 创建新的实例来测试每个新产生的映像和程序

与重启 VM 相比，fork 一个用户应用的时间小到可以忽略

从而，JANUS 用很低的开销为每一次测试用例保证了一个状态干净的 OS

由于 fuzzing 引擎和 executor 运行于同一台机器的用户空间中

* 共享输入文件和 coverage bitmap 是直截了当的，比 VM 方便很多

此外，Library OS 实例比任意类型的 VM 占用更少的计算资源

---

## Implementation

将 JANUS 实现为 AFL 的变种

利用了 AFL 的基本结构，包括：

* forkserver
* coverage bitmap
* test-case scheduling algorithm

在 AFL 上扩展了 image mutator 和 syscall fuzzer

还实现了 image inspector 建立 seed 映像的初始语料库

Program serializer - 在内存和语料库之间传递产生的程序

基于 Linux Kernel Library (LKL) 实现了 executor

修改 LKL，使其支持 kernel address sanitizer (KASAN)

为了便于重现 bug，实现了一套 Proof-of-Concept (PoC) generator

* 从测试样例中产生一个 full-size 映像和一个可编译的 C 程序

### Image Parser and Image Mutator

将 Image Parser 实现为动态库

* 定位 metadata，识别 checksum
* 支持 Linux 上 8 种广泛使用的文件系统
* 引用了用户空间工具 - `mkfs`、`fsck`

Image Mutator 被用于随机突变映像的 metadata 部分

* 八种策略 - 直接从 AFL 中移植

### Image Inspector

对 seed 映像上的文件和路径进行迭代

并记录它们的映像内路径、类型、扩展名

用于产生初始的测试用例

### Program Serializer

将新产生的程序和更新后的状态

* 从内存中保存到磁盘上
* 从磁盘上读取出来用于 fuzzing

### System Call Fuzzer

Syscall fuzzer 被实现为 AFL 的新的扩展

在 JANUS 的 image mutation 没有进展时被调用

Syscall fuzzer 接收一个程序和对应的映像状态

通过 syscall mutation 和 syscall generation

并输出新的程序和更新后的状态

有一些与文件操作相关但主要实现在 VFS 中的系统调用被忽略

* `dup()`、`splice()`、`tee()`、...

在本文的实现中，JANUS 突变 metadata 256 轮

如果 code average 没有增加

JANUS 试着突变已有的系统调用 128 轮

并加入新的系统调用再循环 64 轮

### LKL-based Executor

在 library OS - Linux Kernel Library 上实现了 executor

* LKL 向用户空间程序暴露了内核接口

实现了用户空间应用，与 LKL 链接，作为 fuzzing 目标

对于一个测试用例，executor 通过 forkserver fork 新的实例

并调用 LKL 系统调用，挂载突变后的映像

然后开始执行生成的程序中的文件操作

由于每次将 full-size 的映像写入磁盘需要很长时间

本文实现中，引入了一块永久的内存缓冲区

由 JANUS 的 fuzzing 引擎和 LKL executor 共享

用于存放 image

LKL 的块设备驱动需要被修改

访问内存中的 buffer，而不是磁盘上的映像文件

此外，在运行时还使用了 Copy-on-Write (CoW) 技术

保证除了突变过的块，映像 buffer 上的其它部分不会被产生的程序修改

当设备驱动在运行时想要修改映像上的 block 时

Block 被复制用于修改，并在之后被 LKL 访问

在 LKL 中还集成了 KASAN，用于检测运行时的内存错误

KASAN 会在运行时分配 shadow memory

记录原内存的每一个字节是否可以被安全地访问

* 由于 KASAN 依赖于 MMU，而 LKL 不支持
* 在 LKL 启动时为 LKL 的内存空间和 shadow memory 建立映射

---

## Evaluation

* Q1: JANUS 发现之前未知的文件系统漏洞有多有效？
* Q2: JANUS 搜索文件系统映像、文件操作和两者结合的效率如何？
* Q3: 基于 library OS 的 executor 是否比 VM 重现 bug 的效率更高？
* Q4: 除了发现新 bug，JANUS 还可以为文件系统社区贡献什么？

### Experimental Setup

为每一种文件系统创建一个 seed 映像

* 上面包含各类文件对象 (文件、目录、链接等)

其中，对于 ext4，创建了两个 seed 映像：

* 一个与 ext2/3 兼容
* 一个带有 ext4 的特性

类似地，对于 XFS v4 和 XFS v5，其中后者引入了 checksum，保证了 metadata 的完整性

由于 Syzkaller 依赖于 KCOV 来获得代码覆盖，而 JANUS 依赖 AFL

为了公平比较，在 fuzzing 12h 后，依次挂载 JANUS 突变的每一个映像

并在一个开启了 KCOV 的内核上运行每一个程序，获得 KCOV 风格的代码覆盖

#### A. Bug Discovery in the Upstream File System

大部分的 bug 的重现需要：

* 挂载一个损坏的映像
* 紧接着进行特定的文件操作

是 JANUS 的二维输入共同触发的

有四分之一的 bug 是在挂载一个损坏的映像后触发的

体现了 JANUS 对 image 进行 fuzzing 的有效性

总体上，JANUS 成功发现了 90 个 bug，其中 62 个是之前未知的

#### B. Exploring the State Space of Images

Syzkaller 已经支持挂载一个修改后的 image - `syz_mount_image()`

* 接收一个映像突变后的非 0 段作为输入
* 并这些块放入对应的映像偏移中
* 最终调用 `mount()` 挂载

在一个突变后的映像挂载之后

* LKL-based executor
* Syzkaller

执行一个固定序列的系统调用

对比哪个 fuzzer 对 image 的突变效果更好

JANUS 总是有更高的代码覆盖率 - `1.47-4.17×` Syzkaller

* Syzkaller 只考虑了映像中的非 0 部分
  * 即可能丢失 metadata 块
  * 也可能包含无关的 data 块
* JANUS 利用了映像的语义，只定位和修改 metadata blocks

此外，对于 GFS2 和 Btrfs，具有针对 metadata 的校验和，严重影响了 Syzkaller 的表现

Syzkaller 也不支持具有超过 4096 个不连续的非 0 块的 image

总体上看，对于只突变映像，JANUS 具有更高的效果

#### C. Exploring File Operations

比较 JANUS 和 Syzkaller 对于 fuzzing 27 个文件系统相关的系统调用的效果

只 fuzzing 文件操作，不修改文件系统映像

对于 Syzkaller 的程序生成，在系统调用描述文件中对文件路径进行硬编码

对于 XFS v5、Btrfs、ext4 文件系统

JANUS 的代码覆盖分别是 Syzkaller 的 `2.24×`、`1.27×`、`1.25×`

通过产生上下文相关的程序，JANUS 能够比 Syzkaller 更有效地 fuzzing

Syzkaller 无法获得文件系统的具体信息

#### D. Exploring Two-Dimensional Input Space

比较突变映像和突变程序结合时的效果

在 Syzkaller 的描述文件中引入 `syz_mount_image()`

使其不仅产生系统调用，还会对映像进行突变

JANUS 在两者结合的 fuzzing 条件下，比单独 fuzzing 映像或系统调用效果都要好

Syzkaller 对于两者的调度，优先先对 syscall 进行突变

* 比如，在进行文件操作之前，Syzkaller 不会保证挂载的文件系统是否合法
* 完全丢失了文件系统的上下文信息

而 JANUS 会分别对每个映像使用状态干净的 LKL 实例进行测试

如果挂载一个突变后的映像成功

那么 executor 将会执行上下文相关的程序

并在结束时调用 `umount()`

#### E. Reproducing Crashes

使用 PoC generator 产生了映像和对应的系统调用序列

将映像挂载，并执行对应的系统调用，查看 kernel 是否崩溃

Syzkaller 会将 crash 记录在日志中

但由于使用了老化的 OS，无法重现任何的 bug

相反的是，JANUS 能够重现 95% 的 bug，除了 Btrfs 文件系统

* 因此 Btrfs 启动多个内核线程并行完成不同的事务，导致了不确定的执行顺序
* F2FS 和 GFS2 也使用多线程来完成特定的工作，比如 GC，日志等
* 如果能够控制线程调度，理论上可以重现 100% 的 bug

本文还对比了基于 VM 的 fuzzer 刷新内核状态的开销

* KVM 实例重启 VM 或从 snapshot 恢复的总时间

并与 LKL executor 进行对比

由于 LKL 实例调用 `fork()` 并启动一个新的 LKL 实例

基于 LKL 的实例几乎占用了可以忽略的时间，重新设置了一个干净的 OS 和 FS 状态

#### F. Miscellany

部分文件系统社区使用 JANUS 产生的一些损坏的映像作为测试用例

用于未来的回归测试

F2FS 开发者不仅修复了内核模块中的漏洞

还增强了对应的用户空间工具 - `fsck.f2fs`

* 使用户能够在挂载前，提前检测到映像损坏

---

## Discussion

### Library OS based Executor

JANUS 依赖于 LKL 测试内核文件系统

事实上，其它内核子模块也可以使用 LKL 进行测试

* 除了需要 MMU 的组件

此外，Oracle 的内核 fuzzer - User-Mode Linux (UML) 也有类似功能

但由于其多进程的设计，增加了发现 crash 的复杂性

### Minimal PoC Generator

一个理想化的 PoC 应当只包含错误发生处的字节

和一个有着最少文件描述符的程序

JANUS 目前使用了暴力的方式：

* 恢复每一个突变后的字节
* 移除每一个引用的文件描述符
* 检测内核是否依旧在预想位置 crash

### Fuzzing FUSE Drivers

目前，JANUS 不支持依赖于 FUSE (Filesystem in User space) 的文件系统

* 只要它们将用户数据存储在磁盘映像中
* 支持对应的文件操作，使用户能够访问

JANUS 就可以支持这些文件系统

### Fuzzing File System Utilities

开发者严重依赖于系统工具 - `mkfs` / `fsck` 等，管理文件系统

比如，Linux 自动启动 `fsck` 来恢复磁盘数据

因此，开发者希望，这些工具中是没有 bug 的

开发者能够轻易扩展 JANUS 的 image mutator

产生损坏的映像，用于 fuzzing 这些工具

本文作者使用 JANUS 已经找到了 `fsck.ext4` 中的两个未知的 bug

### Extending to Fuzz File Systems on Other OSes

如果对应系统的 library OS 解决方案存在

那么将 JANUS 扩展到这样的系统上是直截了当的

### Improving Other File System Testing Tools

其它工具需要文件操作序列来进行其它方面的检测

JANUS 可以为其提供一站式的解决方案

---

## Related Work

### Structured Input Fuzzing

对于 fuzzing 的输入 - 高度结构化 - 比如文件系统映像

一些 fuzzer 基于人工描述的输入规格，产生语法上正确的输入

一些基于突变的 fuzzer 通过修改一些合法样本来产生新的输入

* 新的输入拥有正确的结构，带有一些小的错误，并希望能够触发 bug

### OS Kernel Fuzzers

这类 fuzzer 基于预定义的语法规则，产生随机的系统调用

在对于文件系统的 fuzzing 中很低效

而 JANUS 能够产生质量较高的程序

### File System Semantic Correctness Checkers

一些 checker 用于分析一个文件系统的实现是否遵循标准

* 通过静态分析或文件系统行为的高层建模

### File System Abstraction

一些研究提出，通过高层抽象的统一接口访问或修改磁盘上的 metadata

通过这些接口，JANUS 能够用一种增加通用的方式压缩磁盘映像

而不是为每一个目标文件系统实现 image parser

---

## Conclusion

在二维的输入空间中对文件系统进行 fuzzing

* 有效突变了输入映像中的 metadata 部分
* 执行上下文相关的系统调用程序
* 依赖于 library OS 快速装载 OS 功能状态，避免了不稳定执行和不可重现的 bug

总共报告了最新内核中的 90 个 bug

* 43 个已经被修复；其中 32 个被分配了 CVE

JANUS 的表现超过了 Syzkaller：

* 最大 `4.19×` 倍的代码覆盖率
* 能够重现 `88-100%` 的 crashes

被几个文件系统开发团队用于一站式的文件系统测试方案

也可以作为一些新的文件系统 checker 的基础设施

---

## Summary

第一次看文件系统相关 对于文件系统的内部实现

尤其是论文中 metadata 这一块儿还不太懂

据我了解，文件系统内部的 inode 是一级一级找下去的

如果把这些 metadata 打乱了，这些文件还能被找到吗。。。

如果找不到，接下来的文件操作如何进行？

此外，多亏上周了解了一下 Syzkaller

相关的描述看得明白多了

目前就是不知道关于 AFL 的一些细节

有空可以再了解了解

---

