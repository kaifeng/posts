---
layout: post
title: ironic与多租户网络
author: kaifeng
date: 2017-04-06
categories: ironic
---

云计算是IT技术不断发展的产物，OpenStack 是 NASA 和 RackSpace 联合项目的发展物，目前已经是 IaaS 云的事实标准，它从仅有的 swift 和 nova 两个项目不断地发展为今天所看到的规模，并衍生出了大大小小的项目。

云计算的快速发展，离不开用户对资源的需求和管理。IT 系统的三大类资源：计算、存储和网络，在云平台被抽象为资源池，用户请求资源，云平台提供资源，用户只需定义资源规格，不再关心底层是如何实现和管理的。

ironic 是 OpenStack 众多项目中的一个，其功能是裸金属部署。相对于通过 libvirt 将镜像放入虚机，ironic 直接将镜像写入裸金属的硬盘，我们是玩真的！ ironic 可以独立使用，比较像 IT 部门批量安装操作系统，可是部署完了怎么用是个问题，所以还是和 OpenStack 其它组件结合起来更具有使用价值。然而 ironic 直接涉及物理机和物理网络，使得网络环境变得更加复杂。

## ironic 简介

与 ironic 关系最密切的组件是 nova, neutron 及 glance。glance 为 ironic 提供映像服务，neutron 提供网络服务。
nova 通过 ironic 以与虚机相同的界面形式为用户提供裸金属上的实例管理，从 nova 的角度，ironic 相当于 Hypervisor。

ironic 也采用 api, conductor 及 driver 分层结构。ironic api 对外暴露接口，在部署过程中，conductor 与其它组件交互，并通过底层驱动完成部署。目前常用的驱动是 pxe_ipmitool 和 agent_ipmitool，它们是涵盖了电源、引导、部署、终端等驱动的集合名称，其中与部署功能最相关的是电源、引导和部署驱动。两者电源都使用 IPMI 驱动，引导使用 PXE 驱动，区别主要在于部署驱动不同，pxe_ipmitool 使用 ISCSIDeploy，agent_ipmitool 使用 AgentDeploy，两者稍有不同，后面会讲到。

我们首先来了解 ironic 部署裸机的过程，所涉及的网络交互，然后再来看看 ironic 为了支持 VLAN 网络所做的工作。

## ironic如何部署裸机

部署裸机也就是在裸机上创建实例的过程，nova 调度器会根据过滤器挑选符合条件的裸机节点进行部署，这一过程与虚机是相同的。
不同之处在于，裸金属上的实例是独占资源，因此，flavor 的配置必须与裸机的规格完全相同，不支持超配。

选中裸机节点后，nova 将 instance 信息传给下面的驱动，对于使用 ironic 驱动的计算节点，自然是调用到 virt 下的 ironic 驱动。
ironic 驱动向 ironic 服务的对外接口发送部署请求。ironic api 收到请求，通过 rpc 转给 conductor 处理。
conductor 主要是流程实现和节点层面的管理，具体操作则交给底层驱动完成操作。

conductor 依次调用 deploy.prepare 和 deploy.deploy 配置 PXE启动环境，下载部署映像和用户映像，通过 IPMI 驱动设置裸机下次启动方式为 PXE，然后重启裸机。整个部署过程可以分为两个阶段，第一个阶段的流程在这里就结束了。

创建虚机实例，只需要用户映像就可以了，但裸机上并没有运行任何服务，ironic 又如何将用户映像写入裸机的硬盘？答案就是通过部署映像。
部署映像其实是一套定制过的 Linux 内核和文件系统，里面包含了一个 python 实现的 agent 服务，正式的名称为 ironic python agent (IPA)，是 ironic 的关联项目。

裸机重启后以PXE方式启动，第二阶段就此开始，按照事先准备好的环境，裸机通过 DHCP 服务得到 ironic conductor 节点的地址，
通过 tftp 服务下载部署映像并引导，一个麻雀虽小五脏俱全的小 Linux 系统便跑起来了，agent 服务启动。ironic api 的地址通过内核参数传给了部署映像，因此 agent 起来后，会向 ironic api 发送请求，在请求到来之前，节点处于 wait call-back 状态。

