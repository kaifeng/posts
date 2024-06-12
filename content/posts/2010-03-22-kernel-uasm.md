---
layout: post
title: 微汇编器uasm分析
author: kaifeng
date: 2010-03-22
categories: kernel
---

本文分析了Linux内核关于uasm的代码。

微汇编器是内核实现的一组指令构造接口，主要用于构造TLB异常处理函数。或许是因为MIPS硬件做的事情太少，才有了软件这样的灵活实现，它也只在MIPS下存在，算是这个架构的一个特色吧。
MIPS指令大体上分为三类：立即数、跳转、寄存器，如图：

![MIPS指令分类](/posts/assets/mips-instr-cat.png)

图：MIPS指令分类

微汇编器的核心函数是build_insn，而它又是通过调用一系列基本函数build_xxx构造出指令的各个部分。

## 指令描述

指令由下面结构体描述:
```
struct insn {
    enum opcode opcode;
    u32 match;
    enum fields fields;
};
```

其中，enum opcode 定义了一组指令，用来表示可以在这个微型汇编器里产生的指令编码，它与真实的指令编码没有关系，只是用于索引，真正的指令编码是在inst.h里定义的。

match用于保存指令编码，但只会保存指令本身已经明确的编码部分，像可变部分，比如寄存器编号，则留给构造函数完成。

fields包含着该指令由哪些部分构成的信息，这是通过位设置确定的。

## 基本函数


首先看基本函数，它们用于构建指令的对应字段，并带有溢出检查。只要了解MIPS指令编码，不难理解。这些基本函数如下：

- build_rs  产生指令的rs部分，即[25:21]
- build_rt   产生指令的rt部分，即[20:16]
- build_rd  产生指令的rd部分，即[15:11]
- build_re     产生指令的sa部分，即[10:6]，代表5位的偏移量
- build_simm   产生指令的有符号16位立即数，形成低16位，因此该函数的入参是signed类型的。
- build_uimm   产生指令的无符号16位立即数，形成低16位
- build_bimm   构造branch系列指令的立即数部分。
  - branch系列指令的立即数也是存放于指令编码的低16位，但它代表的是18位的有符号数（跳转范围是±128K），低2位被移位。
  - 该函数将入参转换成16位的offset，如果入参低2位不为零，则入参错误。
  - 需要正确处理符号问题：`(arg < 0) ? (1 << 15) : 0) | ((arg >> 2) & 0x7fff)`
- build_jimm   构造j系列指令的立即数部分（只有j和jal带立即数）。立即数是28位的，与branch一样，指令仅保存高26位。立即数代表的是256M内的偏移量，所以无符号。
- build_func   产生指令的低6位，表示功能号。
- build_set    产生设置指令的低3位，表示sel。
- build_insn   根据指令表中的指令定义，调用基本函数进行指令生成。

## 指令构造

### 指令描述表

你见，或者不见，都有一个名为insn_table[]的表就在那里，存放着指令编码相关的信息，不悲，不喜。表元素是struct insn类型的，我们举例分析:

```
{ insn_addiu, M(addiu_op, 0, 0, 0, 0, 0), RS | RT | SIMM },
```

insn_addiu 是 enum opcode 定义过的枚举变量，代表着这一个元素描述的是addiu这条指令的编码信息，fields的设置表示这条指令使用了 rs, rt, simm 三个字段，这与指令定义是一致的。
M是一个宏，它将各字段组成起来，其定义如下:

```
#define M(a, b, c, d, e, f)                      \
    ((a) << OP_SH                              \
    | (b) << RS_SH                             \
    | (c) << RT_SH                             \
    | (d) << RD_SH                             \
    | (e) << RE_SH                             \
    | (f) << FUNC_SH)
```

OP_SH ~ FUNC_SH 对应着指令各部分的偏移量。对addiu指令而言，也就是说只有 opcode 字段是可以确定具体值的，其它的都需要后续再予以处理。

并不是所有指令都会用到所有字段，比如 addiu 就只有 RS 和 RT（立即数不在该宏的处理范围内），只能作赋0就处理。尽管有多余的字段，M宏依然无法列出指令的所有组成部分，比如 sel 就与 function 区间重叠，那么在生成形如 mfc0 t0, $16, 1 这样的指令时，也就只能将f赋为0，再通过 build_set 构造 set 字段。
下面是mfc0指令的编码定义：

```
{ insn_mfc0,  M(cop0_op, mfc_op, 0, 0, 0, 0),  RT | RD | SET},
```

值得注意的是，mfc_op与rs区间是相同的，虽然会容易误解，但结果仍然是正确的。

### build_insn

看到这里，看懂build_insn也就是顺水推舟的事了，这个函数采用了变参的形式:

