# Outline

## DIFUZE - Interface Aware Fuzzing for Kernel Drivers - CCS 2017

Created by : Mr Dk.

2019 / 08 / 05 15:59

Nanjing, Jiangsu, China

---

[TOC]

---

## 1. Introduction

Android 内核中的漏洞已经从 2014 年的 4% 上升到 2016 年的 39%

内核被分为两个部分：

1. 核心内核代码
   * 通过系统调用 (syscall) 接口访问
2. 设备驱动
   * 在服从 POSIX 的系统上，通过 `ioctl` 接口进行访问
   * 这种接口也被实现为一种系统调用
   * 使输入能够被设备驱动接收和分发

Google 指出，已报告的 Android 内核漏洞中，85% 是第三方编写的驱动代码

由于使用的移动设备越来越多，自动化设备驱动中的威胁至关重要

虽然自动化的系统调用分析已经正在被研究

但 `ioctl` 却被忽视

这是由于系统调用遵循一个统一、规范的接口

而与 `ioctl` 的交互根据设备的不同而不同

`ioctl` 接口包含：

* 对于每一条合法的命令
* 都有结构化的参数
* 命令和参数数据结构是驱动独立的

使设备很难被分析

对于设备的分析应当是 interface-aware 的

* 应当使用 `ioctl` 可以接收的命令和数据结构来进行交互

本文提出的 DIFUZE 可以支持 interface-aware fuzzing

促进对 `ioctl` 接口的动态分析

DIFUZE 通过对内核驱动代码的自动化静态分析

恢复特定的 `ioctl` 接口，包含：

* 合法的命令
* 相关的数据结构

使用恢复后的接口产生 `ioctl` 的输入

由用户空间程序注入内核

这些输入能够匹配驱动能够接收的命令和数据结构

从而使 `ioctl` 能够被有效、深入地测试

被恢复出的 `ioctl` 接口能够使 fuzzer 在突变输入时产生有意义的数值

在本文实验中，分析了 7 款移动设备，发现了 36 个威胁

其中 32 个是之前未知的

总体上，本文的贡献：

* Interface-aware fuzzing
  * 能够对接口敏感的目标，比如 POSIX 系统的 `ioctl` 接口进行 fuzzing
* 自动化的驱动分析
  * 自动分析内核驱动代码
  * 识别所有的 `ioctl` 入口点、参数结构、设备文件名
* DIFUZE 原型
  * open-source

---

## 2. Background and Related Work

### 2.1 POSIX Device Drivers

POSIX 标准制定了用户空间与设备驱动交互的接口

* 接口支持通过 __设备文件__ 来与驱动进行交互
* 这些文件是内核驱动的用户层表示形式

用户空间通过 `open()` 系统调用获得设备文件的一个 handler

然后就可以使用这个 handler 与设备进行交互了

不同的设备需要不同的系统调用来实现它们的功能

比如：`read()`、`write()` 和 `seek()` 被用于硬盘设备文件操作

对于语音设备，`read()` 和 `write()` 可能对应于不同的用途

而 `seek()` 则可能没有被使用

然而，有一些传统的功能通过传统的系统调用无法实现

* 比如，对于语音设备，用户空间应用如何配置采样频率呢？

这些操作由 POSIX 标准的 `ioctl` 接口支持

* 这种接口允许驱动暴露难以用文件形式表示的操作

为了保证通用性，`ioctl` 接口接收因驱动而异的结构作为输入

* `int ioctl(int file_descriptor, int request, ...);`
* 第一个参数是文件描述符
* 第二个参数是命令识别符
* 剩下参数的类型和数量依赖于对应的驱动和命令

这种特性使得 `ioctl` 接口很容易受威胁：

* 提供给 `ioctl` 接口的数据通常是复杂、非标准的数据结构
  * 对这些数据结构进行解析时，一点点的错误就会影响到内核上下文
