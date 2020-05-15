---
layout: post
title: MIPS中断在uboot下的实现
author: kaifeng
date: 2010-05-27
categories: bsp
---

摘要：本文阐述了MIPS中断在u-boot下的整个实现过程，并对实践中遇到的问题进行了分析，特别分析了u-boot所采用的位置无关代码及汇编实现的特殊性，对类似应用具有较好的借鉴意义。

随着Linux嵌入式系统的广泛使用，使用uboot作为bootloader越来越多。u-boot发源于ppc架构8xx系列，对ppc架构的支持最为完善，而对其它架构的支持情况则要弱一些，以MIPS为例，目前的最新版本u-boot-2010.03仍然不支持异常和中断的处理。缺少中断机制带来了一些麻烦，比如不能支持统一的点灯规范，更重要的是在无中断的单任务环境下，协议栈缺乏有效的超时重传机制，在复杂的框上环境中，单板容易假死在boot中。

本文描述了MIPS架构中断在u-boot下的实现过程，并对遇到的问题进行了分析和总结。我们首先讨论了通用的异常处理过程，然后结合代码实现对异常处理代码进行了阐述，最后对调试过程中出现的问题进行了分析。u-boot有其自身的特点，本文同时也对位置无关代码进行了一定的分析。

中断处理在实现上并不复杂，其处理过程可以简单地划分为下面几个步骤：

1. 保存中断上下文
2. 处理中断并应答
3. 恢复中断上下文
4. 使能中断

这种处理方式比较简单，整个中断处理是在中断环境中完成的，没有中断嵌套。对于VxWorks这样的实时操作系统，需要允许高优先级中断抢占低优先级中断，因此它的处理步骤要复杂一些，在进入中断后，只保存少量的易失性寄存器，并尽早地开启中断。因为我们只需要实现简单的定时器，所以就按照最简单的方法实现。

MIPS CPU产生中断时，CPU跳转到EBASE+0x180或者0x200（取决于COP0 Cause.IV的设置。IV=1时，中断使用0x200入口，IV＝0时，中断使用通用异常入口0x180）处开始执行中断处理程序。一般来说，异常向量处的大小都是受限的，MIPS也不例外，每个异常向量的大小不能超过0x80的长度，如果要在这里进行上下文保存，空间是不够的。惯例是在这里设一处跳转，到一个大小不受限的地方开始保存上下文。

上下文的保存可以分为两种实现，一种是使用当前任务的栈，一种是使用专门的栈，对于只工作在内核态的boot来说，这两者在使用上并无差别。理论上，开辟专门的栈可靠性更高，不容易被破坏。对于不同的架构，上下文的含义不同，对MIPS而言，上下文包括了32个通用寄存器$0~$31，两个整数运算寄存器HI和LO，还有COP0的几个关键寄存器，如STATUS、CAUSE和EPC等，如果支持MMU，则需要保存更多的寄存器。

在MIPS架构下，中断也是一种异常。CPU在进入异常处理程序前，COP0 Status.EXL被置位，表明CPU进入异常状态。EXL在置位的情况下，会禁止软/硬件中断，这使得中断处理程序在一个安全可靠的环境中运行（当然，不包括中断处理程序再次引发异常）。中断处理程序需要确认中断，否则退出中断时，将再将进入中断，CPU会一直运行在中断处理程序中，不再处理其它事务，也就是常见的中断挂死。

中断处理完毕后，将之前保存在内存中的各类寄存器恢复出来，然后返回到被中断的代码继续执行，这就是中断处理的整个过程。

我们先实现一个最简单的中断处理，借以跑通中断流程，中断是采用Timer中断，也即7号中断。
```
void trap_init(ulong arg)
{
    uint32_t val;
    ebase = K0BASE;
    /* clear SR.BEV */
    val = read_c0_status();
    val &= ~(1<<22);
    write_c0_status(val);
    /* clear CAUSE.IV */
    val = read_c0_cause();
    val &= ~(1<<23);
    write_c0_cause(val);
    /* set ebase */
    write_c0_ebase(ebase);
    /* install generic exception hander */
    set_handler(0x180, except_generic_handler, 0x80);
}
```

Status.BEV控制了异常入口的基地址，清除Status.BEV，让异常入口基地址切换到EBASE，这样在异常产生时，就不会从FLASH上取数据了。Cause.IV则是控制中断的入口偏移，因为我们中断及异常的处理例程是同一个，我们将其清除，使中断入口与通用异常入口一致，这样就不需要分开处理。except_generic_handler是我们的异常处理函数，set_handler的工作就是将函数代码拷贝至异常向量地址并刷新cache。

