# Outline

## An Effcient Scheme to Detect Evil Twin Rogue Access Point Attack in 802.11 Wi‑Fi Networks

Created by : Mr Dk.

2018 / 11 / 19 0:21

Nanjing, Jiangsu, China

---

### Abstract

_MAC_ 层的 _802.11_ 协议存在一些固有弱点，使其易受攻击。

本论文主要讨论 _Rogue Access Point（流氓接入点）_ 攻击中的 _Evil Twin Attack_。

这种攻击的主要思想是，流氓 _AP_  __克隆__ 一个合法 _AP_ 的 _MAC_ 地址和 _SSID_，并引诱原来要连接到合法 _AP_ 的用户连接到该流氓 _AP_ 上，从而实现对用户通信的劫持。

目前存在的对策：

* 维护一个白名单
* 为 _AP_ 和客户端加入补丁（软件或硬件）
* 基于时间的解决方案
* 修改协议

扩展性差、复杂、误报率高

---

### Introduction

#### Evil Twin Attack 攻击方式

假设空间中存在一个合法 _AP_ 和一个具有相同 _MAC_ 地址和 _SSID_ 的流氓 _AP_

大多数的现代操作系统被配置为：对于具有相同 _MAC_ 地址和 _SSID_ 的 _AP_，连接到信号强度更强的那个 _AP_ 上

如果流氓 _AP_ 的信号强度超过了合法 _AP_，就会引诱客户端接入流氓 _AP_

该攻击容易发生在公共的 _Wi-Fi_ 热点、图书馆、咖啡厅、酒店等地

#### Evil Twin 的联网方式

_Evil Twin_ 必须向接入的客户端提供互联网连接

如果 _Evil Twin_ 不提供互联网连接，那么客户端会断开并寻找其它 _AP_

* 若合法 _AP_ 已经提供互联网连接，_Evil Twin AP_ 可以通过接入合法 _AP_ 来提供互联网连接
  * 客户端接入互联网的 __跳数__ 增加了一跳，会花费更多时间
  * 该时间差可能会被一些基于时间的 _IDS_ 检测出来
* _Evil Twin AP_ 也可以使用私有的 _Internet_ 连接

本论文主要基于后一种联网方式

---

### Background & Motivation

#### AP 与 Client 的四次握手

* _client_ &rarr; _AP_ - _Authentication request_
* _AP_ &rarr; _client_ - _Authentication response_
  * _AP_ 的黑名单中有 _client_ 的 _MAC_ 地址 - 失败
  * _AP_ 过载 - 失败
* _client_ &rarr; _AP_ - _Association request_
* _AP_ &rarr; _client_ - _Association response_

每一个成功连接到 _AP_ 上的客户端都需要经历上述四次握手

#### 802.11 Frame categories

* _Management frame_
* _Control frame_
* _Data frame_

_802.11_ 标准提供 _WEP_、_WPA_、_WPA2_ 等加密机制，__只加密 data frame__

* 四次握手的帧都属于 _management frame_
* _management frame_ 与 _control frame_ 明文传递

#### 攻击过程

* _client_ 向合法 _AP_ 发送 _Authentication request_，_Evil Twin AP_ 处于嗅探状态
* 合法 _AP_ 向 _client_ 发送 _Authentication response_
  * _Evil Twin AP_ 不发送 _Authentication response_，因为有可能会将自身暴露给 _IDS_
  * _Evil Twin AP_ 记录 _client_ 的信息，并准备好向 _client_ 发送 _Association response_
* _client_ 向合法 _AP_ 发送 _Association request_
* 合法 _AP_ 与 _Evil Twin AP_ 都向 _client_ 发送 _Association response_
  * _client_ 会与 _Association response_ 先到达的 _AP_ 建立连接

#### 已有解决方法的局限

* 无线网络监控 - 使用嗅探器
  * 由于 _Evil Twin AP_ 具有与合法 _AP_ 相同的 _MAC_ & _SSID_ - 无效
* 特征提取 & 基于时间的解决方案
  * 如果 _Evil Twin AP_ 没有接入合法 _AP_ 获取互联网连接 - 无效
  * 误报率高

---

### Proposed Scheme : IDS for Evil Twin Attack

#### 攻击者属性假设

* 不发送 _beacon_ 广播包以显示自己的存在
* 不回应 _probe_ 试探
* 不依赖合法 _AP_ 提供互联网访问

可避免多种 _IDS_ 的检测

#### IDS 属性假设

* 只监控合法 _AP_
* 合法 _AP_ 一定会回复客户端请求
* _IDS_ 具有 __嗅探__ 的能力

#### IDS 工作原则

嗅探客户端接受到的 _Association response_ 的个数

* 一个 - 正常
* 两个甚至多个 - 疑似遭受 _Evil Twin Attack_，进行下一步分析

#### Analysis of Association Response Frame

* _Retry bit_
  * 第一次设为 `0`
  * 如果 _frame_ 丢失（没有收到确认信息） - 设为 `1` - 重传
* _Sequence Control Field_
  * 每一帧传送成功后自增 `1`
* _Association ID（AID）_

若是重传帧，则除 _Retry bit_ 由 `0` 变为 `1` 以外，其余字段都应不变

#### Algorithm

算法涵盖了八种情况，覆盖了所有的 _Evil Twin Attack_ 可能

理论识别率：100%

#### Exception

若在接收到 _Association Response_ 后接收到 _deauthentication frame_

则该次检测终止

#### Extension

算法可被扩展到多合法 _AP_ 以及多 _Evil Twin AP_ 的场景中

---

### Experimental Results

在现实中，由于无线信道中存在干扰，来自合法 _AP_ 以及 _Evil Twin AP_ 的 _Association Response_ 可能只能被检测到一个，从而被 _IDS_ 认为没有发生攻击。

但理论的识别率是 100%

因此，实验结果为：

* 检测成功率 - `92%` 至 `100%`
* 检测正确率 - `100%`

---

### Summary

很仔细地读完了

但由于只解决 _Evil Twin Attack_

感觉对 _WIDS_ 的流氓 _AP_ 检测部分没什么用

因为我的需求是该 _AP_ 是否接入内网......

---

