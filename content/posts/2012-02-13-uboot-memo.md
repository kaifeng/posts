---
layout: post
title: uboot备忘
author: kaifeng
date: 2012-02-13
categories: bsp
---

# 概述

编译命令
- make board_config
- make all
- make distclean
- make clean

新板移植步骤（假定cpu为是mips类型的newcpu，单板名为newboard）
1. copy cpu/mips as cpu/cpuname.
2. copy board/qemu-mips as board/newboard, rename qemu-mips.c as newboard.c
3. copy include/configs/qemu-mips.h as include/configs/newboard.h
4. modify /makefile, add newboard_config rule. eg. @$(MKCONFIG) -a newboard mips cpuname newboard
5. Compile OK.

编译输出
- u-boot: elf output.
- u-boot.bin: binary raw.

代码结构
- board 单板相关文件
- common 平台无关函数
- cpu CPU相关文件
- disk 磁盘相关驱动
- doc 文档
- drivers 设备驱动
- dtt 传感器驱动
- examples 例子程序
- include 头文件
- lib_xxx 特定架构的库文件
- libfdt 支持平坦设备树的库文件
- net 网络代码
- post 自检代码
- rtc 实时时钟
- tools 编译S-Record或U-Boot映像的工具


# 启动流程(mips)

入口
- RVECENT: Reboot vector entry
- XVECENT: Exception vector entry

一条占用8字节（2条指令）

复位，软复位
- NMI的入口为0xbfc00000
- cache异常入口：0xbfc00300
- 其它异常：0xbfc00200

TLB Refill使用异常基地址 0xbfc00200 (boot)
- EXL = 0时，TLB Refill offset = 0，即0xbfc00200
- EXL = 1时，TLB Refill使用General exception异常入口，该异常偏移量为0x180，因此入口为0xbfc00380
```
    0xbfc00000  挂接 reset
    0xbfc00008  挂接reset
    0xbfc00200  romExcHandle
    0xbfc00280  romExcHandle
    0xbfc00300  romExcHandle
    0xbfc00380  romExcHandle
    其它地址     挂接romReserved
```

从_start(start.S)开始执行。

对CP0进行简单配置后（清CPO WatchLo/WatchHi，清status，清计时器COUNT和COMPARE，以免产生中断，将KSEG0配为uncached等），设置gp（注意跳转需要在gp设置之后）

status BEV=1，其它为0，（BEV默认也为1，但不保证软复位的情况）

将Config0 K0设为3，即cachable

将Cause IV置1，即中断使用特殊向量（0x200）

将Wired置为0，On Wired的TLB条目是固定的，不会被tlbwr覆盖，但可通过tlbwi设置

调用lowlevel_init(lowlevel_init.S)，板级最初的初始化，主要是CPU单元，简单的板子在这里就可以完成dram配置了。

调用mips_cache_reset初始化cache并使能。

如果配置了CONFIG_SYS_INIT_RAM_LOCK_MIPS，则调用mips_cache_lock锁定cache，sp指针由CONFIG_SYS_INIT_SP_OFFSET指定设置sp，跳转到board_init_f，这个时候是要求内存可用的，也就是内存必须在lowlevel_init中完成配置。

cpu内存配置可在跳转至board_init_f前插入完成，然后将sp放到内存去，再跳到board_init_f。

board_init_f依次调用init_sequence注册的函数进行初始化，配置bd信息，然后调用relocate_code(start.S)进行重定位。relocate_code最后配置好c语言环境，调用board_init_r。

堆栈指针是入参设置的。

board_init_r配置单板外设，然后进入main_loop。

main_loop下输入Ctrl+C会打印``<INTERRUPT>``，实际上，中断此时并未使能，loop中不停地读取终端，属于轮询操作。

# Relocation

