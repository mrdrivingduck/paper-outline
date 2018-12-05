# Outline

## Fingerprinting Electronic Control Units for Vehicle Intrusion Detection - Usenix Security 2016

Created by : Mr Dk.

2018 / 12 / 05 17:30

Nanjing, Jiangsu, China

---

### Introduction

对于应用 __车联网__ 和 __自动驾驶__ 的车辆

研究证实，_Electronic Control Units (ECU)_ 可以被远程攻破

攻击者可以通过在车内网络中 __注入数据包__ 来控制车辆

* 研究者攻克并远程使一辆高速上的 _Jeep Cherokee_ 停下来，引发了 _140_ 多万辆召回

车内网络防御的两个大方向：

* 消息认证
  * 车内网络的数据帧中已经没有地方放置 __消息认证码__（_Message Authentication Code (MAC)_）
  * 需要实时的处理和通信
* 入侵检测
  * _State-of-the-art IDS_
    * 监控车内网络消息的 __内容__ 和 __周期性__，判断是否发生了显著变化
    * 车内网络消息不携带 __发送者__ 信息，因而无法确定来源
    * 由此无法识别哪个 _ECU_ 被攻击
  * _Clock-based IDS_
    * 通过控制包头部的时间戳，估计时钟偏差，从而辨别网络设备
    * 车内网络中传输的数据包没有时间戳

### Background

#### CAN Protocol

_CAN_ 是最广泛使用的车内通信协议

* 连接车内各 _ECU_
* 多主体
* 消息广播总线系统

_CAN_ 数据帧 - 维护数据一致性

* _ID_
* _Data Length Code (DLC)_
* _Data_
* _CRC_

_CAN_ 面向消息：

* 数据帧中不携带 发送者 / 接收者 的地址
* 数据帧中携带一个独立的 _ID_，代表消息的 __优先级__ 和 __含义__

总线仲裁：

* 一旦探测到总线空闲，所有已缓存消息的结点同时试图占用总线
* 每个结点每次传输 _CAN_ 数据中的 _ID_ 字段中的一位（从 __最高有效位__ 开始）
* _CAN_ 总线的设计在逻辑上是 __线与__
* 结点发现总线上的数值和自己传输的数值不同时，退出总线竞争并转入监听模式
* 发送 _ID_ 最小的 _ECU_ 赢得总线仲裁
* 退出竞争的结点在总线空闲后试图重新竞争

时钟同步：

* _CAN_ 缺少时钟同步机制
* 每个 _ECU_ 使用自己的石英晶振 - 导致随机的时钟漂移

#### Related Work

##### Message Authentication

_CAN_ 总线的数据载荷只有 _8_ 字节 - 没有空间放入加密的消息认证码

现有解决方案：

* 将消息认证码拆分到多个 _CAN_ 数据帧中
* 使用多个 _CRC_ 字段引入 _64_ 位的 _CBC-MAC_
* 利用信道范围之外的信道进行消息认证

局限：

* 没有抵挡 _DoS_ 攻击的能力
* 需要大量处理功耗
* 增大了消息量和总线利用率

车内场景：

* _ECU_ 资源受限
* 实时性要求高

##### Intrusion Detection

大部分 _CAN_ 帧是 __周期性__ 的

* 监控数据帧周期性到达的间隔，计算 __熵__
* 监控数据帧的内容，分析消息的范围、相关性

### Attack Model

攻击者类型：

* _Weak Attackers_
  * 可以使 _ECU_ 停止或减缓发送数据帧
  * 不能利用 _ECU_ 注入增加的数据帧
* _Strong Attackers_
  * 除了 _Weak Attackers_ 可以做的事情之外
  * 能够注入任意数量的攻击数据帧

攻击场景：

* _Fabrication Attack_
  * 注入高优先级的数据帧以垄断总线
* _Suspension Attack_
  * 使 _ECU_ 暂停发送消息，从而拒绝服务
* _Masquerade Attack_
  * 使 _ECU_ 暂停发送消息
  * 使用另一个 _ECU_ 注入伪造消息
  * 在不改变网络行为的前提下进行了攻击 - 不会被一般的 _IDS_ 发现

### Clock-Based Detection

_State-of-the-art IDSs_ 无法检测出 _Masquerade Attack_ 的原因：

