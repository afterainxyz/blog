---
title: '网络启动'
date: '2025-01-02T16:01:08+08:00'
lastmod: '2025-01-02T16:01:08+08:00'

draft: false

tags: ["homelab", "IaaS", "netboot"]
author: "afterain"
---
### 领域知识

#### 什么是网络启动

简单来说是通过网络加载bootloader，然后通过bootloader无盘启动或安装操作系统。其中预启动执行环境PXE（[Preboot eXecution Environment](https://en.wikipedia.org/wiki/Preboot_Execution_Environment)）是现在裸金属硬件支持最好的标准。

#### 初始化网络

由DHCP服务提供的功能。包含2个关键步骤：

- 获取IP地址

	DHCP给设备分配IP地址。设备有了IP地址后，才能正常通过网络加载bootloader。

- 获取bootloader需要的服务器地址和文件路径

	BOOTP（DHCP完全兼容BOOTP）给设备提供：服务器的IP地址，以及bootloader文件路径。有这些信息后才能继续加载bootloader。

如果不方便修改已有的DHCP服务的配置（例如没有权限配置企业中的DHCP服务，或是无法配置硬件路由器内置的DHCP服务），可以额外安装一个独立的[ProxyDHCP](https://wiki.fogproject.org/wiki/index.php?title=ProxyDHCP_with_dnsmasq)服务。

#### 加载bootloader

设备从TFTP服务器中下载bootloader和配置文件，配置文件中描述了启动菜单和每个启动项的操作系统的内核映像文件路径

也可以继续链式加载另一个增强的bootloader（[iPXE](https://ipxe.org/) 或 [GRUB](https://en.wikipedia.org/wiki/GNU_GRUB)等）

#### 无盘启动或安装操作系统

**注：这部分不是PXE提供的功能**

从TFTP/HTTP服务中下载操作系统的内核映像（例如Linux的vmlinuz/initrd）。无盘启动和安装是不同的流程:

- 安装：一般是从HTTP/S网络服务中下载系统镜像(iso)，结合自动应答（[kickstart](https://en.wikipedia.org/wiki/Kickstart_\(Linux\)) 或 [cloud-init](https://en.wikipedia.org/wiki/Cloud-init)）来自动配置和安装操作系统。

- 启动：一般是从NFS/iSCSI网络服务中加载根分区(rootfs)来启动操作系统。

### 工具链和配置

由于运行服务(DHCP和TFTP、HTTP、NFS、iSCSI)的硬件环境有非常大的差异（例如使用一台Windows/Linux服务器、现成的智能路由器、现成的NAS服务器），所以使用的工具不一样，配置也不一样

#### PXE服务器

根据实际情况选择不同的方案

- Linux主机

	网络启动和自动化安装的前提是需要一台运行着多个网络服务（DHCP/TFTP/HTTP/NFS）的服务器。但是这个服务器本身是如何被自动化安装呢?  => [网络启动的PXE服务器自举]({{< ref "homelab-iaas-netboot-pxe-bootstrapping" >}})

- 群辉NAS

	很多软件的安装配置需用通过群辉的UI界面完成，无法实现`一键执行`，更多的是如何操作和配置 => [群辉NAS配置网络启动]({{< ref "homelab-iaas-netboot-pxe-synology" >}})

#### 设备

- [树莓派]({{< ref "homelab-iaas-netboot-raspberrypi" >}})

- [PC]({{< ref "homelab-iaas-netboot-linux-pc" >}})