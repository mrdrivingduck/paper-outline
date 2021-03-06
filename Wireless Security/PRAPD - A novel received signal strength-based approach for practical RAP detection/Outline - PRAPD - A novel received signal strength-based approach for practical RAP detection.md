# Outline

## PRAPD - A novel received signal strength-based approach for practical rogue access point detection - IJDSN2017

Created by : Mr Dk.

2018 / 11 / 23 0:18

Nanjing, Jiangsu, China

---

### Abstract

使用 _RSS（Received Signal Strength）_ 信息是一个检测流氓 _AP_ 的有效方法

能够收集到的信息是由多个 _Sniffer_ 获取到的 _RSS_ 信号组成的多维向量

而向量中的某几个维度经常会因为一些原因 __缺失__

导致检测流氓 _AP_ 时的 __高误报率__

本文提出了一个数据预处理模式

通过 __填充__、__过滤__、__取平均__ 的方式对 _RSS_ 向量进行预处理

再对 _RSS_ 向量进行聚类分析

对于聚类时使用的距离

本文提出了一种距离度量模式

动态地利用 _RSS_ 向量中的部分数值，以最小化缺失值带来的距离偏差

---

### Related Work

由于可以轻易伪造 - 使用白名单过滤 _SSID_、_MAC_ 地址等参数的方式已经失效

目前检测流氓 _AP_ 的方式可以分为三个等级：

* _User-side detection_
  * _low-cost_ & _lightweight_
  * Difficult to be _widely applied_
* _Wired-side detection_
  * 网络管理员视角
  * 只需要监控通向外网的流量即可（网关）
  * 只能捕捉到通过 __有线介质__ 向外网传输数据的流氓 _AP_，无法捕捉使用 _4G LTE_ 等方式的 _AP_
* _Wireless-side detection_
  * 从无线链路上的信号中提取特征
  * 由于无线信道的不稳定性，缺失的信号值会对流氓 _AP_ 的检测产生重大影响
    * 本文将提出方法解决这个问题

---

### Rouge AP attack

描述一个可能的攻击场景

* 使用 __无线路由器__ 或 __一台运行热点共享软件的笔记本__
* 配置 _SSID_、_MAC_ 地址等参数与已有的合法 _AP_ 完全相同
* 合法 _AP_ 与流氓 _AP_ 共存
* 流氓 _AP_ 通过加大信号强度，引诱设备连接
* 本文不关心流氓 _AP_ 如何接入互联网 - 可以使用 _4G LTE_ 等方式

---

### Design of PRAPD

#### Overview

* 多个处在不同位置的 _Sniffer_ 分别从 _AP_ 的 _beacon_ 帧中获取 _RSS_ 值
* 使用 _beacon_ 帧中的 _timestamp_ 来识别同一个帧
* 对于某一个 _timestamp_，`n` 个 _Sniffer_ 可以得到一个 `n` 维的 _RSS_ 向量

如果没有流氓 _AP_ 存在的情况下：

* 同一个 _BSSID_ 的 _RSS_ 向量之间的 __距离__ 应该十分接近
* 因为所有帧都发自同一个 _AP_

如果流氓 _AP_ 与合法 _AP_ 都存在的情况下：

* _Sniffer_ 接收到的 _RSS_ 向量应该是 合法 _AP_ 与流氓 _AP_ 的混合
* 那么 _RSS_ 向量之间的相对距离应该会变大
* 离线训练模式下，使用聚类分析的方式可以决定一个阈值 &alpha;
* 在线检测模式下，使用聚类分析的方式和阈值 &alpha; 来决定是否出现了流氓 _AP_

#### Data preprocess

在实际环境中，_RSS_ 向量中经常会存在 __缺失值__，原因有以下两点：

* 无线传输范围有限
  * 若 _Sniffer_ 超出了 _AP_ 的传输范围，则无法收到 _RSS_
* 无线链路的不可靠性
  * _Sniffer_ 可能会丢失帧

##### 策略1 - Data Filling

使用 `-100` （信号最小值）作为缺失数据的默认填充值

* 对于 __超出传输范围__ 的情况可以适用 - 可近似认为信号微乎其微