relocation之后的uboot内存分布图
```
    +-------------------------+    Mem top
    |         4k gap          |
    +-------------------------+    16k boundary
    |  uboot code, data, bss  |
    +-------------------------+    <--- addr
    |           gap           |
    +-------------------------+    16k boundary
    |                         |
    |    128k Malloc region   |
    |                         |
    +-------------------------+
    |        bd_t struct      |
    +-------------------------+
    |        gd_t struct      |
    +-------------------------+    <--- id
    |                         |
    |    128k boot params     |
    |                         |
    +-------------------------+
    |    16 bytes stack gap   |
    +-------------------------+    <--- sp
    |                         |
    |         stack           |
    |                         |
    +-------------------------+
    |                         |
    |                         |
    |                         |
    |                         |
    |                         |
    +-------------------------+  RAM base
```

# uboot命令

uboot命令以U_BOOT_CMD声明，命令的原型为
```
int cmd (cmd_tbl_t *cmdtp, int flag, int argc, char *argv[])
```

# 驱动实现

## i2c驱动

i2c驱动与单板相关，若不能利用现有代码的，则需要提供新驱动。可放于cpu目录，也可放于board目录。uboot以i2c_init(CONFIG_SYS_I2C_SPEED, CONFIG_SYS_I2C_SLAVE)调用，参数可由具体实现解释。

定义了CONFIG_HARD_I2C或者CONFIG_SOFT_I2C的，必须提供i2c_init(speed, slave_addr)实现。

定义了CONFIG_CMD_I2C，必须提供下面几个函数实现，原型如下
```
int i2c_read (uchar chip, uint addr, int alen, uchar *buf, int len)
int i2c_write (uchar chip, uint addr, int alen, uchar *buf, int len)
int i2c_probe (uchar chip)
int i2c_set_bus_speed(unsigned int speed)
```

成功返回0,否则返回-1.

```
unsigned int i2c_get_bus_speed(void)
```

返回当前速率

执行i2c probe命令时，会从0~127入参调用i2c_probe，返回0的表明设备存在，其它表明不存在，因此i2c_probe可以实现为一个i2c读操作，如果读失败，认为设备不存在。

由于i2c_read与i2c_write传入len，一般需要实现辅助函数读写单字节
```
static int i2c_read_byte (u8 devaddr, u8 regoffset, u8 * value)
static int i2c_write_byte (u8 devaddr, u8 regoffset, u8 value)
```

这两个函数与具体实现相关。

## netdevice驱动

新增网口驱动需要提供几个函数实现：初始化，发送，接收，停止。原型定义在struct eth_device中，如下
```
int  (*init) (struct eth_device*, bd_t*);
int  (*send) (struct eth_device*, volatile void* packet, int length);
int  (*recv) (struct eth_device*);
void (*halt) (struct eth_device*);
```

执行正常返回1，错误返回0。

如果定义了CONFIG_CMD_NET宏，board_init_r启动主循环前，调用eth_initialize初始化网口，根据CONFIG_NET_MULTI是否定义实现不同，定义该宏表示多个网口，uboot需要从环境变量ethact得到活动的网口。

定义了MULTI的情况下：eth_initialize会依次调用board_eth_init，cpu_eth_init(前一个失败的话)进行网口初始化。这两个是弱符号，可以不实现。

未定义MULTI时，目前的代码中，均以宏分隔实现，新增驱动时需要在此处添加，不如上一种方便。

另一个区别是，不使用MULTI的驱动，函数名必须固定为eth_init, eth_send, eth_rx, eth_halt，是硬解析，如果定义了MULTI，函数则通过函数指针的方式注册，对文件名无要求，这也是为了同时使用多套驱动的体现。两种函数原型有不同，多网口驱动的，多一个入参，类似于驱动结构体。

初始化函数的实现，一般实现过程为：malloc一个eth_device结构体，填充内存数据，包括：init, send, recv, halt函数指针, 指向私有数据的priv。然后调用eth_register将结构注册。注册将其置为eth_current。

接收函数则处理芯片收到的数据，并调用NetReceive进行上报。关于驱动资源释放则与风河实现稍有差异，风河是通过挂接释放钩子，在协议栈完成处理后直接释放，而uboot则通过NetReceive将数据上送给协议栈处理完毕，返回到驱动接收函数，由驱动显示地释放资源。

## mii驱动

