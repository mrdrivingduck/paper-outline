# Outline

## Method and system for detecting wireless access devices operably coupled to computer local area networks and related methods

Created by : Mr Dk.

2018 / 11 / 20 09:16

Nanjing, Jiangsu, China

---

### Abstract

一套系统及相应的方法 - 探测连接到局域网的无线接入设备

方法主要包含：

* 一个连接到局域网中的 _Sniffer device（嗅探设备）_
* 将一个或多个数据包通过局域网 __传输__ 到接入局域网的选定设备
* 使用嗅探设备 __拦截__ 一个或多个传送到选定设备的数据包
* 从拦截到的数据包中 __获取__ 信息
* 使用嗅探设备基于部分拦截到的数据 __产生__ 一个或多个 _marker packet_
* 嗅探设备能够通过局域网传输 _marker packet_ 到选定设备
* 使用一个或多个嗅探设备 __监控__ 选定设备的邻近空间的无线通信状态

---

### Background

传统的局域网安全 - 由于接入端口位置已知，主要通过 __物理空间的接入控制__ 来保证安全

无线局域网：

* 无线电波在 __物理空间__ 内无法被 __物理结构__ 阻拦
* 无线信号 __溢出__ 可控物理空间

为防止未授权的无线接入，_AP_ 引入了 _WEP_ 或加密、防火墙机制

但危险依旧存在 - 一个未授权 _AP_ 接入局域网，使未授权用户连接即可访问局域网

---

### Brief summary of the invention

提供带有 __无线扩展__ 的局域网入侵检测方法和系统

特别地 - 提供测试某个接入局域网的设备的 __无线传输连通性__ 的方法和系统

更特别地 - 提供检测接入局域网的 __未授权__ 无线 _AP_ 的方法和系统

---

### Brief description of the drawings

* _Fig 1_ - 一个简化的 _LAN_ 架构
* _Fig 2_ - 一个示例的 _Sniffer device_ 硬件结构
* _Fig 3_ - 安全策略
* _Fig 4A_ - 检测连接到 _LAN_ 的无线接入设备的方法
* _Fig 4B_ - 本发明中网络组件的互联方式
* _Fig 5_ - 检测无线 _AP_ 与 _LAN_ 连通性的方法
* _Fig 6_ - 列出疑似 _NAT AP_ 的方法
* _Fig 7_ - 通过无线链路数据包中的 _MAC_ 地址识别一个 _NAT AP_ 的方法
* _Fig 8_ - 一个示例的 _Sniffer device_ 功能结构

---

### Detailed description of the invention

#### Fig 1 - LAN 架构

* 一个或多个用户设备可通过网线或电缆连接端口，接入 _LAN_
* 数据库计算机和服务器计算机也可以通过端口接入 _LAN_
* 作为网关的路由器也可以接入 _LAN_
* 已授权 _AP_ 可通过连接到一台交换机，交换机通过端口接入 _LAN_
  * 交换机可负责一些复杂过程
    * 认证
    * 加密
    * 防火墙
    * ...
* 已授权 _AP_ 也可以直接通过端口接入 _LAN_
  * 上述复杂过程由 _AP_ 自身完成
* 一个或多个具有无线通信能力的用户设备可以通过已授权 _AP_ 接入 _LAN_
* 未授权 _AP_ 也可以通过端口接入 _LAN_
  * 恶意 _AP_ - 由具有在物理空间上访问 _LAN_ 的人员布设（比如绕开网管的企业内部人员）
  * 配置错误的 _AP_ - 网管允许，但 _AP_ 参数被无意地错误配置了，容易被入侵
  * 软 _AP_ - 一台允许使用 _Wi-Fi_ 的电脑通过端口接入 _LAN_，同时在软件的控制下扮演 _AP_ 的角色
* 未授权设备（入侵者）通过接入未授权 _AP_ 的方式接入 _LAN_
* __NAT AP__ 扮演 __第三层路由器__ 的角色，将数据包从有线网卡传输到无线网卡，反之亦然
  * 有线网卡和无线网卡通常 __处于不同的子网中__
  * _NAT_ 实现 _IP_ 地址和端口号的转换功能
* 有一类未授权 _AP_ 不会对 _LAN_ 有显著的危害
  * 信号存在于 _LAN_ 的物理区域中
  * 接入或未接入一个邻居 _LAN_
  * 未接入本 _LAN_
  * 也有可能是一台恶意 _AP_
    * 引诱 _LAN_ 物理区域内的设备连接它
    * 发起中间人攻击、_DoS_ 攻击、_MAC_ 欺骗攻击等