* 数据结构的通用性，使分析 `ioctl` 接口变得复杂
  * 分析人员需要知道驱动如何处理不同的命令，接收何种结构的数据

这是 DIFUZE 解决的核心挑战

作者通过自动化地恢复命令标识符和结构体信息

生成需要的复杂数据结构

对设备的 `ioctl()` 接口进行 fuzzing

在最少的人为干预下，找到其中的安全漏洞

### 2.2 Android Operating System

在 2016 年，Android 已经占智能手机 OS 市场的 86.3% 的份额

因此 DIFUZE 使用 Android 来评估分析方法

Android 基于 Linux 内核，是一个整体式结构

虽然内核模块提供了一定程度的模块化

在设计原则上依旧是整体式的

也就是说整个内核运行在一个单一的内存空间中

所有部分的特权级相同

因此，在设备驱动中的任何威胁都会影响到整个内核

Android 开源工程允许设备制造商定制内核驱动来支持新的硬件设备

对于设备制造商来说，市场远比安全重要

因此它们提供的驱动中存在很多的安全威胁

### 2.3 Fuzz Testing

#### Fuzzing

Fuzzing 的核心是产生接近合法的输入，执行目标程序

大范围地测试各个功能

并触发一些引起威胁的边界用例

对于 `ioctl` 这种需要高度依赖输入的函数

* 动态污点追踪的技术有效性较低
* 无法产生高度依赖的输入

DIFUZE 首先采集可能的 `ioctl` 命令值

然后只对不受约束的值进行 fuzzing

如果程序的输入格式一致

通过指定合法输入的格式，fuzzing 能够被加强

然而，这些 fuzzer 无法产生 live data

* 包含指向其它数据的指针

大部分设备驱动接收其中含有指针的结构体

基于语法的技术 - 需要输入的格式固定

#### Kernel and Driver Fuzzing

对 OS 的系统调用接口进行 fuzzing 是一个测试 OS 内核的常用方法

大部分驱动使用 `ioctl` 函数与用户空间进行交互

`ioctl` 很复杂，需要用户给出特定的命令和数据格式

* 因此，识别合法的命令和对应的数据结构是 fuzzing 中的核心问题

DIFUZE 分析了驱动源代码，识别合法的命令和对应的数据结构

并且分析不需要改动驱动的任何代码

Syzkaller 在 fuzzing `ioctl` 时效果很差

* 需要重新编译内核、刷新固件

DIFUZE 是第一个完全自动化的系统

能被通用化为在未修改的内核上对所有 Linux 内核驱动进行 fuzzing

### 2.4 Other Analyses

#### Symbolic Execution

符号执行是一种技术，使用符号变量，产生能够满足复杂检测的约束性输入

因为一些工程上的问题，无法在内核驱动上使用

#### Static Analysis

在不用执行程序的前提下分析程序威胁

由于内核和设备驱动是开源的，可以对其源码进行分析

静态分析的局限在于，会产生较多的 false positives

本文的工作最终使用 fuzzing 进行威胁检测

所有被识别的威胁都是真正的 bug

因此 false positives 被完全避免了

此外，静态分析的另一个缺点是需要人为制定一些规则

---

## 3. Overview

DIFUZE 需要内核源代码作为输入

给定输入，经过一些步骤后，恢复设备驱动的交互接口

产生正确的参数结构体，并对接口进行测试

由于内核 bug 的触发将会导致系统进入不稳定的状态

之前的分析过程会在一台分析机器上完成

只有最终的执行过程在目标机上进行

两者通过网络进行连接

### Interface Recovery

DIFUZE 分析输入的源代码

检测目标机器上哪些驱动被开启

什么设备文件被用于与它们进行交互

驱动能够接收何种 `ioctl` 命令

驱动希望用户传入什么样的结构体

这一系列过程由 LLVM 完成

最终结果是一系列元组，每个元组包含：

