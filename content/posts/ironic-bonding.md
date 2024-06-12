---
layout: post
title: ironic bonding实现
author: kaifeng
date: 2018-01-13
categories: ironic
---

ironic支持bonding的基本思路是：

1. 用户先通过ironic注册portgroup，填写bond配置信息，将port与该portgroup关联。
2. nova boot部署裸机（必须打开configdrive），ironic部署裸机时，将configdrive信息写入裸机磁盘的config-2分区，configdrive数据中已经包含了portgroup信息。
3. ironic在裸机部署最后，执行租户网络切换时，将portgroup信息更新到neutron port，ngs驱动据此信息完成交换机的配置。
4. 裸机部署完毕，上电后运行cloudinit，cloudinit读取configdrive中的数据，实施bond配置。

## portgroup注册

以两个网口的bond为例，先创建两个网口：
```shell
    # ironic port-list
    +--------------------------------------+-------------------+
    | UUID                                 | Address           |
    +--------------------------------------+-------------------+
    | 42b06aa8-27c0-44d8-9280-e6994dad9c26 | 7a:52:00:86:a4:81 |
    | 44af766f-9e54-423e-9652-7f2122d315b6 | 7a:52:00:86:a4:82 |
    +--------------------------------------+-------------------+

    # ironic port-show 42b06aa8-27c0-44d8-9280-e6994dad9c26
    +-----------------------+---------------------------------------------------------------------+
    | Property              | Value                                                               |
    +-----------------------+---------------------------------------------------------------------+
    | address               | 7a:52:00:86:a4:81                                                   |
    | created_at            | 2017-11-29T04:41:54+00:00                                           |
    | extra                 | {}                                                                  |
    | internal_info         | {}                                                                  |
    | local_link_connection | {u'switch_info': u'brbm', u'port_id': u'ovs-node0i0', u'switch_id': |
    |                       | u'8e:df:df:61:61:46'}                                               |
    | node_uuid             | a308bca6-e6a3-4349-b8ea-695e17672898                                |
    | physical_network      | None                                                                |
    | portgroup_uuid        | None                                                                |
    | pxe_enabled           | True                                                                |
    | updated_at            | 2018-01-04T07:06:46+00:00                                           |
    | uuid                  | 42b06aa8-27c0-44d8-9280-e6994dad9c26                                |
    +-----------------------+---------------------------------------------------------------------+

    # ironic port-show 44af766f-9e54-423e-9652-7f2122d315b6
    +-----------------------+---------------------------------------------------------------------+
    | Property              | Value                                                               |
    +-----------------------+---------------------------------------------------------------------+
    | address               | 7a:52:00:86:a4:82                                                   |
    | created_at            | 2017-12-22T02:52:53+00:00                                           |
    | extra                 | {}                                                                  |
    | internal_info         | {}                                                                  |
    | local_link_connection | {u'switch_info': u'brbm', u'port_id': u'ovs-node0i1', u'switch_id': |
    |                       | u'8e:df:df:61:61:46'}                                               |
    | node_uuid             | a308bca6-e6a3-4349-b8ea-695e17672898                                |
    | physical_network      | None                                                                |
    | portgroup_uuid        | None                                                                |
    | pxe_enabled           | True                                                                |
    | updated_at            | 2018-01-04T06:58:42+00:00                                           |
    | uuid                  | 44af766f-9e54-423e-9652-7f2122d315b6                                |
    +-----------------------+---------------------------------------------------------------------+
```

创建portgroup：

```shell
    # openstack --os-baremetal-api-version 1.26 baremetal port group create --node a308bca6-e6a3-4349-b8ea-695e17672898 --name pg0 --mode 1 --property miimon=100
    # ironic portgroup-show pg0
    +----------------------------+--------------------------------------+
    | Property                   | Value                                |
    +----------------------------+--------------------------------------+
    | address                    | None                                 |
    | created_at                 | 2018-01-04T07:15:29+00:00            |
    | extra                      | {}                                   |
    | internal_info              | {}                                   |
    | mode                       | 1                                    |
    | name                       | pg0                                  |
    | node_uuid                  | a308bca6-e6a3-4349-b8ea-695e17672898 |
    | properties                 | {u'miimon': 100}                     |
    | standalone_ports_supported | True                                 |
    | updated_at                 | None                                 |
    | uuid                       | 0675d334-1adb-4d6e-b9a1-73303be8384c |
    +----------------------------+--------------------------------------+
```

