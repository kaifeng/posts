---
layout: post
title: Intel 82599 SR-IOV
author: kaifeng
date: 2011-11-07
categories: virt
---

Intel的网口驱动现有四套：
* igb: 82576驱动
* igbvf: 82576前端驱动（Virtual Function）
* ixgbe: 82599驱动
* ixgbevf: 82599前端驱动（Virtual Function）

HVM域使用82576驱动应该是直接用igb驱动，只要确保这个驱动被编译即可。

原生共享设备一般向每个暴露的接口提供独立的内存空间，工作队列，中断及命令处理，在host接口内使用共享资源。共享资源仍需要管理，一般暴露一组管理寄存器给hypervisor内的可信任的部分。

![Natively Shared](/posts/assets/82599sriov/1-natively-shared.png)

独立的数据通道，使设备可以同时从多个源接收命令，并在传给二级部件前进行合并，虚拟化软件不再需要做将I/O请求复用成串行流的工作。

原生共享设备有多种实现方式，有标准的，也有私有的，因大部分设备通过PCI访问，PCI-SIG定义了SR-IOV规范提供一个标准方法创建管理原生共享设备。

SR-IOV架构设计为允许一个设备支持多个虚功能，而且对新增的功能，尽可能减小硬件开销。SR-IOV引入了两种新的功能类型：

* 物理功能（PF）：完整的PCIe功能，包括SR-IOV扩展能力（用于配置管理SR-IOV功能）。
* 虚拟功能（VF）：轻量级的PCIe功能，包含了数据移动的必要资源（但减少了配置资源）。

虚拟化中，对设备的直接访问可以实现快速的I/O，但它阻碍了I/O设备的共享，SR-IOV提供一种机制，一个单根功能可以表现为多个独立的物理设备。支持SR-IOV的设备在配置后（一般是由hypervisor），在PCI配置空间表现为多个功能，每个功能有自己的配置空间及BAR寄存器。

![Mapping Virtual Function Configuration](/posts/assets/82599sriov/2-map-vf-conf.png)

SR-IOV设备可配置的独立功能的数量，hypervisor将一个或多个虚功能赋给一个虚拟机。像VT-d提供的硬件辅助技术（如地址翻译技术），允许直接的DMA。

![Intel SR-IOV Overview](/posts/assets/82599sriov/3-overview.png)


# 上层架构

## Hypervisor

要使用SR-IOV，Hypervisor（Xen或KVM）需要做一些工作。

VM对PCI配置空间的访问

Hypervisor需要将I/O设备的PCI配置空间显现给虚拟机。在共享I/O模型下，Hypervisor通常提供一个非常简单的以太网控制器给虚拟机，而在SR-IOV的环境下，Hypervisor将实际的配置空间赋给特定的VF，允许VM内的VF驱动访问VF资源。

PCI SIG SR-IOV PCI配置空间

SR-IOV规范定义了设备如何公布SR-IOV能力，VF的PCI配置信息如何保存及访问，Hypervisor必须知道如何读取并解析这些信息。

VT-d初始化及配置

VT-d通过映射DMA操作，允许将I/O设备直接赋给VM，Hypervisor必须配置VT-d对I/O地址执行映射。

## PF驱动

PF驱动管理SR-IOV设备的全局功能及配置共享资源，PF驱动与Hypervisor相关，运行在更具备特权的环境下。PF驱动包含所有传统驱动的功能，向Hypervisor提供I/O资源访问，并且还可以执行影响整个设备的操作。驱动必须运行在可持续的环境中，在所有VF驱动加载前加载，在所有VF驱动卸载后卸载。

Intel SR-IOV PF驱动包含Intel以太网控制器驱动的所有标准功能，还包含SR-IOV相关的实现，如：

* 给VF生成MAC地址
* 通过Mailbox系统与VF驱动通讯
* 由VF驱动配置VLAN Filter
* 由VF驱动配置组播地址
* 由VF配置最大包长大小
* VF驱动请求对资源的复位

## VF驱动

在虚拟化环境下，驱动通常与中间软件层通信，中间软件层对物理设备进行抽象与模拟，多数情况下，驱动不会感知到这个存在。

在本地共享设备被直接分配时，设备行为发生了变化。VF接口不包括PCIe控制的所有方面，一般也不直接控制共享组件和资源，如设置以太网链路速率。这种情况下，改造VF驱动或者将其半虚拟化，使其感知虚拟化环境会更好。

半虚拟化驱动直接与硬件通信，完成数据移动操作，驱动已知设备是一个共享资源，依据PF驱动的服务来处理可能造成全局性影响的操作，如二级部件的初始化和控制。

