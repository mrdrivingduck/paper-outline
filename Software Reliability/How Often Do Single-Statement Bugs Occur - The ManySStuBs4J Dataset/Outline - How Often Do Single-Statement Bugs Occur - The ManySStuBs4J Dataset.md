# Outline

## How Often Do Single-Statement Bugs Occur? The ManySStuBs4J Dataset

Created by : Mr Dk.

2020 / 03 / 23 22:48

Ningbo, Zhejiang, China

---

## Abstract and Introduction

在程序修复研究领域，对于测试用数据集的要求是 high precision + high enough recall。

* Precision - `P = TP / (TP + FP)` - 被识别出的 bug 中，真正的 bug 有多少
* Recall - `R = TP / (TP + FN) = TP / P` - 真正的 bug 中，被识别出的有多少 (覆盖率)

本文从开源的 Java 工程中，挖掘了大量单语句的 bug fix，并将它们分类到 16 个 bug template 中，成为数据集。由于这些单语句的 bug fix 十分简单，开发者会称这些 bug 为 _simple stupid bugs (SStuBs)_ 。本文提出的数据集包含了一大一小两个数据集。

## Methodology

### Select Appropriate Java Projects

首先要选择 bug 的来源。本文选择 GitHub 上最受欢迎的 100 个 Java Maven 工程，受欢迎程度由 forks + watchers 共同决定。

### Classifying Commits as Bug-Fixing or not

对于每一个工程的不同版本，需要搜索其中是否包含了对 bug 的修复。对 commit message 进行检测，看看其中是否包含了 `error` `bug` `fix` `issue` `mistake` `incorrect`  `fault` `defect` `flaw` `type` 等关键字。这种搜索方法在本数据集中达到了 94% 的准确率，即这些 commit 中的 94% 都确实包含了 bug fix。

### Selecting Bug Fix Commits of Single Statement Changes

由于数据集的目标是那些较为简单的 small bug fix，因此还需要进行进一步筛选。首先过滤掉添加或删除 Java 文件的 commit，再过滤掉在 Java 文件的一个位置修改了多个 statement 的 commit。__修改__ 的定义与 _diff_ 算法类似，即删掉了一些行，又新增了一些行。

如何估算修改是否跨越了多个 statement 呢？计算了每个 Java 文件的 diff，对于其中每一个修改 block，计算有几个 statement 被修改了。这种方法能够让数据集中留下那些 __跨越多行__ 的 __single statement__ 。

此外，忽略了注释、空白行、格式变化等影响。

### Creating Abstract Syntax Trees

对修复前和修复后的代码生成 AST，并同时进行深度优先遍历，找到第一个不相同的 AST 结点。

### Filtering out Clear Refactoring

有些 commit 可能不是为了修复 bug，而只是重构。也就是说，可能会有 false positive。数据集中将不会包含重构行。

### SStuB Patterns

与 bug 匹配的模式：

* Change Identifier User - 表达式中的标识符被换成了相同类型的另一个标识符 (可能是由于复制粘贴导致的)
* Change Numeric Literal - 一个数值是否被替换为另一个数值
* Change Boolean Literal - 一个 true 是否被替换为 false，反之亦然
* Change Modifier - 变量、函数、类的修饰符是否改变
* Wrong Function Name - 是否调错了相似的函数名
* Same Function More Args - 对于有重载版本的函数，是否调用了更多参数版本的函数
* Same Function Less Args - 对于有重载版本的函数，是否调用了更少参数版本的函数
* Same Function Change Caller - 对于某个函数，调用对象换成了相同类型的另一个
* Same Function Swap Args - 调用函数的两个参数是否交换
* Change Binary Operator - 二元操作符是否被替换，比如比较大小的操作符
* Change Unary Operator - 一元操作符是否被替换，比如布尔表达式中的 `!` 操作符
* Change Operand - 二元操作符中的一个操作是否错误
* More Specific If - `if` 表达式是否增加了 `&&` 条件
* Less Specific If - `if` 表达式是否增加了 `||` 条件
* Missing Throws Exception - 在函数声明中，是否加入了 `throws` 关键字
* Delete Throws Exception - 在函数声明中，是否删除了 `throws` 关键字

### SStuB Pattern Matching

对于修复前版本和修复后版本的每一对 AST，依次匹配上面的 pattern，将匹配上的 bug 加入 _SStuBs_ 数据集中。

## Research Questions

数据集中的 bug 在代码中的出现频率还是很高的。另外，使用了静态分析工具 _SpotBugs_ 对数据集进行了分析 - 只能定位 12% 的 bug，如果开启警告，会出现超过 200,000,000 个 bug，包含 low-confidence 的。对于开发者来说，需要在大量警告中定位一个 single bug。

---

