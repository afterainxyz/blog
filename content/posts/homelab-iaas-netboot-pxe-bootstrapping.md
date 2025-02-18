---
title: '网络启动的PXE服务器自举'
date: '2025-01-02T16:14:23+08:00'
lastmod: '2025-01-02T16:14:23+08:00'

draft: false

tags: ["homelab", "IaaS", "netboot"]
author: "afterain"
---
### 领域知识

[网络启动]({{< ref "homelab-iaas-netboot" >}})

### 工具链

- netboot-bootstrap.sh

	一个脚本工具，用来自动化安装配置需要的软件和服务。[代码仓库地址](https://github.com/afterainxyz/netboot)

	- bootloader：pxelinux

	- ProxyDHCP：dnsmasq

	- TFTP：dnsmasq

	- HTTP：python

- PXE服务器

	通过bootstrap自动化安装的Linux服务器。自动配置所有必须的软件和服务。包含：

	- 操作系统：ubuntu server 22.04.1

	- 网络：静态IP地址
	
		**注：PXE服务器需要静态IP地址，否则IP变化后，客户端将无法网络启动**

	- DHCP：dnsmasq

	- TFTP：dnsmasq

	- iSCSI：targetcli-fb

	- HTTP：nginx

	- SSH：OpenSSH

#### 前提条件

- Wi-Fi路由器

	- 能访问internet

	- 内置DHCP服务

	- LAN的两个空闲网口

- 个人电脑

	- macOS或Linux操作系统

	- 有线（或无线）连接到Wi-Fi的LAN
 
- PXE服务器

	- BIOS或UEFI支持并配置成网络启动

	- 有线连接到Wi-Fi的LAN

	- 内存 >= 5GB

		加载ubuntu server 22.04.1的iso需要1.5G内存，cloud-init需要3G内存

	- 硬盘 >= 20GB

### 配置代码

以下均已通过工具自动化配置完毕。这里是对配置内容进行说明，以便需要修改时参考。

#### netboot-bootstrap.sh

- 参数

	- `--server-address` CIDR格式，例如192.168.0.6/24

	- `--server-gateway` 例如192.168.0.1

	- `--server-dns` 例如192.168.0.1

- 配置

	以下内容均在`dist`目录

	- ProxyDHCP：dnsmasq => `services/dnsmasq.conf`

	```
		# 禁用DHCP
		port=0
		# 开启日志
		log-dhcp
		# 开启ProxyDHCP
		dhcp-range=192.168.0.255,proxy
	```

	- TFTP

		- dnsmasq => `services/dnsmasq.conf`

			```
				# 设置tftp根目录
				tftp-root="/Users/XXX/homelab/netboot/dist/tftproot"
				# 设置不同硬件启动（BIOS/UEFI）和不同CPU架构（X86-32/64和IA32/64）的bootloader文件
				pxe-service=X86PC,"Boot x86 BIOS",lpxelinux.0
				pxe-service=X86-64_EFI,"PXELINUX (EFI 64)",efi64/syslinux.efi
				pxe-service=IA64_EFI,"PXELINUX (EFI 64)",efi64/syslinux.efi
				pxe-service=IA32_EFI,"PXELINUX (EFI 32)",efi32/syslinux.efi			```

		- pxe配置 => `tftproot/pxelinux.cfg/default`

			指定内核和启动参数的HTTP地址和路径（运行netboot-bootstrap.sh的个人电脑IP地址是192.168.1.2）

			```
				DEFAULT install
				LABEL install
				  KERNEL http://192.168.1.2:8182/casper/vmlinuz
				  INITRD http://192.168.1.2:8182/casper/initrd
				  APPEND root=/dev/ram0 ramdisk_size=1500000 ip=dhcp url=http://192.168.1.2:8182/ubuntu-22.04.1-live-server-amd64.iso autoinstall ds=nocloud-net;s=http://192.168.1.2:8182/cloud-init/			```
		

	- HTTP：python => `httproot`

		PXE服务器的操作系统内核和系统安装配置

		- 系统镜像 => `ubuntu-22.04.1-live-server-amd64.iso`

		- 内核 => `httproot/casper/{vmlinuxz, initrd}`

		- cloud-init => `httproot/cloud-init{meta-data, user-data}`

#### PXE服务器

以下工具均是在cloud-init的user-data文件中指定安装和配置

- 网络：静态IP地址

	```
	  network:
	    version: 2
	    ethernets:
	      eth0:
	        match:
	          name: e*
	        addresses:
	          - 192.168.1.3
	        gateway4: 192.168.1.1
	        nameservers:
	          addresses:
	            - 192.168.1.1
	```

- DHCP：dnsmasq

	- 安装

	```
		#cloud-config
		autoinstall:
		  packages:
		    [dnsmasq]
	```

	- 配置：暂无

- TFTP：dnsmasq

	- 安装

	```
		#cloud-config
		autoinstall:
		  packages:
		    [dnsmasq]
	```

	- 配置：暂无


- iSCSI：targetcli-fb

	- 安装

	```
		#cloud-config
		autoinstall:
		  packages:
		    [targetcli-fb]
	```

	- 配置：暂无


- HTTP：nginx

	- 安装

	```
		#cloud-config
		autoinstall:
		  packages:
		    [nginx]
	```

	- 配置：暂无


- SSH：OpenSSH

	安装ssh服务并禁止密码登录，然后配置了公钥。

	```
		#cloud-config
		autoinstall:
		  ssh:
		    install-server: true
		    allow-pw: false
		    authorized-keys:
		      - -----BEGIN OPENSSH PRIVATE KEY-----
		      xxxxxxxxx
		-----END OPENSSH PRIVATE KEY-----
	```

### 自动化

- 个人电脑

	执行 `./netboot-bootstrap.sh --server-address 192.168.1.3/24 --server-gateway 192.168.1.1 --server-dns 192.168.1.1`

- PXE服务器

	启动并等待自动化安装完成

### netboot-bootstrap.sh工作原理简介

大致流程如下：

- 检查依赖

	如果依赖的命令工具不存在，将提示错误后退出

- 准备发布目录

	当前目录下创建`dist`目录作为发布目录

	```
		dist
		|--services		# dhcp/tftp/http服务的软件和配置的目录，每次执行时清理
		|--tftproot		# tftp文件的根目录，每次执行时清理
		|--httproot		# http文件的根目录，每次执行时清理
		|--download		# 下载文件的保存目录，下载支持断点续传，可以多次反复运行，不清理
		     |--tmp		# 解压/编译等中间过程的临时目录，保留部分软件的中间过程文件
	```

- 安装bootloader

	下载，解压，安装和配置syslinux

- 安装dhcp和tftp服务

	下载，解压，编译，安装和配置dnsmasq

- 安装http服务

	下载，解压，安装ubuntu server live iso，并配置cloud-init
	
	- 网络
		
		需要输入PXE服务器的静态IP地址（必须指定或运行时提示输出）。网关，DNS默认从本机获取配置。
		
		也可以在执行时指定参数`--server-address`（CIDR格式，例如192.168.0.6/24），`--server-gateway`（例如192.168.0.1），`--server-dns`（例如192.168.0.1）
			
	- SSH
		
		默认用户名：ubuntu，密码：ubuntu，执行sudo不需要密码，**虽然只允许SSH密钥方式登录，还是建议安装完成后更改密码**

		PXE服务器的SSH配置为不允许密码登陆，公钥文件默认：`$HOME/.ssh/id_ed25519`，如果文件不存在会提示输入文件路径。
			
		**注：因为只允许密钥方式登陆，如果需要使用其他PC管理PXE服务器，需要同步对应的私钥文件，否则无法SSH登陆PXE服务器。**
	
	- 解压
	
		Linux解压iso需要root权限，执行前会提示输入密码

- 运行dhcp和tftp服务

	运行dnsmasq需要root权限，执行前会提示输入密码

- 运行http服务

	服务端口：8182
