# Outline

## FairFuzz: A Targeted Mutation Strategy for Increasing Greybox Fuzz Testing Coverage - ASE 2018

Created by : Mr Dk.

2019 / 08 / 05 15:59

Nanjing, Jiangsu, China

---

## 1. Introduction

_Coverage-guided greybox fuzzing (CGF)_ - 基于代码覆盖率指引的灰盒模糊测试

比如 _AFL_，被广泛用于测试：

* 浏览器 (Firefox, IE)
* 网络工具 (_tcpdump_, _wireshark_)
* 图像处理器
* 系统库 (_OpenSSH_)
* C 编译器 (GCC, LLVM)
* 解释器 (Perl, PHP, JavaScript)

基于一个现象：

* 覆盖率的提高，通常会带来更好的 crash 检测

CGF 通常从用户提供的 seed input 开始

* 通过 __字节__ 级的操作，对 seed 进行变异
* 测试变异后的用例，并收集代码覆盖信息
* 保存触发了新覆盖路径的输入
* 用保存的输入继续开始循环

虽然 CGF 产生的很多输入是没有用的

但由于较低的计算开销，产生输入的速度比符号执行好很多

AFL 的目标是越快地找到 crash 越好

其核心的搜索策略基于覆盖率反馈

* AFL 试图最大化代码覆盖率
* 因为在没有被覆盖的前提下，代码中的 crash 不会被发现

但是，文本的作者发现，AFL 经常无法覆盖程序的核心功能

* 从而无法发现这些代码区域中的 bug

本文提出了 FairFuzz，帮助 AFL 实现更好的覆盖

* 该工具在 AFL 常规的注解基础之上，不需要任何其它的注解
* 保证了 AFL 的易用性

FairFuzz 基于一种新的变异策略

* 提升了 AFL 之前产生的输入无法命中的代码区域的命中率

FairFuzz 的工作主要分为两步：

1. 识别程序分支中，很少被之前的输入命中的区域 - _rare branch_
   * 通过产生更多能命中这些分支的随机输入，显著提升代码覆盖率
2. 使用了轻量级的变异技术，增加命中 rare branch 的可能性
   * 基于一个观察：命中 rare branch 的输入中的特定部分对于进入该代码区至关重要
   * 因此输入中这样的特定部分不应当被变异
   * FairFuzz 通过进行一些小型的变异实验，识别了这些特定部分
   * 在输入产生中，避免对这些特定部分进行变异

本文作者在 AFL 的基础上实现了 FairFuzz

总体上，本文的主要贡献：

* 提出了新的轻量级变异策略，提升了命中 rare branch 的机会
* 在 AFL 的基础上实现并开源了 FairFuzz
* 对 FairFuzz 和几个不同的最先进的 AFL 版本进行了比较和评估

---

## 2. Overview

AFL 是一个流行的基于变异的灰盒测试工具

* 灰盒测试不像白盒测试那样需要分析源代码
* 灰盒测试也不像黑盒测试那样，灰盒测试需要收集反馈，来指引 fuzzing

### 2.1 AFL Overview

AFL 产生随机的输入，并对程序进行 fuzz

选择一部分之前产生的输入，对它们进行变异，获得新的输入

AFL 的初始输入：

* 程序
* 用户提供的 seed input

在一个循环中：

* 在队列中选择一个输入
* 选择一个输入进行变异
* 运行程序，并同时采集覆盖信息
* 如果触发新的覆盖，则加入队列中

AFL 的变异策略假设程序的输入是一个 __字节序列__

在变异过程中，主要有两个阶段：

1. deterministic stage
   * 对输入进行遍历
   * 对于输入的每一个位置，应用一次变异
     * bit filpping
     * byte filpping
     * arithmetic increment & decrement of integer values
     * replacing of bytes with _interesting_ integer values (0, *MAX_INT*)
   * 每轮产生的变异输入的个数由输入的长度控制
2. havoc stage
   * 应用一系列的随机变异
     * 将随机字节设置为随机值
     * 删除或克隆输入序列
   * 每轮产生的变异输入的个数由性能评分控制

### 2.2 AFL Coverage Calculation

覆盖信息是 AFL 的核心信息

AFL 只保留触发了新路径的输入

AFL 在程序中插入了注解

通过注解，将每个基本块关联一个随机数，作为该块的 unique ID

该块的 unique ID 被用于产生一对基本块之间转移的联线 unique ID

* `ID(A → B) = (ID(A) >> 1) ^ ID(B)`
* 右移是为了保证 `A → B` 和 `B → A` 的 ID 不同

将联线理解为控制流图中的 branch

