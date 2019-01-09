# Outline

## WADAC - Privacy-Preserving Anomaly Detection and Attack Classification on Wireless Traffic - WISEC 2018

Created by : Mr Dk.

2019 / 01 / 09 14:08

Nanjing, Jiangsu, China

---

### Introduction

近年来，_IoT_ 设备的数量越来越多

随之而来增大了安全和用户隐私的威胁

僵尸网络 - _Mirai_ - 将目标指向 _IoT_ 设备 - _DDoS_ 攻击

入侵检测系统（_Intrusion Detection Systems, IDS_）能够帮助用户检测被攻击的设备

_IDS_ 能够在 _OSI_ 模型的不同层次进行流量采集

在网络层及以上采集的数据包含用户和应用的信息

* 被视为违反了用户的隐私
* 只可能由网络操作者完成

在本文中，作者认为只利用链路层数据进行异常检测和攻击分类是可能的

* 可以使用一个被动的 _IDS_ 在附近的第三方网络中进行异常和攻击检测
* 检测框架
  * 异常检测模块 - 无监督
  * 攻击分类模块 - 监督
* 不访问网络层及以上的数据 - 最小化威胁设备和用户的隐私

本文的 _contributions_：

* 设计和实现了一个框架，能够通过分析链路层的特征，被动地检测和识别应用层攻击
* 检测机制在 _IP cameras_ 检测出了 _96.2%_ 的攻击；对攻击进行分类达到了 _97%_ 的准确率
* 对受僵尸网络影响的 _IP camera_ 达到了 _96.1%_ 的准确率，检测 _DDoS_ 攻击达到了 _98.3%_ 的准确率
* 将框架应用到不同类型的设备上，异常检测达到了 _98.8%_ 的准确率
* 对于基于 _ZigBee_ 的智能灯泡的洪泛攻击检测，达到了 _99%_ 的准确率

---

### Background

#### Feature Extraction and Feature Selection

从链路层头部提取特征，不解密 _payload_

通过 _imformation value, IV_ 选择一部分提取的特征

* _IV_ 量化了一个特征在异常检测中的影响
* 选择 _IV_ 较大的特征进行异常检测

#### Autoencoders

使用监督学习很难检测出未知异常（训练数据中没有的异常）

使用 _autoencoder_，一个无监督学习模型

* 试图学习一个近似函数的功能
* 生成一个与输入向量类似的输出向量

在本文中，输入向量是从正常流量通信中提取的特征

* 输入 _d_ 维特征向量
* 在低维空间中表示它
* 重建一个 _d_ 维的输出向量

对于一个最简单的 _autoencoder_ - 一个输入层、一个隐层、一个输出层

* 输入层 - 输入向量
* 隐层 - 低维向量
* 输出层 - 输出向量

每一层都有权重和偏置向量 - 随机初始化

_Autoencoder_ 被训练为使重建错误最小化

在本文中 - _deep autoencoder_

* _tanh_ （双曲正切）作为激活函数
* 随机梯度下降作为优化器

---

### Privacy-Preserving Attack Detection and Classification

#### System Model and Attacker Model

系统模型包含三部分：

* _IoT_ 设备
  * 使用 _802.11_ 或 _802.15_ 连接
* 攻击者
  * 通过无线介质与 _IoT_ 设备交互
* _Wireless Anomaly Detection and Attack Classification, WADAC_

攻击者的目标：使用应用层的攻击攻陷 _IoT_ 设备

#### Problem Statement

本文的目标：通过链路层信息检测出位于第三层及以上的攻击，从而保护用户和设备的隐私

最大化检测准确率，最小化错误警报

在检测出异常后，基于训练过得攻击类型，识别攻击的类型

#### WADAC: Wireless Anomaly Detection and Attack Classification

_WADAC_ 包含四个模块：

* _traffic collector_
  * 被动嗅探链路层通信
  * 根据 _MAC_ 地址识别设备
* _feature extractor & selector_
  * 从采集的通信信息中提取特征，选择有重要作用的特征
* _anomaly detector_
  * 利用特征检测异常
* _attack classifier_
  * 将检测到的异常分类到学习过的攻击类型中

