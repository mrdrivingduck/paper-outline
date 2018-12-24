# Outline

## Gateway independent user-side wi-fi Evil Twin Attack detection using virtual wireless clients - CS 2018

Created by : Mr Dk.

2018 / 12 / 24 17:25

Nanjing, Jiangsu, China

---

### Introduction

公共 _Wi-Fi_ 的不安全性

流氓 _AP_ 使用相同的 _SSID_ 和更好的信号强度，诱使客户端连接

* 监听流量
* 中间人攻击
* _SSL Strip_ - 迫使客户端使用 _HTTP_ 而不是 _HTTPS_
* _DNS_ 欺骗 - 重定向到恶意站点

攻击者为了将连接上的客户端的数据送达互联网，有两种选择：

* 使用另一张网卡，连接到合法 _AP_，成为一个流氓客户端
  * _RAP_ 和 _LAP_ 使用同一个 _ISP_ 网关
* 使用自带的蜂窝通信方式（_4G LTE_）
  * _RAP_ 和 _LAP_ 使用不同的 _ISP_ 网关

论文贡献：

* 提出一种新的检测方法，可检测使用相同网关和不同网关的 _RAP_
* 随机监听多个 _Wi-Fi_ 信道，接收专用服务器发来的帧，可立即检测出使用同一网关的 _RAP_
* 检测方法部署在客户端，实时检测，不需要网络管理员的协助

---

### Related Work

_Evil Twin_ 攻击简单，使用可随意购买到的设备就能发动攻击

可随时停止攻击，使溯源变得困难

在 _AP_ 和客户端之间使用 _VPN_ 存在问题

对于 _Evil Twin Attack (ETA)_ 的检测，可分为三类：

* 修改协议
* 硬件指纹
* 无硬件认证

在本文中，_ETA_ 检测被分为两类：

* 网络管理员端的检测
  * 网络管理员拥有整个网络的信息，包括所有设备的指纹
  * 指纹是可以用来将一个或一组设备区分出来的任何信息
    * _AP_ 的位置
    * _AP_ 的时钟偏差
    * ...
  * 必须拥有设备的指纹才可以进行检测
  * 需要布设 _Sensor_，采集数据流量进行指纹认证
* 客户端的检测
  * 每个客户端只需要保证它自己的安全
  * 基于 _traceroute_ 的检测方式
    * _Traceroute_ 命令可能会被路由器屏蔽
    * _Traceroute_ 命令使用 _ICMP_，容易被重放攻击
  * 基于额外延时的检测方式
    * 判断从每个 _AP_ 上连接到相同 _DNS_ 服务器的时间
    * 由于无线链路上的额外一条，会使 _RAP_ 的延时较高而被检测出来
    * 信号波动或流量波动等问题也会导致额外延时

本文提出的检测方式：

* 使用 __虚拟__ 客户端，并行检测连接到相同网关和不同网关的两种 _RAP_
* 检测位于客户端，不依赖训练数据和预授权的指纹列表
* 网关独立

---

### Intuitive Detection Schemes and their Security Vulnerabilities

#### Adversary Model

_RAP_ 可以模仿与合法 _AP_ 几乎完全相同的特征

* _DHCP_、_DNS_ 分配的 _IP_ 和 _MAC_ 地址
* 网关

包传输时间可以被 _RAP_ 篡改，使其看起来与通过合法 _AP_ 传输类似

假设客户端设备可以同时检测到至少两个具有相同 _SSID_ 的 _AP_

* 如果只能检测到一个，本文假设没有发生 _ETA_

#### Intuitive detection schemes and their security problems

##### ETA using single ISP gateway

_2011_ 年的一种检测机制 - _Open WiFiHop_

* 基于客户端的检测
* 检测单网关 _RAP_

由以下两部分组成：

* 客户端
* 专用服务器

检测过程：

* 客户端连接到附近的一个 _AP_，并向专用服务器发送水印包
* 发送水印包后，客户端立刻监听其它 _Wi-Fi_ 信道是否出现水印包
* 专用服务器不停地回复水印包
* 如果在其它信道中捕获了水印包，则该 _AP_ 是 _RAP_

局限：

* 由于水印包是一个固定序列，很容易被重放攻击
* _RAP_ 只需要适时与 _LAP_ 断开连接或重连，即可绕开检测

_ETA_ 检测不应该基于可被攻击者收集、计算或估计的数据

##### ETA using different ISP gateways

* 基于 _IP_ 头部中的 _route option_
  * 数据包经过的每一个路由器将把它们的 _IP_ 地址放入该字段
  * 客户端接入不同 _AP_ 向同一个目的地——专用服务器发送启用 _route option_ 的数据包
  * 专用服务器可以通过分析路由路径检测 _RAP_
  * 这种包可能会被防火墙丢弃
  * 且该字段最多只能放 _9_ 个 _IP_ 地址，但互联网上的平均路由器跳数是 _19 - 21_
