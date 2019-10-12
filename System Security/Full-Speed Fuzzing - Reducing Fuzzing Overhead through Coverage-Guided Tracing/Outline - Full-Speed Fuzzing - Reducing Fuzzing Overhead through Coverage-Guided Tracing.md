# Outline

## Full-Speed Fuzzing: Reducing Fuzzing Overhead through Coverage-Guided Tracing - IEEE S&P 2019

Created by : Mr Dk.

2019 / 10 / 12 15:04

Nanjing, Jiangsu, China

---

## 1. Introduction

从高层来看，fuzzing 分为以下几步：

1. 生成测试用例
2. 监控用例对目标可执行文件的影响
3. 处理那些暴露 bug 或产生 crash 的测试用例

最先进的 fuzzer 主要致力于 coverage-guided fuzzing

* 追踪执行的控制流，来追踪 coverage
* 使 fuzzer 只需要对一小部分输入 (能够进入之前没有进入的代码区的输入) 进行变异

Code coverage 是一种抽象的概念，包含三种具体的形式：

* Basic blocks
* Basic block edges
* Basic block paths

对于白盒 (源代码可获得) 的可执行文件

* Code coverage 是通过编译时插入 instrumentation 完成的

> Instrumentation 这个词怎么翻译至今搞不懂...

对于黑盒的可执行文件

* 通过动态或静态的 binary rewriting (运行时?)

追踪 code coverage __代价很大__

* 占据了大部分 fuzzer 运行时最多的时间
* 大部分测试用例没有增加 coverage，因此 coverage 信息通常会被丢弃

根据实验，AFL 的黑盒测试会有 13 倍的额外开销，白盒测试会有 1.7 倍的额外开销

Fuzzing 过程中 90% 的时间都用于 __执行__ 和 __追踪__ 测试用例

* 绝大部分的测试用例及其 coverage 被丢弃
* 根据评估，10000 个测试用例中，只有不到 1 个测试用例能够提升 coverage
* 因此，盲目对每个测试用例的 coverage 进行追踪很浪费

本文提出的思想：coverage-guided tracing

* 只追踪那些会引起新的 coverage 的测试用例

实现方式：

* 对目标可执行文件进行转换
* 当测试用例引起新的 coverage 时，可执行文件能够 __自我汇报__
* 将这种变换后的可执行文件称为 _interest oracles_

Interest oracles 以正常的速度执行程序

* 因为它不需要对 coverage 进行追踪

当 interest oracles 报告一个测试用例引发了新的 coverage 时

* 测试用例被标记为 coverage-increasing

* 使用传统的 coverage 追踪，采集 coverage 信息

* 恢复 interest oracles 中被修改的部分，继续进行

  > Interest oracles 中的这种设计还真是妙啊
  >
  > 你还真他娘是个人才 🤙

这样，coverage-guided tracing 处理 coverage-increasing 用例的代价较高

却可以以原始速度运行所有的测试用例

本文实现了 UnTracer，与 AFL 的不同版本进行了比较

* 黑盒测试
  * Dynamic binary rewriter - AFL-QEMU
  * Static binary rewriter - AFL-Dyninst
* 白盒测试
  * AFL-Clang

Fuzzing 的目标是现实世界中的 8 个程序

结果表明，UnTracer 在所有的测试程序中都表现最好

* 平均运行时开销为 0.3% - 相比于 612%、518%、36%

随着时间流逝，coverage-increasing 的测试用例的比例快速趋向于 0

* 使 UnTracer 的性能提升了四个量级

本文的主要贡献：

* 提出 Coverage-guided tracing - 将 tracing 约束到引起 coverage 增多的测试用例
* 量化了 coverage-increasing 的测试用例的不频繁性
* 评估了不同类型的 coverage-guided fuzzer 在 tracing 上浪费的时间
* 实现了 UnTracer
* 将 UnTracer 集成到最先进的混合 fuzzer QSYM 中
* 开源了本文中的实现

