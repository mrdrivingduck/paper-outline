# Outline

## Client-Side Evil Twin Attacks Detection Using Statistical Characteristics of 802.11 Data Frames - IEICE 2018

Created by : Mr Dk.

2019 / 01 / 02 17:33

Nanjing, Jiangsu, China

---

### Introduction

_WLAN_ 的优势：

* 灵活性
* 机动性
* 可扩展性
* 易安装性

_Evil Twin_ - 需要两张网卡才能攻击

* 一张网卡与合法 _AP_ 关联
* 另一张网卡伪装成合法 _AP_ 引诱客户端连接

_802.11_ 标准默认会连接到相同 _SSID_ 中信号最强的 _AP_ 上

因此攻击者通常先对网络流量进行嗅探，并选择目标施行 __中间人攻击__

攻击方式简单：

* 在一台笔记本上，使用特定的软件就可以配置出一个软 _AP_
* 可以以 __被动方式__ 提升信号强度，迫使受害者连接到伪造 _AP_ 上
  * 离受害者更近一些
  * 将天线对准受害者
* 敏感信息泄露
* _DNS_ 劫持 - 发送钓鱼网页

本文的贡献：提出一种基于客户端的 _ETA_ 检测方案

1. 不需要任何白名单
2. 被动方式，不需要连接到任何 _AP_ 上
3. 实时检测
4. 不管 _AP_ 是否是开放的，本文中的方法都实用
5. 可以找出 _Evil Twin_ 使用的两个 _MAC_ 地址

---

### Related Works

流氓 _AP_ 的检测总体上可被分为两类

#### Admin-Side Solutions

网络管理员通常会在需要保护的地区部署 _sensors_

* 花费代价较高
* 攻击者可通过关闭 _AP_、降低信号强度、使用不标准的协议和频率避开检测

在白名单中记录 _AP_ 的网关信息

* 如果 _AP_ 的网关与白名单不符，则属于流氓 _AP_
* 无法提供实时检测

总体来说，基于网络管理员的检测主要依赖一个合法列表

#### Client-Side Solutions

基于时间的解决方案

* 计算客户端到 _DNS_ 服务器之间的 _round trip time (RTT)_
* _Evil Twin AP_ 将会有不可避免的延时
* 造成延时的因素有很多 - 冲突、拥塞、碰撞 - 在网络流量较大时误报率高

基于时钟偏差

* 是一种不可避免的物理现象
* 提取 _beacon frame_ 中的 _TSF_ 时间戳
* 比较记录在数据库中的时钟偏差
* 需要提前在数据库中存入大量授权 _AP_ 的样本

修改协议

* 不实用，需要大范围修改已有的驱动和固件

---

### Problem Statement and Principle

目标：__实时__、__独立__ 地检测 _ETA_，不需要任何网络管理员的协助

假设：

* 合法 _AP_ 和 _Evil Twin AP_ 共存在空间中
* 两者使用相同的网关
* _Evil Twin AP_ 作为合法 _AP_ 和客户端的 __中间人__

在理想情况下（_100% RSSI_，没有无线传输延时）：

* 从合法 _AP_ 发送至 _Evil Twin AP_ 的 _effective data frames (EDFs)_ 与 _Evil Twin AP_ 发送至客户端的 _EDF_ 在每一秒都是相同的
* 在某一个瞬间，两个信道中的 _EDFs_ 在 __数量__ 和 __趋势__ 上都是相似的

基于此，本文通过识别 __转发行为__ 来检测 _ETA_

* 监控、收集目标 _AP_ 的数据流，获取目标 _AP_ 与每个客户端之间的数据特征
* 过滤没用的数据帧
* 剩下的 _EDF_ 用于检测是否存在转发行为

_EDF_ 指的是从 _Internet_ 传输而来的 _802.11_ 非重传帧

* 不包括相同子网中的内部数据传输帧（比如子网中设备的互相 _ping_）

---

### The Proposed Framework

#### System Work Flow