* 设备文件名
* `ioctl` 命令
* 结构体类型定义

### Structure Generation

对于每一个结构体

DIFUZE 根据前一步的恢复结果

不断地产生结构体实例

### On-device Execution

`ioctl` 触发组件，接收：

* 设备文件名
* `ioctl` 命令
* 生成的结构体实例

Executor 触发 `ioctl` 的执行

DIFUZE 记录发送到目标机器上的输入

如果触发 bug，输入可被用于重现或人为分析

### Example

一个简单驱动的例子，包含：

* 参数结构体定义
* `copy_from_user` 的封装 - 分析可能遇到的挑战
* 主驱动的初始化代码
  * 内核初始化时调用
    * 用一个 __设备名__ 注册设备 - 设备文件的绝对路径因设备类型而异
    * 指定 `ioctl` 被用户空间调用时，用于处理的 handler
* `ioctl` handler

---

## 4. Interface Recovery

恢复设备驱动的接口：

* 设备文件的名称 / 路径
* 合法的命令值
* 对于不同命令和参数结构体定义

### 4.1 Build System Instrumentation

是 DIFUZE 能够对 Linux 内核驱动进行 LLVM 的分析

#### GCC Compilation

使用 GCC 对内核进行完全编译

记录执行的所有命令

#### GCC-to-LLVM Conversion

使用 GCC-to-LLVM 命令转换工具

将 GCC 需要的命令行标志翻译为 LLVM 需要的格式

使得可以通过 LLVM 来编译内核

在编译后，LLVM 为每个源代码文件生成了一个 bitcode 文件

在 bitcode 文件中开启调试功能，便于提取结构体定义

#### Bitcode Consolidation

DIFUZE 在每个驱动上分别进行操作

因此，需要将单个的 bitcode 文件合并

每个驱动合并为一个文件

这样，接口恢复的过程只需要对一个文件进行操作，简化了分析

### 4.2 ioctl Handler Identification

用户空间调用 `ioctl()` 系统调用，传递：

* 设备文件的文件描述符
* 命令识别符
* 需要的参数结构体

在内核空间中接收到了系统调用后，对应的 ioctl handler 会被调用

为了恢复合法的 __命令标识符__ 和 __参数结构体定义__

DIFUZE 必须首先识别 `ioctl` handler 的顶层函数

每个驱动能够为它的每个设备文件注册一个顶层的 `ioctl` handler

注册有好几种方法，然而，所有的方法都包含：

* 建立一系列结构体
* 这些结构体的其中一个 field 是 `ioctl` handler 的函数指针

这些结构体的范围是确定的，对应的 field 也是确定的

使用 LLVM 的分析工具，找到所有这些结构体被使用的地方

恢复赋值 `ioctl` handler 函数指针的位置

```c
static struct cdev driver_devc;
static dev_t client_devt;
static struct file_operations driver_ops;
__init int driver_init(void)
{
    // request minor number
    alloc_chrdev_region(&driver_devt, 0, 1, "example_device");
    // set the ioctl handler for this device
    driver_ops.unlocked_ioctl = ioctl_handler;
    cdev_init(&driver_devc, &driver_ops);
    // register the corresponding device.
    cdev_add(&driver_devc, MKDEV(MAJOR(driver_devt), 0), 1);
}
```

比如，从对于结构体 `driver_ops` 的 `unlocked_ioctl` 的赋值中

可以识别出 `ioctl_handler` 就是对应的 `ioctl` handler

### 4.3 Device File Detection

为了确定每个 `ioctl` handler 的设备文件名

需要识别在注册 `ioctl` handler 时提供的设备文件名

比如，在上述例子中，设备文件名为 `/dev/example_device`

根据设备类型的不同，在 Linux 中有几种方法注册设备文件

* character device
* proc device

此外，根据设备类型，设备文件的目录位置也有不同

DIFUZE 的解决方法：

