# Outline

## A Novel Traceroute-based Detection Scheme for Wi-Fi Evil Twin Attacks - GLOBECOM 2017

Created by : Mr Dk.

2018 / 12 / 23 17:04

Nanjing, Jiangsu, China

---

### Introduction

无线传感器网络 _WSN_ 在物联网中的应用越来越多

主要应用于从物理世界中采集数据

通常来说，采集到的数据会被发送到比较强大的 _sink_ 结点或 _IoT_ 终端设备

再通过它们，通过各种不同的方式接入互联网

其中，应用最普遍的方法是 _Wi-Fi_

然而 _Wi-Fi_ 容易遭受 _Evil Twin_ 攻击，流氓 _AP_ 伪造合法 _AP_

* 监听
* 中间人攻击
* _SSL Stripping_ 用于解密

本文提出了一种 __双向的、基于路由追踪__ 的检测手段，利用了一个远程检测服务器

* 特别地，比较了 _client-server_ 和 _server-client_ 路径中，在 _LAN_ 中的跳数

---

### Related Work

为了检测 _Evil Twin_ 攻击，主要存在两类解决方法：

* 基于预定义的合法 _AP_ 列表中的指纹
  * 需要网路管理员的配合
* 基于客户端的检测
  * 基于包传输延时
    * _Inter-packet Arrival Time (IAT)_
    * _Round Trip Time (RTT)_
    * 局限：导致延时的因素可能有很多 - 干扰、碰撞、网络拓扑变化等
  * 如果客户端检测到了不同的 _IP_ 和不同的网络 _ID_
    * 对同一目的地，从两个方向进行路由追踪
    * 若结果中发现多了额外一跳 - 中间人攻击
    * 如果跳数不变但路径不同 - 无法判断哪个是流氓 _AP_ - 只会警告网络不安全
    * 局限：不能保证两个路由追踪路径完全相同 - 网络拓扑可能会变化
  * 如果客户端连接到了流氓 _AP_，包在无线信道中需要经过两跳
    * 会导致比一跳更高的延迟
    * 当无线网络饱和时，检测率急剧下降
  * 在 _SSL/TCP_ 连接的中途切换 _AP_ 连接，检测公网网关是否会改变
    * 如果 _AP_ 使用的是同一个网关，那么 _TCP_ 连接将不会断开
    * 若 _TCP_ 连接断开，则暗示两个 _AP_ 使用了不同的公网网关
  * 基于包转发的行为
    * 判断一个 _AP_ 是否会向另一个 _AP_ 发包
    * 需要为每一个 _AP_ 配一个检测器

---

### Problem Statement

#### Background

_Wi-Fi AP_ 广播它们的 _SSID_

在企业中，为了增大网络覆盖范围，会部署多个 _AP_，可以使用相同的 _SSID_

客户端设备会自动连接到 _RSSI_ 最高的 _AP_ 上

#### Attack Model

基于 _Evil Twin AP_ 连接到互联网的方式，可分为：

* 基于蜂窝网络 - 使用 _2G/3G/4G_
* 基于传播 - 利用合法 _AP_ 的互联网接入
  * 对于合法 _AP_ 来说，_Evil Twin AP_ 作为一个客户端接入以获取互联网连接
  * 用另一张网卡，造出自己的 _Wi-Fi Hotspot_，为受害者提供服务

本文中主要针对后一种方式

#### Traceroute

_Traceroute_ 是一个计算机网络诊断方法

用于检测路由路径，以及数据包穿过一个 _IP_ 网络的延时

主要原理：利用 _time-to-live (TTL)_ 来测试到目的地之前的中间路由器

* 检测发起者发送 _traceroute_ 包，使用逐渐增大的 _TTL_ 值（从 `1` 开始）
* 收到该包的路由器将 _TTL_ 的值减去 `1`
* 中间路由器丢弃 _TTL_ 减至 `0` 的包，并返回 _ICMP_Time_Exceeded_ 错误
* 若目标到达，则返回 _ICMP_Echo_Reply_ 信息
* _Traceroute_ 基于收到的信息，可以获得数据包传输时经过的路由器列表

注意：

* _Traceroute_ 的过程是不对称的
* 相反方向的路径可能完全不同
  * 路由表的不对称性
  * 网络拓扑的变化

