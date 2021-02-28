# Outline

## A Manual Inspection of Defects4J Bugs and its Implications for Automatic Program Repair

Created by : Mr Dk.

2021 / 01 / 01 🎆 16:26

Nanjing, Jiangsu, China

---

## Abstract

由于现实程序中的 test suite 太过于脆弱，以至于无法揭露程序中的一些缺陷。为了理解缺陷到底能在什么程度被修复，作者对 50 个 *Defects4J* 数据集中的 bug 进行 **人为分析** (按照人的理解方式试图去修复 bug)，并正确修复了 82% 的 bug。在人为分析的过程中，作者归纳出了七条 **缺陷定位策略** 和七条 **补丁生成策略**，可以被未来的缺陷定位工具和 APR 工具使用。

## Dataset and Environment

人为分析的几个前提：

* 对要分析的程序没有预先知识 (即不清楚程序的完整规约)
* 只能依赖程序源代码产生补丁，不能查看开发者提供的补丁
* 可以查看源代码中的 Javadoc 和注释，但不能参阅额外文档
* 定位 + 修复 + 验证，直到产生 plausible patch，上限五小时

## RQ1: Manual Analysis Result of Defects

41/50 个 bug 在上述条件下被成功修复，说明大部分的缺陷在现有的 test suite 下也具备被修复的潜质。对于 9 个无法被修复的 bug，无法被修复是因为这些 bug 的修复需要对项目工程的 domain knowledge。

## RQ2: Fault Localization Strategies and Implications

### Strategy 1

如果一条语句没有被 failed test case 执行到，那么这条语句肯定不是缺陷的 root cause。

这条策略已经被任意的缺陷定位技术所采用 - 疑似度的计算方式决定了没有被 failed test case 执行到的语句的疑似度一定为 0。

### Strategy 2

对于一系列可能的 root cause (程序调用图)，移除掉其中不可能包含缺陷的部分：

* 库函数
* 简单的封装函数

### Strategy 3

考虑 stack trace 中出现的位置附近的语句，疑似度应该被提升。

### Strategy 4

识别程序中不期望出现的值变化。

### Strategy 5

简称程序中违背常理编程习惯的语句 (比如 `if (a = 0)`)。

### Strategy 6

使 `if` 语句的条件反转后，failed test case 走过了另一条路径并通过了测试，那么就可以考虑 `if` 语句内是否存在缺陷。

### Strategy 7

对程序进行理解：

* 注释
* 函数名与函数功能之间的联系

## RQ3: Patch Generation Strategies and Implications

### Strategy 8

加入空指针 checker。

### Strategy 9

如果 failed test case 是一个边界值，那么考虑使用直接返回一些特殊的常量。

### Strategy 10

如果两个标识符很相似，将其中一个替换为另一个。

### Strategy 11

通过对失败测试用例和通过测试用例进行对比，找出可能有用的信息。

### Strategy 12

理解注释。

### Strategy 13

模仿类似的代码片段。

### Strategy 14

理解程序的功能。

## Conclusion

1. 很多策略十分简单，不需要对程序有完整的理解，或着深入的语义理解
2. 单独的策略不管用，组合使用效果更好

---

