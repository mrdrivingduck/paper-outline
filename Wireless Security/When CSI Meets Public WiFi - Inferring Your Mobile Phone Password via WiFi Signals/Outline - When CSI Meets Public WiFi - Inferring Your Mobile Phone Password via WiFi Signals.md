# Outline

## When CSI Meets Public Wi-Fi - Inferring Your Mobile Phone Password via Wi-Fi Signals - CCS 2016

Created by : Mr Dk.

2019 / 01 / 29 01:41

Ningbo, Zhejiang, China

---

### Introduction

智能手机和平板电脑广泛用于移动支付

与静态设备连接到一个安全的网络不同

移动设备通常接入一个动态的网络环境中

攻击者可以通过直接或非直接的方式进行窃听

* 直接攻击
  * 通过直接观察屏幕和键盘上的输入
* 间接攻击
  * 使用侧信道的方式推断输入

为了能够获取侧信道的信息，已有研究共同假设：

* 外部信号采集设备距离目标设备很近
* 目标设备的传感器被攻破

然而，移动设备上的输入并不总是敏感信息

* 攻击者可能只对 _PIN_ 感兴趣
* 本文将展示如何利用应用情境提高推断用户输入的效率

_WindTalker_ 建立一个免费流氓热点，吸引受害者连接

只要受害者连接到热点上

* 截获流量
* 采集目标设备到热点之间的 _channel state information, CSI_

_WindTalker_ 的三项技术挑战：

* 输入时，手和手指移动对 _CSI_ 产生的影响微乎其微
* 在之前的研究中，需要部署两个额外设备采集 _CSI_，在移动场景中不够灵活
* 秘密推断过程应当在一些敏感时段完成，比如支付时；之前的研究没有实现面向情境的 _CSI_ 采集

本文的主要贡献：

* 提出了一种实用的 _CSI_ 采集方式，使用公共的热点，不需要攻破目标设备或部署额外硬件
* 提出了键盘输入识别算法，利用低通滤波器过滤高频噪声，使用 _PCA_ 进行降维
* 提出了面向情境的 _CSI_ 采集方法，能够识别出在支付宝上输入 _PIN_ 的时间
* 广泛的评估

---

### Background

#### In-band keystroke inference model

在目标设备附近部署 _Wi-Fi_ 热点

当目标设备连接到热点时

热点能够通过数据包的模式监控应用程序的情境

* 通过 _Wi-Fi_ 通信的元数据，热点可以得知何时敏感操作正在进行

同时，热点周期性地发送 _ICMP_ 数据包以获得 _CSI_ 信息

* 利用 _CSI_ 信息推断出用户的输入

_Out-of-band keystroke inference, OKI_ - 之前的研究普遍使用的方式

* 攻击者在目标设备附近部署两个 _Wi-Fi_ 设备
* 目标设备位于两个部署设备中间
* 发送设备不断发送信号，接收设备不断接收信号
* 通过信号的多路径扭曲判断键盘输入

_IKI_ 的优势：

* 不需要两个额外设备的部署
* _OKI_ 模型无法区分敏感操作进行的时间
* _IKI_ 模型使攻击者能够获得未被加密的元数据和 _CSI_ 数据，发起更细粒度的攻击

#### Channel State Information

_WindTalker_ 的基本目标：衡量手和手指的移动对 _Wi-Fi_ 信号的影响，利用 _CSI_ 和特殊的手的动作的相关性来识别 _PIN_

> CSI measures Channel Frequency Response (CFR) in different subcarriers.

输入密码时手指的移动能够产生一个独一无二的模式（_CSI_ 值的时间序列），可被用于键盘输入识别

---

### Motivation

两个导致 _CSI_ 的值发生变化的主要因素：

#### Hand coverage and finger position

_CSI_ 会反映几个多路径信号的碰撞

那么不同的手指位置和手的覆盖将不可避免地引起 _CSI_ 值的变化

#### Finger click

对 _CSI_ 的值有更直接的影响

---

### The Design of WindTalker

#### System Overview

一石二鸟：

* 分析 _Wi-Fi_ 流量，识别出敏感攻击窗口（如输入 _PIN_ 时）
* 一旦攻击窗口被识别，_WindTalker_ 会发动基于 _CSI_ 的键盘输入识别攻击

系统组成部分：

* _Sensitive Input Window Recognition Module_
  * 负责识别敏感输入的时间窗口
* _ICMP Based CSI Acquirement Module_
  * 当目标设备连接到热点时，采集 _CSI_ 数据
* _Data Preprocessing Module_
  * 对 _CSI_ 数据进行预处理，去除噪声，降低维度
* _Keystroke Extraction Module_
  * 自动判断键盘输入波形的起始位置和结束位置
* _Keystroke Inference Module_
  * 对于不同的键盘输入波形，决定对应的键盘输入

#### Sensitive Input Window Recognition Module

