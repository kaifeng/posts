---
layout: post
title: ironic P版本all-in-one纯虚机vlan环境搭建
author: kaifeng
date: 2017-11-07
categories: ironic
---

首先使用packstack安装一个全新的P版本all in one环境，下面的环境以192.168.122.59为例。
packstack有的地方keystone配置不全，如果后面部署遇到了，根据log排查一下，把缺的加上，遇到几处是缺auth_url。

## 配置nova

虽然P版本支持resource class调度，但是traits还没有完全支持，至少ironic还有些问题没解决，所以仍然用精确匹配。ironic需要关注的配置为以下注解部分
```
    [DEFAULT]
    rootwrap_config=/etc/nova/rootwrap.conf
    compute_driver=ironic.IronicDriver # 使用ironic驱动
    allow_resize_to_same_host=True
    vif_plugging_is_fatal=True
    vif_plugging_timeout=300
    force_raw_images=True
    reserved_host_memory_mb=0 # 不预留host内存
    cpu_allocation_ratio=16.0
    ram_allocation_ratio=1.0
    heal_instance_info_cache_interval=60
    metadata_host=192.168.122.59
    dhcp_domain=novalocal
    firewall_driver=nova.virt.firewall.NoopFirewallDriver
    state_path=/var/lib/nova
    report_interval=10
    service_down_time=60
    enabled_apis=osapi_compute,metadata
    osapi_compute_listen=0.0.0.0
    osapi_compute_listen_port=8774
    osapi_compute_workers=2
    metadata_listen=0.0.0.0
    metadata_listen_port=8775
    metadata_workers=2
    debug=False
    log_dir=/var/log/nova
    transport_url=rabbit://guest:guest@192.168.122.59:5672/
    image_service=nova.image.glance.GlanceImageService
    osapi_volume_listen=0.0.0.0
    volume_api_class=nova.volume.cinder.API
    [api]
    auth_strategy=keystone
    use_forwarded_for=False
    fping_path=/usr/sbin/fping
    [api_database]
    connection=mysql+pymysql://nova_api:a51358c6d7f0466a@192.168.122.59/nova_api
    [cinder]
    catalog_info=volumev2:cinderv2:publicURL
    [conductor]
    workers=2
    [database]
    connection=mysql+pymysql://nova:a51358c6d7f0466a@192.168.122.59/nova
    [ephemeral_storage_encryption]
    [filter_scheduler]
    host_subset_size=1
    max_io_ops_per_host=8
    max_instances_per_host=50
    available_filters=nova.scheduler.filters.all_filters
    # 调试去掉了RetryFilter，正式版本应该加上
    baremetal_enabled_filters = AvailabilityZoneFilter,ComputeFilter,ComputeCapabilitiesFilter,ImagePropertiesFilter,ExactRamFilter,ExactDiskFilter,ExactCoreFilter
    use_baremetal_filters = True
    weight_classes=nova.scheduler.weights.all_weighers
    [glance]
    api_servers=192.168.122.59:9292
    [ironic] # 这一节配置访问ironic的权限信息
    api_endpoint=http://192.168.122.59:6385/v1
    auth_url=http://192.168.122.59:35357
    project_name=services
    username=ironic
    password=keystone
    auth_plugin=password
    [keystone_authtoken]
    auth_uri=http://192.168.122.59:5000/
    auth_type=password
    auth_url=http://192.168.122.59:35357
    username=nova
    password=02d3e37db0ff460d
    project_name=services
    [libvirt]
    vif_driver=nova.virt.firewall.NoopFirewallDriver
    [neutron]
    url=http://192.168.122.59:9696
    region_name=RegionOne
    ovs_bridge=br-int
    default_floating_pool=nova
    extension_sync_interval=600
    service_metadata_proxy=True
    metadata_proxy_shared_secret=3d6b38be75164615
    timeout=60
    auth_type=v3password
    auth_url=http://192.168.122.59:35357/v3
    project_name=services
    project_domain_name=Default
    username=neutron
    user_domain_name=Default
    password=891b918106034b04
    [notifications]
    notify_api_faults=False
    [oslo_concurrency]
    lock_path=/var/lib/nova/tmp
    [oslo_messaging_rabbit]
    ssl=False
    [oslo_policy]
    policy_file=/etc/nova/policy.json
    [placement]
    os_region_name=RegionOne
    auth_type=password
    auth_url=http://192.168.122.59:5000/v3
    project_name=services
    project_domain_name=Default
    username=placement
    user_domain_name=Default
    password=02d3e37db0ff460d
    [scheduler]
    host_manager = ironic_host_manager
    driver=filter_scheduler
    max_attempts=3
    [vnc]
    enabled=True
    keymap=en-us
    vncserver_proxyclient_address=192.168.122.59
    novncproxy_base_url=http://192.168.122.59:6080/vnc_auto.html
    novncproxy_host=0.0.0.0
    novncproxy_port=6080
    [wsgi]
    api_paste_config=api-paste.ini
    [placement_database]
    connection=mysql+pymysql://nova_placement:a51358c6d7f0466a@192.168.122.59/nova_placement
```