因此，简单地比较两个方向的 _traceroute_ 不足以检测 _Evil Twin AP_

本文中没有比较完整路径，而是比较了 __子网（局域网）__ 中的跳数

* 在局域网中，_NAT_ 技术用于将客户端设备与网关的 _IP_ 地址与端口号进行映射
* _client-gateway_ 和 _gateway-client_ 的 _traceroute_ 是对称的
  * 只要不违反协议
  * 只要不修改 _traceroute_ 包

---

### Our Detection Scheme

为了保护客户端不要连接到 _Evil Twin AP_

每当客户端从现在的 _Wi-Fi AP_ 上断开连接，并重连到另一个具有相同 _SSID_ 的 _AP_ 上时

检测都需要被进行

所以必须能够分辨 _Evil Twin AP_ 和另一个合法 _AP_

由于 _Evil Twin AP_ 需要额外的一跳

为了避免暴露，_Evil Twin AP_ 需要使合法 _AP_ 的那一跳消失

* 需要篡改 _traceroute_ 包
* 将 _traceroute_ 包中的 _TTL_ 增加 `1`
* 因此，光靠由客户端发出的 _traceroute_ 是无法检测出 _Evil Twin Attack_ 的

本文提出，使用一个 __专用__ 的检测服务器协助检测

* 从相反的方向进行 _traceroute_
* _Evil Twin AP_ 可以在 _client-server_ 的 _traceroute_ 结果上动手脚，掩盖合法 _AP_ 的存在
* 但不能避免 _server-client_ 的 _traceroute_ 检测到合法 _AP_
* 因为在 _server-client_ 的路径中，合法 _AP_ 位于 _Evil Twin AP_ 的前面

将 _client-gateway_ 的路径加密后传输到专用 _Server_ 进行比对

* 首先判断两次 _client-gateway_ 的网关是否是同一个（前后连接的两个 _AP_ 是否在同一子网中）
* 判断两次 _traceroute_ 的结果是否一致（尤其要注意 _traceroute_ 结果被篡改的例子）

论文假设用户设备已经订阅了 _Evil Twin_ 检测服务：

* 因此从设备到专用服务器的通信是被加密的，且不会被篡改
* 除此之外，不考虑故意丢弃 _traceroute_ 包的行为（可被视为 _DoS_ 攻击）

---

### Implementation

使用一台 _Laptop_ 作为 _Evil Twin AP_

* 内置无线网卡连接到合法 _AP_，以获得互联网连接
* 插入一个 _USB_ 无线网卡，作为热点供受害者连接

#### Traceroute from Client to Server

_Cranogenmod Busybox_ - 包含了一些轻量的 _UNIX_ 工具

执行检测功能的 _APP_ 可以通过 _RootShell_ 的 _Java Lib_ 以编程方式调用这些命令

#### Traceroute from Server to Client

由于 _client_ 位于局域网中，只有私有 _IP_

网关将会为 _client_ 分配一个特定的端口

检测服务器可以通过 `netstat` 命令，对已经建立的 _TCP_ 连接进行查找

并选择目标用户的 _IP_ 地址和进程名，并向对应端口发送信息，进行地址变换，最终转发到 _client_

由于该端口会在连接结束后被收回，可使用 _Netcat_ 解决这个问题：

* 它能在 _client_ 和 _server_ 之间维护一个 _TCP_ 连接
* 直到 _server_ 返回检测结果，_client_ 主动与 _server_ 断开连接

在 _server_ 端，使用 _InTrace_ 来进行 _traceroute_

---

### Performance Evaluation

通过一系列过程，拿到 _client-server_ 和 _server-client_ 的 _traceroute_ 结果（具体过程没懂...）

利用正则表达式对结果进行处理，得到两个 _traceroute_ 在内网中的跳数，并判断是否一致

---

### Conclusion

对于基于转发的 _Evil Twin AP_ 有效

---

### Summary

第一次看有关 _traceroute_ 方面的检测方法

大致把原理搞清楚了

但是实验我是真没有看懂。。。

这篇论文的检测方式还是有一定的局限性的

只能检测基于转发的流氓 _AP_

考虑一下能否检测具有独立互联网接入能力的流氓 _AP_ ？（蜂窝）

---

