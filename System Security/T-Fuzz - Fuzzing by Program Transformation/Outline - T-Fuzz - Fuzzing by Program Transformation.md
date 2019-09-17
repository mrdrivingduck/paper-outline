# Outline

## T-Fuzz: Fuzzing by Program Transformation - S&P 2018

Created by : Mr Dk.

2019 / 07 / 30 19:52

Nanjing, Jiangsu, China

---

## Introduction

Fuzzer 基于输入如何被产生，可以被粗略分为两类：

* Generational fuzzers
  * 根据给定的格式说明产生输入
  * 需要人为给定输入格式
* Mutational fuzzers
  * 随机突变分析人员提供的或随机产生的 seed

Fuzzing 是一个动态技术

为了找到 bug，必须触发包含这些 bug 的代码

Mutational fuzzing 受限于代码覆盖

Fuzzer 产生的输入绕过复杂合法性验证的概率极低

主要原因：fuzzer 不了解程序期望的输入格式

* 这种固有的局限导致 fuzzer 很难触发由合法性验证保护的代码路径
* 和 deep bugs

Fuzzer 通过采取一些方法来更好地修改输入

满足程序中复杂的检测

AFL 使用代码覆盖来指引突变算法

* 为了绕开对输入程序的合法性检查
* AFL 使用代码覆盖反馈来推断文件中 magic value 的值和位置

一些最近的工作使用符号执行或污点分析来提升代码覆盖

* 产生能够绕开合法性检查的目标程序

然而，最先进的 fuzzer，如 AFL 和 Driller

在流行的威胁分析测试数据集中，只能找到不到一半的 bug

最近的研究中集中于找到新的方法，对输入进行产生和突变

然而，没有必要只对程序输入进行突变

程序本身也可以被突变，以协助 bug 的寻找

基于这个直觉，本文提出了 _Transformational Fuzzing_

* 通过禁用程序的输入合法性检查，提升 bug 的寻找能力

对于动态代码覆盖的问题

不是使用费时的重量级程序分析技术，产生绕开输入检查的代码

本文只是简单地检测并禁用合法性检查

从而可以探索原本由输入检查保护的代码空间

当然，移除合法性检查可能会破坏原程序的逻辑

* 因此这些 bug 可能包含 false positives
* 从而过度干扰了分析人员

本文开发了基于符号执行的后处理分析

* 分析过滤后剩下的输入能够在原程序中触发真正的 bug

虽然这个过程较为复杂和重量级

但只需要在检测之后进行，并不会拖慢分析过程本身

T-Fuzz 使用已有的 coverage-guided fuzzer 来搜索程序

当 fuzzer 无法再产生能够触发新的代码路径的输入时

一个轻量级的动态追踪方法可以用于发现所有的输入检查

通过选择性地禁用这些检查来变换程序

Fuzzing 得以继续进行

与已有的基于符号执行的方法相比

* T-Fuzz 的可扩展性更好，不会被绕开输入检查的需求影响
* 能够对被 hard 检测保护的代码路径进行覆盖

评估：

* 一个恶意程序数据集 - DARPA Cyber Grand Challenge (CGC)
* LAVA-M 数据集
* 4 个被广泛依赖的 library

经过验证，证明了：

* Transformation fuzzing 的有效性
* T-Fuzz 对于 hard 的输入检测，比如校验和，具有较好的支持
* T-Fuzz 有能力过滤 false positives，代价是 6%-30% 的 false negatives

总体上，论文的贡献如下：

1. Transformation fuzzing 能有有效找到 bug，不使用重量级的程序分析技术
2. 使 fuzzing 能够同时修改输入和程序
   1. 对合法性检查进行自动检测
   2. 使用程序变换，移除检测到的合法性检查
   3. 通过过滤 false positives，重现原程序中的 bug
3. 在 CGC 数据集、LAVA-M 数据集和 4 个程序中评估了 T-Fuzz
4. 找到了 3 个新 bug

---

## Motivation

目前先进的 coverage-guided fuzzer 使用代码覆盖作为反馈

指引突变算法产生输入

特别地，这类 fuzzer 会追踪触发新的代码路径的输入

在产生新输入的同时，对这些输入进行突变

然而，由于不知道程序期望的输入格式

这种突变很难产生能够通过复杂合法性检查的输入

* 如果 fuzzer 不能绕过合法性检查，就被卡住
* 不断产生新的输入，但无法覆盖新的代码路径

例子：

在一个程序中，有 bug 的代码被几层合法性检验代码保护

只要任意一个合法性检验不通过

深层存在 bug 的代码就不会被触发

Fuzzer 很难在没有其它技术的帮助下绕过合法性检验

而符号执行可以快速绕过简单的判断

却无法绕过较为复杂的校验和算法

