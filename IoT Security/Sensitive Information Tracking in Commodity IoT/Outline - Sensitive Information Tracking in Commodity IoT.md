# Outline

## Sensitive Information Tracking in Commodity IoT - 27th USENIX

Created by : Mr Dk.

2018 / 12 / 01 22:32

Ningbo, Zhejiang, China

---

### Introduction

物联网领域中存在很多隐私信息的安全问题

联网设备可以访问到非常隐私的信息，比如：

* 何时睡觉
* 门锁的解锁密码
* 收看的电视频道

设备本身的状态信息也是敏感信息

现在商用的物联网系统缺少信息使用的分析工具

需求：__一套工具和技术，识别出物联网应用中的隐私泄露隐患__

本文贡献：

* _SAINT_ - 通过从敏感数据源头 _source_ 到接收端 _sink_ 跟踪信息流，从而找出物联网应用中的敏感数据流
* 使用 _SAINT_ 评估了 _230_ 款 _SmartThings_ 应用
* 提出一个包含了 _19_ 款手工恶意应用的测试套件 _IOTBENCH_，用于测试 _SAINT_ 的能力

### Problem Scope & Attacker Model

#### Problem Scope

_SAINT_ 分析物联网应用的源代码

* 从 __泄露源__ 识别出敏感数据
* 为敏感数据关联标签，描述敏感数据的 __源头__ 和 __类型__
* 启动静态的分析，追踪被标记的数据如何在应用中传递
* 报告敏感数据如何从 _sink_ 发出
  * 通过 _Internet_
  * 通过 _SMS_
* 在警告中，_SAINT_ 报告标签中的来源，和 _sink_ 的细节
  * _Internet_ - _URL_
  * _SMS_ - _Phone number_
* _SAINT_ 不负责决定信息泄露是否恶意或危险

#### Attacker Model

攻击者向用户提供一个恶意的应用

该应用在用户 __授权或未授权__ 的条件下泄露数据

* 偏离应用阐明的功能，授予的权限可能会侵犯用户隐私
* 物联网平台授予的权限也可能造成数据泄露

论文假设：

* 攻击者无法绕开物联网平台的安全机制
* 攻击者无法通过侧信道攻击

### Background of IoT Platforms

#### Overview of IoT Platforms

##### SmartThings

由 _Samsung_ 开发，包含三个组件：

* _hub_ - 负责所有连接设备、_cloud backend_、_apps_ 之间的通信
* _apps_ - 使用 _Kohsuke sandboxed environment_ 限制下的 _Groovy_ 语言开发
* _cloud backend_ - 创建物理设备的软件封装，并运行 _apps_

权限系统允许开发者在应用安装时指定 __设备__ 和 __用户输入__

* 用户输入用于实现 _app_ 的逻辑
* 设备拥有 _capabilities_，分为 _events_ 和 _actions_
  * _actions_ 代表如何控制或驱动设备
  * _events_ 代表设备的状态信息

_apps_ 是 __事件驱动__ 的 - 

* _apps_ 订阅设备的 _events_ 或其它预先定义的 _events_
* 当 _event_ 发生时，对应的 _event handler_ 会被调用，采取某个 _action_

安装 _app_ 的方式：

* 从官方的应用市场下载
  * 在应用市场上发布需要提交源代码
  * 需要两个月左右的审核时间
* 通过 _Web IDE_ 安装第三方应用

特性：

* 支持更多的设备
* 官方和第三方的应用数量都在上升

##### OpenHAB

提供灵活、自定义的 __设备集成__ 和 __应用 (rules)__

_rules_ 由三种触发器实现：

* _Event-based triggers_ - 监听来自设备的命令
* _Timing-based triggers_ - 响应特定时间
* _System-based triggers_ - 响应特定的系统事件（开机 / 关机）

安装方式：

* 将 _apps_ 放入 _rules folder_ 中
* _Eclipse IoT marketplace_

##### Apple's HomeKit

每一个设备拥有：

* _Capabilities_ - 一个设备能做什么
* _Actions_ - 向特定设备发送命令
* _Triggers_ - 基于地点、设备、时间执行响应动作

开发者编写脚本，描述特定的 _actions_、_triggers_ 以控制设备

* _Swift_
* _Objective C_

#### Information Tracking in IoT Apps

物联网平台与其它平台相比，在信息流追踪方面有一些独特的特征和挑战：

* _Android_ 等移动操作系统有定义明确的中间语言，可供直接分析；物联网平台较为多样化，每个平台使用它们自己的编程语言——论文中提出一套新式中间语言，使用了物联网应用事件驱动的特性，并可适用于多个物联网平台
* 在物联网应用中追踪 _source_ 和 _sink_ 较为困难，论文描述了常见的 _source_ 和 _sink_，用于说明为何它们会造成隐私泄露
* 每个物联网平台的特性为泄露追踪带来挑战

#### IoT Application Structure

描述了常见的泄露 _source_ 和 _sink_

##### Taint Source

* _Device States_ - 比如门锁的开闭
* _Device Information_
* _Location_ - 可获得日出 / 日落时间，用于控制窗帘
* _User Inputs_ - 包含可辨识的用户数据，可用于描绘用户行为
* _State Variables_ - 程序执行状态，包含一些敏感数据

##### Taint Propagation

一个特定的 _event_ 发生

物联网应用调用对应的 _event handler_ 中的 _action_ 用于控制设备，改变设备状态

_Event handler_ 不限于只实现 _action_，还可以实现应用逻辑、发送消息、记录到外部数据库等

* 记录疑似泄露数据 __被拷贝__ 或 __被用于计算__
* 对泄露数据的所有追踪都被移除时，取消泄露怀疑

##### Taint Sinks

* _Internet_
  * 发送敏感数据到外部设备 - 使用 _HTTP_ 接口
  * 提供 _Web_ 服务，通过外部请求获取敏感信息 - 暴露一个 _URL_ 使外部能够发送请求
* _Messaging Services_
  * 推送通知、发送短信给指定接收人 - 使用 _Messaging API_

### SAINT

#### From Source Code to IR

