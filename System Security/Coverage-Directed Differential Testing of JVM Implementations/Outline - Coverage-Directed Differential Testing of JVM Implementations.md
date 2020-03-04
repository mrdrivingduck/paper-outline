# Outline

## Coverage-Directed Differential Testing of JVM Implementations - PLDI 2016

Created by : Mr Dk.

2020 / 02 / 06 18:50

Ningbo, Zhejiang, China

---

## Abstract

对 JVM 实现进行测试需要大量努力构造 `.class` 文件

一种替代方案是盲目地对字节码文件进行变异，但会导致变异后的文件不合法

本文提出的方案

使用具有代表性的字节码文件

对于 JVM 的启动阶段进行差别化测试

* 使用一系列预定义的变异动作，以及 MCMC 采样法，来引导变异过程
* 使用一个范例 JVM 实现执行字节码，使用 coverage 唯一性来选择具有代表性的字节码
* 具有代表性的字节码文件被作为测试多个 JVM 实现的输入

目标是寻找 JVM 的实现漏洞

---

## 1. Introduction

JVM 负责加载、链接、执行 Java 字节码

目前存在各种各样的 JVM 实现

* Oracle 的 HotSpot
* IBM 的 J9
* Jike 的 RVM
* Azul 的 Zulu
* GNU 的 GIJ

这些 JVM 使用了不同的实现技术，适配不同的 OS 和 CPU 架构

为了保证兼容，它们必须遵循并实现同样的 JVM 规范

然而，对于一些 corner cases 和不合法的字节码文件，不同 JVM 的行为不一致

* 一些 Java 类可以运行在一个 JVM 上，却不能运行在另一个 JVM 上，导致一些应用在一些 JVM 上运行时可能会不安全
* JVM 自身可能会崩溃

JVM 的差异主要体现在两个场合：

1. JVM 实现存在缺陷，开发者不可能在开发各自的 JVM 时犯同样的错误
   * 程序 bug (空指针)
   * 违反一致性
     * JVM 规范有 590 页，可能存在歧义
     * 且对于一些未知状态 `undefined`
     * JVM 实现很难严格服从 JVM 规范
2. 在 JVM 实现之间存在兼容性问题
   * _Java Native Interface_ (JNI) 允许 Java 代码调用 C/C++ 或汇编
   * HotSpot 继续运行，而 J9 发生崩溃

定义 JVM 的 __差异__ 和 __缺陷__ - 将 JVM 执行定义为 `r = jvm(e, c, i)`

* `jvm` 对应不同的 JVM 实现
* `e` 对应于与执行相关的环境
* `c` 对应于一个 Java 类
* `i` 对应于输入
* `r` 对应于可被观测到的行为 (输出或者错误)

JVM 差异：

* `r1 = jvm1(e1, c, i)`
* `r2 = jvm2(e2, c, i)`
* `r1` 和 `r2` 具有不同的输出或错误

给定相同的类，不同的 JVM 有不同的行为

比如，一个 JVM 可以正常运行，另一个会报出错误

当拒绝一个字节码文件时，不同的 JVM 会给出不同的错误原因

JVM 缺陷：

* `r1 = jvm1(e, c, i)`
* `r2 = jvm2(e, c, i)`
* `r1` 和 `r2` 具有不同的输出或错误

> 没看出有什么区别......

JVM 差分测试面对两大核心挑战：

1. 字节码文件需要被用心构造
   * 一种代替方案是对字节码种子文件进行盲目变异，并在不同的 JVM 上运行
   * 字节码文件的数量过多，且存在大量冗余，大量的用例可能只能触发小部分的 JVM 差异
2. JVM 差异很频繁，但大部分只是兼容性问题，而不是缺陷
   * 这种问题可以通过强制 JVM 在特定环境 (比如 JRE 7) 中运行而被消除
   * 不必修复 JVM 自身

本文提出的 _classfuzz_ 致力于构造 __具有代表性的__ 字节码文件

用于测试不同 JVM 的启动过程

具有代表性，指构造出的字节码文件对于测试来说是 __不同且唯一__ 的

作者将被测试的字节码文件放到一个参考 JVM 实现上运行

使用 coverage 的唯一性作为指标来选择具有代表性的字节码文件

_Classfuzz_ 的本质是不停地对字节码文件进行变异，只留下具有代表性的字节码文件

这些留下的字节码文件用于进行差别化测试

本文的主要贡献：

1. 测试用例的代表性
2. Coverage 指引的输入变异
3. 实现和评估

---

## 2. Approach

### 2.1 Background: Class Format and JVM Startup

