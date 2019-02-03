# Outline

## A Wireless Intrusion Detection System for 802.11 Networks - IEEE WiSPNET 2016

Created by : Mr Dk.

2019 / 02 / 03 19:23

Ningbo, Zhejiang, China

---

### Introduction

_802.11_ 标准的安全问题严重

虽然提出了 _802.11w_

但还是存在大量使用旧标准的老式设备

本文主要讨论 __身份欺骗__ 及其可能导致的攻击

#### Background and Motivation

安全领域的三大支柱：

* _Confidentiality_
* _Integrity_
* _Availability_

_WEP_、_WPA_、_WPA2_ 保证了 _Confidentiality_ 和 _Integrity_

* 主要用于保护 _Data Frames_

_Management Frames_ 和 _Control Frames_ 没有被保护

* 可以轻易使用工具进行伪造
* 攻击者可以进行 _DoS_ 攻击，破坏 _Availability_

典型的提升安全性的方式：

* _Firewall_
  * __Firewall__ 通常集成在运行 _WLAN_ 协议的嵌入式设备的操作系统中，提供基本的过滤功能
* _IDS_
  * 使用受限 - _WLAN_ 结点的资源受限，无法运行重量级的安全机制
* 以上机制大部分集中在 _OSI_ 的第三层以上
* 在无线网络情境中，主要关注 _OSI_ 的最低两层

本文提出了一个开源的 _WIDS_

* 针对运行于 _Infrastructure Mode_ 的 _802.11_ 网络
* 不需要运行于网络中的每一个结点上
* 能够检测两种 _Layer 2_ 攻击
  * _Deauthentication Attack_
  * _Evil Twin Attack_

#### Related Work

* 没有达到成熟的等级，因此没能继续开发
* 开发新的协议，用于认证 _Management Frames_ - 修改了设备驱动，闭源
* 依赖阈值的判断 - _false positives_ 较多，且容易被避开

#### Novelty and Contribution

1. 适合资源受限的无线网络情境
2. _WIDS_ 的开发细节讨论
3. 多种检测两种 _OSI Layer 2_ 攻击的逻辑 - 基于认证和异常
4. 原型实现
5. 真实攻击场景下的 _demo_
6. 评估

---

### The Deauthentication Attack

_Deauthentication Frame_ 是 _Management Frame_ 中的一种子类型

用于结束 _802.11_ 网络中已有的 _authentication_

在解除认证攻击中

攻击者注入伪造的 _deauthentication frame_

当通信双方收到这个帧时

* 停止正在进行的通信
* 重置连接状态

---

### The Evil Twin Attack

攻击者将 _AP_ 设置为与周围合法 _AP_ 使用相同的 _SSID_

发送 _beacon frames_ 来欺骗用户

如果用户连接到了 _Evil Twin AP_ 上

_Evil Twin AP_ 将会成为 _MITM_

攻击成功的关键：_Evil Twin AP_ 要有更高的 _transmit power level_

---

### Proposed Solution: The Wireless IDS

#### Components

* The collection module
  * 运行于 _Monitor Mode_ 的网卡，采集无线通信数据
* The logging module
  * 将嗅探到的数据记录，用于以后的分析
  * 将已经存储到 _File System_ 中的数据包丢弃，防止内存占用过多
* The analysis and detection module
  * 读取存储的无线通信数据
  * 转换为可以理解的形式
  * 使用逻辑进行攻击检测
  * 触发警报

#### Design, Architecture and Deployment

* 使用 _Python_ 实现
* 部署在一个单独的结点上
* 假设运行 _WIDS_ 的结点是安全可信的，且具有足够的计算和能耗资源

---

### Detection of The Deauthentication Attack

#### Indicators

1. __Number of Deauthentication frames (threshold)__ - 识别和记录一定时间内 _Deauthentication Frames_ 的个数，并与 _threshold_ 进行比较（_maximum acceptable deauthentication frames_）
2. __Time span or duration__ - 进行分析的时间窗口大小；一个较大的窗口将导致攻击无法被及时检测
3. __Numbers of duplicates (same src MAC to same dst MAC)__ - 在正常情况下，一段采样时间内，不太可能出现重复的 _deauthentication frames_，否则就表明出现了 _flooding_
4. __Reason for deauthentication__ - _IEEE 802.11_ 标准定义了不同的 _reason code_，区分 _station_ 断开连接的原因；大部分攻击工具使用了一个固定的 _reason code_；在一个时间窗口中，所有 _deauthentication frames_ 使用同一个 _reason code_ 是不正常的
5. __Data frames after deauthentication__ - _Deauthentication Frames_ 是结束通信的通知信息，在此之后应当没有数据帧发送了；然而在攻击场景中，由于 _Deauthentication Frames_ 是被伪造并注入的，因此设备还是会继续发送 _Data Frames_；因此可通过检测在 _Deauthentication Frames_ 之后是否还有 _Data Frames_ 来检测攻击

#### Operation

Algorithm 1

#### Robustness of Detection

如果是一次 _flood attack_ （大部分情况）：

* 无法避开所有的 _indicators_

如果攻击者试图通过修改 _reason code_ 来避开检测，或发送较少的帧避开 _flood_ 检测：

* 无法避开帧注入后依旧存在 _Data Frames_ 的检测机制

---

### Detection of The Evil Twin Attack

#### Indicators

1. __Number of beacon frames__ - _AP_ 需要广播 _beacon frames_ 来宣誓自身的存在；这些帧的数量被记录并被用于和一个阈值进行比较；阈值被设定为正常状况下 _beacon frames_ 平均数量的两倍
2. __SSID__ - 每一个 _beacon frame_ 的 _SSID_ 都会被记录
3. __Power__ - _Beacon Frames_ 中也有 _RSS_ 字段，也会被记录下来
4. __Timestamp__ - _Beacon Frames_ 中包含时间戳，会随着每一个 _beacon frame_ 的传输而增大；然而，在攻击场景中，时间戳通常会被设定为一个常数

#### Operation

首先检查 _beacon frames_ 的数量，判断是否出现了 _beacon frames_ 的 _flooding_

如果出现了 _flooding_，_WIDS_ 检查 _beacon frames_ 中的 _timestamp_ 是否是常数

如果上述没有问题，_WIDS_ 检查信号强度的变化来检测攻击

* 检测是否存在 _AP_ 显著地提升自身的传输功率

#### Robustness of Detection

一个高级的攻击者会使用定制化的攻击工具

绕开基于 _beacon flood_ 和 _constant timestamp_ 的检测

但是无法避开基于传输功率的检测

* 因为更高的传输功率是 _Evil Twin_ 攻击成功的关键

---

### Evaluation of The Proposed WIDS

#### Evaluation using standard security metrics

使用 _confusion matrix analysis_

由 _WIDS_ 对每一个 _802.11_ 帧进行分类

衡量指标：

* True Positive
* True Negitive
* False Positive
* False Negitive

#### Evaluation using benchmark requirements

满足指标数量

---

### Summary

对两种攻击的 _detection accuracy_ 分别为 `89%` 和 `93%`，还行吧

_Evil Twin_ 的检测机制有点过于简单了

_Deauthentication Attack_ 的检测倒是可以在 _WIDS_ 系统里实现

但是目前 _kismet_ 好像已经集成了 _DoS_ 攻击的检测功能

使用 _signal strength_ 的问题在于，_RSSI_ 太不稳定了

所以目前我们的 _WIDS_ 里都没有使用 _RSSI_ 作为检测依据

---