1. 提取所有正在将一个地址存入上述 operation 结构体某一个 field 中的所有 LLVM store 指令
2. 查找在注册函数中所有对 operation 结构体的引用
3. 分析参数值中的设备文件名，返回其是否是一个常量

```c
static struct cdev driver_devc;
static dev_t client_devt;
static struct file_operations driver_ops;
__init int driver_init(void)
{
    // request minor number
    alloc_chrdev_region(&driver_devt, 0, 1, "example_device");
    // set the ioctl handler for this device
    driver_ops.unlocked_ioctl = ioctl_handler;
    cdev_init(&driver_devc, &driver_ops);
    // register the corresponding device.
    cdev_add(&driver_devc, MKDEV(MAJOR(driver_devt), 0), 1);
}
```

比如这个例子，根据 `ioctl` handler 被注册的结构体 `driver_devc`

找到了 `driver_devt`，从而在 `alloc_chrdev_region()` 中找到了 `"example_device"` 是一个字符串常量

并由 `cdev_add()` 函数隐含了它是一个字符设备

因此恢复了设备文件名：`/dev/example_device`

驱动可能使用了动态创建的文件名，比如：

```c
VOS_INT __init RNIC_InitNetCard(VOS_VOID) {
    ...
    snprintf(pstDev->name, sizeof(pstDev->name),
    "%s%s",
    RNIC_DEV_NAME_PREFIX,
    g_astRnicManageTbl[ucIndex].pucRnicNetCardName);
    ...
}
```

* 由于静态分析的局限性，无法自动恢复这类文件名
* 可以通过人为恢复，或直接忽略它们

### 4.4 Command Value Determination

给定 `ioctl` handler，进行了静态的过程间、路径敏感的分析

采集所有 `cmd` 数值的 __相等约束__

* 几乎所有的驱动都是用相等约束来比较 cmd 值

```c
DriverStructTwo dev_data1[16];
DriverStructTwo dev_data2[16];
static bool enable_short; static bool subhandler_enabled;

long ioctl_handler(struct file *file, int cmd, long arg) {
    uint32_t curr_idx;
    uint8_t short_idx; void *argp = (void*) arg;
    DriverStructTwo *target_dev = NULL;
    switch (cmd) {
        case 0x1003:
            target_dev = dev_data2;
        case 0x1002:
            if(!target_dev)
                target_dev = dev_data1; // program continues to execute
            if(!enable_short) {
                if (copy_from_user_wrapper((void*)&curr_idx, argp, sizeof(curr_idx))) {
                    return -ENOMEM; // failed to copy from user
                }
            } else {
                if (copy_from_user_wrapper((void*)&short_idx, argp, sizeof(short_idx))) {
                    return -ENOMEM; // failed to copy from user
                }
            curr_idx = short_idx;
            }
            if(curr_idx < 16)
                return process_data(&(target_dev[curr_idx]));
            return -EINVAL;
        default:
            if(subhandler_enabled)
                return ioctl_subhandler(file, cmd, argp);
    }
    return -EINVAL;
}

long ioctl_subhandler(struct file *file, int cmd, void *argp) {
    DriverStructOne drv_data = {0};
    DriverStructTwo *target_dev;
    if(cmd == 0x1001) {
        if(copy_from_user_wrapper((void*)&drv_data, argp, sizeof(drv_data))) {
            return -ENOMEM; // failed to copy from user
        }
        target_dev = dev_data1;
        if(drv_data.subtype & 1)
            target_dev = dev_data2;
        // Arbitrary heap write if drv_data.idx > 16
        if(copy_from_user_wrapper((void*)&(target_dev[drv_data.idx]), drv_data.subdata, sizeof(DriverStructTwo))) {
            return -ENOMEM; // failed to copy from user
        }
        return 0;
    }
    return -EINVAL;
}
```

### 4.5 Argument Type Identification

`ioctl` 命令标识和对应的数据结构定义具有 __多对多__ 的关系

