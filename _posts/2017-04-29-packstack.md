---
layout: post
title: packstack
author: kaifeng
date: 2017-04-29
lastmod: 2018-03-24
categories: openstack
---

packstack(www.rdoproject.org) 是 Red Hat 的布署工具，基于puppet实现，适合稳定版本的部署。建议内存不少于8G。

以 CentOS 7 为例，先安装系统，安装完成后，根据需要修改 yum 源，配置网络和静态IP地址等。
Minimum版本没有ifconfig命令，若不熟悉ip命令，可以安装net-tools包。

安装openstack元数据包 `centos-release-openstack-<release>`，release对应不同的openstack发行版，比如pike。

装完后，在/etc/yum.repos.d里多出几个源，根据需要修改镜像源。执行 yum update -y 更新系统。

建议关闭防火墙及NetworkManager服务:
```shell
systemctl disable firewalld
systemctl stop firewalld
systemctl disable NetworkManager
systemctl stop NetworkManager
systemctl enable network
systemctl start network
```

SELinux也可以禁用。

安装 packstack:
```shell
yum install openstack-packstack
```

快速部署虚机单节点环境用 packstack --allinone 即可，packstack还提供了许多命令行参数，适合脚本操作。大多数情况下需要对配置进行手工调整，通过answerfile的方式更为合适。
通过 packstack 生成 answerfile:
```shell
packstack --gen-answer-file=<filename>
```

修改answerfile中的配置，然后指定answerfile启动安装:
```shell
packstack --answer-file <filename>
```

安装完成，用户根目录下会生成 keystonerc_admin 用于CLI。

> 若packstack命令行不指定 `--provision-demo=n`，或answerfile中 `CONFIG_PROVISION_DEMO=y`，安装过程会下载 cirrors 映像，可以安装完毕自行添加。

> puppet安装在`/usr/share/openstack-puppet`，供打patch参考。

> Q版本运行packstack报错找不到pkg_resource，安装python2-setuptools

> 可参考 `puppet-<project>` 的代码如何实现组件的配置，如puppet-ironic中关于pxe的配置。
