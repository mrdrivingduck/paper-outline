# Outline

## History Driven Build Failure Fixing: How Far Are We? - ISSTA 2019

Created by : Mr Dk.

2020 / 04 / 25 13:49

Ningbo, Zhejiang, China

---

## Abstract and Introduction

这篇文章主要专注于修复 Gradle 脚本中的错误。对比以往工作，修复 Gradle 脚本需要大量的历史修复信息，并且只能修复一些特定的错误。本文提出了一种新的修复方式，通过对 **当前工程** 和一些 **外部资源** 而不是 **历史修复信息** 进行修复，并取得了更好的效果。

在 *ICSE 2018* 的一篇文章中，其作者使用项目历史上的成功的 fix 记录作为训练集，学习到了 fix pattern，应用到现有的错误上。本文作者重新构建了一个更大的信息数据集，并对这种方法进行了分析，发现结果并没有那篇论文中那么好。

## Background

### Build Failure Fixing: Challenges

首先的挑战是 **错误定位** 的问题。错误在某一行被揭露，但错误的根源并不在这一行。

其次的挑战是，已知错误位置后，如何产生正确的补丁。产生补丁意味着要从脚本或其它位置搜索修复原料，因此需要外部资源相关的信息。比如，修复某个依赖的版本号，那么就需要知道它有多少个合法版本号。

### State-of-the-Art HireBuild

这是目前最先进的技术，也是本文的比较对象。它使用了 Gradle 历史 build failure 的 fix 作为训练集学习 fix pattern，并基于 **预定义的规则** 产生补丁。

首先，在训练集中，选取 error message 和当前错误比较相近的 fix，从中提取 fix pattern (AST)。在补丁生成阶段，预定义了一些规则，并将 pattern 中的抽象部分用具体的值填充：

* Identifiers (task name, block name, variable name)
* Gradle 第三方插件或库的名字
* 工程内的文件路径
* 版本号

然而，除了这四种元素以外，Gradle 脚本中还有很多元素，特别是在该工程脚本中特有的一些变量 (这是 fix pattern 不可能从其它脚本中学习到的)。所以作者认为这种方法的局限性较大。

## Study on HireBuild

作者对本文的对比目标进行了深入的实验分析，并得到了几点启示：

* 历史信息没啥用
* 脚本中目前信息的可用性
* 需要有更多的 fix pattern
* 考虑利用脚本中更多的元素

## A New Technique: HoBuff

总体来看，Gradle 脚本可被视为很多 configuration 的集合，每个 configuration 包含其 value。大多数的构建错误可被认为是 configuration 错误，包括给 configuration value 的错误赋值，或者 configuration 的缺失。

### Dataflow-Based Fault Localization

主要是使用 build log 中的错误信息，来推断错误发生的位置。首先通过一些 NLP 的方法，提取出 build log 中相关的错误，推断出 **揭露** 错误的 statement。然后通过以下分析得到 root cause 疑似度：

* Inter-procedural analysis
* Context-insensitive analysis
* Field-sensitive analysis

### Search-Based Patch Generation

如果在应用了 patch 之后错误消失，则认为 patch 是合法的。

本文定义了以下三种操作符：

* Update
* Insertion
* Deletion

其中，delete 操作符不需要原料。应用顺序为 update-insertion-deletion。

对于用于修复错误的原料代码，主要分为两部分：

* 内部原料 - 主要是工程内定义的 configuration element
* 外部原料 - 主要是来自网络：
    * Gradle central repository
    * Gradle DSL document
    * Android DSL document

## Comparison Evaluation

### Research Questions

* HoBuFF 在修复 build failure 的数量上表现如何？
* HoBuFF 在修复时间和产生的候选补丁数量上表现如何？

对于 error message 过于抽象的错误，或者修复逻辑过于复杂的错误，HoBuFF 无法修复。

---

