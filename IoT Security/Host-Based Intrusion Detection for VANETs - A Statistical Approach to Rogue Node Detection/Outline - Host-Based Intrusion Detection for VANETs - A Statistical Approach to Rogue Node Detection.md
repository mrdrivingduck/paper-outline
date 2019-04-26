# Outline

## Host-Based Intrusion Detection for VANETs - A Statistical Approach to Rogue Node Detection - IEEE Trans. Vehicular Technology 2016

Created by : Mr Dk.

2019 / 04 / 26 21:47

Nanjing, Jiangsu, China

---

## Abstract

本文提出了一个针对 Vehicular ad hoc networks (VANETs) 的 IDS

可检测多种攻击

讨论了用于训练 IDS 的 VANETs 模型和实现

之后在不同场合下进行了大量的仿真

此外，对仿真中获得的大量数据进行了图形和统计学上的展示

本文还提出了一个检测流氓结点的算法

---

## Introduction

VANETs 将会极大程度上改变人们的生活

现代工业会在车辆上装备 Wireless access vehicular environment 设备

使其有能力与其它车辆交换安全信息

WAVE 协议基于 802.11p 标准

为 VANETs 提供 Dedicated Short-Range Communication (DSRC) 标准

* vehicle to vehicle (V2V)
* vehicle to infrastructure (with RSUs)

保证共享信息的真实性和可用性很重要

在 ad hoc 网络中

维护和依赖信任或信誉是一个昂贵且复杂的概念

在 VANETs 中，中心化的信任机制因为维护、更新、使用的复杂性而具有争议

现有的认证机制包括：

* 密码学
  * 使用密钥对
  * 计算、存储、时间开销
* 信任

可能遭受密钥窃取或攻陷信任机制等问题

由于 VANETs 的无线、移动特性，以及动态拓扑

不可能采用和有线网络中相同的入侵检测机制

本文提出的 IDS 能够检测不同种类的攻击

并能够通过采取一些必要的措施

最小化对网络带来的破坏

该 IDS 以分布式的方式部署在 VANETs 网络的每个结点上

本文的贡献：

1. 一个使用统计学技术检测异常，识别 RN 的 IDS
2. 采集了大量数据，并用统计学的方法进行分析，使用基于假设检验的决策方法
3. 展现了不同参数的影响
4. 提出的 IDS 不依赖于任何基础设施或昂贵的硬件
5. 防止了广播风暴，并能够在流氓结点高达 40% 时保持网络的功能

---

## Related Work

用户将会用接收到的信息来做一些关乎生命的决定

所以检测假的信息十分重要

已有研究提出使用加密和数字签名来保证消息的完整性和不可抵赖性

如果内部结点转变成了恶意结点

再强的加密也无法抵挡攻击

紧急消息的传播主要通过以下两种方式：

* 多跳
* 广播

恶意用户可能通过散布假的警告信息，为他自己请出道路

或通过散布假的事故信息，造成交通的拥堵

目前主要有两种方法对付伪造信息攻击：

* 基于信任或信誉的模式
  * 基于基础设施
  * 自组织
* 以数据为中心的模式

自组织：根据之前或现在的交互，为另一用户分配一个信任分数

信任分数代表了信誉度

可以投票的方式帮助附近的结点决定该结点是否可信

在快速移动和变化的网络中很难实现

基于信誉度的模式不能用于检测假的紧急消息

* 信任的形成需要一段时间
* 无法防御来自可信结点的错误消息

以数据为中心的模式需要一个模型用于检测和修正错误

* 符合模型的数据被接受，否则被拒绝

---

## Preliminaries

### Authentication and Privacy Preservation

结点应当能够被正确区分，同时隐私受到保护

因此所有结点都应当被 CA 认证

假设所有车辆都已被 CA 认证，获得了合法证书和公私钥

假设所有车辆有足够的密钥，并不断变换密钥以保护隐私

即使改变了密钥，结点的最近消息已经可以被关联到同一结点上

### VANET Model

