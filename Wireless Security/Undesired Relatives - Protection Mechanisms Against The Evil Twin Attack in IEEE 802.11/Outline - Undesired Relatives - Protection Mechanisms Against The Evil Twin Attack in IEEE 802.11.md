# Outline

## Undesired Relatives - Protection Mechanisms Against The Evil Twin Attack in IEEE 802.11 - Q2SWinet 2014

Created by : Mr Dk.

2019 / 04 / 29 13:52

Nanjing, Jiangsu, China

---

## Abstract

IEEE 802.11 AP 通常使用的标识符

* SSID
* BSSID

通常会被很轻易地伪造

Evil Twin 攻击能够使攻击者截获、采集、替换（甚至是加密的）数据

本文提出了对已有攻击的分类

并提出了检测软件 AP 的新方法

---

## Introduction

本文的贡献：

1. 不同攻击场合的精确定义
2. 提出了针对软件 AP （_aircrack-ng_）的检测方法

---

## Attack Scenarios

Evil Twin 是一个软件或硬件的 802.11 AP

通过克隆合法 AP 的特征伪装自己

Evil Twin 攻击场景分为：

1. Replacement - 合法 AP 被关闭，并被附近的 Evil Twin 取代
2. Coexistence - 合法 AP 和 Evil Twin AP 共存在同一地点，Evil Twin 通过更高的信号强度吸引用户，Evil Twin 的网络来源可能来自自身，也可能来自合法 AP
3. Remote Clone - Evil Twin AP 被设置在另一个地方，如果合法 AP 的连接设置存在，则会自动连接到 Evil Twin
4. Ad hoc Clone - 攻击者监听用户的 probe request，并创建出一个符合 probe request 配置的 Evil Twin

---

## Classification of Countermeasures

### Device Fingerprinting VS Evil Twin Detection

设备指纹是指 AP 尽可能唯一的属性

因此区分设备的指纹显然是一种 Evil Twin 的检测方法

### Single-AP VS Group-of-APs

单 AP 的检测方式将 AP 视为一个独立的设备

一组 AP 包含其余的一些设备

### Client VS Operator

客户端试图检测自身是否连接到了 Evil Twin 上

网络管理员运行系统检测 Evil Twin 是否在其管理范围内

### Active VS Passive Detection

主动检测需要与 AP 进行交互（发包）

* 需要 AP 的配合
* 可能会干扰正常通信

被动检测仅仅是监视流量

### Commodity VS Specialized Hardware

对于一些特定的方法（射频指纹），需要专用的设备

对于其它方法，只需要商用硬件即可（笔记本、智能手机）

### With VS Without Deviation from Standard Protocols by AP

有些方法需要 802.11 协议被改动

### Single-entity VS Group-based

单实体方法能被由单个客户端设备实施

多实体方法需要额外设备的配合

### Software- VS Hardware-based

大部分 Evil Twin AP 都是基于软件

### Ad Hoc VS Pre-gathered Information

大部分方法需要提前收集信息（比如 AP 指纹）

有部分方法不需要提前收集信息

> 总体来说，完美的检测方式应当是能够远程标识 AP 的设备指纹，被动，可以用未被改装的商业硬件完成，不需要任何的协议改动，能够由单个实体实施

---

## Solutions

### Protocol Modifications

客户端的驱动，AP 的固件都需要被更换，可用性不高

### Hardware Fingerprinting

#### Radio Frequency Fingerprinting

识别射频信号特征

虽然可以获得较高的准确率

但测量需要专用的硬件

#### Clock Skew

利用不可避免的物理现象

时钟晶振中存在微小但可观测的偏差

通过从 beacon frame 中提取 TSF (Timing Synchronization Function) timestamp

但在现实中不适用（需要修改驱动）

本文提出了一种轻量级的 clock skew 测量方法

* 独立于指纹

### Non-Hardware Based Identification

第一类方法，用于检测 Evil Twin 是否存在额外的一跳

* 测量到本地 DNS 服务器的 RTT
* Inter-packet Arrival Time
* 注入水印包，在其它信道上检测是否会出现

假设条件苛刻

* Evil Twin 连接到合法 AP 不是伪造 AP 的必备条件
* 攻击者在同一设备上运行多个 SSID 也不是伪造 AP 的必备条件

第二类方法，通过 AP 的行为特征

* 设备如何对人为制造的帧作出反应
* 会干扰正常通信

第三类方法，根据 AP 的环境特征

* `[SSID, RSSI]`
* 对于某个 SSID，查看 RSSI 是否会发生显著的变化

没有方法使用了软件 AP 的特有性质

攻击者一般更倾向于使用软件 AP，以防止被发现

---

## Filling the Gap: Detection of Software APs

为了检测软件 AP 攻击

目标是将运行 _airbase-ng_ 的软件 AP 和硬件 AP 区分开

由于软件 AP 需要模仿硬件 AP 的一些行为

会带来不可避免的延时

更精确得说，软件 AP 无法在 TSF timestamp 上完美模仿硬件 AP

在 802.11 网络中

AP 作为所有关联 STA 的 timing master

AP 和所有的 STA 维护一个 64-bit 的 TSF 计数器

AP 周期性地发送带有 TSF 时间戳的 beacon frame

所有 STA 的 TSF 被 beacon frame 中的 TSF 同步

在 AP 构造 beacon frame 时

在其中的 TSF 字段需要填入发送时刻的 TSF timer 的值

这一过程由优化后的硬件和固件完成

因此 beacon frame 中的 TSF 和 TSF timer 中的值的误差很小

而 _airbase-ng_ 使用一个特定的线程产生 beacon

其中的 TSF 时间由 `gettimeofday()` 系统调用获得

先产生这个时间戳，再写入报文中

因此等报文发送出去时，TSF 时间早已过时（系统处理延时）

### Experimental Setup

从 beacon frame 中提取 TSF 时间戳和对应的接收时间

`offset o = t<TSF> - t<REC>`

从实验结果看出，软件 AP 的 TSF timestamp 的精确性较差

### Detection Method

### Evaluation

最差的硬件 AP 也比最好的软件 AP 精确两倍以上

该衡量标准在接收到小于 100 个 beacon frame 的条件下就能达到稳定

因此该检测能在几秒内完成

---

## Summary

最近想着要开始搞毕设了

但一直想不出用什么新的方法

所以又要开始读论文了

这篇论文是 2014 年的

比较老 但是对已有的工作进行了很好的整理

比如攻击场景、检测方式的比较等

同时也提出了一种检测方法，机制上还是挺简单的

我在想的是 能不能用这个机制

套用新的方法（比如机器学习）做一做

---

