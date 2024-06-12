---
layout: post
title: XLR内嵌BOOT分析
author: kaifeng
date: 2009-10-23
categories: bsp
description: 为了支持多种内存类型，配置复杂的DDR/DDR2内存参数，再配备强劲的DQS参数调整，XLR使用了专门的内存配置模块，称之为boot1_5。
---

> 幸福的BOOT都是相似的，但不幸的BOOT各不相同。——列夫*托尔斯基

XLR的BOOT就是不幸之一。为了支持多种内存类型，配置复杂的DDR/DDR2内存参数，再配备强劲的DQS参数调整，XLR使用了专门的内存配置模块，称之为boot1_5。

# BOOT的布局

XLR BOOT的布局如图所示，在bootrom中，有一个内嵌BOOT，也即提到的boot1_5，这是一个单独的模块， 以专门的boot1_5段嵌入在bootrom之中，我们先看看它是怎么运行的，然后再看看这个它是怎么生成的。

![XLR内嵌BOOT](/posts/assets/xlr-embedded-boot.png)

mips从0xbfc00000开始执行第一条指令，在进行L1指令、数据cache初始化后，每一个boot都迫切地希望自己能够开始使用内存，XLR首先将内嵌boot的所有.data, .bss, .stack锁入cache，然后跳转到内嵌boot的首地址开始执行， 内嵌boot的首地址位于kseg0段，这与bootrom不同，在图中我们以虚实线来表示。 内嵌boot有自己单独的代码段，数据段，及堆栈空间，数据及堆栈被锁入cache，指令则因为运行在kseg0空间，得到了L1 I-cache的支持。 内嵌boot在执行完内存配置后，返回bootrom，并清空cache，这时的内存已经是可以使用的了，后续的流程则与其它boot无太大差别。

# BOOT的生成

从前一部分，我们得到了这样的信息：
- bootrom在kseg1运行，内嵌boot在kseg0运行
- bootrom需要知道内嵌boot的数据段范围，才能将相应地址锁入cache，而这个地址是链接时动态产生的。

好，我们来看看VxWorks下的XLR的BOOT是如何编译的：

1. 首先，与正常单板一样，编译出bootrom，但是链接脚本(ld.boot1)，跟普通的有那么一点不同，也就是在整个lds的最后，多了一句boot1_5_start = .;
2. 在编译完bootrom后，进行内嵌boot的编译，因为bootrom已经生成，那么boot1_5_start的地址已由链接器确定，我们nm一下这个符号，得到它的地址。
3. 用这个地址替换内嵌boot使用的lds文件的ENTRY地址，并编译生成boot1_5模块，but wait，boot1_5不是在kseg0运行的么？嗯，我们还需要修改一下nm出的地址，把它改到kseg0段去，当然就是将0xbxxx改成0x9xxx了，这个由老朋友sed完成。
4. 删除bootrom。因为bootrom在boot1_5编译前就生成了，那bootrom又如何知道boot1_5的各段地址呢？只能再来一遍。这时，老朋友nm又来了，它解析出boot1_5中的各段地址(同样是通过在lds上的符号确定)，转换为宏定义，重新编译bootrom，第二次编译bootrom使用的链接脚本是ld.boot2，这个与ld.boot1又有一点不同，就是多了一个boot1_5段的插入:
```
boot1_5_data = .;
    .boot1_5 : {
    *(.boot1_5)
}
```

这样，bootrom就将boot1_5链接进去了。我们知道elf本身是可重链接的文件，如果将boot1_5与boorom原样链接，相同的段会被合并，那么我们还是需要对boot1_5进行一点点处理的，这个步骤如下：
1. boot1_5编译生成elf文件
2. 用objcopy强制将boot1_5转换为bin文件
3. 再使用链接脚本将boot1_5编译为elf文件，当然，这个elf里只有一个段，这就是ld.boot2中的boot1_5段。这个链接文件并不那么神奇，也就是长成下面这个样子。

```
SECTIONS
{
    .boot1_5 :
    {
        *(.text)
        *(.data)
        *(.bss)
    }
}
```

# 小小的改造

当我们把这个boot移植到uboot上时，这个多遍编译似乎有那么点 dirty。让我们改造一下。

因为uboot是用的是-G 0编译，gp实际上是用不到的，那么也就是堆栈和数据段需要处理一下，既然不再有内嵌boot， 当然也无法再单独将内存配置相关的数据锁入cache（当然，如果用扩展语法将每个全局变量都放到指定段，可以实现，但是比较繁琐），可以使用的就是堆栈了，这要求内存配置相关函数不能再使用全局变量，只能使用堆栈，这要修改一大堆函数的入参，更为简洁的方法是使用DECLARE_GLOBAL_PTR，将一个所谓的全局变量与寄存器绑定， 这个可以参考uboot中的gd_t。

编译的问题是好处理了，可boot在配置内存时速度非常慢，因为在内存配置完成前，uboot代码跑在kseg1段， 没有指令cache的支持，慢也是在所难免的，还好，mips的特殊结构使我们可以有选择地优化最费时的函数， 将0xbxxxxxx的指针强制转换成0x9xxxxxxx地址，然后再执行函数调用，这下速度就快多了。
