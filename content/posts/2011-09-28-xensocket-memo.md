---
layout: post
title: XenSocket备忘
author: kaifeng
date: 2011-09-28
categories: virt
---

# 编译

基于Xen 3.03，CentOS 5.3也是采用的Xen 3.03。需要安装kernel-xen-devel包，否则不能编译模块。

遇到编译错误：
```
    make -C /lib/modules/2.6.18-128.el5xen/build M=/home/voycat/xen/xensocket modules
    make[1]: Entering directory `/usr/src/kernels/2.6.18-128.el5-xen-i686'
      CC [M] /home/voycat/xen/xensocket/xensocket.o
    /home/voycat/xen/xensocket/xensocket.c: In function ‘server_allocate_event_channel’:
    /home/voycat/xen/xensocket/xensocket.c:395: warning: passing argument 1 of ‘HYPERVISOR_event_channel_op’ makes integer from pointer without a cast
    /home/voycat/xen/xensocket/xensocket.c:395: error: too few arguments to function ‘HYPERVISOR_event_channel_op’
    /home/voycat/xen/xensocket/xensocket.c: In function ‘client_bind_event_channel’:
    /home/voycat/xen/xensocket/xensocket.c:614: warning: passing argument 1 of ‘HYPERVISOR_event_channel_op’ makes integer from pointer without a cast
    /home/voycat/xen/xensocket/xensocket.c:614: error: too few arguments to function ‘HYPERVISOR_event_channel_op’
    make[2]: *** [/home/voycat/xen/xensocket/xensocket.o] Error 1
    make[1]: *** [_module_/home/voycat/xen/xensocket] Error 2
    make[1]: Leaving directory `/usr/src/kernels/2.6.18-128.el5-xen-i686'
    make: *** [all] Error 2
```

修改HYPERVISOR_event_channel_op的调用：
HYPERVISOR_event_channel_op(&op)为HYPERVISOR_event_channel_op(op.cmd, &op)
效果待验证。在dom0上同时跑receiver和sender，机器重启了。gref是-1应该是不对的。

XenSocket的代码还是3.0.2以前的，在这个版本后，Xen的事件通道接口改了，按照书上讲的接口改写了编译出错的位置，receiver可以得到看似正确的gref了。但在guest下运行sender，guest崩溃。

# XenSocket测试程序引起系统崩溃的定位

通过在xensocket里添加打印，发现是在client_bind_event_channel里产生了异常。以前在虚拟机上运行，崩溃时直接没有画面，无法查看异常打印，后面发现virt-manager的View菜单下，有个Serial Console可以打开虚拟机的终端，虚拟机崩溃后，终端信息仍然存在，对定位有很大帮助。

目前发现是访问入参提供的地址x->descriptor_addr时产生异常：
```
    bind_interdomain.remote_port = x->descriptor_addr->server_evtchn_port;
```

这个地址在client_map_descriptor_page分配，是通过grant reference的超级调用得到的，怀疑超级调用不对，或者调用入参不对，或者Xen版本问题？
继续跟踪，是HYPERVISOR_grant_table_op返回了-1，但罗斌看了一下xen源码，这个操作并没有返回-1的地方，怀疑调用是否进去了，需要修改xen的源码来跟踪。
GOOGLE了一下，发现该问题有记录：

> I've been looking through some back-end drivers and I notice that
grant tables are used to map domU memory into the privileged domain,
is the opposite possible? Can I map dom0 memory into a domU?
I've written code grants foreign access to a page from dom0 (take the
grant ref_id) then on domU I try to map it (basically the same way ex
blkback does it, or just any other driver for that matter) and all I
**get from HYPERVISOR_grant_table_op() is a -1 return value, op.status
is empty=0.**
I'll send more information on the specific error, if I at least know
that this can be done and I'm just not falling of a cliff. Also, if it
is possible, can you point me to some example code?
The problem you are facing is that the ** grant_operation_permitted() test (in the GNTTABOP_map_grant_ref case of the do_grant_table_op function in xen/common/grant_table.c) is failing. At present, only domains with iomem permissions can map grant references: typically, the only such domain is dom0. ** This is due to a TLB flush issue which was addressed in a recent patchset from Kieran Mansley. You can either try giving some dummy iomem privileges to your domU, or change the definition of grant_operation_permitted(d) to 1, so that it always succeeds.

在dom0上运行sender，domu上运行receiver，xensocket可以正常通讯了。
见：http://lists.xensource.com/archives/html/xen-devel/2007-07/msg00786.html

# 性能测试结果

单向通信: 6400000000 bytes, 237 sec (270Mbps)

receiver: 收

sender: 发

阻塞、超时保护、包长64字节

双向通信: 640000000 bytes, 140 sec (45.7Mbps)

receiver: 发->收

sender: 收->发

阻塞、超时保护、包长64字节

# 源码分析

## 实现机制

使用共享内存，对外呈现POSIX socket API接口。

单向通道，每个XenSocket的单向连接使用一个描述符页，涉及传输的domain对其进行读写映射。

## 代码分析

receiver通过bind系统调用进入xen_bind

调用xen_bind的进程设置为server

x-otherend_id设为receiver的入参，sender的domid。

依次调用:
* server_allocate_descriptor_page
* server_allocate_event_channel
* server_allocate_buffer_pages

xen_bind返回授权引用x-descriptor_gref，也即系统调用bind返回值。

server_allocate_descriptor_page:
* 给x->descriptor_addr分配了空间，由initialize_descriptor_page初始化为默认值。
* x->descriptor_gref通过gnttab_grant_forein_access调用得到
* x->descriptor_gref = gnttab_grant_foreign_access(x->otherend_id, virt_to_mfn(x->descriptor_addr), 0)
* descriptor_addr只是一个信息结构体，里面记了其它buffer的授权引用，这个处理在server_allocate_buffer_pages。

server_allocate_event_channel:
* 分配一个unbound事件通道，得到端口号
server_allocate_buffer_pages:
* 分配buffer空间给x->buffer_addr和x->buffer_grefs，分别用于记录PAGE地址和授权引用。
* 给每个buffer调用gnttab_grant_foreign_access得到授权引用
* 第一个授权引用赋给x->buffer_first_gref，然后给每个页面的头部写上下一个页面的授权引用，类似于一个链表，
* 客户端可以依次将所有页面映射到自己空间。目前buffer空间是5个page，如果空间是固定的，应该不需要这么麻烦了，以数组形式写在信息结构体里就可以了。

