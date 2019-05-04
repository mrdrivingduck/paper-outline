# Outline

## ETGuard: Detecting D2D Attacks Using Wireless Evil Twins - CS 2019

Created by : Mr Dk.

2019 / 05 / 04 10:09

Nanjing, Jiangsu, China

---

## Introduction

Android 是世界上最流行的操作系统之一

* `20` 亿活跃用户量
* `87.5%` 的市场份额

Android 因为以下三个原因，成为黑客和恶意软件的攻击目标：

1. 大量的用户基础，需要他们作出一些安全性决定
2. 开发 Android 应用较为简单
3. Android 的安全设想使其容易被攻击

Evil Twin 能够将安装 APP 的请求重定向到恶意 apk 文件上

本文的贡献：

* Evil Twin 版本的实用性 D2D 攻击
* 基于指纹的 pre-association ET 检测方案
* 利用 beacon frames 检测 ET
* 实现了 ETGurad 原型

---

## Threat Model and Assumptions

传统的攻击方式：

* 用户流量嗅探
* MITM Attack

只能当用户关联到 ET 并产生流量时才能被检测到

本文提出一个在产生流量之前造成破坏的攻击场景

### Invoking Malicious Component

攻击者将恶意服务重新打包到良性 APP 中

并发送额外收费的短信

这个服务不会被 APP 的任何组件调用

所以能够通过应用商店的检测

#### Assumptions

假设无线网络 _CSE_ 满足以下特征：

* 使用 portal 设置接入互联网
* 不使用任何加密
* 目标 Android 设备安装了加入恶意服务的 APP
* 网络中不存在 NAT，也不被防火墙保护

#### Attack Scenario

* 使 ET 也使用 portal 的网络接入方式，伪造 SSID、BSSID、beacon frame
* ET 被放置在比合法 AP 信号更强的位置上，并对合法 AP 进行解除认证攻击
* 一旦用户自动重连到 ET，发送登录到网络的通知
* 用户被重定向到伪造的 portal 页面，并填写账号、密码
* 用户按下提交按钮后，恶意服务启动，ET 从网络中消失
* ET 侵入了网络，损害了设备，并不会留下任何痕迹

如果恶意服务悄悄打开了一个端口

* 可以执行 `netstat` 命令
* 读取 `/proc/<pid>/net/tcp` 文件，识别设备所有的开放端口
* 并发动更深层的攻击

通过在 `href` 属性中设置 `intent` 来触发恶意服务

其余的 ET 攻击方式：

* Karma Attack - 嗅探 probe request，根据请求参数来创建对应的 AP
* Catch-All-Evil-Twin Attack - 建立相同 SSID 下所有配置的 ET
* DNS Spoofing - 将用户请求替换为网页
* SSL Stripping - 将 HTTPS 链路转换为 HTTP 链路

没有一种攻击可以在用户连接上网之前损害用户

或损害设备 或造成经济损失

但在上述攻击场景中

当用户点击了 LOGIN 按钮

调用了一个已经存在的 APP 的恶意组件

* 造成经济损失 - 发送收费短信
* 造成信息丢失 - 共享位置 / 浏览历史
* 给设备开后门 - 打开一个端口

这种攻击在 ET 转发流量之前就已经进行

且攻击者不需要在网络中出现很长时间

只要设备被感染，攻击者可以立刻关闭 ET

从而使追踪该种 ET 很难

---

## Preliminaries

### Beacon Frame Components

beacon frame 是一种管理帧

AP 广播 beacon frame，显示其一些性能参数

封装了 AP 的相关信息

包含一些必须字段和可选字段（定长 / 可变长）

可变长度的字段为 Information Element (IEs)

在本文中，作者使用 beacon frame 指纹来区别合法 AP 和 ET

用到了以下字段：

* Beacon Interval - 两个 beacon frame 之间的传输间隔时间
* Capability Information - 包含 14 个子域，代表了网络的一些参数
* SSID - 字母数字的字符串，长度在 0-32 Byte之间
* Supported Rates - 是一个可变长度的 IE，显示了所有必须和支持的数据传输率
  * 一个 AP 必须有一个必须数据传输率
  * 可以有多个支持的数据传输率