为了检测 _WADAC_ 的性能，开发了一个 _attack generator_ 进行各种攻击的生成

#### Anomaly Detector and Attack Classifier

底层原理：_IoT_ 设备的通信是有高度规律的

##### Anomaly Detector

使用 _autoencoder_ 检测异常通信模式

通过无监督训练，_autoencoder_ 学习到了一对非线性变换：

* 从原始空间到隐藏空间的映射 - _encode_
* 从隐藏空间到原始空间的映射 - _decode_

这个变换用于以最小的重建偏差，重建通信模式

* 重建后，重建偏差高于阈值的数据点被视为异常

阈值的选择：

在不同的阈值下，衡量多个指标：

* _accuracy_
* _sensitivity_
* _specificity_

阈值的计算：`Eth` 和 `IQR`

##### Attack Classifier

使用一个已知的攻击集合，对随机森林模型进行监督训练

---

### Implementation

#### Attack Generotor

使用 _ARP-scan_ 发现目标设备

* 端口扫描
* _vulnerability checks_
* _brute force Telnet logins_
* _DoS_
  * 发起不完整的 _HTTP_ 请求，不让 _TCP_ 会话关闭，枯竭服务器资源
* _DDoS_
* _Mirai_
  * 建立 _Control and Commad (C&C)_ 服务器
  * 僵尸病毒安装在 _IoT_ 设备上
  * 由感染病毒的 _IoT_ 设备向其它设备发动攻击
* _Zigbee flooding attacks_

#### Traffic Collector

嗅探 _AP_ 的工作信道以获得更多数据包

嗅探到的数据包按照 _MAC_ 地址分组

* 便于按照设备人为标记它们的活动，方便之后的攻击分类

先采集正常通信下，不同设备的数据包

再采集恶意通信下，不同设备的数据包

#### Feature Extractor & Selector

输入：包含关联到同一个设备的数据帧的 `.pacp` 文件

##### Extracting Packet Features

_packet features_ - 根据每一个帧的 _header_ 信息，提取一系列特征

> Dest. MAC
>
> src. MAC
>
> Trans. MAC
>
> Recv. MAC
>
> frame type
>
> frame subtype
>
> more frag. flag
>
> more data flag
>
> frame size
>
> frame time

##### Extracting Traffic Features

将所有数据包按 `Sblk` 进行分块 &rarr; 分出 `k = nf / Sblk` 个块

对在每个块中，对每个 _MAC_ 地址产生一个对应的 _signature_

利用 _Packet Features_，可以计算出 _Traffic Features_，分为四类：

* _cencus_ - 每块中不同类型的帧的个数
* _ratio_ - 每块中帧的数量的比例
* _load_ - 每块中帧的尺寸的均值和标准差
* _gap_ - 每块中帧的到达时间差的均值和标准差

在 _Wi-FI_ 案例中，不同类型的帧的尺寸在攻击过程中显著不同，而 _ZigBee_ 案例中没有出现这种情况

因此可将每一块中的帧按照帧类型分为三种：

* _data frames_
* _control frames_
* _management frames_

在攻击过程中，_data frames_ 的变化最为显著

因此它们被分入相同长度 `Lbin` 的箱子中？？？？？？？？？？？？？？？？？？？

从每一个块中总共提取了 _45_ 个 _traffic feature_

在每个 _bin_ 中提取了 _18_ 个特征，对于每块中的两个 _bin_，一共有 _36_ 个特征

加上每个块中总体的 _45_ 个特征，一共有 _81_ 个特征

类似的，从 _ZigBee_ 的帧中可以提取 _72_ 个特征

这些特征包含了 _Traffic Features_ 的四个类型

##### Feature Selector

对每一个特征计算 _IV_

#### Anomaly Detector and Attack Clasifier

##### Anomaly Detector

异常检测模块严重依赖于设备功能

* 当设备功能变化时，设备的通信模式可能会很不一样
* 因此 _autoencoder_ 需要学习的通信模式也会变化

比如 _IP camera_ 的行为，在空间时和活跃时是不一样的

* 因此需要对两种设备状态分别训练
* 如果两个模型都检测出了异常，异常才会被报告

而对于智能灯泡来说，一个模型已经足够用来检测异常了（功能有限）