为了对交通状况建模，需要一个数学模型 - Greenshield 模型

标准参数：

* flow (vehicles per hour)
* density (vehicles per kilometer)

该模型描述速度 `v` 和密度 `k` 的关系呈现负相关

* 当 `k` 为 0 时，车辆在路上自由移动
* 当密度升高至最大时，速度降为 0，发生了堵车
* flow 与 `k` 和 `v` 有关

每个车辆能够根据接收到的消息的 ID 在一定的密度窗口内计算车辆密度

密度窗口可以是车辆前后 500m

这样可以计算出密度 `Density<calc>`

同理可以计算出周围车辆的平均速度 `Speed<AVG>`

车辆使用密度和其它车辆的平均速度计算 `flow`

每个车辆不仅传输其位置和速度，还传输其计算得到的 `flow`

在相同交通状况下的相邻车辆应当计算得到较为接近的 `flow`

如果信息符合模型，则被认为是正确的，否则则是错误的

在这一机制下：

如果发生了紧急情况（事故或急刹车）

事件发生地后面的所有车辆将会刹车

* 因此，flow 值将会减小

降低的 flow 数值将会被传送到后面的其它车辆上

随着信息的传播，之后的车辆将会预先知道前方有拥塞

这是该模型所期待的效果：不需要将警告信息洪泛到整个网络

对于一个恶意攻击者来说：

* 通过降低 flow 值和速度并传输，营造出事故发生的假象

由于只有一个车辆正在传输较低的 flow 值

因此能够被轻易地识别出来

每个车辆传输它的 `Flow<AVG>`，会成为其它车辆的 `Flow<RCVD>`

如果一个车辆接收到的 flow 不符合 VANET 模型，该消息将会被拒收

如果模型能够接受这个数值

进一步比较接收值和自己计算的值，以判断数据是否确实正确

如果数值不符合结点自己计算得到的 flow、speed、density

则该数值会被丢弃，并报告发送者的 ID

`Flow<OWN> = Speed<AVG> × Density<calc>`

`Flow<AVG> = mean(Flow<RCVD>)`

### Message Format

每个车辆会创建自己的 beacon 帧 `m`

包含：`m(Speed<OWN>, Density<calc>, Flow<AVG>)`

每个 beacon 帧 `m` 被 Hash 后使用车辆的私钥签名

在紧急场合下，紧急消息的格式如下：

`Emergency Msg(Type, Flow<AVG>, Speed<OWN>, Density<calc>)`

* `Type` 可以是紧急情况的类型
* 紧急消息不被加密
* 必须一被接收就被快速处理

### Attack Model

#### False Information Attack

流氓结点注入错误数据：

* 恶意目的
* 传感器错误

RN 伪造它们的 speed 和计算得到的 flow 和 density

并通过 beacon 或紧急消息发送出去

伪造前方出现事故的假象

#### Sybil Attack

一个流氓车辆传输不同 ID 的多重信息

伪装成多个车辆

从而通过降低 flow 的数值伪造出拥塞的错误假象

ID 可以伪造，也可以从被攻陷的结点窃取

---

## Intrusion Detection System Overview

基于主机的 IDS 被部署在每一个车辆上，用于检测入侵

为了训练 IDS，需要一个正常条件下的网络模型

从而使对于正常条件的偏差能够被检测出来

车辆向外发送：

* Speed
* Calculated average flow
* Calculated density
* Location

每个车辆还要计算自身的 average flow

### Cooperative Data Collection

每个结点通过向周围结点采集数据来描绘附近的交通状况

因此，每个结点能够计算平均值

在所有条件下，flow 的值将会非常接近

分布在平均值的两个标准差之间

这意味着：

* 通信范围内的所有车辆计算得到非常相近的 `Flow<AVG>`

当足够多的的数据被采集到

应用中心极限定理，接近一个正态分布

建立假设检验，使用 `t-test` 来检测错误值

数据从所有的相邻结点采集

并检验计算值和接收值是否存在明显偏差