一个字节码文件是按结构组织的，用于表示一个 Java 类或接口编译后的形式

JVM 的启动过程 - 加载一个类，并调用其 `main()` 函数

* 加载
* 链接
* 调用

在任意一步中，如果任何约束不能满足，JVM 都会抛出内置的错误或异常

### 2.2 Coverage-Directed Class File Mutation

_Classfuzz_ 会选择一个初始的种子库，并迭代对它们进行变异

之后将其放到参考 JVM 上进行执行

只有其产生了唯一的执行路径，才会被接受为差别化测试的 test case

对这样的 test case 进行变异，更容易产生新的具有唯一执行路径的 test case

#### 2.2.1 Mutating Class Files

使用 _Soot_ 来对 test case 进行变异

* 这是一个先进的 Java 分析转换框架
* 提供了丰富的 API 进行解析、转换、重写

本文作者定义了 129 个变异方法，用于对字节码进行修改

_Classfuzz_ 将一个字节码文件读取为 _SootClass_

使用变异方法来重写该类

* 改变父类
* 增加接口
* 重命名一个域或函数
* 删除一个抛出的异常
* ......

最终将 _SootClass_ 注入到一个新的字节码文件中

为了便于测试，在每个产生的字节码文件中加入一个 `main()` 函数

并在函数中加入一行简单的输出信息，表示该类被正常加载

#### 2.2.2 Mutator Selection

129 个变异方法不是同样有效的，因此怎样选择变异方法成为一个问题

_Classfuzz_ 使用了 _Metropolis-Hastings_ 算法

这个算法属于 _Markov Chain Monte Carlo_ (MCMC) 方法

用于从一个概率分布中进行随机采样

具体方法是产生与期望分布接近的样本序列

样本是挨个迭代产生的

后一个样本是否被接受只取决于前一个样本

在本文中，样本对应于待选择的变异方法

设期望分布为 __几何分布__，基于 __Bernoulli trials__

* 如果每次尝试的成功概率为 _p_
* 那么第 _k_ 次尝试获得第一次成功的概率为 `p(X = k) = (1-p)^(k-1) * p`

定义每一个变异方法 `mu` 的成功率 `succ(mu) = mu 构造的有效 test case / mu 被选择的次数`

(选择一个 mu 构造 test case，只有这个 test case 在参考 JVM 上的执行路径唯一，才会被留下)

将所有变异方法的成功率倒序排序

MCMC 过程保证了具有较高成功率的变异方法更容易被选中

_Metropolis-Hasting_ 算法随机选择变异方法，通过 _Metropolis choice_ 接受或拒绝下一个变异方法

化简得到最后的接受概率：`A(mu1 → mu2) = min(1, (1-p)^(k2-k1))`

* `k1` 和 `k2` 分别为 `mu1` 和 `mu2` 的下标
* 如果 `mu2` 的成功率比 `mu1` 高，`mu2` 就总是会被接受
* 如果 `mu2` 的成功率比 `mu1` 低，就按 `(1-p)^(k2-k1)` 的概率接受
  * `k2 - k1` 越大，接受概率越低

在每一轮迭代中，所有变异方法的成功率都需要被重新计算、重新排序

参数估计 - 需要满足三个条件：

1. 全概率接近 1
2. 选择成功率最高的变异方法的概率高于 `1/129`
3. 具有最低成功率的变异方法依旧有被选到的机会

得到 `p` 的范围在 `(0.022, 0.025)`，选择 `p = 3/129(≈0.023)`

#### 2.2.3 Accepting Representative Class Files

_Classfuzz_ 在一个参考 JVM 上执行变异后产生的字节码文件

只留下具有代表性的字节码文件，作为未来测试的 test case

本文作者采用 coverage 的唯一性来决定接受或拒绝一个字节码文件

设 `TestClasses` 是已被选中的 test case，现要判断 `cl` 是否应当被加入 `TestClasses`

提出了以下三种选择指标：

1. `[st]` - `cl` 的语句覆盖和 `TestClasses` 中的每一个 test case 都不同
2. `[stbr]` - `cl` 的语句覆盖和分支覆盖和 `TestClasses` 中的每一个 test case 都不同
3. `[tr]` - 同 (2)，但忽略语句的执行顺序、分支、频率

### 2.3 Differential Testing of JVMS

将选为 test case 的字节码文件放到五个最先进的 JVM 实现上执行

并比较执行结果，查看是否存在不一致的行为

所有的 JVM 都能够通过执行 `java classname` 来执行

检查 `main()` 函数是否正常被调用，或查看错误 / 异常是从哪个阶段被抛出的

