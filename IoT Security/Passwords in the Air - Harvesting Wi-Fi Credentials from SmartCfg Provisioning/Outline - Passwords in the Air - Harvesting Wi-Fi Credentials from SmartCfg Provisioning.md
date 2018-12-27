# Outline

## Passwords in the Air- Harvesting Wi-Fi Credentials from SmartCfg Provisioning -  WISEC 2018

Created by : Mr Dk.

2018 / 12 / 27 15:44

Nanjing, Jiangsu, China

---

### Introduction

场景：智能设备认证

* 智能设备通常为 _headless device_，没有传统的输入设备，没有交互式 _UI_
* 无线芯片厂商提供的辅助认证手段缺乏保护机制

_SmartCfg_ - 一个用于向智能设备提供 _Wi-Fi_ 证书（_SSID_ 和 _PSK_）的认证技术

* 证书信息被编码后负载在特定的广播包上
* 利用辅助的 _APP_ 编码、广播证书
* 广播包被智能设备捕获，从而接入 _Wi-Fi_

本文研究了八个主流 _Wi-Fi_ 芯片厂商的 _SmartCfg_ 解决方案

* 审查了八个厂商提供的 _SmartCfg SDK_，其中六个存在 _Wi-Fi_ 证书泄露隐患
* 对现实中的智能家居设备进行了系统化的安全分析
  * 如何识别部署在设备内部的 _SmartCfg_ 解决方案
  * 如果恢复证书编码模式
* 对 _821_ 款智能家居 _APP_ 进行了分析，_64_ 款使用了 _SmartCfg_ 认证技术，其中 _42_ 款存在安全问题
  * 意味着一堆设备将受到影响

---

### Background

#### Smart Home

智能家庭，也称家庭自动化（_home automation_）

* 使主人拥有远程控制智能设备的能力
* 需要接入互联网，已实现远程的监控和管理

包含三个组成部分：

* 智能设备
  * 智能灯泡、开关等
* 智能家庭 _APP_
  * 安装在手机上，用于控制智能设备
* 智能家庭云端
  * 桥接不在同一个 _LAN_ 中的设备和 _APP_
  * 存储和管理数据

智能设备大多使用 _Wi-Fi_ 模块

* _Headless device_ - 没有输入设备和交互式 _UI_
* 在第一次部署时需要配置网络（认证）
* 设备自己无法完成认证过程，需要其它设备提供 _SSID_ 和 _PSK_，以及交互式 _UI_

#### SmartCfg Provisioning

由 _Texas Instruments (TI)_ 在 _2012_ 年提出，用于给没有交互式 _UI_ 的设备提供 _Wi-Fi_ 证书

多家无线芯片厂商基于 _SmartCfg_ 实现了各自的变种

_SmartCfg_ 的三个典型特征：

* 依赖一个移动 _APP_ 用于编码和发送广播的证书信息
* 智能设备在不知道发送者身份的条件下，被动监听编码信息
* 证书被编码为 _802.11_ 帧的元数据，而不是内容（因此无所谓加不加密）

认证过程：

* 智能设备的网卡进入混杂模式，不停捕获空间中的数据包
* 用户从手机 _APP_ 中输入证书（_SSID_ & _PSK_），编码为指定格式并发送到接入的网络中
* 智能设备的 _Wi-Fi_ 模块捕获所有的数据包，并用预置的算法试图解码并获得证书，使用证书连入网络

三种主流的数据编码方式：

* _Data in Multicast Address (DMA)_
  * 利用 _Destination Address_ 字段的最后 _23 bit_ 编码
  * 证书的每两个字节被编码在 _DA_ 的最后两个字节中
  * 倒数第三个字节作为 _index_
* _Data in Packet Length (DPL)_
  * 证书被编码在每个包的 _length_ 字段中
  * 字段内容根据 _length_ 的值随意填充
* _Hybrid_
  * 以上两个字段都被用于编码
  * _length_ 被用于存放证书，_Address_ 用于存放索引
  * 好处是一个包里存放了更多的信息，因而更有效率

数据编码的特征 - _Preamble_

* 一个相同长度和内容的包序列
* 帮助设备定位随后的证书序列

---

### Security Analysis of SmartCfg

#### Thread Models and Challenges

假设：

* 证书是攻击者的主要目标（_SSID_ & _PSK_）
* 攻击者位于 _WLAN_ 之外
* 不能 _physically_ 或 _remotely_ 连接到智能设备
* 攻击者只能从空间中的无线数据里获得证书

攻击者通过嗅探数据包，利用提前准备好的解密算法获得证书，从而接入 _WLAN_

挑战：

* 如何识别智能家居设备使用的具体 _SmartCfg_ 解决方案？（不同厂家实现不同）
  * 设备厂商不会告诉你某个设备用的是哪种解决方案
  * 需要通过一些侧信道的方式解决
* 如何恢复认证协议，尤其是证书编码模式？
  * 对于设备厂商来说，无线芯片厂商提供的 _SmartCfg_ 解决方案只是模板和参考
  * 设备厂商可能修改并实现其自己的认证协议

#### SmartCfg Solution Identification

首先从 _SmartCfg Solution_ 的 _SDK_ 信息入手，包含：

* _device SDK_
  * 用于开发设备固件
* _mobile SDK_
  * 集成在智能设备 _APP_ 中