## 配置neutron

参考了devstack的配置，组网方式如下:
```
    int_brbm           phy_brbm      ovs-node-0i0           tap-node-0i0
    br-int -------------------- brbm ---------------------------- node-0
                    (patch port) |
                                 |
                             brbm.1028 (vlan id = 1028)
```

安装genericswitch。

创建ovs桥brbm

配置ml2（ml2.conf）
```
    [DEFAULT]
    [ml2]
    type_drivers=vxlan,flat,vlan # 增加vlan
    tenant_network_types=vxlan,flat
    mechanism_drivers=openvswitch,genericswitch # 增加genericswitch
    extension_drivers=port_security
    path_mtu=0
    [ml2_type_flat]
    flat_networks=extnet
    [ml2_type_geneve]
    [ml2_type_gre]
    [ml2_type_vlan]
    network_vlan_ranges = physnet1:1000:2000 # vlan设置
    [ml2_type_vxlan]
    vni_ranges=10:100
    vxlan_group=224.0.0.1
    [securitygroup]
    firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    enable_security_group=True
```

配置openvswitch插件:
```
    [DEFAULT]
    [agent]
    tunnel_types=vxlan
    vxlan_udp_port=4789
    l2_population=False
    drop_flows_on_start=False
    [ovs]
    integration_bridge=br-int
    tunnel_bridge=br-tun
    local_ip=192.168.122.59
    bridge_mappings = extnet:br-ex,physnet1:brbm # physnet1使用brbm
    [securitygroup]
    firewall_driver=neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    [xenapi]
```

社区CI直接将brbm当作ovs桥，所以brbm会被挂到br-int桥，int-brbm和phy-brbm都是neutron创建的，无需配置。

配置genericswitch(ml2_conf_genericswitch.ini)，添加:
```
    [generic_switch:brbm]
    ngs_mac_address=<mac of brbm>
    device_type=netmiko_ovs_linux
    ip=localhost
    username=<username>
    password=<password>
    # key_file=<PERM RSA private key>
```

帐号认证和免密认证选其一，免密通过key_file指定密钥，没有用过。

Neutron-server.service增加genericswitch的configfile，重启neutron服务。

packstack装完没有创建网络，先创建两个vlan网络和子网，分别用作部署网络和租户网络，记录部署网络的uuid或者名字，后面填到ironic配置文件里。

查看部署网络的vlan tag，在brbm桥上建vlan子接口，根据实际值配，上图例子里分配的是1028:
```
ip link add link brbm brbm.1028 type vlan id 1028
```

在部署网络的子网里挑一个dhcp范围外的ip用作tftp地址，比如192.168.10.50，把这个地址配进brbm.1028

tap-node-0i0和ovs-node-0i0在注册ironic节点时再创建。

## 配置ironic

