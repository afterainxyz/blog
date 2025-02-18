---
title: '网络启动的PXE服务器 Synology 群辉'
date: '2025-02-09T17:12:00+08:00'
lastmod: '2025-02-09T17:12:00+08:00'

draft: false

tags: ["homelab", "IaaS", "netboot", "synology"]
author: "afterain"
---
### 领域知识

[网络启动]({{< ref "homelab-iaas-netboot" >}})

### 工具链

- 硬件和系统

	群辉DS216play + DSM 6.1.7
	
- 软件

	- ProxyDHCP

		群辉自带的DHCP服务：dnsmasq

	- TFTP

		群辉自带的TFTP服务：opentftp

	- iSCSI

		群辉自带的`存储空间`支持iSCSI，底层使用的软件未知

### 配置代码

以下配置仅针对树莓派的无盘启动

#### ProxyDHCP

- 启用DHCP

	- 控制面板 => 连接性 => DHCP Server => 网络接口 => 局域网 => 编辑 => DHCP Server

		勾选 `启用DHCP服务器`

- 启用PXE

	- 控制面板 => 连接性 => DHCP Server => PXE

		- 勾选 `启用 PXE（预启动执行环境）`

		- 本地TFTP服务器 => 启动加载项

			填写`start4.elf`

- 配置

	由于群辉DSM的UI界面无法配置ProxyDHCP，需要SSH登录进行配置。下面假设已经使用管理员账号SSH远程登录NAS。
	
	- 三个配置文件：/etc/dhcpd/dhcpd.info，/etc/dhcpd/dhcpd-eth0.info，/etc/dhcpd/dhcpd-eth0-subnet0.info

		内容如下
		
		```
			 enable="yes"
		```		

	- /etc/dhcpd/dhcpd-eth0-subnet0.conf

		内容如下

		```
			interface=eth0
			# 禁用DHCP
			port=0
			# 开启日志
			log-dhcp
			log-facility=/var/log/dnsmasq.log
			# 开启ProxyDHCP
			dhcp-range=192.168.1.0,proxy
			# 配置树莓派bootloader启动项
			pxe-service=X86PC,"Raspberry Pi Boot"
		```

	- 重启DHCP服务

		执行`sudo /etc/rc.network nat-restart-dhcp`

		**注意：手工修改重启后，只能通过配置文件修改对应参数，无法在UI界面配置**

#### TFTP

- 开启服务

	- 控制面板 => 文件共享 => 文件服务 => TFTP

		勾选 `启用TFTP服务器`

- 配置

	- 控制面板 => 文件共享 => 文件服务 => TFTP

		- TFTP根文件夹

			点选择，然后可以在根目创建一个 `tftproot`目录

#### iSCSI

存储空间管理员 => iSCSI LUN => 新增

- iSCSI LUN（文件级）

	- 名称：填写树莓派的设备Id，例如 7092246e

	- 容量：树莓派建议32G以上

	- iSCSI Target 链接：选择`新增一个 iSCSI target`

	记下`IQN`，例如`iqn.2000-01.com.synology:nas-afterainxyz.Target-1.2be99b1442`，后续配置树莓派需要

### 自动化

- 群辉DS216play

	软件的安装配置需用通过群辉DSM的UI界面完成，无法实现`一键执行`，更多的是如何操作和配置。

- 树莓派

	详细操作和配置参考[树莓派]({{< ref "homelab-iaas-netboot-raspberrypi" >}})