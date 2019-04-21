# Outline

## Learning-based Rogue Edge Detection in VANETs with Ambient Radio Signals - IEEE ICC 2018

Created by : Mr Dk.

2019 / 04 / 21 20:10

Nanjing, Jiangsu, China

---

## Abstract

在 Vehicular Ad-hoc Networks (VANETs) 中

移动设备必须解决流氓的边缘结点攻击

即流氓的边缘结点伪装成合法的服务结点

从而窃取用户隐私，或进行 MITM 攻击

VANET 中的流氓检测比室内无线网络中的检测更具挑战性

* Onboard units (OBUs) 的高移动性
* Roadside units (RSUs) 的大规模网络基础设施

在这篇论文中，作者提出了一种物理层的流氓边界检测

根据位于同一车辆内的 __移动设备__ 和 __服务结点__ 之间共享的相邻信号

待测边缘结点必须向移动设备发送物理层的测量值

* RSSI
* MAC Address
* Given time slot

移动设备将接收到的相邻信号与自己的记录进行对比

采用了强化学习技术，使移动设备能够获得最佳的检测策略

---

## Introduction

随着 _Augmented Reality_ 等计算密集型应用技术的发展

移动设备能够将计算任务转移到 VANET 中的 OBU 上

* 利用位于车辆上的边缘结点的计算、能量、存储资源
* 降低了处理延时
* 提升了电池寿命
* 减小了云端的隐私威胁

作为一种欺骗攻击

攻击者可以向移动设备发送伪造信号

假装自己是一个合法的服务结点，导致：

* 隐私泄露
* MITM 攻击
* DoS 攻击

已有的 VANET 认证包含加密、信任和证书

然而移动设备在计算、存储、电池等方面受到限制

因此更适合使用轻量级的认证协议，比如物理层认证

但由于 VANET 中的：

* 高移动性
* 资源受限

普通的物理层认证的检测效果与室内环境相比有所下降

本篇论文中，利用了移动设备和服务结点之间

相邻信号的去相关性

* 在同一车辆中的移动设备和服务结点有着同样的定位序列
* 对于路边的同一个信号源（BSs、APs、RSUs），接收到的 RSS 序列应当相同
* 对位于车辆外的服务结点来说，在足够长的时间内，无法接收到相同的相邻信号

__移动设备__ 和 __合法的边缘设备__ 都可以提取相邻信号的物理层信息

* 不同 BS、AP、RSU 的 RSSI

此种检测方式依赖更多的信号源和更多的物理层信息，用于提高检测的准确率

在检测中必须选择：

* 检测模式 - 比如相邻信号的物理层序列长度
* 检测阈值

---

## System Model

在车辆内：

* 移动设备 Bob
* 服务结点 Alice

Bob 将实时数据发送到 Alice 上进行处理

* Bob 必须确认边缘结点确实是 Alice

由于 OBU 的高移动性

* Bob、Alice、Eve 的相邻信号环境快速变化
* Bob、Alice 都可以提取周围信号的物理层信息
* Bob 在发送数据之前，命令进行检测的结点提供周围信号的物理层信息
* Alice 发送特征序列信息
* Bob 比较服务结点和自身的特征序列

不失一般性

在 time slot `k` 内，车辆带着移动设备在道路上以 `v1(k)` 的速度行驶

在 `k` 中，以 `r(k)(i)` 表示服务结点发送的第 `i` 个包的 RSSI

以 `r^(i)` 表示 Alice 发送的第 `i` 个 RSSI 信号

Eve 假设位于另一辆车上，以 `v2(k)` 的速度行使

通过发送与 Alice 的 MAC 地址相同的伪造信息

向 Bob 发送假的计算结果

设 `r(k)E(i)` 是 Eve 发送的第 `i` 个包

设 `f^(k) = [r^i(k), MAC^i(k), t^i(k)]` 为 Bob 周围的特征序列

* `r^i(k)` - RSSI
* `MAC^i(k)` - MAC Address
* `t^i(k)` - Arrival time

第 `i` 个 Bob 监测到的周围信号

类似的，用 `f(k)` 表示边缘结点监测到的周围信号

---

## PHY-Layer Rogue Edge Detection With Q-Learning

设 `R(k)` 为 time slot `k` 中的序列向量

定义 Bob 的记录序列向量为 `R^`

### Mode 1

Bob 申请发送方的历史 RSSI 数据

* `R(k) = r(k)(i)`
* `R^ = r^(i)`

### Mode 2

Bob 申请发送方的历史 RSSI 数据和共享的相邻信号特征记录

* `R(k) = [ r(k)(i), f(k) ]`
* `R^ = [ r^(i), f^(k) ]`

在检测中，Bob 让待检测的结点发送物理层信息

如果 `R(k)` 和 Bob 的 `R^` 相似

那么认证信息来自 Alice

否则是由 Eve 发出

Bob 计算 `R(k)` 和 `R^` 之间的 △

* 如果 `△ < threshold`  - 接受该测试结点
* 否则发出伪造警报

---

## Summary

算法部分我都不想看了

怎么这么抽象这么难懂啊

总体上的原理还是能明白的

---

