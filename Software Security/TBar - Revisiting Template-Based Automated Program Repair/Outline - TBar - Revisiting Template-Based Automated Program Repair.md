# Outline

## TBar: Revisiting Template-Based Automated Program Repair - ISSTA 2019

Created by : Mr Dk.

2020 / 03 / 12 🌳 17:46

Ningbo, Zhejiang, China

---

## Abstract and Conclusion

关于 __程序自动修复 (Automated Program Repair, APR)__ 主要分为三步：

1. 缺陷定位 (Fault Localization) - 程序缺陷的位置 (行 / 函数 / 文件)
2. 缺陷检测 - 程序有什么缺陷 (哪种 bug)
3. 缺陷修复 - 自动修复 bug

很多的缺陷检测和修复是通过一种特定的 __修复模式 (fix pattern)__ 来完成的。本文收集了 15 个修复模式，集成在本文提出的工具 _TBar_ 中。并与目前最先进的 APR 工具在 _Defects4J_ 数据集上进行了对比。评估了不同 fix pattern 在修复程序上的效果：

1. 多样性 - 目前最先进的 APR 工具都用了哪些 fix pattern？
2. 修复效果 - 能发现多少 bug？能正确修复多少 bug？
3. 对于缺陷定位偏差的敏感性 - 如果程序缺陷定位不准，会对修复 bug 有影响吗？

研究结论：

1. 性能 - 在缺陷定位完全准确的 (理想) 前提下，_TBar_ 正确修复的 bug 最多，能够成为新的 baseline
2. 对于 fix pattern 的选择 - 大部分的 bug 都只会被一个 pattern 修复，意味着对于 pattern 的正确选择能够避免产生不正确或看起来正确的补丁
3. 补丁生成时，对于原料代码的选择 (?)
4. 缺陷定位的噪声对于修复的性能会有很大影响

## About Fix Patterns

不同的 APR 工具中都包含了其支持的 fix pattern，即用特定的代码模式去识别软件中的缺陷并修复。这些 pattern 的来源途径包含：

1. 人工归纳
2. 挖掘
3. 预定义
4. 统计学特征

本文归纳了一些 APR 工具中提出的 fix pattern，总共 15 个大类，35 个小类。以其中的第一个类为例。第一个 pattern 叫做 _Insert Cast Checker_ - 识别一个 statement 中没有被检查的强制转换：

```java
var = (T) exp;
```

```java
if (exp instanceof T) {
    var = (T) exp;
}
```

这些 fix pattern 的指标包含：

1. 动作 - 更新代码 / 删除代码 / 插入代码 / 移动代码位置
2. 修复粒度 - method / statement / expression
3. Bug 上下文
4. 修改程度 - 有多少 statement 在修复后受到影响

## Experiment

_TBar_ 中使用了 _GZoltar_ (自动执行测试用例) + _Ochiai_ (威胁代码疑似度度量) 框架来进行缺陷代码的定位。在程序的 AST 上，对于每一个疑似结点，应用每一条 fix pattern 进行检测。

如果有一个 fix pattern 匹配到了威胁，就进行自动修复，产生一个对应的补丁。如果这个补丁能够通过所有测试用例，那么这个补丁就被认为是一个 __貌似可信的__ 补丁 (这个补丁不一定真的有用，只不过能通过所有的测试用例罢了；是否真的有用需要人为检查)。一旦找到这样的补丁，_TBar_ 就不会再为这个 bug 继续花时间了。否则就一直搜索到 AST 的所有子结点都被遍历完为止。

用于评估的 benchmark 是 _Defects4J_ 。这个数据集中包含很多有缺陷的 Java 程序，以及相应的开发者提供的补丁。数据集的 395 个 bug 中，有 101 个 bug 已经被现有的工具正确修复，单个工具能修复的最多补丁数为 34 个。

## Assessment

### Assessment 1 - _TBar_ 中 fix pattern 的效果

第一个评估实验用于检验 _TBar_ 中包含的不同 fix pattern 的效果如何。由于是单纯比较 fix pattern 的能力，所以 bug 的位置全部已知 (以免因为 bug 位置的偏差导致 fix pattern 能力的偏差)。

结果 - 能够为 101 个 bug 产生补丁 (貌似可信的补丁)，其中 74 个补丁是真正正确的。虽然结果不错，但接近 79% 的 bug 没能被任何一个 fix pattern 识别并修复。原因可能在于：

1. Fix pattern 不够多
2. 对于产生补丁时的代码选择，有些候选代码可能在别的文件中

另外，86% 被正确修复的 bug 是被单个的 fix pattern 修复的，没有同时被多个 fix pattern 识别。因此谨慎选择合适的 fix pattern 有助于避免产生貌似可信的补丁。在大部分场合下，__单个 fix pattern 已足够用于产生正确补丁__ 。

有几个 fix pattern 一个 bug 都没能识别，原因可能在于：

* 产生补丁的候选代码在别的源文件中
* _Defects4J_ 数据集中不包含这些 fix pattern 所能检测到的 bug

最后，根据 fix pattern 的各项属性不同，其修复能力也不同：

* 通过 __更新代码__ 进行修复的 fix pattern 效果更好
* 对于 expression 层面进行修复的 fix pattern 效果更好

### Assessment 2 - _TBar_ 与目前最先进的 APR 工具的对比

前提：使用了通常场景下的缺陷定位。因此缺陷定位上的偏差可能会对产生补丁的数量和质量都有所影响。_TBar_ 最终为 81 个 bug 产生了补丁，其中 43 个是真正正确的。这比最好的 APR 工具的结果要好，但在精确度上略低。另外， _TBar_ 修复了其它工具从未找到过的三个 bug。

可以看到，在没有准确缺陷定位的前提下，_TBar_ 的效果比上一个 assessment 显著下降。因此，缺陷定位对于 _TBar_ 的性能有重大影响。其中，每个 fix pattern 对于缺陷定位噪声的敏感度各有不同。

## Discussion

### Threats to Validity

* 由于 fix pattern 是在 AST 层面描述的，因此目标语言可以扩展到 Java 以外
* Fix pattern 的选择机制可以更高级一些
* 新的 benchmark

### Limitations

1. 选择 fix pattern 的方式过于简单
2. 只能在相同的源文件中搜索补丁素材
   * 可能因为找不到合适的素材而无法产生补丁
   * 而扩展到别的源文件中搜索会导致搜索空间加大 (maybe 爆炸)

---

## Thoughts

1. 更准确的缺陷定位
2. 更多的 fix pattern (我认为这个更多算是横向工作，工程问题)
3. Fix pattern 的选择方式
4. 补丁素材的搜索

---