作者提出了几点发现：

1. 合法性检查可以被分为两类：
   1. NCC (Non-Critical Check)
      * 过滤一些无关的数据
   2. CC (Critical Check)
      * 检查对于程序功能至关重要的值
2. NCC 可以被直接移除，不会引发任何的 bug
   * 移除 NCC 使得 fuzzing 简化 - 由检查保护的代码被暴露
3. 在转换后的程序中发现的 bug 能够在原程序中被重现
   * 通过使输入满足绕开的合法性检查中的条件
4. 移除 CC 将会引发 bug，并且无法在原程序中被重现
   * 这类 false positives 需要在后处理阶段被过滤
   * 保证只有在原程序中能被重现的 bug 被报告

NCC 在程序中很常见

* 在 Unix 系统中，所有常见的文件格式使用前几个字节作为 magic value
* 在网络程序中，校验和用于广泛检测数据的损坏

基于这些观察，T-Fuzz 通过检测并移除程序中的 NCC 来提升 fuzzing 效果

通过移除 NCC，由它们保护的代码路径将会被暴露

从而使潜在的 bug 被发现

T-Fuzz 还能够帮助 fuzzer 覆盖被 hard 合法性检查保护的代码

---

## T-Fuzz Intuition

### Fuzzer

T-Fuzz 使用已有的 coverage-guided fuzzer - AFL

T-Fuzz 依赖于这些 fuzzer 来追踪代码覆盖路径

和 fuzzing 是否卡住的实时状态信息

所有引发 crash 的输入将被记录，用于未来分析

### Program Transformer

当 fuzzer 卡住时

T-Fuzz 调用 Program Transformer 产生变换后的程序

首先，最终输入程序，检测所有的 NCC 候选

然后变换原程序的一份拷贝，移除检测到的 NCC 候选

### Crash Analyzer

对于变换后程序产生的 crash 输入

Crash Analyzer 使用基于符号执行的技术过滤 false positives

算法 1：

* 在每轮迭代中，T-Fuzz 从队列中选择一个程序，并进行 fuzzing，直到无法产生新的 coverage
* 使用卡住之前的程序检测其它的 NCC，并产生多个变换后的程序，加入队列
* 在每轮迭代中发现的 crash 由 Crash Analyzer 进行后处理，识别所有的 false positives

---

## T-Fuzz Design

总体分为三步：

1. Detecting NCCs
2. Program Transformation
3. Filtering out False Positives and Reproducing true bugs

### A. Detecting NCCs

在 T-Fuzz 中，将 NCC 定义为一个过度估计的概念：

* Fuzzer 产生的输入无法绕开的所有检查

可以使用复杂的 data flow 或 control flow 分析来寻找合法性输入

* 这种方法具有较好的准确率，但包含较大开销

T-Fuzz 利用一种不准确、轻量级的动态追踪技术来检测

合法性检查在程序中会被编译为条件分支指令

* 在 CFG 中被表示为一个原 basic block 和 2 条输出边
* 每条输出边对应了条件的 `true` 或 `false`
* 无法绕开合法性检查意味着 fuzzer 产生的输入只进入了 T 或 F 边

在 T-Fuzz 中，将 CFG 中所有的 boundary edges -

* 连接所有基本块结点的边

作为 NCC 近似候选

假设程序 CFG 中的所有结点为 N，所有边为 E

* `CN` - 所有被 fuzzer 产生的输入执行过的结点
* `CE` - 所有执行过的边

所谓的 boundary edges 可被形式化为：

1. `e` 不在 `CE` 中
2. `e` 的源节点在 `CN` 中

在算法上，T-Fuzz 首先采集 fuzzer 产生的输入执行到的所有 node 和 edge

T-Fuzz 使用动态追踪技术，得到输入产生的累积 node 和 edge 的覆盖信息

构造程序的 CFG，迭代所有的边

返回所有满足 NCC 候选条件的边

### B. Pruning Undesired NCC Candidates

使用上述算法得到的 NCC 候选是一种过度估计

其中可能包含不需要的合法性检查

需要将这些候选剪枝

1. 上述算法在所有的执行或未执行的代码中检测 NCC 候选
   * Fuzzing 只对可执行程序和特定的库感兴趣
   * 剪枝所有不在这些对象中的候选
2. 用于立刻结束程序的错误代码检测
   * 由于错误很少发生，错误检测代码没有被执行，从而被检测为 NCC 候选
   * 移除这类 NCC 只会导致错误处理代码被执行
3. 由于程序在检测到错误之后，通常会在很短的代码路径内结束
   * 启发式地使用检测到的 NCC 候选者之后跟随的基本块数量
   * 定义一个阈值，来辨别错误处理代码路径
   * 把注意集中在会导致大量 coverage 增加的 NCC 上，而不是会迅速终结程序的 NCC