* 基于 _TCP_ 连接的检测
  * 客户端通过一个 _AP_ 进行与专用服务器的 _TCP_ 三次捂手
  * 客户端与专用服务器将会创建一个包含双方 _IP_ 与端口号的 _Socket_ 连接
  * 在握手成功后，客户端立即切换到另一个 _AP_，并发送 _heartbeat request_
    * 如果切换到合法 _AP_，由于 _DHCP_ 服务器是同一个，_IP_ 不变，将不会对 _Socket_ 连接产生影响
    * 客户端应当成功接收到 _heartbeat response_
    * 如果 _TCP_ 连接中断，那么说明两个 _AP_ 使用了不同的网关
    * 攻击者可以用 _RAP_ 单独与服务器建立一个 _TCP_ 连接，并截获 _heartbeat request_，伪造成专用服务器

---

### Proposed ETA Detection

#### Assumptions

使用同一网关的检测：

* 当客户端通过 _RAP_ 发送数据时，相同的数据将会出现两次
* 其它合法 _AP_ 通过网线接入互联网，不可能重发客户端数据

使用不同网关的检测：

* 为了更好的信号覆盖，同一个热点会部署多个 _AP_
* 多个 _AP_ 使用的网关是同一个
* 接入无线网络的客户端被分配私有 _IP_ 地址
* 这些私有地址在经过网关时会被 _NAT_ 转换为公有 _IP_ 地址

#### Detection Mechanism

##### Detection of ETA using single ISP gateway

由两部分组成：

* 客户端
* 专用服务器

检测过程：

* 监听 _beacon_，客户端记录附近所有待检测网络（相同 _SSID_） _AP_ 的 _MAC_ 地址和 _Wi-Fi_ 工作信道
* 客户端随机连接到其中的一个 _AP_ 上
  * 网络中的 _DHCP_ 服务会为其分配 _IP_ 地址
  * 与专用服务器建立连接，发送一个 _hello_ 包，服务器为客户端分配一个用于区分其它客户端的独立 _ID_
  * 客户端接收到自己的 _ID_ 后，将 _AP_ 的 _MAC_ 地址与自身 _ID_ 一起发送到专用服务器
  * 服务器与客户端分别保存连接信息
* 客户端随机连接到另一个 _AP_ 上
  * 同时，客户端切换自身的 _MAC_ 地址
  * 与专用服务器建立新的连接，将 _AP_ 的 _MAC_ 地址和自身 _ID_ 一起发送到专用服务器
* 迭代直到同一网络中的所有 _AP_ 都被连接过
* 通过连接的最后一个 _AP_，客户端向服务器发送 _Info Start_ 包
* 服务器该包后，向每个连接分别发送 _Info Start_
  * 包中带有 _AP_ 的 _MAC_ 地址
  * 包中带有自增序列号，以防止重放攻击
* 客户端发送 _Info Start_ 包后，立刻随机切换到某个 _AP_ 的工作信道上
  * 根据客户端 _ID_ 过滤 _Info Start_
  * 所有经过过滤的帧的目的 _MAC_ 地址都应该是客户端使用过的 _MAC_ 地址之一
  * 如果出现了没有被客户端使用过的 _MAC_ 地址，则说明是发送给 _RWC_ 的帧，说明存在 _RAP_
  * 如果没有从属于当前信道的 _AP_ 上收到 _Info Start_，那么该 _AP_ 也是 _RAP_
  * 客户端检查每一个收到的包的序列号，丢弃序列号小于等于前一个包的包

##### Detection of ETA using different ISP gateways

* 客户端连接任意一个 _AP_，初始化一个安全的 _TCP_ 三次握手
  * 切换 _AP_ 不会影响现有会话
  * 客户端保持现有的 _MAC_ 地址，重用之前的 _IP_ 地址
* 如果网关不同，专用服务器将无法回应客户端的 _heartbeat response_

##### Comprehensive ETA detection

基于同一网关的检测和基于不同网关的检测使用同一个物理网卡并行

* 使用虚拟客户端 _VMCs_ 实现
* 两个 _VMC_ 使用不同的 _MAC_ 地址，因此 _DHCP_ 分配不同的 _IP_ 地址
* _VMC1_ 与专用服务器连接并获取 _ID_
* _VMC2_ 与专用服务器建立 _TCP_ 连接

切换到下一个 _AP_ 进行检测：

* _VMC1_ 更换 _MAC_ 地址，从 _DHCP_ 服务器分配新的 _IP_ 地址
* _VMC2_ 维持原有的 _MAC_ 地址，维持原有的 _IP_ 地址

如果 _VMC2_ 没有收到 _heartbeat_ 回应，则检测停止

如果 _VMC2_ 收到回应，则 _VMC1_ 继续检测

##### Proposed detection efficiency

检测所有记录的 _AP_ 信道一次，_ETA_ 检测率为 _0.75_

检测四次将使 _ETA_ 检测率接近 _100%_

#### Inplementation

_LORCON2_ - 开源库，制造 _802.11_ 数据帧

---

### Summary

_VWC1_ 的检测部分没有太看懂

感觉论文里的很多描述都不严谨 累了

---

