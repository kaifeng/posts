---
layout: post
title: 通过虚拟磁盘制作Windows镜像
author: kaifeng
date: 2017-02-24
categories: imaging
---

## 环境要求

这里使用 Windows Server 2012 R2 作为镜像制作环境。也可使用 Windows 7, 8 等 Windows 系统作为镜像制作环境，
使用 Windows 7 时，需安装微软提供的AIK/ADK 镜像工具。

## 创建虚拟磁盘文件

1. 打开 Server Manager -> Tools -> Computer Management
2. 左侧导航栏选择 Storage -> Disk Management
3. 菜单 Action -> Create VHD
4. 选择目标 VHD 文件位置
5. 选择硬盘大小，如10G
6. 选择 VHD 格式。用户可以选择 VHD 或 VHDX，VHD 最大支持 2T 磁盘，2T 以上空间请选择 VHDX。
7. 选择 VHD 类型，固定大小或动态扩展均可，推荐 Dynamic expanding。
8. VHD文件创建后，会自动挂载到系统，若没有自动挂载，可通过菜单 Action
   -> Attach VHD 选择 VHD 文件挂载。

## 磁盘分区

进入命令行或 PowerShell，执行 diskpart 工具。list disk 列出磁盘，根据实际情况选择刚创建的虚拟磁盘进行分区操作。
Windows 支持 BIOS/MBR 和UEFI/GPT 两种模式，用户根据硬件实际环境，确认合适的分区方式。

下面是两种分区类型的操作示例，其中，卷标 S 分配给系统分区，卷标 W 分配
给 Windows 分区，卷标 R 分配给恢复分区，后面的示例使用此卷标约定。

按照微软推荐的典型分区，BIOS/MBR 分区操作举例如下:
```
    rem 选择磁盘，根据实际情况选择，可通过 list disk 查看磁盘列表
    select disk 0
    rem 清空磁盘
    clean
    rem 创建系统分区，使用 MBR 时，可以不创建系统分区
    create partition primary size=100
    format quick fs=ntfs label="System"
    assign letter="S"
    rem 设置分区为可启动
    active
    rem 创建 Windows 分区
    create partition primary
    rem 根据需要，缩减分区大小，用户也可在上一句指定分区大小
    rem 缩减的空间可用于数据分区
    rem 在本例中，缩减 500M 用于创建恢复分区
    shrink minimum=500
    rem 格式化 Windows 分区
    format quick fs=ntfs label="Windows"
    assign letter="W"
    rem 创建恢复分区
    create partition primary
    format quick fs=ntfs label="Recovery"
    assign letter="R"
    set id=27
    exit
```

按照微软推荐的典型分区，UEFI/GPT 分区操作举例如下:
```
    rem 选择磁盘，根据实际情况选择，可通过 list disk 查看磁盘列表
    select disk 0
    rem 清空磁盘
    clean
    rem 转换磁盘为 GPT 分区格式
    convert gpt
    rem 创建 100M 的 EFI 分区
    create partition efi size=100
    rem 注意，需格式化为 FAT32
    format quick fs=fat32 label="System"
    assign letter="S"
    rem 创建 MSR(微软保留) 分区
    create partition msr size=16
    rem 创建 Windows 分区
    create partition primary
    rem 根据需要，缩减分区大小，用户也可在上一句指定分区大小
    rem 缩减的空间可用于数据分区
    rem 在本例中，缩减 500M 用于创建恢复分区
    shrink minimum=500
    rem 格式化 Windows 分区
    format quick fs=ntfs label="Windows"
    assign letter="W"
    rem 创建恢复分区
    create partition primary
    format quick fs=ntfs label="Recovery tools"
    assign letter="R"
    set id="de94bba4-06d1-4d40-a16a-bfd50179d6ac"
    gpt attributes=0x8000000000000001
    exit
```

> 使用 UEFI/GPT 方式，必须创建系统分区用于EFI引导。

## 安装Windows

打开命令行工具或 PowerShell 输入命令:

    dism /Apply-Image /ImageFile:<install.wim> /Index:1 /ApplyDir:W:\

* install.wim是Windows安装镜像Source目录里install.wim文件的位置。
* Index：指定 WIM 包内的镜像索引号，比如一个 Windows Server 2012 R2 WIM
  文件包含了Standard Core, Standard, Datacenter Core, Datacenter四种镜
  像。可以通过 dism 工具查看:
  ```
  dism /Get-ImageInfo /ImageFile:<install.wim>
  ```
* ApplyDir：指定目标 Windows 分区。

执行命令后，Windows 文件被写入目标分区，写入时间取决于磁盘速度，大约需要 10 分钟。

## 安装引导程序

如果采用了系统分区加Windows分区的方案，请将引导程序安装在系统分区，如:

    W:\Windows\System32\bcdboot W:\Windows /s S: /f <firmware>

fireware 指定固件类型，根据实际情况选择 UEFI, BIOS 或直接选择 ALL。

BIOS/MBR：

  a. 仅 Windows 分区:

      W:\Windows\System32\bcdboot W:\Windows

  b. Windows 分区 + 系统分区:

      W:\Windows\System32\bcdboot W:\Windows /s S:

GPT:
```
W:\Windows\System32\bcdboot W:\Windows /s S: /f UEFI
```

注意：制作Windows7镜像发现其自带的bcdboot不支持/f参数，而Windows 2012R2支持。
实验用Windows 2012R2自带的bcdboot设置Windows7镜像的UEFI是可行的。

## 设置恢复工具（可选）

如果分区方案包含恢复分区，则安装恢复工具:
```
md R:\Recovery\WindowsRE
copy W:\Windows\System32\Recovery\winre.wim R:\Recovery\WindowsRE\winre.wim
W:\Windows\System32\reagentc /setreimage /path R:\Recovery\WindowsRE /target W:\Windows
```

## 缷载和验证

回到 Disk Management，右键选择虚拟磁盘，选择 Detach VHD，保存磁盘。
镜像制作完成后，可进一步进行定制化修改，如需验证镜像，可以使用制作出的磁盘镜像创建虚拟机并运行。