* 如果存在明显的偏差，则结点被监视
* 开始采集某些参数
* 当足够的参数样本被采集，进行 `t-test` 检验

如果 `t-test` 给出的结果在接收范围内，则数据被接收

否则被拒绝 被拒绝结点被标亮为流氓结点

攻击被识别为假信息攻击，随后的值将会被结点拒绝

发送信息通知其它结点，汇报 RN 及其攻击信息

### Hypothesis Testing for Data Correctness

假设检验被用于测试两个声明，其中只能有一个声明是正确的

假设检验分配了一个置信区间

使我们能够在一定置信度上接收一个声明

适用于 VANET 模型：

* 结点是良性的，接受数据
* 结点是流氓的，拒绝数据

使用假设检验决定接收到的参数是否该被接受

如果接收到的值在 `99%` 的置信区间内，数据被接受

如果没有足够的证据接受接收到的值，则被拒绝

---

## Performance Evaluation

### Simulation Setup

通过指定速度、类型、行为、车辆数量，生成车辆信息

采集数据的场景：

* 没有事故，没有 RN
* 发生事故，没有 RN
* 没有事故，存在 RN
* 发生事故，存在 RN

仿真中的参数设定：

* 密度 - 通过改变结点被插入仿真的时间来改变结点密度
* beacon rate - 向其它结点发送数据的时间间隔
* RN 的数量 - 评价该模式的性能

### Simulation Results

#### Actual Accident Scenario - No RNs

车辆密度和传输速率作为变量变化

* 车辆密度对 IDS 的工作几乎没有影响 - 所有结点同时收到了攻击信息传输
* 传输速率对 IDS 的工作有重要影响 - 当传输速率较高时，flow 下降最快

#### Normal Traffic - No RNs

和预想的一致，`Flow<AVG>`、`Flow<OWN>`、`Flow<RCVD>` 都很接近

#### No Accident - RNs

IDS 拒绝了 RNs 发送的较低数值，防止了 flow 的降低

#### Accident Scenario - RNs

IDS 拒绝了 RNs 发送的较高 flow

### Evaluation Metrics

#### True Positive

RNs 的检测率：`TP = RNs detected / Total number of RNs`

#### False Positive

良性结点被错误分类为 RN 的比例：

`FP = 1 - HNs identified correctly / Total number of HNs`

#### Overhead

IDS 的工作造成的额外开销

### Effectiveness of Hypothesis Testing

仿真确认了一个事实：

当结点是良性的时，靠得很近的车辆将会有非常接近的 flow 值

---

## Discussion

### False Information Attack Detection

IDS 能够有效检测错误信息攻击

* 只分析数据，不需要考虑信任和信誉度

### Resilience to Sybil Attacks

### Overhead Comparison

### Quick Response of IDS

IDS 能够使结点快速决定是否接受数据

### Countermeasures and Fault Tolerance

IDS 能够容忍一定的错误

### Effective Information Dissemination

传统做法会重复广播紧急信息，导致广播风暴

在提出的模式中，没有信道拥塞，不需要多跳重传

### Limitations of the Proposed IDS

RN 互相配合并逐渐降低或提高相关参数的话

IDS 可能无法检测出来

但是这种做法已经违背了流氓结点的主要目的，不太可能出现

---

## Conclusion and Future Work

IDS 扩展性较强

当 RN 数量较小时表现很好

性能随着 RN 数量的增多而下降

但依旧工作得不错

提出的 IDS 证明了统计学方法决策的有效性

从而不需要使用信任或信誉值机制

IDS 不依赖于基础设施

如果错误的数据和计算数据相差过大，则很容易被检测出来

如果相差不大，则不容易被检测

但 RN 的目的就是通过快速提高或降低数据，达到破坏网络的目的

因此 IDS 能够有效应对

---

## Summary

中间的假设检验和中心极限定理真的看不懂

感觉当时学概率论就是会做题了

但真的没有从形式上理解它

感觉得干点啥弥补一下了

---

