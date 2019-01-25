# Outline

## A client-side detection mechanism for evil twins - Computer & Electrical Engineering 2017

Created by : Mr Dk.

2019 / 01 / 25 17:48

Ningbo, Zhejiang, China

---

### Introduction

一个流氓 _AP_ 本身可以通过 __有线__ 或 __无线__ 的方式连接到 _Internet_

* 有线的流氓 _AP_ 被称为 _wired rogue_
* 无线的流氓 _AP_ 被称为 _evil twin_

_Evil Twin AP_

* 与合法 _AP_ 拥有相同的 _SSID_
* 普通用户无法区分 _Evil Twin AP_ 和合法 _AP_
* 大部分现代操作系统会连接到 _RSSI_ 最强的相同 _SSID_ 的 _AP_ 上
* 通过各种各样的手段增强自身的 _RSSI_
  * 靠近目标
  * 使用有向天线
* 使用 _de-authentication_ 攻击，强迫受害者与合法 _AP_ 的连接掉线
* 嗅探明文隐私信息
* 发动 _MITM_ 攻击

文本提出了一种基于客户端的解决方案 _ET Detector_：

* 不需要进行数据训练
* 使用开启 _Monitor_ 模式的无线网卡
* 检测 _AP_ 是否将无线数据包转发给另一个 _AP_

_ET Detector_ 的优势：

* 不需要任何合法列表（基于客户端）
* 检测时不需要与任何 _AP_ 关联
* 检测时不需要通过网页认证
* 不需要任何的参数训练
* 攻击者很难避开检测（因为检测利用了 _Evil Twins_ 的转发性质）

---

### Related Work

检测 _Evil Twin_ 的方法主要有两类：

* 基于网络管理员的解决方案
  * 实现于核心网络中
  * 基于预定义的合法列表，认证 _AP_ 的 _fingerprint_
* 基于客户端的解决方案
  * 如果用户不确定网络是否安全，能够自己发起检测
  * 已有的方法需要首先将设备连接到 _AP_ 上，才能进行检测，诸多局限

---

### Principle and Detection Algorithm

#### Monitor Mode

_Monitor_ 模式下，_WNIC_ 能够在不关联到 _AP_ 的条件下抓包

由于 _Evil Twin_ 需要连接到合法 _AP_ 以连接到 _Internet_ - 需要两张无线网卡

* 一张网卡用于模仿合法 _AP_，引诱受害者
* 另一张网卡用于连接到合法 _AP_ 上

#### ET Detector

_Evil Twin_ 需要在受害者和合法 _AP_ 之间转发相应的数据包 - 这个性质是不变的

系统包含四个组件：

* _Progress Controller_
  * 主要组件，协同其它组件
  * 控制 _WNIC_
  * 显示检测结果
* _Packet Monitor_
  * 从 _WNIC_ 上获取数据包
  * 向 _AP Record_ 中存储信息
* _Redirection Detector_
  * 从 _AP Record_ 中读取信息
  * 产生检测结果
  * 检测结果发送至 _Progress Controller_ 用于显示
* _AP Record_

检测机制：

* _Default Testing_
  * 在大多数场景下使用
* _Secondary-device Testing_
  * 在特定场景下使用
  * 使用任何具备 _Wi-Fi_ 能力的额外设备与 _AP_ 关联，建立 _TCP_ 连接
  * 当 _WLAN_ 内只有 _ET Detector_ 一个客户端时也能进行检测

#### Detection Algorithm

1. Initialize
   * 用户被提示选择使用的 _WNIC_
   * _WNIC_ 扫描所有的可用 _AP_，并记录扫描结果
     * _SSID_、加密方式、_BSSID_、协议、_channel_、...
2. Find _APs_ with duplicate _SSIDs_
   * 过滤 _SSID_ 唯一的 _AP_
   * 将所有 _SSID_ 重复的 _AP_ 显示给用户进行选择
   * 记录某个相同 _SSID_ 的所有 _AP_
   * 嗅探这些 _AP_ 的 _TCP_ 数据包
