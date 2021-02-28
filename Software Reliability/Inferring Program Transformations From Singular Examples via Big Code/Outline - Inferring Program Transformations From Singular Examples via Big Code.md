# Outline

## Inferring Program Transformations From Singular Examples via Big Code - ASE 2019

Created by : Mr Dk.

2020 / 04 / 23 15:55

Ningbo, Zhejiang, China

---

## Abstract

如何找到一种合适的 **程序变换** 的表示方式？

已有的方法主要是通过大量程序变换例子的统计信息来进行推断。然而，在实际中，可能并没有这么多的例子用于推断。

本文提出的方法，能够只使用一个例子，就能推断出程序变换的模式。由源代码文件中大量的类似代码，来决定程序变换是否需要被抽象化，以变得更加通用，还是应该保持个性化。

## Introduction

对于程序变换模式的推断，其中最重要的一点：决定其是否应当被 **通用化**。

对于这个例子 `f(a, b) ⇒ f(g(a), b)`，一种可能的 transformation pattern：

* 一个变量
* 变量类型为 integer
* 变量名称为 `a`
* 对这样的变量，用 `g()` 包裹

更加通用化的 pattern：

* 一个变量
* 变量类型为 integer
* 对这样的变量，用 `g()` 包裹

如果考虑上下文信息，使得这个 pattern 变得较为个性化：

* 一个变量
* 变量类型为 integer
* 是 `f()` 的第一个参数

总体来看，使 pattern 变得较为个性化之后，会降低 recall (有些本该被转换的 code 就不会被转换了)；而使 pattern 太通用化，会降低 precision (转换了一些本不该转换的 code)。可见，找到一个合适的 **通用化等级** 对于 pattern 的质量至关重要。

一种典型的方法是使用大量的类似 example，并根据其统计信息，来决定哪一部分应当保留具体信息，哪一部分应当被抽象。但是这种方法需要大量的 example，有时很难有这么多的 example。

另一些方法，通过预定义一些规则 - 哪些部分应当抽象，哪些部分应当具体，来减少需要的 example 的数量。但是基于规则的方法无法适应不同的场景。

本文的方法，只需要通过一个例子，就能推断 transformation pattern。需要利用从一个巨大的代码语料库中提取的统计信息，来指引 code change 的通用化过程。具体地说，计算某个元素出现在语料库中的文件个数，如果该元素出现在了很多文件中，则可能是一个通用的元素，需要被保留在 pattern 中；否则就可能是一个比较个性化的元素，应当被抽象掉。需要解决的三个挑战：

* 合适的 transformation 表示方法
* 表示方法能够灵活地匹配代码
* 被匹配的代码应该能够与 transformation 一致

## Framework of Transformation Inference

### Code Element

是一对 `<id, attrs>`，其中 `id` 是元素 ID，`attrs` 是属性集合，其中每个属性是一对 `<name, value>`。目前实现中考虑的属性：

* AST 结点类型 (statement / variable)
* Content (子树内容的字符串表示)
* Static value type (`String` 或 `int`)

### Code Hypergraph

是一对 `<E, R>`，`E` 是一系列元素，`R` 是一系列边，每个边由一对 `<rname, r>` 组成。边主要用于表示 code element 之间的关系：

* Parent relation
* Data dependency relation
* Ancestor relation

### Element Match

对于 `∀<name, value> ∈ attrs, <name, value> ∈ attrs'`，那么 `<id, attrs>` 与 `<id', attrs'>` 匹配。

### Hypergraph Match

Code element 匹配 + relation 匹配。

也就是说，给定一个 hypergraph，通过移除 code element 和 attribute，可以得到更为抽象的 pattern。

在 element 被匹配上以后，对 element 去应用相应的修改操作。修改操作包含：

* 将 AST 作为结点的第 i 棵子树插入
* 将字符串作为结点的第 i 棵子树插入
* 用 AST 子树替换另一棵 AST 子树
* 用字符串替换为另一棵 AST 子树
* 删除结点的某一棵子树

### Transformation

是一对 `<g, m>`，`g` 是 hypergraph，`m` 是一系列修改动作的集合。

## The *GENPAT* Approach

### Extracting Hypergraphs

* 解析代码，提取 AST nodes
* 提取 AST node 中的类型、内容、关系
* 提取 value type
* 分析 data dependency

### Extracting Modifications

### Inferring Transformations

* Element selection
* Attribute selection
    * 在语料库中，计算 attribute 出现的频率，如果频率大于阈值 (0.5%)，就选用 attribute

### Matching and Applying Transformations

可能存在多种匹配，选择匹配度最高的。

