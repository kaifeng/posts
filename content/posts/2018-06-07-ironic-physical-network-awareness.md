---
layout: post
title: ironic physical network awareness
author: kaifeng
date: 2018-06-07
categories: ironic
---

在部署裸机实例的过程中，当需要使用多个网口且处于不同租户网络时，裸机的端口是按顺序挑选的，不可控制。如果不同的租户网络在底层属于不同的物理网络，则有可能使端口的分配与实际网络环境不符，导致网络不通。ironic在P版本实现的物理网络感知功能，即在port对象增加一个physical_network字段记录该网口所属的逻辑网络，在部署时，ironic优先按照对应原则分配port，以保证网口能分配到对应的网络中。

配置物理网络感知功能，用户需要为网口指定所属的物理网络。

1. 获取租户网络对应的物理网络信息。
```
openstack network show <net-uuid> -f value -c provider:physical_network
```

2. 为裸金属网口指定物理网络。
```
openstack --os-baremetal-api-version 1.34 baremetal port set --physical-network physnet1 dbc1ccdc-5ad6-4a85-bc26-faae11e49828
```

现有两个租户网络：
```
    [root@localhost ~(keystone_admin)]# neutron net-list
    neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
    +--------------------------------------+-----------+----------------------------------+------------------------------------------------------+
    | id                                   | name      | tenant_id                        | subnets                                              |
    +--------------------------------------+-----------+----------------------------------+------------------------------------------------------+
    | 44387a34-e8d9-4498-8eb0-63ac1bc8256f | tenant2   | ac9cab947db74c0c999380a4f306693b | f59be5cb-f8e5-461a-823b-b15e38f1ebef 192.168.40.0/24 |
    | 596fcf91-37a3-4a86-8135-f8e267b0624b | tenant1   | ac9cab947db74c0c999380a4f306693b | dbfcccf2-7c03-4c52-827b-e91180b0e24a 192.168.20.0/24 |
    | 83b424ca-deb4-437c-a5d7-3ad0db3d19ba | provision | ac9cab947db74c0c999380a4f306693b | 9fbbbb4a-e58f-43cc-a5ff-bf3784526a0e 192.168.10.0/24 |
    +--------------------------------------+-----------+----------------------------------+------------------------------------------------------+
```

tenant1为physnet1，tenant2为physnet2：
```
    [root@localhost ~(keystone_admin)]# neutron net-show tenant1
    neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | availability_zone_hints   |                                      |
    | availability_zones        | nova                                 |
    | created_at                | 2017-11-06T01:22:09Z                 |
    | description               |                                      |
    | id                        | 596fcf91-37a3-4a86-8135-f8e267b0624b |
    | ipv4_address_scope        |                                      |
    | ipv6_address_scope        |                                      |
    | mtu                       | 1500                                 |
    | name                      | tenant1                              |
    | port_security_enabled     | True                                 |
    | project_id                | ac9cab947db74c0c999380a4f306693b     |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  | 1055                                 |
    | revision_number           | 3                                    |
    | router:external           | False                                |
    | shared                    | True                                 |
    | status                    | ACTIVE                               |
    | subnets                   | dbfcccf2-7c03-4c52-827b-e91180b0e24a |
    | tags                      |                                      |
    | tenant_id                 | ac9cab947db74c0c999380a4f306693b     |
    | updated_at                | 2017-11-06T01:22:42Z                 |
    +---------------------------+--------------------------------------+
    [root@localhost ~(keystone_admin)]# neutron net-show tenant2
    neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | availability_zone_hints   |                                      |
    | availability_zones        | nova                                 |
    | created_at                | 2018-06-05T03:39:28Z                 |
    | description               |                                      |
    | id                        | 44387a34-e8d9-4498-8eb0-63ac1bc8256f |
    | ipv4_address_scope        |                                      |
    | ipv6_address_scope        |                                      |
    | mtu                       | 1500                                 |
    | name                      | tenant2                              |
    | port_security_enabled     | True                                 |
    | project_id                | ac9cab947db74c0c999380a4f306693b     |
    | provider:network_type     | vlan                                 |
    | provider:physical_network | physnet2                             |
    | provider:segmentation_id  | 2065                                 |
    | revision_number           | 3                                    |
    | router:external           | False                                |
    | shared                    | True                                 |
    | status                    | ACTIVE                               |
    | subnets                   | f59be5cb-f8e5-461a-823b-b15e38f1ebef |
    | tags                      |                                      |
    | tenant_id                 | ac9cab947db74c0c999380a4f306693b     |
    | updated_at                | 2018-06-05T04:06:18Z                 |
    +---------------------------+--------------------------------------+
    [root@localhost ~(keystone_admin)]#
```

