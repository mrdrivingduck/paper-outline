# Outline

## Task Allocation in Spatial Crowdsourcing: Current State and Future Directions - IEEE Internet of Things Journal

Created by : Mr Dk.

2020 / 02 / 05 21:48

Nanjing, Jiangsu, China

---

## 1. Introduction

__众包__ (Crowdsourcing) 是 __crowd__ 和 __outsourcing__ 两个词的组合

定义为，将传统意义上由员工完成的工作外包给未知群体的人完成

随着移动设备的发展，crowdsourcing 向 __Spatial Crowdsourcing, SC__ (空间众包) 发展

SC 的三个要素：

1. 任务
2. 工作者
3. 服务者

任务请求者将任务信息提交给服务者

工作者向服务者请求任务，并将完成后的数据上角服务者

任务请求者从服务者处获得收集到的数据

其中，工作者的工作质量直接影响了任务的效率和质量

核心的挑战是，如何有效选择工作者，同时满足特定的约束

总体上，众包有两种任务分配方式：

1. Worker-selection (WS) - 工作者浏览服务者发布的任务并挑选
2. Server-assignment (SA) - 采集工作者的信息，由服务者向工作者分配合适的任务

后者能最好地利用工作者的资源，提升任务的质量

---

## 2. Characterizing SC Task Allocation

### 2.1 The Conceptual Model

1. 任务 - 任何人可以发布关于任务的请求
2. 工作者 - 任何有能力完成任务的个体
3. 服务者 - 将任务分配给最优集合的工作者

#### 2.1.1 Tasks

根据时间信息分类：

* Urgent task - 需要尽可能快地完成
* Normal task - 不急着立刻完成，会持续一段时间

根据空间信息分类：

* Point task - 在特定地点完成
* Region task - 在一个特定区域内完成
* Complex task - 由多个子任务组成

同时根据时间和空间信息分类：

* Static task
* Dynamic task

__时空上下文__ 一般用于定义 SC 任务

* 空间信息 - 位置 (Point / area)
* 时间信息 - 开始时间 / 结束时间

完成任务的方式：

* 由一群独立个体完成
* 由社区合作完成
* 由分享服务完成

任务可能会涵盖不同的知识域

完成任务后可能会有一定的奖励

#### 2.1.2 Workers

时空上下文

* 时间：可以完成任务的时间段
* 空间：可以完成任务的位置

工作者对于特定知识域的技能

工作者对于任务类型、地点、时间的偏好

工作者正确完成任务的概率

#### 2.1.3 SC Server

SC Server 的主要功能是分配任务，存储数据

服务者的任务分配策略影响了任务的完成质量

通常，服务者会以不同的最优化目标和约束解一个优化问题

* 最大化任务完成质量
* 最小化系统开销

通常来说，参与的人越多，任务完成质量越高

系统开销通常来自于设备和回馈参与者的酬劳

### 2.2 A Generic Framework

#### 2.2.1 Task Publishing

任务请求者将任务放置于任务池中

平台首先选择从任务池中选择一些任务，并进行 __组合__ 或 __拆分__

* 将一些非紧急的任务进行组合，在平台上发布一段时间
* 将一个复杂的任务拆分为很多可以独立完成的子任务

工作者被平台挑选并执行任务

#### 2.2.2 Task Allocating

将 SC tasks 分配给合适的工作者是必须的

服务者根据最优化模型，选出最佳工作者集合

#### 2.2.3 Task Performing

* 一群独立个体
* 群体合作
* 共享服务

---

## 3. Key Challenge and Techniques

### 3.1 Single Task Allocation

SC task 只与某个特定对象的某一特定感知数据有关

通常来说，这种任务可被视为，从所有参与者中选择最优集合参与者完成任务，同时满足多个约束

根据参与者是否需要改变其行为模式：

* Unintentional movement
  * 用户不需要改变日常行为模式
  * 研究者需要仔细分析参与者的历史活动模式
  * 这样工作者有很大几率在和平时一样的活动模式中完成任务
* Intentional movement
  * 用户需要特意移动到某些地点以完成任务
  * 移动时间和距离成为重要考虑因素

根据时间和任务分配模式：

* Offline allocation
  * 在任务开始之前，任务分配策略就已经确定
  * 通常任务的时空信息已知，参与者的未来位置也可以被预测
* Online allocation
  * 任务和参与者动态出现

### 3.2 Multiple Task Allocation

