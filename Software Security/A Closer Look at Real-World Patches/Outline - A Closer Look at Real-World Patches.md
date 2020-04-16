# Outline

## A Closer Look at Real-World Patches

Created by : Mr Dk.

2020 / 04 / 16 11:36

Ningbo, Zhejiang, China

---

## Abstract and Introduction

程序自动修复 (APR) 是使用类似遗传算法的方式，不断对一系列 buggy code 进行编辑，直到所有的 test case 能够被满足为止。带来的局限是产生的 patch 质量较低，本文作者推测局限来源于 APR 技术对于缺陷代码检测的 **粒度**。

大部分 APR 技术在 statement level 进行修复，而本文研究表明大部分的 bug 出现在 statement 结点的 AST 子结点中。因此，直接对整个 statement 进行修复很低效，并且容易影响到本没有错的部分。Buggy part 可以通过更细粒度的方式被定位。

## Background

### Code Entities

Code entity 是组成 AST 的基本部分。本文使用了 Eclipse JDT API，其中包含了 22 个 statement 类型和 35 个 expression 类型。

### AST Diffs

GNU diffs 是以 line 为单位的纯文本表示。而 AST diffs 能够提供不同的 code entity 在不同层次的结构关系。包含以下三个概念：

* Code entity - 代表 AST 中的一个结点
* Change operator - `UPDATE` / `DELETE` / `INSERT` / `MOVE`
* Repair action - code eneity + change operator

## Research Questions

1. 真实的 patch 通常被应用在哪类 statement 类型上？
2. Statement 中的哪些 code element (子结点) 更可能存在缺陷？
3. 在 buggy 的表达式中，哪些部分更可能存在缺陷？

## Study Design

数据来源于较为流行的开源 Java 项目。从这些项目的 commit 记录中，识别出所有的 bug fix commit，从而得到开发者实现的 patch (通过 commit message 中的关键词匹配，或 *JIRA* 中的 fixed bug)。

加下来通过预处理，移除以下的干扰：

1. Bug fix commit 需要包含 `.java` 文件
2. 不包含 test
3. 需要能够被 *GumTree* 转化为 AST

其中，只关注 code change 中规模较小的 hunk。研究表明，规模较大的 hunk 一般属于新功能或重构，而不是 bug fix。

将 buggy 版本和 fixed 版本的代码输入给 *GumTree* 转换为 AST。所有被 **删除** 或 **移动** 的结点都被视为是 buggy 结点，所有被 **插入** 的 statement 被视为是 fixed 结点。对于 **更新** 的结点，进一步判断发生变动的子结点。

## Analysis Results

### Insight 1

声明实体 (class, enum, method, field declaration) 也可能存在 bug，占据了 26.7% 的修复行为。

### Insight 2

Statement 是最主要的 buggy code entities (73.3%)。其中，一半的 repair action 是 `UPDATE`。并且，buggy statement 的类型一般不会改变，而其子结点的 buggy statement 的类型会发生改变。这意味着需要更细粒度的分析。

### Insight 3

表达式 level 的粒度能够降低 APR 工具的 search space，从而提升其效率。

### Insight 4

5.4% 的 patch 的修复方式是将一个 code entity 从一处移动到另一处。很难通过这个简单的修复动作获得一些有价值的信息，需要更多的上下文信息。

### Insight 5

对于一个 statement 的所有子结点，可被分为四类：

* Modifier - 修饰符
* Type - 类型
* Identifier - 名称
* Expression

其中，对于修饰符，Java 语言中只有 12 种。在修复过程中，在修饰符 level 只需要枚举这 12 种即可，能够减少搜索空间。但更改修饰符可能会出现新的问题，比如破坏后向兼容。

### Insight 6

对于 type level 的修复，主要是针对比较具体的问题，比如 integer overflow。这时，可以换用内存更大的数据类型来进行修复。

### Insight 7

对于 identifier 层面的修复可能破坏后向兼容，或影响开发者对程序的理解。

### Insight 8

82% 的修复行为发生在 expression 级别。其中，现实世界中的补丁总是应用在某几个类型的 expression 上。

### Insight 9

在一个 expression 中，也不是所有的部分都是有 bug 的。对于每个 expression 类型，每个子结点的出错概率各不相同。统计信息可以用于对 bug fix 进行分级，提升 APR 工具的效率。在现实世界的 patch 中，很少出现 operator 的错误。

---

