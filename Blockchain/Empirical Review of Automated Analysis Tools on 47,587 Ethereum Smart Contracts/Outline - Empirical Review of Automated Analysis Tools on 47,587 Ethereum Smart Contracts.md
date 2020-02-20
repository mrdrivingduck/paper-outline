# Outline

## Empirical Review of Automated Analysis Tools on 47,587 Ethereum Smart Contracts

Created by : Mr Dk.

2020 / 02 / 20 20:59

Ningbo, Zhejiang, China

---

## Abstract

本文评估了 9 个最先进的自动分析工具，并使用了两个新的数据集：

1. 69 个已被标记为存在漏洞的 smart contract
2. 47518 个有 Solidity 源代码的 smart contract

本文提出了新的可扩展执行框架 _SmartBugs_，用于比较多个分析工具对以太坊 smart contract 的分析。

---

## 1. Introduction

Smart contract 中的漏洞很难比较和重现。虽然有些分析工具是开源的，而数据集却不是。

本文提出了两个新的数据集：

* 一个数据集是 69 个已被人为标记为带有威胁的 smart contract，可被用于评估分析工具的准确性
* 一个数据集是所有可获得 Solidity 源代码的 smart contract

作者使用了 9 个最先进的自动分析工具应用在这两个数据集上，并对结果进行了分析。

本文提出了一个易用、可扩展的执行框架 _SmartBugs_，用于在相同的环境中执行这些工具。

本文的主要贡献：

1. 两个新的数据集
2. 执行框架
3. 9 个工具对 47587 个 smart contract 的分析

---

## 2. Study Design

### 2.1 Research Questions

1. 最先进的分析工具检测漏洞已知的 smart contract 有多高的准确率？
2. 目前，在以太坊区块链中，有多少 smart contract 的威胁？
3. 工具分析 smart contract 需要多少时间？

### 2.2 Subject Tools

最初选用了 35 个工具，但无法全部都使用。只保留符合以下条件的工具：

1. 工具是公开的，且具有命令行 UI (灵活性)
2. 以 Solidity 作为输入
3. 启动分析只需要 contract 的源代码
4. 工具用于分析 contract 中的漏洞

最终，只剩下了 9 个工具。

* _HoneyBadger_
  * 用于寻找蜜罐 - 蜜罐的开发人员将蜜罐设计得看起来有漏洞很容易被攻击，其实没有漏洞
  * 基于符号执行
* _Maian_
  * 寻找可以从任意地址自毁的 contract
  * 或没有向外支付功能的 contract
  * 可以在 private blockchain 中进行动态分析
* _Manticore_
  * 符号执行
  * 寻找 EVM 字节码中的可重入威胁，以及自毁操作
* _Mythril_
  * 符号执行、污点追踪、控制流检查
* _Osiris_
  * 检测 integer bugs
* _Oyente_
  * 对 EVM bytecode 进行符号执行
* _Sucurify_
  * 静态分析 EVM bytecode
* _Slither_
  * 静态分析框架
  * 将 Solidity contract 转换为中间表示，并进行数据流分析和污点追踪
* _Smartcheck_
  * 静态分析

### 2.3 Datasets of Smart Contracts

第一个数据集包含了 69 个有漏洞的 smart contract - 要么来自已经被标记为有漏洞的 smart contract，要么是为了阐述一个漏洞而被故意创建的。这个数据集可被用于评估 smart contract 分析工具的有效性。

第二个数据集包含了从以太坊区块链中提取的 47518 个 smart contract - 其中是否存在漏洞未知。

#### 2.3.1 A Dataset of 69 Vulnerable Smart Contracts

根据 DASP 的分类规则，将漏洞分为 10 类。

作者手动标记 smart contract 中出现漏洞的行数，并将每个 contract 归类到十个类之一。Smart contract 的来源：

1. GitHub (80%)
2. 分析 contract 的 blog
3. 以太网网络

#### 2.3.2 47518 Contracts from the Ethereum Blockchain

从以太坊 blockchain 上采集尽可能多的 smart contract。采集下来后，由于很多是重复的，因此将 contract 的 Solidity 源代码去掉空格和 Tab 后进行 MD5 校验，校验相同的就是重复的。经过去重后，得到了 47518 个独一无二的 contract。

### 2.4 The Execution Framework: _SmartBugs_

特性：

1. 插件系统，可以轻易加入新的分析工具
2. 工具并行执行，减少执行时间
3. 规范化的输出系统

### 2.5 Data Collection and Analysis

---

## 3. Results

### 3.1 Precision of the Analysis Tools (RQ1)

比较 9 个工具检测 69 个 contract 的能力。

9 个工具都无法检测的漏洞有两类：

* Bad Randomness
* Short Addresses

一个漏洞被识别定义为：分析工具在正确位置 (行数) 检测到了正确类型的漏洞。

* 哪几类漏洞的识别率最高
* 哪几类漏洞的识别率不高
* 哪个工具的识别准确率最高
* 哪个工具能识别最多种类的漏洞

### 3.2 Vulnerabilities in Production Smart Contract (RQ2)

对于分析这些 smart contract，没有漏洞信息的先验知识。

总体来看，44589 (93%) 的 smart contract 至少被 9 个工具之一标出了至少 1 处漏洞。

将多个工具对同一个漏洞的共识作为判断依据显然更为准确。

### 3.3 Execution Time of the Analysis Tools (RQ3)

执行工具的时间与工具本身有关。实验记录了每个工具执行每个 contract 的时间 (最多 30 min)。

有一些对 EVM bytecode 进行分析的工具需要额外的 Solidity 编译的时间。

有些工具执行很慢是因为难以并行；最慢的工具准确率却不是最高的。

---

## 4. Discussion

### 4.1 Practical Implications and Challenges

未来挑战：

1. 分析工具的分析质量 - 准确率？误报率？
2. 解决问题的视角 - 组合工具？动静结合？
3. 开发过程 - 将工具集成到开发过程中？IDE？
4. 分类 - 基于 DASP10 的分类方法？新的攻击类型？

### 4.2 Threats to Validity

_SmartBugs_ 框架本身可能出现漏洞。尽量避免。

---