---

## 2. Background

### A. An Overview of Fuzzing

Fuzzers 通常根据 __产生测试用例__ 和 __监控用例执行__ 的方式被分类：

* Grammar-based
  * 产生的测试用例由预定义的输入语法约束
* Mutational
  * 测试用例由对其它测试用例进行变异而产生
  * 在第一轮迭代中，对 seed 用例进行变异
  * 在随后的迭代中，对前一轮的用例进行变异

对于大型应用来说，输入语法十分复杂

对于闭源程序来说，输入语法可能不可获得

因此，大部分流行的 fuzzer 都是 mutational 的

Mutational fuzzer 利用程序分析来决定变异哪一个测试用例

* Directed fuzzers - 目标是到达指定的程序区域
  * 变异目标是那些看起来距离指定区域更近的测试用例
* Coverage-guided fuzzers - 目标是探索整个可执行文件的所有代码
  * 变异目标是那些能够到达新的代码区的测试用例

后者更为流行

基于 fuzzer 进行程序分析的程度，还可以分为：

* Black-box fuzzers - 只监视输入、输出和执行行为
* White-box fuzzers - 使用重量级的程序分析，进行细粒度的执行路径监控
* Grey-box fuzzers - 折衷

本文的实现基于 grey-box fuzzer AFL

### B. Coverage-guided Fuzzing

最大化测试用例的 coverage

1. 队列中的初始种子
2. 测试用例产生 - 选择一个种子，并变异多次
3. 监控执行行为 - 追踪 coverage
4. 如果测试用例引发了 coverage 增多，加入队列；否则丢弃
5. Crash triage
6. 回到第 2 步

追踪 coverage 的方法：

* binary instrumentation
* system emulation
* 硬件支持

所有的 coverage-guided fuzzer 都基于三种 coverage 度量机制：

* basic blocks
  * CFG 中的结点
* basic block edges
  * `<src, dst>` 元组
* basic block paths
  * 需要硬件支持，暂时没有相关工作

### C. Coverage Tracing Performance

白盒 fuzzer

* 在编译 / 汇编阶段，将 instrumentation 插入代码
* 由定制的 GCC 或 Clang 实现

黑盒 fuzzer 由于没有源代码

* 需要花费很大代价重建 control-flow
* 引入的开销与正常执行速度相比，多了 1000%

### D. Focus of This Paper

Coverage-guided fuzzing 的一个特性就是追踪所有的测试用例的 coverage

通过 "smarter" 的用例生成策略，能够提升 coverage-increasing 的用例比例

* 而 coverage-increasing 的测试用例依旧很少
* 对 non-coverage-increasing 的用例进行 tracing
* 意味着有提升 fuzzing 性能的机会

本文提出了 coverage-guided tracing

* 只对 coverage-increasing 的测试用例进行 coverage 追踪

---

## 3. Impact of Discarded Test Cases

传统的 coverage-guided fuzzer AFL

* 基于一种盲目 (随机变异) 的测试用例生成策略
* 不引起 coverage 增加的测试用例会被丢弃

为了减少这种测试用例的比例

* 一些 fuzzer 使用了聪明一些的测试用例生成策略
* 利用源代码分析，从而能够产生更高比例的 coverage-increasing 测试用例

然而，how effective?

在这一章节，研究了 non-coverage-increasing 的测试用例对 fuzzer 执行 / 追踪的影响

* AFL - blind test case generation
* Driller - smart test case generation

AFL 追踪测试用例 coverage 的方式：

* 黑盒 - QEMU-based dynamic instrumentation
* 白盒 - 编译 / 汇编时的 instrumentation

Driller 追踪测试用例 coverage 的方式：

* 选择性的符号执行 - 解决部分路径约束
* 产生更少的 non-coverage-increasing 用例
* 符号执行能够使 fuzzer 突破 AFL 可能会被卡死的地方 (magic value)