程序的覆盖被表示为：`(branch ID, branch hits)`

* 代表了某个 branch 被执行了多少次
* AFL 所谓的发现了新的分支 - 即发现了新的 `(branch ID, branch hits)` pair

### 2.3 Limitations of AFL

虽说 AFL 的搜索策略是基于覆盖率的反馈

但 AFL 经常无法覆盖一些重要功能，从而无法检测这些功能中的 bug

比如，在 _libxml2_ 的 parser.c 文件中：

```c
if (CMP9(ptr, '<', '!', 'A', 'T', 'T', 'L', 'I', 'S', 'T')) {
    ptr += 9;
    while ((ptr != '>') && (ptr != EOF)) {
        int type = 0;
        if (CMP5(ptr, 'C', 'D', 'A', 'T', 'A')) {
            ptr += 5;
            type = XML_ATTRIBUTE_CDATA;
        } else if (CMP6(ptr, 'I', 'D', 'R', 'E', 'F', 'S')) {
            ptr += 6;
            type = XML_ATTRIBUTE_IDREFS\;
        } else if (CMP5(ptr, 'I', 'D', 'R', 'E', 'F')) {
            ptr += 5;
            type = XML_ATTRIBUTE_IDREF;
        } else if ((ptr == 'I') && ((ptr + 1) == 'D')) {
            ptr += 2;
            type = XML_ATTRIBUTE_ID;
        }
        
        if (type == 0) {
            ptr++;
            break;
        }
        
        if (CMP9(ptr, ......)) {
            
        }
        // ...
        
        ptr++;
    }
}
```

对于这个程序，在 AFL 上运行了 24h，并运行了 20 次

只有一次，AFL 产生了能够进入第一行的输入

并在进入该区域之后，无法再产生有效输入了

AFL 无法产生能够覆盖这些代码的输入的原因：

* AFL 并不关心哪部分输入需要被用于覆盖程序

比如，产生 `<!ATTLIST BD` 之后，AFL 不会优先对 `<!ATTLIST` 之后的部分进行变异，而是产生：

* `<!CATLIST BD`
* `<!!ATTLIST BD`
* `???!ATTLIST BD`

如果仅对 `<!ATTLIST` 之后的部分进行变异

能够提升 AFL 产生 `<!ATTLIST ID` 的概率，从而命中无法覆盖的代码

* 更小的变异搜索空间
* 更高的深入代码的概率

### 2.4 Overview of FairFuzz

大致方法分为两个部分

第一，识别守卫大量未覆盖代码的程序声明

这类声明通常很难被 AFL 产生的输入覆盖

因此很容易通过追踪命中每个分支的输入个数来识别

直观地说，如果分支被输入命中的次数越少，那么越有可能是一个 rare branch

识别 rare branch 之后，修改输入变异策略，使 rare branch 的条件被满足

* 在 deterministic 阶段大致估计出输入中不能被变异的部分
* 随后的突变中，不允许突变输入中的这些关键部分

从而显著提高了命中 rare branch 的输入的产生率

能够更好地探索被这些 rare branch 守卫的部分程序

---

## 3. FairFuzz Algorithm

### 3.1 Mutation Masking

`satisfies(x, T)` 为真，如果输入 x 满足测试目标 T

变异表示为元组 `(c, m)`

* m 为变异影响的字节数
* c 是下列变异策略之一
  * O - 从 k 位置开始，用一些值覆盖 m 个字节
  * I - 从位置 k 开始，插入 m 字节的序列
  * D - 从位置 k 开始，删除 m 字节

以 `mutate(x, μ, i)` 来表示

* x 为输入
* μ 为 (c, m)，即变异的字节数和变异策略
* i 为开始变异的位置

计算一个所谓的 _mutation mask_

直观上来说，mutation mask 指明了在 i 位置可以对 x 使用哪些变异策略

#### 3.1.1 Biasing Mutation with the Mutation Mask

`OkToMutate(mask(x, T), μ, k) = for (i = k...k+m-1), satisfies(mutate(x, (c, 1), i), T)`

该函数被用于过滤违反测试目标的变异输入

FairFuzz 和 AFL 一样，随机选择了突变策略，和突变作用的字节长度

* 但与 AFL 不同，突变的位置只能在 `ok-to-mutate` 的子集中随机选择
* 如果 `ok-to-mutate` 的位置为空，那么 FairFuzz 跳过本次迭代

### 3.2 Targeting Rare Branches

#### 3.2.1 Selecting Inputs to Mutate

为了使输入生成向 rare branch 偏移

FairFuzz 只选择命中 rare branch 的输入进行突变

