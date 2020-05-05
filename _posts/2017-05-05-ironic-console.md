---
layout: post
title: ironic console
author: kaifeng
date: 2017-05-05
categories: ironic
---

ironic 支持两种 console 方式：shellinabox 和 socat 方式。镜像必须是Linux操作系统，不支持Windows。shellinabox 是比较早的方式，它的限制在于只能在conductor上运行web服务，所以社区又用 socat 实现了一套。socat 提供 tcp 连接，可以和 nova 的 serial console 对接。

shellinabox 可以将终端输出转换成 Ajax 实现的 http 服务，可以直接通过浏览器访问，呈现出类似 terminal 的界面。socat 与 shellinabox 类似，都是充当一个管道作用，只不过 socat 是将终端流重定向到 tcp 连接，socat 本身功能还是很强大的。

要使用这两者，需要在conductor节点安装相应的工具，socat 在 yum 源里就可以找到，shellinabox不在标准源里，要从EPEL源下载，它没有外部依赖，直接下载rpm包安装即可。

以socat结尾的驱动是支持socat的，比如agent_ipmitool_socat（经典驱动已被hardware interface取代），其它的则是shellinabox方式。使用哪种终端方式，在创建节点时要确定好。
这两种方式都是双向的，可以查看终端输出，也可以键盘输入。串口数据通过IPMI的SOL(Serial Over Lan console)得到，串口数据首先通过socat工具转换成TCP连接，再由nova串口代理服务（即nova-Serialproxy）将TCP连接重定向为websocket连接，由web界面的websocket客户端使用。

## BMC配置

两者都是通过ipmi sol获取串口数据，因此需要打开裸机BIOS的串口重定向功能，配置串口波特率等，并将相同的配置写入 ironic 配置文件，以使生成对应的内核命令行，例如：
```
pxe_append_params = ... console=ttyS0,115200n8
```

## shellinabox

shellinabox 比较简单，在conductor节点安装 shellinabox，创建节点过程中，需要给 driver_info/ipmi_terminal_port 配置一个端口号，这个是自己定义的：
```
ironic node-update <node-uuid> add driver_info/ipmi_terminal_port=10000
```

使能节点的 console：
```
ironic node-set-console-mode <node-uuid> true
```

如果配置没有问题，可以获得 console 的连接信息：
```
ironic node-get-console <node-uuid>
```

比如 pxe_ssh 驱动，从 shellinabox 的服务进程可以看出，它只是将 virsh console 转换成了 web 服务：
```
    [root@localhost ~(keystone_admin)]# ps aux | grep shellinabox
    ironic   14318  0.0  0.0  42584  1724 ?        Ss   20:41   0:00 shellinaboxd -t -p 9002 --background=/tmp/1bba4b0d-bf68-4915-b39e-9e02e254cb93.pid -s /:991:988:HOME:virsh console baremetal
    ironic   14319  0.0  0.0  42340  1112 ?        S    20:41   0:00 shellinaboxd -t -p 9002 --background=/tmp/1bba4b0d-bf68-4915-b39e-9e02e254cb93.pid -s /:991:988:HOME:virsh console baremetal
    root     15571  0.0  0.0 112648  1008 pts/0    S+   21:07   0:00 grep --color=auto shellinabox
```

如果使用的是 pxe_ipmitool 驱动，它就是将 ipmitool sol activate 转换成 web 服务。

console 的连接地址使用的是配置文件里的 $my_ip，也要配置正确。

## socat

首先 ironic conductor 要安装 socat 包。
现在支持 socat 的只有 agent_ipmitool_socat 和 pxe_ipmitool_socat，也就是说底层是通过 ipmi 的串口重定向来实现的。

要支持串口重定向，需要配置 BIOS，找找 Console Serial Redirect 之类的配置，然后记录一下波特率，一般是 115200, no-parity, 8bit 这样的配置。
使用支持 socat 的驱动来创建节点，在 pxe_append_params 增加 console 参数，如：
```
pxe_append_params = ... console=ttyS0,115200n8
```

配置端口号：
```
ironic node-update <node-uuid> add driver_info/ipmi_terminal_port=<port>
```

port是本机端口，低的值一般被系统预留了，需要自己挑选合适的值，比如 10000-20000 之间。ironic尚不支持自动分配端口，主要是takeover的问题。

使能 console：

1. 检查配置是否完全：
```
ironic node-validate <node>
```

2. 打开 console：
```
ironic node-set-console-mode <node> true
```

3. 获得连接：
```
ironic node-get-console <node>
```

node-get-console 会打印出 socat 类型的连接信息，比如 tcp://ip:port 之类。

也可以用 socat 工具直接连上地址来看：socat - tcp:ip:port

### socat和nova对接

要在 nova 使用 serial console，需要在nova所在的控制节点安装 nova-serialproxy，对应 rpm 包为 openstack-nova-serialproxy。
并配置 nova 的 serial console 服务。检查配置文件 serial_console 部分，将 enabled 设为 true，port_range 根据情况配置，完成后重启 nova 服务。

部署实例后，通过nova可以得到实例的一个websocket类型的终端连接：
```
nova get-serial-console <instance-uuid>
```

dashboard 通过 nova 的 serialproxy 服务，在界面上呈现终端，如果 nova 配置没有问题，在实例的控制台就可以看到裸机的终端了。
