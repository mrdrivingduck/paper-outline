# Outline

## Hacker's Toolbox - Detecting Software-Based 802.11 Evil Twin Access Points - CCNC 2015

Created by : Mr Dk.

2018 / 12 / 28 20:16

Nanjing, Jiangsu, China

---

### Introduction

_802.11_ 标准没有为 _Wi-Fi AP_ 提供很严格的标识，客户端辨认 _AP_ 的仅有参数：

* _SSID_
* _BSSID_

只要伪造这两个参数，即能伪造 _AP_

_802.11_ 网络的已有安全机制：

* _WPA-PSK_ - 只有攻击者得不到 _PSK_ 时才能有效
  * 在公共场合，_PSK_ 一般来说都是开放的
* _WPA-Enterprise / 802.1X_ - 需要复杂的安装、维护
  * 在公共场合不可能应用
* _Web-based_ - 重定向到一个网页，使用各类信息登录
  * 网页很容易被克隆，从而窃取重要信息
  * 很多工具已经可以自动化地实行攻击

攻击者已经不需要专用的硬件设备来施行攻击，移动设备内置的热点已经足够用于攻击

* 已有工具能够使攻击完全自动化
* 使用移动设备可以使被发现的概率最小化

论文贡献：

1. 对用于 _Evil Twin Attack_ 的工具进行综合评价
2. 提出了一些可用工具的辨别方式，提出了基于软件的 _AP_ 的检测方法
3. 在多个平台上验证了方法的效果

---

### Related Work

_Evil Twin Attack_ 的防御可被分为三类：

* 协议修改
  * 使用类似 _SSL/TLS_ 的方式认证 _AP_
  * 在已有网络中无法轻易部署
* 利用 _AP_ 的硬件特征建立指纹
  * 时钟偏差
  * _Device-intrinsic temperature dependency_
  * 无线信号
  * 需要特定的专用设备
* 基于非硬件的检测
  * 检测在路由中是否出现了额外的一跳
    * 假设伪造 _AP_ 使用合法 _AP_ 转发通信
  * 分析到本地 _DNS_ 服务器的 _RTT_
  * 只能解决部分问题

没有方法能够检测出一个 _AP_ 是硬件 _AP_ 还是软件 _AP_

---

### Software-based 802.11 Access Points

* _hostapd_
  * 适用于 _Linux_ 系统
  * 需要一个可以运行于 _master mode_ 的无线网卡
  * 以及一个用户空间的守护进程，用于进行 _AP_ 管理和所有的认证
* _MadWifi_
  * 基于硬件抽象层，可以建立多个虚拟 _AP_
  * 该项目在几年前被终止，代替品现在已经是 _Linux_ 内核的一部分
    * 不提供建立软 _AP_ 的高级功能
    * 过时的 _MadWifi_ 驱动已经不被现代的硬件和操作系统内核支持
* _Aircrack-ng_
  * _Airbase-ng_ - 软 _AP_ 的实现
  * 开箱即用的工具
* _Karma_
  * _AP_ 根据用户的 _probe_ 帧迅速建立网络
  * 在低层基于 _hostapd_ 和 _aircrack-ng_

目前主流的移动操作系统，也有内置的建立软 _AP_ 的特性

* _Android_ - _Tethering and portable hotspot_
* _iOS_ - _Personal Hotspot_

只能将通信路由至蜂窝网络

在 _root_ 的移动设备上运行特殊工具也可以建立软件 _AP_

硬件 _AP_ 和软件 _AP_ 的界线通常会被混淆

* _AP_ - 支持 _master mode_ 的芯片集，和用于执行功能、认证、管理的配套软件
* 硬件 _AP_ - 只用来完成上述功能的嵌入式设备

现在很多 _AP_ 内部的无线芯片与客户端中的无线芯片不同

但也有不少芯片同时用于客户端和 _AP_

不少 _AP_ 的操作系统使用的是基于 _Linux_ 的 _OpenWRT_，内部使用 _hostapd_

---

### Method For Detection

研究主要基于 _802.11_ 的两种 _management frame_：

* _beacon frame_
  * 由 _AP_ 周期性发送，以显示它们的存在，以及时钟同步
  * 能够以完全被动的方式分析
* _probe frame_
  * 由 _stations_ 主动发出（主动扫描）

因为这两种帧能够在与不信任的 _AP_ 建立连接之前就被分析

#### Analyzing beacons to disclose TSF (in-)accuracy

_airbase-ng_ 对 _Timing Synchronization Function (TSF)_ 时间戳的准确性有较大影响

这一特性被用于检测基于 _airbase-ng_ 的软件 _AP_

_AP_ 是所有关联上的 _stations_ 的 _timing master_

因此在 _AP_ 周期性广播的 _beacon_ 中，带有 _TSF_ 计时器时间戳

所有的客户端都需要将它们的 _TSF_ 计时器调整到这个值

_802.11_ 标准对 _TSF_ 时间戳的精度要求严格