为了找到这些结构体，必须识别所有通向 `copy_from_user()` 函数的路径

* 用于从用户空间拷贝数据

比如上述例子中的 `copy_from_user_wrapper()`

```c
int copy_from_user_wrapper(void *buff, void *userp, size_t size) {
    // copy size bytes from address provided by the user (userp)
    return copy_from_user(buff, userp, size);
}
```

* 忽略：第二个参数不是 `ioctl` 函数传入参数的调用点

找到用户空间指针的类型

* 通过类型可以找到对应的结构体定义

但是，指针的类型强制转换可能会导致数据类型的隐藏 - `void *`

此外，`copy_from_user()` 可能被封装在一个函数中，间接被调用

为了解决这个问题，进行了过程间的路径敏感的 type propagation

* 找出可能传递到 `copy_from_user()` 函数的所有类型

为了将每个命令标识符和结构体类型联系起来

在路径中还采集了相等约束

如果路径上的相等约束能够到达 `copy_from_user()`

那么对应的命令标识符就与结构体产生了联系

```c
DriverStructTwo dev_data1[16];
DriverStructTwo dev_data2[16];
static bool enable_short; static bool subhandler_enabled;

long ioctl_handler(struct file *file, int cmd, long arg) {
    uint32_t curr_idx;
    uint8_t short_idx; void *argp = (void*) arg;
    DriverStructTwo *target_dev = NULL;
    switch (cmd) {
        case 0x1003:
            target_dev = dev_data2;
        case 0x1002:
            if(!target_dev)
                target_dev = dev_data1; // program continues to execute
            if(!enable_short) {
                if (copy_from_user_wrapper((void*)&curr_idx, argp, sizeof(curr_idx))) {
                    return -ENOMEM; // failed to copy from user
                }
            } else {
                if (copy_from_user_wrapper((void*)&short_idx, argp, sizeof(short_idx))) {
                    return -ENOMEM; // failed to copy from user
                }
            curr_idx = short_idx;
            }
            if(curr_idx < 16)
                return process_data(&(target_dev[curr_idx]));
            return -EINVAL;
        default:
            if(subhandler_enabled)
                return ioctl_subhandler(file, cmd, argp);
    }
    return -EINVAL;
}

long ioctl_subhandler(struct file *file, int cmd, void *argp) {
    DriverStructOne drv_data = {0};
    DriverStructTwo *target_dev;
    if(cmd == 0x1001) {
        if(copy_from_user_wrapper((void*)&drv_data, argp, sizeof(drv_data))) {
            return -ENOMEM; // failed to copy from user
        }
        target_dev = dev_data1;
        if(drv_data.subtype & 1)
            target_dev = dev_data2;
        // Arbitrary heap write if drv_data.idx > 16
        if(copy_from_user_wrapper((void*)&(target_dev[drv_data.idx]), drv_data.subdata, sizeof(DriverStructTwo))) {
            return -ENOMEM; // failed to copy from user
        }
        return 0;
    }
    return -EINVAL;
}
```

在这个例子中可以看到：

| ID   | Path                | cmd constraints | User argument type |
| ---- | ------------------- | --------------- | ------------------ |
| 1    | Line 10 → L11 → L16 | cmd == 0x1003   | uint32_t           |
| 2    | Line 10 → L11 → L21 | cmd == 0x1003   | uint8_t            |
| 3    | Line 12 → L16       | cmd == 0x1002   | uint32_t           |
| 4    | Line 12 → L21       | cmd == 0x1002   | uint8_t            |
| 5    | Line 30 → L32 → L41 | cmd == 0x1001   | DriverStructOne    |
| 6    | Line 30 → L32 → L49 | cmd == 0x1001   | N/A                |

其中，最后一条路径中，`copy_from_user()` 的第二个参数不是用户参数，因此被忽略

可以看到，对于同一个命令标识，可能存在多种用户参数类型