* 一个无线入侵检测系统可使用一个或多个 _Sniffer_ 以有线或无线的方式接入 _LAN_
  * _Sniffer_ 能够监控某一区域内无线活动
    * _AP_ 与无线设备之间的控制包、管理包、数据包
    * _AP_ 与无线设备建立连接时的 _Association_ 过程
  * _Sniffer_ 能够监听多个无线电信道
    * 循环监听各个信道
    * 同时监听各个信道
  * _Sniffer_ 能够获取并记录抓到的包中的信息
* 未授权 _AP_ 对无线链路进行加密，勾结入侵者设备
  * _Sniffer_ 通过抓包很难解密包中的信息

#### Fig 2 - Sniffer 硬件结构

* _CPU_ - 监测和记录功能
* _Flash Memory_ - 存放 _Sniffer_ 功能代码
* _RAM_ - 程序运行时的易失性存储器
* _802.11 Wireless Network Interface cards（NICs）_ - _MAC_ 层功能
* _Ethernet NIC_ - 接入物理 _LAN_
* _Serial port_ - 用于排解故障
* _LED_ - 用于显示工作状态

#### Fig 3 - 安全策略

将所有的 _AP_ 分为三类：

* 授权 _AP_ - 网络管理员允许的 _AP_
* 流氓 _AP_ - 网络管理员不允许，但接入 _LAN_ 的 _AP_
* 外部 _AP_ - 网络管理员不允许，但没有接入 _LAN_ 的 _AP_

安全策略：

* 允许 - 授权的无线设备连接到授权 _AP_
* 忽略 - 未授权的无线设备连接到外部 _AP_
* 禁止 - 其余情况

#### Fig 4 - 检测无线接入设备是否连接到 LAN 的方法

__对于 LAN 中的某个已知设备，但无法确定它是否具有无线接入功能（是否是 AP）__

_Marker Packets_

* 形式可以是：
  * _MAC_ 层数据包
  * _TCP_ / _UDP_ 数据包
  * _ICMP_ 数据包
  * _ARP_ 数据包
  * ...
* 格式可以是：
  * 选定的一个或多个 _size_
  * 选定的 _bit pattern_
  * 选定的 _time instants_

初级方法：

* 通过局域网，_Sniffer_ 向疑似 _AP_ 的选定设备发送一个或多个 _marker packets_
* 如果选定设备已接入 _LAN_，它一定能接收到一个或多个 _marker packets_
* 如果选定设备是 _AP_，它一定会在无线链路中发射一部分收到的包
  * 可能对包做过处理及修改
* 如果 _Sniffer_ 在无线链路中探测到了相关 _marker packets_，则该设备是 _AP_，且接入了 _LAN_

采用初级方法的局限性：

* 绝大部分 _AP_ 提供 _NAT_ 功能
  * 只有在 _NAT AP_ 收到存在对应端口映射的数据包时，才会向无线链路发送
  * 否则 _AP_ 将会把包丢弃
  * 端口映射表在无线设备与 _AP_ 进行会话初始化时建立
  * 在不知道端口映射表的条件下，_market packets_ 无法被传送到 _NAT AP_
* _AP_ 与无线设备的通信链路被加密
  * _Sniffer_ 无法解密，从而无法获取端口映射信息
  * _Sniffer_ 无法获取带有特征的相关 _market packets_

__Fig 4A 中的方法能够有效检测接入内网的 NAT AP，即使通信链路被加密__

* 前提：_Sniffer_ 连接到 _LAN_
* 前提：能够通过 _LAN_ 向选定的设备发送一个或多个包
* _Sniffer_ 拦截到在 _LAN_ 上传输的包
* _Sniffer_ 记录拦截到的包中的信息
  * 目标端口类型
  * 目标端口号
  * ...
* _Sniffer_ 将拦截到的包重新发出 - 可能经过了处理和修改
* _Sniffer_ 产生 _marker packets_
  * 选定格式 - _size_ / _bit pattern_ / _time instants_
  * 使用部分拦截到的包中的信息（相同的端口类型及端口号）
* _Sniffer_ 通过 _LAN_ 发送 _marker packets_
* _Sniffer_ 监控附近物理空间内的无线链路
  * 若选定设备是 _NAT AP_
    * _marker packets_ 的端口号有效
    * 采用了特定的格式
    * 它一定会被发送到无线链路中并被嗅探到
  * 若选定设备不是一个无线接入设备
    * _marker packets_ 不会被路由至无线链路中

方法也可以稍作变化：

* _Sniffer_ 不产生 _marker packets_，直接使用拦截到的包作为 _maker packets_
* _Sniffer_ 需要记录拦截包的格式信息，用于在无线链路中匹配
* 同样 - 如果选定设备是 _NAT AP_，该包一定会在无线链路中被 _Sniffer_ 嗅探到

#### Fig 5 - 检测无线 AP 与 LAN 的连通性



