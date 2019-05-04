# Outline

## Rogue Access Point Detection: Taxonomy, Challenges, and Future Directions - WPC 2016

Created by : Mr Dk.

2018 / 12 / 20 14:55

Nanjing, Jiangsu, China

---

### Introduction

无线上网的两种选择：

* _Wi-Fi_ 网络 - 免费
* 蜂窝网络 - 收费

_Wi-Fi_ 网络容易出现伪造 _AP_ 等安全问题

#### Overview of the 802.11 Standard

_802.11_ 帧类型

* _Management_ - 建立和维护连接
* _Control_ - 管理无线链路，允许或拒绝设备进入传输介质
* _Data_ - 传输协议栈高层的数据

连接建立过程：

* 网络发现 - _AP_ 广播 _beacon frame_
* _Clients_ 被动监听，或主动发送 _probe request_ 认证 _AP_
* _AP_ 回应 _probe response_，包含一些重要信息如支持速率、网络信息等
* _Authentication frames_ 交换
* _Association frames_ 交换，_AP_ 为 _client_ 分配资源，同步网卡
* 建立安全连接的握手（_WPA_ 等）
* 数据帧交换

帧的安全性：

* _Management frames_ 和 _Control frames_ 不受保护
* 在 _802.11w_ 修正案中开始保护，但在实际中应用很少（需要升级设备硬件和固件）
* 解除关联/解除认证帧属于 _Management frames_，可进行 _DoS_ 攻击

#### Taxonomy of RAPs

* _Evil-twin_
  * 伪造合法 _AP_，克隆尽可能多的特征
  * 一个便携式设备，携带外接网卡，使用 _Airbase-ng_，即可完成攻击
  * 共存型、替代型
  * 拦截、操纵、重放、解密
* _Improperly Configured AP_
  * 缺乏背景知识的网络管理员
  * 设备瘫痪
  * 软件更新
  * 需要插入交换机和路由器，因此它的存在本没有恶意
* _Unauthorized AP_
  * 由员工或新手私自安装（没有网络管理员的允许），本意是为了方便
  * 连接到了网络的有线侧，所以可被认为是网络的一部分
* _Compromised AP_
  * 破解 _WEP_ 或 _WPA_
* _RAP-Based Deauthentication/Deassociation_
  * 冒充 _client_ 向 _AP_ 发送帧，_AP_ 收到后与 _client_ 解除连接
  * 冒充 _AP_ 向 _client_ 发送帧，_client_ 收到后与 _AP_ 解除连接
  * 冒充 _AP_，通过广播地址向所有 _clients_ 发送帧，所有关联的 _clients_ 都与 _AP_ 解除连接
* _Forged First Message in a Four-Way Handshake_
  * 由于四次握手的 _Msg1_ 未被加密

---

### Classification of Existing Solutions

* 检测位置分配
  * _IDS_ 实现于 _AP_ 或 _Router_ 上
  * 客户端的检测 - 受到一些限制
* 被动 _or_ 主动
  * 被动方法 - 通过监控无线流量检测 _RAP_
  * 主动方法 - 发送测试包，检测 _AP_ 如何回应
    * 问题：_RAP_ 不回应主动探测包
* 需要特定硬件
* 需要改动协议
* 有线 _or_ 无线

---

### Available Security Countermeasures

### Classification of Existing RAP Detection Approaches

### Road Map and Future Directions

理想解决方案：

* 能够检测出所有类型的 _RAP_
* 被动检测更好，因为不会产生额外流量
* 避免使用除了 _sensor_ 以外的额外硬件
* 避免改动协议
* 不应该依赖高层协议
  * 延迟了检测时间
* 不应该依赖易于伪造的特征，如 _MAC_，_IP_ 等

---

### Summary

好水的文章

涉及的面倒是很广 算是一个综述吧

以后写论文的时候说不定会用到

---