定义CONFIG_CMD_MII，增加MII命令支持及操作接口，可用于配置/调试PHY。miiphy操作函数注册接口为(定义在miiphyutil.c)。
```
    void miiphy_register (char *name,
                    int (*read) (char *devname, unsigned char addr,
                           unsigned char reg, unsigned short *value),
                    int (*write) (char *devname, unsigned char addr,
                            unsigned char reg, unsigned short value))
```

CONFIG_MII_INIT：如果定义该宏，则在mii命令操作前，会先调用mii_init执行初始化。

mii 命令的简单说明：mii device列出当前所有注册的驱动，这并非是probe过程。同一时刻只有一个驱动工作，由current_mii决定，对 current_mii->name的操作由miiphy_set_current_dev和miiphy_get_current_dev完成，其它mii辅助函数如mii_read, mii_write都是遍历已注册的驱动列表来查找驱动接口。mii命令集则通过miiphy_get_current_dev得到名称，再调用mii辅助函数实现。

# 编译体系

Makefile为顶层Makefile，$(CURDIR)为makefile内置变量，代表当前目录

## make board_config

编译单板前先要配置一个单板，直接指定Makefile的目标，生成相关的配置文件。
```
    newboard_config : unconfig
         @mkdir -p $(obj)include
         @echo "#define CONFIG_NEWBOARD 1" >$(obj)include/config.h
         @$(MKCONFIG) -a newboard mips cpuname newboard
```

第二行，生成config.h文件，后面还会由makefile添加#include语句，最终的结果是这样：
```
#define CONFIG_NEWBOARD 1
/* Automatically generated - do not edit */
#include <configs/newboard.h>
#include <asm/config.h>
```

## mkconfig

$(MKCONFIG)也就是源码树根目录下的mkconfig:
```
MKCONFIG := $(SRCTREE)/mkconfig
```

该脚本的入参为： ``target architecture cpu board [vendor] [soc]``

针对上面的例子，也就是 ``target=newboard, architecture=mips, cpu=cpuname, board=newboard``

建立软链接，分两种情况，一种是SRCTREE != OBJTREE，要复杂一些，主要做这些事：建立目录 ``$(OBJTREE)/include`` 和 ``$(OBJTREE)/include2``， 在 ``$(OBJTREE)/include2`` 中建立软链接，将asm链接到 ``$(SRCTREE)/include/asm-$2``，进入 ``../include``，建立asm至asm-$2的软链接，不过这两个目录都是空的。

$2是mkconfig的第二个参数，也即architecture

第二种情况很简单，进入include目录，建立一个链接到 ``asm-$2`` 的asm软链接（先删掉以前的asm软链接）。

mkconfig然后在 ``asm-$2`` 中建立一个软链接arch指向 ``arch-$3``，如果SOC不为空则使用 ``arch-$6``。对于newboard的例子，链接为arch指向arch-cpuname，这个目前应该尚未用到。

创建 ``include/config.mk``，将ARCH, CPU, BOARD 都写入 ``config.mk``，下面是一个例子：
```
    ARCH   = mips
    CPU    = cpuname
    BOARD  = newboard
```

将 ``#include <configs/$1.h>``, ``#include <asm/config.h>`` 添加到config.h中，mkconfig的工作便已完成。

## make all

编译配置的单板。

如果定义了BUILD_DIR，则OBJTREE取BUILD_DIR，称之为REMOTE_BUILD，否则取CURDIR。如果取BUILD_DIR，则将obj, src设为指定的地址，否则为空，对应当前目录，以正确地设置obj, src为目标输出的路径：
```
    ifneq ($(OBJTREE),$(SRCTREE))
    obj := $(OBJTREE)/    # 输出到指定目录
    src := $(SRCTREE)/    # 代码仍然是在代码树里
    else
    obj :=
    src :=
    endif

include $(obj)include/config.mk
```

得到ARCH, CPU, BOARD, VENDOR, SOC信息。如果没有设置CROSS_COMPILE，则根据ARCH设置默认的编译器。

注意：只有make board_config之后，才会生成这个文件。

``include $(TOPDIR)/config.mk`` 包含顶层目录的 ``config.mk``