```
static void __cpuinit build_insn(u32 **buf, enum opcode opc, ...)
{
    struct insn *ip = NULL;
    unsigned int i;
    va_list ap;
    u32 op;
    /* 首先在指令表里查到opc的指令信息 */
    for (i = 0; insn_table[i].opcode != insn_invalid; i++)
        if (insn_table[i].opcode == opc) {
            ip = &insn_table[i];
            break;
        }
    if (!ip || (opc == insn_daddiu && r4k_daddiu_bug()))
        panic("Unsupported Micro-assembler instruction %d", opc);
    /* 把指令半成品取出来 */
    op = ip->match;
    /* 从参数列表中取出参数来，检查fields的设置，比如设置了RS，那么第一个变参就需要提供rs的数据，
    调用者一定要按照下面的条件判断顺序来提供变参（只按置位的fields），
    这样处理就需要调用者非常小心谨慎地处理入参顺序，这是个不小的麻烦。
    是的，代码作了很好的处理，我们后面再说 */
    va_start(ap, opc);
    if (ip->fields & RS)
        op |= build_rs(va_arg(ap, u32));
    if (ip->fields & RT)
        op |= build_rt(va_arg(ap, u32));
    if (ip->fields & RD)
        op |= build_rd(va_arg(ap, u32));
    if (ip->fields & RE)
        op |= build_re(va_arg(ap, u32));
    if (ip->fields & SIMM)
        op |= build_simm(va_arg(ap, s32));
    if (ip->fields & UIMM)
        op |= build_uimm(va_arg(ap, u32));
    if (ip->fields & BIMM)
        op |= build_bimm(va_arg(ap, s32));
    if (ip->fields & JIMM)
        op |= build_jimm(va_arg(ap, u32));
    if (ip->fields & FUNC)
        op |= build_func(va_arg(ap, u32));
    if (ip->fields & SET)
        op |= build_set(va_arg(ap, u32));
    va_end(ap);
    /* 将指令编码写入buf，指针自加，就像pc一样，写一条指令往后跳一个，多么优美 */
    **buf = op;
    (*buf)++;
}
```

### uasm_i_xxx系列函数的形成

在build_insn之上，还有一组宏uasm_i_xxx，在此基础上，形成具体的指令生成接口。
I_xxx(op)系列宏定义被展开为一组函数实现，以I_u1u2u3(op)为例:

```
#define I_u1u2u3(op)                         \
Ip_u1u2u3(op)                                \
{                                            \
    build_insn(buf, insn##op, a, b, c);    \
}
```

Ip_u1u2u3(op)定义在uasm.h中:

```
#define Ip_u1u2u3(op)                                   \
void __cpuinit                                          \
uasm_i##op(u32 **buf, unsigned int a, unsigned int b, unsigned int c)
```

将宏展开，便形成了 uasm_i_xxx(u32 **buf, unsigned int a, unsigned int b, unsigned int c) 的函数定义，这个函数的实现就是调用build_insn。

注意到I_xxx与Ip_xxx的xxx部分都是u加上数字，这里的u表示unsigned int（自然也有s表示signed int）。那么数字呢？数字的用途在于调换入参顺序。因为build_insn是按从左到右的顺序确定入参来执行指令构造的，从build_insn函数的实现可以看出，顺序为RS, RT, RD, RE, SIMM, UIMM, BIMM, JIMM, FUNC, SET，在生成指令时，如果指令格式与机器码字段顺序不是一致的对应关系，就用数字来调整。调用者使用架构手册所定义的指令语法传入参数，底层实现则根据语法与实际条件判断的差异，由数字进行调整，PERFECT！我们以addiu为例，它的函数声明由宏展开得到：

```
#define Ip_u2u1s3(op)                                   \
void __cpuinit                                          \
uasm_i##op(u32 **buf, unsigned int a, unsigned int b, signed int c)
Ip_u2u1s3(_addiu);
```

也即:

```
uasm_i_addiu(u32 **buf, unsigned int a, unsigned int b, signed int c)
```

声明也只是声明，它的实现在uasm.c里:

```
#define I_u2u1s3(op)                        \
Ip_u2u1s3(op)                               \
{                                           \
    build_insn(buf, insn##op, b, a, c);   \
}
I_u2u1s3(_addiu)
```

展开一下，就是：

```
uasm_i_addiu(u32 **buf, unsigned int a, unsigned int b, signed int c)
{
    build_insn(buf, insn_addiu, b, a, c);
}
```

哈，参数被置换了。有句话怎么说来着，流氓不可怕，就怕流氓有文化。因为build_insn是一个变参，所以编译器也不会对变参进行类型检查，这种被认为是缺陷的特性，在这里却被利用得恰到好处。

我们看到Ip_xxx系列函数入参定义其实并不考虑数字顺序，而将I_xxx与Ip_xxx定义一致，就可以很直观地在I_xxx里进行顺序调整。今后，看这些函数时，就可以直接对照手册的指令格式来看了。

## 标号与跳转