要点：在目标 _AP_ 与用户之间是否存在疑似的转发行为

1. 在一段时间内监控目标网络（相同 _SSID_）
2. 过滤子网通信、控制帧、管理帧、重传数据帧
3. 从目标 _AP_ 到每一个客户端的 _EDF_ 被记录
4. 计算 _EDF_ 之间的 __皮尔森相关系数__，判断是否超过一定的阈值，从而判断是否出现 _ETA_
5. 根据 _Evil Twin AP_ 的 _MAC Address_ 和 _RSSI_，定位 _Evil Twin AP_

#### Main Steps of Proposed Framework

##### Passive WLAN Frames Monitoring Stage

从 _beacon frames_ 和 _probe response frames_ 中获取基本信息

* _SSID_
* _MAC Address_
* _Channel numbers_

##### Data Frames Filtering and Statistics Stage

过滤所有的控制帧、管理帧、重传帧、子网通信，得到 _EDFs_

* 对于一个 _AP_ `APi`，存储一个 `dict`
  * `key` 为 _User_ `Uj` 的 _MAC Address_
  * `value` 为 `Uj` 的数据帧数组 `Di = [d1, d2, ..., dn]`
  * 即 `APi[Uj] = [d1, d2, ..., dn]`
* 每个 _AP_ 都有一个 `SumAPi`，即所有 `Uj` 的叠加

##### Correlation Coefficient Calculation

计算 `SAP1` 和每一个 `AP2[Uj]` 的皮尔森相关系数

计算 `SAP2` 和每一个 `AP1[Uj]` 的皮尔森相关系数

* 比较多个 _EDF_ 数组中的异常相似度
* 不计算不活跃用户的相关系数
  * 提升算法效率，减少不必要的计算和比较

使用皮尔森相关系数的好处：

* 将相似性量化到了 `[-1, 1]` 之间，更为直观
* 当数据由于无线网络质量而不标准时，准确率上升（？）

##### Evil Twins Detection and Location

如果 `C(SAPi, APk[Uj])` 超过了阈值

那么 `SAPi` 和 `Uj` 就是 _Evil Twin AP_ 使用的 _MAC_ 地址

基于 _MAC_ 地址和信号强度定位 _Evil Twin AP_

---

### Evaluation

使用两个网卡，同时监听两个信道（保证不漏包）

使用了经验主义的阈值

* 对于 _RSSI_ 较强的情况，由于丢包较少，能收到足够的 _EDF_ 用于检测
* 对于 _RSSI_ 较弱的情况，由于 _EDF_ 中的缺失值增多，检测率有所下降

然而，如果 _Evil Twin AP_ 的 _RSSI_ 比合法 _AP_ 弱，那么对于客户端来说就失去了连接它的兴趣

造成 _ETA_ 攻击失败

---

### Discussion and Future Work

#### Analysis

* 同时适用于开放/加密的网络
* 不需要很大的 _fingerprint_ 数据库
* 被动检测，不会被攻击者察觉

#### Discussion

攻击者可能采用 __占据带宽__ 的方式降低转发相关性

但是将导致网速下降

用户会自发切换网络

导致 _ETA_ 失败

* 施行 _ETA_ 的设备是 _Laptop_，而不是专用设备
* 如果攻击设备使用更先进的专用设备，传输速率更高（_802.11ac_），那么将能避开检测

#### Limitation

* 必须至少有一个用户关联在目标 _AP_ 上
* 如果用户只连接 _AP_ 但不使用网络，则无法检测
* 必须使用多个网卡

#### Future Work

* 使用单网卡进行检测
* 支持除 _Linux_ 之外的更多 _OS_

---

### Summary

和要写的论文思路很类似

不过本文的假设还是基于 _MAC_ 地址不同而 _SSID_ 相同的 _ETA_

而我考虑检测的是基于 _MAC_ 地址也相同的 _ETA_

使用 _802.11_ 帧的某些字段编码一个序列

判断这个序列是否在信道中重复出现

从而完成对 _ETA_ 的检测

---

