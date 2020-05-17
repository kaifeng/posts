---
layout: post
title: VxWorks 5.5下访问MIPS64位地址空间
author: kaifeng
date: 2008-10-14
categories: bsp
description: 如何在 32 位的 VxWorks 5.5 下访问 MIPS64 位地址空间。
---

MIPS64提供64位地址空间，按bit63..62划分为4个段。兼容的32位地址空间通过bit31的符号扩展，划分到两个非连续的区间。32位空间以bit31..29划分为段。

Mapped是指需要经过TLB翻译的空间。Unmapped是指不通过TLB，提供一个窗口访问低端同等大小物理地址的空间。Cache与Uncached指是否经过缓存，比如kseg1是不经过缓存的。

要访问64位地址空间，我们必须使能64位地址，这在CP0 Status寄存器中体现为：

- Status[KX]=1时，可以访问内核64位地址空间
- Status[SX]=1时，可以访问超级用户64位地址空间
- Status[UX]=1时，可以访问用户64位地址空间

处理器的工作模式与64位地址使能与否无关。比如，处理器工作在内核模式下，访问用户空间地址由UX控制，而非KX。 如果试图访问没有使能的64位地址空间，将引起地址错误异常。

然而，由于vxworks5.5不支持64位系统，即使直接使能64位地址也无法访问，因为指针就是32位的，那么我们必须就要提供一个接口，这个接口可以是下面这样:
```
	 __asm__ (
	    ".set push\n"
	    ".set noreorder\n"
	    "or $8, %1, $0\n"
	    "lw %0, 0($8)\n"
	    ".set pop":"=r"(val):"r"(addr):"$8");
```

其中，addr为64位地址，我们定义为unsigned long long，val为一个32位的数据类型。可以这样做的原因是:

1. mips cpu本身就是64位的
2. vxworks虽不支持64位指针，但我们可以利用内嵌汇编的形式，将64位数据类型unsigned long long作为指针使用。

同时，按照MIPS规范，xkphys实际上划分为8个段，每个段都是到物理地址的一个窗口，每个段对应一种cache coherency，而对我们有用的只有两个地址段，即：

- Uncached，对应空间0x9000000000000000 ~ 0x9000000000000000 + 2^(PABITS -1)
- Cacheable，对应空间0x9800000000000000 ~ 0x9800000000000000 + 2^(PABITS -1)

可以看到，每个地址段提供了一个巨大的窗口空间(2^59字节)，通过访问这两段空间，我们事实上取得了对全部物理地址空间的直接访问能力，并且不需要TLB的支持。提供访问64位空间的方法可以解决虚地址空间不足的问题。

xkphys 64位地址访问代码:
```
	WORD32 enable_kx()
	{
	    int status;
	    __asm__ __volatile__(
	        ".set push\n"
	        ".set noreorder\n"
	        "mfc0 %0, $12\n"
	        "or %0, %0, 0x80\n"
	        "mtc0 %0, $12\n"
	        ".set pop\n"
	        :: "r"(status));
	   return 0;
	}
	void read_xkphys(unsigned int offset)
	{
	    unsigned int val;
	    unsigned long long addr = 0x9000000000000000;
	    addr += offset;
	         __asm__(
	        ".set push\n"
	        ".set noreorder\n"
	        ".set mips64\n"
	        "or $8, %1, $0\n"
	        "lw %0, 0($8)\n"
	        ".set pop":"=r"(val):"r"(addr):"$8"
	    );
	 
	    printf("%04x\n", val);
	 
	    return;
	}
```
