# Outline

## CC2Vec: Distributed Representations of Code Changes - ICSE 2020

Created by : Mr Dk.

2020 / 04 / 14 16:36

Ningbo, Zhejiang, China

---

## Abstract and Introduction

**补丁 (patches)** 包含了以下两部分：

* Log message (我理解相当于 commit comment)
* Code changes

本文的目标是，如何从一个 patch 中提取出一个向量，用于表示这个 patch。这个向量可用于对项目的历史补丁进行分析等等。

本文提出的 *CC2Vec* 能够以层次化 (words, lines, hunks, files) 的结构表示一个 patch。这种表示法可用于提升其它监督学习任务的效果：

* Log message 生成
* 识别 bug fixing patches
* 补丁的缺陷预测

## Approach

本文提取了每个补丁的 log message 的第一行，可被认为是开发者为补丁提供的语义标签。目标是训练一个预测函数 `f = P → y`，`p` 是补丁，`y` 是补丁对应的 log message 第一行的单词集合。`F` 的目标是最小化预测出被用于形容这个 patch 的单词集合和实际描述这个 patch 的单词集合的差异。

### Preprocessing

由于一个 patch 包含对一个或多个文件的修改，而每个文件中又包含多处增加行和删除行。因此：

* 根据受影响的文件，拆分 code change
* 将删除的行和增加的行进行 token 化，使用 NLP 技术转换为单词序列
* 构造 code vocabulary (所有在 code change 中出现的 token)

### Input Layer

由于 code change 可能涵盖多个文件，而每个文件中又涵盖了多个 hunk。对于 **每一个修改过的文件**，将其删除/增加的代码分别表示为一个三维矩阵：

* 每个文件中 hunk 的个数
* 每个 hunk 中删除/增加的代码行
* 每行中删除/增加的单词

### Feature Extraction Layers

本层目标，为 patch 中的指定文件，产生一个表示其 code change 的向量。

本层的输入是两个矩阵，一个代表删除的代码，一个代表增加的代码。将这两个矩阵输入 *HAN (Hierarchical Attention Network)* 构造出一个代表删除代码的向量，一个代表增加代码的向量。再将这两个向量输入 *比较层 (Comparison Layers)*，得到代表删除代码与增加代码区别的向量。将这些区别向量拼接，得到代表每个文件 code change 的向量。

HAN 中包含了对 word、line、hunk 的编码方式，对一些重要的词语能够体现出重要性。

比较层中包含了五种计算删除代码与增加代码区别的算法。每种算法输入分别代表删除和输入代码的向量，输出一个代表两者区别度的向量。将五种算法输出的五个向量拼接在一起，得到这个文件的区别度向量。五种算法分别为：

* Neural Tensor Network
* Neural Network
* Similarity
* Element-wise Subtraction
* Element-wise Multiplication

### Feature Fusion and Word Prediction Layers

对于一个 patch 中改变的多个文件，将表示每一个文件 code change 的向量拼接，得到代表 patch code change 的向量。将该向量输入当前层，得到与 log message 单词集合中每个单词的匹配概率。

## Experiments

工作目标：构建 code change 的向量化表示，从而用于多个其它工作。

### Task 1: Log Message Generation

开发者不能总是很好地编写高质量的 log message。通过本文的工作，可以自动根据 code change 生成 log message。

目前相关领域的 state-of-the-art 工作的做法是从一个训练集的每个 patch 中提取 code change 的表示，并对一个 new code change 也进行提取，然后根据 new code change 与训练集中的 code change 的余弦相似度，找出训练集中最相近的 k 个 code changes。对这 k 个 code change 与 new code change 的 *BLUE-4 score (用于衡量机器翻译系统与人类翻译的相似程度)*，选择该 score 最高的那个 code change 的 log message 作为 new code change 的 log message。

利用本文的方法，提取每个 patch 的 code change 的向量表示。对于一个 new code change，直接计算其向量表示与训练集中的所有向量的距离，并选择距离最小的那个 code change 的 log message 进行重用。

实验效果得到提升。说明使用本文方法得到的 log message 更贴近于人类翻译。

### Task 2: Bug Fixing Patch Identification

一个二分类问题。给定一个 code change 和 log message，预测出这个 patch 是否是一个 bug fixing patch。

已有的 state-of-the-art 方案也是将 code change 进行矩阵化或向量化表示，然后可以分类得到这个 code change 是否是一个 bug fixing patch。然而，其向量化表示并不关注对一些重要的 word、line、hunk 的突出。如果把其中的向量化机制换成本文中的方法，是否可以取得更好的效果呢？

以目前的 state-of-the-art 作为 baseline 进行对比，效果得到了提升。

### Task 3: Just-in-Time Defect Prediction

也是一个二分类问题。给定一个 patch 的 code change 和 log message，将这个 patch 分类为是一个带有缺陷的 patch 还是没有缺陷的 patch。

目前的 state-of-the-art 的输入是一个给定 patch 的 code change 和 log message，输出 patch 是否带有缺陷的概率值。其中，这种方法忽视了 patch 中删除代码和插入代码的结构，使用 *CNN* 来自动提取 patch 中的结构。

换用本文提出的方法，对 code change 和 log message 提取向量表示并拼接，输入目前 state-of-the-art 的工作。也使效果得到了提升。

## Discussion

这一部分论证了本文的方法中，提到的五个差别函数的作用。为了进行对比实验，对于本文中使用的方法，作者分别阉割了五种方法中的一种，得到了五种配置 (相当于每种配置中只保留了四个差别函数)，再加上完整配置 (五个差别函数)，进行了对比实验。

结果显示，使用完整配置得到的向量表示在上述的三个 task 中都表现最好。另外，对于五种阉割配置的平行比较，可以得到五个差别函数中，效果最好的，和效果最差的。

## Summary

本文的主要工作：

* 采用深度学习的方法，对一个 patch (包含结构化的 code change 和自然语言 log message) 进行向量化的表示
* 并通过三个不同的 task 验证了这种向量化表示的效果

---

