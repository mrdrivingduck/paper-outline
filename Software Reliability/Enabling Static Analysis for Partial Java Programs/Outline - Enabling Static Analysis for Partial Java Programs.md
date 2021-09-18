# Outline

## Enabling Static Analysis for Partial Java Programs

Created by : Mr Dk.

2020 / 12 / 20 13:39

Nanjing, Jiangsu, China

---

这篇文章提出了一种 **带有类型** 的程序源代码 **中间表示形式** (IR)。

核心问题在于，当只能访问到 **部分程序**，比如一个工程中的一个源代码文件，那么如何确定该文件中每一个表达式的数据类型。比如，当表达式中访问某个类的成员变量时，由于无法获得该类的定义，那么理论上就无法获知该表达式的数据类型；或者，如果表达式中调用了类的成员函数，由于无法获得类的定义，理论上也无法获知表达式的数据类型；另外，由于成员函数可能被重载，甚至可能无法确定表达式中调用的是哪一个函数。

本文的核心贡献有三个：

1. 提出了一种能够表示表达式类型的 IR (实际上是一个三元组，`<表达式，关系，类型>`)
2. 一系列用于类型推断的启发式规则
3. 对这些规则的推断准确率进行分析

## 类型推断

类型推断基于一个假设：程序能够被正确地编译通过。如果程序编译出错，那么再做推断也就没多大意思了。作者观察到提交到版本控制系统的代码中，绝大部分应当都是编译通过的。

* `dt(expr) = Type` - 表达式的数据类型
* `t1 <: t2` - `t1` 是 `t2` 的子类
* `t1 >: t2` - `t1` 是 `t2` 的超类
* `t1 ~ t2` - 互为继承关系，或拥有相同的祖先类或后继
* `target(field / method)` - 成员变量或成员函数所属的类

推断出的类型可以被重用、合并。

## *PPA (Partial Program Analysis)* 算法

包含以下三个过程。

### Seed Pass

将源代码 parse 为 AST，后序遍历每一个结点 (因为父结点的类型通常由子结点决定)，并应用类型推断规则。对于无法确定的类型，将 AST 结点类型设置为 `unknown`。

### Type Inference Pass

使用推断出的结果，修改 AST 上的类型。修改后的类型被追加到类型序列或与已有的类型推断合并。

### Method Binding Pass

对于存在二义的函数调用，强制选择其中一种函数声明，并继续推断过程。

最终，PPA 算法产生了一颗带有类型的 AST。AST 中的每个结点包含：

* 类型
* 其声明信息是否可以获得
* 类型是否确定

---
