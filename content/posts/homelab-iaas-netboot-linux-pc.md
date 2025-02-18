---
title: '网络启动安装LinuxPC'
date: '2025-02-09T21:59:00+08:00'
lastmod: '2025-02-09T21:59:00+08:00'

draft: false

tags: ["homelab", "IaaS", "netboot"]
author: "afterain"
---
### 领域知识

[网络启动]({{< ref "homelab-iaas-netboot" >}})

### 工具链

- pxe-server.sh

	[PXE服务器]({{< ref "homelab-iaas-netboot-pxe-bootstrapping" >}})上运行的一个脚本工具，用来自动化配置需要的软件和服务。[代码仓库地址](https://github.com/afterainxyz/netboot)

	- DHCP：dnsmasq

	- TFTP：dnsmasq

	- HTTP：nginx

- Linux PC

	通过[PXE服务器]({{< ref "homelab-iaas-netboot-pxe-bootstrapping" >}})自动化安装的Linux服务器。自动配置所有必须的软件和服务。包含：

	- 操作系统：ubuntu server 22.04.1

	- 网络：静态IP地址
	
	- SSH：OpenSSH

#### 前提条件

- Wi-Fi路由器

	- 能访问internet

	- 内置DHCP服务

	- LAN的两个空闲网口

- PXE服务器

	- 有线连接到Wi-Fi的LAN
 
- Linux PC

	- BIOS或UEFI支持并配置成网络启动

	- 有线连接到Wi-Fi的LAN

	- 内存 >= 5GB

		加载ubuntu server 22.04.1的iso需要1.5G内存，cloud-init需要3G内存

	- 硬盘 >= 20GB

### 配置代码

以下均已通过工具自动化配置完毕。这里是对配置内容进行说明，以便需要修改时参考。

#### pxe-server.sh

- 运行模式

	- probe

		探测`设备Id`，这样可以给不同的设备安装不同的软件，配置不同的参数。

	- add

		添加设备参数：`<device_id> <options>`

		- 参数`device_id`

			探测阶段获取的`设备Id`，例如：
			
			- PC：01-00-0c-29-4f-62-c7 (MAC地址)

			- 树莓派：7092246e (设备序列号)

		- 参数`options`有以下几个选项

			- `--node-address` CIDR格式，例如192.168.0.6/24

			- `--node-gateway` 例如192.168.0.1

			- `--node-dns` 例如192.168.0.1

- 配置

	- ProxyDHCP：dnsmasq => `/etc/dnsmasq.conf`

	```
		# 禁用DHCP
		port=0
		# 开启日志
		log-dhcp
		# 开启ProxyDHCP
		dhcp-range=192.168.0.255,proxy
	```

	- TFTP

		- dnsmasq => `/etc/dnsmasq.conf`

			```
				# 设置tftp根目录
				tftp-root="/var/homelab/tftproot"
				# 设置不同硬件启动（BIOS/UEFI）和不同CPU架构（X86-32/64和IA32/64）的bootloader文件。注意：因为树莓派的特殊性，以下的顺序能同时兼容PC和树莓派
				pxe-service=X86PC,"Boot x86 BIOS",lpxelinux.0
				pxe-service=X86PC,"Raspberry Pi Boot"
				pxe-service=X86-64_EFI,"PXELINUX (EFI 64)",efi64/syslinux.efi
				pxe-service=IA64_EFI,"PXELINUX (EFI 64)",efi64/syslinux.efi
				pxe-service=IA32_EFI,"PXELINUX (EFI 32)",efi32/syslinux.efi
				pxe-prompt="PXE",0
			```

		- pxe配置 => `/var/homelab/tftproot/pxelinux.cfg/01-00-0c-29-4f-62-c7`

			指定内核和启动参数的HTTP地址和路径（运行pxe-server的PXE服务器IP地址是192.168.1.3）

			```
				DEFAULT install
				LABEL install
				  KERNEL http://192.168.1.3:12345/01-00-0c-29-4f-62-c7/casper/vmlinuz
				  INITRD http://192.168.1.3:12345/01-00-0c-29-4f-62-c7/casper/initrd
				  APPEND root=/dev/ram0 ramdisk_size=1500000 ip=dhcp url=http://192.168.1.3:12345/01-00-0c-29-4f-62-c7/ubuntu-22.04.1-live-server-amd64.iso autoinstall ds=nocloud-net;s=http://192.168.1.3:12345/01-00-0c-29-4f-62-c7/cloud-init/
			```

	- HTTP：nginx

		PXE服务器的操作系统内核和系统安装配置

		```
			server {
				listen 12345 default_server;
				listen [::]:12345 default_server;
				root /var/homelab/httproot;
				server_name _;
			}
		```

#### Linux PC

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
	          - 192.168.1.4
	        gateway4: 192.168.1.1
	        nameservers:
	          addresses:
	            - 192.168.1.1
	```

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

- PXE服务器

	- 探测设备Id

		执行 `./pxe-server.sh probe`

	- 添加设备

		执行 `./pxe-server.sh add --node-address 192.168.1.4/24 --node-gateway 192.168.1.1 --node-dns 192.168.1.1`

- Linux PC

	- 探测设备Id

		BIOS或UEFI设置为网络启动后重启机器，等待`pxe-server.sh`探测PC的`device_id`

	- 添加设备

		执行`./pxe-server.sh add xxx`后，重启PC等待网络启动和自动化安装完成

### pxe-server.sh工作原理简介

大致流程如下：

- 检查依赖

	如果依赖的命令工具不存在，将提示错误后退出

- 准备发布目录

	当前目录下创建`dist/download`目录作为下载目录

	```
		dist
		|--download		# 下载文件的保存目录，下载支持断点续传，可以多次反复运行，不清理
		     |--tmp		# 解压/编译等中间过程的临时目录
	```

	创建`/var/homelab`目录作为服务根目录

	```
		/var/homelab
		|--tftproot		# tftp服务根目录，包含bootloader和配置
		|--httproot		# http服务根目录，包含操作系统iso和cloud-init配置
		|--iscsiroot		# iscsi服务根目录，包含无盘设备的磁盘镜像文件，例如树莓派
	```

- 检查服务状态

	PXE相关服务（proxyDHCP+tftp:dnsmasq和http:nginx）是否安装并启动，如果未启动则尝试启动

- 配置服务

	- proxyDHCP和tftp

		dnsmasq设置为proxyDHCP模式，开启tftp服务，指定tftp根目录并配置bootloader文件路径。

	- http

		删除nginx默认的`server`配置，新增`server`（设置根目录和端口：12345）

- 安装bootloader

	下载，解压，安装syslinux到tftp根目录

- 探测设备

	读取并分析tftp日志，根据[pxelinux文档](https://wiki.syslinux.org/wiki/index.php?title=PXELINUX)，提取mac地址作为唯一的`device_id`

- 添加设备

	下载，解压，安装ubuntu server live iso到http目录，配置cloud-init（包含主机名/用户密码/网络/SSH/防火墙）

	- 主机名

		为了避免冲突，把`device_id`作为主机名

	- 用户密码

		默认用户名：ubuntu，密码：ubuntu，执行sudo不需要密码，**虽然只允许SSH密钥方式登录，还是建议安装完成后更改密码**
			
	- 网络
			
		需要输入PC节点的静态IP地址。网关，DNS默认从本机获取配置。
			
		也可以在执行时设置参数`--node-address`（CIDR格式，例如192.168.0.6/24），`--node-gateway`（例如192.168.0.1），`--node-dns`（例如192.168.0.1）
				
		**注：为了方便IaaS的管理，需要节点有固定不变的静态IP地址**
				
	- SSH
			
		PC节点的SSH配置为不允许密码登陆，公钥文件默认是`/home/ubuntu/.ssh/authorized_keys`的第一个key，如果文件不存在会提示输入正确的文件路径。
				
		**注：因为只允许密钥方式登陆，PXE服务器的公钥是在bootstrap过程时配置的，所以需要使用bootstrap的私钥才能SSH登陆到PC节点**

	- 防火墙

		[K3S只支持旧版iptables](https://docs.rancher.cn/docs/k3s/advanced/_index/#raspberry-pi-os-setup-的额外准备工作)，从默认安装的`nftables`切换到`iptables`

