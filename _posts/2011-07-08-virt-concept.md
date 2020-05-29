---
layout: post
title: 虚拟化基本概念
author: kaifeng
date: 2011-07-08
categories: virt
---

虚拟化的概念事实上早已存在，比如现代操作系统对CPU的分时共享。虚拟化技术只不过是在更大一个层面上的抽象，是对整个系统的抽象。
虚拟化可以分为三个部分：CPU虚拟化、内存虚拟化、I/O虚拟化。

CPU虚拟化也就是分时。在一个架构上实现虚拟化，必须满足的一个条件是：敏感指令必须也是特权指令。在x86架构上，这一点并没有完全符合。
I/O虚拟化相对于CPU虚拟化来说，要复杂一些。有的设备比较容易，比如内存、块设备，可以进行划分隔离。但其它设备就复杂了，比如图形卡、DMA，还有很多设备并不是设计为可以虚拟化的。

为什么虚拟化？有下面一些原因：

* 有空闲的计算能力
* 冗余备份
* 克隆非常方便
* 迁移方便
* 节能
* 比物理资源更容易移植
* 隔离

x86不满足前面说的架构虚拟化条件，因为它的一些指令不符合要求。但x86使用太广泛了，所以也有一些解决方案：

* 二进制重写 (Binary Rewrite)：这个方案流行自VMWare。它通过扫描指令流，确定哪些是特权指令，然后将其修改为模拟的版本，实现方式类似于调试器的工作方式。这种方式毫无疑问会有性能上的损失。
* 半虚拟化 (Paravirtualization)：半虚拟化需要修改操作系统，最高特权级运行的是hypervisor，操作系统的特权级被降低到hypervisor以下。假如x86有4个特权级，hypervisor运行在ring 0，操作系统被降到ring 1。如果有的架构只提供2级特权级，则操作系统会被降低到与用户态相同的非特权级。
* 硬件辅助虚拟化 (Hardware-Assisted Virtualization)：又称HVM (Hardware Virtual Machine)，HVM上可以运行不经修改的操作系统。HVM技术有Intel的VT，AMD的AMD-V。

虚拟化的学习可以从这些资料开始：
* The Definite Guide To The Xen Hypervisor
* Professional Xen Virtualization
* Xen初学者指南：http://www.linuxsir.org/main/node/188
