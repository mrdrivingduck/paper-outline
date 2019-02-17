# Outline

## A Video-based Attack for Android Pattern Lock -  ACM Trans. Priv. Sec. 2018

Created by : Mr Dk.

2019 / 02 / 18 01:20

Ningbo, Zhejiang, China

---

### Introduction

在 Android 系统中，pattern lock 使用广泛

心理学研究表明

相比数字和字母，人脑更擅长回忆视觉信息

之前针对 pattern lock 的攻击都依赖于过于强大的假设

在现实中不可能实现

最近，基于视频的分析在重建 PIN 或文字密码显示出了有效性

早期的攻击依赖于利用相机直接记录正对屏幕或键盘的视频片段

而没有利用侧信道攻击的范例

攻击者应当能够将用户的指尖动作映射到图形结构

并将图形结构翻译为解锁模式

本文贡献：

* 第一个提出通过视频侧信道采用计算机视觉算法重建 Android 模式锁
* 提出了一种新的威胁
  * 录像距离可以在两米之外
  * 相机不需要正对目标设备
* 反直觉的发现 - 复杂的 pattern 更容易遭受威胁
* 新的对策 - 小幅改动 pattern lock

---

### Android Pattern Lock

用户需要在 pattern 网格上绘制提前定义好的连接点序列

Android pattern 的创建规则：

1. 一个 pattern 必须至少包含四个点
2. 每个点只能被访问一次
3. 一个之前没有被访问过的点如果属于水平、垂直或对角线段的一部分，则将被访问

据统计，根据以上约束，`3*3` 的网格共有 `389112` 种模式

暴力破解不可能 - 如果连续失败五次，Android 设备将被自动锁定

---

### Threat Model

这种攻击通常由可以从物理上接近目标设备的攻击者发动

为了快速使用这些设备，攻击者需要提前获得设备的解锁 pattern

攻击开始于对用户解锁过程进行录像

* 录像可以在两米之外进行 - 记录的视频看不清屏幕上的内容
* 相机不需要直接对准目标设备

因此这种攻击很难被发现

假设：

* 视频中能够看到用户的指尖和设备的局部
* 攻击者已经知道网格的布局 - `3*3` or `6*6`

---

### Overview of our Attacking System

系统输入记录整个解锁过程的视频片段

输出几个候选的 pattern 用于在目标设备上进行测试

#### Video Filming and Preprocessing

对解锁过程进行录像（2-3米之外）

系统自动剪辑出包含完整解锁过程的录像片段

攻击者需要在视频帧中标记出两个区域：

1. 用于绘制 pattern 的指尖
2. 设备的局部

#### Track Fingertip Locations

两个区域被标记后，计算机视觉算法将会定位每一帧中的指尖位置

算法将追踪的指尖位置聚合，得到指尖移动轨迹

此时的追踪轨迹是相机视角的

#### Filming Angle Transformation

将追踪到的指尖移动轨迹，从相机视角转换为用户视角

使用 __边缘探测算法__ 自动计算录像角度

得到用户视角的指尖移动轨迹

#### Identify and Rank Candidate Patterns

自动将指尖移动轨迹映射为几个候选 pattern

基于启发式算法，对这些候选 pattern 进行打分

大部分 pattern 最终只能剩下不到五个候选 pattern 用于测试

#### Light Weight Trials

攻击者在目标设备上依次尝试候选 pattern

---

### Implementation

#### Video Preprocessing

目标：从视频片段中识别解锁过程

其实是一个可以手动完成的过程

但本文采用了启发式的自动化过程

观察到：

1. 在解锁前或解锁后，用户通常会暂停几秒
2. 两个连续的屏幕操作 - swiping/zooming

首先过滤一些指尖位置在 `1.5s` 内没有改变的屏幕操作

再利用解锁前后的暂停时间进行识别

如果识别出多个解锁过程，则询问用户确认使用哪一个片段

攻击者也可以手工剪辑合适的解锁过程

#### Track Fingertip Locations

部署视频追踪算法 - _Tracking-Learning-Detecting (TLD)_ 算法

* 这个算法能自动检测边界盒中定义的物体
* 在此场景中，该算法追踪指尖的移动

在视频的第一帧中，指尖和设备局部的位置被标记

##### Generate the Fingertip Movement Trajectory

对于每一个追踪的物体，_TLD_ 算法会给出一个 `0` 到 `1` 之间的置信度

如果置信度大于阈值，则追踪被认为是成功的

阈值设定为 `0.5`

TLD 算法包含三个模块：

1. tracker - 在连续的帧中跟随物体
2. detector - 扫描每一个独立帧，定位物体的所有出现位置
3. learner - 估计 detector 的误差，对 detector 进行更新，避免误差出现在将来的帧中

在感兴趣区域选择不佳的情况下，算法可能无法检测物体

* 系统会让用户重新选择追踪区域

同时还要记录每个指尖位置的时间信息

* 用于之后重叠线段的检测

##### Camera Shake Calibration

默认情况下，TLD 算法输出相对视频帧左上角的追踪物体位置

然而，拍摄可能产生抖动

因此某一帧左上角的位置可能在后面的帧中处于不同位置