异常检测模型将会基于设备的类型进行选择

##### Attack Classifier

利用不同攻击的 _signatures_，训练一个随机森林模型

每一个异常通信的 _signature_ 都被手动标记为对应的攻击类型

用于分类的特征和之前用于异常检测的特征是同一个相同的特征集合

* 该模块只有在异常检测模块检测到异常后才会启动

---

### Experimental Setup

#### Wireless Traffic Collection

##### IP Cameras

收集到了五个通信子集：

1. `Dcam, a` - 只包含除了 _Mirai_ 之外的所有攻击的通信
2. `Dcam, i` - 只包含设备空闲的通信（设备没有连接到相应的 _APP_）
3. `Dcam, l` - 只包含设备活跃的通信（设备连接到相应的 _APP_）
4. `Dcam, ail` - 混合了设备空闲、活跃、异常状态的通信
5. `Dcam, Mirai` - 设备感染僵尸病毒，被用于攻击其它设备的通信
6. `Dcam, m-DDoS` - 从被僵尸网络 _DDoS_ 攻击的受害设备上收集的通信

使用 `Dcam, i` 和 `Dcam, l` 用于训练 _autoencoder_，用剩余数据测试

##### Wi-Fi enabled smart bulbs

收集到了两个通信子集：

1. `Dbulb, l` - 包含设备正常活动时的通信（包括改变颜色、主题等）
2. `Dbulb, ld` - 包含设备正常活动和遭受 _DDoS_ 攻击时混合的通信

使用 `Dbulb, l` 训练 _autoencoder_，用另一个数据集测试

##### ZigBee enabled smart bulbs

同上

#### Signature Formation

通过实验尝试不同的 `Sblk`，以提升异常检测模块的性能

用不同的 `Sblk` 构造 _signatures_，每个 _signature_ 根据对应的活动打上标签

---

### Results

#### Performance Metric

* Accuracy (ACC) - `Acc = (Ntp + Ntn) / N` 
  * 分类正确的样本占样本的比例
* Sensitivity (Sens) - `Sens = Ntp / (Ntp + Nfn)`
  * 正确检测出的正常样本占所有正常样本的比例
* Specificity (Spec) - `Spec = Ntn / (Ntn + Nfp)`
  * 正确检测出的异常样本占所有异常样本的比例

#### Anomaly Detection: IP Camera

分别开发了 `enci` 和 `encl`，分别使用 `Dcam, i` 和 `Dcam, l` 进行训练

* `enci` 在设备空闲状态下进行异常检测
* `encl` 在设备活跃状态下进行异常检测
* 当两个 _autoencoders_ 都发出警报时，说明检测到了异常

改变 `Sblk` 和 `Lbin` 会影响每一个特征的分布

* 因此找到最佳的参数是很重要的
* 为了找到最佳的组合，修改 _autoencoder_ 的配置
  * 隐层个数
  * 每层中的结点个数

##### ROC Curve Analysis

选择重建错误的最佳阈值

##### Optimal Block Size and Bin Length

##### Feature Importance

#### Detecting Botnet Attacks

#### Attack Classification: Wi-Fi Devices

使用 `Dcam, a`、`Dcam, m-DDoS` 和 `Dcam, Mirai` 数据集训练和测试随机森林分类器

数据集被随机分为了训练（65%）和验证（35%）

#### Detecting DDoS in TP-Link Bulb

#### Execution Time Analysis

#### Anomaly Detection: ZigBee Devices

---

### Related Work

基于采集通信的层次

1. _L1_ - 采集物理层特征，包括信号强度和噪声
2. _L2_ - 采集链路层特征，包括 _MAC_ 地址、帧类型、帧长度和 _payload_
3. _L3+_ - 采集网络层及更高层特征，包括 _IP_ 地址、包长度、延时等

---

### Conclusion

本篇论文提出了一个框架

* 检测 _IoT_ 设备的异常通信
* 将攻击分类到已知集合类型中

只分析链路层数据，保护用户隐私

---

### Summary

看着非常费力

尤其是最后实验结果的参数选择

完全看不懂

查了一些资料发现全是深度学习的知识

我恨啊...... :sob:

---

