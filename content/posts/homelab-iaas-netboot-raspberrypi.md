---
title: '网络启动的无盘系统树莓派'
date: '2025-02-09T21:58:54+08:00'
lastmod: '2025-02-09T21:58:54+08:00'

draft: false

tags: ["homelab", "IaaS", "netboot", "树莓派"]
author: "afterain"
---
### 领域知识

[网络启动]({{< ref "homelab-iaas-netboot" >}})

### 工具链

- 硬件

	树莓派4b 4G
	
### 配置代码

#### 前提条件

- PXE服务器

	[网络启动的群辉]({{< ref "homelab-iaas-netboot-pxe-synology" >}})

- 个人电脑

	- SD读卡器

- 树莓派节点

	- 有线连接到和PXE服务器相同的LAN

	- SD卡，键盘，显示器，HDMI线各一个（设置网络启动方式时需要）

#### 树莓派

- 设置网络启动

	使用SD卡的安装方法，参考[安装操作系统](https://www.raspberrypi.com/documentation/computers/getting-started.html#installing-the-operating-system)。登陆后的操作流程参考[客户端配置文档](https://www.raspberrypi.com/documentation/computers/remote-access.html#client-configuration)
		
	安装的操作系统需要和PXE服务器提供的相同（Raspberry Pi OS 2022-09-06-raspios-bullseye-arm64-lite.img），否则后续步骤生成initramfs时无法匹配内核版本会导致无法正常启动

- 获取`device_id`

	`grep Serial /proc/cpuinfo | cut -d ' ' -f 2 | cut -c 9-16`
		
	另外，也可以使用`pxe-server.sh`的`probe`命令自动探测`device_id`

- 生成支持iscsi的initramfs

	```
		sudo apt install open-iscsi initramfs-tools
		sudo touch /etc/iscsi/iscsi.initramfs
		sudo update-initramfs -v -k "$(uname -r)" -c
	```

	在`/boot`目录下有一个新文件：`initrd.img-5.15.61-v8+` （版本和内核版本相同）
	
- 配置tftp

	- 创建目录

		PXE服务器的tftp根目录创建子目录，目录名是树莓派设备的`device_id`，例如 `7092246e`

	- 复制文件

		复制树莓派`/boot`目录下所有文件到tftp的子目录

- 配置无盘启动

	iSCSI详细配置参考[[HowTo] Booting from iSCSI](https://forums.raspberrypi.com/viewtopic.php?t=151302)

	- cmdline.txt

		树莓派bootloader读取/`TFTP根目录`/`serial number目录`/`cmdline.txt文件`获取内核启动参数
		
		在树莓派原有内容基础上，增加iSCSI内容如下（IQN/LUN来自iSCSI配置，IQN `iqn.2000-01.com.synology:nas-afterainxyz.Target-1.2be99b1442`，LUN `7092246e`，`192.168.1.4`是指定树莓派的IP地址，`192.168.1.2`是iSCSI服务IP地址）
		
		```
			ip=192.168.1.4::192.168.1.1:255.255.255.0:7092246e:eth0:off
			ISCSI_INITIATOR={IQN}-{LUN}
			ISCSI_TARGET_NAME={IQN}
			ISCSI_TARGET_IP=192.168.1.2
			ISCSI_TARGET_PORT=3260
		```
		
		增加容器需要的内核参数（iSCSI无盘启动不是必须，只是为后续安装容器做准备）
		
		```
			cgroup_memory=1 cgroup_enable=memory
		```
		
		完整例子如下
		
		```
			console=serial0,115200 console=tty1 root=UUID=e84ca015-c09a-4fa2-a17b-ba7edda6e353 rootfstype=ext4 ip=192.168.1.4::192.168.1.1:255.255.255.0:7092246e:eth0:off rw rootwait elevator=deadline fsck.repair=yes cgroup_memory=1 cgroup_enable=memory ISCSI_INITIATOR=iqn.2000-01.com.synology:nas-afterainxyz.Target-1.2be99b1442-6059186e ISCSI_TARGET_NAME=iqn.2000-01.com.synology:nas-afterainxyz.Target-1.2be99b1442 ISCSI_TARGET_IP=192.168.1.2 ISCSI_TARGET_PORT=3260
		```		

	- config.txt

		保持树莓派原有内容，修改initramfs内容如下
		
		```
			[all]
			initramfs initrd.img-5.15.61-v8+ followkernel
		```

- 取下SD卡后重启，等待网络启动完成

### 自动化

有2个原因导致树莓派的准备工作无法自动化：
		
- 设置网络启动

	树莓派设置从网络启动需要先有一个正常运行的操作系统。所以还是需要先使用SD卡的安装方法，参考[安装操作系统](https://www.raspberrypi.com/documentation/computers/getting-started.html#installing-the-operating-system)。登陆后的操作流程参考[客户端配置文档](https://www.raspberrypi.com/documentation/computers/remote-access.html#client-configuration)
		
	**注意：如果PXE服务器使用`pxe-server.sh`启动，安装的系统要相同（Raspberry Pi OS 2022-09-06-raspios-bullseye-arm64-lite.img），否则后续步骤生成initramfs时无法匹配内核版本会导致无法正常启动**

- 支持iSCSI

	由于NFS的网络磁盘不支持overlayfs文件系统，所以后续无法正常安装容器，使用iSCSI网络磁盘则不存在这个问题。
			
	树莓派官方提供的内核并不支持iSCSI，需要使用`update-initramfs`工具来制作，然而`update-initramfs`需要在树莓派机器上执行才能生成正确的initramfs文件