把网口加到portgroup：
```shell
    # openstack --os-baremetal-api-version 1.26 baremetal port set 42b06aa8-27c0-44d8-9280-e6994dad9c26 --port-group 0675d334-1adb-4d6e-b9a1-73303be8384c
    # openstack --os-baremetal-api-version 1.26 baremetal port set 44af766f-9e54-423e-9652-7f2122d315b6 --port-group 0675d334-1adb-4d6e-b9a1-73303be8384c
```

这个操作不能在available状态，建议在注册阶段完成这个工作。


## 裸机部署

nova boot --config-drive true 启动部署。

ironic给所有注册网口都生成一套pxe配置（与bond无关)，见pxe_utils._link_mac_pxe_configs

```shell
    [root@localhost pxelinux.cfg(keystone_admin)]# ll
    total 0
    lrwxrwxrwx 1 ironic ironic 46 Jan  4 15:26 01-7a-52-00-86-a4-81 -> ../a308bca6-e6a3-4349-b8ea-695e17672898/config
    lrwxrwxrwx 1 ironic ironic 46 Jan  4 15:26 01-7a-52-00-86-a4-82 -> ../a308bca6-e6a3-4349-b8ea-695e17672898/config
```

在provision阶段，不涉及bond，只有在切换到租户网络时，才会携带`local_group_information`。

布署前的neutron port状态，初始状态：
```shell
    # neutron port-show 79c2aa4a-f3c4-490c-9b61-4d70dada7359
    +-----------------------+---------------------------------------------------------------------------------------+
    | Field                 | Value                                                                                 |
    +-----------------------+---------------------------------------------------------------------------------------+
    | admin_state_up        | True                                                                                  |
    | allowed_address_pairs |                                                                                       |
    | binding:host_id       |                                                                                       |
    | binding:profile       | {}                                                                                    |
    | binding:vif_details   | {}                                                                                    |
    | binding:vif_type      | unbound                                                                               |
    | binding:vnic_type     | normal                                                                                |
    | created_at            | 2018-01-04T07:22:03Z                                                                  |
    | description           |                                                                                       |
    | device_id             | 9a8f7cf5-dab3-4a38-9406-044fbec01eaa                                                  |
    | device_owner          | compute:nova                                                                          |
    | extra_dhcp_opts       |                                                                                       |
    | fixed_ips             | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.105"} |
    | id                    | 79c2aa4a-f3c4-490c-9b61-4d70dada7359                                                  |
    | mac_address           | fa:16:3e:5b:b8:3b                                                                     |
    | name                  |                                                                                       |
    | network_id            | 596fcf91-37a3-4a86-8135-f8e267b0624b                                                  |
    | port_security_enabled | True                                                                                  |
    | project_id            | ac9cab947db74c0c999380a4f306693b                                                      |
    | revision_number       | 5                                                                                     |
    | security_groups       | 6b470632-0ab9-4f5a-a651-052f517afdfc                                                  |
    | status                | DOWN                                                                                  |
    | tags                  |                                                                                       |
    | tenant_id             | ac9cab947db74c0c999380a4f306693b                                                      |
    | updated_at            | 2018-01-04T07:22:07Z                                                                  |
    +-----------------------+---------------------------------------------------------------------------------------+
```

部署时的neutron port状态，只是更新了tftp信息：

