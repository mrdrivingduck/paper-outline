# Outline

## Fast and Precise On-the-fly Patch Validation for All

Created by : Mr Dk.

2020 / 12 / 08 14:55

Nanjing, Jiangsu, China

---

这篇文章提出了在 APR 领域中加速 **补丁验证** 过程的新方法。

在传统 APR 的补丁验证过程中，在验证每一个 patch 时，首先需要编译整个工程 (或至少重新编译被修改的类)，然后启动一个 JVM 进程来运行程序并验证结果。这个过程中包含了过多的重复开销。本文 (确切地说不是本文) 提出了一种基于 *HotSwap* 热更新技术的补丁验证方法，在无需重启 JVM 的前提下，动态重新加载被修改过的类，从而提升补丁验证的性能。这种不重启 JVM 的方法被称为 *on-the-fly*。

## Java Agent and *HotSwap*

*Java Agent* 是一个编译后的 Java 程序 (一个 JAR 文件)，与 JVM 共同运行，能够 **截获并修改** JVM 要加载的字节码文件。有两种方式可以使用 Java Agent：

* 静态 - 实现一个代理类，然后在启动 Java 程序时加入 `-javaagent` 参数指定这个代理类
* 动态 - 在程序运行期间，用另一个 JVM 进程 attach 到程序运行的 JVM 进程上

Java Agent 实际上提供了 JVM 级别的 AOP 实现方式。比如，静态的代理类中需要实现如下函数：

```java
public static void premain(String agentArgs, Instrumentation inst);
public static void premain(String agentArgs);
```

其中，`agentArgs` 传入了程序运行参数，而 `Instrumentation` 则提供了对类定义进行修改的接口。可以实现一个具有字节码加载、替换功能的函数并注册到 `Instrumentation` 对象上。每当 JVM 加载一个类时，都会调用这个函数。

## Approach

有了上述技术的支持，整体流程显而易见：

1. 编译整个工程，启动 JVM，将所有的字节码文件加载到 JVM
2. 增量式地编译打过补丁 (修改过的) 类文件
3. 备份原有的未被修改的类文件 (用于复原)，JVM 重新加载被打过补丁的类文件
4. 执行测试 / 验证
5. 复原被打过补丁的类文件，JVM 重新加载这些类
6. 复原 JVM 中的全局变量，为下一个补丁的验证提供一个干净的 JVM 环境

## JVM Reset

上述流程中有一个问题：JVM 没有被重新启动，如果每一个补丁只修改了自己创建的对象，则不会有任何问题；如果某个补丁修改了 JVM 中的全局变量 (比如某些类的 `static` 域)，而后一个补丁恰好需要使用这个全局变量，那么前一个补丁的执行将会影响下一个补丁的执行，论文中称这个问题为 **污染**。理论上，每一个补丁的执行环境都应当是一样的。因此，在两个补丁执行之间，既然不重启 JVM，就一定需要使 JVM 复原到一个干净的状态。

或许，将所有的 `static` 域重新 reset 就可以解决问题。但是其中的技术挑战在于：

1. 一些 `static` 域是 `final` 的，无法被直接复原，`static` 域之间存在数据关联性；只有重新调用类的构造函数才可能复原这些值；而根据 JVM 规范，只有 JVM 才能调用构造函数
2. 类的构造函数之间也有依赖性，因此需要重新调用所有依赖类的构造函数才能恢复 JVM 状态

作者选择在用户空间 (相对于 JVM 来说) 模拟 JVM 的类初始化来实现 JVM 状态的复原。

### Static Pollution Analysis

通过轻量级的静态分析，找出所有被 *污染* 的字节码文件 (即全局状态被一次补丁运行修改的字节码文件)，包含：

* 应用程序代码
* 第三方库代码
* JDK 库代码 (可以通过 JDK 提供的 API 很方便地重置状态)

### Runtime Bytecode Transformation

根据 Java 语言规范，类初始化函数会在以下几个时候被调用：

* 类的实例被创建
* 类的静态函数被调用
* 类的静态变量被赋值
* 类的静态变量被使用，且该变量不是常量
* 类是一个顶层类，且类中的 `assert` 语句被执行

在字节码层面，将类的 `<clinit>()` 重命名为另一个名字 A，然后在以上五个地方分别调用 A；另外重新加一个 `<clinit>()`，里面啥也不干，只调用被重命名后的 A (因为 JVM 需要这个 `<clinit>()` 函数)。由于 JVM 只会初始化一个类一次，而 A 可能会在上述五种情况下被触发多次，因此作者保证了每次补丁执行期间动态判断，每个类只会被初始化一次。使用一个 `ConcurrentHashMap<Class, boolean>` 来实现。对于 JDK 自身的类，则使用 JDK 自身的 API 来复原类的状态。

### Dynamic State Reset

在每个补丁执行前，所有的类在 map 中都被标记为 `false`，代表未被初始化；在重新加载该类并初始化后，相应的 Class 对象将会在 map 中被标记为 `true`，之后不会再进行初始化。

## Experimental Result

通过实验，肯定了 *HotSwap* 技术对补丁验证的性能带来的提升：补丁的数量越多，带来的性能提升越明显。同时，论文也确认了该技术带来的副作用：补丁执行期间对 JVM 全局状态带来的破坏，导致了一些补丁没能被成功验证。在使用了论文中的 JVM 状态复原技术后，这些没能被成功验证的补丁最终也被成功验证了。

## References

[简书 - Java Agent 简介](https://www.jianshu.com/p/63c328ca208d)

[简书 - ☆ 基于 Java Instrument 的 Agent 实现](https://www.jianshu.com/p/b72f66da679f)

---

