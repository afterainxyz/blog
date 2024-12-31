---
title: "在树莓派上运行Kubernetes"
subtitle: ""
date: 2022-10-26T18:13:52+08:00
lastmod: 2022-10-26T18:13:52+08:00
draft: false
author: ""
authorLink: ""
description: ""
license: ""
images: []

tags: ['Kubernetes', '树莓派']
categories: ['技术']

featuredImage: ""
featuredImagePreview: ""

hiddenFromHomePage: false
hiddenFromSearch: false
twemoji: false
lightgallery: true
ruby: true
fraction: true
fontawesome: true
linkToMarkdown: true
rssFullText: false

toc:
  enable: true
  auto: true
code:
  copy: true
  maxShownLines: 50
math:
  enable: false
  # ...
mapbox:
  # ...
share:
  enable: true
  # ...
comment:
  enable: true
  # ...
library:
  css:
    # someCSS = "some.css"
    # located in "assets/"
    # Or
    # someCSS = "https://cdn.example.com/some.css"
  js:
    # someJS = "some.js"
    # located in "assets/"
    # Or
    # someJS = "https://cdn.example.com/some.js"
seo:
  images: []
  # ...
---

树莓派的资源有限，不适合安装标准版的kubernetes，推荐在受限资源环境下使用的[k3s](https://k3s.io)，k3s是经过官方认证，完全兼容标准。

<!--more-->

### 安装k3s

只有一个设备，只需安装简单配置的单机版即可，不需要安装更复杂配置的高可用版。详细完整的文档参考[官方安装文档](https://rancher.com/docs/k3s/latest/en/installation/)

- 前置依赖

	使用`Raspberry Pi OS`时，有几个前置依赖需要先设置：

	- 从nftables切换到iptables

		执行命令

			sudo iptables -F
			sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
			sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy

	- 启用cgroups

		修改`/boot/cmdline.txt`，在现有内容的最后增加：

		`cgroup_memory=1 cgroup_enable=memory`

- 安装

	k3s默认会安装为系统服务，重启也确保服务自动启动

	- 最简方法

		执行`curl -sfL https://get.k3s.io | sh -`

	- 在线（国内加速）安装

		 由于k3s安装时是从github下载文件，没有安装翻墙代理时无法正常下载安装，参考[官方（中文）文档](https://docs.rancher.cn/docs/k3s/quick-start/_index/)，提供了国内加速安装的方法，很好的解决访问github需要翻墙问题。
			 
		 `curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn sh -`


	- 离线安装

		在一些特殊情况完全无法访问外部网络时，可以使用k3s提供的离线安装方式。
		
		- 下载k3s二进制

			[官方下载地址](https://github.com/k3s-io/k3s/releases)，树莓派是下载`k3s-armhf`文件，选择最新稳定版本

		- 下载k3s安装脚本

			`curl -sfL https://get.k3s.io -o install.sh`

		- 复制到树莓派

			在PC电脑上执行

			`scp k3s-armhf pi@xxx.xxx.xxx.xxx:/home/pi/`

			`scp install.sh pi@xxx.xxx.xxx.xxx:/home/pi/`

			在树莓派上执行

			`chmod +x k3s-armhf && sudo cp /home/pi/k3s-armhf /usr/local/bin/k3s`

		- 安装

			`INSTALL_K3S_SKIP_DOWNLOAD=true /home/pi/install.sh`

### 配置k3s

- 配置外部kubectl/helm访问集群

	详细参考[官方文档](https://docs.rancher.cn/docs/k3s/cluster-access/_index/)

	- 获取kubeconfig

		树莓派默认用户pi没有权限访问系统目录，root用户默认是无法直接ssh登录，所以直接使用scp无法复制到本地，分两步：

		`ssh pi@xxx.xxx.xxx.xxx 'sudo cp /etc/rancher/k3s/k3s.yaml /home/pi/ && sudo chown pi.pi /home/pi/k3s.yaml'`

		`scp pi@xxx.xxx.xxx.xxx:/home/pi/k3s.yaml ~/.kube/config`

	- 修改集群IP地址/机器名/域名

		编辑本地`~/.kube/config`文件，找到内容`server: https://127.0.0.1:6443`，根据实际情况修改127.0.0.1为真实的IP地址（或者能使用机器名/域名访问树莓派时，也可以替换为机器名/域名）

	- 验证

		PC电脑上执行（需要先确保PC上已安装kubectl命令）`kubectl get node`，有类似输出：

			NAME          STATUS   ROLES                  AGE     VERSION
			raspberrypi   Ready    control-plane,master   9m46s   v1.22.5+k3s1

- 更换容器镜像源-新

	以 root 身份创建`/etc/rancher/k3s/registries.yaml`文件

	使用USTC的例子：

		mirrors:
		    docker.io:
		        endpoint:
		            - https://docker.mirrors.ustc.edu.cn/
		            - https://registry-1.docker.io
		        rewrite: {}
		configs: {}
		auths: {}

	参考文档：

	- 英文 [Private Registry Configuration](https://docs.k3s.io/installation/private-registry)
	- 中文 [私有镜像仓库配置参考](https://docs.rancher.cn/docs/k3s/installation/private-registry/_index/#mirrors)

- 更换容器镜像源-旧（可忽略）

	参考k3s官方的[配置containerd文档](https://rancher.com/docs/k3s/latest/en/advanced/#configuring-containerd)
		
	- 复制配置文件

		`sudo cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl`

	- 修改`config.toml.tmpl`，添加如下内容，把默认的docker.io地址设置为镜像加速地址（这里选择了网易提供的镜像加速）

			[plugins.cri.registry.mirrors]
  			  [plugins.cri.registry.mirrors."docker.io"]
			    endpoint = ["https://hub-mirror.c.163.com","https://mirror.baidubce.com"]

	- 验证

		- 重启k3s服务

			`sudo systemctl restart k3s`

		- 检查当前配置

			`crictl info|grep  -A 7 registry`，有类似输出：

			    "registry": {
				  "mirrors": {
			        "docker.io": {
			          "endpoint": [
			            "https://hub-mirror.c.163.com",
			            "https://mirror.baidubce.com"
			          ],

### 安装Kubernetes Dashboard

详细参考k3s官方的[Kubernetes Dashboard文档](https://rancher.com/docs/k3s/latest/en/installation/kube-dashboard/)

前面已经配置好了`外部kubectl/helm访问集群`，就可以通过PC电脑方便的操作Kubernetes了，下面的操作在PC电脑上执行即可（需要先确保PC上已安装kubectl命令）

- 安装

		GITHUB_URL=https://github.com/kubernetes/dashboard/releases
		VERSION_KUBE_DASHBOARD=$(curl -w '%{url_effective}' -I -L -s -S ${GITHUB_URL}/latest -o /dev/null | sed -e 's|.*/||')
		kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/${VERSION_KUBE_DASHBOARD}/aio/deploy/recommended.yaml

- 添加管理用户

	需要创建几个配置文件

	- dashboard.admin-user.yml

			apiVersion: v1
			kind: ServiceAccount
			metadata:
			  name: admin-user
			  namespace: kubernetes-dashboard

	- dashboard.admin-user-role.yml

			apiVersion: rbac.authorization.k8s.io/v1
			kind: ClusterRoleBinding
			metadata:
			  name: admin-user
			roleRef:
			  apiGroup: rbac.authorization.k8s.io
			  kind: ClusterRole
			  name: cluster-admin
			subjects:
			- kind: ServiceAccount
			  name: admin-user
			  namespace: kubernetes-dashboard
 
 	- 添加

		`kubectl create -f dashboard.admin-user.yml -f dashboard.admin-user-role.yml`
  
  	- 获取登录Token

		`kubectl -n kubernetes-dashboard describe secret admin-user-token | grep '^token'`
  
- 本地访问Dashboard

	利用kubectl命令本身提供的代理功能，简单方便。

	- 启动代理

		`kubectl proxy`

		默认使用8001端口，如果被其他服务占用，可以指定其他端口，例如：

		`kubectl proxy --port=8010`

	- 浏览器访问

		`http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/`

		**注意端口8001要保持和启动代理的端口一致**

	- 登录

		输入前面已经获取的Token即可

-  卸载

		kubectl delete ns kubernetes-dashboard
		kubectl delete clusterrolebinding kubernetes-dashboard
		kubectl delete clusterrole kubernetes-dashboard