### A. Experimental Setup

为了计算每个 fuzzer 的执行 / 追踪时间

在 AFL 的测试用例执行函数 `run_target()` 中插入计时代码

由于计时在每个测试用例被执行时都会进行

* 顺便可以用于计算产生的测试用例的总数

计算每个 fuzzer 的 coverage-increasing 测试用例

* 在 AFL 的队列目录中，寻找所有保存的用例中，附加了 `+cov` 的样例

### B. Results

AFL 和 Driller 将主要时间 (90%) 都花费在了测试用例执行 / coverage 追踪上

而其中引发 coverage 增多的用例占有的比例：

* AFL-Clang: 0.0062%
* AFL-QEMU: 0.0257%
* Driller-AFL: 0.00653%

实验表明，fuzzer 把主要的时间，都花在执行 / 追踪 non-coverage-increasing 的测试用例上

---

## 4. Coverage-guided Tracing

目前的 coverage fuzzer 追踪所有的测试用例

* 将用例自身的 coverage 与某个全局累加的 coverage 进行比较
* 引发 coverage 增加的样例被留下用于变异
* 不引发 coverage 增加的样例及其 coverage 信息一起被丢弃

Coverage-guided tracing 的目标是：

* 只对引起 coverage 增加的测试用例进行追踪

### A. Overview

在 __test case generation__ 和 __code coverage tracing__ 之间引入一个中间步骤 - __interest oracle__

* 这是一个对于目标二进制文件的修改版本
* 一个预选择的 __软件中断__ 被覆盖到每个 __未被覆盖__ 的 basic block 的开头处
* 触发软件中断的测试用例被标记为 coverage-increasing，并被追踪 coverage
* 新被覆盖的 basic block 被记录后，对应 basic block 中的软件中断被移除 (unmodifying)
* 随着 fuzzing 的继续进行，只有覆盖新的 basic block 的测试用例才会触发软件中断

Coverage-guided tracing 对 fuzzing 的增强体现在：

1. Determine Interesting: 将测试用例运行于 interest oracle 上，若触发软件中断，则标记为 coverage-increasing
2. Full Tracing: 对于每个 coverage-increasing 的测试用例，追踪其完整的 code coverage
3. Unmodify Oracle: 对于 coverage 中第一次被访问的 basic block，移除其中的软件中断

### B. The Interest Oracle

过滤不引起 coverage 增加的测试用例，防止它们被追踪 coverage

给定一个目标二进制文件

Interest oracle 是一个修改版本，每个 basic block 的头部被覆盖了一个软件中断

如果一个测试用例触发了中断，意味着新的 coverage 被发现

这个用例之后被追踪，获得其 full coverage

其中，新被发现的 basic block 中的软件中断被移除

Interest oracle 的构造需要目标二进制文件的每个 basic block 的地址

* 有很多种方法可以 handle (静态分析等)

在每个 basic block 上插入中断很简单

但是存在两个重点：

1. 任何中断信号都可以被使用，但需要避免和 fuzzing 过程中需要用到的信号冲突
   * 比如和 crash 和 bug 等有关的信号
2. 中断指令的 size 不能超过任何 basic block 的 size
   * 一个 1B 的基本块，肯定装不下 2B 的中断指令

### C. Tracing

对于 coverage-increasing 的测试用例

用一个 tracing-only 版本的二进制文件 (也就是 AFL 本来用于 fuzzing 的版本) 来追踪这个测试用例的 full coverage

### D. Unmodifying

将新发现 coverage 的 basic block 中的中断指令，用未被修改的二进制文件中的指令替代 (复原)

### E. Theoretical Performance Impact

随着 fuzzing 的进行

越来越多触发 coverage 增多的测试用例导致更多的 basic block 被复原

* Interest oracle 和原目标二进制文件的差异将越来越小
* 一个测试用例触发 coverage-increasing 的概率也越来越小

