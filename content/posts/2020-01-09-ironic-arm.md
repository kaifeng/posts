---
layout: post
title: ironic和ironic-inspector支持ARM裸机
author: kaifeng
date: 2020-01-09
categories: ironic
---

ironic对于多架构的支持还是不错的，而ironic-inspector的dnsmasq服务则需要用户自行配置一下。

要支持ARM裸机的部署，首先需要制作出支持ARM的部署镜像，目前ironic已经迁移到使用ironic-python-agent-builder来制作镜像，
diskimage-builder中的ironic-agent element则已经作废。

# ipa-b编译aarch64镜像

编译环境需要先安装一些依赖包：kpartx, qemu-img, PyYAML, git, policycoreutils-python (semanage)

安装ipa-b：
```
pip install --user ironic-python-agent-builder
```

编译脚本如下：
```
#!/bin/bash
BUILDROOT=$(readlink -e .)
export DIB_DEBUG_TRACE=1
export DIB_AVOID_PACKAGES_UPDATE=1
export ARCH=aarch64
#export DIB_LOCAL_IMAGE=$BUILDROOT/Fedora-Cloud-Base-31-1.9.aarch64.raw.xz
export DIB_INSTALLTYPE_pip_and_virtualenv=package
DIB_DISTRIBUTION_MIRROR=https://mirrors.tuna.tsinghua.edu.cn/fedora
export DIB_PYPI_MIRROR_URL=https://pypi.tuna.tsinghua.edu.cn/simple
export DIB_DEV_USER_USERNAME=ironic
export DIB_DEV_USER_PASSWORD=ironic
export DIB_DEV_USER_PWDLESS_SUDO=yes
# More secure way is to use pubkey
# export DIB_DEV_USER_AUTHORIZED_KEYS=$HOME/.ssh/id_rsa.pub
ironic-python-agent-builder -o ironic-deploy-fedora -e pip-cache -e devuser -e dynamic-login fedora
```

因为国内镜像源基本没有提供aarch64的源，从国外源下载比较慢，可以先下载基础镜像到本地，然后通过DIB_LOCAL_IMAGE指定本地镜像，避免重复下载。
当然，DIB本身也支持本地缓存。此外，连接yum源和pypi源的网络不是很好，基本上要反复多次才能下载完成需要的包。

NOTE: 制作centos7镜像最后会报找不到内核，这是基础镜像的问题，因此这里使用了Fedora版本。

# ironic-inspector的dnsmasq配置

ARM服务器只支持UEFI启动，使用PXE引导时需要通过grub，启动文件为grubaa64.efi，可以从官方发行版ISO的BOOT/EFI路径下获取。

配置/etc/ironic-inspector/dnsmasq.conf，将aarch64 DHCP请求导向使用grubaa64.efi：
```
# This is the recommend minimum for using introspection
port=0
bind-interfaces
interface=enp11s0f0

enable-tftp

dhcp-match=aarch64, option:client-arch, 11 # aarch64 efi
dhcp-boot=tag:aarch64, grubaa64.efi
dhcp-range=192.168.200.100,192.168.200.200
tftp-root=/tftpboot
dhcp-sequential-ip
```

这里启用dnsmasq的tftp，可以方便查看grubaa64.efi查找配置文件的路径。例如下面的引导记录：
```
    Nov 27 16:12:08 arm-computer07 dnsmasq-tftp[15417]: sent /tftpboot/grubaa64.efi to 192.168.200.101
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-01-00-07-3e-92-a9-a8- not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-C0A8C865 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-C0A8C86 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-C0A8C8 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-C0A8C not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-C0A8 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-C0A not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-C0 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg-C not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/grub.cfg not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-01-00-07-3e-92-a9-a8- not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-C0A8C865 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-C0A8C86 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-C0A8C8 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-C0A8C not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-C0A8 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-C0A not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-C0 not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/grub.cfg-C not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: sent /tftpboot/EFI/BOOT/grub.cfg to 192.168.200.101
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: sent /tftpboot/EFI/BOOT/grub.cfg to 192.168.200.101
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/arm64-efi/command.lst not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/arm64-efi/fs.lst not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/arm64-efi/crypto.lst not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: file /tftpboot/EFI/BOOT/arm64-efi/terminal.lst not found
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: sent /tftpboot/EFI/BOOT/grub.cfg to 192.168.200.101
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: sent /tftpboot/EFI/BOOT/grub.cfg to 192.168.200.101
    Nov 27 16:12:09 arm-computer07 dnsmasq-tftp[15417]: sent /tftpboot/EFI/BOOT/grub.cfg to 192.168.200.101
```