### C. Program Transformation

考虑了集中不同的选择来移除检测到的 NCC

* 动态二进制标注
  * 较大开销
* 静态二进制重写
  * 由于 CFG 的改变，导致了额外的复杂度
* 翻转条件跳转语句的条件
  * 直截了当、折衷

维护了程序中翻转的路径条件

原程序中的路径条件可以被轻易恢复

T-Fuzz 通过将检测到的 NCC 替换为相反的条件来变换程序

* 维持了原程序的结构
* 保持了恢复路径条件的必要信息

由于 basic block 的地址与原程序中相同

对于转换后程序的追踪可以直接与原程序相映射

显著减少了分析变换程序和原程序之间区别的开销

算法 4：

* 输入：用于转换的程序、需要移除的 NCC 候选
* 由于每个 basic block 中最多只有一条跳转指令，系统扫描 source block 中的所有指令
* 将扫描到的第一个条件跳转指令替换为相反的指令
* 每条被修改后的指令的地址被记录和返回

### D. Filtering out False Positives and Reproducing Bugs

被移除的 NCC 可能在原程序中是有意义的

移除这些 NCC 将会带来新的 Bug

因此，T-Fuzz 的 Crash Analyzer 验证每个在变换后程序中出现的 bug 在原程序中是否也存在

从而过滤 false positives

对于剩下的 true positives，一个用于重现 bug 的原程序被产生

Crash Analyzer 使用技术采集原程序从入口到 crash 的路径依赖

这些依赖能否被满足，决定了 crash 是否是 false positives

如果依赖能被满足，那么 Crash Analyzer 通过满足这些依赖

产生能够使原程序发生 crash 的输入，并重现 bug

Crash Analyzer 维护两个约束集合

* 一个用于追踪转换后程序中的约束 - `CT`
* 一个用于追踪原程序中的约束 - `CO`

导致 crash 的输入表示为 `I`

如果 basic block 中包含一个否定的条件跳转

* 那么翻转后的路径约束被加入 CO
* 否则约束被直接加进 CO

当追踪达到最终 crash 的指令

Crash 的原因被编码为一个约束，并加到 CO 中

如果 CO 中的约束能够被满足

这意味着可能能够产生一个触发原程序发生相同 crash 的输入

否则就被标记为 false positive

Crash Analyzer 的输入：

* 变换后的程序
* 条件跳转的地址
* 导致 crash 的输入
* 变换后的程序中 crash 的地址

随着追踪的进行，将反置的约束存入 `CO` 中

最后，`CO` 中的约束被检查

如果约束不可被满足，那么输入被标记为 false positive

否则 CO 中的约束可被用于产生输出，在原程序中重现 bug

### E. Running Examples

```c
void main() {
    int x, y;
    read(0, &x, sizeof(x));
    read(0, &y, sizeof(y));
    if (x == 0xdeadbeef) {
        *(int *)y = 0;
    }
}
```

含义：

* 程序接受一个用户输入作为检测条件
* 程序接受第二个用户输入，将其作为指针，向对应内存单元写入一个值

算法检测到 NCC 候选

Program Transformer 产生变化后的程序 - `x != 0xdeadbeef`

假设本次输入为 `x = 0, y = 1`

从而，fuzzer 能够绕开合法性判断

`┐(x != 0xdeadbeef)` 被加入 `CO` 中

对应的内存写操作将被视为 crash 条件，因此 `y == 1` 被加入 `CO`

由于 `{┐(x != 0xdeadbeef), y == 1}` 是可以被满足的

因此这个 crash 是一个 true bug，输入 `x = 0xdeadbeef, y = 1` 可被用于重现 bug

```c
void main() {
    char secrets[4] = "DEAD";
    int index;
    read(0, &index, sizeof(index));
    
    if (index >= 0 && index <= 3) {
        output(secrets[index]);
    }
}
```

含义：

* 内存中存在一些秘密数据
* 从用户空间读取一个索引，取出对应的秘密
* 为了防止 out-of-bound，需要在合法范围内检查 index

算法将会产生一个对应条件相反的程序 - `index < 0 || index > 3`

导致相反的条件被加入 `CO` 中 - `index >= 0 && index <= 3`

对转换后的程序进行 fuzzing

得到了一个导致 crash 的输入 - `index = 0x12345678`，并加入 `CO` 中

`CO` 中的约束 `{index >= 0 && index <= 3, index == 0x12345678}` 无法被满足

总而被报告为 false positive

---

## Implementation and Evaluation

### A. DARPA CGC Dataset

该数据集包含一个易受攻击的程序集合

共有 248 个 challenges，包含 296 个二进制程序

对于 challenge 中的每个 bug

