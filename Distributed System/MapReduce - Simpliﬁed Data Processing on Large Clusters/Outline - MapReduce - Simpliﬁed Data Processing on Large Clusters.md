# Outline

## MapReduce: Simplified Data Processing on Large Clusters

Created by : Mr Dk.

2019 / 11 / 02 17:29

Nanjing, Jiangsu, China

---

## Abstract

MapReduce 是一个计算模型及其相关的实现

用于处理和生成大数据集合

* 使用一个特定的 `map` 函数处理 key/value 对，生成 key/value 形式的中间值
* 使用一个特定的 `reduce` 函数归并所有 key 相同的中间值

写成这种风格的函数可以自动地在大型集群上并行执行

运行时系统负责：

* 切分输入数据
* 在一大堆机器上调度程序执行
* 处理机器故障的问题
* 管理机器之间的通信

使没有任何并行系统或分布式系统的经验的程序员

可以轻松利用大型分布式系统中的资源

---

## 1. Introduction

为了处理大量的原始数据

Google 实现了很多专用目的的计算程序

大部分计算在概念上是很直接的

然而，输入数据过大，为了在合理的时间内完成

因此这些计算通常会被分布在成百上千台机器上

问题在于：

* 如何将计算并行化
* 如何分布数据
* 如何处理阻碍计算进行的错误

本文提出了一种抽象：

* 能够表达简单的计算
* 隐藏了复杂的并行化、容错、数据分布、负载均衡等细节
* 由 `map` 和 `reduce` 操作组成
    * 对输入中的每个记录应用 `map` 操作，计算一个 key/value 形式的中间值集合
    * 对所有 key 相同的中间值应用 `reduce` 操作，合并对应的数据

---

## 2. Programming Model

用两个函数来表示计算：`map` 和 `reduce`

* `map` 由用户编写
    * 接收输入，产生一系列中间结果的 key/value pair
    * 将所有中间结果中 key 相同的 pair 传递给 `reduce`
* `reduce` 也由用户编写
    * 接收一个中间值的 key 以及对应的 value 集合
    * 将结果归并为一个可能更小的集合并输出

### 2.1 Example

word-count

检索一个超大文件中每个词出现的次数

* `map` 函数对于其处理的数据块中的每个词，发送 `<word, 1>`
* `reduce` 函数将同一个词的出现次数累加

用户编写代码填充 _mapreduce specification_ 对象

* 输入文件
* 输出文件
* 可选参数

用户代码和 C++ 实现的 MapReduce 库链接在一起

### 2.2 Types

```
map    (k1, v1)        ->  list(k2, v2)
reduce (k2, list(v2))  ->  list(v2)
```

从用户输入中读取或向用户输出 __字符串__

* 由用户代码实现字符串和特定类型之间的转换

### 2.3 More Examples

* Distributed Grep
    * `map` 函数在匹配上特定模式时会发送一行
    * `reduce` 函数将对应的中间结果拷贝到输出即可
* Count of URL Access Frequency
    * `map` 函数处理网页请求的日志并输出 `<URL, 1>`
    * `reduce` 函数将所有相同 URL 的值累加 - `<URL, total count>`
* Reverse Web-Link Graph
    * `map` 函数处理所有 URL 的 `<target, source>`
    * `reduce` 函数拼接 target URL 相同的 source URL - `<target, list(source)>`
* Inverted Index
    * `map` 函数读取每个文档，并输出 `<word, doc ID>`
    * `reduce` 函数接受所有的给定词，排序，输出 `<word, list(doc ID)>`
* Distributed Sort
    * `map` 函数从每个记录中提取 key，并输出 `<key, record>`
    * `reduce` 函数直接输出中间结果
    * 直接利用 __分区__ 和 __排序__ 两个功能

---

## 3. Implementation

Mapreduce 接口可以有不同的实现

Google 的实现环境：

* 普通的 Linux 系统，双核 CPU，2-4 GB 内存
* 普通的网络硬件
* 集群中有成百上千台机器，机器故障很常见
* 分布式文件系统用于管理磁盘上的数据
* 用户向一个调度系统提交作业，每个作业包含多个任务，任务被调度器分配到可用机器上

### 3.1 Execuation Overview

集群自动将输入数据切分为 M 块

* 这些块能够被不同的机器并行处理

中间结果的搜索空间被 partition 函数分为 R 块，并交付 reduce

1. 将输入数据划分为 M 块
    * 16-64 MB 每块
    * 在集群机器上启动程序的多个副本
2. 程序的某个副本很特殊 - master；剩下的是 worker
    * master 给空闲的 worker 分配 map 或 reduce 任务
3. 被分配到 map 任务的 worker 读取对应的输入块
    * 将输入块转换为 key/value 形式，并传递给 map 函数
    * 中间结果的 key/value 对缓存在内存中