由于ironic-inspector主机发现并没有MAC和IP的信息，因此只能提供默认配置，比如可以将grub配置放在/tftpboot/grub.cfg或者/tftpboot/EFI/BOOT/grub.cfg

> grub查找的位置和文件名与grub实现有关，不同架构也可能存在差异

> grub.cfg-01-00-07-3e-92-a9-a8- 多一个后缀是grub bug，后续版本应该会移除末尾的-。

grub.cfg可以简单地写为：
```
set default="1"
set timeout=5

menuentry 'Introspection for aarch64' {
    linux ironic-agent.kernel text showopts selinux=0 ipa-inspection-callback-url=http://{IP}:5050/v1/continue ipa-inspection-collectors=default ipa-collect-lldp=1 systemd.journald.forward_to_console=no
    initrd ironic-agent.initramfs
}
```

将IP替换为inspector服务本机地址。

## 异架构裸机管理

如果环境中存在多种不同架构的裸机，则不能通过一个grub.cfg配置，此时可以借用ironic的思路，使用一个主grub.cfg配置入口，
通过grub变量的方式引导到不同的grub配置，与架构有关的grub参数是$grub_cpu。

主grub.cfg可以写为：
```
set default=master
set timeout=5
set hidden_timeout_quiet=false

menuentry "master"  {
configfile /tftpboot/grub-${grub_cpu}.cfg
}
```

对于grubaa64.efi，它的grub_cpu是arm64，对于x86服务器则是i386，那么就需要编写两套grub.cfg文件，grub-arm64.cfg存放ARM相关配置，grub-i386.cfg存放x86相关配置。

> grub参考：https://www.gnu.org/software/grub/manual/grub/html_node/grub_005fcpu.html

# ironic的配置

ironic通过pxe_bootfile_name_by_arch和pxe_config_template_by_arch来支持多架构。ARM裸机在ironic节点的cpu_arch为aarch64，
ironic会优先从arch相关的配置项中读取配置，可以为ARM裸机指定引导文件及配置模板：
```
[pxe]
pxe_bootfile_name_by_arch = aarch64:grubaa64.efi
pxe_config_template_by_arch = aarch64:/tftpboot/grub-aarch64.template
```

grub模板可以沿用grub_pxe_config.template，可以根据需要调整，比如grubaa64.efi不支持linuxefi和initrdefi命令，需要改成使用linux和initrd：
```
set default=deploy
set timeout=5
set hidden_timeout_quiet=false

menuentry "deploy"  {
    linux {{ pxe_options.deployment_aki_path }} selinux=0 troubleshoot=0 text {{ pxe_options.pxe_append_params|default("", true) }} boot_server={{pxe_options.tftp_server}}
    initrd {{ pxe_options.deployment_ari_path }}
}

menuentry "boot_partition"  {
    linux {{ pxe_options.aki_path }} root={{ ROOT }} ro text {{ pxe_options.pxe_append_params|default("", true) }} boot_server={{pxe_options.tftp_server}}
    initrd {{ pxe_options.ari_path }}
}

menuentry "boot_ramdisk"  {
    linux {{ pxe_options.aki_path }} root=/dev/ram0 text {{ pxe_options.pxe_append_params|default("", true) }} {{ pxe_options.ramdisk_opts|default('', true) }}
    initrd {{ pxe_options.ari_path }}
}

menuentry "boot_whole_disk"  {
    linux chain.c32 mbr:{{ DISK_IDENTIFIER }}
}
```

剩下的就可以交给ironic处理了。