except_generic_handler的实现很简单，如下:

```
NESTED(except_generic_handler, 0, sp)
    .set push
    .set noreorder
    .set noat
    la  k1, handle_exception
    jr  k1
    nop
    .set pop
END(except_generic_handler)
```

前面提到过异常向量的大小限制，因此这里我们直接跳转到handle_exception，这才是我们真正的异常处理程序。

```
NESTED(handle_exception, PT_SIZE, sp)
    move k1, sp
    addi sp, sp, -PT_SIZE
    SAVE_ALL
    move a0, sp
    la  ra, ret_from_exception
    la  t9, do_exception
    jr t9
    nop
END(handle_exception)
```

前面提到u-boot有其自身的特点，它需要一个不使用的寄存器来作为引用重要数据的指针，也就是gd。在MIPS架构下，u-boot使用k0作为gd，这也表明，在u-boot下的中断处理里，我们只有一个免费寄存器k1可以使用。少一个可用的寄存器是个麻烦事，在异常处理里，不能在汇编阶段做更多的事，好在我们还足以用它来保存上下文。我们先将堆栈指针赋给k1，然后预留一段栈空间，大小为 PT_SIZE。预留栈的大小取决于需要保存的寄存器数目，以及每个寄存器占用的空间。

SAVE_ALL 是一个宏，用于将寄存器保存至堆栈。MIPS使用a0-a3传递参数，将sp传给a0，使C函数do_exception可以从入参取得已保存的栈数据，方便打印异常信息。设置返回值寄存器ra，在do_exception函数返回时，可以使代码从ret_from_exception继续执行。

一切就绪，就该进入C函数了，这由两条指令实现：加载函数地址到t9，以t9作为跳转寄存器执行跳转（以t9作为跳转寄存器是MIPS ABI的约定）。为什么需要经过寄存器跳转呢？这是由于u-boot使用了位置无关代码（PIC）进行编译。理解位置无关代码对我们的汇编实现非常重要，我们后面将以代码为例对它作一个分析。

我们实现的是一个定时器中断，它涉及两个寄存器：Count和Compare。COP0 Count寄存器以CPU核时钟频率计数，当与Compare寄存器相等时，触发一个Timer中断。在do_exception异常处理函数里，我们读出当前的Count值，加上一个将频率换算为周期的数值，保存到Compare寄存器中，写Compare寄存器将清除Timer中断，中断处理就至此结束了，而Count将持续增长，直到与我们设置的Compare值相等时再次产生中断:

```
void do_exception(struct pt_mips_regs *regs)
{
    uint32_t count;
    count = read_c0_count();
    count += FREQ/1000;
    write_c0_compare(count);
    counter++;
    return;
}
```

作为中断流程的实验，do_exception目前只处理一种情况，即只处理定时器中断，后面我们可以继续完善它，现在是该返回的时候了。

由于在handle_exception中，我们将ra设置为ret_from_exception，因此在do_exception函数返回时，将从ret_from_exception继续执行。

```
FEXPORT(ret_from_exception)
    RESTORE_ALL
    eret
    nop
```

同样，RESTORE_ALL是一个宏，用于从堆栈恢复上下文，在所有寄存器恢复后，调用eret指令退出异常状态。eret会清除 Status.EXL，在中断处理过程中，并未对中断使能位进行修改，因此EXL的清除将使能所有的中断，同时eret会跳转至EPC寄存器保存的地址（在发生Reset, Soft Reset, NMI及Cache Error异常时，eret使用ErrorEPC的地址，请参考MIPS手册）。

使用-fpic选项，可以指示gcc以位置无关的方式编译代码。这对于C语言编写来说通常是不可见的，编译器隐藏了所有的细节，然而在编写汇编代码时，我们必须要了解它，否则就无法理解变量的引用及函数的跳转。

经过PIC编译的代码，对外部符号的引用都是根据pc相对地址的寻址方式，这使得u-boot代码在relocation到任意地址后都可以正确运行，因为相对地址早在编译期间就被编译器所确定了。通过反汇编do_exception，我们可以看到这个过程。

