# Outline

## Deep Content: Unveiling Video Streaming Content From Encrypted Wi-Fi Traffic -  NCA 2018

Created by : Mr Dk.

2019 / 02 / 05 17:17

Ningbo, Zhejiang, China

---

### Introduction

目前全球超过 _50%_ 的 _Internet_ 通信是端到端加密的

__HTTPS__ 已经成为了很多互联网通信的准则

* 消除了利用流量分析进行入侵检测的可能性

目前主要的研究兴趣：从加密的通信数据流中进行推断和预测

之前的研究中，捕获的数据包位于 _IP_ 层（网络层）

尽管信息内容被加密，但一些元数据依旧可以揭露加密后的通信行为

* Packet length
* Inter-packet time
* Burst size
* Burst interval
* IP address
* Ports
* TLS Header

在本文中，作者试图通过被动监听 _Wi-Fi_ 流量的方式

对在 __传输层__ （TLS） 和 __链路层__ （WPA2）双重加密的流量进行推断

特别的，本文专注于识别来自一个已知在线视频集合的流量

用途：网络保护应用、反情报应用、态势感知应用

* 刚需：认证已知的特定视频是否被特定个体或在特定地点被观看

本文的贡献：

* 通过构造基于深度学习的分类器，证明了从加密的 _Wi-Fi_ 流量中识别 _YouTube_ 上一个封闭集合的视频流
* 利用多层次感知（MLP）架构，通过从 _Wi-Fi_ 层纯被动采集的数据，对 _10_ 个视频的集合进行识别，达到了 _97%_ 的准确率
* 将提出的模型与最先进的 _CNN_ 在网络层上做相同任务进行比较，论文证明了 _CNN_ 对 _Wi-Fi_ 通信的预测也是适合的，但是需要更多的 _CPU_ 和时间资源（大约三倍）
* 在两周的时间后再次做了相同实验，本文的分类器依旧维持了相同的准确率水平

---

### Related Work

之前的研究证实了深度学习对流量推测的可用性

并评估了经典深度学习算法的性能

* Recurrent Neural Networks
* Convolutional Neural Networks
* Deep Neural Networks

使用 _KDD cup_ 或 _NSL-KDD cup_ 数据集

_Soft-max regression + Sparse autoencoder_

使用 _Long Short-Term Memory recurrent neural networks_

以上技术得到了高 _True Positive_ 的结果，但也带来了高 _False Positive_

上面的技术基于端到端通信流的完整知识进行入侵检测

* 流的长度
* 数据包总数

在提出的大部分解决方案中，作者要么依赖通信流的完整追踪，要么依赖每一个流的摘要

相比之下，本文没有使用统计学的知识识别已知视频，因此不需要采集完整的流用于预测

在最近的研究中，使用 _CNN_ 不仅能够检测通信的类型，还能检测加密通信的内容

* 作者能够使用一些暂时性的通信特征，识别在 _HTTPS_ 协议下下载的视频，并达到了很高的 _accuracy_

还有使用 _CNN_ 对一个给定流的数据包序列的特征进行识别

最近的研究已经在没有使用深度学习算法的条件下，成功识别了加密通信中的视频流

相比之下，本文提出的视频识别方法：

* 不依赖视频通信、_TCP/IP_ 流信息、完整的流统计信息的数学模型
* 在无线网络环境中使用更少的资源需求
* 达到 `97%` 的 _accuracy_

---

### Experiment Setup

#### DASH Streaming

_Internet_ 上的视频流通常被称为 _HTTP-based Adaptive Streaming, HAS_

工作方式：

将原始内容，根据不同的 __比特率__ 和 __分辨率__，编码为多个版本

每一个版本的视频被分为较小的段（2—4秒），保存在 _WEB server_ 上

视频流客户端使用 _HAS_ 算法，通过 _HTTP GET_ 访问服务器

由于 _HTTP_ 运行在 _TLS_ 层上，包括 _HTTP Header_ 在内的所有数据被端到端加密

客户端的视频播放器 -  _Dynamic Adaptive Streaming over HTTP, DASH_ player 利用服务器的 _manifest file_ 和网络状态动态适配视频流

因此，_DASH_ 播放器的视频流产生的通信波形：

* 周期性的下载峰值
* 波峰的不同量级（根据网络条件）

__波峰__ 与下载视频的下一序列的数据块有关

波峰之间的等待时间是 - 用户观看到已下载数据块的特定百分比的等待时间

_DASH_ 播放器通过请求不同质量的数据块来适配不同的网络环境

#### Data Collection

一台 _Laptop_ 用于从 _YouTube_ 上反复下载相同的 `10` 个视频