```shell
    # neutron port-show 79c2aa4a-f3c4-490c-9b61-4d70dada7359
    +-----------------------+---------------------------------------------------------------------------------------+
    | Field                 | Value                                                                                 |
    +-----------------------+---------------------------------------------------------------------------------------+
    | admin_state_up        | True                                                                                  |
    | allowed_address_pairs |                                                                                       |
    | binding:host_id       |                                                                                       |
    | binding:profile       | {}                                                                                    |
    | binding:vif_details   | {}                                                                                    |
    | binding:vif_type      | unbound                                                                               |
    | binding:vnic_type     | normal                                                                                |
    | created_at            | 2018-01-04T07:22:03Z                                                                  |
    | description           |                                                                                       |
    | device_id             | 9a8f7cf5-dab3-4a38-9406-044fbec01eaa                                                  |
    | device_owner          | compute:nova                                                                          |
    | extra_dhcp_opts       | {"opt_value": "pxelinux.0", "ip_version": 4, "opt_name": "bootfile-name"}             |
    |                       | {"opt_value": "192.168.10.50", "ip_version": 4, "opt_name": "tftp-server"}            |
    |                       | {"opt_value": "192.168.10.50", "ip_version": 4, "opt_name": "server-ip-address"}      |
    |                       | {"opt_value": "/tftpboot/", "ip_version": 4, "opt_name": "210"}                       |
    | fixed_ips             | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.105"} |
    | id                    | 79c2aa4a-f3c4-490c-9b61-4d70dada7359                                                  |
    | mac_address           | fa:16:3e:5b:b8:3b                                                                     |
    | name                  |                                                                                       |
    | network_id            | 596fcf91-37a3-4a86-8135-f8e267b0624b                                                  |
    | port_security_enabled | True                                                                                  |
    | project_id            | ac9cab947db74c0c999380a4f306693b                                                      |
    | revision_number       | 10                                                                                    |
    | security_groups       | 6b470632-0ab9-4f5a-a651-052f517afdfc                                                  |
    | status                | DOWN                                                                                  |
    | tags                  |                                                                                       |
    | tenant_id             | ac9cab947db74c0c999380a4f306693b                                                      |
    | updated_at            | 2018-01-04T07:24:23Z                                                                  |
    +-----------------------+---------------------------------------------------------------------------------------+
```

部署完成时的neutron port状态：

