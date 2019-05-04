# Outline

## PatternListener: Cracking Android Pattern Lock Using Acoustic Signals -  CCS 2018

Created by : Mr Dk.

2019 / 02 / 12 19:09

Ningbo, Zhejiang, China

---

### Introduction

__模式锁__ 在移动设备上被广泛用于认证用户

许多的安全机制保证，当用户绘制图形模式时，屏幕不会被其它应用捕获

但应用依旧能够获得设备其它资源的使用权限

* 加速器
* 相机
* 麦克风
* GPS

现在有很多攻击，通过捕获输入 PIN 时的特征，破解 PIN 码

* 比如安装恶意软件，利用动作传感器识别按压的位置
  * 按压屏幕的力度、设备方向的改变会显著影响攻击准确率
* 利用麦克风检测输入事件、利用前置摄像头估计手机的方向改变
  * 推断出受害者按压的数字位置
* 利用 Wi-Fi 的 CSI 数据推断
  * 容易受到周边物体的影响

由于模式锁比键盘输入有更少的扰动，因此利用侧信道推断模式更具有挑战性

* 利用无线信号推断
  * 需要复杂的网络设置
  * 易受移动物体的影响
* 利用记录受害者手指移动信息的视频信息推断
  * 需要在物理上足够接近受害者

本文提出了一种新式的声学攻击，利用察觉不到的声学信号推断模式

本文的贡献：

1. 揭露了模式锁的新威胁
2. 提出了利用声学信号破解 Android 模式锁的攻击方式
3. 提出了几个算法，用于分析从手指反射回的声学信号，推断模式
4. 实现了攻击原型，在五次尝试内成功破解 `90%` 的模式

---

### Cracking Pattern Lock

#### Android Pattern Lock

认证方式：在一个 `3*3` 的网格上绘制一个模式

模式锁在 Android 生态圈中被广泛使用

#### Threat Model

产生并播放察觉不到的声音信息

与此同时，受害者设备的麦克风记录从手指处返回的声学信号

分析记录的信号，根据手指活动重建模式

为了发动攻击，恶意软件需要获得 __扬声器__、__麦克风__、__运动传感器__ 以及 __网络__ 权限

大部分权限可以在未经用户同意的前提下被授予，除了 __麦克风__

然而在 Android APP 中，麦克风权限是很普遍的

因此恶意软件 _Pattern Listener_ 可以轻易获得这些权限

在破解模式后，只要攻击者有机会在物理上接近设备，就可以完成攻击

#### Overview of Pattern Listener

* Unlock Detection
  * 检测受害者何时绘制解锁图案
  * Pattern Listener 能够立刻播放声音并记录反射的声学信号
* Audio Capturing
  * 一旦检测到解锁动作，利用扬声器播放产生的声学信号
  * 触发麦克风记录声学信号，捕获解锁期间屏幕上的手指移动
  * 信号被上传至服务器
* Pre-processing
  * 提取对应于手指动作的声学信号
  * 利用 __相干检测__ 解调基带信号，降低取样信号，用于有效的信号处理
  * 移除静态噪声，获得反射回的真实声学信号
* Pattern Reconstruction
  * 分析信号，获得指尖在屏幕上绘制的轨迹
  * 根据轨迹，恢复轨迹的每一条线
  * 通过线来推断候选模式
  * 包含四个步骤：
    * Signal Segmentation
    * Relative Movement Measurement
    * Pattern Line Inference
    * Candidate Patterns Generation

---

### Pattern Listener Design

#### Unlocking Detection

目标：检测受害者何时绘制解锁模式

* Screen Unlock
  * 屏幕状态：
    * __non-interactive__ - 设备正处于睡眠
    * __pre-interactive__ - 屏幕正在工作，用户正在唤醒设备
    * __interactive__ - 设备完全开始工作，用户能够通过屏幕与设备交互
  * 在 Android 中，在状态变化时，屏幕状态信息将会被自动广播
  * 通过监控 __non-interactive &rarr; pre-interactive__ 的广播信息，检测解锁行为
* App Unlock
  * 用户经常在屏幕上 __左右滑动__，寻找 APP
  * 用户 __点击__ 以选择一个 APP
  * 等待几秒钟的 APP 启动延时
  * 利用运动传感器检测 __点击__
  * 利用声学信号检测 __滑动__ 动作

Pattern Listener 不需要在每次用户解锁的时候都捕获信息，只需要一次就够了

受害者会在一天内多次解锁手机，因此解锁行为的检测是比较容易的

#### Audio Capturing

利用声学信号重建解锁模式的原因：可以从反射的声学信号中提取手指动作

##### Audio Play with the Speaker

产生的声音是连续的声学信号波 - `Asin2πft`

* 大部分扬声器和麦克风的反应频率在 `50Hz` - `20kHz` 之间
* 大部分人不能听见 `18kHz` 以上的声音

因此，产生的波的频率范围是 `18kHz` - `20kHz`

可能有些人能够听到 `18kHz` 以上的声音，但可以通过降低音量的方式使其不被察觉

同时，环境噪声在 `18Hz` 以上的频率中变得可以忽略，因此 Pattern Listener 不会被噪声影响

##### Audio Record with the Microphone

手指的动作将会显著干扰声学信号

##### Valid Signal Identification

解锁过程开始于手指触碰屏幕，结束于手指离开屏幕

运动传感器可被用于检测这两个关键时间点

这两个时间点之间的声学信号将被悄悄上传至服务器进行分析

#### Audio Preprocessing

##### Coherent Detection

扬声器发出的声音可被视为 __载波信号__

相关于手指动作的信号可被视为 __基带信号__