而对于不触发 coverage-increasing 的测试用例来说

* 在目标二进制上执行，和在 oracle 上执行，速度是相同的

随着 fuzzing 的进行，Coverage-guided tracing 的总体开销会趋近于 0

---

## 5. Implementation: UnTracer

### A. UnTracer Overview

UnTracer 基于 AFL 实现

首先，AFL 完成其初始化

UnTracer 对两个版本的目标二进制文件进行 instrumentation

- Interest oracle - 用于识别引发新的 coverage 的测试用例
  - 使用静态分析，识别所有的 basic block
  - 插入中断指令
- Tracer - 用于识别新的 coverage

两个二进制文件都会得到 forkserver

在 fuzzer 的主循环中

* UnTracer 在 oracle 上执行每一个 AFL 产生的测试用例
* 如果测试用例触发中断，那么将其标记为 coverage-increasing
* 使用 tracer 采集其 coverage
* 停止 forkserver，unmodify 每一个新覆盖的 basic block
* 重启 forkserver

AFL 完成其 coverage-increasing 的测试用例的处理逻辑

### B. Forkserver Instrumentation

在 fuzzing 过程中，oracle 和 tracer 都会被执行很多次

* Oracle 会执行所有的测试用例
* Tracer 会执行所有的 coverage-increasing 的测试用例

因此在实现中，作者决定对这两个二进制文件的执行进行优化

* 利用 AFL 的 forkserver 模型

通过利用 `fork()`，forkserver 避免了重复的进程初始化工作

* 比传统的基于 `execve()` 的执行好很多 - ??? 难道 `execve()` 不需要 `fork()` ?

首先，在二进制文件的 `.text` 区域中插入 forkserver 的函数

然后通过一个回调，链接到 `main()` 的第一个 basic block

### C. Interest Oracle Binary

目标二进制文件的修改版本

通过在没有被覆盖的 basic block 中插入软件中断

引起 coverage 增加的测试用例能够进行 __自我汇报__

为了插入软件中断，首先需要知道每个 basic block 的地址

本文使用了 Dyninst 的静态控制流分析，输出一个 basic block 的列表

并迭代插入软件中断

使用了 `SIGTRAP` 信号作为软件中断：

1. 被广泛用于细粒度的执行控制和分析
2. 其二进制表示为 `0xCC`，只有 1B，可以覆盖任何 size 的 basic block

### D. Tracer Binary

用于执行上一步中选中的引发 coverage 增加的测试用例

* 将 coverage 回调插入到每个 basic block 中
* 每次执行时，basic block 中的 callback 会将 basic block 的地址追加到 trace 文件中

问题：存在反复被执行的 basic block 被重复记录 (比如循环)

* 拖慢了 trace 文件的写入操作
* 读取 trace 文件用于移除中断时，也存在重复操作

优化：只记录被覆盖的 basic block 一次，防止多次读写相同的 basic block 信息

* 在 forkserver 中初始化一个全局的 hashmap
* 记录所有被覆盖的 basic block (unique)
* 当 fork 时，该 hashmap 也被继承
* 在每个 callback 中，通过查找这个 hashmap 来判断该 basic block 是否已被 cover

### E. Unmodifying the Oracle

对于新 cover 的 basic block

用原来的目标二进制文件中的对应字节替换软件中断指令

之后测试用例运行到这个 basic block 将不再会触发软件中断

从而不再会被识别为 coverage 增加

作者观察到一个现象：

* 引起 coverage 增加的测试用例与之前的测试用例有较多重叠
* 导致 UnTracer 试图恢复那些已经被恢复的 basic block

对策：一个用于追踪全局 coverage 的 hashmap

* 在复原每个 basic block 之前，查找 hashmap
* 如果该 basic block 已经在之前被复原，则跳过
* 否则移除 basic block 中的软件中断

保证了只有新 cover 的 basic block 被处理