* Homogeneous task
  * 只有不同的任务地点
  * MPFT (More participants, few tasks)
  * FPMT (Few participants, more tasks)
* Heterogeneous task
  * 各式各样的时间、感知要求
  * 根据每个任务的要求，需要考虑更多的因素

### 3.3 Low-Cost Task Allocation

* Piggyback crowdsensing
  * 参与者的移动设备在后台提交数据，与其它应用同时运行
  * 节约能耗
* Compressive crowdsensing
  * 基于采集的数据，推断未感知的数据
* Opportunistic encounter-based allocation
  * 在数据上传过程中，给参与者分配不同的角色
  * 最小化全局数据上传能耗

### 3.4 Quality-Enhanced Task Allocation

工作者完成任务的特点：

* 不确定
* 不可控

因此需要在线对任务完成的质量进行评估

选择能够提交高质量数据的合适工作者

难点在于如何衡量提交数据的合法性

另外，数据的质量与酬劳高度相关

---

## 4. Future Trends and Open Issues

### 4.1 Toward New Forms of SC Tasks

传统的 SC tasks 大多要求用户共享移动设备中感知到的数据

但并非所有任务都这么简单

#### 4.1.1 Object Delivery

让用户在日常生活中顺便递送一些个体 / 物体到特定的位置

比如，顺风车带个人，顺路送个餐

但是送餐对于空间 (从特定的餐厅) 和时间 (到达时间) 都有着严格的要求

同时，送餐意味着单件物品的尺寸小，运输成本相对较高，可能需要同时送多个才能抵消成本

这些都是较为复杂的优化问题

#### 4.1.2 Object Tracking

已有的 SC 主要集中于静态物体的感知

未来可能扩展到对于动态物体的追踪

#### 4.1.3 Towards Complex Tasks

### 4.2 Skill-based Task Allocation

已有的工作者模型假设工作者对于不同的任务有着相同的专业度

而现实中，需要根据个体的技能和任务的要求，选择合适的工作者

#### 4.2.1 Skill-based Task Allocation

检测任务和工作者的 __域__

#### 4.2.2 Data-driven Skill Learning

如何获知工作者的技能信息？

* 人类显式输入较为耗时，且信息不完全
* 使用 AI 技术对用户的历史数据进行分析，并与相应的技能挂钩

### 4.3 Group Recommendation and Collaboration

#### 4.3.1 Group Recommendation

选择 k 个既能满足空间约束，又能满足既能约束的队伍作为推荐队伍

#### 4.3.2 Collaboration among Workers

#### 4.3.3 Collaboration with Online Communities

### 4.4 Task Composition and Decomposition

已有的 SC tasks 分配机制中，任务之间的相关性被忽略

#### 4.4.1 Task Composition

根据空间属性和任务之间的关联，将一些任务绑定，交给工作者

工作者必须完成集合内的所有任务才能获得报酬

#### 4.4.2 Multi-task Partitioning

当任务数量较多时，为了加快任务分配速度

将全局的分配处理为本地分配

#### 4.4.3 Complex Task Decomposition

加强复杂任务的完成度

将任务分解为可以被独立完成的子任务

子任务的结果被组合为复杂任务的最终输出

### 4.5 Privacy-Preserving Task Allocation

暴露地理位置可能会导致个人信息的泄露

需要有机制是参与者可以模糊其信息

### 4.6 Incentive Mechanism

报酬显著影响数据的质量

感知的质量和向工作者支付的报酬是 SC 任务分配的两大主要因素

需要在两者之间找到平衡点

---

## 5. Real-world Deployment and Evaluation

### 5.1 Spatial Crowdsourcing in the Wild

### 5.1.1 Human Participation

现有的工作假设服务者分配任务永远不会被拒绝

实际上，工作者可能会忽略一个被分配的任务

如何最大化工作者的任务接受率？

### 5.1.2 Load Balancing

提升任务的负载分布

最小化任务完成时间，最优化资源利用，保持长期的人员参与

### 5.1.3 Uncertainty and Reliability

人类行为很难预测，可能会带来各种各样的不确定性

### 5.2 Task Allocation in Existing SC Platforms

### 5.3 Large-scale User Study

早期的很多工作基于仿真进行

最近的工作主要采用合成的数据

* 将仿真数据集和真实数据集整合，模拟真实环境的数据

现在也出现了从大量用户完成大量任务的过程中收集的数据

---