sender通过connect调用进入xen_connect，调用xen_connect的进程被调置为client，sender的两个入参被传入：domid和gref

依次调用:
* client_map_descriptor_page
* client_bind_event_channel
* client_map_buffer_pages

client_map_descriptor_page:
* 调用alloc_vm_area分配一个PAGE的页面(虚地址空间，无实地址对应)
* 通过HYPERVISOR_grant_table_op映射到刚才分配的虚地址空间
* 这样就可以访问server的那个信息结构体页面了

client_bind_event_channel:
* 绑定通道，通道号来自信息结构体

client_map_buffer_pages:
* 映射buffer到自己空间，仍然是通过alloc_vm_area分配虚址空间，HYPERVISOR_grant_table_op执行映射
* 映射方式与server端的处理保持一致，从第一个页面开始逐个映射。
send流程
* 用户态send进入xen_sendmsg
* 调用memcpy_fromiovecend将用户态数据拷贝到内核，拷贝地址为初始化时申请的共享buffer，
* 调用notify_remote_via_evtchn通知接收端
recv流程
* 用户态recv进入xen_recvmsg
* 调用local_memcpy_toiovecend将数据拷贝至用户态

# 改写的代码

## xensocket
```
    /* xensocket.c
    *
    * XVMSocket module for a shared-memory sockets transport for communications
    * between two domains on the same machine, under the Xen hypervisor.
    *
    * Authors: Xiaolan (Catherine) Zhang <cxzhang@us.ibm.com]] > 
    * Suzanne McIntosh <skranjac@us.ibm.com]] > 
    * John Griffin
    *
    * History:
    * Suzanne McIntosh 13-Aug-07 Initial open source version
    *
    * Copyright (c) 2007, IBM Corporation
    *
    */
    #include <linux/module.h>
    #include <linux/kernel.h>
    #include <linux/spinlock.h>
    #include <linux/timer.h>
    #include <linux/proc_fs.h>
    #include <net/sock.h>
    #include <net/tcp_states.h>
    #include <xen/driver_util.h>
    #include <xen/gnttab.h>
    #include <xen/evtchn.h>
    #include "xensocket.h"
    #define BUFF_PAGE_ORDER 1
    #define xen_sk(__sk) ((struct xen_sock *)__sk)
    #undef DEBUG
    #undef DEBUG_TRACE
    #ifdef DEBUG
    #define DPRINTK(x, args... ) printk(KERN_CRIT "%s: line %d: " x, __FUNCTION__ , __LINE__ , ## args );
    #else
    #define DPRINTK(x, args...)
    #endif
    #ifdef DEBUG_TRACE
    #define TRACE_ENTRY printk(KERN_CRIT "Entering %s\n", __func__)
    #define TRACE_EXIT printk(KERN_CRIT "Exiting %s\n", __func__)
    #else
    #define TRACE_ENTRY do {} while (0)
    #define TRACE_EXIT do {} while (0)
    #endif
    #define TRACE_ERROR printk(KERN_CRIT "Exiting (ERROR) %s\n", __func__)
    #define QUEUE_SIZE (PAGE_SIZE - sizeof(int)*2)
    struct buffer_ring
    {
         int txpos, rxpos; /* tx & rx position */
         unsigned char rb[QUEUE_SIZE]; /* ring buffer */
    };
    #define QUEUE_LEN(p) (((p)->txpos - (p)->rxpos + QUEUE_SIZE) % QUEUE_SIZE)
    #define QUEUE_AVAIL(p) (((p)->txpos + 1) % QUEUE_SIZE == (p)->rxpos) ? 0 : (QUEUE_SIZE - QUEUE_LEN(p) - 1)
    struct descriptor_page
    {
         uint32_t server_evtchn_port;
         int buffer_order;
         int buffer_gref[1<<BUFF_PAGE_ORDER]; /* buffer grant reference */
         unsigned int total_bytes_sent;
         unsigned int total_bytes_received;
         unsigned int sender_is_blocking;
         atomic_t sender_has_shutdown;
         atomic_t force_sender_shutdown;
    };
    struct xen_sock
    {
         struct sock sk;
         unsigned char is_server, is_client;
         domid_t otherend_id;
         struct descriptor_page *descriptor_addr; /* server and client */
         int descriptor_gref; /* server only */
         struct vm_struct *descriptor_area; /* client only */
         grant_handle_t descriptor_handle; /* client only */
         unsigned int evtchn_local_port;
         unsigned int irq;
         unsigned long buffer_addr; /* server and client */
         /* buffer_addr_tx and buffer_addr_rx is derived from buffer_addr */
         unsigned long buffer_addr_tx; /* buffer for data out */
         unsigned long buffer_addr_rx; /* buffer for data in */
         struct vm_struct *buffer_area; /* client */
         grant_handle_t *buffer_handles; /* client */
         int buffer_order;
    };
    static int xs_proc_read(char *page, char **start, off_t off, int count, int *eof, void *data)
    {
         int len = 0;
         off_t begin = 0;
         struct xen_sock *x = (struct xen_sock*)data;
         struct buffer_ring *tx_hdr = (struct buffer_ring *)(x->buffer_addr_tx);
         struct buffer_ring *rx_hdr = (struct buffer_ring *)(x->buffer_addr_rx);
        len += sprintf(page + len, "Tx: txpos=%d, rxpos=%d\n", tx_hdr->txpos, tx_hdr->rxpos);
        len += sprintf(page + len, "Rx: txpos=%d, rxpos=%d\n", rx_hdr->txpos, rx_hdr->rxpos);
         *eof = 1;
         *start = page + (off - begin);
         len -= (off - begin);
         if (len > count)
              len = count;
         if (len < 0)
              len = 0;
         return len;
    }
    static void initialize_xen_sock(struct xen_sock *x)
    {
         x->is_server = 0;
         x->is_client = 0;
         x->otherend_id = -1;
         x->descriptor_addr = NULL;
         x->descriptor_gref = -ENOSPC;
         x->descriptor_area = NULL;
         x->descriptor_handle = -1;
         x->evtchn_local_port = -1;
         x->irq = -1;
         x->buffer_addr = 0;
         x->buffer_area = NULL;
         x->buffer_handles = NULL;
         x->buffer_order = -1;
    }
    irqreturn_t client_interrupt (int irq, void *dev_id, struct pt_regs *regs);
    irqreturn_t server_interrupt (int irq, void *dev_id, struct pt_regs *regs);
    static struct proto xen_proto;
    static const struct proto_ops xen_stream_ops;
    static void initialize_descriptor_page(struct descriptor_page *d)
    {
         d->server_evtchn_port = -1;
         d->buffer_order = BUFF_PAGE_ORDER;
         d->total_bytes_sent = 0;
         d->total_bytes_received = 0;
         d->sender_is_blocking = 0;
         atomic_set(&d->sender_has_shutdown, 0);
         atomic_set(&d->force_sender_shutdown, 0);
    }
    static void init_buffer_page(unsigned long addr)
    {
         /* only server who provide buffers call this */
         struct buffer_ring *fifo = (struct buffer_ring *)addr;
         fifo->txpos = 0;
         fifo->rxpos = 0;
    }
    static inline int is_writeable (struct xen_sock *x)
    {
         int rc = 0;
         struct buffer_ring *tx_hdr = NULL;
         if(x == NULL)
         {
              TRACE_ERROR;
              return 0;
         }
         tx_hdr = (struct buffer_ring *)(x->buffer_addr_tx);
         if(QUEUE_AVAIL(tx_hdr) > 0) /* queue not full */
              rc = 1;
         else
              rc = 0;
         return rc;
    }
    static inline int is_readable (struct xen_sock *x)
    {
         int rc = 0;
         struct buffer_ring *rx_hdr = NULL;
         if(x == NULL)
         {
              TRACE_ERROR;
              return 0;
         }
         rx_hdr = (struct buffer_ring *)(x->buffer_addr_rx);
         if(QUEUE_LEN(rx_hdr) > 0) /* queue not empty */
              rc = 1;
         else
              rc = 0;
         return rc;
    }
    static long send_data_wait (struct sock *sk, long timeo)
    {
         struct xen_sock *x = xen_sk(sk);
         struct descriptor_page *d = x->descriptor_addr;
         DEFINE_WAIT(wait);
         TRACE_ENTRY;
         d->sender_is_blocking = 1;
         notify_remote_via_evtchn(x->evtchn_local_port);
         for (;;)
         {
              prepare_to_wait(sk->sk_sleep, &wait, TASK_INTERRUPTIBLE);
              if (is_writeable(x)
                  || !skb_queue_empty(&sk->sk_receive_queue)
                  || sk->sk_err
                  || (sk->sk_shutdown & RCV_SHUTDOWN)
                  || signal_pending(current)
                  || !timeo
                  || atomic_read(&d->force_sender_shutdown))
              {
                   break;
              }
              timeo = schedule_timeout(timeo);
         }
         d->sender_is_blocking = 0;
         finish_wait(sk->sk_sleep, &wait);
         TRACE_EXIT;
         return timeo;
    }
    static long receive_data_wait (struct sock *sk, long timeo)
    {
         struct xen_sock *x = xen_sk(sk);
         struct descriptor_page *d = x->descriptor_addr;
         DEFINE_WAIT(wait);
         TRACE_ENTRY;
         for (;;)
         {
              prepare_to_wait(sk->sk_sleep, &wait, TASK_INTERRUPTIBLE);
              if (is_readable(x)
                  || (atomic_read(&d->sender_has_shutdown) != 0)
                  || !skb_queue_empty(&sk->sk_receive_queue)
                  || sk->sk_err
                  || (sk->sk_shutdown & RCV_SHUTDOWN)
                  || signal_pending(current)
                  || !timeo)
              {
                   break;
              }
              timeo = schedule_timeout(timeo);
         }
         finish_wait(sk->sk_sleep, &wait);
         TRACE_EXIT;
         return timeo;
    }
    static void server_unallocate_buffer_pages (struct xen_sock *x)
    {
         struct descriptor_page *d = x->descriptor_addr;
           int buffer_num_pages = (1 << x->buffer_order);
         int i;
         for (i = 0; i < buffer_num_pages; i++)
         {
              if (d->buffer_gref[i] == -ENOSPC)
              {
                   continue;
              }
              gnttab_end_foreign_access(d->buffer_gref[i], 0, 0);
              d->buffer_gref[i] = -ENOSPC;
         }
         if (x->buffer_addr)
         {
              free_pages(x->buffer_addr, x->buffer_order);
              x->buffer_addr = 0;
              x->buffer_order = -1;
         }
    }
    static void server_unallocate_descriptor_page(struct xen_sock *x)
    {
         if(x->descriptor_gref != -ENOSPC)
         {
              gnttab_end_foreign_access(x->descriptor_gref, 0, 0);
              x->descriptor_gref = -ENOSPC;
         }
         if(x->descriptor_addr)
         {
              free_page((unsigned long)(x->descriptor_addr));
              x->descriptor_addr = NULL;
         }
    }
    static void client_unmap_buffer_pages (struct xen_sock *x)
    {
         if(x->buffer_handles)
         {
              struct descriptor_page *d = x->descriptor_addr;
              int buffer_order = d->buffer_order;
              int buffer_num_pages = (1 << buffer_order);
              int i;
              struct gnttab_unmap_grant_ref op;
              int rc = 0;
              for (i = 0; i < buffer_num_pages; i++)
              {
                   if (x->buffer_handles[i] == -1)
                   {
                        break;
                   }
                   memset(&op, 0, sizeof(op));
                   op.host_addr = x->buffer_addr + i * PAGE_SIZE;
                   op.handle = x->buffer_handles[i];
                   op.dev_bus_addr = 0;
                   lock_vm_area(x->buffer_area);
                   rc = HYPERVISOR_grant_table_op(GNTTABOP_unmap_grant_ref, &op, 1);
                   unlock_vm_area(x->buffer_area);
                   if (rc == -ENOSYS)
                   {
                        printk("Failure to unmap grant reference \n");
                   }
              }
              kfree(x->buffer_handles);
              x->buffer_handles = NULL;
         }
         if (x->buffer_area)
         {
              free_vm_area(x->buffer_area);
              x->buffer_area = NULL;
         }
    }
    static void client_unmap_descriptor_page (struct xen_sock *x)
    {
         int rc = 0;
         struct descriptor_page *d = NULL;
         d = x->descriptor_addr;
         if(x->descriptor_handle != -1)
         {
              struct gnttab_unmap_grant_ref op;
              memset(&op, 0, sizeof(op));
              op.host_addr = (unsigned long)x->descriptor_addr;
              op.handle = x->descriptor_handle;
              op.dev_bus_addr = 0;
              lock_vm_area(x->descriptor_area);
              atomic_set(&d->sender_has_shutdown, 1);
              rc = HYPERVISOR_grant_table_op(GNTTABOP_unmap_grant_ref, &op, 1);
              unlock_vm_area(x->descriptor_area);
              if (rc == -ENOSYS)
              {
                   printk("Failure to unmap grant reference for descriptor page\n");
              }
              x->descriptor_handle = -1;
         }
         if (x->descriptor_area)
         {
              free_vm_area(x->descriptor_area);
              x->descriptor_area = NULL;
         }
    }
    static int server_allocate_descriptor_page (struct xen_sock *x)
    {
         TRACE_ENTRY;
         if(x->descriptor_addr)
         {
              DPRINTK("error: already allocated server descriptor page\n");
              goto err;
         }
         if(!(x->descriptor_addr = (struct descriptor_page *)__get_free_page(GFP_KERNEL)))
         {
              DPRINTK("error: cannot allocate free page\n");
              goto err_unalloc;
         }
         initialize_descriptor_page(x->descriptor_addr);
         if((x->descriptor_gref = gnttab_grant_foreign_access(x->otherend_id, virt_to_mfn(x->descriptor_addr), 0)) == -ENOSPC)
         {
              DPRINTK("error: cannot share descriptor page %p\n", x->descriptor_addr);
              goto err_unalloc;
         }
         TRACE_EXIT;
         return 0;
    err_unalloc:
         server_unallocate_descriptor_page(x);
    err:
         TRACE_ERROR;
         return -ENOMEM;
    }
    static int server_allocate_event_channel (struct xen_sock *x)
    {
         struct evtchn_alloc_unbound alloc_unbound;
         int rc;
         TRACE_ENTRY;
         memset(&alloc_unbound, 0, sizeof(alloc_unbound));
         alloc_unbound.dom = DOMID_SELF;
         alloc_unbound.remote_dom = x->otherend_id;
         if ((rc = HYPERVISOR_event_channel_op(EVTCHNOP_alloc_unbound, &alloc_unbound)) != 0)
         {
              DPRINTK("Unable to allocate event channel\n");
              goto err;
         }
         x->evtchn_local_port = alloc_unbound.port;
         x->descriptor_addr->server_evtchn_port = x->evtchn_local_port;
         /* Next bind this end of the event channel to our local callback
         * function. */
         if ((rc = bind_evtchn_to_irqhandler(x->evtchn_local_port, server_interrupt, SA_SAMPLE_RANDOM, "xensocket", x)) <= 0)
         {
              DPRINTK("Unable to bind event channel to irqhandler\n");
              goto err;
         }
         TRACE_EXIT;
         return 0;
    err:
         TRACE_ERROR;
         return rc;
    }
    static int server_allocate_buffer_pages (struct xen_sock *x)
    {
         struct descriptor_page *d = x->descriptor_addr;
         int buffer_num_pages;
         int i;
         TRACE_ENTRY;
         if(!d)
         {
              /* must call server_allocate_descriptor_page first */
              DPRINTK("error: descriptor page not yet allocated\n");
              goto err;
         }
         if(x->buffer_addr)
         {
              DPRINTK("error: already allocated server buffer pages\n");
              goto err;
         }
         x->buffer_order = BUFF_PAGE_ORDER;
         buffer_num_pages = (1 << x->buffer_order);
         if (!(x->buffer_addr = __get_free_pages(GFP_KERNEL, x->buffer_order)))
         {
              DPRINTK("error: cannot allocate %d pages\n", buffer_num_pages);
              goto err;
         }
         /* set first page to tx fifo, second page to rx fifo
         * client is reversed.
         */
         x->buffer_addr_tx = x->buffer_addr;
         x->buffer_addr_rx = x->buffer_addr + PAGE_SIZE;
         /* init buffer page fifo */
         init_buffer_page(x->buffer_addr_tx);
         init_buffer_page(x->buffer_addr_rx);
         for(i = 0; i < buffer_num_pages; i++)
         {
              d->buffer_gref[i] = -ENOSPC;
              if((d->buffer_gref[i] = gnttab_grant_foreign_access(x->otherend_id, virt_to_mfn(x->buffer_addr + i * PAGE_SIZE), 0)) == -ENOSPC)
              {
                   DPRINTK("error: cannot share buffer page #%d\n", i);
                   goto err_unallocate;
              }
         }
         TRACE_EXIT;
         return 0;
    err_unallocate:
         server_unallocate_buffer_pages(x);
    err:
         TRACE_ERROR;
         return -ENOMEM;
    }
    static int client_map_descriptor_page(struct xen_sock *x)
    {
         struct gnttab_map_grant_ref op;
         int rc = -ENOMEM;
         TRACE_ENTRY;
         if(x->descriptor_addr)
         {
              DPRINTK("error: already allocated client descriptor page\n");
              goto err;
         }
         if((x->descriptor_area = alloc_vm_area(PAGE_SIZE)) == NULL)
         {
              DPRINTK("error: cannot allocate memory for descriptor page\n");
              goto err;
         }
         x->descriptor_addr = x->descriptor_area->addr;
         memset(&op, 0, sizeof(op));
         op.host_addr = (unsigned long)x->descriptor_addr;
         op.flags = GNTMAP_host_map;
         op.ref = x->descriptor_gref;
         op.dom = x->otherend_id;
         lock_vm_area(x->descriptor_area);
         rc = HYPERVISOR_grant_table_op(GNTTABOP_map_grant_ref, &op, 1);
         unlock_vm_area(x->descriptor_area);
         if(rc == -ENOSYS)
         {
              goto err_unmap;
         }
         if(op.status)
         {
              DPRINTK("error: grant table mapping operation failed\n");
              goto err_unmap;
         }
         x->descriptor_handle = op.handle;
         TRACE_EXIT;
         return 0;
    err_unmap:
         client_unmap_descriptor_page(x);
    err:
         TRACE_ERROR;
         return rc;
    }
    static int client_bind_event_channel(struct xen_sock *x)
    {
         struct evtchn_bind_interdomain bind_interdomain;
         int rc;
         TRACE_ENTRY;
         DPRINTK("client_bind_event_channel: x = %p\n", x);
         DPRINTK("client_bind_event_channel: x->otherend_id = %d\n", x->otherend_id);
         DPRINTK("client_bind_event_channel: x->descriptor_addr = %p\n", x->descriptor_addr);
         /* Start by binding this end of the event channel to the other
         * end of the event channel. */
         memset(&bind_interdomain, 0, sizeof(bind_interdomain));
         bind_interdomain.remote_dom = x->otherend_id;
         bind_interdomain.remote_port = x->descriptor_addr->server_evtchn_port;
         DPRINTK("client_bind_event_channel: remote_dom = %d, remote_port = %d\n",
                                                x->otherend_id,
                                                x->descriptor_addr->server_evtchn_port);
         if((rc = HYPERVISOR_event_channel_op(EVTCHNOP_bind_interdomain, &bind_interdomain)) != 0)
         {
              DPRINTK("Unable to bind to server's event channel\n");
              goto err;
         }
         x->evtchn_local_port = bind_interdomain.local_port;
         DPRINTK("client_bind_event_channel: evtchn_local_port = %d\n", bind_interdomain.local_port);
         DPRINTK("Other port is %d\n", x->descriptor_addr->server_evtchn_port);
         DPRINTK("My port is %d\n", bind_interdomain.local_port);
         /* Next bind this end of the event channel to our local callback
         * function. */
         if((rc = bind_evtchn_to_irqhandler(x->evtchn_local_port, client_interrupt, SA_SAMPLE_RANDOM, "xensocket", x)) <= 0)
         {
              DPRINTK("Unable to bind event channel to irqhandler\n");
              goto err;
         }
         x->irq = rc;
         TRACE_EXIT;
         return 0;
    err:
         TRACE_ERROR;
         return rc;
    }
    static int client_map_buffer_pages(struct xen_sock *x)
    {
         struct descriptor_page *d = x->descriptor_addr;
         int buffer_num_pages;
         int i;
         struct gnttab_map_grant_ref op;
         int rc = -ENOMEM;
         TRACE_ENTRY;
         if(!d)
         {
              /* must call client_map_descriptor_page first */
              DPRINTK("error: descriptor page not yet mapped\n");
              goto err;
         }
         if(x->buffer_area)
         {
              DPRINTK("error: already allocated client buffer pages\n");
              goto err;
         }
         if(d->buffer_order == -1)
         {
              DPRINTK("error: server has not yet allocated buffer pages\n");
              goto err;
         }
         x->buffer_order = d->buffer_order;
         buffer_num_pages = (1 << x->buffer_order);
         if(!(x->buffer_handles = kmalloc(buffer_num_pages * sizeof(grant_handle_t), GFP_KERNEL)))
         {
              DPRINTK("error: unexpected memory allocation failure\n");
              goto err;
         }
         else
         {
              for(i = 0; i < buffer_num_pages; i++)
              {
                   x->buffer_handles[i] = -1;
              }
         }
         if(!(x->buffer_area = alloc_vm_area(buffer_num_pages * PAGE_SIZE)))
         {
              DPRINTK("error: cannot allocate %d buffer pages\n", buffer_num_pages); 
              goto err_unmap;
         }
         x->buffer_addr = (unsigned long)x->buffer_area->addr;
         /* for client: set first page to rx fifo, second page to tx fifo
         */
         x->buffer_addr_tx = x->buffer_addr + PAGE_SIZE;
         x->buffer_addr_rx = x->buffer_addr;
         for(i = 0; i < buffer_num_pages; i++)
         {
              memset(&op, 0, sizeof(op));
              op.host_addr = x->buffer_addr + i * PAGE_SIZE;
              op.flags = GNTMAP_host_map;
              op.ref = d->buffer_gref[i];
              op.dom = x->otherend_id;
              lock_vm_area(x->buffer_area);
              rc = HYPERVISOR_grant_table_op(GNTTABOP_map_grant_ref, &op, 1); 
              unlock_vm_area(x->buffer_area);
              if(rc == -ENOSYS)
              {
                   goto err_unmap;
              }
              if(op.status)
              {
                   DPRINTK("error: grant table mapping failed\n");
                   goto err_unmap;
              }
              x->buffer_handles[i] = op.handle;
         }
         TRACE_EXIT;
         return 0;
    err_unmap:
         client_unmap_buffer_pages(x);
    err:
         TRACE_ERROR;
         return rc;
    }
    /* copied from kernel 2.6.32 */
    int memcpy_toiovecend(const struct iovec *iov, unsigned char *kdata, int offset, int len)
    {
         int copy;
         for (; len > 0; ++iov) {
              /* Skip over the finished iovecs */
              if (unlikely(offset >= iov->iov_len)) {
                   offset -= iov->iov_len;
                   continue;
              }
              copy = min_t(unsigned int, iov->iov_len - offset, len);
              if (copy_to_user(iov->iov_base + offset, kdata, copy))
                   return -EFAULT;
              offset = 0;
              kdata += copy;
              len -= copy;
         }
         return 0;
    }
    irqreturn_t client_interrupt (int irq, void *dev_id, struct pt_regs *regs)
    {
         struct xen_sock *x = dev_id;
         struct sock *sk = &x->sk;
         TRACE_ENTRY;
         if (sk->sk_sleep && waitqueue_active(sk->sk_sleep))
         {
              wake_up_interruptible(sk->sk_sleep);
         }
         TRACE_EXIT;
         return IRQ_HANDLED;
    }
    irqreturn_t server_interrupt (int irq, void *dev_id, struct pt_regs *regs)
    {
         struct xen_sock *x = dev_id;
         struct sock *sk = &x->sk;
         TRACE_ENTRY;
         if (sk->sk_sleep && waitqueue_active(sk->sk_sleep))
         {
              wake_up_interruptible(sk->sk_sleep);
         }
         TRACE_EXIT;
         return IRQ_HANDLED;
    }
    static int xen_create(struct socket *sock, int protocol)
    {
         int rc = 0;
         struct sock *sk = 0;
         TRACE_ENTRY;
         sock->state = SS_UNCONNECTED;
         switch(sock->type)
         {
              case SOCK_STREAM:
              {
                   sock->ops = &xen_stream_ops;
                   break;
              }
              default:
              {
                   rc = -ESOCKTNOSUPPORT;
                   goto out;
              }
         }
         sk = sk_alloc(PF_XEN, GFP_KERNEL, &xen_proto, 1);
         if(!sk)
         {
              rc = -ENOMEM;
              goto out;
         }
         sock_init_data(sock, sk);
         sk->sk_family = PF_XEN;
         sk->sk_protocol = protocol;
         initialize_xen_sock(xen_sk(sk));
    out:
         TRACE_EXIT;
         return rc;
    }
    static int xen_connect (struct socket *sock, struct sockaddr *uaddr, int addr_len, int flags)
    {
         int rc = -EINVAL;
         struct sock *sk = sock->sk;
         struct xen_sock *x = xen_sk(sk);
         struct sockaddr_xe *sxeaddr = (struct sockaddr_xe *)uaddr;
         struct proc_dir_entry *entry = 0;
         TRACE_ENTRY;
         if (sxeaddr->sxe_family != AF_XEN)
         {
              goto err;
         }
         /* Ensure that connect() is only called once for this socket.
         */
         if(x->is_client)
         {
              DPRINTK("error: cannot call connect() more than once on a socket\n");
              goto err;
         }
         if(x->is_server)
         {
              DPRINTK("error: cannot call both bind() and connect() on the same socket\n");
              goto err;
         }
         x->is_client = 1;
         x->otherend_id = sxeaddr->remote_domid;
         x->descriptor_gref = sxeaddr->shared_page_gref;
         if((rc = client_map_descriptor_page(x)) != 0)
         {
              goto err;
         }
         if((rc = client_bind_event_channel(x)) != 0)
         {
              goto err_unmap_descriptor;
         }
         if((rc = client_map_buffer_pages(x)) != 0)
         {
              goto err_unmap_buffer;
         }
         entry = create_proc_read_entry("xensocket", 0 /* def mode */ ,
                               0 /* parent */ ,
                               xs_proc_read /* proc read function */ ,
                               x /* no client data */);
         if(!entry)
         {
              printk("[%s]: Unable to create proc read entry for xensocket!\n",
                     __FUNCTION__);
         }
         TRACE_EXIT;
         return 0;
    err_unmap_buffer:
         client_unmap_buffer_pages(x);
    err_unmap_descriptor:
         client_unmap_descriptor_page(x);
         notify_remote_via_evtchn(x->evtchn_local_port);
    err:
      return rc;
    }
    static int xen_bind(struct socket *sock, struct sockaddr *uaddr, int addr_len)
    {
         int rc = -EINVAL;
         struct sock *sk = sock->sk;
         struct xen_sock *x = xen_sk(sk);
         struct sockaddr_xe *sxeaddr = (struct sockaddr_xe *)uaddr;
         struct proc_dir_entry *entry = 0;
         TRACE_ENTRY;
         if (sxeaddr->sxe_family != AF_XEN)
         {
              goto err;
         }
         /* Ensure that bind() is only called once for this socket.
         */
         if (x->is_server)
         {
              DPRINTK("error: cannot call bind() more than once on a socket\n");
              goto err;
         }
         if (x->is_client)
         {
              DPRINTK("error: cannot call both bind() and connect() on the same socket\n");
              goto err;
         }
         x->is_server = 1;
         x->otherend_id = sxeaddr->remote_domid;
         rc = server_allocate_descriptor_page(x);
         if(rc != 0)
              goto err;
         rc = server_allocate_event_channel(x);
         if(rc != 0)
              goto err;
         rc = server_allocate_buffer_pages(x);
         if(rc != 0)
              goto err;
         entry = create_proc_read_entry("xensocket", 0 /* def mode */ ,
                               0 /* parent */ ,
                               xs_proc_read /* proc read function */ ,
                               x /* no client data */);
         if(!entry)
         {
              printk("[%s]: Unable to create proc read entry for xensocket!\n",
                     __FUNCTION__);
         }
         /* A successful function exit returns the grant table reference. */
         TRACE_EXIT;
         return x->descriptor_gref;
    err:
         TRACE_ERROR;
         return rc;
    }
    static int xen_release(struct socket *sock)
    {
         struct sock *sk = sock->sk;
         struct xen_sock *x;
         struct descriptor_page *d;
         TRACE_ENTRY;
         if(!sk)
         {
              return 0;
         }
         sock->sk = NULL;
         x = xen_sk(sk);
         d = x->descriptor_addr;
         // if map didn't succeed, gracefully exit
         if (x->descriptor_handle == -1)
              goto out;
         if(x->is_server)
         {
              while(atomic_read(&d->sender_has_shutdown) == 0)
                   ;
              server_unallocate_buffer_pages(x);
              server_unallocate_descriptor_page(x);
         }
         if (x->is_client)
         {
              if ((atomic_read(&d->sender_has_shutdown)) == 0)
              {
                   client_unmap_buffer_pages(x);
                   client_unmap_descriptor_page(x);
                   notify_remote_via_evtchn(x->evtchn_local_port);
              }
              else
              {
                   printk(KERN_CRIT " xen_release: SENDER ALREADY SHUT DOWN!\n");
              }
         }
    out:
         sock_put(sk);
         TRACE_EXIT;
         return 0;
    }
    static int xen_shutdown(struct socket *sock, int how)
    {
         struct sock *sk = sock->sk;
         struct xen_sock *x;
         struct descriptor_page *d;
         x = xen_sk(sk);
         d = x->descriptor_addr;
         if (x->is_server)
         {
              atomic_set(&d->force_sender_shutdown, 1);
         }
         return xen_release(sock);
    }
    static int xen_sendmsg(struct kiocb *kiocb, struct socket *sock, struct msghdr *msg, size_t len)
    {
         int rc = -EINVAL;
         struct sock *sk = sock->sk;
         struct xen_sock *x = xen_sk(sk);
         struct buffer_ring *tx_hdr = (struct buffer_ring *)(x->buffer_addr_tx);
         unsigned int copied = 0;
         unsigned int bytes; /* bytes to be copied in a loop */
         unsigned int avail; /* available bytes in queue */
         long timeo;
         TRACE_ENTRY;
         timeo = sock_sndtimeo(sk, msg->msg_flags & MSG_DONTWAIT);
         while (copied < len)
         {
              bytes = len - copied;
              avail = QUEUE_AVAIL(tx_hdr);
              bytes = min(bytes, avail);
              /* sleep if no space left */
              if(bytes == 0)
              {
                   timeo = send_data_wait(sk, timeo);
                   if(!timeo)
                        break;
                   if (signal_pending(current))
                   {
                        rc = sock_intr_errno(timeo);
                        goto err;
                   }
                   continue;
              }
              if(tx_hdr->txpos + bytes > QUEUE_SIZE)
              {
                   unsigned int bytes_part1, bytes_part2;
                   /* require two steps to copy */
                   /* step one */
                   bytes_part1 = QUEUE_SIZE - tx_hdr->txpos;
                   bytes_part2 = bytes - bytes_part1;
                   if(memcpy_fromiovecend((unsigned char *)(tx_hdr->rb + tx_hdr->txpos),
                     msg->msg_iov, copied, bytes_part1) == -EFAULT) {
                        DPRINTK("error: copy_from_user failed\n");
                        goto err;
                   }
                   /* step two */
                   if(memcpy_fromiovecend((unsigned char *)(tx_hdr->rb),
                     msg->msg_iov, copied + bytes_part1, bytes_part2) == -EFAULT) {
                        DPRINTK("error: copy_from_user failed\n");
                        goto err;
                   }
              }
              else
              {
                   if(memcpy_fromiovecend((unsigned char *)(tx_hdr->rb + tx_hdr->txpos),
                     msg->msg_iov, copied, bytes) == -EFAULT) {
                        DPRINTK("error: copy_from_user failed\n");
                        goto err;
                   }
              }
              copied += bytes;
              tx_hdr->txpos = (tx_hdr->txpos + bytes) % QUEUE_SIZE;
         }
         notify_remote_via_evtchn(x->evtchn_local_port);
         TRACE_EXIT;
         return copied;
    err:
         TRACE_ERROR;
         return copied;
    }
    static int xen_recvmsg(struct kiocb *iocb, struct socket *sock, struct msghdr *msg, size_t size, int flags)
    {
         int rc = -EINVAL;
         struct sock *sk = sock->sk;
         struct xen_sock *x = xen_sk(sk);
         struct descriptor_page *d = x->descriptor_addr;
         struct buffer_ring *rx_hdr = (struct buffer_ring *)(x->buffer_addr_rx);
         int copied = 0;
         unsigned int bytes;
         unsigned int avail;
         long timeo;
         TRACE_ENTRY;
         timeo = sock_rcvtimeo(sk, flags & MSG_DONTWAIT);
         while(copied < size)
         {
              bytes = size - copied;
              avail = QUEUE_LEN(rx_hdr);
              bytes = min(bytes, avail);
              /* Block if the buffer is empty */
              if(bytes == 0)
              {
                   timeo = receive_data_wait(sk, timeo);
                   if(!timeo)
                        break;
                   if (signal_pending(current))
                   {
                        rc = sock_intr_errno(timeo);
                        DPRINTK("error: signal\n");
                        goto err;
                   }
                   continue;
              }
              if(rx_hdr->rxpos + bytes > QUEUE_SIZE)
              {
                   unsigned int bytes_part1, bytes_part2;
                   /* step one */
                   bytes_part1 = QUEUE_SIZE - rx_hdr->rxpos;
                   bytes_part2 = bytes - bytes_part1;
                   if (memcpy_toiovecend(msg->msg_iov, (unsigned char *)(rx_hdr->rb + rx_hdr->rxpos),
                                             copied, bytes_part1) == -EFAULT) {
                        DPRINTK("error: copy_to_user failed\n");
                        goto err;
                   }
                   /* step two */
                   if (memcpy_toiovecend(msg->msg_iov, (unsigned char *)(rx_hdr->rb),
                                             copied + bytes_part1, bytes_part2) == -EFAULT) {
                        DPRINTK("error: copy_to_user failed\n");
                        goto err;
                   }
              }
              else
              {
                   /* no wrap around, proceed with one copy */
                   if (memcpy_toiovecend(msg->msg_iov, (unsigned char *)(rx_hdr->rb + rx_hdr->rxpos),
                                             copied, bytes) == -EFAULT) {
                        DPRINTK("error: copy_to_user failed\n");
                        goto err;
                   }
              }
              copied += bytes;
              rx_hdr->rxpos = (rx_hdr->rxpos + bytes) % QUEUE_SIZE;
              if (d->sender_is_blocking)
              {
                   notify_remote_via_evtchn(x->evtchn_local_port);
              }
         }
         TRACE_EXIT;
         return copied;
    err:
         TRACE_ERROR;
         return copied;
    }
    static struct net_proto_family xen_family_ops = {
         .family = AF_XEN,
         .create = xen_create,
         .owner = THIS_MODULE,
    };
    static struct proto xen_proto = {
         .name = "XEN",
         .owner = THIS_MODULE,
         .obj_size = sizeof(struct xen_sock),
    };
    static const struct proto_ops xen_stream_ops = {
         .family = AF_XEN,
         .owner = THIS_MODULE,
         .release = xen_release,
         .bind = xen_bind,
         .connect = xen_connect,
         .socketpair = sock_no_socketpair,
         .accept = sock_no_accept,
         .getname = sock_no_getname,
         .poll = sock_no_poll,
         .ioctl = sock_no_ioctl,
         .listen = sock_no_listen,
         .shutdown = xen_shutdown,
         .getsockopt = sock_no_getsockopt,
         .setsockopt = sock_no_setsockopt,
         .sendmsg = xen_sendmsg,
         .recvmsg = xen_recvmsg,
         .mmap = sock_no_mmap,
         .sendpage = sock_no_sendpage,
    };
    static int __init xensocket_init (void)
    {
         int rc = -1;
         TRACE_ENTRY;
         rc = proto_register(&xen_proto, 1);
         if (rc != 0)
         {
              printk(KERN_CRIT "%s: Cannot create xen_sock SLAB cache!\n", __FUNCTION__);
              goto out;
         }
         sock_register(&xen_family_ops);
    out:
         TRACE_EXIT;
         return rc;
    }
    static void __exit xensocket_exit (void)
    {
         TRACE_ENTRY;
         sock_unregister(AF_XEN);
         proto_unregister(&xen_proto);
         TRACE_EXIT;
    }
    module_init(xensocket_init);
    module_exit(xensocket_exit);
    MODULE_LICENSE("GPL");
```

