# Outline

## Mining Fix Patterns for FindBugs Violations

Created by : Mr Dk.

2020 / 04 / 12 11:59

Ningbo, Zhejiang, China

---

## Abstract and Conclusion

静态分析工具能够检测出很多的缺陷，但是很多都是 false positive。本文中，通过对静态分析工具报告出的缺陷，以及软件不同版本之间的代码变动，使用 CNN (Convolutional Neural Networks) 提取代码变动的特征，使用 *X-means* 聚类算法将代码变动特征类似的聚为一类，挖据出开发者的 fix pattern，也就是修复缺陷代码的一种抽象表示。然后将 fix pattern 应用到不同的场景中，评估将其用于缺陷修复的效果。

贡献：

1. 静态分析工具在项目版本控制系统中不同版本的检测报告 - 数据集
2. 分析了静态分析工具的报告会被开发者忽视的原因，以及不同类型缺陷的优先程度分析
3. 对缺陷修复模式进行挖掘，得到抽象的、可重用的 fix pattern
4. 对于静态分析工具得到的未被修复的缺陷，应用 fix pattern 产生补丁，评估 fix pattern 用于产生补丁的效果

## Methodology

主要分为五部：

1. 使用静态分析工具采集程序中的缺陷
2. 在程序的历史版本中追踪缺陷
3. 找到缺陷被修复前和修复后的版本
4. 在每类缺陷中，挖掘共同的 code pattern
5. 在修复版本中，挖掘共同的 fix pattern

### Collecting Violations

依次在程序的每一个版本上运行静态分析工具 *FindBugs*，开启最敏感的检测选项，记录信息：

* 缺陷类型
* 缺陷所在的程序、文件、类、函数、起始终止行号
* Commit ID

此外，由于静态分析工具依赖于字节码，所以需要使用 *Maven* 自动化编译流程。

### Tracking Violations

对程序的每一个版本应用静态分析工具，并得到每个版本的缺陷集合。依次遍历每个版本，可能会发现缺陷有增多有减少。由于每个版本的代码会增多或减少，同一个缺陷在两个版本中的位置也会不一样。如何 **在两个版本之间匹配同一个缺陷** 是一个挑战。

1. Location-based matching - 两个版本中缺陷的位移 ≤ 3 行，则认为是同一个缺陷
2. Snippet-based matching - 两个同类型的缺陷的 text string 完全相同
3. Hash-based matching - 当包含缺陷的文件被移动或重命名时使用 - 计算缺陷附近 token 的 hash，并比较两个版本的 hash 是否相同

### Identifying Fixed Violations

本文关注的缺陷：

* 在最新版本中依旧存在的缺陷
* 在以前的版本中存在但在较新版本中修复的缺陷

### Mining Common Code Patterns

对于静态分析工具报告缺陷的位置的代码，进行 code pattern 的提取、挖掘。

一些定义：

* Source Code Entity (Sce) = (Type, Identifier)，是 AST 形式的结点
    * Type 为 AST 结点类型
    * Identifier 是 AST 结点的文本表示 (raw token)
* Code Context (Ctx) = (Sce, Scep, cctx)
    * Sce 定义如上
    * Scep 是 sce 的 parent entity
    * cctx 是 ctx 的子结点
* Code Pattern (CP) = (Scea, Scec, cctx)
    * Scea 是抽象形式的 sce
    * Scec 是具体形式的 sce
    * cctx 是代码上下文，形容了 code pattern 中所有 sce 的关系

如何将相似的代码分组，就涉及到用什么衡量机制来计算代码的相似度。对于变长的 code pattern，一些特征提取策略可能没用。本文使用了 *CNN* 来提取 local features 和 global features。*X-means* 是 *K-means* 算法的拓展 - 由于缺乏先验知识，*X-means* 能够基于贝叶斯信息准则自动推断 *K* 值。

对于 code pattern 中的 AST，需要进行信息提炼。