### 4.6 Finding the Structure Definition

寻找类型定义，包括寻找所有其包含类型的结构定义

比如，一个结构体中可能包含有子结构体

使用 4.1 中的调试信息，找到对应 `copy_from_user()` 函数的源代码文件名

知道源文件后，使用 GCC-to-LLVM 产生对应的预处理文件

预处理文件中应当包含所需所有类型的定义

运行 `c2xml` 将 C 结构体定义转换为 XML 格式

---

## 5. Structure Generation

DIFUZE 恢复接口之后，会开始生成结构体实例了：

* 实例化结构体
* 用随机数据填充数据域
* 合理地设置指针，构建复杂的输入

### Type-Specific Value Creation

特定的值更可能覆盖更多的代码

> 比如，buffer length 对齐位边界？

有些指针引用的数据：

* 非结构化 - `char *`
* 结构定义无法被恢复 - `void *`

对于这类数据，DIFUZE 会分配一页随机内容

### Sub-Structure Generation

`ioctl` 的输入通常带有嵌套结构体

DIFUZE 独立地产生结构体，并发送到执行设备

执行组件在将参数传入 `ioctl` 之前

会将嵌套的结构体合并

---

## 6. On-device Execution

之前的所有步骤运行于 analysis host 上

真实的执行必须运行在 target host 上

Analysis host 将产生的结构体、目标设备文件、命令标识符一起发送

Target host 上的 execution component 触发最后的 `ioctl`

### 6.1 Pointer Fixup

一些结构体包含由多个指针联系的多片内存区

为了节省空间

每个结构体的内存区域被独立传输

以及它们如何被组合的 metadata

* 减轻了 analysis host 和 target host 之间的带宽
* 一个结构体可以被多个父结构体使用

有一些结构体是递归的，比如链表结点

为了限制结构体组合时的边界

DIFUZE 限制了这种结构体的递归阈值

### 6.2 Execution

在 analysis host 和 target host 之间维护一个心跳信号

当 DIFUZE 发现 bug 时，记录发送到 target host 的输入

#### System Restart

当 bug 被触发时，target host 要么在一个不一致的状态，要么 crash

对于前者，触发一次重启

对于后者，根据 crash 发生的方式，选择性地自动重启

因此 DIFUZE 不需要人为干预

---

## 7. Implementation

系统完全自动化：

* 提供可编译的源文件
* 将 analysis host 与 target host 连接
* 启动 target host 上的执行组件
* 一条启动命令

所有的 interface extraction 被实现为 LLVM passes

### 7.1 Interface-Aware Fuzzing

Section 5 & 6 中的实现被称为 MangoFuzz

* 结构体生成
* 在设备上执行

这是一个用于测试 interface-aware 有效性的原型

没有任何其它优化

本文作者也将 DIFUZE 集成进了 syzkaller 中

Syzkaller 能够处理结构体作为系统参数

* 只要对应的格式被人为指定

为了集成 DIFUZE

* 将 Interface Recovery 的结果自动转换为 syzkaller 能够接受的格式

---

## 8. Evaluation

评估了 Interface Recovery 能力和 bug-finding 能力

### 8.1 Interface Extraction Evaluation

在 7 台设备的内核中，总共识别了 789 个 `ioctl` handlers

* handler 的数量与设备数量呈正相关

#### Device Name Identification

DIFUZE 自动识别了 469 个设备名

* 涵盖了 59.44% 的 `ioctl` handlers

大部分的识别失败来源于内核主线驱动

* 主线驱动通常使用动态产生的设备文件名 - 手动提取
* 第三方驱动通常使用静态的设备文件名

#### Valid Command Identifiers

总共提取出了 3565 个合法的命令标识符

Crash 的数量与命令标识符的数量呈正相关

11% 的 handler 不需要任何命令

* 被条件编译的 handler，由内核 configuration 守护
* 产生空的 bitcode 文件

