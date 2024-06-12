---
layout: post
title: ironic Queens/Rocky/Stein/Train 四个版本的主要新增功能
author: kaifeng
date: 2020-03-31
categories: ironic
---

## ironic特性

### rebuild的时候可以指定configdrive

此前在rebuild时，只能沿用原有的元数据，不能指定新的configdrive数据，如果重建改配参数，则不能生效。

此项因为涉及API版本变化，因此归到特性，实际上是一个bug。



### 增加ansible驱动

主要用在云下环境，定制性高，但playbook这些需要做很多定制工作，使用起来比较麻烦，另外不支持并发部署，所以性能不高。

目前使用不多，TripleO做了集成。


### 新增trait api

API支持配置/删除节点的trait，这是与nova调度对齐的一项功能，trait会通过nova compute的ironic驱动上报placement，nova使用trait与resource class进行调度。


### 支持rescue/unrescue节点

跨了两个迭代完成的功能，也是与虚机对齐的一项功能。

状态机增加了rescue相关的状态迁移，执行rescue时，ironic将裸机网络从租户网切换到rescue网，通过pxe下载rescue ramdisk，然后停在rescue状态，
rescue ramdisk是事先制作好的急救盘，打包一些恢复工具，等操作结束后，租户再通过unrescue结束操作，ironic会将裸机切回租户网。


### 新增healthcheck

社区目标，oslo中间件实现，ironic这边只是将中间件加入wsgi调用链。

它实现的功能和容器里的healthcheck是一样的。


### 增加inspect wait状态及支持abort操作

带内监控增加abort操作，以及为了支持abort引入了一个inspect wait状态。

之前如果发现失败，需要等默认30分钟超时才能再次触发主机发现，现在可以提前终止。


### 新增bios驱动接口

bios框架代码，需要厂商驱动支持（主要靠带外配置）。bios配置目前是嵌在clean流程中的，而且不作为一个clean step实现。

用户先要将目标bios配置写到node里，在执行clean操作时，执行bios配置。


### 新增redfish管理接口

通过redfish驱动设置和读取当前启动设备。


### 新增fault字段及电源自愈功能

node新增一个fault字段，对当前四种会进入xx failed的状态，会记录到fault字段。

电源自愈功能是基于该字段实现的，当fault字段的值为power failure，并且maintenance为true时，有独立的周期任务继续查询电源状态，
如果查询成功，将maintenance设为false，并清除fault字段。


### 支持SIGHUP

社区目标。旨在不重起服务可以改变一些服务配置，目前只支持少量的配置，如debug的开关。


### 支持conductor group分组

在conductor分组以前，node是通过哈希环均匀分配到conductor节点的，group分组相当于增加了一个逻辑层，

每个conductor服务可以配置一个组名，节点也可以指定自己的组名，组名为X的节点只会被分配的组名为X的conductor服务，

没有指定组名的节点只能由不指定组名的conductor服务。这个是与大型分布式环境（特别是边缘计算）相关的一个功能。


### 支持ramdisk驱动用于无盘裸机

ramdisk驱动主要用在无盘裸机上，基本上就是pxe跑起来就可以用，省去了写盘。这是欧洲某个科研机构要用的功能，
他们主要是用一些裸机跑科学计算，另外这个驱动也可以用来跑裸机的benchmark测试。


### 自动清理可以按节点使能

automated_clean是一个全局开关，要么全开，要么全关，这个功能可以配置单个节点是否使能自动清理，可以个性化定制。

也用来改进CI性能，同时保留功能可测。


### redfish oob主机发现

通过redfish协议进行带外主机发现，目前redfish这块社区主要是参照规范来实现，还是在打基础阶段，社区CI靠模拟器验证。
国外有一些厂商实现了redfish协议，但协议本身也在不断修改，这一块还不稳定。

### redfish bios配置管理

bios接口的实现，据社区的消息，redfish规范定义了很多，但基本上都是可选实现，所以硬件厂商是否做这些功能支持都是不确定的。


### 支持agent驱动从conductor节点下载镜像

针对agent驱动的功能，iscsi不受影响。需要conductor节点运行http服务，agent驱动从conductor节点下载镜像。


### 支持并行擦盘

简单实用的好功能，无需多言，需要升级IPA。


### 增加protected字段，保护节点

主要用于ironic standalone环境，与maintenance字段类似，node增加protected和protected_reason两个字段，
只有active/rescue两个状态下可以设置该字段，如果节点被保护了，则不能改变状态，比如删除节点，unrescue节点。

当nova管理实例时，最好不要设这个，否则nova会删实例失败。


### 可以获取注册到ironic的conductor服务

运维改进。环境有多个conductor的时候，可以查到每个conductor管理哪些节点，每个节点属于哪个conductor，方便找到对应主机定位问题。


### 增加description字段和owner字段

description是4k的文本字段，可以填一些节点相关的信息，比如服务器型号，序列号什么的，主要是考虑到provider硬件管理功能加进去的，

ironic不控制，和extra字段类似，不过支持内容搜索。

owner原本是一个描述性字段，给用户记录这个节点归谁所有，但是U版本在此字段上进行了新的特性用来支持多租户，此时owner填一个keystone user的uuid，

然后通过policy控制每个user只能看到属于自己的裸机资源。


### ironic支持json-rpc

ironic往轻量化发展的一个改进，ironic api/conductor可以使用json-rpc通信，这样就不需要消息队列服务。