Intel VF驱动与其它驱动的加载方式相同，基于设备ID。VF的PCI配置信息具有设备ID来标识它是一个虚功能，并实现适当驱动的加载。

## PF-VF驱动间通信

设备共享需要VF能与PF驱动通信，以请求全局影响的操作。这个通信通道需要消息传递与中断生成的能力。SR-IOV规范并没有定义这个通信通道的机制，Intel的实现方式是为每个以太网控制器内的每个VF提供一套硬件邮箱及门铃。Intel提供硬件邮箱系统的考虑是保证通信机制与Hypervisor无关，且持续可用。但这种通信通道也可以基于Hypervisor提供的软件机制。

# Intel 82599 10G以太网控制器SR-IOV支持

每个VF配有一定的资源，包括池子，还包括VF驱动使用的一组寄存器，控制收发，邮箱消息等。

这里简介82599如何为特定VF排序报文，如何为每个VF分配资源，PF接收哪些资源。

## 队列

82599有128个发送和接收队列，一个发送和一个接收队称作一对队列。

![Queue Pair](/posts/assets/82599sriov/4-queue-pair.png)


## 池

池是指分配给同一个VF用于收发的队列对组合。82599的池子数量可配，最多支持64个池（即每个池含2个队列对），在SR-IOV模式下，可以配为16，32或64个池。

数据根据L2过滤器（MAC地址过滤器和VLAN Tag过滤器）放置于池子，每个池子可以配置多个MAC地址，并共享64个VLAN Tag。

## 二层分类器/排序器

该单元可看作是一个简单的二层交换机，负责根据目的MAC地址或VLAN Tag对报文进行排序。报文进入控制器时，二层分类器对其特征进行匹配，根据匹配结果发给相应的池子。

## 池到池（VF到VF）桥接

在传统的虚拟化环境中，Hypervisor以软件方式实现虚拟交换，Hypervisor检查来自一个VM的报文的目的MAC地址是否为系统中的另一个VM，如果匹配则Hypervisor不必将报文发给实际硬件，而是通过软件交换的方式发给目标VM。

82599的"switch"部分可以在硬件层面实现VF到VF的桥接，无需Hypervisor的参与。

## Anti-spoofing

82599支持两种反欺骗机制：

  * MAC地址。如果发送报文的源MAC地址与VM配置的接收MAC地址不匹配，丢弃。

  * VLAN Tag。检查VF是否属于发送报文的VLAN组。

## 收包数据流举例

![Packet Sent to a VM](/posts/assets/82599sriov/9-pkt2vm.png)


步骤1~2：报文到达，进入L2分类器/交换

步骤3：报文按照目的MAC地址分类，图例中与池/VF 1匹配。

步骤4：NIC发起DMA操作，请求将报文移到VM。

步骤5：DMA请求到达芯片组，VT-d执行地址翻译，然后将报文DMA至VM的VF驱动Buffer中。

步骤6：NIC产生MSI-X中断，表示Rx事务已经完成，这个中断由Hypervisor接收。

步骤7：Hypervisor向VM产生虚中断，表示Rx事务已完成，VM的VF驱动开始处理报文。

# 邮箱通信系统

有些配置无法通过VF的PCI资源完成（比如VF驱动定义VLAN过滤），此时VF驱动就需要与PF驱动通信，让PF代为完成。

![Mailbox Communication](/posts/assets/82599sriov/10-mailbox.png)


## VF邮箱

当VM准许访问VF时，可用的VF资源里有一块就是邮箱。邮箱比较简单，包含一个缓冲区，消息从缓冲区读出或向其写入，还有一个寄存器VFMailbox提供PF与VF间的同步/信号量功能。

VF和PF都可以访问缓冲区，因此消息可以来回传递。邮箱是VF资源的一部分，因此VF驱动只能访问允许当前VM访问的VF的邮箱。

示范代码为ixgbevf驱动代码，实现在ixgbe_mbx.h及ixgbe_mbx.c两个文件里，实现了ixgbevf驱动用于邮箱通信的API接口。

## PF邮箱

PF驱动可以访问所有VF邮箱，通过VFMBMEM（VF Mailbox Memory）和PFMailbox寄存器组。

![Mailbox from PF Perspective](/posts/assets/82599sriov/12-pf-mailbox.png)

当VF驱动向VFMBMEM缓冲区写入消息，并置VFMailbox相应位时，会产生一个中断给PF，PF驱动取出消息，并确认处理。此时，PF驱动使用Mailbox给VF发送一个异步消息。


---

> Intel 82599 SR-IOV Driver Companion Guide