``config.mk`` 的工作为：根据源码，目标的设置，设置dir, obj, src变量，建了一个脚本cc-option来检查CFLAGS中是否有不支持的选项，未见到使用之处。设置编译相关的命令，包括AS, LD, CC, CPP, AR, NM, LDR, STRIP等。
```
sinclude $(OBJTREE)/include/autoconf.mk
sinclude $(TOPDIR)/$(ARCH)_config.mk  包含架构规则
sinclude $(TOPDIR)/cpu/$(CPU)/config.mk  包含CPU特定规则
sinclude $(TOPDIR)/board/$(BOARDDIR)/config.mk  包含单板相关规则
```

SOC和VENDOR不定义，此处不研究。不定义LDSCRIPT的话，默认使用board下的：
```
LDSCRIPT := $(TOPDIR)/board/$(BOARDDIR)/u-boot.lds
```

设置CPPFLAGS，CFLAGS，LDFLAGS等，设置编译规则。

现在回来看看包含的这几个makefile。

顶层下的mips_config.mk是存在的，主要用于添加架构相关的编译选项。

cpu下的config.mk用于配置cpu相关的编译选项，比如大小端配置。

board 下的 ``config.mk`` 则用于配置单板相关的参数，比如TEXT_BASE，链接脚本也是放于单板目录的。值得注意的是第一个 ``autoconf.mk``，这个文件是不存在的，然而makefile的规则使得include的目标可以被生成，由于顶层的Makefile中有autoconf.mk的生成规则，make在找到这个规则时会生成这个文件，然后接着处理，规则如下：
```
    #
    # Auto-generate the autoconf.mk file (which is included by all makefiles)
    #
    # This target actually generates 2 files; autoconf.mk and autoconf.mk.dep.
    # the dep file is only include in this top level makefile to determine when
    # to regenerate the autoconf.mk file.
    $(obj)include/autoconf.mk.dep: $(obj)include/config.h include/common.h
         @$(XECHO) Generating $@ ; \
         set -e ; \
         : Generate the dependancies ; \
         $(CC) -x c -DDO_DEPS_ONLY -M $(HOST_CFLAGS) $(CPPFLAGS) \
              -MQ $(obj)include/autoconf.mk include/common.h > $@
    $(obj)include/autoconf.mk: $(obj)include/config.h
         @$(XECHO) Generating $@ ; \
         set -e ; \
         : Extract the config macros ; \
         $(CPP) $(CFLAGS) -DDO_DEPS_ONLY -dM include/common.h | \
              sed -n -f tools/scripts/define2mk.sed > $@.tmp && \
         mv $@.tmp $@
    sinclude $(obj)include/autoconf.mk.dep
```

主要的处理在$(CPP)这一句，``-dM`` 在预处理过程中输出所有 ``#define`` 语句，包括宏。由于include在预处理时会被展开，因此包含的头文件中的定义也会被列出。标准输出被管道送至 sed处理，define2mk.sed将#define CONFIG_*替换成如下的样子，供编译过程使用：
```
CONFIG_MIPS=y      # 定为 1 或者空值的会被设为 y
CONFIG_SYS_LOAD_ADDR="0x81000000"
CONFIG_BOOTDELAY=10
```

注意sed的替换，如果不指定全局，则只替换每行的第一个匹配。因此如果CONFIG后的值后还有空格，也不会被替换成 = 号。

现在回到Makefile继续往后，这里设置了OBJS变量，将cpu/$(CPU)/start.o加入了这个变量。在cpu目录下，这个还可以再根据需要添加.o至OBJS参与编译。设置LIBS包含:
- lib_generic下的库libgeneric.a, liblzma.a, liblzo.a
- cpu/$(CPU)/lib$(CPU).a库
- lib_$(ARCH)/lib$(ARCH).a库
- fs下的一些库，libnet.a, libdisk.a, 各类驱动库
- LIBBOARD=board/$(BOARDDIR)/lib$(BOARD).a

## 设置SUBDIRS

设置ALL为u-boot.srec, u-boot.bin, System.map的集合，make all即编译这几个目标。