pxe_impitool 和 agent_ipmitool 的 deploy 驱动 ISCSIDeploy 和 AgentDeploy 的主要差异就从这里开始，不同的驱动所构造的 PXE 启动环境，使 IPA 在裸机上启动后执行了不同的流程。简单地说，使用 pxe_ipmitool 时，IPA 将裸机的硬盘通过 iscsi 协议暴露给 conductor，由 conductor 发起写盘操作，而使用 agent_ipmitool 时，IPA是主动下载映像执行写盘操作，两者访问 ironic api 的接口也不同。

我们先看使用 pxe_ipmitool 的情况，IPA 开始运行后，向 ironic api 的 /v1/vendor_passthrough 发送请求并将 iscsi target设备号传过来，请求转给 conductor 处理，节点由 wait call-back 状态转为 deploying 状态，conductor 执行 iscsi 连接和设置，如果用户映像是 raw 格式的，直接用 dd 写盘，如果是其它格式，则用 qemu-img 转换并直接输出到目标盘。用户映像就这样被写到裸机硬盘上了。如果是设置为本地启动，安装 bootloader，然后清理 PXE 启动环境，重启裸机，实例就这样跑起来了。

agent_ipmitool 的情况差不多，只是请求变成了 lookup，主动获取一些部署信息，然后自行完成写盘，IPA 与 ironic 保持心跳及交换数据。
在 Mitaka 版本还是通过 driver passthrough 透传给驱动实现的，至 Ocata 版本已经独立为 /v1/lookup 接口了。agent 是社区推荐的方式，有可能会取代 pxe 驱动组，真是一代新人胜旧人哪。

## 扁平网络下的部署

从部署流程可以看到，整个过程在逻辑上涉及到两个网络，一个是用于 PXE 启动、部署映像下载和执行写盘操作的部署网络，一个是部署完成后裸机所在的租户网络。对于扁平网络来说，这两个网络只能是同一个网络，也就是 nova boot 指定的网络。

Mitaka 版本要简单得多，创建实例时，nova 分配包括网络在内的资源，创建逻辑端口，最终进入 ironic 驱动的 spawn 入口，分配的 vif 列表以 network_info 传给驱动。ironic 的处理也很简单，就是 `_plug_vifs`，先把代码简化一下：

```python
    def _plug_vifs(self, node, instance, network_info):
        ports = self.ironicclient.call("node.list_ports", node.uuid)
        if len(network_info) > 0:
            for vif in network_info:
                for pif in ports:
                    if vif['address'] == pif.address:
                        # attach what neutron needs directly to the port
                        port_id = six.text_type(vif['id'])
                        patch = [{'op': 'add',
                                  'path': '/extra/vif_port_id',
                                  'value': port_id}]
                        self.ironicclient.call("port.update", pif.uuid, patch)
                        break
```

首先通过 ironic api 得到裸机的物理端口列表，然后把分配的 vif 信息更新到 ironic 的 port 表，用 vif_port_id 来维护 port 与 vif 的关系。
因为在 M 版本里，分配网络资源使用的 MAC 地址是由驱动的 macs_for_instance 提供的，这个网络从一开始就是连通的。
记录的 vif_port_id 只有ilo 和 irmc 驱动在virtual media boot 流程使用，属于厂商驱动的功能，大概类似于远程传递一个虚拟光驱过去之类的功能吧，
反正跟我们要讲的通用的 PXE 驱动组不一样，就不在此分析了。

## VLAN 网络支持

仅支持扁平网络限制了 ironic 的实际应用，扁平网络意味着没有隔离功能，无法支持多租户使用，也不能将 ironic 部署所需的辅助网络区分开来。
在 neutron 所支持的几种网络类型里，最基本的支持隔离的网络就是 VLAN，ironic 从 N 版本开始并最终在 Ocata 版本完成了 VLAN 网络支持。

VLAN 通过报文内的 tag 实现隔离，交换机端口可配置为 access 或 trunk 口，进出 access 口的报文被打上或剥掉 tag 标记，
交换机只会将报文转发到具有相同 tag 的端口，因此隔离了不同 tag 的报文，从而产生了逻辑上隔离的网络。
trunk 口允许指定 tag 范围通过端口，不改变报文，用来汇聚或者级联。

支持 VLAN 网络后，ironic 具备了多租户的支持能力，也做到了将部署网络和租户网络分离开来。在 VLAN 环境下，ironic 的网络在逻辑上可以分为部署网络、清理网络和不同租户的网络，我们在分析时可以把环境简化为一个部署网络和一个租户网络来讨论，道理是一样的。