```
    [DEFAULT]
    auth_strategy=keystone
    enabled_drivers=pxe_ipmitool
    enabled_hardware_types=ipmi
    enabled_network_interfaces = neutron,flat,noop # 使能neutron
    default_network_interface = neutron # 使用neutron
    debug=True
    log_dir=/var/log/ironic
    transport_url=rabbit://guest:guest@192.168.122.59:5672/
    [agent]
    [api]
    host_ip=0.0.0.0
    port=6385
    max_limit=1000
    [conductor]
    api_url = http://192.168.10.50:6385 # api服务地址
    force_power_state_during_sync=True
    automated_clean = false
    max_time_interval=120
    [database]
    connection=mysql+pymysql://ironic:keystone@192.168.122.59/ironic
    [dhcp]
    [glance]
    auth_type=password
    glance_host=192.168.122.59
    project_name=services
    username=ironic
    [inspector]
    [keystone]
    [keystone_authtoken]
    auth_uri=http://192.168.122.59:5000/v3
    auth_type=password
    auth_url=http://127.0.0.1:35357
    username=ironic
    password=keystone
    project_name=services
    [matchmaker_redis]
    [neutron]
    auth_url=http://127.0.0.1:35357 # packstack没有配置这项，要加上
    auth_type=password
    cleaning_network = provision
    password = keystone
    project_name=services
    provisioning_network = provision # 部署网络名称
    username=ironic
    [oslo_concurrency]
    [oslo_messaging_notifications]
    [oslo_messaging_rabbit]
    ssl=False
    [oslo_policy]
    policy_file=/etc/ironic/policy.json
    [pxe]
    tftp_server = 192.168.10.50 # ftp服务地址
    [service_catalog]
    [swift]
```

## 部署准备

其它的创建flavor，上传glance镜像就不多说了。

另外packstack不安装pxe环境，所以xinetd要装，tftp要配置，这些可以参考其它文档。

安装virtualbmc:
```
yum install -y python2-virtualbmc
```
创建veth pair并把虚机与brbm连起来:
```
ip link add ovs-node-0i0 type veth peer name tap-node-0i0
ovs-vsctl add-port brbm ovs-node-0i0
```

ovs-node-0i0必须UP

虚机XML的网口配置，目前是这样写的:
```
    <interface type='direct'>
    <mac address='7a:52:00:86:a4:81'/>
    <source dev='tap-node-0i0' mode='vepa'/>
    <model type='virtio'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
    </interface>
```

社区是用的ethernet类型，试了下设为ethernet的话，libvirt会去创建接口，又和veth pair冲突了，后面可以再研究:
```
    <interface type='ethernet'>
        <target dev='tap-node-0i0' />
```

虚机define到libvirt，用vbmc添加一个虚bmc:
```
    vbmc add --port 6230 node-0
    vbmc start node-0
```

注册ironic节点，driver_info和properties的配置参考：

* driver_info
* ipmi_port: 6230
* ipmi_username: admin
* deploy_kernel: acbf6952-fc72-4b71-bf33-4cf1ea22a284
* ipmi_address: 127.0.0.1
* deploy_ramdisk: def732dc-5e65-4d7e-be39-a78c699f4292
* ipmi_password: password
* properties
* memory_mb: 2048
* cpu_arch: x86_64
* local_gb: 10
* cpus: 1
* capabilities: boot_option:local

注册port，local_link_connection的配置参考：

* address 7a:52:00:86:a4:81
* local_link_connection
* switch_info: brbm
* port_id: ovs-node0i0
* switch_id: 8e:df:df:61:61:46

注意local_link_connection的信息，switch_id是brbm桥的mac，port_id是虚机网口veth pair连到brbm桥的那一端。

注册完ironic节点后，如果到available状态后，nova hypervisor列表仍然为空，手动执行一下:
```
nova-manage cell_v2 discover_hosts
```
这是新增的placement服务引入的，用于将节点资源映射到cell，只有分配到cell的资源才能被使用。

在这个阶段，环境就具备跑tempest的条件了。