### 电源同步周期任务可以并发执行

功能如其名，默认8协程。


### 增加allocation api

主要用于ironic standalone场景的功能，allocation也是一个资源，它代表一个节点的分配，创建一个allocation时，
ironic会从node池中寻求满足要求的node，条件可以是resource class, trait, 候选节点列表等等，找到后会把node与该allocation关联起来，
相当于节点被锁住，属于资源预留的功能。


### 增加deployment templates api

可以给节点设置deploy template，deploy template与deploy steps配合使用，用于定制部署过程中的部署操作，
但是这个功能太大，已经开发了几个周期了，U版本仍没有实现分离最大的deploy.deploy，
马上U版本冻结，也不太可能落地了，这个功能只能继续跟进，还没有达到实用的阶段。


### 允许控制是否可以删除available状态的节点

将节点带入available状态是有代价的，比如主机发现这些，加了一个配置，控制是否能直接把available状态的节点从ironic数据库中删除。


### 支持通过mdns发布ironic api

ironic启动一个mdns服务发布自己的api地址，原本由pxe内核参数带给ramdisk的callback地址可以不填了，让ramdisk的ipa自动发现。


### 支持smart nic

功能实现主要在neutron，还不知道使用场景，ironic在遇到smartnic的port时，是调用neutron接口处理的。


### 电源状态变化时通知nova

nova管理ironic时并不像管理虚机一样有准确的电源状态，真实的电源状态在ironic，nova呈现的instance电源状态与ironic看到的电源状态，

两者容易有不一致的情况。nova api实现了一个电源通知接口，ironic这边在电源状态改变时，通过nova client调用nova api，将电源状态更新到nova。

这个功能需要nova和ironic的电源控制方向相同，nova <-- ironic <-- bm。


### deploy_kernel, deploy_ramdisk, rescue_kernek, rescue_ramdisk支持全局配置

易用性改进。conductor全局配置ramdisk的uuid，但node上的ramdisk配置会优于全局配置，因此适合新装环境。

使用全局配置的好处是升级ramdisk时只需要修改配置并重启服务，不需要每个节点更新ramdisk uuid。


### 支持pxe自动重试

tftp传输不可靠，增加了一个参数，比如10分钟，如果10分钟还在deploy wait状态，认为pxe失败，让裸机重新pxe一次，可以设置重试次数。

如果整个时间超过了deploy wait的超时时间则进入deploy failed。


### 支持软raid，支持组合：raid1, raid1+raidN，N可以是0, 1, 1+0三种。

软raid问题还比较多，主要是uefi、bootloader这块还有问题，U版本还有一些bug在修复。另外需要更新IPA镜像，并且不兼容旧版本。


### fasttrack

从主机发现到部署都不重启裸机，用于快速交付。但是这个功能只做了概念验证，U版本PTL没有时间投入，所以没有进展，目前社区CI也没有覆盖。


## ironic-inspector特性


### api支持policy配置

ironic早就支持的功能，可以通过policy配置访问api的权限。


### 增加了pxe filter驱动层

框架改进。原基于firewall的filter实现为iptables pxe filter。firewall的filter是使用iptables进行防火墙管理。


## 新增dnsmasq pxe filter

在上述驱动接口之下实现的一套防火墙管理，通过inotify机制来独发dnsmasq配置更新，

基于mac地址的白名单/黑名单功能


### 增加manage_boot参数，控制inspector是否控制电源管理

inspector实际上也是通过ironic来管理裸机电源的，增加这个功能就是让inspector不要去控制电源，

这个功能单独没有什么用，需要结合U版本ironic的带内inspection功能使用。

U版本新增的功能是只需要做一次主机发现，ironic得到lldp数据后，后面的网络切换都可以由ironic完成。


### 增加ip_version配置，如果配为6，使用ip6tables管理防火墙

小功能，控制执行iptables还是ip6tables。inspector支持ipv6还是有问题的。


### 增加add-trait/remove-trait规则操作

增加了rule的操作。rules是满足一定条件的时候可以执行指定的操作（更新ironic节点的数据），比如ipmi地址之类的。
add-trait/remove-trait就是加删trait。


### 可以配置将introspection data存放在数据库

introspection data之前只能放在swift，现在可以存放在数据库，配置一下就可以了。


### 增加enable_mdns配置，允许inspector通过mdns发布api地址

与ironic的对应功能一样，通过mdns发布inspector的api地址。


### 可以通过API提供数据触发inspector进行数据处理

reapply是将已发现的数据重新处理一次更新到节点上，已发现的数据是inspector管理的，只能从主机发获得。

这个功能是允许用户手工提交发现数据，触发inspector进行数据处理并更新ironic节点数据。


### 允许节点在处于active/rescue状态时，接收introspection数据

一种另类用法，主机上驻留了IPA服务，管理员可以手工执行新增的inspect命令，触发一次主机数据收集，并发给inspector，再更新数据到ironic节点。


### 支持注册IPv6的BMC地址

目前还没有见到IPv6地址的BMC，有人贴了输出，就支持了，说到Vendor，那也是有很多坑的。


### 允许分离ironic-inspector-api和ironic-inspector-conductor

做了两个周期的功能，现在和ironic一样可以分为两个服务，社区CI有覆盖。

原本是要支持etcd的，但是tooz的etcd backend当时有bug，所以社区CI现在跑的是memcached。
