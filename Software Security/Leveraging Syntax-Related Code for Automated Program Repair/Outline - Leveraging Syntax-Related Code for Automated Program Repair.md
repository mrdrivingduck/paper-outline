# Outline

## Leveraging Syntax-Related Code for Automated Program Repair

Created by : Mr Dk.

2020 / 04 / 29 17:20

Ningbo, Zhejiang, China

---

## Abstract and Conclusion

本文提出利用来自代码数据库中，与缺陷程序存在 **语法相关** 的已有代码，来修复程序中的缺陷。具体方式，通过缺陷定位，找到一些疑似存在缺陷的 statement。对于每个疑似 statement，识别出一个代码块，其中包含 statement 以及相关的上下文信息。然后，在代码数据库中寻找 **语法相关** 的匹配代码块，并利用这些匹配到的代码块产生补丁。

本文指出，用于进行修复的代码片段可能存在于错误程序本地，也可能存在于非本地。已有的思路是使用 **基于语义的代码搜索**，但是代价比较昂贵。本文提出 **基于语法的代码搜索** (结构相似、概念相关)，更加轻量。

## Overview

### Fault Localization

缺陷定位不是本文的重点。使用 *GZoltar 0.1.1* 进行定位。

### Code Search

首先，对于定位到的缺陷 statement，产生代码块 *tchunk* (其中包含 statement + 一些上下文信息)。这个代码块不能太小也不能太大 - 太小就不足以包含足够的上下文信息；太大的话，这个代码块就过于 unique 了，无法与代码数据库进行匹配。因此本文选择了 40 个 token 左右的代码段。

第二部，从 *tchunk* 中提取特征 - k-gram tokens 和 conceptual tokens，并使用 *Apache Lucene* 搜索引擎在代码数据库中进行搜索，找到相似的 *cchunks*。

### Patch Generation

在找到 *cchunks* 以后，就可以进行补丁的生成。首先，将 *cchunk* 中的一些标识名 (变量名、类型名等) 换成与 *tchunk* 相同。然后根据两个代码块中的语法差异，对 *tchunk* 进行操作 - 替换、插入、删除。

### Patch Validation

运行 test cases 对产生的补丁进行验证。

## Methodology

### Code Search

提取缺陷代码附近最多 40 个 token 左右 (或大约 5-7 行) 代码作为 *tchunk*。然后利用搜索引擎提取出相应的 *cchunks* (引擎中自带了一套相似度衡量机制)。

提取 *tchunk* 中的 **k-gram tokens** 和 **conceptual tokens**：

* K-gram token - 对非 JDK 的变量或域进行符号化，然后将 token 序列的拼接为固定长度但内容不同的子序列，比如，对 `str.charAt(1) == 'e'`，产生如下的子序列：
    * `$v$.charAt($ln$`
    * `.charAt($ln$)`
    * `charAt($ln$)==`
    * `($ln$)=='e'`
* Conceptual tokens - 将代码块文本 token 化，只包含 Java 标识符，比如，对 `str.getChars(0, strLen, buffer, size)` 序列化为：
    * `str`
    * `getchars`
    * `chars`
    * `strlen`
    * `str`
    * `len`
    * `buffer`
    * `size`

然后对 *cchunks* 进行匹配，保留与 *tchunk* 匹配度高于 `n/8` 个 token 的 *cchunk*。

### Patch Generation

之后将 *tchunk* 与 *cchunk* 中的变量进行映射匹配，然后将 *cchunk* 中的标识名翻译为 *tchunk* 中的标识名。

接下来对 *tchunk* 进行补丁生成。其中，替换的优先级高于插入，插入的优先级高于删除。另外，较小的补丁优先级更高。这种优先级来源于一些已有工作：

* 删除代码的修复方式经常导致过拟合 (导致原来的 pass case 运行失败)
* 简单的补丁过拟合的可能更小

根据优先级对补丁进行排序，并去除冗余补丁。选择排名前 50 个补丁进行验证。

---