* Traffic Indication Map (TIM) - 可变长度的 IE
* Country - 不同国家规定了不同的允许信道和允许功耗
* Robust Security Network (WSN) - 可变长度的 IE，声明了用于认证和加密的套件
* Extended Support Rates - Supported Rates 只能保存 8 个数据传输率，如果要支持更多，则使用 Extended Support Rates
* Vendor-Specific - 可变长度的 IE，总是 beacon 的最后一个元素，由厂商填写信息
* MAC Header - 管理帧的 MAC Header 中的所有字段都是必须的，且是定长的
  * BSSID - MAC Address of AP

### Definition of D2D Attack

设备 A 通过无线信道向设备 B 发动恶意活动

* A 可以是 Android 设备 / 笔记本 / 无线路由器
* B 是 Android 设备

### Types of ETs

* Hardware - 路由器，昂贵，攻击者很少使用
* Software - 最普遍的方法
* Mobile Devices - 允许攻击者修改的字段有限

### Launching ET Attacks

* Substitution - 攻击者在同一地点将合法 AP 替换为 ET
* Colocation - 攻击者在合法 AP 附近假设 ET，并传输强度更高的信号
* Remote Location
  * 合法 AP 曾经出现过
  * 设备中缓存了合法 AP 的参数
  * 攻击者在别处重新假设相似的 AP，用户自动重连
  * ET 不需要传输强度更高的信号
  * ET 的位置也不需要和合法 AP 相同

---

## ETGuard

### ETGuard Overview

在线、自动化、增量的实时 pre-association 分析工具

采集设备唯一的指纹，提供一个 Android APP 形式的用户接口

包含了以下七个模块：

* Request Handler
  * 并发捕获来自 APP 的 ET 检测请求
  * 被 TLS 保护
* Packet Handler
  * 采集 beacon frames
* Extractor
  * 从 beacon frame 中提取相关的数据域，构造指纹
* Processing Module
  * 指纹被传送到 Fingerprint Storage Engine
* Fingerprint Storage Engine
  * 指纹与数据库中已有的指纹进行比较
  * 如果存在完美匹配，则 AP 是合法 AP
* Fingerprint Update Engine
  * 如果新的 AP 被发现，则加入指纹数据库中
  * 因此系统是增量的
  * 即使 AP 的 beacon frame 个别域被修改，对应的指纹会被更新
* Deauth
  * 对检测到的 ET 广播 de-authentication frame
  * 强制断开所有与其关联的客户端设备

架构上，ETGurad 分为客户端和服务器

* 客户端作为用户接口
* 服务端处理网络中的 beacon frame 指纹

在与 AP 关联之前

客户端通过 APP 先向 ETGurad server 发送检测请求

服务器捕获 beacon frame，并与数据库中指纹进行匹配

如果 AP 不是 ET，则客户端设备发送 association request

否则，APP 将 ET 标红

ETGurad server 也将广播 de-authentication frame

### Server Analysis of ETGurad

为了嗅探 ET

server 被动扫描网络

在所有可用的信道上捕获 beacon frame

并与储存的指纹进行比较

#### Do all fields of beacon frame contribute in ET detection?

不是所有的 field 都对检测 ET 有用

有一些 field 在每个 beacon frame 中都不一样

* channel information
* timestamp
* sequence number
* frame check sequence

只有那些在一个 AP 所有的 beacon frame 中都相同的 field 可以被用于构造指纹

#### Why beacon frame fingerprinting is efficient in differentiating legitimate AP from the fake ones?

beacon frame 是 AP 广播，用于告知自身参数的管理帧

参数因制造商而异

也因硬件 AP、软件 AP、移动热点而异

* TIM
  * 用于为低功耗设备缓存数据
  * 不可配置
  * 对于移动热点，默认值为 9；对于软件或硬件 AP，默认值是 4
* Capability Information
  * 某些 field 与硬件和驱动有关，不能被修改
* Supported Rates
  * 硬件 AP 支持更多的数据传输率
* Vendor-Specific
  * 因制造商而异
  * 不可被修改

两个具有相同 BSSID 和安全设置的 AP 会约束 Android 设备自动连接

因此攻击者通常不能使用完全相同的 BSSID

#### Do the fields that change dynamically in beacon frames in a real environment can contribute in ET detection?

对于 TIM 域，DTIM count 是固定的

并且因制造厂商、软件或硬件 AP 而异

* 对于硬件 AP，值是 1
* 对于软件 AP 或移动热点，值是 2

硬件 AP 和移动热点无法修改，软件 AP 可以配置该域

#### If AP and ET belong to the same OEM, can ETGuard detect the fake one?