```shell
    # neutron port-list
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
    | id                                   | name | tenant_id                        | mac_address       | fixed_ips                                                                             |
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
    | 41a7b36d-e81f-42d3-b513-acae9797d1f3 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:eb:a5:ad | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.100"} |
    | 68b334d1-0620-437e-ab39-03c6df0e1651 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:14:be:3b | {"subnet_id": "9fbbbb4a-e58f-43cc-a5ff-bf3784526a0e", "ip_address": "192.168.10.100"} |
    | 79c2aa4a-f3c4-490c-9b61-4d70dada7359 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:5b:b8:3b | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.105"} |
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
    # neutron port-show 79c2aa4a-f3c4-490c-9b61-4d70dada7359
    +-----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Field                 | Value                                                                                                                                                                                                                                                                                                                                                       |
    +-----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | admin_state_up        | True                                                                                                                                                                                                                                                                                                                                                        |
    | allowed_address_pairs |                                                                                                                                                                                                                                                                                                                                                             |
    | binding:host_id       | a308bca6-e6a3-4349-b8ea-695e17672898                                                                                                                                                                                                                                                                                                                        |
    | binding:profile       | {"local_link_information": [{"switch_info": "brbm", "port_id": "ovs-node0i0", "switch_id": "8e:df:df:61:61:46"}, {"switch_info": "brbm", "port_id": "ovs-node0i1", "switch_id": "8e:df:df:61:61:46"}], "local_group_information": {"bond_mode": "1", "id": "0675d334-1adb-4d6e-b9a1-73303be8384c", "name": "pg0", "bond_properties": {"bond_miimon": 100}}} |
    | binding:vif_details   | {}                                                                                                                                                                                                                                                                                                                                                          |
    | binding:vif_type      | binding_failed                                                                                                                                                                                                                                                                                                                                              |
    | binding:vnic_type     | baremetal                                                                                                                                                                                                                                                                                                                                                   |
    | created_at            | 2018-01-04T07:22:03Z                                                                                                                                                                                                                                                                                                                                        |
    | description           |                                                                                                                                                                                                                                                                                                                                                             |
    | device_id             | 9a8f7cf5-dab3-4a38-9406-044fbec01eaa                                                                                                                                                                                                                                                                                                                        |
    | device_owner          | compute:nova                                                                                                                                                                                                                                                                                                                                                |
    | extra_dhcp_opts       | {"opt_value": "pxelinux.0", "ip_version": 4, "opt_name": "bootfile-name"}                                                                                                                                                                                                                                                                                   |
    |                       | {"opt_value": "192.168.10.50", "ip_version": 4, "opt_name": "tftp-server"}                                                                                                                                                                                                                                                                                  |
    |                       | {"opt_value": "192.168.10.50", "ip_version": 4, "opt_name": "server-ip-address"}                                                                                                                                                                                                                                                                            |
    |                       | {"opt_value": "/tftpboot/", "ip_version": 4, "opt_name": "210"}                                                                                                                                                                                                                                                                                             |
    | fixed_ips             | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.105"}                                                                                                                                                                                                                                                                       |
    | id                    | 79c2aa4a-f3c4-490c-9b61-4d70dada7359                                                                                                                                                                                                                                                                                                                        |
    | mac_address           | fa:16:3e:5b:b8:3b                                                                                                                                                                                                                                                                                                                                           |
    | name                  |                                                                                                                                                                                                                                                                                                                                                             |
    | network_id            | 596fcf91-37a3-4a86-8135-f8e267b0624b                                                                                                                                                                                                                                                                                                                        |
    | port_security_enabled | True                                                                                                                                                                                                                                                                                                                                                        |
    | project_id            | ac9cab947db74c0c999380a4f306693b                                                                                                                                                                                                                                                                                                                            |
    | revision_number       | 12                                                                                                                                                                                                                                                                                                                                                          |
    | security_groups       | 6b470632-0ab9-4f5a-a651-052f517afdfc                                                                                                                                                                                                                                                                                                                        |
    | status                | DOWN                                                                                                                                                                                                                                                                                                                                                        |
    | tags                  |                                                                                                                                                                                                                                                                                                                                                             |
    | tenant_id             | ac9cab947db74c0c999380a4f306693b                                                                                                                                                                                                                                                                                                                            |
    | updated_at            | 2018-01-04T07:30:30Z                                                                                                                                                                                                                                                                                                                                        |
    +-----------------------+-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

可以看到local_group_information已经update到vif上。

把binding profile格式化来看：

```json
    {
        "local_link_information": [
            {
                "switch_info": "brbm",
                "port_id": "ovs-node0i0",
                "switch_id": "8e:df:df:61:61:46"
            },
            {
                "switch_info": "brbm",
                "port_id": "ovs-node0i1",
                "switch_id": "8e:df:df:61:61:46"
            }
        ],
        "local_group_information": {
            "bond_mode": "1",
            "id": "0675d334-1adb-4d6e-b9a1-73303be8384c",
            "name": "pg0",
            "bond_properties": {
                "bond_miimon": 100
            }
        }
    }