也许你现在已经有点头晕了，我也是，结果遗憾终身。如果上天给我一次重新来过的机会，我会说对自己说，再坚持一会，所以现在让我们一鼓作气，把它一举拿下。

与跳转有关的一共是两个结构：uasm_label, uasm_reloc。简单地说，uasm_label用于记录标号与地址的对应关系。uasm_reloc用于记录哪些指令地址用到了标号，执行哪种跳转。

围绕这两个数据结构，也自然地划分了标号与跳转两组函数。

### 标号函数

uasm_l_xxx是标号函数，它们由UASM_L_LA宏生成（定义在uasm.h），其实现也就是调用uasm_build_label:

```
#define UASM_L_LA(lb)                                                          \
static inline void __cpuinit uasm_l##lb(struct uasm_label **lab, u32 *addr)    \
{                                                                              \
    uasm_build_label(lab, addr, label##lb);                                  \
}
```

比如uasm_l_leave这个函数，就是创建了一个label_leave标号，标号就是预定义的整数，所以在使用标号前，你得先定义好它，就像tlbex.c里定义的label_id那样，然后在可见范围内用UASM_L_LA声明像uasm_l_leave这样的标号函数。

uasm_build_label的实现倒是很简单：

```
void __cpuinit uasm_build_label(struct uasm_label **lab, u32 *addr, int lid)
{
    (*lab)->addr = addr;
    (*lab)->lab = lid;
    (*lab)++;
}
```

就是把标号与地址关系记录下来，简单得一塌糊涂。一句uasm_l_leave(&l, p)，就可以把当前地址标记上label_leave了。

### 跳转函数

跳转函数组的实现基本类似，以beqz为例:

```
void __cpuinit
uasm_il_beqz(u32 **p, struct uasm_reloc **r, unsigned int reg, int lid)
{
    uasm_r_mips_pc16(r, *p, lid);
    uasm_i_beqz(p, reg, 0);
}
```

基本上是通过uasm_r_mips_pc16记录标号，然后构造相应的跳转指令。不过让我们再看一眼，beqz的指令生成，为什么立即数部分是0呢？因为这时还无法确定跳转偏移是不是，所以后面会有一个标号解析的过程。现在我们继续分析uasm_r_mips_pc16:

```
void __cpuinit
uasm_r_mips_pc16(struct uasm_reloc **rel, u32 *addr, int lid)
{
    (*rel)->addr = addr;
    (*rel)->type = R_MIPS_PC16;
    (*rel)->lab = lid;
    (*rel)++;
}
```

依然是那么地，不复杂是吧？uasm_reloc记录了跳转指令的地址，跳转的标号，以及跳转类型。现在只有一种跳转类型，事实上，构造TLB异常处理函数，也不需要支持更多的跳转，麻雀虽小，五脏俱全。

### 地址解析

在指令代码生成完后，该派上用场了吧，但是等等，我们可以直接把生成的指令拷过去运行吗？不行，因为标号还没有解析，所以当你在诸如build_r4000_tlb_refill_handler这样的函数最后看到uasm_resolve_relocs(relocs, labels)这样的调用，可以选择淡定，你可以对大家说，我这个年纪，什么都可以接受。

```
void __cpuinit
uasm_resolve_relocs(struct uasm_reloc *rel, struct uasm_label *lab)
{
    struct uasm_label *l;
    for (; rel->lab != UASM_LABEL_INVALID; rel++)
        for (l = lab; l->lab != UASM_LABEL_INVALID; l++)
            if (rel->lab == l->lab)
                __resolve_relocs(rel, l);
}
```

uasm_resolve_relocs遍历uasm_labels调用__resolve_relocs进行标号解析。
UASM_LABEL_INVALID=0，所以在构造指令前，memset一下uasm_labels结构体，轻易地就将不使用的标号置为无效。当然，在enum label_id时，第一个元素记得要赋一个非零值。简单是美好的，但若是爱上简单，你需要配置一台带“好记性”的大脑（不是好记星哈），如果没有，那就跟我一样用烂笔头吧。

```
static inline void __cpuinit
__resolve_relocs(struct uasm_reloc *rel, struct uasm_label *lab)
{
    long laddr = (long)lab->addr;
    long raddr = (long)rel->addr;
    switch (rel->type) {
    case R_MIPS_PC16:
        *rel->addr |= build_bimm(laddr - (raddr + 4));
        break;
    default:
        panic("Unsupported Micro-assembler relocation %d",
                rel->type);
    }
}
```

laddr是标号对应的地址，而raddr是跳转指令所在的地址，将laddr - (raddr + 4)即得到跳转指令所需的offset，解析完成，干净利落。

为什么要加4呢？这困扰了我好大一会，虽然怀疑是延迟槽的问题，但没有找到直接证据，2B or not 2B, that is the question。最终还是在架构手册里找到了说明：

> A taken branch assigns the target address to the PC during the instruction time of the instruction in the branch delay slot.

终于可以美满地洗洗睡了。
