# Outline

## Method and system for detecting wireless access devices operably coupled to computer local area networks and related methods

Created by : Mr Dk.

2018 / 11 / 19 23:35

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

