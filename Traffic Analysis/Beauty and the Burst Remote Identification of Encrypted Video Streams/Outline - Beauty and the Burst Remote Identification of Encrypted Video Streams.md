# Outline

## Beauty and the Burst Remote Identification of Encrypted Video Streams -  Usenix Security 2017

Created by : Mr Dk.

2019 / 02 / 09 0:22

Ningbo, Zhejiang, China

---

### Introduction

每个东西都有指纹

_TLS_ 只能加密内容，但不能加密网络特征

已经有一部分研究，试图通过流量分析来识别加密的视频流

但都是基于 __closed-world__ 的假设：

* 攻击者必须提前知道正在传输的视频是否属于一个已知集合

之前的研究假设，攻击者能够直接观察到加密的视频流：

* 通过网络层（恶意的 _Wi-Fi AP_）
* 通过物理层（_Wi-Fi Sniffer_）
* 攻击者的虚拟机和用户的虚拟机位于同一物理机

没有考虑到基于 _WEB_ 或移动设备的攻击者，能够远程在用户浏览器中执行 _JS_ 代码

本文贡献：

1. 分析了加密后的视频流展现出的突发性的流量模式的根源
   - MPEG-DASH 视频流标准，由于使用可变比特率编码，视频分段的大小可变
   - 客户端按段请求视频流内容
   - 加密视频流中的 packet bursts 对应 segment requests，与段大小高度相关
2. 证明了 `20%` 的 _YouTube_ 视频指纹泄露，因为它们的流量模式高度独特
   * 攻击者可以在其自己的网络中获取视频指纹，在目标网络中识别视频流
   * 如果视频流不属于攻击者的已知集合，将不会被错认为一个已知视频
3. 开发了基于 _CNN_ 的视频检测方法
4. 证明了基于流量模式的视频识别不需要直接接触到视频流
   * 攻击者能够远程在另一台设备上执行攻击代码
   * 攻击代码使和视频流共享的链路饱和，通过延时时间估计视频流的流量大小

---

### Information Leak in Video Streams

#### Video streams are bursty

视频流的特征：

* 开始阶段的短时间缓存
* 稳定阶段的交替起伏（__起__ 即为 __burst__）

客户端维持一个目标缓冲区大小

当缓存低于一个目标时，向服务器发送请求

虽然视频流的内容在应用层被分段

但是流的大小和到达时间，分别对应数据包突增的大小和突增的时间间隔

能够被网络内的所有人观测到

如果说观测到的流量特征与应用层的分段相关

那么视频流的内容信息就被泄露了

#### MPEG-DASH standard

DASH 基于展示时间将视频分成段

视频内容在服务器上以段文件的方式存储

当流会话被初始化时，服务器发给客户端一个 manifest，包含段的时间和可用编码

客户端提交请求，获取独立的段

并根据网络条件请求可用的编码

#### DASH standardizes a leak

视频压缩和编码算法利用了一个事实：

不同的视频场景包含不同数量的视觉有效信息

所有流行的视频流技术使用了 variable-bitrate (VBR) 编码：

* 视频编码的比特率根据视频内容进行变化

因此相同时间间隔的视频片段的大小（byte）不同

DASH 视频总是以段大小为块进行下载

当客户端缓存低于目标值时，会提交请求获取新的段

因此，在稳定状态，burst 的起伏与存储的视频段大小有关

而视频段大小是从 VBR 的编码中泄露视频内容的

在视屏场景中，如果场面变化较多，则会用较高的比特率编码

---

### Attack Scenarios

#### On-path network attacker

攻击者接入目标网络，被动嗅探流量

能够直接测量攻击需要的流量

#### Cross-site and cross-device attacker

攻击者不需要直接接入网络

* 首先，攻击者使受害者到服务器之间的网络链路饱和
* 在拥挤的网络中，估计受害者的流量波动，揭示流量模式

特别地，本文关注了能够在受害者浏览器中执行 _JavaScript_ 代码的远程攻击者

* 攻击客户端与视频客户端在同一浏览器的不同 tab 中
* 不同浏览器中
* 同一个局域网的不同机器上

#### Wi-Fi sniffer

将网卡设定为混杂模式，嗅探所有的数据包

可以探测到数据包的方向和大小

并可以通过 _Headers_ 丢弃管理帧

帧重传可能会引入一些噪声，但重传的几率很小

#### Fully remote attacker

在目标网络中没有立足点的远程攻击者

可以使用基于 _JS_ 的拥塞侧信道攻击

#### Shared-machine attacker

如果攻击者能够在接收视频的机器上执行代码

那么就可以通过侧信道的方式估计流量

---

### Overview of the Attack

#### Create detectors

目的：测量流通信，判断流中内容是否属于某个视频文件

使用机器学习的模型作为 detectors

攻击者用自己的电脑接收自己感兴趣的视频流，并捕获流量

同时，也接收其它视频作为 negative

在不同网络中传输的同一个段文件会展现出相同的 burst 模式

* 因此使用在自己网络中采集的数据训练模型
* 在目标网络中采集数据用于识别视频

传输到受害者的段文件和传输到攻击者的段文件必须相同

如果攻击者的客户端和网络支持最高质量的编码

那么他可以通过手动调节至低质量编码，或对网络进行限速

#### Apply detectors

假设攻击者能够知道视频开始播放的大致时间

随后攻击者利用 detectors 采集流量数据，并判断是否属于他要检测的视频

---

### Experiment Setup

### From Leaks to Fingerprints

### Video Identification Using Neural Networks

### Bayesian Detection Rate

### Off-path Attacks

### Limitations

### Related Work

### Mitigations

### Conclusions

---

### Summary

这篇论文已经超出了我的能力。。。

中间一段根本不知道它在说什么

难道这就是顶会论文吗。。。。。。

---