因此，麦克风记录的声学信号是载波信号和基带信号的叠加

可以用传统的相干检测，从记录的信号中解调基带信号

经过低通滤波、降低取样，可以得到基带信号同相和正交的分量

由于发出的声学信号的振幅是常量

因此在没有手指运动时，C/O 的波形是一条平线

当指尖在屏幕上移动时，C/O 的波形会波动

##### Static Components Removal

大部分的声学噪声信号是静态的，可对其进行过滤

利用了 _Local Extreme Value Detection (LEVD)_ 算法

#### Signal Segmentation

为了重建解锁模式，首先需要识别组成模式的每一条线

因此该阶段的目的是将 C/O 部分分段，对应模式中的每一条线

提出了 _Turning Points Identification (TPI)_ 算法，实现了自动化的信号分段

在解锁过程中，一条新的线开始于指尖的转动

指尖转动的点成为 _turning point_

* 只要知道每个 _turning point_ 的时间，就能将 C/O 信号分成对应每条线的段

如何识别 _turning point_ ？

* 通过观察发现，当到达 _turning point_ 时，指尖会暂停一段时间
* 因此，在 _turning point_ 反射的声学信号会相对稳定

如果发现两个相邻的极点之间的时间间隔比平均时间间隔大很多，这个点就是 _turning point_

#### Relative Movement Measurement

##### Startpoint and Endpoint Re-identifiation

##### Relative Movement of the Fingertip

利用基于相位的方法，衡量手指的相对移动

* 计算反射的声学信号的相位差
* 将相位差转换为路径长度差

指尖移动会同时影响反射信号的频率和相位

* 频率由移动速度影响
* 相位由移动距离和方向影响

对于同一个模式的移动距离和方向是不会改变的，而移动速度会变化

因此采用了基于相位的方法提取手指移动特征

根据公式，可以获得在任何时间间隔内的路径差（由指尖移动决定）

#### Pattern Line Inference

利用路径长度差来推断组成模式的每一条线

* 首先描绘滑动指尖的运动特征
* 利用相似性度量推断每一条线

##### Movement Feature Extraction

两种情况：

* 扬声器和麦克风在不同侧
* 扬声器和麦克风在同一侧

使用一个二维向量 `(d1, d2)` 表示指尖的移动特性

Pattern Listener 至少需要一对扬声器和麦克风工作

如果有多对扬声器、麦克风工作，效果会更好

为了防止多对扬声器、麦克风之间的干扰

扬声器可产生不同频率的声学信号

##### Similarity based Line Inference

建立一个数据库，存放每一对扬声器、麦克风每一条线的特征向量

对于一个 `3*3` 的网格

给定一个起点，可以滑向另外八个点，构造八条不同的线

对比它们的特征向量，计算对应的相似度

将多对扬声器、麦克风的测量结果按权重叠加，得到相似度

如果相似度大于阈值，则将这条线作为候选线

可能存在多个候选线的情况，都需要列举出来

#### Candidate Patterns Generation

构造一个模式树

* 用于模式重建
* 根据模式都应该是相连的事实，过滤不可能的候选结果
* 根结点是模式的起始点；如果有多个候选起始点，则产生多个模式树
* 将第一条线的候选线加在模式树的第一层上，依次类推
* 每条边的权重是对应候选线的相似度

每条从根结点到最后一层叶子结点的路径都是一个候选模式

* 有些树枝没有到达最后一层 - 在上一条线之后，找不到合适的下一条候选线了

候选模式的相似度被定义为：路径上所有线的平均相似度

相似度最高的 `5` 个模式被选为最终的候选解锁模式

---

### System Evaluation

#### Experiment Setup

对于每一个手机型号，扬声器和麦克风的位置是固定的

需要对每一个手机型号建立对应的 ground-truth database

只需要提取所有可能的线（`8*9` 条）的特征；不需要提取所有可能的模式（`389112` 个）

Android 在 `5` 次失败的模式尝试后会锁定设备 `30` 秒

因此，在实验中，成功定义为在五次尝试之内推断出了解锁模式

#### Experimental Results

##### Overall Success Rate

_Sample_ - 对应一次解锁过程的声学信号

捕获多个样本有利于提高成功率

##### Impact of Pattern Complexity

定义模式的复杂度为：模式中线的条数

观察到对于较为复杂的模式，破解成功率提高

原因：线越多的模式，包含的指尖动作特征越多

##### Impact of Gesture

不同的人有不同握持手机的习惯和解锁习惯

Pattern Listener 在不同的握持手势下有相对较好的鲁棒性

##### Impact of Drawing Speed

* 中速 - 效果最好
* 低速 - 信号采集过程中可能会带来错误
* 高速 - 声学信号分段可能不是特别准确

Pattern Listener 对绘制速度的改变有相对较好的鲁棒性，尤其是在多次尝试之后

##### Impact of Surrounding Objects

利用周围的物体很难干扰 Pattern Listener

##### Impact of Smartphone Models and Noise

大部分噪声的频率很低

屏幕较小的手机的成功率较低，因为候选线更难识别

##### Stealthiness

CPU 占用率、电池消耗、网络传输水平都比较低，隐蔽性较好

---

### Countermeasurements

#### Preventing Usage of Microphone in Background

不过这种对策会影响其它 APP 的可用性

将程序作为 Android 的守护程序运行

#### Random Layout of Pattern Grids

影响用户体验 - 用户需要在每次解锁前，在随机的布局中找到每一个点

---

### Summary

太偏物理层了...我佛了

为什么最近读的论文都这么偏向物理层

仿佛读的是电子信息而不是计算机

---