* 对于 __丢失帧__ 的情况，会造成数据明显的夸张化 - 比如下面的向量：

  | Time | RSS1 | RSS2 | RSS3 | RSS4 | RSS5 | RSS6 |
  | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
  | t    | -41  | -    | -    | -43  | -    | -42  |

  | Time | RSS1 | RSS2 | RSS3 | RSS4 | RSS5 | RSS6 |
  | ---- | ---- | ---- | ---- | ---- | ---- | ---- |
  | t    | -41  | -100 | -100 | -43  | -100 | -42  |

  数据显然不合理 - _AP_ 的位置没有变，信号偏差不可能这么大

##### 策略2 - Data Filtering

将向量数据 __缺失率__ < &beta; (&beta; = 0.15) 时，缺失的数据原因才被认为是 __帧丢失__

##### 策略3 - Data Averaging

对于判定为 __帧丢失__ 的向量，使用已有的 _RSS_ 平均值填充缺失值

#### Clustering analysis

使用 _k-medoid_ 算法（_k = 2_）

对于每个 _timestamp_，存在一个由多个 _Sniffer_ 捕获到的 _RSS_ 值组成的 _RSS_ 向量

对于多个 _timestamp_，存在所有的 _RSS_ 向量构成的一个向量集合

* 为提升性能，该向量集合没有重复值
* 单独用一个向量记录对应向量的 __重复出现的次数__

一个示例的向量集合：

* 每行为一个 _RSS_ 向量

| Timestamp | RSS1 | RSS2 | RSS3 |
| --------- | ---- | ---- | ---- |
| t1        | -50  | -60  | -90  |
| t2        | -49  | -62  | -93  |
| t3        | -48  | -61  | -95  |
| t4        | -49  | -61  | -90  |
| t5        | -50  | -60  | -91  |
| t6        | -51  | -63  | -92  |

##### 距离度量方式

传统的 __欧氏距离__ 考虑了所有维度，可能会因为 __缺失值__ 而产生较大偏差

提出的度量方式：__动态__ 地使用 _RSS_ 向量中的 __部分__ 数值

<img src="http://chart.googleapis.com/chart?cht=tx&chl= {D_{p,q}^B = \sqrt{\frac{\sum_{b\in{BL}}(S_p(s_b)-S_q(S_b))^2}{B}}}" style="border:none;">

_B (B ≤ n)_ 指参与距离计算的 _RSS_ 值个数 - 体现了 __部分性__ - 如果 _B = n_，那么相当于欧氏距离

_BL_ 是指 `p` 向量和 `q` 向量 __分量相差最大__ 的前 _B_ 个分量

##### 算法执行过程

初始化时，随机选择两个 _RSS_ 向量作为聚类中心

将向量集合中的所有向量分别分入两类中

更新聚类中心，迭代，重新对所有向量分类

直到到达指定的次数或聚类中心不再改变

##### 聚类中心计算

考虑到集合中有一些 _RSS_ 向量重复多次，在计算聚类中心时需要考虑重复出现的次数

<img src="http://chart.googleapis.com/chart?cht=tx&chl= {S_{nu} = \arg\min_{{S_p}\in{C_1}}\sum_{{S_q}\in{C_1}}w_qD_{p,q}^B}" style="border:none;">

含义：

存在一个向量 `p`，使其它所有向量 `q` 到 `p` 的距离 × `q` 的重复次数之和最小

那么 `p` 就是新的聚类中心

#### Offline profile

在没有流氓 _AP_ 的条件下进行聚类分析

用两个聚类中心的距离决定阈值 &alpha;

#### Online  detection

对获取到的未知数据进行聚类分析

计算得到的两个聚类中心的距离 _D_

若 _D > &alpha;_，则表明检测出了流氓 _AP_

---

### Evaluation

指标：_Detection rate_ & _False alarm rate_

* 聚类中心的阈值 &alpha; 是检测率和误报率的折衷
* 距离计算的部分参数 _B_ 是检测率额误报率的折衷
* 流氓 _AP_ 与合法 _AP_ 距离越远，检测率越高

---

### Summary

使用了多 _Sniffer_ 与机器学习的聚类方法检测

但是考虑到检测率无法达到较高水平

存在误报率也较高

我个人认为不适合引入 _WIDS_ 中

---

