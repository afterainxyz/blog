---
title: "安装配置RaspberryPi"
date: 2022-09-29T10:04:39+08:00
draft: false
tags: ['树莓派']
categories: ['技术']
featuredImage: /img/posts/raspberrypi.png
---

### 安装系统

- 下载 [操作系统镜像](https://www.raspberrypi.com/software/operating-systems/)

	树莓派有很多发行版，官方支持的`Raspberry Pi OS`就有多个版本（桌面版本，桌面版本+推荐软件，精简版）可供选择。场景是用树莓派做服务器，不连接显示器/键盘/鼠标，所以选择的精简版。

<!--more-->

- 下载安装程序[`Raspberry Pi Imager`](https://www.raspberrypi.com/software/)

	安装程序运行在PC电脑上，支持在windows/Ubuntu/macOS多种操作系统下使用。

- 安装

	`Raspberry Pi Imager`只要3步就能把操作系统写入SD卡：选择 操作系统镜像 -> 选择 SD卡 -> 写入

- 分区

	安装程序在SD卡上创建了两个分区：boot(启动)分区和root(系统)根分区

	- boot包含了启动最关键的linux内核文件。分区是vfat格式。现在常用的PC操作系统windows/Linux/macOS都能直接访问（读写文件）

	- root包含了除内核之外，操作系统的各种文件（二进制/配置等），分区是ext4格式。Linux各种发行版默认都支持ext4格式，但是大家常用的windows/macOS系统PC电脑是无法直接访问的（需要安装特殊软件才支持）。

	所以安装完成后，如果把SD卡重新插入PC电脑，windows/macOS只能看到一个boot磁盘。

### 无显示器启动时的配置机制

其实在安装系统完成后，只需要把SD卡插入树莓派中，接通电源即可启动系统使用了。不过由于不接显示器/键盘/鼠标，所以需要在启动前先配置好网络和SSH服务（远程登录控制），否则无法使用。

由于大家常用的windows/macOS系统PC电脑无法直接访问root分区，也就无法修改配置文件。这时树莓派提供了一个有趣的方案，只要在boot分区按照规范的文件名和格式创建好相关配置，就会在启动时自动加载。具体支持哪些，参考官方[配置文档](https://www.raspberrypi.com/documentation/computers/configuration.html#the-boot-folder)。

后面的wifi配置和SSH服务都使用了这个机制。

### 配置wifi网络

- 配置文件

	在boot磁盘的根目录创建文件`wpa_supplicant.conf` ,内容如下：

		country=CN
		ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
		update_config=1

		network={
		  ssid="xxx"
		  psk="yyy"
		  key_mgmt=WPA-PSK
		  priority=1
		}

	根据实际情况修改ssid（wifi网络名）和psk（wifi密码）两个参数即可。其实key_mgmt也需要根据实际情况修改，不过考虑安全性，家庭wifi一般都推荐使用WPA-PSK。

- 如何获取IP地址

	启动树莓派后，登录路由器，查看客户端名称是`raspberrypi`的IP地址即可。

	最好使用路由器提供的“手工指定IP地址”的方法，给树莓派一个固定的IP。一般需要先在路由器查询已DHCP分配的客户端列表，找到`raspberrypi`对应的MAC地址，然后进行设置。

	这样树莓派就可以作为固定IP地址的服务器使用了。

### 配置SSH服务

在boot磁盘的根目录创建文件`ssh`或`ssh.txt`，文件内容为空即可。这样树莓派启动时会开启SSH服务，默认的账号是`pi`，密码是`raspberry`。

### 更改apt源

SSH`ssh pi@xxx.xxx.xxx.xxx`登录树莓派，输入默认密码`raspberry`

修改文件`etc/apt/sources.list`（改为清华提供的源），内容如下：

	deb http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib
	deb-src http://mirrors.tuna.tsinghua.edu.cn/raspbian/raspbian/ buster main non-free contrib

执行`apt-get update`更新源信息
