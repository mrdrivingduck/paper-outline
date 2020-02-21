# Outline

## Docker: Lightweight Linux Containers for Consistent Development and Deployment

Created by : Mr Dk.

2020 / 02 / 21 22:34

Ningbo, Zhejiang, China

---

## Abstract

将应用和其所有的依赖打包在一起，能够平稳过渡到一个全不相干的开发、测试、生产环境中 - 这就是 Docker 项目的目标。

Docker 试图解决所谓的 _dependency hell_ 问题 - 现代应用总是依赖于已有组件提供的服务：比如一个 Python 程序可能使用 PostgreSQL 存储数据，使用 Redis 作缓存，使用 Apache 作 Web 服务器。每个组件都有其自身的依赖，又可能与其它组件的依赖产生矛盾。Docker 解决了如下的问题：

* 依赖冲突 - 比如需要使用同一个组件的不同版本
* 缺失依赖 - 所有的依赖都与应用打包在一起了
* 不同平台 - 如果两个系统都运行有 Docker，那么把应用从一个系统迁移到另一个系统将不再是个问题

## Docker: A Little Background

Docker 由 Go 实现，开源。

## Under the Hood

Docker 使用了一些内核级的技术。Docker 利用了内核中的 _LXC_、_cgroups_ 和一个 _copy-on-write_ 的文件系统，组成了一个比这三项技术都要厉害的工具。

_LXC (Linux Container)_ 是一个 Linux 容器的用户空间控制包，是 Docker 的核心。LXC 使用内核级的 __namespace__ 来实现 container 和 host 的隔离。命名空间将 container 和 host 的用户数据分开，使 container 的 root 不会上升到 host 的 root 权限。进程命名空间用于显示和管理只在 contain 中运行的进程，而不是 host 的进程。另外，网络命名空间使 container 有自己的网络设备和虚拟 IP 地址。

_cgroups (Control Groups)_ 用于实现对资源的审计和限制。Docker 能够对 container 使用的资源进行限制，也可以对 container 中各进程的资源消耗量进行监视。

此外，Docker 使用 _AuFS_ (Advanced Multi-Layered Unification Filesystem) 作为 container 的文件系统。AuFS 是一个分层的文件系统，能覆盖 (或者说隐藏) 已经存在的文件系统，使其看起来就好像只有一层。当进程需要修改一个文件时，AuFS 创建一份该文件的副本 - 这个过程被称为 _copy-on-write_ 。

AuFS 使 Docker 能够使用 __images__ (镜像) 作为容器的基础。如果多个不同容器都需要用到 _CentOS_ 镜像，AuFS 保证了 _CentOS_ 镜像只会有一份，从而节省了磁盘和内存空间；同时也加快了部署的速度。

Docker 使用 AuFS 带来的另一个好处是，对 images 可以进行版本控制。每一个新版本就是与前一个版本的 diff change - 从而使镜像文件尽可能小。可以和 Git 作类比。

由于 Docker 基于 Linux 的技术运行，因此可以运行在所有的 Linux 发行版上。对于非 Linux 系统，Docker 将运行在这些系统的虚拟机中。

## Containers vs. Other Types of Virtualization

Container 和基于虚拟机的虚拟化有什么区别？

* 容器的虚拟化位于操作系统层
* 虚拟机的虚拟化位于硬件层

### Virtualization

Hypervisor 将硬件进行分块。Hypervisor 主要分为两类：

1. 直接运行在裸的硬件之上
2. 运行在一个 guest OS 的额外一层软件上 (比如 VMware Server)

而 container 使用的是操作系统层面的保护机制 - 也就是对 OS 进行虚拟化。Containers 运行在同样的 OS 中，并且互相不知道对方的存在 - 因为彼此的进程和网络通过 Linux 的命名空间进行隔离。

### Operating Systems and Resources

基于 hypervisor 的虚拟化可以直接访问硬件 - 因此用户需要安装操作系统。在硬件上会同时有很多个 OS 运行，从而快速占用了服务器的各项资源。

Container 将一个正在运行的 OS 作为其寄生环境，在寄生 OS 互相隔离的空间中各自运行。好处在于：

1. 资源利用率更加有效
2. 容器的成本低，能够快速创建和销毁，不需要重启或关闭整个 OS - 只是相当于创建和销毁进程

### Isolation for Performance and Security

Docker container 中运行的进程与其它 container 中的进程和 host OS 中的进程是相互隔离的。虽然如此，所有的进程都在同一个内核中执行。

Docker 利用 的 _LXC_ 技术已经在 Linux 内核中存在了 5 年以上 ()，而 _Control Groups_ 更是在 Linux 内核中存在了更长的时间。虽然这种隔离已经非常强大，但对于基于 hypervisor 的 VM 来说，隔离性还是要差一些。如果 contain 所在的 kernel 崩溃了，那么所有的容器也就崩溃了。另外，VM 在技术上比 Docker 要更为成熟，并为高可用场景做了加固。

### Docker and VMs - Frenemies

Docker 和 VM 技术可以相辅相成。

## Docker Repositories

Docker 可以快速寻找、下载其它开发者创建的 images，并启动 container。存储镜像的地方称为仓库，可以理解为 Node 的 NPM。

Docker 官方提供有公开的仓库，另外也可以搭建自己的私人仓库。

## Hands-On with Docker

Docker 可以以三种不同的方式运行：

1. 以守护进程的方式运行，用于管理 containers；守护进程暴露 RESTful API，可被本地或远程访问
2. 以客户端命令行的形式访问守护进程的 RESTful 接口
3. 以客户端访问远程仓库中的 images，将其 pull 到本地，或将本地镜像 push 到远程

## Docker Workflow

一个 container 中应当只运行独立的应用或服务，而不是一堆应用或服务的集合。在容器中只运行一个主要的进程，使得整个系统的组件能够松耦合。

## Creating a New Docker Image

Dockerfiles 是用于形容 image 构建过程的文本文件。类似一个 shell 脚本吧。文件中的命令可以一步一步地描述 image 是如何被构建出来的。

---