* _AP_ 应在帧的第一个 _bit_ 发送的瞬间设定时间戳为当时的 _TSF_ 计时器时间
* 软件 _AP_ 需要模仿这个过程（硬件 _AP_ 会由硬件和固件实现），因此准确性下降

检测方法：追踪记录的 _beacons_

* `Trec` - 收到 _beacon_ 的时间
* `o` - 收到 _beacon_ 的时间与 _beacon_ 中的 _TSF_ 时间戳的差值

对于硬件 _AP_ 来说，这一模式是准确的线性模式

对于基于 _airbase-ng_ 的软件 _AP_ 来说，存在异常性的分散

原因：软件 _AP_ 产生 _beacon_，特别是 _TSF_ 时间戳的方式

* 硬件 _AP_ 使用优化过的硬件与固件结合的方式保证准确性
* _airbase-ng_ 不能模仿出这种准确性，因为 _beacon_ 的产生和传输之间存在系统延时

显然，其它软件 _AP_ 不依赖于模仿这一过程，而是把这一过程委托给了更低层的驱动或者硬件

实验表明，硬件与 _airbase-ng_ 的 _TSF_ 时间戳准确性没有关系

* 只受额外因素的影响 - 驱动、_OS_

在网络流量密集的情况下，软件 _AP_ 的 _TSF_ 时间戳的不准确性会更加明显

#### Probe frames

##### Active probing on adjacent channels

公共 _Wi-Fi_ 热点一般使用 _2.4 GHz_ 频道，以保证大部分客户端都能使用

* 该频道被划分为 _14_ 个信道
* 信道中心频率相差 _5 MHz_，但要求每个信道带宽为 _16.25-22 MHz_，因此相邻信道重叠

不可避免地，_Wi-Fi_ 设备会在其工作信道上，收到不是该工作信道发出的帧 - __信道干扰__

实验证明，硬件 _AP_ 和软件 _AP_ 对于这种帧的处理方式不同：

* 硬件 _AP_ 总是只在工作信道上回应
* 软件 _AP_ 除了在工作信道上回应外，在相邻信道上也会回应

_Wi-Fi_ 接收方很难确定接收到的帧是来自哪里特定信道

为了选择有效帧，过滤噪声帧，无线设备会根据信号强度和比较信号调制的同步性，计算 _adjacent channel rejection_

论文推测，基于固件的硬件 _AP_ 与软件 _AP_ 相比，会更加严格地过滤相邻信道干扰

这种推测是合理的：

* _AP_ 通常工作在固定信道，因此它们希望将干扰过滤得越多越好

* 对于客户端来说，接收到的相邻信道的帧有益于它们的快速网络发现过程

从所有重叠信道上向 _AP_ 发送 _probe request_，并监听各个信道上收到 _probe response_ 的速率

如果某个信道上的接受速率超过工作信道接收速率的某个阈值，则检测出了软件 _AP_

* 需要考虑到距离 _AP_ 较远，或者带宽过载的情况，会导致总体的接受速率下降 - 选定合适的阈值

##### Malformed Probe Request Stimuli

硬件 _AP_ 和软件 _AP_ 对于伪造的 _probe request_ 反应一致

在 _probe request_ 中：

* _Address 1_ 携带目标 _MAC_ 地址
* _Address 3_ 携带 _BSSID_，即 _AP_ 的 _MAC_ 地址
* 因此两个地址字段相同

根据标准，_AP_ 应当对 _Address 1_ 和 _Address 3_ 都携带其 _MAC_ 地址的 _probe request_ 回应 - 

* 但实际上，_AP_ 不检查 _Address 3_
* 而包括 _Windows_ 和 _iOS_ 在内的软件 _AP_ 实现会检查 _Address 3_

出现这种情况的原因：

* _AP_ 通常不是 _IBSS_ 或网状网络的一部分，因此 _Address 3_ 是不相关的
* 因此验证可以被跳过，用于提升效率
* 而软件 _AP_ 显然避免了这种优化，严格地执行了标准

检测方式：

* 向 _AP_ 发送伪造的 _probe request_，_Address 3_ 使用随机的 _MAC_ 地址
* 硬件 _AP_ 会回应 _probe response_，软件 _AP_ 不会回应

---

### Evaluation And Discussion

_airbase-ng_ 是唯一自己实现 _time-critical_ 部分的工具

* 其它工具将该部分委托给更低层的驱动或硬件
* 虽然 _airbase-ng_ 存在这种问题而容易被检测为软件 _AP_
  * 这种特性不太可能被修改
  * 因为 __不依赖低层功能__ 正是 _airbase-ng_ 的重要特性之一
  * 因此它能够在几乎任意的 _Wi-Fi_ 硬件上运行

---

### Conclusion

软件 _AP_ 运行的硬件被优化于用作一个客户端而不是 _AP_

因此在其中提取一些差异性，能够区分出它们属于硬件 _AP_ 还是软件 _AP_

---

### Summary

了解了几个可以用于 _Evil Twin AP_ 攻击的工具

以及它们的低层实现的区别

也为以后的软件 _AP_ 检测作一些理论铺垫

---