4. 周期性地把中间结果写入本地磁盘
    * 由 partition 函数划分为 R 个部分
    * 中间结果在本地磁盘上的保存位置被传递回 master
    * master 负责将中间结果的位置告诉 reduce 任务
5. 负责 reduce 的 worker 使用 RPC 将 map worker 本地磁盘上的中间结果拷贝
    * 对中间结果的 key 进行排序，key 相同的归为一组
    * 如果排序的量太大，需要用到外部排序
6. Reduce worker 将每个独立的 key 和对应的中间结果集合传递到 reduce 函数中
    * reduce 函数的输出结果被追加到该 reduce 分区的最终输出文件中
7. 当所有 map 和 reduce 任务结束后，master 唤醒用户程序
    * 返回用户代码

在执行完毕后，mapreduce 的结果位于 R 个输出文件中

用于可能不需要将 R 的结果合并到一个文件中

因为它们可能是另一个 mapreduce 任务的输入

### 3.2 Master Data Structures

Master 保存每个 map 任务和 reduce 任务的：

* 状态 - idle / in-progress / completed
* Worker machine 的身份

Master 通过中间文件的位置，联系 map 任务和 reduce 任务

* 所以 master 保存了 R 个中间文件区的位置和大小
* map 任务结束时，位置和大小会更新
* 信息被增量式地推送给 in-progress 的 reduce 任务

### 3.3 Fault Tolerance

Mapreduce 库必须优雅地忍受结点错误

#### Worker Failure

Master 会周期性地 ping 每个 worker

* 如果在指定时间内，worker 没有回应，master 认为 worker 失效

Map 任务结束后，worker 会恢复空闲状态

* 可被用于执行其它的任务

任何发生错误的 worker 上的 map 或 reduce 任务也会变为空闲

并被重新调度

如果 worker 发生错误，已经执行完毕的 map 任务会被重新执行

* 因为保存在 worker 本地磁盘上的中间结果不可获得
* 已经执行完毕的 reduce 结果不必被重新执行，因为结果文件存放在全局文件系统中

如果 map worker 发生错误而被重新执行

所有的 reduce worker 都会被通知

MapReduce 能弹性应对大规模的 worker 出错

#### Master Failure

Master 周期性地将数据结果保存到检查点中

如果 master 发生错误

可以从最后一个检查点状态立刻恢复

#### Semantics in the Presence of Failures

如果用户提供的 `map` 和 `reduce` 操作是确定性函数

整个程序会按顺序执行

依赖 map 和 reduce 任务提交的 __原子性__ 来实现这个特性

* 每个任务将输出写入自己的私人临时文件中
    * 一个 reduce 任务一个文件
    * 一个 map 任务 R 个文件

当 map 任务完成时

Worker 向 master 发送消息，包含 R 个临时文件的名字

Master 在其数据结构中记录这些信息

当 reduce 任务完成时

Reduce worker 会原子地将临时输出文件重命名为最终输出文件

依赖底层文件系统的原子性重命名操作

保证文件系统的最终状态只包含一次 reduce 任务执行后的数据

> 对于非确定性执行？emm...

### 3.4 Locality

网络带宽是一个比较匮乏的资源

输入数据被 GFS 管理在一些机器的本地磁盘上

* GFS 将文件分为 64MB 的块
* 在集群中会存多份副本

考虑到这个因素

* Master 会优先把 map 任务调度到存储了输入文件副本的机器上
* 如果不行，则将 map 任务调度到与副本较近的机器上

这样，大部分输入数据都在本地被读取，从而不消耗网络带宽

### 3.5 Task Granularity

M 和 R 的数量最好远大于 worker 的数量

* 有利于动态的负载均衡
* 当 worker 出错时，加速恢复的过程
    * 已经完成的 map 任务分布在其它的机器上

Master 必须要做 O(M + R) 次决策

并保存 O(M * R) 个状态在内存中

通常应当指定 M

* 使得输入数据在 16-64MB 之间
* 使得局部性优化的效果最好

### 3.6 Backup Tasks

导致 MapReduce 的完成时间较长的机器 - straggler

* 需要很长时间才能完成 map 或 reduce 操作
* 磁盘错误影响性能
* Master 在该结点上调度了其它 MapReduce 任务，导致资源争用

Google 的解决方法：

当整个 MapReduce 任务快要完成时

Master 针对剩下的任务，调度 "备份" 执行

当主要执行和备份执行中的一个完成，整个任务就宣告完成

这个机制显著减少了 MapReduce 操作的时间

---

## 4. Refinements

### 4.1 Partitioning Function

MapReduce 的用户需要指定 reduce 任务的数量

中间结果经过 partition 函数划分

默认的划分函数是哈希 - `hash(key) mod R`

### 4.2 Ordering Guarantees

保证在一个 partition 中

所有的中间变量 key/value 会以 key 的升序被处理