可利用 _SDK_ 的特征来识别具体的 _SmartCfg Solution_

无线芯片厂商在官网提供 _SDK_ 的源码和文档

本文只关注 _mobile SDK_，原因：

* 厂商如果不提供固件，_flash memory_ 如果只读，那么无法获得固件，从而无法分析；大部分固件闭源，逆向工程很费时间
* 协议只能通过分析 _mobile SDK_ 得到
  * 通信协议要么从 _device SDK_ 中恢复，要么从 _mobile SDK_ 中恢复
  * 对于 _mobile SDK_ 的分析可以利用更多的逆向工程工具

从 _APP_ 商店中下载 _APP_，从网上下载 _SDK_，对它们进行相似性分析

* _SDK_ 的形式为 `.jar` 或 `.so`，暴露出函数用于调用

除去一些较短的函数名、经常使用的函数名，如果还会出现相同函数名，则解决方案就被识别出来了

#### Credential Encoding Scheme Recovering

一旦对应的解决方案被识别出来，那么就可以进一步进行代码层面的逆向工程

##### Code Analysis

* Locating Credential Input Activity
  * 在 _APP_ 中寻找输入证书的 `Activity`（_Android_ 界面）
  * 在这个过程中，_APP_ 还会调用 _Android_ 的 _Wi-Fi API_，通过这一特性可以定位该 `Activity`

* Locating Credential Encoding Functions
  * 从存储证书的变量开始寻找，直到 _APP_ 向外部环境（网络）发送数据为止
  * 任何参与数据传递的过程都被认为是编码过程
  * 在定位完成后，进行人工的逆向工程，推出编码算法

##### Network Traffic Analysis

* Sniffing
  * 捕获所有数据包，记录为三元组 `<src MAC, dst MAC, length>`
* Data Payload Locating
  * 观察数据包之间的差异，猜出编码模式
* Credential Differentiating
  * 通过对比，确定数据包中存储证书的位置
  * 使用两个相同长度的密码进行测试
    * 变化的部分不只有密码部分，还有校验和部分
    * 密码变化很小，校验和变化很大

---

### Experimental Results

#### Solution Identification

信息采集：八个 _SmartCfg Solution_ 的 _summary_

* _device SDK_
  * 预编译的二进制文件
* _mobile SDK_
  * _demo APP_、_APP_ 的 _Java_ 源代码（`.jar` 或 `.so`）
* 文档

提取每个 _solution_ 的特征，用于识别所有 _APP_

* 某些特定的库文件

通过语义分析，在应用市场中爬取所有的智能家居 _APP_ - _821_ 个，其中 _64_ 款使用了 _SmartCfg_ 解决方案

#### Encoding Scheme Analysis

首先，分析出了每种解决方案分别对应哪种编码模式

无线芯片厂商提供的安全（不安全）的编码机制可能会被设备厂商的开发人员实现为不安全（安全）的机制

八个解决方案中，六个存在不安全的编码机制

* 四个解决方案没有加密措施，直接广播编码后的数据 - 不安全
* 两个解决方案加密了编码数据，但密钥使用不当 - （密钥不变、密钥重用、密钥泄露） - 不安全

在分析所有 _APP_ 之后发现

* 使用了不安全编码机制解决方案的 _APP_ 并没有修补措施
* 使用了安全编码机制解决方案的 _APP_，在开发时由于使用不当也不安全

在 _64_ 款使用了 _SmartCfg_ 解决方案的 _APP_ 中：

* _29_ 款使用了不加密的四个解决方案
* _7_ 款使用了硬编码的 _AES_ 密钥
* _6_ 款使用了 _RC4_ 加密，但有密钥重用的问题
* _8_ 款使用了安全的解决方案，但 _AES_ 密钥依旧固定
  * 本应该使用 _PIN_ 作为密钥生成，但获取失败，所以使用了默认 _PIN_ 用于生成密钥

---

### Discussions

为了保证安全，需要设备厂商、开发者、云平台共同协作，保证一个安全的目标

本文提出的解决方案：

* 设备厂商给每个设备分配一个独立的随机 _Device ID_
  * 只能通过物理层面的方式获得，比如 _QR Code_
  * 该 _ID_ 出厂时被固化在固件中
* 认证时，对称密钥用于保护证书
* 一个云服务器被用于帮助设备认证 _APP_（攻击者无法伪造云服务器的认证 _token_）

具体细节：

1. _APP_ 通过物理接触（扫 _QR Code_）的方式获得 _product key_ （认证产品）和 _device ID_ （认证设备）
2. _APP_ 向云服务器认证自己的身份，云服务器返回给 _APP_ 一个设备的 _UUID_
3. _APP_ 使用 _product key_ 和 _device ID_ 作为材料生成密钥
4. _APP_ 使用密钥加密证书和 _UUID_，并发送信息
5. 设备使用密钥（安全存储）解密，使用证书中的信息连入指定网络
6. 设备向云服务器查询接收到的 _UUID_；如果不合法，设备拒绝被 _APP_ 远程控制

---

### Summary

这个研究还是很有意思的

可能工作量重在一开始的各种逆向工程吧

读这篇论文的目的在于

这个编码到数据包的机制可否用来进行流氓 _AP_ 检测呢

比如从客户端编码一个序列发送出去

如果有中转型流氓 _AP_ 的存在

那么这个序列一定会被 _Sniffer_ 监听到两次

---