配置了CONFIG_NAND_U_BOOT或CONFIG_ONENAND_U_BOOT时，会参与编译。这几个目标都从u-boot生成。u-boot从SUBDIR, OBJS, LIBBOARD, LIBS, LDSCRIPT生成：
```
    $(obj)u-boot: depend $(SUBDIRS) $(OBJS) $(LIBBOARD) $(LIBS) $(LDSCRIPT)
          UNDEF_SYM=`$(OBJDUMP) -x $(LIBBOARD) $(LIBS) | \
          sed  -n -e 's/.*\($(SYM_PREFIX)__u_boot_cmd_.*\)/-u\1/p'|sort|uniq`;\
          cd $(LNDIR) && $(LD) $(LDFLAGS) $$UNDEF_SYM $(__OBJS) \
               --start-group $(__LIBS) --end-group $(PLATFORM_LIBS) \
               -Map u-boot.map -o u-boot
```

SUBDIRS即编译tools, examples, api_examples下的文件：
```
    $(SUBDIRS):     depend
              $(MAKE) -C $@ all
```

OBJS即编译cpu下的文件：
```
    $(OBJS):     depend
              $(MAKE) -C cpu/$(CPU) $(if $(REMOTE_BUILD),$@,$(notdir $@))
```

LIBS即编译设置的各类库文件：
```
    $(LIBS):     depend $(SUBDIRS)
              $(MAKE) -C $(dir $(subst $(obj),,$@))
```

LIBBOARD即编译单板下的文件，并与LIBS的编译结果链接：
```
    $(LIBBOARD):     depend $(LIBS)
              $(MAKE) -C $(dir $(subst $(obj),,$@))
```

所有目标链接起来形成u-boot。

UNDEF_SYM从LIBBOARD中提取 __u_boot_cmd_xxx 的符号信息，强制将其设为未定义符号传给LD（通过ld的-u选项），等价于链接脚本的EXTERN命令，让ld链接相关的库。

–start-group –end-group 用于解决库之间的符号依赖

rules.mk是用于生成依赖的。

## 常见修改

假如一个cpu/SOMECPU目录下面还要建子目录LOADER，并按uboot正常流程编译，有两种方法：
- 在SOMECPU/Makefile中，如COBJS-y中，直接按相对路径加上.o，比如LOADER/loader.o，这种方法是可行的，但如果obj和src配置是不同时，编译时会提示路径或文件不存在，这需要在Makefile中添加一项$(shell mkdir -p $(obj)LOADER)，建立路径。
- 在TOPDIR/Makefile中，以LIB的形式添加，和drivers下面的层次目录差不多。uboot会遍历LIB来编译，生成库的形式，然后再与libSOMECPU.a链接。

## cmd_link_o_target

这个宏是新的uboot在模块makefile中使用的新的方式。宏在模块的makefile中使用，下面是旧的uboot实现：
```
    $(LIB): $(obj).depend $(OBJS)
            $(AR) $(ARFLAGS) $@ $(OBJS)
```

这个是新的uboot实现：
```
    $(LIB): $(obj).depend $(OBJS)
            $(call cmd_link_o_target, $(OBJS))
    cmd_link_o_target = $(if $(strip $1),\
                             $(LD) -r -o $@ $1 ,\
                             rm -f $@; $(AR) rcs $@)
```

这个宏在OBJS为空时，生成一个空的库，而不为空时，使用LD生成一个可链接的.o。推测这个会加快最终的链接速度。

# 库实现

## malloc

uboot的malloc函数簇定义在include/malloc.h中，实现在common/dlmalloc.c中

# Qemu下运行uboot

步骤:
1. 从源码编译Qemu (e.g. qemu-0.12.3)
2. 编译 u-boot.bin (e.g. u-boot-2010.03, qemu_mips_config)
3. dd of=flash bs=1k count=32k if=/dev/zero
4. dd of=flash bs=1k conv=notrunc if=u-boot.bin
5. qemu-system-mips -M mips -pflash flash -monitor null -nographic

不需要打补丁及任何代码修改。 