* _CAN_ 消息不携带发送方信息
* 无法识别被攻破的 _ECU_

_CIDS_ 采用的方法：

* 利用消息的频率来识别 _ECU_
* 利用 _ECU_ 特有的行为进行入侵检测

#### Fingerprinting ECUs

定义：

* _offset_ - 两个时间戳的数值差
* _frequency_ - 时间戳的增长率（斜率）
* _skew_ - 两个时间戳增长率的差（斜率的差）

如果 _offset_ 和 _skew_ 为 _0_，那么两个时钟被认为是 __同步__ 的

#### Clock skew as a fringerprint

由于不同步结点的 _offset_ 和 _skew_ 由它们本地的时钟决定

因此可以被用来当做结点的指纹

但是在车内环境中，数据帧中没有 _timestamp_

所以需要用其它方式来计算 _skew_

#### Clock skew estimation

从某一个结点的视角来看，另一个结点发来的周期数据帧的时间间隔由几部分组成：

* 固定的理想时间间隔 _T_
* 晶振固有的 _offset_ _O_
* 网络延时 _d_
* 时间戳数字化延时 _n_

得到时间间隔的计算公式

时间间隔均值 - 近似于 _T_

可以得到两个时间：

* 数据帧收到时的实际时间戳
* 通过时间间隔均值和初始值计算出的 __估计时间戳__

这两个时间的 __平均差值__ 为 _O_ 的均值 - 随不同的设备而异

_O_ 的均值计算 - 接收 _N_ 条消息的 _offset_ 并累加

累加的 _offset_ 直线的斜率就是 _clock skew_，即设备指纹

车内入侵检测的两种方式：

* _Per-message Detection_
* _Message-pairwise Detection_

#### CIDS — Per-message Detection

##### Modeling

对于某一个消息 _ID_，_CIDS_ 利用到达时间戳获得累积 _offset_

由于 _skew_ 是一个常量，累积 _offset_ 应当随时间线性增长

采用线性回归模型

常数项为 _identification error_ _e_

未定回归参数为线性模型的斜率 _S_，即 _skew_

使用 _RLS_ 算法决定 _S_，_e_ 应当趋向于 _0_

_RLS_ 算法：

* 每次采集 _N_ 条信息的 __到达时间戳__ 和 __时间间隔__
* 如果在很长时间内没有收到信息 - 将剩余的时间戳和时间间隔设定为非常大
* _N_ 条消息采集完成后，计算累积 _offset_，从而决定 _e_
* 基于 _e_、增益、协方差，更新 _RLS_ 回归参数 _S_，即 _skew_

如果没有被攻击，那么 _e_ 应当趋向于 _0_，_skew_ 应当趋向于一个常数

在算法中，还有一个 __遗忘因子__ 用于降低旧数据的权重

##### Detection

对于一个给定的 _ID_，_CIDS_ 运行 _RLS_ 算法估计 _skew_，并创建一个正常模型

判断是否出现了与正常模型有偏差的异常测量值

* _Fabrication Attack_ 显著增大了 __测量到达时间__ 和 __估计到达时间__ 的偏差，使累积 _offset_ 的增长率突然增大，导致 _e_ 的增大
* _Suspension Attack_ 会产生如上的同样结果
* _Masquerade Attack_ 由于使用不同的 _ECU_ 发送消息，因此也会改变累积 _offset_ 的增长率，导致 _e_ 的增大

在正常的时钟行为中，_e_ 应当趋向于 _0_，当 _e_ 为一个很高的非 _0_ 数值时，说明出现了入侵行为

_CUSUM_ ???

#### CIDS — Message-pairwise Detection

通过两个周期性消息的平均 _offset_ 的相关性检测入侵

#### Verification

#### Root-cause Analysis

遭受攻击时，根据消息的 _ID_ 提取 _skew_ 信息，并与根据其余 _ID_ 提取出的 _skew_ 信息进行比较

判断某两个消息是否发送自同一个 _ECU_

### Evaluation

### Discussion

### Conclusion

_CIDS_ 能够检测多种针对车内网络的入侵

并能够定位攻击者入侵的 _ECU_，并进行根本原因分析

### Acknowlegments

---

### Summary

线性回归那一块实在是看不进去了......

---