50% 的 handler 接收一个单独的命令标识符

* 大部分是 `v4l2_ioctl_ops`
* 有嵌套的 handler

98.3% 的 handler 有少于 20 个的合法命令标识符

#### User Argument Types

1688 (47%) 个命令标识符中

没有任何 `copy_from_user()`

1. 用户参数被视为 C 的 `long` 类型，被当做 raw 值处理
2. 出现了 `copy_to_user()`

本文不关心这两种类型标识

在剩下 1877 (53%) 个命令标识符中

预期用户参数是一个结构体指针，由 `copy_from_user()` 来拷贝数据

这些指针被分为三种类型：

1. 526 (15%) 的命令标识符接收标量指针
2. 961 (30%) 的命令标识符接收没有嵌套指针的 C 结构体指针
3. 390 (11%) 的命令标识符接收带有嵌套指针的 C 结构体指针
   * 这类命令在没有参数类型信息的条件下很难被 fuzz

#### Random Sampling Verification

从 7 台 Android 设备上各自随机挑选 5 个 `ioctl`

人工检验提取出的结构体类型的正确性

在 35 个 `ioctl` 的 327 条命令中，294 个是正确的，达到了 90% 的准确率

### 8.2 Evaluation Setup

采取了如下几种配置：

* Syzkaller base
  * 只对 `open()` 和 `ioctl` 进行 fuzzing 的普通版 syzkaller
  * 提供几个标准的设备文件名和几个标准的参数结构
* Syzkaller + path
  * 加入了 DIFUZE 提取出的设备文件路径
* DIFUZEi
  * DIFUZE 提取出接口信息设备 + 产生的 ioctl 通过 syzkaller 进行 fuzzing
* DIFUZEs
  * 集成了全部的 interface recovery 功能，包括自动识别参数结构体格式
  * 使用 syzkaller 进行 fuzzing
  * 应当具有最好的表现，因为能够使用 syzkaller 的优化
* DIFUZEm
  * 集成了全部的 interface recovery 功能
  * 使用了简单的 fuzzer 原型

对于每个 target device，执行组件运行在 root 权限上，确保能够 fuzz 所有的驱动

如果 crash 在某一驱动上频繁出现，那么这个 `ioctl` handler 将被列入黑名单

防止手机被频繁重启

### 8.3 Results

基本配置的 syzkaller 无法找到任何 bug

给定正确设备文件名的条件下，只能触发共三次 crashes

* 盲目 fuzzing 驱动的效率很低
* 此外，类似的测试在出厂前已经被设备制造商做过，因此找不出什么 bug

加入部分的接口信息时，DIFUZEi 能够找到 22 个 bug

加入完整的接口信息时，能够找到 34 个 bug，上升 54.5%

* 体现了 interface-aware fuzzing 的有效性
* 体现了同时恢复 __命令标识符__ 和 __参数结构体__ 的重要性

DIFUZEm 只比 DIFUZEs 少找到 4 个 bug

* 体现了准确的接口信息的重要性

### 8.4 Case Study I: Design Issue in Honor 8

在 fuzzing 几轮后，设备的序列号被改变

* 序列号应当是只读的，只有 boot loader 能够改变
* 证明了通过驱动，序列号可以从用户空间被改变

Honor 8 在 flash 上有一个分区 `nvme`

* 保存了设备配置信息
* 部分配置选项是非特权的，能够被 Android 修改
* 而部分配置，如序列号和主板编号，只能被 boot loader 修改

然而，`/dev/nve` 的 `ioctl` handler 提供了读写这些选项的渠道

并且不会区分配置的特权级

因此，用户空间的程序可以读写特权级配置

### 8.5 Case Study II: qseecom bug