## sender

```
    /* sender.c
    *
    * Sender side test software for testing XenSocket.
    *
    * Authors: Xiaolan (Catherine) Zhang <cxzhang@us.ibm.com]] > 
    * Suzanne McIntosh <skranjac@us.ibm.com]] > 
    * John Griffin
    *
    * History:
    * Suzanne McIntosh 13-Aug-07 Initial open source version
    *
    * Copyright (c) 2007, IBM Corporation
    *
    */
    #include <stddef.h>
    #include <stdio.h>
    #include <errno.h>
    #include <stdlib.h>
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <sys/un.h>
    #include <unistd.h>
    #include "../xensocket.h"
    #define PKTLEN 64
    int main(int argc, char **argv)
    {
         struct sockaddr_xe sxeaddr;
         struct timeval tv;
         long long i;
         long long total_bytes = 0;
         int sock;
         int rc = 0;
         int loop;
         unsigned char data[PKTLEN];
         if(argc > 4)
         {
              printf("Usage: %s <peer-domid> <grant-ref>\n", argv[0]);
              return -1;
         }
         sxeaddr.sxe_family = AF_XEN;
         sxeaddr.remote_domid = atoi(argv[1]);
         printf("domid = %d\n", sxeaddr.remote_domid);
         sxeaddr.shared_page_gref = atoi(argv[2]);
         printf("gref = %d\n", sxeaddr.shared_page_gref);
         if(argc == 4)
              loop = atoi(argv[3]);
         else
              loop = 10000000;
         printf("loop = %d\n", loop);
         /* Create the socket. */
         sock = socket (21, SOCK_STREAM, XS_PROT_STREAM);
         if (sock < 0) {
              errno = ENOTRECOVERABLE;
              perror ("socket");
              exit (EXIT_FAILURE);
         }
         tv.tv_sec = 5;
         tv.tv_usec = 0;
         setsockopt(sock, SOL_SOCKET, SO_SNDTIMEO, &tv, sizeof(tv));
         rc = connect (sock, (struct sockaddr *)&sxeaddr, sizeof(sxeaddr));
         if(rc < 0)
         {
              printf ("connect failed\n");
              exit(1);
         }
         printf("Sending...\n");
         fflush(stdout);
         for(i = 0; i < loop; i++)
         {
              rc = send(sock, data, PKTLEN, 0);
              if (rc < 0) {
                 printf("send returned error = %i\n", rc);
                 errno = ENOTRECOVERABLE;
                 perror ("send");
                 rc = 1;
                 break;
            }
            if(rc == 0)
                 printf("no space timeout, i=%lld\n", i);
            if(rc > PKTLEN)
                 printf("invalid packet size %d\n", rc);
            total_bytes += rc;
         }
         printf("total bytes: %lld\n", total_bytes);
         do
         {
              printf("cmd:\r");
         }
         while(getchar() != 'q');
         return rc;
    }
```