```

ngs驱动要做的就是根据这些信息配置交换，当存在local_group_information时，local_link_information下的端口都属于该bond口，如果不是portgroup，则不存在local_group_information。

> 见ironic.drivers.modules.network.common.plug_port_to_tenant_network的实现。

## 租户网络切换

ironic将binding:profile更新到vif port时，ml2 mechanism driver也就是ngs来实施配置。

机制驱动的结构不复杂，因为这个操作只涉及端口，所以涉及的接口只有update_port_precommit, update_port_postcommit及bind_port三个。P版本安装出来的ngs版本与最新的master有差异，不过流程不会有很大差异，ngs驱动能拿到port的所有数据，neutron可以搞定。只是ngs的api设计得不灵活，要动接口。

P版本的代码，只在bind_port和delete_port做了处理，master分支代码在update_port_postcommit还有处理，增加了vif从bind到unbind状态的清理操作。

部署时ngs收到的：
```
2018-01-08 13:50:11.744 3732 INFO networking_generic_switch.generic_switch_mech [req-ba105786-f333-4a64-9c69-ca4558cb6ac6 7560a3ccb61d436aa66e9a1f5b96ddf6 98675d365278409bbfa1afb18eeea124 - default default] bind_port: port {'allowed_address_pairs': [], 'extra_dhcp_opts': [{'opt_value': u'/tftpboot/', 'ip_version': 4, 'opt_name': u'210'}, {'opt_value': u'192.168.10.50', 'ip_version': 4, 'opt_name': u'tftp-server'}, {'opt_value': u'192.168.10.50', 'ip_version': 4, 'opt_name': u'server-ip-address'}, {'opt_value': u'pxelinux.0', 'ip_version': 4, 'opt_name': u'bootfile-name'}], 'updated_at': '2018-01-08T05:38:42Z', 'device_owner': u'compute:nova', 'revision_number': 10, 'binding:profile': {u'local_link_information': [{u'switch_info': u'brbm', u'port_id': u'ovs-node0i0', u'switch_id': u'8e:df:df:61:61:46'}, {u'switch_info': u'brbm', u'port_id': u'ovs-node0i1', u'switch_id': u'8e:df:df:61:61:46'}], u'local_group_information': {u'bond_mode': u'1', u'bond_properties': {u'bond_miimon': 100}, u'name': u'pg0', u'id': u'0675d334-1adb-4d6e-b9a1-73303be8384c'}}, 'port_security_enabled': True, 'fixed_ips': [{'subnet_id': u'dbfcccf2-7c03-4c52-827b-e91180b0e24a', 'ip_address': u'192.168.20.104'}], 'id': u'4858bc68-cca7-42c6-aa7a-fdbb7c8b043d', 'security_groups': [u'6b470632-0ab9-4f5a-a651-052f517afdfc'], 'binding:vif_details': {}, 'binding:vif_type': 'unbound', 'mac_address': u'fa:16:3e:f5:06:6d', 'project_id': u'ac9cab947db74c0c999380a4f306693b', 'status': 'DOWN', 'binding:host_id': u'a308bca6-e6a3-4349-b8ea-695e17672898', 'description': u'', 'tags': [], 'device_id': u'fb3189b1-e0b3-4534-acf1-6aa953d649b7', 'name': u'', 'admin_state_up': True, 'network_id': u'596fcf91-37a3-4a86-8135-f8e267b0624b', 'tenant_id': u'ac9cab947db74c0c999380a4f306693b', 'created_at': '2018-01-08T05:36:22Z', 'binding:vnic_type': u'baremetal'}
```

格式化后的：
```json
    {
        'allowed_address_pairs': [

        ],
        'extra_dhcp_opts': [
            {
                'opt_value': u'/tftpboot/',
                'ip_version': 4,
                'opt_name': u'210'
            },
            {
                'opt_value': u'192.168.10.50',
                'ip_version': 4,
                'opt_name': u'tftp-server'
            },
            {
                'opt_value': u'192.168.10.50',
                'ip_version': 4,
                'opt_name': u'server-ip-address'
            },
            {
                'opt_value': u'pxelinux.0',
                'ip_version': 4,
                'opt_name': u'bootfile-name'
            }
        ],
        'updated_at': '2018-01-08T05: 38: 42Z',
        'device_owner': u'compute: nova',
        'revision_number': 10,
        'binding: profile': {
            u'local_link_information': [
                {
                    u'switch_info': u'brbm',
                    u'port_id': u'ovs-node0i0',
                    u'switch_id': u'8e: df: df: 61: 61: 46'
                },
                {
                    u'switch_info': u'brbm',
                    u'port_id': u'ovs-node0i1',
                    u'switch_id': u'8e: df: df: 61: 61: 46'
                }
            ],
            u'local_group_information': {
                u'bond_mode': u'1',
                u'bond_properties': {
                    u'bond_miimon': 100
                },
                u'name': u'pg0',
                u'id': u'0675d334-1adb-4d6e-b9a1-73303be8384c'
            }
        },
        'port_security_enabled': True,
        'fixed_ips': [
            {
                'subnet_id': u'dbfcccf2-7c03-4c52-827b-e91180b0e24a',
                'ip_address': u'192.168.20.104'
            }
        ],
        'id': u'4858bc68-cca7-42c6-aa7a-fdbb7c8b043d',
        'security_groups': [
            u'6b470632-0ab9-4f5a-a651-052f517afdfc'
        ],
        'binding: vif_details': {

        },
        'binding: vif_type': 'unbound',
        'mac_address': u'fa: 16: 3e: f5: 06: 6d',
        'project_id': u'ac9cab947db74c0c999380a4f306693b',
        'status': 'DOWN',
        'binding: host_id': u'a308bca6-e6a3-4349-b8ea-695e17672898',
        'description': u'',
        'tags': [

        ],
        'device_id': u'fb3189b1-e0b3-4534-acf1-6aa953d649b7',
        'name': u'',
        'admin_state_up': True,
        'network_id': u'596fcf91-37a3-4a86-8135-f8e267b0624b',
        'tenant_id': u'ac9cab947db74c0c999380a4f306693b',
        'created_at': '2018-01-08T05: 36: 22Z',
        'binding: vnic_type': u'baremetal'
    }