对于数据的处理，以深度优先的方式遍历 code pattern AST，将 AST 转换为一个 token vector。Token vector 使用 *Word2Vec* 将其中的每一个 token 转换为数值向量，输入 CNN。由于深度学习技术需要相同长度的数值向量作为输入，对于长度不相等的 vector 用 0 补齐。CNN 衡量向量之间的 *余弦相似度*。

使用 CNN 的输出作为缺陷代码的特征。使用这些特征，采用 *X-means* 算法进行聚类。最后，人为地对每一个聚簇打上标签，表示这是哪类缺陷。

### Mining Common Fix Patterns

对开发者修复一个缺陷的行为进行挖掘，找出一种 fix pattern。

定义：

* Patch - 一对源代码片段
    * 在 GNU diff 形式的表示下，buggy version 包含 `-` 开头的行，fixed version 包含 `+` 开头的行
* Fix pattern - 从 buggy code 中提取的一对 code context，以及一序列相关的修改操作
* 修改操作
    * 动作 - 插入、删除、更新、移动
    * 代码实体
    * 子代码实体的所有动作

Patch 代码的变化被转换为 AST 级别的变化 (与 GNU diff 的纯文本级别变化不同)。通过对 AST diff tree 进行深度优先遍历，得到每个 patch 的 token 向量。然后每个 token 又会被表示为一个数值向量。

之后类似地使用 *CNN* + *X-means* 的方法对 fix pattern 进行特征提取和聚类，并人为打上标签。

## Empirical Study

### Statistics on Detected Violations

缺陷在程序中发生的频率到了什么样的程度？

* 绝大部分缺陷都与正确性、编程习惯相关，与 security 相关的较少

### What Types of Violations Are Fixed?

只有很少比例的缺陷会被开发者修复。

* 静态分析工具的 high false positive ratio
* 由于严重程度过低，吸引不了开发者的注意
* 开发者只会关心很少一部分的缺陷
* 如果有工具支持，开发者比较容易接受 *correctness* 类型的问题

### Code Patterns Mining

Code pattern 的挖掘产生了与缺陷描述一致的 pattern，但依旧有很多缺陷由于检测不准确而没有被 fix。另外，对于有一些缺陷类型，其 pattern 很难被挖掘出来。

### Fix Patterns Mining

Fix pattern 的挖掘能够有效将行为相似的缺陷修复聚类到一起。

### Usage and Effectiveness of Fix Patterns

对于一个未被修复的缺陷集合，接近一半的缺陷能够被挖掘出的 fix pattern 解决。这里，选择一个 unfix 缺陷最适合的 k 个 fix pattern 产生补丁 - 具体方式是使用缺陷代码的特征向量，和每个 fix pattern 的聚簇中心点的特征向量的距离 (缺陷代码和哪些 fix pattern 中的未修复代码最相似)。其中 1/4 的缺陷都能直接被第一个选中的 fix pattern 解决。

将 fix pattern 直接应用到 *Defects4J* 的已经 bug 上。其中 14 个 bug 能够被静态分析工具检测到 - 原因是这个数据集中的 bug 主要是功能性 bug，而不是违反了一个编程规则，因此只能在 runtime 中由 test case 触发，很少能够通过静态分析工具检测出来。在 14 个被静态分析工具检测出的 bug 中，有 4 个能够被 fix pattern 完全修复。

对 10 个开源  Java 项目的最新版本，使用静态分析工具进行检测，并应用 fix pattern 产生补丁 (补丁的产生暂时是人为的)，提交 PR。在产生的 116 个补丁中，67 个被立即 merge，2 个补丁被开发者确认。因此 fix pattern 有能力修复目前广泛存在的 bug。

几个规律：

1. 被维护得较好的项目不太容易出现常见的缺陷类型
2. 开发者能够根据本文方法提供的 fix pattern 来写出有效的补丁
3. 开发者不会接受一些看似有用实则没必要的补丁
4. 一些基于 fix pattern 的补丁可能会产生后向兼容的问题
5. 一些缺陷类型几乎不会产生什么影响
6. 一些 fix pattern 会使程序无法编译
7. 一些 fix pattern 会使程序无法 check style
8. 一些 fix pattern 产生的补丁会引发争议

---

