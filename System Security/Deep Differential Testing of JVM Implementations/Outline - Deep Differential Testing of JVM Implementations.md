# Outline

## Deep Differential Testing of JVM Implementations - ICSE 2019

Created by : Mr Dk.

2020 / 02 / 08 21:11

Ningbo, Zhejiang, China

---

## Summary

本文基于作者三年前的工作 [Coverage-Directed Differential Testing of JVM Implementations](https://mrdrivingduck.github.io/#/markdown?repo=paper_outline&path=System%20Security%2FCoverage-Directed%20Differential%20Testing%20of%20JVM%20Implementations%2FOutline%20-%20Coverage-Directed%20Differential%20Testing%20of%20JVM%20Implementations.md) 继续进行。之前的工作只能测试 JVM 的启动阶段 (类加载、链接、初始化)，但是 JVM 内部还有很多其它模块可以被测试。本文的目标是能够进一步测试 JVM 的字节码验证模块和 JVM 的执行引擎 - 当然，为了使产生的字节码能够进入 JVM 的这两个阶段，产生字节码的方式肯定要比之前工作的工作更为严格。之前工作中，产生的字节码只需要保证能被 JVM 加载、链接、初始化就可以了；而本文中产生的字节码不仅需要能够通过上述这三个步骤，还要能够经过字节码验证模块的验证，并交给执行引擎进行执行。

可能是由于本文发现了 _J9_ JVM 的 CVE，因此上了顶会

## 1. Introduction

之前的 _classfuzz_ 在语句上对字节码进行变异，用于测试 JVM 实现的启动阶段 - 加载、链接、初始化，但无法测试 JVM 的 __字节码验证__ 和 __执行引擎__ ：

* 字节码验证确保每个 Class 文件满足链接阶段的约束
* 执行引擎负责即时编译和执行 Java 字节码

_Classfuzz_ 产生的大部分字节码文件无法深度测试 JVM 实现。

本文提出了新的 __活代码变异__ 机制，用于从 seed 产生合法的、可执行的字节码。

解决的挑战：

1. 采用简单策略插入、删除或修改字节码指令大部分产生的是不合法的 test case
2. 字节码文件的选择
   * 由于进入执行引擎的字节码会被即时编译，JVM 执行引擎的 coverage 不是固定的
   * 无法使用 _classfuzz_ 中的 coverage 唯一性来选择字节码文件是否要被留下

本文引入了 _classming_ ：

1. 记录 seed 字节码文件的活代码
2. 修改活代码中的控制流和数据流，产生语义上不同的代码
3. 产生的变异代码用于测试 JVM，暴露不同 JVM 的实现差异和潜在的威胁

_Classming_ 使用了一种迭代式的变异代码生成方式，通过算法来判断是否要接受产生的变异代码。

本文的贡献：

1. 目标是测试 JVM 的字节码验证模块和执行引擎
2. LBC(Live Byte Code) 变异方式
3. 测试了 _OpenJDK_ 的 HostSpot 虚拟机和 _IBM_ 的 J9 虚拟机

---

## 2. An Illustrative Example

与之前的工作类似，还是使用 _Soot_ 框架对字节码进行重写。将字节码文件转换为 _Soot_ 的 _Jimple_ 代码，修改，然后生成字节码 dump 到文件中：

在代码中插入了改变控制流的语句，不同 JVM 实现产生了不同的行为：

* 一个抛出异常，一个正常运行 (对 JVM 规范的理解偏差)
* 两者抛出了不同的异常
* 有的虚拟机直接发生了 crash

---

## 3. Approach

总体的思路是，不断地产生，并有选择性地接受产生的 test case。用 test case 在不同的 JVM 实现上执行，目标是观测到一些感兴趣的差异：

* JVM crashes
* 字节码验证过程中的差异
* 执行过程中的差异

### 3.1 Live Bytecode Mutation

活代码指的是会被 JVM 在运行时执行到的字节码指令序列。LBC 变异的算法步骤如下。

#### 3.1.1 Select LBC Mutators

使用 _Soot_ 的五条 _Jimple_ 指令

* `goto`
* `return`
* `throw`
* `lookupswitch`
* `tableswitch`

因为只有这五条指令可以在运行时改变程序控制流，算法选择一个指令插入到字节码中。当然，一个已经被插入到字节码中的指令也可以被移除。

#### 3.1.2 Select Methods to Mutate

在每一轮迭代中，选择一个类函数进行变异。具有复杂结构和指令更多的函数应当被变异地更频繁，而不是随机挑选一个函数进行变异。

定义一个函数的变异 _潜力_ - `potential(m) = #inst / #mutation`

* `#inst` 代表指令条数
* `#mutation` 代表该函数被变异的次数

函数的潜力越高，就越容易被选中进行变异。在变异后，函数的潜力下降。选择不同函数的概率满足几何分布，使得潜力越高的函数有更高的概率被选中。

#### 3.1.3 Insert Hooking Instructions

* Select hooking points
  * 插入点故意摧毁 seed 程序的数据依赖，产生一些 corner case
  * 假设插入点能够截断经过该点的全部数据依赖，那么截断地越多，产生异常数据流的概率越大
  * 在程序中随机选几个插入点，并计算数据依赖的截获量，选择最多的那个作为指令插入点
* Select target points
  * 一些跳转指令需要插入 label
  * 偏向于将 label 插入到之前没有被执行过的指令前
    * 如果指令从未被执行，则接受成为跳转目标
    * 如果指令在上一次变异中没有被执行到，则以较高概率接受成为跳转目标
    * 否则以较低概率接受

### 3.2 Mutant Acceptance

_Classming_ 与 _classfuzz_ 类似，以一定指标来判断是否留下变异后产生的 test case

#### 3.2.1 Seed Coverage

用于衡量变异后的 test case 中覆盖了多少条原始 seed case 中的指令

* 不同的 seed coverage 必然会有不同的语义，即不同的运行时行为
* 从具有较高 seed coverage 的 seed 中变异出的代码质量更高

记录下每个变异后 test case 的活代码

Seed coverage 定义为 `cov(g) = y / x`

* `x` 为所有的指令
* `y` 为变异后的 test case 在 JVM 上运行时可以被覆盖到的指令

#### 3.2.2 A Sampling Process

算法需要选择将部分变异后的 test case 留下，作为新的 seed。算法的目标是以指定的概率分布接受样本序列，文章中选取的是 _指数分布_ - 产生 test case 后，运行该 test case 得到 seed coverage，计算出接受概率：

* 对于可以被 JVM 运行的 test case (活代码)，以接受概率为概率接受它
* 对于不能被 JVM 运行的死代码，则拒绝

---

## 4. Evaluation

### 4.1 Evaluation Setup

* _Classfuzz_ 的变种
  * 使用六个指令级的 mutator 对输入进行变异
  * (插入、替换、删除、交换、重复、克隆)
* _Clrandom_ - 随机修改控制流和数据流
* _Clgreedy_ - 在接受变异后 test case 时，只保留能使累计 seed coverage 增加的 test case

以 Java 9 的 HotSpot JVM 作为参考 JVM，用于 test case 的 coverage 数据的收集和选择。

衡量指标：

* Stillborn rate
  * `sbr = 1 - MUTANT / #iter`
    * `MUTANT` 是产生的合法 test case 数量
    * `#iter` 是迭代次数
  * 用于衡量算法能否有效产生合法的 test case
* Accumulative seed coverage
  * 衡量一个 seed 被用于变异产生 test case 的完整度
  * `asc = cov(g1) + cov(g2) + ...` (由 seed 变异产生的所有 test case 的 seed coverage 的合并)
* JVM code coverage
  * 衡量 test case 对 JVM 代码覆盖的提升
  * 在参考 JVM 上使用 _Jcov_ 得到 coverage，计算在原有 seed 的 coverage 基础上的提升
  * `Jinc = Jcov(MUTANT) + Jcov(seed) - Jcov(seed)` (`+` 不是加法，是合并)
* JVM differences
  * JVM 行为差异

### 4.2 RQ1: Sufficiency of Classming-generated Mutants

_Classfuzz_ 的 stillborn rate 显著较高，因为其产生了大量的不合法 test case，甚至无法被 _Soot_ 转换为字节码文件。因此，基于活代码的变异更为有效。

### 4.3 RQ2: Effectiveness of Classming-generated Mutants

#### 4.3.1 Accumulative Seed Coverage

_Classming_ 的 accumulative seed coverage 高于 _clgreedy_ ，说明算法使得 _classming_ 能更合理地探索种子的变异空间。

#### 4.3.2 JVM Code Coverage

_Classming_ 使得 JVM 的 code coverage 有所提高。特别地，字节码验证模块、即时编译器的 coverage 占增多部分的 72.1%。说明变异产生的 test case 能更好地测试 JVM 的字节码验证模块和执行引擎。

#### 4.3.3 JVM Differences

只有 _classming_ 产生的 test case 引发了 JVM 字节码验证部分的行为差异。另外，行为差异也有可能是冗余的，有意义的指标应该是行为差异的唯一种类 - _Classming_ 能够触发更多种独立的 JVM 行为差异。

### 4.4 RQ3: Difference Analysis and Bug Report

* Security vulnerability in _J9_
  * 字节码验证模块未能拒绝一个对未被初始化对象的使用
* Defects in bytecode verifiers
  * 对语义相同的代码的处理不同
* JVM crashes
* Other execution differences

---

