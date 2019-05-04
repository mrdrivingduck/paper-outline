# Outline

## CETAD: Detecting Evil Twin Access Point Attacks in Wireless Hotspots - CNS 2014

Created by : Mr Dk.

2019 / 05 / 01 11:42

Ningbo, Zhejiang, China

---

## Introduction

移动热点在生活中为人们带来了方便

* 提供便捷的网络接入
* 吸引用户

但存在安全性问题

热点提供方通常会将责任推到用户身上 - _Terms and Conditions_

因此，在客户端需要有 Evil Twin AP 的检测机制

已有的检测机制不够实用：

* 使用额外的硬件
* 需要预先训练

最好只需要在客户端设备上安装软件

客户端检测方式的挑战：

* 客户端具有受限资源 - 无法预先获知网络结构
* 热点可能应用多种 Wi-Fi 设置 - 机制应当能够在所有设置场合中使用
* 使用定制的硬件 - 限制了实用性

---

## Background

客户端设备通过选择 SSID 与 AP 连接

如果多个 AP 使用相同的 SSID

客户端设备关联到 RSSI 最强的 AP 上

一旦关联，客户端设备从热点上通过 DHCP 取得网络参数

### Hotspot Architecture

#### Single AP Architecture

最简单的架构，单个 AP 直连路由器

#### Multiple AP Architecture

多个 AP 连接到同一个路由器，构成一个 _Extend Service Set (ESS)_

这些 AP 通常使用类似的配置

使客户端设备可以根据 RSSI 强度在 AP 之间平稳切换

> 有些 AP 可以被配置为 _Wireless Distribution System (WDS)_ 模式。只有一个 AP 连接到路由器，其余 AP 与该 AP 通过无线信道进行通信。（暂未被标准化）

### Automatic Network Configuration

客户端设备广播 _DHCP DISCOVER_

每个 DHCP 服务器回复 _DHCP OFFER_，其中包含：

* Host IP Address
* Subnet Mask
* Router IP Address
* DNS Servers
* Server Identifier

客户端设备选择其中一个 offer，广播 _DHCP ACK_

---

## Evil Twin Attacks

### Launching Evil Twin AP Attacks

#### Attack Using Mobile Internet Access

_Mobi Attacks_ - 使用了蜂窝网络的网络连接（比如智能手机开热点）

#### Attack Utilizing the Victim AP's Internet Access

_Multihop Attacks_ - 攻击者连接到一个受害 AP 上，并作为一个 AP 共享其连接

##### Single Wi-Fi Interface (Si-Fi Attack)

通过软件将同一张物理网卡虚拟成两张网卡

* 网卡 A 用于连接合法 AP
* 网卡 B 用于伪造 AP

##### Dual Wi-Fi Interfaces (Du-Fi Attack)

使用外置的 USB 网卡实现上述功能

### The Consequence of Evil Twin AP Attacks

通过用户的自动登录窃取 cookie

### Threat Model

* 控制发包时间
* 用本地伪造报文响应用户
* 修改报文内容

---

## CETAD Overview

### Design Requirements

* 不需要任何管理权限（纯客户端）
* 不需要任何定制基础设施的支持（比如安装 sensor）
* 不能使用只有在手机上存在的 sensor 或 interface（保证通用性）
* 自动化

### Hotspot Features

经过观察，大部分热点符合如下特征：

* 支持 DHCP 服务
* 不适用 WDS 架构
* 攻击者不能获得网络管理权限，但可以连接到网络
* 热点使用单一 ISP

### CETAD Framework Overview

场景：

* No Attack - 所有合法 AP 使用相同的 ISP 和 IP 地址，相似的 RTT 值
* Mobi Attacks - 合法 AP 和 Evil Twin AP 的 ISP、公网 IP 不同
* Multihop Attacks - ISP 信息相同，但由于多了一跳，RTT 值不同

#### Secure Data Collection Phase

扫描可用的 AP 和 SSID

根据一定阈值过滤掉信号强度较弱的 AP

1. 与 AP 关联，采集 DHCP 信息
2. 通过与公共服务器进行通信，采集 ISP 信息
3. 通过与公共服务器建立 HTTPS 连接，采集多个 RTT 数值

#### Detection Phase

##### ISP-based Scheme

使用 ISP 信息来判断两个 AP 是否使用相同的 ISP

足够用于检测 _Mobi Attacks_

##### Timing-based Scheme

使用 RTT 值来检测 _Si-Fi Attacks_ 和 _Du-Fi Attacks_

### Observations

ISP 信息包含：

* 公网 IP 地址
* zip code
* ISP name

RTT：

重新定义为，与公网服务器建立 HTTPS 的时间

#### Network Analysis

* 大部分热点有 2-3 个 AP
* 每个热点都支持 DHCP，并连接到同一个 ISP
* 一个热点中的所有 AP 采用类似的配置
* 合法 AP 和 Evil Twin AP 的 ISP 信息不同
* 没有热点使用 WDS 架构

#### Timing Analysis

##### No Attack Scenario

RTT 会由于网络负载、用户数量等原因而变动

同一热点的不同 AP 的 RTT 值接近

##### Attack Scenario

对于 _Si-Fi Attacks_ 和 _Du-Fi Attacks_

平均 RTT 值远高于无攻击场景

* 对于 _Si-Fi Attacks_ - 软件处理延时 + 信道碰撞延时（使用同一物理网卡，同一信道）
* 对于 _Du-Fi Attacks_ - 报文转发延时

---

## CETAD Description

### Secure Data Collection

### Collecting Data

某个 IP 地址的 ISP 信息是公开可见的

可以通过 HTTPS 采集到

CETAD 测量 _Client Hello_ 到 _HTTP Response_ 的时间差作为 RTT

### Detection Phase

#### ISP-based Detection

为了提供互联网服务

一个热点必须有一个具有公网 IP 的网关

在合法热点中

合法 AP 连接到相同路由器，使用相同的公网 IP

CETAD 使用这个指标检测 _Mobi Attacks_

#### Timing-based Detection

相同热点的相同 AP 的 RTT 在某一短暂时间内类似

使用两种方法进行检测：

##### Unsupervised Clustering

由于 RTT 值可能偏差很大

基于简单的阈值可能效果不佳

使用 _Mean Shift Clustering (MSC)_ 算法进行聚类

在聚类之前，先将偏差过大的噪声根据平均值和标准差过滤

MSC 算法会返回一个或多个类

##### Standard Deviation Analysis

在攻击场景中，RTT 的波动非常大

在合法 AP 之间，RTT 的值类似

因此只需要计算标准差，并判断是否大于一定阈值，即可判断

##### Combined Detection Technique

将两种方法组合进行判断

### Security Analysis

HTTPS 保证了信息的不可篡改性

#### No Attacks

合法 AP 的 ISP 信息和 RTT 类似

但当合法 AP 的 RTT 变动剧烈时，可能产生误报

#### Mobi Attacks

基于 ISP 信息的检测保证了攻击者无法使用自己的网络进行攻击

#### Multihop Attacks

基于时间的检测能够检测出攻击者无法控制的 RTT 相似性

* 转发延时
* 软硬件处理延时
* 信道碰撞延时

---

## Summary

又是挺早之前的论文了

方法挺简单的 但是很实用

读完这篇突然被启发到

有些聚类算法可以只输出一个类的

不像 K-Means 那样必须输出指定数量个类

这么一想 好像可以在毕设论文中套用类似的算法了

---