```

删除port时ngs收到的：

```
2018-01-08 14:08:32.421 3731 INFO networking_generic_switch.generic_switch_mech [req-15a6395e-ecb2-42e5-bf01-bdf58b8be031 0d290bceca6743078ff27de47d92fd40 ac9cab947db74c0c999380a4f306693b - default default] delete_port_postcommit: port {'status': u'DOWN', 'binding:host_id': u'', 'description': u'', 'allowed_address_pairs': [], 'tags': [], 'extra_dhcp_opts': [{'opt_value': u'/tftpboot/', 'ip_version': 4, 'opt_name': u'210'}, {'opt_value': u'192.168.10.50', 'ip_version': 4, 'opt_name': u'tftp-server'}, {'opt_value': u'192.168.10.50', 'ip_version': 4, 'opt_name': u'server-ip-address'}, {'opt_value': u'pxelinux.0', 'ip_version': 4, 'opt_name': u'bootfile-name'}], 'updated_at': '2018-01-08T06:08:30Z', 'device_owner': u'compute:nova', 'revision_number': 13, 'port_security_enabled': True, 'binding:profile': {}, 'fixed_ips': [{'subnet_id': u'dbfcccf2-7c03-4c52-827b-e91180b0e24a', 'ip_address': u'192.168.20.104'}], 'id': u'4858bc68-cca7-42c6-aa7a-fdbb7c8b043d', 'security_groups': [u'6b470632-0ab9-4f5a-a651-052f517afdfc'], 'device_id': u'fb3189b1-e0b3-4534-acf1-6aa953d649b7', 'name': u'', 'admin_state_up': True, 'network_id': u'596fcf91-37a3-4a86-8135-f8e267b0624b', 'tenant_id': u'ac9cab947db74c0c999380a4f306693b', 'binding:vif_details': {}, 'binding:vnic_type': u'baremetal', 'binding:vif_type': u'unbound', 'mac_address': u'fa:16:3e:f5:06:6d', 'project_id': u'ac9cab947db74c0c999380a4f306693b', 'created_at': '2018-01-08T05:36:22Z'}
```

格式化后的：
```json
    {
        'status': u'DOWN',
        'binding: host_id': u'',
        'description': u'',
        'allowed_address_pairs': [

        ],
        'tags': [

        ],
        'extra_dhcp_opts': [
            {
                'opt_value': u'/tftpboot/',
                'ip_version': 4,
                'opt_name': u'210'
            },
            {
                'opt_value': u'192.168.10.50',
                'ip_version': 4,
                'opt_name': u'tftp-server'
            },
            {
                'opt_value': u'192.168.10.50',
                'ip_version': 4,
                'opt_name': u'server-ip-address'
            },
            {
                'opt_value': u'pxelinux.0',
                'ip_version': 4,
                'opt_name': u'bootfile-name'
            }
        ],
        'updated_at': '2018-01-08T06: 08: 30Z',
        'device_owner': u'compute: nova',
        'revision_number': 13,
        'port_security_enabled': True,
        'binding: profile': {

        },
        'fixed_ips': [
            {
                'subnet_id': u'dbfcccf2-7c03-4c52-827b-e91180b0e24a',
                'ip_address': u'192.168.20.104'
            }
        ],
        'id': u'4858bc68-cca7-42c6-aa7a-fdbb7c8b043d',
        'security_groups': [
            u'6b470632-0ab9-4f5a-a651-052f517afdfc'
        ],
        'device_id': u'fb3189b1-e0b3-4534-acf1-6aa953d649b7',
        'name': u'',
        'admin_state_up': True,
        'network_id': u'596fcf91-37a3-4a86-8135-f8e267b0624b',
        'tenant_id': u'ac9cab947db74c0c999380a4f306693b',
        'binding: vif_details': {

        },
        'binding: vnic_type': u'baremetal',
        'binding: vif_type': u'unbound',
        'mac_address': u'fa: 16: 3e: f5: 06: 6d',
        'project_id': u'ac9cab947db74c0c999380a4f306693b',
        'created_at': '2018-01-08T05: 36: 22Z'
    }