---

## 6. Tracing-only Evaluation

### A. Evaluation Overview

* UnTracer
* AFL-Clang
* AFL-QEMU
* AFL-Dyninst

计算 8 个现实世界中的程序在各个 fuzzer 上的开销

移除 AFL 中与 tracing 无关的功能

对 `run_target()` 函数进行 instrument，采集每个测试用例所花的时间

为了较少 OS 调度带来的噪声，对每个数据集进行 8 次测试

### B. Experiment Infrastructure

只记录每个测试用例用于执行 / 追踪的时间

使用 QEMU 作为 baseline tracer

* AFL-Clang
* AFL-QEMU
* AFL-Dyninst
* QEMU (Full speed execution)
* UnTracer (Full speed execution + coverage-guided tracing)

1. 依次重现数据集中的测试用例
2. 测量每个 test case 的 tracing 时间
3. 记录时间到文件中

### C. Benchmarks

* bsdtar - 归档
* cert-basic - 密码学
* cjson - Web
* djpeg - 图像处理
* pdftohtml - doc
* readelf - dev
* sfconvert - audio
* tcpdump - net

有关 baseline：

* 由于使用了 AFL 的 forkserver 作为优化
* 理论上最快的执行方式：
  * 静态 instrumented 的 forkserver，不作任何的 coverage tracing
* 以 baseline 的执行时间作为比较各个 fuzzer tracer 的标准

### D. Timeout

### E. UnTracer versus Coverage-agnostic Tracing

将最后的执行时间转换为相对于 baseline 的相对执行时间

UnTracer 降低黑盒二进制文件的开销四个数量级

### F. Dissecting UnTracer's Overhead

对于 UnTracer 来说，哪个操作对性能影响最大？

所有操作包含：

1. 启动 oracle 和 tracer 的 forkserver
2. 在 oracle 上执行所有的 test case
3. 在 tracer 上执行 coverage-increasing 的 test case
4. 停止 oracle 的 forkserver
5. Unmodifying basic blocks
6. 重启 oracle 的 forkserver

运行 non-coverage-increasing 的测试用例与在 baseline 上执行等价

因此 UnTracer 的唯一开销来自处理 coverage-increasing 的测试用例

为了测量每个操作使用的时间

* 在每个操作的开始和结束处加入计时代码
* 计算每个操作在开销中所占的比例

结果表明，两个最耗时的操作为：

* Tracing - 占开销的 80%
* Forkserver 重启
  * Forkserver 的停止时间是恒定的
  * 重启需要使用大量的进程创建、进程间通信

### G. Overhead versus Rate of Coverage-increasing Test Cases

与 AFL 相比，UnTracer 的 coverage tracing 平均来看要慢一些

* 因为使用了文件操作
* 因此，coverage-increasing 的测试用例越少，UnTracer 的相对性能越好

---

## 7. Hybrid Fuzzing Evaluation

---

## 8. Discussion

### A. UnTracer and Intel Processor Trace

利用硬件的支持，进行更有效的 coverage tracing

Intel 的 Processor Trace (PT) 技术在保留内存中，保存了程序的控制流行为

由于监控位于硬件层，能够完整记录各类 coverage 信息，而开销适度

但 PT 技术需要指定型号 CPU 的支持，并只兼容 x86 的可执行文件

### B. Incorporating Edge Coverage Tracking

### C. Comprehensive Black-box Binary Support

---

## Summary

很有意思。。。

虽然这篇文章的思路第一次读的时候还没有读懂

后面仔细推演了一下 AFL 在指令层所需要做的工作以后

品出了些味儿来 🤤

别的工作都在想尽办法提升变异出的 test case 的质量

尽可能增大 coverage-increasing 的 test case 的比例

而这篇论文直接换了一个思路

用一个挺妙的方法，反而大幅提升了 fuzzing 的性能

我考虑在下次的组会上分享一哈 👏

---

