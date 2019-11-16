# Outline

## Grimoire: Synthesizing Structure while Fuzzing - USENIX Security 2019

Created by : Mr Dk.

2019 / 11 / 16 23:11

Nanjing, Jiangsu, China

---

## 1. Introduction

关于 fuzzing 的发展目标：

* 更少的人为交互
* 较少的预先知识，使软件能够自动化地被测试

比如，如何 fuzzing 处理 __高度结构化数据__ 的程序？

* 解释器
* 编译器
* 基于文本的网络协议

这种程序对输入进行两个阶段的处理：

* 语法分析
* 语义分析

如果通不过语法分析

更容易遭受攻击的语义分析代码根本无法被 fuzz

比如浏览器

* 需要对大量结构化输入进行语法分析
     * XML
     * CSS
     * JavaScript
     * SQL

之前的工作如何解决这个问题？

* 人为提供语法
* 人为提供种子语料库

本文提出的方法：

* 不需要人为干预或预先知识
* 通过 coverage 来自动推断输入的结构属性

不需要预处理，而是直接集成到了 fuzzing 过程中

生成算法分析每个新找到的输入

并试图识别输入中可以被替换或重用的 substring

基于这些信息，通过碎片重组输入

在不需要预先知识的前提下，合成了结构化的输入

* 不需要人为协助
* 不需要格式说明
* 不需要种子语料库

在以下程序上测试了 Grimoire

* 脚本语言解释器
* 编译器
* 汇编器
* 数据库
* 语法分析器
* SMT 结算程序

---

## 2. Challenges in Fuzzing Structured Languages

总体来说，fuzzing 试图在应用的状态空间中定位能够导致错误行为的输入

### 2.1 Blind Fuzzing

不管目标程序的内部状态而进行的 fuzzing

策略：

* generation - 给定一个语法说明
* mutation - 对种子或新发现的测试用例施加随机的变化
     * bit flipping
     * splicing (两个输入重组)
     * repetitions

这种变异称为小规模变异 - 因为只改变输入中的一小部分

缺点：需要大量的语料库，或一个完备的标准说明

### 2.2 Coverage-guided Fuzzing

加入了轻量级的程序 coverage 测量

Fuzzer 使用 coverage 来决定这个输入应当被扔掉

还是被用于扩展语料库

通过揭露新的路径，逐渐搜索程序状态

对于多字节的 magic value

AFL 等 fuzzer 通常会进入挣扎状态 (因为猜不出来)

### 2.3 Hybrid Fuzzing

将 coverage-guided fuzzing 与程序分析技术结合

可以覆盖难以到达的程序区域

### 2.4 Coverage-guided Grammar Fuzzing

接收结构化输入的程序

基于生成 (geneartion) 策略的 fuzzer 使用输入语言的标准来产生合法输入

另外，基于 coverage 的 fuzzer 根据语法来对输入进行变异

这种变异属于大规模变异 - 修改了输入的大部分

根据提供的语法说明，fuzzer 能够将触发新路径的输入进行组合，产生更有意思的输入

两个缺陷：

* 需要人为的工作，提供准确的格式说明
* 如果标准不完整或不准确，fuzzer 缺乏解决这种问题的能力

### 2.5 Grammar Inference

### 2.6 Shortcomings of Existing Approaches

* 需要人为协助
* 需要源代码
* 需要一个准确的环境模型 - OS 等
* 需要一个较好的语料库
* 需要输入的格式说明
* 受限于 parser ???
* 只能提供小范围的变异

---

## 3. Design

本文提出的 GRIMOIRE 是一种全自动的技术

能够在 fuzzing 期间合成目标程序的结构化输入

并提供大范围的变异，可以跨越程序状态

不需要关于输入结构的任何信息

GRIMOIRE 基于识别和重组 fuzzing 过程中触发新的 coverage 的输入碎片

在 coverage-guided fuzzer 的基础上，实现了额外的 fuzzing 进程

* 将 fuzzer 发现的新输入中，被修改或替换后不影响 coverage 的部分用符号 □ 代替
* 一般化 - 将输入切分为可以触发新的 coverage 的块，同时维持了候选的信息 (□)

比如，对于 `if(x>1) then x=3 end`

* 移除 `x=3` 不会影响 coverage
* 从而得到了一般化的输入 - `if(x>1)□then □ end`

在得到大量一般化输入的分片后

使用多种策略将已有的分片、token、字符串自动组合

比如，已知一般化输入 `if(x>1)□then □ end` 和 `□x=□y+□`

* 将前一个输入的每个 □ 分别用后一个输入代替

比如 - 

* `if(x>1)□then □ x= □ y+ □ end`
* `if(x>1)□then □ x= □ y+ □ y+ □ end`
* 将所有 □ 替换为空格 - `if(x>1) then x=y+y+end`

### 3.1 Input Generalization

一般化过程 - 试图识别输入中与 coverage 无关的部分

并对输入进行分片