* `(0)` - 正常调用
* `(1)` - 在加载阶段被拒绝
* `(2)` - 在链接阶段被拒绝
* `(3)` - 在初始化阶段被拒绝
* `(4)` - 在运行时被拒绝

这五种输出被编码为序列，如果序列不是五个相同的数，说明出现了 JVM 差异

这里可能引入误差

* 对于具有多个不合法结构的字节码文件，不同的 JVM 可能会报告不同的错误

当 JVM 差异出现时，需要有一定的方法来追溯字节码中触发 JVM 差异的 root cause：

1. 给定可以触发 JVM 差异的字节码，删除其中的一个函数、语句、域，重新产生字节码
2. 重新测试产生的字节码，如果 JVM 差异依旧存在，则用这个字节码替掉原来的字节码

直到留下一个足够简单的字节码，且可以触发 JVM 差异

通过这个字节码文件，可以相对简单地找出哪一部分可以产生 JVM 差异

---

## 3. Evaluation

### 3.1 Setup

#### 3.1.1 Preparation

使用 Java 9 的 HotSpot JVM 作为参考 JVM

1. 在所有的 JVM 实现中，支持最多的功能 (2016 年)
2. 开源，便于提取 coverage

HotSpot 源码达到 260K LOCs，提取一次执行的 coverage 需要 30+ min

因此只追踪用于 JVM 启动功能的某个目录下的 coverage

* 11977 LOCs
* Coverage 分析减少到 90s

随机从 JRE 7 的库中选择 1216 个字节码文件作为 seed

将字节码的 major version 设置为 `51`，五个 JVM 都能识别

* JVM 对不同版本的字节码文件有着不同的验证算法
* 如何创建不同版本的字节码文件不在本文的讨论范围中

#### 3.1.2 Evaluated Methodology

评估 _classfuzz_ 使用 `[st]` `[stbr]` `[tr]` 中的哪一种衡量指标效果更好

除了这三种衡量方式，还引入了另外三种变异算法：

* Randfuzz - 随机变异 seed 字节码
* Greedyfuzz - 当变异引发 code coverage 提升后，才接受该变异
* Uniquefuzz - 只有当 coverage 唯一时才接受变异，但区别是，随机选择变异方法

#### 3.1.3 Metrics

算法的成功率 `succ(X) = |TestClasses| / Iterations`

* 体现算法产生 test case 的能力

对于 `TestClasses` 中的所有字节码文件，分别放到五个 JVM 上运行

差异率 `diff = |Discrepancies| / |TestClasses|`

* `Discrepancies` 是产生 JVM 差异的字节码文件
* 差异率越高，说明五个 JVM 越不一致
* 当两个差异序列有相同的编码时，说明这两个差异属于同一类

### 3.2 Results on Class File Generation

_Randfuzz_ 产生的字节码文件多 20 倍，其余算法产生字节码的速度较慢

_Classfuzz[stbr]_ 产生的有效 test case 最多 (成功率最高)，原因在于：

1. 很多 seed 不合法，无法进行变异
2. 很多变异后的字节码文件不合法

另外，通过变异具有代表性的 test case，更容易变异出具有代表性的 test case

_Classfuzz_ 可以利用已有的知识选择变异方法

* 变异方法的成功率越高，被选中的概率越大
* 有一些变异方法很难产生 test case
* 排名靠前的变异方法更容易重写程序结构

MCMC 采样在三天内帮助多变异了 43% 的 test case

### 3.3 Results on Differential JVM Testing

JVM 的差异能够被 test case 放大

使用 `[stbr]` 配置的 _Classfuzz_ 算法能够找到数量最多的 __唯一性差异__ (62 种)

_Randfuzz_ 虽然能触发 5537 个 JVM 差异，但只能被归并为 14 种唯一差异

其中，28 种差异体现了 JVM 内部实现存在缺陷

1. JVM 规范中阐述不明确造成了误解
2. JVM 实现有其自己的字节码验证和类型检查方式
3. JVM 实现对于访问一些特定的类不兼容
4. _GIJ_ 对字节码的限制比 _HotSpot_ 和 _J9_ 更宽松

---

## 4. Related Work

* Directed random testing
* Mutation-based fuzz testing
* JVM testing
* MCMC sampling for testing

---

## Summary

正好寒假读完了 _深入理解 Java 虚拟机_

所以看了这篇一直没有打开的论文

目前，fuzzing JVM 这个方向好像很少有人做

但是本文的出发点是寻找不同 JVM 实现中的差异

并从差异中找到 JVM 的实现缺陷

---