对于同一厂商生产的 AP

* Country
* Supported Rates
* Extended Support Rates
* Vendor-Specific

都是相似的

因此只通过 beacon frame 是不足够分辨出合法 AP 和 ET 的

beacon frame 也会携带一个头部 - Radiotap Header

* 其中包含 signal strenth

两个 AP 的信号强度不可能完全相同

信号强度一定是会波动的，但总是在一个固定的范围内

为了吸引用户，ET 通常会使用较高的信号强度

因此，ETGurad 保存了 AP 的最高信号强度

用于分辨属于相同生产厂商的 AP

### Methodology

将上述提到的所有字段，以及最高信号强度用于检测

如果 ET 使用的是软件 AP 或移动热点，则会被 ETGurad 检测到

如果攻击者使用与合法 AP 完全相同的厂商和型号的硬件 AP

则通过 SSI 的最高强度进行检测

### Detection Algorithm

ETGurad 将 AP 分为三类：

* Legitimate AP
* ET
* Unregistered on the network

如果提取出的指纹与任意一条记录都不匹配

ETGurad 将会比较 SSID

* 如果 SSID 与某一合法 AP 的 SSID 相同，则该 AP 为 ET
* 否则是未注册的 AP

---

## Results and Discussion

### Implementation

Packet Handler 使用 _tshark_ 捕获 beacon frame

### Case Study

#### Dataset

在 _mysql_ 中创建了指纹数据集

#### Experimental Setup

APP 被安装在 5 款不同 Android 版本的设备上

* 正常场景 - 只有合法 AP
* 攻击场景 - 合法 AP、ET、ETGurad

#### Detection

分别使用硬件 AP、软件 AP 和移动热点制造 ET

* 对于硬件 AP
  * 配置 SSID、BSSID、channel、frequency band 与合法 AP 类似
  * signal strength 设定比合法 AP 更高
  * ETGurad 成功检测出了所有的硬件 ET
* 对于软件 AP
  * 有些 field 不可被配置，导致与硬件 AP 的不同
  * 所有的软件 AP 的软件都不支持很好的用户控制
  * ETGurad 成功检测出了所有的软件 ET
* 对于移动热点
  * 移动热点为用户提供了最少的自定义配置
  * ETGurad 能够成功检测

评估了三种攻击场景：

* Colocation
  * ETGurad 有效检测
* Substitution
  * 除非 ET 使用了相同厂商、相同型号的设备，且信号强度恰好相同
* Remote Location
  * 除非 ET 使用了相同厂商、相同型号的设备，且信号强度恰好相同

然而完全相同的场景出现的概率极低

#### Accuracy

Less FPs, no FNs

### Client APP Interface for ETGuard

发送检测请求到服务器

收到服务器回复后

刷新 Wi-Fi 列表，并将 ET 标红

辅助用户连接到合法 AP

### Discussion

ETGurad 会在收到客户端请求和响应客户端之间经历延时

延时发生在 Packet Handler 模块

* 该模块在 13 个信道上捕获 beacon frame

如果 AP 在 ETGurad 的正常工作范围内

* 采集的时间可被缩短
* 否则需要更长的采集时间

由于 AP 每 `102.4ms` 传输一次 beacon frame，采集时间至少大于这个时间

Extractor Module 会产生 `650ms` 的延时

* 从 beacon frame 中逐个提取 field
* 提取的 field 在 beacon frame 中不按顺序
* field 的长度不定

---

## Related Work

### ET Detection

* pre-association - 在连接和流量产生之前检测 ET
* post-association - 在连接和流量产生之后检测 ET

#### Pre-association

* 协议修改 - 不实用
* Inter-packet arrival time - 在实用中准确率差
* Clock skew phenomenon - 特征不够唯一
* Radiometric signal properties - 信号波动
* TSF

#### Post-association

* Inter-packet arrival time - 一跳或多跳
* 发送水印包，检测是否在其它信道上出现
* 检测 ET 的转发行为
* ET 的 ISP、RTT、RTT 的标准差

Post-association 无法防御之前提到的恶意组件攻击

---

## Summary

这篇论文的点其实我以前有想到过

因为发现一些软件 AP 的报文格式上与硬件 AP 有一些差异

这篇论文抓住了这一个点

提取了所有可能不同的 field 进行检测

没有用机器学习算法

但是覆盖了所有可能的攻击场景和 ET 类型

所以感觉对我的毕设还是有一定帮助的

---