physnet1和physnet2由bridge_mappings指定到不同的物理网络：
```
bridge_mappings = extnet:br-ex,physnet1:brbm,physnet2:br-eth2
```

部署前，需要用户设置 physical_network字段，先把7a:52:00:86:a4:81指定到physnet1网络，7a:52:00:86:a4:82指定到physnet2网络：
```
    [root@localhost ~(keystone_admin)]# ironic port-show 42b06aa8-27c0-44d8-9280-e6994dad9c26
    +-----------------------+---------------------------------------------------------------------+
    | Property              | Value                                                               |
    +-----------------------+---------------------------------------------------------------------+
    | address               | 7a:52:00:86:a4:81                                                   |
    | created_at            | 2017-11-29T04:41:54+00:00                                           |
    | extra                 | {}                                                                  |
    | internal_info         | {u'tenant_vif_port_id': u'92172f4a-f8f3-4fcc-b8c1-76dd809792bd'}    |
    | local_link_connection | {u'switch_info': u'brbm', u'port_id': u'ovs-node0i0', u'switch_id': |
    |                       | u'8e:df:df:61:61:46'}                                               |
    | node_uuid             | a308bca6-e6a3-4349-b8ea-695e17672898                                |
    | physical_network      | physnet1                                                            |
    | portgroup_uuid        | None                                                                |
    | pxe_enabled           | True                                                                |
    | updated_at            | 2018-06-05T07:19:50+00:00                                           |
    | uuid                  | 42b06aa8-27c0-44d8-9280-e6994dad9c26                                |
    +-----------------------+---------------------------------------------------------------------+
    [root@localhost ~(keystone_admin)]# ironic port-show 44af766f-9e54-423e-9652-7f2122d315b6
    +-----------------------+---------------------------------------------------------------------+
    | Property              | Value                                                               |
    +-----------------------+---------------------------------------------------------------------+
    | address               | 7a:52:00:86:a4:82                                                   |
    | created_at            | 2017-12-22T02:52:53+00:00                                           |
    | extra                 | {}                                                                  |
    | internal_info         | {u'tenant_vif_port_id': u'87bf6c89-c354-4237-be5c-14e03b1c2566'}    |
    | local_link_connection | {u'switch_info': u'brbm', u'port_id': u'ovs-node0i1', u'switch_id': |
    |                       | u'8e:df:df:61:61:46'}                                               |
    | node_uuid             | a308bca6-e6a3-4349-b8ea-695e17672898                                |
    | physical_network      | physnet2                                                            |
    | portgroup_uuid        | None                                                                |
    | pxe_enabled           | True                                                                |
    | updated_at            | 2018-06-05T07:19:50+00:00                                           |
    | uuid                  | 44af766f-9e54-423e-9652-7f2122d315b6                                |
    +-----------------------+---------------------------------------------------------------------+
```

部署裸机，指定tenant1和tenant2两个网络：
```
nova boot --flavor baremetal --image cirros --config-drive true --nic net-id=596fcf91-37a3-4a86-8135-f8e267b0624b --nic net-id=44387a34-e8d9-4498-8eb0-63ac1bc8256f cirros
```

部署完成，查看neutron的端口：
```
    [root@localhost ~(keystone_admin)]# neutron port-list
    neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
    | id                                   | name | tenant_id                        | mac_address       | fixed_ips                                                                             |
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
    | 21e9f776-ee5c-43b4-bb81-bbf0e3f04651 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:8b:57:a7 | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.103"} |
    | 41a7b36d-e81f-42d3-b513-acae9797d1f3 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:eb:a5:ad | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.100"} |
    | 68b334d1-0620-437e-ab39-03c6df0e1651 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:14:be:3b | {"subnet_id": "9fbbbb4a-e58f-43cc-a5ff-bf3784526a0e", "ip_address": "192.168.10.100"} |
    | 87bf6c89-c354-4237-be5c-14e03b1c2566 |      | ac9cab947db74c0c999380a4f306693b | 7a:52:00:86:a4:82 | {"subnet_id": "f59be5cb-f8e5-461a-823b-b15e38f1ebef", "ip_address": "192.168.40.14"}  |
    | 92172f4a-f8f3-4fcc-b8c1-76dd809792bd |      | ac9cab947db74c0c999380a4f306693b | 7a:52:00:86:a4:81 | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.109"} |
    | 97607b2b-9703-41a8-b573-b516e755d7e6 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:ba:ae:ad | {"subnet_id": "f59be5cb-f8e5-461a-823b-b15e38f1ebef", "ip_address": "192.168.40.10"}  |
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
    [root@localhost ~(keystone_admin)]#
```