## receiver

```
    /* receiver.c
    *
    * Receiver side test software for testing XenSockets.
    *
    * Authors: Xiaolan (Catherine) Zhang <cxzhang@us.ibm.com]] > 
    * Suzanne McIntosh <skranjac@us.ibm.com]] > 
    * John Griffin
    *
    * History:
    * Suzanne McIntosh 13-Aug-07 Initial open source version
    *
    * Copyright (c) 2007, IBM Corporation
    *
    */
    #include <stddef.h>
    #include <stdio.h>
    #include <errno.h>
    #include <stdlib.h>
    #include <sys/types.h>
    #include <sys/socket.h>
    #include <sys/un.h>
    #include <unistd.h>
    #include <time.h>
    #include "../xensocket.h"
    #define PKTLEN 64
    int main (int argc, char **argv)
    {
         int sock;
         struct sockaddr_xe sxeaddr;
         int rc = 0;
         int gref;
         long long i = 0;
         unsigned char data[PKTLEN];
         time_t t_before, t_after;
         int loop;
         long long total_bytes = 0;
         struct timeval tv;
         if(argc > 4)
         {
              printf("Usage: %s <peer-domid>\n", argv[0]);
              return -1;
         }
         sxeaddr.sxe_family = AF_XEN;
         sxeaddr.remote_domid = atoi(argv[1]);
         if(argc == 3)
              loop = atoi(argv[2]);
         else
              loop = 10000000;
         printf("domid = %d, loop = %d\n", sxeaddr.remote_domid, loop);
         /* Create the socket. */
         sock = socket (21, SOCK_STREAM, XS_PROT_STREAM);
         if(sock < 0)
         {
              errno = -ENOTRECOVERABLE;
              perror ("socket");
              exit (EXIT_FAILURE);
         }
         tv.tv_sec = 5;
         tv.tv_usec = 0;
         setsockopt(sock, SOL_SOCKET, SO_RCVTIMEO, &tv, sizeof(tv));
         gref = bind(sock, (struct sockaddr *)&sxeaddr, sizeof(sxeaddr));
         printf("gref = %d\n", gref);
         printf("press any key to continue...\n");
         getchar();
         printf("begin...\n");
         t_before = time(NULL);
         for(i = 0; i < loop; i++)
         {
              rc = recv(sock, data, PKTLEN, 0);
              if (rc < 0) {
                   shutdown(sock, SHUT_RDWR);
                   printf("recv returned error = %i\n", rc);
                   errno = ENOTRECOVERABLE;
                   perror ("recv");
                   rc = 1;
                   break;
             }
             if(rc == 0)
                  printf("no data, i = %lld\n", i);
             total_bytes += rc;
         }
         t_after = time(NULL);
         printf("end. time = %lu (%lu ~ %lu)\n", t_after - t_before, t_before, t_after);
         printf("total bytes: %lld\n", total_bytes);
         do
         {
              printf("cmd:\r");
         }
         while(getchar() != 'q');
         return rc;
    }
```