捕获受害者所有的数据包，并记录每个 _CSI_ 数据的时间戳

数据包被 _HTTPS_ 加密，但不能加密元数据，比如目标服务器的 _IP_ 地址

_WindTalker_ 构造了一个 _Sensitive IP Pool_

* 当支付宝在进行敏感操作时，数据包将会被发送到某几个固定的 _IP_ 地址上
* 同一个网络内的用户将会被定向到同一个服务器 _IP_ 上去，该 _IP_ 将会维持一段时间

只要观测到存在到 _Sensitive IP Pool_ 的流量，说明正在进行敏感操作

之后开始对 _CSI_ 数据的分析

#### IMCP Based CSI Acquirement Module

##### Collecting CSI Data by Enforcing ICMP Reply

以较高频率周期性发送 _ICMP Echo Request_

不需要目标设备的任何允许

##### Reducing Noise via Directional Antenna

挑战：最小化周围人活动带来的干扰

* 全向天线对每个方向都有相同增益
* 有向天线对每个方向的增益不同

#### Data Preprocessing Module

除去噪声

* 低通滤波，除去高频噪声
* 利用 _PCA_ 降低特征向量维度

#### Keystroke Inference Module

* Keystroke Extraction
* Keystroke Recongnition
* Discrete Wavelet Transform
* Dynamic Time Warping
  * 利用动态规划，计算两个不同长度的输入模型时间序列的距离
  * 一个较小的距离，说明两个序列高度相似
* Classifier Training
  * 计算输入波形和所有按键波形的 _DTW_ 距离
  * 距离越小，输入越可能是对应的按键

---

### Evaluation

#### Classification Accuracy

键盘输入波形之间的差异是否足够用于识别出不同的键盘输入

分类准确率通过交叉验证的准确性进行评估

* 对每 _10_ 个循环的数据集
* 选择一个循环作为测试数据，另外 _9_ 个循环作为训练数据

#### Password Inference

新的衡量指标：_recovery rate with Top N candidates_

* 尝试 _N_ 次后成功恢复出密码的概率

对于每一个输入波形，匹配不同的按键都有一个概率

对于一个 _6_ 位的密码，它的概率是六个按键概率的乘积

* 共有 _1000000_ 种组合
* 按它们的概率降序排列
* 一次成功的密码推断定义为：真实的密码包含在了 _N_ 次尝试的候选中

如果 _N_ 的值较大，那么推断的准确率会大幅上升

#### Impact of Distance and Direction

##### Distance

在真实场景中，目标设备和 _AP_ 的距离不是固定的

但由于 _Wi-Fi_ 发送方（智能手机）和受害者的手指的距离相对稳定，因此不影响密码识别

在不同的场合下，_CSI_ 的形状会发生改变，_WindTalker_ 需要重新训练

##### Direction

不同的方向意味着不同的多路径传播

* 对于右手用户，当 _AP_ 在受害者左边时，_WindTalker_ 的效果更好
  * _WindTalker_ 能更轻易得感知受害者的手指点击动作和手的移动
* _WindTalker_ 当攻击者在受害者身后，无法看到屏幕时，也能够良好工作

---

### Real-world Experiment: Mobile Payment Password Inference Towards Alipay

---

### Discussions

#### Limitations

##### Hardware Limitations

当实验显卡与其它显卡一起工作时容易 _crash_

##### Fixed Typing Gesture

在实验中，受害者以相对固定的姿势触摸屏幕输入密码

手机被放置在相对稳定的环境中

在现实中，不存在这样的情况

##### User Specific Training

不同的人有不同的手指覆盖面积和点击方式

在现实中，没有提前训练的方式

* 但是可以通过在接入免费 _Wi-Fi_ 前，完成一定的测试来获取用户输入习惯进行训练

#### Defending Strategies

* 随机化 _PIN_ 的输入键盘布局
  * 无法根据手指触碰位置判断出输入内容
* 阻止 _CSI_ 数据的采集
* 在 _CSI_ 数据中加入一些随机的噪声
* 检测、阻止高频率的 _ICMP ping_

---

### Related Work

#### Public Free Wi-Fi with Malicious Behaviors

#### Keystroke Inference Methods

* Motion - 利用加速度传感器
* Acoustic Signals - 利用麦克风
* _Camera Based_ - 利用相机
* _Wi-Fi Signal Based_ - 无法识别出敏感操作的窗口

---

### Summary

利用信号的论文看起来总是那么的玄乎

中间那段数据处理实在是看不懂了

机器学习倒是其次了

滤波、降维这些的我是真的理解不能

不过那个 _DTW_ 距离倒是在别的论文里也看到过

可以计算不同长度的时间序列之间的距离

距离用于判断相似性

那么 对于我的那篇要写的论文

可否用这个距离来判断流氓 _AP_ 的转发行为呢？

---

