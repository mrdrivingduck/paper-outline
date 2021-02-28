# Outline

## AVATAR: Fixing Semantic Bugs with Fix Patterns of Static Analysis Violations

Created by : Mr Dk.

2020 / 03 / 25 17:30

Ningbo, Zhejiang, China

---

## Abstract and Introduction

本文提出利用静态分析工具的 fix pattern 作为补丁生成的原料。通过从普通的补丁中挖掘 fix template，自动化地推断出 fix pattern。其中，聚焦于那些开发者修复的针对违反静态分析工具的 bug 的补丁。好处在于：

1. 有工具可以直接用于评估 code change 是否是一个 bug fix
2. 可以对特定的违反规则进行分类，从而对这类 bug 提取一个一致的 fix pattern

本文的贡献：

1. 对于静态分析工具检测到的 bug，从修复这些 bug 的补丁中，可以推断出 fix pattern 并为缺陷自动修复提供原料
2. _AVATAR_ - 基于被确认为 bug fix 的补丁中提取的 fix pattern
3. 与最先进的 APR 工具进行了比较

## Background

### Automated Program Repair with Fix Patterns

基于 fix pattern 的缺陷修复 - 将 code change 抽象为一个 pattern，并应用到具有缺陷的代码上。其中，需要用到缺陷代码的上下文信息 (即需要被处理为 AST)，对给定的 fix pattern 进行匹配。

对于 pattern 的挖掘，有两种策略：

1. 人为设计
2. 自动挖掘

### Static Analysis Violations

静态分析工具能够帮助开发者在编程时检测到一些编程错误。当代码与一些分析规则不兼容时，静态分析工具就会发出警告。由于静态分析工具只能使用部分信息来进行分析，因此可能出现误报 (false positive) 或无关警告 (在运行时不可能发生)。

假如，对于静态分析工具报告的一些 bug，开发者通过补丁修复了它们。那么可以在这些补丁中来挖掘 fix pattern。

* 由于静态分析工具会指明警告的分类，有助于对 bug 进行分类，减少了寻找 bug 及其补丁的人力工作
* 这些 code change 可以通过静态分析工具来进行定位

## Mining Fix Patterns for Static Violations

过程分为以下三步。

### Data Collection

这一步的目标是采集补丁及其相关的静态分析 bug。通过对开源 project 的 commit history 进行分析，保留与静态分析 bug 相关的 code change。在实现上，对于 project 的每一个版本，运行静态分析工具。如果在某一次 commit 中出现了警告，而在后一次 commit 中警告消失，那么其中的 code change 就可以被视为是 bug fix。

另外，还需要检查 code change 是否是真的消除了静态分析工具警告的 bug，还是因为只是巧合。具体方式是，静态分析工具报告的受影响代码行，是否位于 code change 的 diff 中。如果是，则这样的 code change 才被认为一次真正的 bug fix。

### Data Preprocessing

对于这些 code change，即补丁，提取它们 code change 中具体的 __更改动作__ 。在 Java 中，单独的代码行已经不能作为一个语义实体 (一个实体可以跨多行)，因此用 diff 来挖掘 fix pattern 很难。所以将其转换为 AST，利用 AST 的 edit script 来进行 pattern 的挖掘。

```diff
--- a/src/com/google/javascript/jscomp/Compiler.java
+++ b/src/com/google/javascript/jscomp/Compiler.java
@@ -1283,4 +1283,3 @@
    // Check if the sources need to be re-ordered.
    if (options.dependencyOptions.needsManagement() &&
-       !options.skipAllPasses &&
        options.closurePass) {
```

比如，如上的 diff，即删除一个 `if` 中的条件，会被转化为 AST 的 edit script：

```
UPD IfStatement@@‘‘if statement code’’
---UPD InfixExpression@@‘‘infix-expression code’’
------DEL PrefixExpression@@‘‘!options.skipAllPasses’’
------DEL Operator@@‘‘&&’’
```

### Fix Pattern Mining

通过上一步，得到了大量的 edit script。这一步的目标是将 __类似__ 的 edit script 分到一个组中，从而推断这些 edit action 中的共同点。可以通过计算 edit script 之间的距离，也可以使用深度学习来提取特征。最终，利用距离对这些 edit script 进行聚类，聚类最大的那个子集就被视为是一个 pattern。

这种挖掘方法已在相关工作中被证明是有效的。

## Our Approach

_AVATAR_ 的设计细节。

### Fault Localization

_GZoltar_ + _Ochiai_ 的配置。输出的形式是按 __疑似度__ 倒序排序的 code location。

### Fix Pattern Matching

对于缺陷定位结果中的 code location，依次使用给定的 pattern 进行匹配。这些 pattern 都是在相关工作中被挖掘出来的。只选择其中能够改变程序行为的 13 个 fix pattern。

每个 pattern 实际上都是一个 AST edit script。因此需要将缺陷代码的 code location 视为匹配 fix pattern 的上下文。比如，在上面的 edit script 例子中，这个 pattern 首先需要在缺陷代码中匹配到 `if` 语句，然后还要至少匹配到两个子表达式。

### Patch Generation

一旦 pattern 和缺陷代码匹配上。_AVATAR_ 就会应用 edit script 中的修复动作。

### Patch Validation

对于产生的补丁，还要用 test case 来验证这个补丁是否正确。

* Fully-fixing - 产生的补丁通过了所有的 test case
* Partially-fixing - 产生的补丁通过了所有之前通过的 test case，只通过了部分之前失败的 test case

## Assessment

1. _AVATAR_ 体现了通过静态分析工具来挖掘 fix pattern 的有效性
2. 能够从 _defects4J_ 中有效修复语义错误的 bug，并能够修复具有多个缺陷位置的 bug
3. _AVATAR_ 利用了静态分析工具的 fix pattern 中提供的原料，修复了很多的 bug
4. 在质量上和数量上都超过了最先进的缺陷自动修复工具，并修复了最先进的 APR 工具无法修复的 bug

---

