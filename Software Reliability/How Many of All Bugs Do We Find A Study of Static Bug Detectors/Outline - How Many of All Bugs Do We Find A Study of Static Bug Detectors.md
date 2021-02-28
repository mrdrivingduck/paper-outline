# Outline

## How Many of All Bugs Do We Find A Study of Static Bug Detectors - ASE 2018

Created by : Mr Dk.

2020 / 03 / 27 19:26

Ningbo, Zhejiang, China

---

## Abstract and Introduction

静态的 bug 检测工具正越来越受欢迎。本文与已有研究的侧重点不同。已有研究大多关注于检测出的 bug 中，有多少 bug 是真正的 bug - 即检测的 precision 如何。而本文关注在一个真实的 bug 集合中，有多少 bug 能被静态工具检测到 - 即检测的 recall 如何。

本文使用了 _Defects4J_ 作为评估的 bug 集合，比较了三个静态分析工具：

* _Google_ 的 _Error Prone_
* _Facebook_ 的 _Infer_
* _SpotBugs_

这些工具都是使用基于 AST 的 pattern 匹配，或者进行 data-flow analysis 进行检测的，并且能够扩展更多的 checker 进去。静态分析的优势在于能够在程序开发的过程中就能发现 bug，并且不需要搭建特定的测试环境。

本文注重于研究这些静态分析工具的 recall，即静态分析工具在所有 bug 中能检测出多少，而不是 precision。已有研究不注重 recall 的原因在于，所谓的 __所有 bug__ 是未知的。另外，对于这些工具的横向对比能够让我们获知，这些工具是能够互相补充，还是一个工具能完全替代另一个工具。

本文工作的主要挑战在于，静态分析工具分析出的结果 (警告) 是否与 _Defects4J_ 中的 bug 是相关联的。作者提出用 __自动化的行级匹配__ + __人工分析__ 来进行判断，原因在之后的章节中具体分析。

## Experimental Procedure

首先，最简单的方法 - 用分析工具去分析 _Defects4J_ 的 buggy 版本，然后人工检查工具输出的每一个警告是否与 bug 相关。由于数据集中的 bug 较多，不可行。

本文提出了一种 __自动化地匹配 warning 和 bug__ 的想法。假设分析工具报告警告的位置就是 bug 的位置。这种假设可能会产生一个过度估计的问题 - 可能分析工具在 bug 的位置发出了警告，但警告的内容与实际的 bug 并无关系。

因此，为了避免这种过度估计，在自动匹配之后，本文的作者会进行人工分析，判断 warning 和 bug 是否真的有关联。如果有关联，分析工具才算真正检测到了这个 bug。

### Identifying Candidates for Detected Bugs

实际上，bug 位置及其 fix 位置不一定是一样的；但本文假设它们在一起。这也成为了本文的 threat to validity。基于这种假设，作者将 buggy 版本和 fix 版本的 diff 作为 bug 的位置。另外，让分析工具只分析具有 diff 的文件。

对于自动化匹配 warning 和 bug，本文提出了三种方法：

1. Diff-based - 只保留分析工具对具有 diff 的代码行发出的警告
    * 在 buggy 版本中，提取至少被分析出一个警告的代码行
    * Diff 中发生更改的代码行 ∩ 被分析出警告的代码行
2. Fixed Warning-based - 比较 fix 前后分析工具报告的 warning，一个因 bug 而产生的警告应该会随着 bug 的修复而消失
    * 只保留在 buggy 版本中存在而在 fix 版本中消失的 warning
3. 前两种方法的组合 (取并集)

### Manual Inspection and Classification of Candidates

自动匹配中会有很多的错误。因为分析工具对某一行发出了警告，理由有千千万万，可能与这个位置的 bug 完全没有关系。如果是这样的情况，匹配完全是巧合。

所以要通过人工的工作，将以上的候选者分为三类：

1. Full match - 工具发出的 warning 与这个位置的 bug fix 完全相关
2. Partial match - bug 的修复不仅与 warning 相关，还有一些不相关的工作
3. Mismatch - bug 的修复与 warning 完全没有关系

## Experimental Results

* _Error Prone_ 检测出的最频繁的 bug 是 `@Override` 注解的缺失
* _Infer_ 检测出的最频繁的 bug 是潜在的空指针解引用
* _SpotBugs_ 检测出的最频繁的 bug 是 `switch` 中缺失的 `default`

自动匹配的工作显著减少了候选匹配的人工分析工作 (降低 97%)。自动匹配得到了 153 个 warning 以及 89 个对应的 bug。经过人工检查，三个工具联袂检测出了 31 个 bug，其中 27 个是唯一的 - 因此静态分析有着不可忽视的作用。

而三个工具联袂检测出来的 bug 只占所有 bug 的 4.5% - 因此有着较大的提升空间。其中，这三个工具各有其独自发现的 bug - 因此可以作为互相的补充。

对于 4.5% 以外的 bug，为什么静态分析工具检测不出来呢？

### Domain-specific Bugs

这是每个程序中特有的 bug，而适用于一种通用的 bug pattern，因此不能被 pattern checker 检测出来。通常是一些算法的错误实现，或者漏掉了某种特殊情况。

* 比如 Math-67 的 bug 是一个数学最优化算法，本该返回最优化的值，却错误返回了最后一个计算的值

### Near Misses

有一些 bug 的根本原因其实分析工具是支持的，但是 checker 却没有捕捉到。比如，checker 只能进行过程内分析，而有一些死递归或是 out-of-bound 访问是跨过程的。在一个函数中计算 array index，在另一个过程中进行 array 的访问。

另一些 bug 的根本原因其实与检测机制类似，但 checker 没有做特定的实现。比如对于普通数组的 out-of-bound 访问，checker 是支持的；而真实的 bug 使用的是 ArrayList，checker 就没有检测出来，但是检测的原理是类似的。

### Assessment of Methodologies

对于自动匹配提出的两种方法 - diff-based 和 fixed warnings-based，在一定程度上两者是互补的。所以将这两种方法结合，效果更好 - 即第三种方法。

Diff-based 的方法比 fixed warning-based 方法产生了更多的 mismatch。此外，在经过人为分析后发现，fixed warning-based 方法足以识别所有正确检测到的 bug。因此，这个方法只需要较少的人力投入，就能够揭露所有检测到的 bug。

另外，作者发现了大量的警告，虽然位于 bug 的实际位置，但只是凑巧，与 bug 并没有实际关系。因此，作者得出结论，人为分析也是重要的一环。

## Summary

1. 静态分析的结果不可忽视
2. 不同的 bug detector 能够互补
3. 静态分析工具无法检测 95.5% 的 bug，有巨大的提升空间
4. 大部分 miss 的 bug 与程序本身相关，没有共同的 pattern，可以用其它更有效的方法来检测

---