```

ngs可以按照vlan配置的实现方式来做，在bind_port里处理bond配置，在delete_port里清除bond配置，正常流程下这个不会有问题。

一个隐患是，如果port没有被正常清除，会残留bond信息，下次再部署该裸机时，由于provision网络是不考虑bond的，若交换端口未清除bond配置，裸机无法再部署成功，需要人为干预，这个需要确认是否存在可能。

如果交换机需要其它配置，可以填到portgroup的properties字段，这个会像mii_mon一样带下来，这需要用户提供。

## configdrive信息

裸机的配置由cloudinit实施，cloudinit读取configdrive里的信息，下面是环境中的数据：

```shell
    [root@localhost latest(keystone_admin)]# cat network_data.json
    {
        "services": [],
        "networks": [
            {
                "network_id": "596fcf91-37a3-4a86-8135-f8e267b0624b",
                "link": "tape1293336-53",
                "type": "ipv4_dhcp",
                "id": "network0"
            }
        ],
        "links": [
            {
                "bond_miimon": 100,
                "vif_id": "e1293336-5343-4bfa-883a-0f5ba1681760",
                "ethernet_mac_address": "fa:16:3e:c1:b0:6d",
                "mtu": 1500,
                "bond_mode": "1",
                "bond_links": [
                    "42b06aa8-27c0-44d8-9280-e6994dad9c26",
                    "44af766f-9e54-423e-9652-7f2122d315b6"
                ],
                "type": "bond",
                "id": "tape1293336-53"
            },
            {
                "ethernet_mac_address": "7a:52:00:86:a4:81",
                "type": "phy",
                "id": "42b06aa8-27c0-44d8-9280-e6994dad9c26"
            },
            {
                "ethernet_mac_address": "7a:52:00:86:a4:82",
                "type": "phy",
                "id": "44af766f-9e54-423e-9652-7f2122d315b6"
            }
        ]
    }
```

links里的信息可供cloud init作配置。

## cloudinit

Centos7的官方源提供的cloudinit是0.7.9版本，这个比较旧，手工测试cloudinit的结果是对bond的支持不好，有些配置没有正确生成，看了master分支的代码，这一块有明显改进。

而配合configdrive使用时，net/sysconfig.py:_render_bond_interfaces会抛异常，报没有指定bond-master，不能生成ifcfg-bondX配置文件。

修改代码，对不存在bond-master时，跳过MASTER和SLAVE的配置，可以生成配置文件，不过这个版本对bond的配置支持不好，0.7.9在生成bond配置的代码上本身也存在问题，可以用高版本的再测试。

若不采用userdata的方式，需要针对cloudinit做一定的开发，但用户镜像可能搭配不同版本，userdata可能更合适一点。

如果采用userdata的方式，就只利用了cloudinit触发脚本执行的功能，优点是对用户系统依赖小，缺点是需要脚本需要区分操作系统，若做不到，还可能要暴露到用户层面来提供。