nova 创建实例所携带的网络信息是租户网络的信息，nova 显然看不到 ironic 的部署网络，因此 ironic 在部署时，必然要经过网络的切换过程：先将裸机切换到部署网络，部署完成后再将裸机切换回租户网络。这需要解决两个问题，如何动态地告知 neutron 来更新网络，以及如何控制交换机完成 VLAN 的切换，这正是 Ironic Neutron Integration spec 所要实现的事情。

### 网络初始化

我们来分析 Ocata 的代码，还是从nova 驱动的 spawn 开始，与网络相关的工作仍然在 _plug_vifs，简化代码如下：

```python
    def _plug_vifs(self, node, instance, network_info):
        for vif in network_info:
            port_id = six.text_type(vif['id'])
            try:
                self.ironicclient.call("node.vif_attach", node.uuid, port_id,
                                       retry_on_conflict=False)
            except ironic.exc.BadRequest as e:
                msg = (_("Cannot attach VIF %(vif)s to the node %(node)s due "
                         "to error: %(err)s") % {'vif': port_id,
                                                 'node': node.uuid, 'err': e})
                LOG.error(msg)
                raise exception.VirtualInterfacePlugException(msg)
            except ironic.exc.Conflict:
                # NOTE (vsaienko) Pass since VIF already attached.
                pass
```

这里通过 vif_attach 接口，将分配给实例的所有逻辑端口 id 注册到了 ironic 节点，具体的工作则通过 api 接口 `/v1/nodes/{node_ident}/vifs`，
由 conductor 调用 ironic 网络驱动的 vif_attach 处理端口绑定。在 O 版本里，分配 vif 的 MAC 地址不再从驱动获取，因此所分配的逻辑端口 vif 是没有 MAC 地址的。

vif_attach从 ironic port 表得到mac地址，调用 update_port_address 更新vif的MAC地址，然后把 port id 保存下来留待后面使用，我们来看 update_port_address 代码：
```python
    def update_port_address(port_id, address):
        client = get_client()
        port_req_body = {'port': {'mac_address': address}}
        try:
            msg = (_("Failed to get the current binding on Neutron "
                     "port %s.") % port_id)
            port = client.show_port(port_id).get('port', {})
            binding_host_id = port.get('binding:host_id')
            binding_profile = port.get('binding:profile')
            if binding_host_id:
                # Unbind port before we update it's mac address, because you can't
                # change a bound port's mac address.
                msg = (_("Failed to remove the current binding from "
                         "Neutron port %s, while updating its MAC "
                         "address.") % port_id)
                unbind_neutron_port(port_id, client=client)
                port_req_body['port']['binding:host_id'] = binding_host_id
                port_req_body['port']['binding:profile'] = binding_profile
            msg = (_("Failed to update MAC address on Neutron port %s.") % port_id)
            client.update_port(port_id, port_req_body)
        except (neutron_exceptions.NeutronClientException, exception.NetworkError):
            LOG.exception(msg)
            raise exception.FailedToUpdateMacOnPort(port_id=port_id)
```

update_port_address 首先通过port id反查neutron关于port的完整信息，更新 MAC 地址字段，再通过 neutron 接口更新 vif 信息。
binding:host_id 是 neutron 记录 vif 绑定在哪个节点上的信息，对于 ironic 来说，先进入部署网络，再进入租户网络，
因此绑定流程被推迟到部署完成，要进入租户网络的时候，我们后面会看到。binding:profile 则是让 neutron 得以控制交换机，
进行 VLAN 网络切换的必要信息，对于新实例的创建，这两类信息此时都是没有的，函数的作用就很直接了，把裸机的 MAC 地址更新到 vif。

### 进入部署网络

ironic 在 deploy.prepare 阶段将裸机切换到部署网络，以 ISCSIDeploy 为例：

```python
    def prepare(self, task):
        node = task.node
        if node.provision_state in [states.ACTIVE, states.ADOPTING]:
            task.driver.boot.prepare_instance(task)
        else:
            if node.provision_state == states.DEPLOYING:
                # Adding the node to provisioning network so that the dhcp
                # options get added for the provisioning port.
                manager_utils.node_power_action(task, states.POWER_OFF)
                # NOTE(vdrok): in case of rebuild, we have tenant network
                # already configured, unbind tenant ports if present
                task.driver.network.unconfigure_tenant_networks(task)
                task.driver.network.add_provisioning_network(task)
            deploy_opts = deploy_utils.build_agent_options(node)
            task.driver.boot.prepare_ramdisk(task, deploy_opts)
```