```c
static int qseecom_mdtp_cipher_dip(void __user *argp)
{
    struct qseecom_mdtp_cipher_dip_req req;
    u32 tzbuflenin, tzbuflenout;
    char *tzbufin = NULL, *tzbufout = NULL;
    int ret;

    do {
        ret = copy_from_user(&req, argp, sizeof(req));
        if (ret) {
            pr_err("copy_from_user failed, ret= %d\n", ret);
            break;
        }
        ...
       /* Copy the input buffer from userspace to kernel space */
       tzbuflenin = PAGE_ALIGN(req.in_buf_size);
       tzbufin = kzalloc(tzbuflenin, GFP_KERNEL);
       if (!tzbufin) {
           pr_err("error allocating in buffer\n");
           ret = -ENOMEM;
           break;
       }

       ret = copy_from_user(tzbufin, req.in_buf, req.in_buf_size);
       ...
    } while (0);
    ...
    return ret;
}

long qseecom_ioctl(struct file *file, unsigned cmd, unsigned long arg)
{
    int ret = 0;
    void __user *argp = (void __user *) arg;
    switch (cmd) {
        ...
        case QSEECOM_IOCTL_MDTP_CIPHER_DIP_REQ: {
           ...
           ret = qseecom_mdtp_cipher_dip(argp);
           break;
        }
        ...
    }
    return ret;
}
```

* 给定 `QSEECOM_IOCTL_MDTP_CIPHER_DIP_REQ`，进入 `qseecom_mdtp_cipher_dip()` 函数
* 第一次 `copy_from_user()` 将用户空间结构体拷贝到变量 `req` 中
* `PAGE_ALIGN` 函数根据用户空间结构体中指定的大小，为嵌套指针的深拷贝分配空间
* 如果用户空间指定一个较大的值，`PAGE_ALIGN` 将会 overflow
* 导致分配了一个比嵌套指针指向的结构体的实际大小要小的 buffer
* 第二次调用 `copy_from_user()` 时，内核缓冲区溢出，crash

这个 crash 如果想被可见，用户空间提供的 `req.in_buf` 必须是一个合法的指针

* 否则 `copy_from_user()` 将会异常退出

因此，如果没有合理的结构体实例

`ioctl()` 中的 bug 将永远不会被触发

### 8.6 Augmenting with Coverage-guided Fuzzing

Interface-aware 在 coverage-guided 使用的前提下是否还需要被使用？

答案是肯定的

使用 syzkaller fuzz 一个参数简单的接口和一个参数复杂的接口

使用 / 不使用结构化的接口信息

当接口信息被提供时，覆盖的 basic block 的百分比显著上升

因此接口信息对于 coverage-guided fuzzing 来说很重要

---

## 9. Discussion

### 9.1 Weaknesses

带有 bug 的驱动将会很早发生 crash

阻止 fuzzer 探索更深的功能

此外，DIFUZE 无法提取结构体 fields 之间的复杂关系

### 9.2 Future Work

没有使用到运行时的 coverage 来指引 fuzzer

为了使用运行时的 coverage 信息，需要重新编译并烧写内核

设备制造商会锁住 bootloader，阻止重新烧写新的内核

---

## 10. Conclusion

提出了 interface-aware fuzzing 来提升自动化分析接口敏感代码的有效性

比如 Linux 内核驱动

提出了一系列技术恢复 `ioctl` 接口定义

并实现为自动化工具，只需要内核源代码和一条命令就能开始运行

本文的技术能够在功能和性能上有效恢复设备文件名、合法命令标识符、对应的接口参数类型

DIFUZE 被用于 7 款移动设备

共发现 36 个 bug，32 是之前未知的

---

## Summary

Syzkaller 的原理之前已经看过

因此动态分析这一块儿的原理大致了解

燃鹅，感觉局部的静态分析也是很有用的

可以恢复一些函数间的调用关系、数据类型约束等

这些信息也正好是 syzkaller 等动态分析工具输入的一部分

所以两者结合能够取得更好的效果

感觉有空得玩一玩 LLVM 了

---