一台 _Laptop_ 被动捕获所有的无线网络数据包

对于每个视频，只捕获视频流最开始的三分钟

此外，使用了 _YouTube Red Account_ 用于避免广告

---

### Deep Content Methodology

#### Preprocessing & Feature Engineering

##### Data Filter

假设数据包被 _WPA2_ 加密，是不可能提取出 _Layer 3_ 及其上层的协议信息的

因此主要获取 _MAC_ 层的参数：

* Frame size
* Frame type
* Frame duration time
* Signal strength
* Noise level
* MAC address of source and destination

双向通信数据流的相同参数

##### Frame Type Identification

在 _IEEE 802.11_ 标准中，_data frames_ 是和视频分类最相关的

只捕获 _frame size_ ≥ 有线网络中 _minimum packet size_ 的 _data frames_

##### Feature Engineering

捕获的数据包含每一个视频流的前三分钟，在之后被分组为：

* Up-link frames
* Down-link frames
* Combination frames

对每一个组，为了方便计算，分为 `0.36s` 的 `500` 个 _bin_

特征从这些 _bin_ 的滑动窗口中产生

对于每一个暂时的 _bin_，计算出描绘流量动态信息的特征：

* Number of packets in Data frames
* Number of bytes in Data frames
* Number of packets in Management frames & Control frames
* Number of bytes in Management frames & Control frames

此外，还有不同视频的 _MPEG-DASH_ 视频流流量波形

还有四个额外特征：

* Minimum size of packets
* Maximum size of packets
* Average size of packets
* Variance of packet size

#### Classifier Architectures

使用了三种神经网络架构

打乱了捕获到的流量，并拆分为 `80%` 的 Training Set 和 `20%` 的 Testing Set

##### Convolutional Neural Network (CNN) Model

作为参考架构

* `1` - Input Layer
* `3` - Convolution Layers
* `1` - Max Pooling Layer
* `2` - Fully Connected Layers

Adam Optimiser - mini-batches with 64 samples

##### Long Short-Term Memory (LSTM) Model

视频的流量波峰分布服从于一个特定的时间序列

这暗示了特征值的时间相关性至关重要

因此使用 _RNN_，因为此模型训练时间序列数据的优越性

利用 _LSTM_ 模型解决经典 _RNN_ 中的消失梯度问题

LSTM 网络的输入是固定尺寸的 `500 × 1` array

* `500` - time steps
* `1` - single feature

Architecture

* `2` hidden layers
* `32` neural nodes each hidden layer

##### Multi-Layer Perceptron (MLP) Model

对于每个特征，输入是一个固定尺寸为 `500` 的数组

数组将通过一堆 _Fully-Connected (FC) Layers_

通过实验：

* `2` hidden layers
  * `300` neural nodes in the first hidden layer
  * `100` neural nodes in the second hidden layer
* `1` dropout layer

分类的准确性最高

利用 Adam Optimiser on batches of 64 samples

Categorical cross-entropy 作为损耗函数

---

### Results

#### Classification Performance  on CNN Model

输入是 `500 × 1` 的 array：

* `500` - number of temporal bins of `0.36s`
* `1` - one feature

每个特征一个一个地被用于训练

与有线网络数据相比，本文中采集到的数据中有更多的 noise

但模型的 accuracy 表现出了对这些 noise 的 robustness

#### Classification Performance on LSTM Model

每次只选择一个特征进行训练

#### Classification Performance on MLP Model

视频流以 `500 × 1` 的 array 的形式，只选中一个特征，输入 MLP Model

为了选择最佳的 MLP configuration：

* Feature selection
* Number of hidden layers
* Number of nodes in each hidden layer
* Choice of activation function

由经验主义和实验结果共同确定

#### Result Analysis

直观上看，`Number of packets in Data Frames` 这一特征具有最好的分类性能

* 因为视频内容被封装在 Data Frames 中
* 这一特征随着时间的波动是区分视频的重要特征

非 Data Frames 也能被用来准确地区分视频流

不仅仅是视频内容本身带来的信息，服务器与客户端之间的交互信息也携带了信息

_MLP_ 模型比 _CNN_ 模型至少快了三倍

#### Summary

作者将视频流中提取的流量信息表示为类波形信号

将波形编码为具有两个 hidden layers 的 MLP Model

这个模型作为解码器应用到 Testing Set 上用于识别视频流

与其余技术相比，在节约了三倍训练资源的同时，达到了 `97%` 的 accuracy

---

### Summary

这是老师让我接触的新研究方向

主要还是对这些深度学习模型的理解上有些空白

不知道这些模型之间有什么差别、参数有什么意义

---

