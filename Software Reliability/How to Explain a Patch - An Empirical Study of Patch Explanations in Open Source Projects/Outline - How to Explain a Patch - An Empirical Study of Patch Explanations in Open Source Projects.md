# Outline

## How to Explain a Patch: An Empirical Study of Patch Explanations in Open Source Projects

Created by : Mr Dk.

2020 / 12 / 06 17:16

Nanjing, Jiangsu, China

---

这篇文章主要分析了组成一个 patch 的要素。通过分析几个不同类型、不同性质的 Java 项目：

* *RxJava* - 异步事件库
* *Spring* - Java Web 框架
* *PocketHub* - *GitHub* 的 Android 客户端
* *Nextcloud* - 用于文件共享和传输的 Android 客户端
* *IntelliJ* - Java IDE

通过分析这些项目的代码仓库中的 pull request，归纳得到了 patch 中的要素，以及每种要素的表示形式：

1. Condition - bug 被触发的的条件 (可以是具体的或抽象的)
   * 通过特定的状态 (变量等于某个值 / 变量属于某种类型) 来形容条件
   * 通过一系列事件 (调用某个函数 / 点击某个按钮) 来形容条件
2. Consequence - 描述 bug 导致的非预期结果
   * 预期中的结果
   * 实际上的结果
3. Position - bug 出现或被修复的位置
   * 文件
   * 内部类
   * 函数
   * 变量
   * 模块 (抽象)
4. Cause - 解释为什么 bug 会在特定位置触发
   * 缺失处理 (漏掉了对于一些特定输入的处理)
   * 错误处理
5. Change
   * 插入
   * 替换
   * 删除

另外，文章还分析了现实生活中的 pull request 中，上述五个要素出现的频率。结果表明，大部分 PR 只包含其中的两三个要素 - 因为这五个要素提供的信息实际上有一部分的重叠。另外，根据 bug 类型的不同，PR 中出现的要素类型也不相同。

文章还分析了每一个要素中，不同表达形式的分布。

---

