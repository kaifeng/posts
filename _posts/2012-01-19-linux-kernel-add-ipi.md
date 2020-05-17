---
layout: post
title: 如何添加核间中断
author: kaifeng
date: 2012-01-19
categories: kernel
---

注意：以下内容 MIPS 为例，其它小系统会有差异，但原理应该相同。

只要是运行 SMP，是必须向内核注册一组 SMP 操作函数的，内核定义这个 API 接口如下：
```
    struct plat_smp_ops {
        void (*send_ipi_single)(int cpu, unsigned int action);
        void (*send_ipi_mask)(const struct cpumask *mask, unsigned int action);
        void (*init_secondary)(void);
        void (*smp_finish)(void);
        void (*cpus_done)(void);
        void (*boot_secondary)(int cpu, struct task_struct *idle);
        void (*smp_setup)(void);
        void (*prepare_cpus)(unsigned int max_cpus);
    #ifdef CONFIG_HOTPLUG_CPU
        int (*cpu_disable)(void);
        void (*cpu_die)(unsigned int cpu);
    #endif
    };
```

小系统在上电过程中需要通过 register_smp_ops 将以上操作注册给内核。

send_ipi_single 和 send_ipi_mask 是核间中断的底层实现，由 SDK 完成，内核是不关心的，内核作 SMP 调度时会调用这两个接口。其它接口是与从核启动相关的接口，最后两个则用于 cpu 热拔插。

send_ipi_single 将中断发给指定的某个 cpu，而 send_ipi_mask 是根据 cpumask 将中断广播给一组 cpu，它实际是根据掩码逐一调用 send_ipi_single 完成的。

一般 IPI 只用于两个用途，通知其它 cpu 运行某个函数，或者通知其它 cpu 重新发起调度，这由 send_ipi_single 的 action 参数决定。要增加一个新的中断，除了要增加一个新的 irq 外，还需要增加一个 action 与之对应。

以 XLP 的 send_ipi_single 实现函数 nlm_send_ipi_single 为例，也就是添加下面的部分：
```
else if (action & IPI_DUMMY)
    ipi |= IRQ_IPI_DUMMY;
```

这部分与 SDK 实现有关，可参考具体实现代码来做，一般来说，SDK 会调用通过中断控制器向指定 cpu 产生一个 IRQ 为 IRQ_IPI_DUMMY 的中断（如果没有物理中断-IRQ映射关系的话）。

以上即为中断发送部分，接下来是处理工作。

收到 IPI 中断的 cpu，当然还是从中断流程开始，内核首先完成现场保护工作，便调用 plat_irq_dispatch 让小系统派发中断，当然这个是比 do_IRQ 更早的故事了。可以直接处理该中断，也可以调用 do_IRQ，如果是后者，这个 irq 当然是需要通过 request_irq 注册后才会得到处理，如果是前者，那就是 work on your own。

是不是很简单，那么如何触发这个 IPI 接口呢？只需要下面两行代码就可以了::
```
extern struct plat_smp_ops *mp_ops;
mp_ops->send_ipi_single(cpu, IPI_DUMMY);
```

mp_ops 是内核维护的一个全局结构指针，register_smp_ops 也就是向该指针赋值，在 smp-ops.h 里可以看到操作该变量的一些具体实现。

好了，代码加完了，跑跑看，什么？没收到中断？噢，你一定是忘了使能新增的 IRQ 了吧。记得在 arch_init_irq 添加代码来使能哦，亲！