从字面意思不难理解，这里先将裸机脱离租户网络，然后加入部署网络，我们来看看具体怎么做：
```python
    def unconfigure_tenant_networks(self, task):
        node = task.node
        LOG.info(_LI('Unbinding instance ports from node %s'), node.uuid)
        ports = [p for p in task.ports if not p.portgroup_id]
        portgroups = task.portgroups
        for port_like_obj in ports + portgroups:
            vif_port_id = (
                port_like_obj.internal_info.get(common.TENANT_VIF_KEY) or
                port_like_obj.extra.get('vif_port_id'))
            if not vif_port_id:
                continue
            neutron.unbind_neutron_port(vif_port_id)
```

unconfigure_tenant_networks 遍历裸机所有的 port 和 portgroup，依次调用 unbind_neutron_port 解除绑定关系。
在 vif_attach 中，我们已经看到 ironic 的 port 记录了与之对应的 vif id，这里就是通过它来操作 neutron。
portgroup 是正在实现的链路聚合功能所引入的概念，在此可以先忽略：

```python
    def unbind_neutron_port(port_id, client=None):
        if not client:
            client = get_client()
        body = {'port': {'binding:host_id': '',
                         'binding:profile': {}}}
        try:
            client.update_port(port_id, body)
        except neutron_exceptions.NeutronClientException as e:
            msg = (_('Unable to clear binding profile for '
                     'neutron port %(port_id)s. Error: '
                     '%(err)s') % {'port_id': port_id, 'err': e})
            LOG.exception(msg)
            raise exception.NetworkError(msg)
```

unbind_neutron_port 清理掉对应租户网络的 vif 绑定信息，避免同一个 MAC 地址同时出现在两个网络。

接下来就要将裸机加入部署网络了，因为 vif 由 ironic 自己创建管理，相对就简单一些。add_provisioning_network  通过 neutron 为每个打开 PXE 的口调用 neutron 创建 vif，包含binding:host_id, MAC 地址及binding:profile等完整信息，neutron 根据配置调度 agent 实施网络变化：
```python
    def add_provisioning_network(self, task):
        # If we have left over ports from a previous provision attempt, remove
        # them
        neutron.rollback_ports(task, self.get_provisioning_network_uuid())
        LOG.info(_LI('Adding provisioning network to node %s'),
                 task.node.uuid)
        vifs = neutron.add_ports_to_network(
            task, self.get_provisioning_network_uuid(),
            security_groups=CONF.neutron.provisioning_network_security_groups)
        for port in task.ports:
            if port.uuid in vifs:
                internal_info = port.internal_info
                internal_info['provisioning_vif_port_id'] = vifs[port.uuid]
                port.internal_info = internal_info
                port.save()
```

add_ports_to_network 还是有点长的，尽量简化一下：
```python
    def add_ports_to_network(task, network_uuid, security_groups=None):
        client = get_client()
        node = task.node
        body = {
            'port': {
                'network_id': network_uuid,
                'admin_state_up': True,
                'binding:vnic_type': 'baremetal',
                'device_owner': 'baremetal:none',
            }
        }
        body['port']['binding:host_id'] = node.uuid
        ports = {}
        failures = []
        portmap = get_node_portmap(task)
        pxe_enabled_ports = [p for p in task.ports if p.pxe_enabled]
        for ironic_port in pxe_enabled_ports:
            body['port']['mac_address'] = ironic_port.address
            binding_profile = {'local_link_information':
                               [portmap[ironic_port.uuid]]}
            body['port']['binding:profile'] = binding_profile
            client_id = ironic_port.extra.get('client-id')
            try:
                port = client.create_port(body)
            except neutron_exceptions.NeutronClientException as e:
                failures.append(ironic_port.uuid)
            else:
                ports[ironic_port.uuid] = port['port']['id']
        return ports
```

portmap 是 port 的 local_link_connection 信息，包含端口所连接的交换端口和交换 id 标志，neutron 使用这些信息来控制交换机对应的端口。

### 进入租户网络

裸机部署完成后，要离开部署网络，加入租户网络，这个工作是在deploy.reboot_and_finish_deploy 里完成的：
```python
    task.driver.network.remove_provisioning_network(task)
    task.driver.network.configure_tenant_networks(task)
```

