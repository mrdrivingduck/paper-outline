# Outline

## METDS - A Self-contained, Context-Based Detection System for Evil Twin Access Points - Financial Cryptography 2015

Created by : Mr Dk.

2019 / 04 / 30 18:45

Ningbo, Zhejiang, China

---

## Abstract

本文提出了一种基于情境的识别算法

* 自动部署
* 不需要任何改变任何基础设施

---

## Introduction

Mobile Evil Twin Detection System (METDS)

独立的，基于情境的检测系统

只检查 AP 的 SSID 是不够的

需要使用更多的环境数据配合检测

* 目前为止
* 附近的无线网络
* ...

---

## Related Work

与以前的类似

---

## Types of Mobile Evil Twin Attacks

### Faking an Access Point's SSID (Type A)

### Faking an Access Point's BSSID (Type B)

### Faking an Access Point's Environment (Type C)

### Faking the Entire Environment (Type D)

---

## Field Study

一堆废话

---

## METDS: Mobile Evil Twin Detection System

使用周围环境中尽可能多的数据进行检测

两种情况：

1. 用户想要连接到一个从未连接过的网络
   * METDS 没有网络的预先知识
   * 无法协助用户进行检测
   * _Trust On First Use (TOFU)_
2. 用户连接到一个已经连接过的网络
   * 由于已经有了数据集
   * 系统根据环境状态执行检测

__SSID__ 被用于识别已知网络

__BSSID__ 也被考虑在其中

由于 SSID 和 BSSID 都可能被伪造

计算会记录和比较无线网络环境

不可信的指标：

* 信号强度 - 强烈依赖设备的位置和方向
* 工作频率 - 大部分现代 AP 具有自动切换信道的功能

由于大部分的智能手机或平板都能连接到蜂窝网络

* 基站信息也被视为一个环境参数

设备的定位信息

* Google Play services API
* Android location API

### Unknown SSID

数据库中没有的 SSID 将会被算法接受

但该 AP 的信息和周围环境的信息将被记录到数据库

### Unknown BSSID

给用户两个选项：

* 如果用户知道该 AP 合法，则信任该 AP，创建新条目并储存
* 如果用户不愿意连接，则连接过程被中断

### Unknown Environment

在后台进行 Wi-Fi 扫描

扫描到的环境信息将与数据库中的信息进行比较

如果 AP 的网络环境未知，会比较基站信息

* Location Area Code (LAC)
* Cell ID

如果没有适合的基站信息 tuple 出现

则连接过程被中断

### Unknown Location

调用 Google Play services API 可以在一秒内获得定位

如果不可用，则调用 Android 的 Location API

检查定位是否在数据库中的定位的 100m 半径内

> 在上述算法中，用到了尽可能多的环境数据
>
> 所有参数都是通过传感器或 Android API 获取
>
> 数值和条目都存储在设备的数据库中
>
> 因此 METDS 是独立的，不需要任何后端服务器

---

## Summary

原理也太简单了叭

就是引入尽可能多的环境数据进行判断

感觉论文的工作量主要体现在做调查上

SSID 和 MAC 地址容易被软件伪造

但是环境数据不那么容易伪造

不过对于它的检测率存疑

---