* 首先，使用一系列规则，获得分片的边界
* 移除一个独立的分片
* 检查这个输入是否还能触发相同的 coverage
* 如果可以，则保留该输入，并将被移除的分片替换为 □

为了尽可能使输入一般化，采取了几种分片策略

* 首先，将输入切分为重叠的 256, 128, 64, 32, 2, 1 字节的块
     * 尽可能早地移除大的无效块
* 接着，对不同的分隔符进行切分
     * `.`, `;`, `,`, `\n`, `\r`, `\t`, `#`, ` `
     * 能够移除注释等无用信息
* 在括号处进行切分 - `()`, `[]`, `{}`, `<>`
     * 从找到左括号的位置开始，先找最远的右括号
     * 试图移除两个括号中间的子串，并检验 coverage 是否发生变化
          * 如果不变，则说明中间的部分与 coverage 无关，可以直接移除
          * 如果发生变化，就找第二远的右括号，依次类推，直到回到左括号

将原始输入和其一般化形式全部保存下来

此外，还将一般化输入从每个 □ 处切分开，将这些 token 保存在集合中

### 3.2 Input Mutation

经过前一步，通过这一步产生较易触发新的 coverage 的输入

* 重组一般化的输入、token、从目标二进制文件数据区中薅出来的字符串

将变异分为三个独立的操作

#### 3.2.1 Input Extension

对一个一般化的输入进行扩展

* 另一个随机选择的一般化输入
* 分片
* token
* 字符串

在前面或后面都可以扩展

比如，`pprint '□'` 和 `□x=□y+□`

* 先对两个输入进行具体化 - `pprint '$$'` 和 `x=y+`
* 然后将两个输入拼接 - `pprint '$$'x=y+` 和 `x=y+pprint '$$'`

#### 3.2.2 Recursive Replacement

选择输入中的任一 □ 进行替换

被替换进来的 □ 也能被之后的步骤替换

随机选择 `n ∈ {2,4,8,16,32,64}`，替换 n 次

比如，将 `pprint '□'` 扩展为 `□pprint '□'□`

* 对于另一个随机输入 `if(x>1)□then□end`
     * 选择其中的一个切片 `□if(x>1)□` 并获得 `□if(x>1)□pprint '□'□`
* 对于另一个随机输入 `□x=□y+□`
     * 获得 `□if(x>1)□pprint '□x=□y+□'□`
* 将所有的 □ 替换为空格，得到 - `if(x>1) pprint ' x= y+ ' `

#### 3.2.3 String Replacement

结构化输入的关键词是很重要的元素

单单改变一个关键字都可能带来完全不同的行为

给定一个输入，找到所有与字典中的关键字匹配的子串，并随机选择一个

* 用一个字典中的随机字符串来替代它
* 将这个子串的所有出现都用相同的字符串代替

这两部的变异版本都会被发送给 fuzzer

替代所有的相同字符串可以产生语义上更正确的输入

例如，对于输入 `if(x>1)pprint 'x=y+'`

* 字段中包含 `if`, `while`, `key`, `pprint`, `eval`, `+`, `=`, `-`
* 可以产生 `while(x>1)pprint 'x=y+'`
* 如果 `x` 也在字典中，替换所有相同字符串可以表示为 - 
     * `if(key>1)pprint 'key=y+'`

---

## 4. Implementation

基于 AFL 和 REDQUEEN

在发现新的输入后，不使用 AFL 的输入最小化算法

而是使用本文提出的一般化算法

---

## 5. Evaluations

* RQ1 - GRIMOIRE 与其它最先进的 bug 寻找工具相比如何？
* RQ2 - 当语法给定时，本文中的方法是否有效？
* RQ3 - 对于需要高度结构化输入的目标，本文的方法能给出什么样的提升？
* RQ4 - 与其它语法推断技术相比表现如何？
* RQ5 - 本文的变异策略对 fuzzing 性能有何影响？
* RQ6 - GRIMOIRE 能够在现实应用中识别新的 bug？

### 5.1 Measurement Setup

### 5.2 State-of-the-Art Bug Finding Tools

### 5.3 Grammar-based Fuzzers

需要给定语法的 fuzzer

显然 GRIMOIRE 的效果不如这些 fuzzer

因为它们不需要推断语法

但是，将 GRIMOIRE 作为这些 fuzzer 的增量补充

可以覆盖更多的路径

因为语法说明是死的，而 GRIMOIRE 的变异可以发现更多隐藏的关系

### 5.4 Grammar Inference Techniques

### 5.5 Mutations Statistic

测量了 GRIMOIRE 不同变异策略上花费的时间

以及每个策略发现了多少个输入

### 5.6 Real-World Bugs

---

## 6. Discussion

本文提出的方法能够在没有任何先验知识和人工干预的前提下

对接收高度结构化输入的程序进行 fuzzing

也可用于支持基于给定语法说明的 fuzzer

不依赖符号执行或污点追踪等方法

因此可以应用于 binary-only 的情况

但是，对于语法上较为复杂的 XML 效果一般

---