3. Capture packets via monitor mode
   * 如果同一 _SSID_ 下的 _AP_ 工作在不同的信道，则在嗅探时以固定的频率切换信道（频率越快越好）
   * 一个独立线程用于专门处理数据包
   * 对于一个 _SSID_，每一个 _AP_ 都有一个独立的 _BSSID_（_MAC_ 地址）
   * 每一个 _BSSID_ 都有一个 _Hash Table_
     * _key_ 由 _TCP_ 数据包头部的 _sequence number_ 和 _acknowledgment number_ 计算得出
     * _value_ 为 `0` 或 `1`，初始化为 `0`
4. Detecting packet forwarding behavior
   * _Evil Twin_ 在转发数据包时，不会改变 _sequence number_ 和 _acknowledgment number_
   * 在嗅探到某个 _AP_ 的 _TCP_ 数据包时，从数据包头部提取上述字段，计算 _key_
   * 将 _AP_ 对应 _BSSID_ 的 _Hash Table_ 中的 _key_ 对应的 _value_ 置为 `1`，表示这个数据包已经出现过一次
   * 检查其它 _BSSID_ 对应的 _Hash Table_ 的相同 _key_ 对应的 _value_ 是否为 `1`
     * 如果为 `1`，说明出现了数据包转发，那么该 _AP_ 就是 _Evil Twin_ - __hit condition__
   * 与此同时，记录每个 _TCP_ 数据包的 _destination IP_，用于 _secondary-device testing_
5. Determine evil twin and show up results
   * 打印警告信息

---

### Evaluation

#### TCP/IP connection establishment pattern

使用两个专用的代理，仿真正常用户的客户端代理，用于向远程服务器建立 _TCP/IP_ 连接

#### Evaluation of detection accuracy

两个代理作为客户端：

* 一个连接到 _Evil Twin_ 上，一个连接到 _Good Twin_ 上
* 每分钟建立五次 _TCP/IP_ 连接，仿真普通的用户上网行为
* 实验进行十轮，记录 __hit condition__ 的次数（即发现数据包被转发的次数）

实验结果：

* _hit count_ 足够用于确定 _Evil Twin_
* 准确率在 _RSSI ≤ 25%_ 时显著下降，但此时信号过低，_AP_ 无法提供稳定的 _Wi-Fi_ 服务
* 理论上，合法 _AP_ 的 _hit count_ 应该为 `0`
  * 但实际中可能会存在重传帧，导致 _hit count_ 不为 `0`
  * 然而这种情况不会经常发生，所以根据较大的 _hit count_ 依然可以判断出 _Evil Twins_

#### Time efficiency

尝试了多种检测时间的长度

但在每一种情况下，合法 _AP_ 的 _hit count_ 总比 _Evil Twin AP_ 小得多

因此，较短的检测时间足以进行准确的检测

---

### Discussion

#### Limitation

* 当整个网络只有 _ET Detector_ 一个客户端设备时，无法检测 _Evil Twin AP_
  * 需要引入一个 _secondary device_ 协助检测
* 攻击者使用 _3G/4G_ 接入 _Internet_，或使用不同的信道进行转发
* 需要 _Monitor_ 模式网卡的支持

#### Analysis

* 攻击者很难避开检测
* 攻击者可能通过重新产生数据包，将 _sequence number_ 和 _acknowledgment number_ 偏置固定数量的方式避开检测

#### Future work

由于目前接入 _Wi-Fi_ 的大部分设备是移动设备

* 将软件从 _Laptop_ 上迁移至移动 _OS_ 中

---

### Summary

是一个比较简单粗暴的解决办法

但是我觉得有些过于理论化了

实验环节让人不太能信服

数据帧的重传真的不会经常发生吗？

---