数据集提供用于验证的输入集

这个数据集中的包含大量合法性检查

三种实验配置：

* AFL - 评估 T-Fuzz 的启发式方法
* Driller - 评估 T-Fuzz 中基于符号执行的方法
* T-Fuzz

#### Comparison with AFL and Driller

* T-Fuzz - `166/296`
* Driller - `121/296`
* AFL - `105/296`

AFL 发现的所有 bug 都被 Driller 和 T-Fuzz 发现

T-Fuzz 发现的所有 bug 都是 true positives

在 166 个二进制程序中，有 45 个包含复杂的合法性检验

很难通过解决约束来产生输入

Driller 无法绕开 hard 合法性检查

而 T-Fuzz 在 fuzzing 卡住后，会移除对应的条件检查，cover 之前被保护的代码

T-Fuzz 没能发现 Driller 发现的 10 个 bug

是由于 T-Fuzz 目前实现的两点局限导致的：

1. 如果移除 NCC 带来的 bug 发生在了 true bug 的执行路径上，那么转换后程序的执行将会提前终结，从而不会执行 true bug 的代码
2. True bug 藏在过深的代码路径中，导致“转换爆炸”，需要 fuzzing 太多版本的转换程序

T-Fuzz 比 Driller 发现了 45 个额外的 bug

* 这些 bug 都由 hard 检测保护，藏在较深的代码层中

#### Comparison with Other Tools

需要人为工作，或无法运行整个数据集

### B. LAVA-M Dataset

该数据集包含一个受威胁程序的集合

通过自动注入 bug 产生

与 Steelix 相比

T-Fuzz 在 `base64` 和 `uniq` 程序中发现差不多相同数量的 bug

在 `md5sum` 中发现了更多的 bug，在 `who` 中发现了更少的 bug

* `base64`、 `uniq`、`who` 中，被注入的 bug 全部都由合法性检查保护，数值全部从输入中拷贝，而不是在程序中硬编码的 magic bytes；
  * 因此静态分析工具能够轻易恢复合法性检查中的 value
* LAVA-M 数据集提供了格式化的种子，使 fuzzer 能够到达注入 bug 的代码路径
* 如果任意两个条件之一不满足，Steelix 和 VUzzer 的表现将会变差

T-Fuzz 能在被 hard 检查保护的代码路径中触发 bug

这些 bug 被 MD5 的计算值保护

而这个值很难使用硬编码的数值进行构造

所以静态分析 fuzzer 无法绕开这种合法性检查

总体上，T-Fuzz 与最先进的 fuzzer 相比表现更佳

尤其当输入中存在 hard 检查时

T-Fuzz 能够更好地 fuzzing 由这些检查保护的代码

### C. Real-world Programs

比较了 T-Fuzz 与 AFL

由于输入是随机产生的 seed

AFL 很快就卡住了

无法绕过文件类型字节的合法性检查

此外，这些大型程序还体现了底层符号执行引擎的局限：

* 无法扩展支持这些程序运行的环境

### D. False Positive Reduction

对于每个 true positive bug

T-Fuzz 为 Crash Analyzer 平均报告 2.8 次警告

与静态技术对比，即使没有 Crash Analyzer

分析人员只需要分析三个警告就能定位一个 bug

此外，Crash Analyzer 会将确实能引发 bug 的检测结果标记为 false positive

从而导致 false negative

在 checksum 的例子中会出现

### E. Case Study

```c
int main() {
    int step = 0;
    Packet packet;
    while (1) {
        memset(packet, 0, sizeof(packet));
        if (setp >= 9) {
            char name[5];
            // stack buffer overflow BUG
            int len = read(stdin, name, 25);
            printf("Well done, %s\n", name);
            return SUCCESS;
        }
        // read a packet from the user
        read(stdin, &packet, sizeof(packet));
        // initial sanity check
        if (strcmp((char *)&packet, "1212") == 0) {
            return FAIL;
        }
        // other trivial checks on the packet ommtted
        if (compute_checksum(&packet) != packet.checksum) {
            return FAIL;
        }
        // handle the request from the user, e.g., authentication
        if (handle_packet(&packet) != 0) {
            return FAIL;
        }
        // all tests in this step passed
        step++;
    }
}
```

T-Fuzz 在 1h 的常规 fuzzing 时间后，卡在了校验和的合法性检查处

T-Fuzz 停止了 fuzzing，并进行 NCC 的检测、剪枝

返回了一个含有 9 个 NCC 候选的集合

T-Fuzz 产生了九个不同的二进制程序，每个程序禁用了其中一个 NCC 候选

它们按照先进先出的顺序被 fuzzing，从而找到 true bug

而基于符号执行的 fuzzer 由于合法性检查过于复杂而无法产生能够绕开检查的输入

---