可见7a:52:00:86:a4:81分配到了tenant1的子网dbfcccf2-7c03-4c52-827b-e91180b0e24a
7a:52:00:86:a4:82分配到了tenant2的子网f59be5cb-f8e5-461a-823b-b15e38f1ebef

现在交换ironic port的设置，把81放到physnet2，82放到physnet1：
```
    [root@localhost ~(keystone_admin)]# ironic port-show 42b06aa8-27c0-44d8-9280-e6994dad9c26
    +-----------------------+---------------------------------------------------------------------+
    | Property              | Value                                                               |
    +-----------------------+---------------------------------------------------------------------+
    | address               | 7a:52:00:86:a4:81                                                   |
    | created_at            | 2017-11-29T04:41:54+00:00                                           |
    | extra                 | {}                                                                  |
    | internal_info         | {}                                                                  |
    | local_link_connection | {u'switch_info': u'brbm', u'port_id': u'ovs-node0i0', u'switch_id': |
    |                       | u'8e:df:df:61:61:46'}                                               |
    | node_uuid             | a308bca6-e6a3-4349-b8ea-695e17672898                                |
    | physical_network      | physnet2                                                            |
    | portgroup_uuid        | None                                                                |
    | pxe_enabled           | True                                                                |
    | updated_at            | 2018-06-05T07:47:02+00:00                                           |
    | uuid                  | 42b06aa8-27c0-44d8-9280-e6994dad9c26                                |
    +-----------------------+---------------------------------------------------------------------+
    [root@localhost ~(keystone_admin)]# ironic port-show 44af766f-9e54-423e-9652-7f2122d315b6
    +-----------------------+---------------------------------------------------------------------+
    | Property              | Value                                                               |
    +-----------------------+---------------------------------------------------------------------+
    | address               | 7a:52:00:86:a4:82                                                   |
    | created_at            | 2017-12-22T02:52:53+00:00                                           |
    | extra                 | {}                                                                  |
    | internal_info         | {}                                                                  |
    | local_link_connection | {u'switch_info': u'brbm', u'port_id': u'ovs-node0i1', u'switch_id': |
    |                       | u'8e:df:df:61:61:46'}                                               |
    | node_uuid             | a308bca6-e6a3-4349-b8ea-695e17672898                                |
    | physical_network      | physnet1                                                            |
    | portgroup_uuid        | None                                                                |
    | pxe_enabled           | True                                                                |
    | updated_at            | 2018-06-05T07:47:17+00:00                                           |
    | uuid                  | 44af766f-9e54-423e-9652-7f2122d315b6                                |
    +-----------------------+---------------------------------------------------------------------+
```

再部署一次：
```
    [root@localhost ~(keystone_admin)]# neutron port-list
    neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
    | id                                   | name | tenant_id                        | mac_address       | fixed_ips                                                                             |
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
    | 21e9f776-ee5c-43b4-bb81-bbf0e3f04651 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:8b:57:a7 | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.103"} |
    | 41a7b36d-e81f-42d3-b513-acae9797d1f3 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:eb:a5:ad | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.100"} |
    | 5e90768d-a8a6-4994-a15d-afdd7d405a58 |      | ac9cab947db74c0c999380a4f306693b | 7a:52:00:86:a4:81 | {"subnet_id": "f59be5cb-f8e5-461a-823b-b15e38f1ebef", "ip_address": "192.168.40.13"}  |
    | 68b334d1-0620-437e-ab39-03c6df0e1651 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:14:be:3b | {"subnet_id": "9fbbbb4a-e58f-43cc-a5ff-bf3784526a0e", "ip_address": "192.168.10.100"} |
    | 69fea7df-922a-4b00-b2fa-0e668c18e828 |      | ac9cab947db74c0c999380a4f306693b | 7a:52:00:86:a4:82 | {"subnet_id": "dbfcccf2-7c03-4c52-827b-e91180b0e24a", "ip_address": "192.168.20.106"} |
    | 97607b2b-9703-41a8-b573-b516e755d7e6 |      | ac9cab947db74c0c999380a4f306693b | fa:16:3e:ba:ae:ad | {"subnet_id": "f59be5cb-f8e5-461a-823b-b15e38f1ebef", "ip_address": "192.168.40.10"}  |
    +--------------------------------------+------+----------------------------------+-------------------+---------------------------------------------------------------------------------------+
```

可见7a:52:00:86:a4:81分配到了tenant2的子网f59be5cb-f8e5-461a-823b-b15e38f1ebef
7a:52:00:86:a4:82分配到了tenant1的子网dbfcccf2-7c03-4c52-827b-e91180b0e24a

注意：physical_network影响的是port选择的顺序，不正确或缺失的配置仍然会导致无法达到预期行为。