这个排序保证了在每个 partition 中产生有序的输出

### 4.3 Combiner Function

可选的 Combiner 函数可以在 map 传递给 reduce 之前做部分的 merge

Combiner 函数在每一台进行 map 操作的机器上执行

输出会被写入中间结果文件中，发送给 reduce 任务

显著加速某些特定类型的 MapReduce 任务

### 4.4 Input and Output Types

支持读取不同格式的输入

用户可以自己实现 Reader 接口来支持新的输入类型

### 4.5 Side-effects

可以从 map 或 reduce 操作中产生一些额外的输出

### 4.6 Skipping Bad Records

Map 或 reduce 函数中可能存在 bug

导致 MapReduce 操作无法完成

提供了可选的模式

可以跳过这些坏的记录，使得操作可以继续进行

每个 worker 安装一个信号处理程序捕获错误

在 MapReduce 过程之前，先将一个序列号变量保存在全局变量中

如果用户代码产生了信号

信号处理程序会向 master 发送 "last gasp" UDP 包

如果 master 看到对一个特定的记录有超过 1 次失败

则会指示跳过这些记录

### 4.7 Local Execution

为了方便本地调试

提供了一种特殊的 MapReduce 实现

可以在本地机器上顺序执行所有的 MapReduce 操作

### 4.8 Status Information

Master 运行一个内部的 HTTP 服务器

使用户可以查看状态

* 计算的状态

还可以链接到每个任务的标准输出和标准错误

顶层状态页展示了哪些 worker 发生了错误

以及是在处理哪一个 MapReduce 任务时发生的错误

### 4.9 Counter

计数多种类型的事件

用户代码可以创建一个计时器对象

并在 map 或 reduce 操作中增加

在每个独立 worker 上的计时器会周期性地同步到 master

Master 从成功的 map 和 reduce 任务中收集计数值

当前计数值也可以在状态页上看到

Master 会负责处理同一个任务的重复执行，防止两次计数

有些计数器由 MapReduce 库自动维护

* 比如，处理了多少个输入，产生了多少个输出

---

## 5. Performance

使用了两个典型的 MapReduce 程序

代表了很典型的两类：

* 将同一个类型的数据从一种形式 shuffle 为另一种形式
* 从一个较大的数据集中提取一小部分

### 5.1 Cluster Configuration

1800 台机器的集群

### 5.2 Grep

在 10^10 条 100B 的记录中

寻找较少出现的三字节 pattern (92337 次)

输入被分为 64MB 的块，M = 15000，R = 1

启动开销：需要把程序发送到各个 worker 机器上

### 5.3 Sort

对 10^10 个 100B 记录进行排序

Map 函数提取 10B 的排序 key

将 key 和整个记录作为中间结果的 key/value pair

使用内置的 Identity 函数作为 reduce 操作

* 将中间结果直接作为最终的输出结果

Partition 函数利用记录的前 10B 作为 key 切分为 R 块

由于磁盘的本地性优化

* 输入速度比 shuffle 速度和输出速度高
* 大部分数据都是从本地磁盘直接读取，绕过了带宽受限的网络

Shuffle 速度比输出高

* 因为输出到 GFS 要写多个副本，保证可靠性和可用性

### 5.4 Effect of Backup Tasks

备份任务执行能够节约 44% 的时间

### 5.5 Machine Failures

故意从 1746 个 worker 进程中杀死了 200 个

底层的集群调度器立刻重启了新的 worker 进程

Worker 的死亡对输入速率带来了一定影响

* 因为一些已经完成的 map 工作需要重做

---

## 6. Experience

MapReduce 模型可以处理很多类问题：

* 大规模机器学习问题
* Google 产品的集群问题
* 从大量的查询中提取查询热点
* 提取网页信息
* 大规模图计算

更重要的是，可以在短时间内写出简单程序

可以在上千台机器上运行

并且可以让没有分布式并行系统经验的人轻松利用集群资源

---

## 7. Related Work

---

## 8. Conclusions

1. MapReduce 是一个简单易用的计算模型
2. 很大一部分问题能够用 MapReduce 轻易表示
3. Google 实现了这一模型，能够扩展到上千台机器的集群上

限制编程模型，能够轻易实现并行和分布计算

网络带宽是稀缺资源

* 本地性优化
* 将中间结果写入本地磁盘

减少了网络上传递的数据量

冗余执行能够减少慢机器带来的影响

并处理机器故障和数据丢失

---

## Summary

两年前已经用过了 Hadoop 的 MapReduce

但是并不知道底下是怎么转起来的

能搭上环境跑出结果已经不错了

今天看了一波原理

其实还是有点一知半解

但这套复杂的机制确实可以有效地操纵整个集群来并行执行任务

另外觉得 Google 和 Baidu 两家公司的情怀也差得太多了...

---