设输入 x 命中 branch b 为 `hits(x, b)`，`hit count` 为输入测试该分支的次数

`numHits[b] = {x ∈ I : hits(x, b)}`

* `numHits` 相当于一个表，每次运行一个输入后都会更新

很自然的想法是：

* 将被命中最少的 n 个分支视为 rare branch
* 将命中数小于 p% 的分支视为 rare branch

在实际中，这两种方法都不稳定，且需要根据输入程序的变化而调整

解决方法：分支为 rare branch，当且仅当：

* `numHits[b] ≤ rarity_cutoff`
* `rarity_cutoff = 2^i, where 2^(i-1) < min(numHits[b']) ≤ 2^i`

rarest branch: `b* = argmin numHits[b]`

FairFuzz 只选择这样的输入：

* 该输入的 rarest branch 是一个 rare branch

#### 3.2.2 Computation of the Mutation Mask

对于一个给定的输入 x 和 rare branch b

如何计算 Mutation Mask

* 程序输入 x 和 rare branch b
* 对于 x 中的每一个位置 i，分别进行变异：
  * O - 从 i 开始覆盖字节
  * I - 从 i 开始插入字节
  * D - 从 i 开始删除字节
* 运行程序，如果变异后的输入命中了 b，就将位置 i 记录为 O/I/D

当然，对于 O 和 I 来说，覆盖或插入的字节是随机产生的，因此不一定能保证每次都命中 b

但是，搜索每个字节的代价过于昂贵

从经验主义来看，这种估计已经足够有效了

```c
void ComputeMask(Prog, input, branch) {
    mask = InitWithEmptySet(input);
    for (int i = 0; i < input.len; i++) {
        inputO = Mutate(input, flipByte, i)
        if branch ∈ BranchesHitBy(Prog, inputO) then
            mask[i] = mask[i] ∪ {O}
        inputI = Mutate(input, addRandomByte, i)
        if branch ∈ BranchesHitBy(Prog, inputI) then
            mask[i] = mask[i] ∪ {I}
        inputD = Mutate(input, deleteByte, i)
        if branch ∈ BranchesHitBy(Prog, inputD) then
            mask[i] = mask[i] ∪ {D}
    }
    return mask;
}
```

由此，FairFuzz 的总体算法：

```c
void FairFuzz(Prog, Seeds)
{
    Queue = Seeds;
    while (true) {
        for (input in Queue) {
            rarestBranch = RarestHitBy(Prog, input, numHits);
            if (numHits[rarestBranch] > rarity_cutoff) {
                // rarest branch, but not a rare branch
                continue;
            }
            score = PerformanceScore(Prog, input);
            mask = ComputeMask(Prog, input, rarestBranch);
            for (int i = 0; i < input.len; i++) {
                for (mutation in deterministicMutationTypes) {
                    if (!OkToMutate(mask, mutation, i)) {
                        // 违反了进入分支的规则
                        continue;
                    }
                    newInput = Mutate(input, mutation, i);
                    RunAndSave(Prog, newInput, Queue);
                }
            }
            for (int i = 0; i < score; i++) {
                newInput = MutateHavoc(input);
                RunAndSave(Prog, newInput, Queue);
            }
        }
    }
}

void MutateHavoc(Prog, input)
{
    numMutataions = RandomBetween(1, 256);
    newInput = input;
    for (int i = 0; i < numMutations; i++) {
        mutation = RandomMutationType();
        position = RandomOkToMutate(mask, mutation);
        newinput = Mutate(newInput, mutation, position);
    }
    return newInput;
}

void RunAndSave(Prog, input, Queue)
{
    runResults = Run(Prog, input);
    for (branch in runResults) {
        numHits[branch] += 1;
        if (NewCoverage(runResults)) {
            AddToQueue(input, Queue);
        }
    }
}
```

### 3.3 Trimming Inputs for Testing Targets

AFL 的高效来源于其能够快速产生和修改输入

FairFuzz 在计算 Mutation Mask 也需要高效

计算的复杂度与输入长度呈线性关系

因此 FairFuzz 需要保持队列中的输入的长度较小

AFL 中有两种策略，保证输入较小：

1. 优先选择较短的输入进行变异
2. 在变异前对 parent input 进行剪枝
   * 对输入进行最小化，找到能够命中该分支的最短输入

但这些方法很难在输入较长的条件下，降低输入的长度

FairFuzz 只是基于输入是否命中 rare branch 而选择输入

因此 FairFuzz 可以对剪枝约束适当宽松

---

## Summary

我累了 这篇论文我不想再看下去了...... 😖

---