```
bfc09c30 <do_exception>:
bfc09c30: 3c1c0006  lui     gp,0x6
bfc09c34: 279c1c10  addiu   gp,gp,7184
bfc09c38: 0399e021  addu    gp,gp,t9
bfc09c3c: 40034800  mfc0    v1,$9
bfc09c40: 3c02000e  lui     v0,0xe
bfc09c44: 34427ef0  ori     v0,v0,0x7ef0
bfc09c48: 00621821  addu    v1,v1,v0
bfc09c4c: 40835800  mtc0    v1,$11
bfc09c50: 8f83859c  lw      v1,-31332(gp)
bfc09c54: 8c620000  lw      v0,0(v1)
bfc09c58: 24420001  addiu   v0,v0,1
bfc09c5c: 03e00008  jr      ra
bfc09c60: ac620000  sw      v0,0(v1)
```

C代码中的counter是一个全局变量，对比反汇编代码，我们可以看到counter变量的地址是通过gp加偏 移量确定的。首先gp被赋为0x61c10，然后与t9相加，也即与do_exception的地址0xbfc09c30相加，得到0xbfc6b840，再减去-31332得到0xbfc63ddc，这就是counter符号在GOT表中的地址，我们可以用readelf –x.got u-boot打印出GOT表中的内容，同时用nm查得counter的实际地址进行对照:

```
GOT: 0xbfc63dd0 bfd3ea80 bfc65418 bfc1ec68 bfc65400
NM:  bfc65400 B counter
```

可见，0xbfc63ddc中存放的正是counter的地址。

一切看起来都那么顺利，不是吗？遗憾的是，现实总是那么地残酷。在每秒钟产生1~5个中断时，还比较稳定，一旦超过10Hz，单板比较容易挂死，而设为500Hz~1000Hz时，简直就是必死无疑。单板跑死时，串口没有打印，CPU目前处于一个什么状态，不得而知，只能添加少量打印来判断，在中断上下文打印，有两个限制：

* 中断频率也不能设置过快，否则就是满屏的打印。
* 打印的位置不是随意的，因为打印需要用到寄存器，在寄存器未被保存之前是不能使用的。

因此通过打印是不太可行的，经过分析后，认为CPU应该是跑飞了。影响pc的有哪些？除了epc外，很重要的就是gp，因为前面我们也提到过，gp用于索引GOT表，有了这个明确的提示，脑海中突然浮现出中断入口函数的代码实现，这个可能有问题！我们再回头看看这两个指令:

```
la  k1, handle_exception
jr  k1
```

话不多说，直接看反汇编:
```
bfc0a2ac:    8f9b832c    lw k1,-31956(gp)
bfc0a2b0:    03600008    jr k1
```

一切都恍然大悟。我们在取handle_exception的地址时，已经默认地使用了gp寄存器的值，而如之前的反汇编所见，在pic代码中，所有的符号地址，都是通过gp计算的，那么在中断进入时，gp的值是不确定的。假设我们刚进入一个函数，gp已被修改，突然中断来临，中断入口取出的 handle_exception的地址就肯定是错误的，必然会跑飞。

知道了问题所在，是一个大的突破，下面的问题就是如何设置正确的gp，并且不能破坏被中断代码的gp。从u-boot的链接脚本，我们可以看到_gp被设计为指向GOT表:
```
_gp = ALIGN(16) +0x7ff0;
.got  : {
__got_start = .;
*(.got)
__got_end = .;
}
```

而在start.S的最初代码里，进行第一个函数调用前，将gp设置成了GOT表的首地址:
```
    bal     1f
    nop
    .word   _gp
1:
    lw      gp, 0(ra)
```

在relocation时，gp与GOT表也会被修改，他们的相对地址仍然保持不变。如果我们在relocation后，保存gp的值，是否可以留给中断处理程序使用呢？答案是肯定的，不过需要想办法把gp保存在某个地方。很自然地，想到了k0。k0指向gd_t数据，一旦设置后，在整个boot阶段都不会发生改变，我们可以在gd_t中增加一个成员来保存gp值，为了方便索引，就把它作为第一个成员吧:
```
typedef struct  global_data {
    unsigned long   gp;  /* saved gp */
    bd_t            *bd;
    unsigned long   flags;
    ...
```

board_init_r是u-boot在relocation后执行的第一个函数，我们在这里保存重定向后的gp寄存器。
```
asm __volatile__("move %0, $28\n":"=r"(gd->gp));
```

异常向量入口except_generic_handler修改如下:
```
/* trade gp */
lw k1, 0(k0)
sw gp, 0(k0)
move gp, k1
la  k1, handle_exception
jr  k1
nop
```

先将gd->gp的值读出到k1，然后将被中断的代码的旧gp保存到gd->gp中，将GOT表基地址赋给gp指针，此时gp已经是确定的值了，对handle_exception的地址解析也不会再出错了。

