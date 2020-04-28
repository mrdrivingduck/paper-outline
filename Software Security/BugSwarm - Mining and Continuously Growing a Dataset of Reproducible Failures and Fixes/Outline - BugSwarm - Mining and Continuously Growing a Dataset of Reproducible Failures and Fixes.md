# Outline

## BugSwarm: Mining and Continuously Growing a Dataset of Reproducible Failures and Fixes - ICSE 2019

Created by : Mr Dk.

2020 / 04 / 28 17:57

Ningbo, Zhejiang, China

---

## Abstract and Introduction

如何建立一个 **灵活的、多样的、真实的、持续增长的** 缺陷数据集，并且缺陷版本和修复版本是可重现的。本文提出的 *BUGSWARM* 工具集产生了 3091 个 fail-pass 对，并且全部位于可重现的容器中。

对于 fail-pass 数据基的几个要求：

* 规模 - 需要有足够多的数据
* 多样 - 项目成熟性、语言、缺陷等级、年龄等
* 真实 - 需要由真实程序员修复的 bug
* 持续 - 需要能够持续更新
* 重现 - 需要能够稳定地复现问题

目前已有的一些数据集 (比如 *Defects4J* 都是人工维护的)。本文致力于自动化这一过程。

两个关键技术起到了自动化这一过程的作用：

* 基于容器的虚拟化技术 - 简化了处理工程的复杂依赖的问题
* 脚本化的 **持续集成 (CI)** 服务 - 自动化构建和测试流程

## Background and Motivation

本文工具基于 *TRAVIS-CI*。这是一个位于云端的 CI 服务，与 GitHub 相集成。它能够自动构建并测试 GitHub 中的 PUSH 或 PR。要使用这个功能，需要在工程仓库中配置 `.travis.yml`，指明工程被测试的环境。在 PUSH 或 PR 事件的触发下，*TRAVIS-CI* 会根据配置自动进行环境初始化和测试，并记录每次测试的结果并产生 build log，记录测试失败的 test case。

本文工具提取的信息：

1. 测试失败的程序版本
2. 紧接着的、修复的程序版本
3. 两个版本之间的 diff
4. 失败 test cases 的名字
5. 构建配置的完整描述 (用于重现)

通过这些信息，来自动化创建一个缺陷数据集。

---