严重影响了指尖定位的准确性

本文中的方法：记录指尖相对于目标设备某个固定位置的坐标

因此追踪指尖和设备两个区域

算法输出两个区域中心点的相对坐标

#### Filming Angle Transformation

为了避免引起怀疑，录像设备不是直接面对目标设备的

因此，追踪算法产生的指尖移动轨迹和实际的 pattern 不太一样

必须将相机视角的指尖移动轨迹转换为用户视角

首先需要估计相机和目标设备的角度

使用边缘检测算法 _Line Segment Detector (LSD)_ 检测设备的长边

摄影角度是检测出的边线和垂直线的夹角

对于每一帧，算法分别计算摄影角度并进行变换

* 因为摄影角度会随着视频变化

#### Identify and Rank Candidate Patterns

将指尖移动轨迹映射到几个候选 patterns 中

目标：包含尽可能多的 patterns，只选择最有可能的几个 patterns 用于测试

使用轨迹的几何信息来过滤不满足约束的 patterns

* 线段的长度和方向
* 转折点的个数

##### Extracting Structure Information

Pattern 可被定义为一些线段的集合

每个线段有两个特性

* 长度
* 方向

定义 pattern `P = {L, D}`，`L = {l1, l2, ..., ln}`，`D = {d1, d2, ..., dn}`

##### Identify Line Segments

从轨迹中识别出独立的线段

* 通过寻找转折点识别线段（两点确定一个线段）
* 通过线性适应算法寻找转折点

挑战：如何区分重叠的线段

* 使用之前记录的时间信息

##### Extract the Line Length

线段的物理长度由屏幕的尺寸和 pattern 网格，以及两个触控点之间的距离决定

为了保证方法的设备独立性

将物理长度规范化为轨迹中最短线段的长度

##### Extract Direction Information

在 `3*3` 的网格中，共有 `16` 种可能的方向

计算指尖移动的轨迹和水平线的斜率

并与可能的方向进行对比，决定线段的方向

#### Map the Tracked Trajectory to Candidate Patterns

##### Identify Candidate Patterns

从最上角的触控点开始，列举所有可能的 patterns

过滤不符合约束条件的 patterns

##### Rank Patterns

使用启发式的算法给候选 pattern 打分

假设开始于左上角的 pattern 更有可能是正确 pattern

如果 pattern 从同一个点开始，总长度更长的 pattern 更有可能是正确 pattern

......

最终对于 `3*3` 的网格产生不超过 `5` 个候选 pattern

测试顺序对成功率几乎没有影响

---

### Experiment Setup

#### Data Collection

#### Pattern Complexity Classification

将 pattern 的复杂度量化

* 连接触控点的个数
* 所有线段的总长度
* 线段交叉点个数
* 重叠线段个数

##### Pattern Grouping

根据评分，将 patterns 分为简单、中等、复杂三类

#### Video Recording and Preprocessing

---

### Experimental Results

#### Overall Success Rate

在 `5` 次尝试之内，成功破解 `95%` 的 pattern

复杂的 pattern 比简单的 pattern 更不安全

#### Impact of Filming Distances

在 `2.5m` 以内拍摄，成功破解 `80%` 以上的 pattern

#### Impact of Camera Shake

本文提出的方法可以忍受一定程度的相机抖动

#### Impact of Lighting Conditions

低亮度对攻击成功率有消极影响

#### Impact of Filming Angle Estimation

当角度估计误差在 `5` 度以内时，攻击效果较好

攻击中的算法能够给出精确角度

误差只是理论探讨

#### Evaluation on Other Pattern Grids

点数更多的网格能够提供更强的保护，但本文的方法依旧能破解大部分 patterns

#### Impact of Different Camera Brands

不同品牌的相机对攻击成功率几乎没有影响

#### Guessing Patterns with Eyes

本文的攻击方法比直接用肉眼观察更容易成功

#### Attacking PIN-based Passwords

对本文的方法进行改进，可在 `5` 次尝试之内破解 `85%` 的 PIN 密码

PIN 的特征与 pattern lock 不太一样

#### Limited Study

当设备在视频中无法被看见时，攻击成功率较低

当指尖没有出现在视频中时，攻击成功率低

---

### Countermeasurements

#### Possible Countermeasurements

* 随机打乱触控点的位置
  * 带来低使用性的副作用
* 动态改变屏幕的颜色和亮度
* 使用户在绘制 pattern 时完全遮住指尖
* 将解锁操作与其它屏幕动作混合

#### A Feasible Remedy

在使用性和安全性之间找到折衷

1. 在绘制 pattern 时，可以跳过水平、垂直或对角线中间的点
2. 在绘制正确的 pattern 之前，用于需要绘制一个随机生成的 pattern
3. 放松规则，pattern 中的点可以被多次访问

攻击很难识别中间被跳过的点

攻击者很难识别哪一部分追踪轨迹是真正的 pattern

---

### Summary

听起来挺邪乎的

读完论文就明白了

好像也就是那么回事

但我感觉我做不出这个实验

我觉得这个问题也不是几个人就研究得透的

应该是团队的力量

---