在ret_from_exception中，还需要做相同的工作，将旧的gp恢复，GOT表基址保存回gd->gp中去。至此，中断处理程序已经可以稳定正常工作了，目前在5MHz的中断频率下，也可以稳定工作并正常下载版本。

中断处理已经跑通，我们可以继续完善异常处理了。由于我们在汇编环境下只有一个k1寄存器可以使用，因此判断异常类型的工作就交到了do_exception中。入参regs是在handle_exception传入的，它是预留了栈后的sp指针，我们以图说明。
```
     +-------------------------+
     |                         |
     |  Stack before exception |
     |                         |
     +-------------------------+    <---------- old sp
     |      cp0 registers      |  \
     +-------------------------+   |
     |           ...           |   |
     +-------------------------+   |
     |           $31           |   |
     +-------------------------+   |  PT_SIZE
     |            $2           |   |
     +-------------------------+   |
     |            $1           |   |
     +-------------------------+   |
     |            $0           |  /
     +-------------------------+    <---------- new sp
     |                         |
     |  Stack for do_exception |
     |                         |
     +-------------------------+
```

old sp是中断产生时的栈指针，new sp是我们开辟了PT_SIZE大小的栈空间后，调整后的栈指针，而PT_SIZE所包含的栈空间则是用于保存我们的寄存器，new sp作为入参传入do_exception。

struct pt_mips_regs是一个结构体，只要我们将此结构体的定义与栈结构相同，就可以直接利用编译器产生的偏移取用栈数据。最终的处理框架如下：是中断产生时的栈指针，new sp是我们开辟了PT_SIZE大小的栈空间后，调整后的栈指针，而PT_SIZE所包含的栈空间则是用于保存我们的寄存器，new sp作为入参传入do_exception:
```
void do_exception(struct pt_mips_regs *regs)
{
    unsigned int excode;
    unsigned int int_mask;
    unsigned int status;
    int irq;
    status = regs->c0_status;
    excode = (regs->c0_cause >> 2) & 0x1f;
    int_mask = (status >> 8) & 0xff;
    switch(excode)
    {
        case 0:
            irq = 7;
            while(int_mask != 0)
            {
                if((int_mask & (1<<irq)) != 0)
                {
                    do_int(irq);
                }
                int_mask &= ~(1<<irq);
                --irq;
            }
            break;
        default:
            /* unhandled exception goes to default,
             * if you can handle certain exceptions,
             * process it above.
             */
            printf("\n\nUnable to handle %s exception:\n\n", exc_desc[excode]);
            dump_stackframe(regs);
            printf("\n\nAttempting to reboot ...\n");
            _machine_restart();
            while(1);
    }
       
    return;
}
```

当异常码为0时，我们进入中断处理，这由do_int完成。当异常码为其它值时，打印出异常信息，这由dump_stackframe实现，它将 struct pt_mips_regs结构中的数值全都打印出来，类似于内核的panic打印。值得一提的是u-boot在重定向后，如果产生异常，它的epc是重定向后的RAM地址，而通过反汇编u-boot得到的是ROM地址，这给精确定位带来不便，好在问题的解决比较容易，u-boot在重定向后，会将偏移保存在gd->reloc_off中。
```
gd->reloc_off = dest_addr - CFG_MONITOR_BASE;
```

在出现异常时，可以利用该值对epc进行换算，得到异常地址对应的ROM地址，只需要简单地用epc减去gd->reloc_off就可以了。由于 CFG_MONITOR_BASE一般是0xbfc00000，而u-boot重定向后一般运行在K0段，这样得到的reloc_off是一个补码，只要统一按照等位宽的无符号数处理，则无需关心这个细节。

是否需要在bootloader中实现中断，一直是一个有争议的问题，大多数人认为bootloader的功能应当最简单，它的任务就是尽可能早地启动操作系统，也有不少人因为实际应用需要，希望在boot里实现更多的功能，这是个哲学问题，这里不打算深入讨论。从应用来看，有时需要实现MIPS下的中断，以解决一些实际问题。目前还没有开源boot支持MIPS下的中断，因此本文的工作是一个新的尝试，相信对类似应用会有很好的指导作用。

关于MIPS架构的知识，请参考架构手册
* MIPS64 Architecture for Programmers Volume II - The MIPS64 Instruction Set v2.50
* MIPS64 Architecture for Programmers Volume III - The MIPS64 Privileged Resource Architecture v2.50