remove_provisioning_network 很简单，就是通过 neutron 接口把之前在部署网络创建的 vif 端口删除，代码也很简单。

前面我们看到，要加入租户网络的 ironic 端口有 vif id 与之关联，在 ironic 部署开始时就将其置于游离状态，现在要做的事就是把这些属于租户网络的 vif 恢复到可用的状态。

configure_tenant_networks 遍历裸机节点的 port 和 portgroup，给每个 port 和 port group 下的 port 添加 local_link_info，
调用 neutron 接口更新 port：
```python
    def configure_tenant_networks(self, task):
        node = task.node
        ports = task.ports
        LOG.info(_LI('Mapping instance ports to %s'), node.uuid)
        ports = [p for p in ports if not p.portgroup_id]
        portgroups = task.portgroups
        portmap = neutron.get_node_portmap(task)
        client = neutron.get_client()
        for port_like_obj in ports + portgroups:
            vif_port_id = (
                port_like_obj.internal_info.get(common.TENANT_VIF_KEY) or
                port_like_obj.extra.get('vif_port_id'))
            LOG.debug('Mapping tenant port %(vif_port_id)s to node '
                      '%(node_id)s',
                      {'vif_port_id': vif_port_id, 'node_id': node.uuid})
            local_link_info = []
            client_id_opt = None
            if isinstance(port_like_obj, objects.Portgroup):
                pg_ports = [p for p in task.ports
                            if p.portgroup_id == port_like_obj.id]
                for port in pg_ports:
                    local_link_info.append(portmap[port.uuid])
            else:
                # We iterate only on ports or portgroups, no need to check
                # that it is a port
                local_link_info.append(portmap[port_like_obj.uuid])
                client_id = port_like_obj.extra.get('client-id')
                if client_id:
                    client_id_opt = (
                        {'opt_name': 'client-id', 'opt_value': client_id})
            # NOTE(sambetts) Only update required binding: attributes,
            # because other port attributes may have been set by the user or
            # nova.
            body = {
                'port': {
                    'binding:vnic_type': 'baremetal',
                    'binding:host_id': node.uuid,
                    'binding:profile': {
                        'local_link_information': local_link_info,
                    },
                }
            }
            try:
                client.update_port(vif_port_id, body)
            except neutron_exceptions.ConnectionFailed as e:
                msg = (_('Could not add public network VIF %(vif)s '
                         'to node %(node)s, possible network issue. %(exc)s') %
                       {'vif': vif_port_id,
                        'node': node.uuid,
                        'exc': e})
                LOG.error(msg)
                raise exception.NetworkError(msg)
```

neutron 得到了更新的信息，便可以操作网络，将裸机加入租户网络，从而实现了网络的切换过程。

## 物理交换机控制

现在还有最后一个问题，neutron 如何控制交换机执行 VLAN 修改？显然，neutron 必须要知道物理交换机的登录信息，以及裸机与交换机的连接信息。

目前，交换机的 IP 地址，登录帐号等信息是记录在驱动的配置文件里，所以驱动登录交换机修改配置是没有问题，剩下的，neutron 只需要知道物理网口连接的是哪一台交换机的哪一个端口就可以完成这个工作。

ironic 在端口信息里增加了 local_link_connection 来记录交换机连接信息，看起来是这样的:
```python
    local_link_connection = {u'port_id': u'gei_5/2', u'switch_id': u'0c:12:62:b2:28:34'}
```

前面代码所看到的portmap里保存的就是它了，switch_id 是交换机的唯一编号，一般来说就是交换机的 MAC 地址，port_id 是该端口所连接的交换端口。
ironic 将连接信息作为 binding:profile 的一项内容传递给 neutron。

连接信息可以通过 LLDP 协议自动收集，也可以手工输入，相信没有人会愿意手工输入吧。ironic 的关联项目 ironic inspector 支持 LLDP 信息发现，
并且自动将 switch_id 和 port_id 写入 ironic 的端口表，实在是太方便了。然而，switch_info 在 LLDP 协议里是可选字段，可能有的交换机有，有的没有，因此 inspector 只将switch_id 和 port_id 写到 ironic port 表里，neutron 也就没有 switch_info 数据，如何识别网络环境下的不同类型交换机，是驱动值得考虑的问题。

## 参考资料

http://specs.openstack.org/openstack/ironic-specs/specs/7.0/ironic-ml2-integration.html
