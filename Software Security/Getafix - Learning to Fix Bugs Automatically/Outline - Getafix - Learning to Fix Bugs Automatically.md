# Outline

## Getafix: Learning to Fix Bugs Automatically - S&P 2020

Created by : Mr Dk.

2020 / 06 / 06 21:19

Nanjing, Jiangsu, China

---

## Abstract and Introduction

本文由 Facebook 发表，主要使用了历史已有的 bug fix 对目前的缺陷代码进行自动修复。Getafix 基于一种层次化的聚类算法，能够归纳出从通用到具体的一系列 fix pattern。另外，为了能够在实际中使用，Getafix 使用了一种能够快速选择候选修复代码的排名技术，使用 code change 的上下文，选择最适合某个 bug 的修复代码。

有别于目前相关工作的两个点：

1. 这个工具的目标是直接产生开发者可以接受的 patch，而不是一些等价但不可接受的 patch
2. 产生 patch 的速度要求尽可能快，能够在开发过程中使用，与静态分析工具类似
3. 产生 patch 的个数尽量地小，尽可能只产生一个

本文的灵感来自于，对于静态分析工具报出的警告，对某个特定类型的 bug 的修复通常是类似的。能否通过对于类似 bug 的修复历史，自动化修复一些目前存在的缺陷？

其中的挑战在于，对于某个特定类型的 bug，可以有多种修复方法。本文的工具可以根据代码的上下文，选择出其中的一种。

1. 由于修复需要在很短的时间内完成，本工具没有时间使用 test suites 去验证效果，完全基于过去的修复来对当前的修复进行评级，根据上下文来判断哪个修复更好；只想开发者提供排名最高的几个 fix
2. 工具使用静态分析工具来判断是否成功修复了 bug，而不是使用 test suites，因此无法保证代码能够通过 test suites
3. 不使用缺陷定位技术，直接使用静态分析工具给出的 bug 位置
4. 目标不是通过验证，而是产生人类可以接收的 patch

## Overview

一个 bug 集合以及相应的 fix 被作为输入来进行训练。然后，Getafix 对一个没有被修复的 bug 进行修复。

静态分析工具带有 bug 的类型与位置信息。首先，识别 bug 被修复之前和被修复之后在 AST 级别上的差异，将这些差异归纳为 fix pattern。Fix pattern 中可能有 **洞** (有点类似于函数参数)，用于对类似的 pattern 进行抽象。将不同抽象等级的 fix pattern 按层次组织，越往上层的 fix pattern 越抽象、越通用。

在学习到 fix pattern 后，将 fix pattern 应用到待修复的代码上，不通过计算复杂的验证步骤，对候选的修复代码进行排名，选择排名最靠前的修复代码返回给使用者。

## Tree Differencer

将修复前后的 code diff 进行 AST 级别的表示。将修复前后的代码解析为 AST 之后，对两个 AST 的结点进行匹配。根据匹配结果，产生了以下四种行为：

* Deletion
* Insertion
* Move
* Update

## Learning Fix Patterns

得到上一步中的大量 AST diff 后，下一步是根据 bug 的类型，将类似的 bug 的 AST diff 进行归纳提炼，得到一个通用化的 fix pattern。为了将几个具体 diff 中的信息进行抽象、通用化，可以将一些具体信息用 **洞** 来替换。

算法不断地将两个具体的 fix pattern 进行合并，得到一个抽象一些的 pattern，不断迭代，直到最终得到一个最抽象的 fix pattern。在这一过程中，相当于形成了一个层次化的 fix pattern 树，越在上层的 fix pattern 越抽象。

这样，对于一个新出现的 bug，匹配可以从最抽象的 fix pattern 开始不断向下搜索，直到找到一个最匹配的 fix pattern。

在 fix pattern 中，除了代码的变化以外，还包括了上下文信息，使得 fix pattern 更加具体，有助于 fix pattern 与新代码进行匹配：

* 代码上下文
* 错误上下文

## Applying and Ranking Fix Patterns

1. 对于给定的代码片段，找到合适的 fix pattern
2. 决定产生的候选 fix 的排名

匹配包含以下几步：

1. 在 bug AST 中找到与 pattern 修复前 AST 匹配的部分
2. 借助 bug AST 将 pattern 中的洞补全
3. 将 bug AST 替换为补全后的 pattern AST

Fix pattern 的排名指标：

* Prevalence score - 覆盖越多人为 fix 的 bug fix 分值越高
* Location score - 距离警告位置的行数
* Specialization score - 更加具体化的 pattern 分值越高

将三个分值相乘。

---

