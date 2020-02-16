# Outline

## EVMFuzzer: Detect EVM Vulnerabilities via Fuzz Testing - FSE 2019

Created by : Mr Dk.

2020 / 02 / 16 19:16

Ningbo, Zhejiang, China

---

## Abstract

__Ethereum Virtual Machine (EVM)__ 是 smart contract 的运行时环境。现在有很多技术用于持续检验 smart contract 的合法性，而对于 EVM 的测试具有较大挑战。原因在于特殊的输入格式和缺少先验知识。

本文提出检测 EVM 中漏洞的工具。核心思想是不断产生 seed contract 并将其输入目标 EVM 和对比 EVM 中。根据执行结果的不一致性，找到 EVM 中的漏洞。给定目标 EVM 及其 API，工具通过预定义的 mutator 产生 seed contract，并使用动态优先级调度算法指引对 seed constract 的选择，以最大化 EVM 执行的不一致性。

在实验中，使用不同 EVM 作为目标 EVM 进行交叉验证。找到了 5 个 CVE。

---

## 1. Introduction

Ethereum 可被看作是一个基于 transaction 的状态机。EVM 通常被称为以太坊的操作系统。目前，EVM 有超过十种不同编程语言的实现，而缺乏成熟的测试工具。

本文提出的 _EVMFuzzer_ 是一个自动化的差别测试工具。_EVMFuzzer_ 使用 __操作码序列__ 和 __gas 使用量__ 作为每个 contract 在不同 EVM 上执行差异的衡量指标。集成了目前广泛使用的几个 EVM 实现作为基准 EVM，为目标 EVM 和基准 EVM 提供了一个统一的运行环境。

_EVMFuzzer_ 的生成模块能不断产生是差异衡量指标变大的 seed contract，并试图得到那些会在 EVM 上得到不同执行输出的 contract。作者从现实世界中采集了 36295 个 smart contract，通过 fuzzing，其中 1596 个 contract 在不同的 EVM 得到了不同的输出。找到了 5 个 CVE。

---

## 2. Related Work

* Fuzzing technique
* Differential Testing - 测试功能类似的程序的不同实现之间的差异
* Smart contract validation

---

## 3. _EVMFuzzer_ Design

_EVMFuzzer_ 的主要目的是不断提供变异出的 smart contract 给目标 EVM 平台，包括目标 EVM 和基准 EVM。然后观测这些 EVM 的输出是否一致 - 如果不一致，则说明 EVM 存在 bug。

### 3.1 Seed Contract Generation

可被看作是 test case generator。Seed contract 被存放在种子池中，_EVMFuzzer_ 使用动态优先级来对这些种子进行排名 - 排名第一的 contract 将会被用于下一轮变异。在选中用于变异的 contract 后，_EVMFuzzer_ 使用 8 种预定义的 mutator 和组合策略进行变异，目标是使产生的 contract 在不同 EVM 上执行时，能够产生尽可能大的差异 (衡量指标是之前定义的操作码序列和 gas 使用量)，从而使 EVM 对同一 contract 的输出不同。

#### 3.1.1 Seed Mutation

8 个 mutator 的变异层次不同：

* word-level
* character-level
* statement-level

基于反馈得到的 EVM 差异衡量指标，维护一个优先级队列。如果衡量指标显示差异性增强，那么就将对应的 mutator ID 在队列中向前 push；否则队列不更新。

#### 3.1.2 Seed Prioritization and Selection

所有 seed contract 都被存储在 seed contract pool 中。根据每个 contract 的优先级，决定下一轮变异使用哪个 contract。通常来说，造成在不同 EVM 上运行的差异越大的 contract，越应该在下一轮被选中用于变异。同时，其它候选 contract 也应当有一定几率被选中。

因此，_EVMFuzzer_ 使用动态优先级维护了一个候选队列。对于每一个 contract，会有一个初始优先级，随着等待时间的增长，其优先级也会发生改变。从而使每个 seed 都有机会被选中。

### 3.2 Unified EVM Execution

为各个 EVM 提供一个统一的运行环境。当接收到 contract 后，将其编译为 EVM 字节码。_EVMFuzzer_ 自动运行所有的 EVM，根据差异衡量指标，计算不同 EVM 实现的运行差异，并比较 EVM 的运行输出结果。根据 contract 所带来的运行差异，决定要不要把这个 contract 放到 seed contract pool 中保留下来。

---

## 4. Using _EVMFuzzer_

### 4.1 Tool Implementation

_EVMFuzzer_ 集成了四个常用的 EVM 实现作为基准 EVM。

### 4.2 Running Example

将目标 EVM 放在指定目录下，输入其 API，设置 fuzzing 时间，启动程序。

当 fuzzing 结束时，_EVMFuzzer_ 会产生一个报告，从三个维度评估目标 EVM：

* 代码实现完成度
* Gas 计算的准确度
* 执行路径规划的合理度

---

## 5. Preliminary Evaluation

四个 EVM 作交叉验证：一个 EVM 作为目标 EVM，另外三个作为基准 EVM。初始的 seed contract 为 36295 个真实的 contract。

### 5.1 Inconsistency among EVMs

33424 个执行几乎相同操作码序列的 contract 被用于进行 gasUsed 比较 - 这样避免了执行不同的操作码序列而导致的 gas 用量不同。

实验发现，几乎每个平台的 gasUsed 与其它平台相比都有 50% 的不同。

总共有 1275 个 seed contract 在四个 EVM 上执行得到了相同的输出，但是操作码序列的长度不同。

实验证明了用于衡量不同 EVM 差异性的指标 - 操作码序列和 gasUsed - 是有意义的。

### 5.2 Vulnerabilities Detected by _EVMFuzzer_

在发现了上千个不一致的输出结果后，作者通过人工分析来得出 root cause。通过 review EVM 的源代码，找到了 5 个 CVE。

---

