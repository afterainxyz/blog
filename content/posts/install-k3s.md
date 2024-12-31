---
title: "一键快速安装K3s"
subtitle: ""
date: 2022-10-26T21:27:01+08:00
lastmod: 2022-10-26T21:27:01+08:00
draft: false
author: "afteraincc"
authorLink: "afterain.cc"
description: ""
license: ""
images: []

tags: ['Kubernetes', 'K3s']
categories: ['技术']

featuredImage: "/img/posts/k3s-horizontal-color.svg"
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

### 简介

[K3s](https://github.com/k3s-io/k3s) 是一个轻量级的 Kubernetes 发行版，它针对边缘计算、物联网等场景进行了高度优化。

<!--more-->

### 一键快速安装

K3s 提供了一个安装脚本，可以方便的在 systemd 或 openrc 的系统上将其作为服务安装。但是简单只是相对于单节点，如果需要多节点集群，需要较为复杂一些的配置。

官方和开源社区都有提供简化的工具：

- [AutoK3s](https://github.com/cnrancher/autok3s) 是用于简化 K3s 集群管理的轻量级工具，您可以使用 AutoK3s 在任何地方运行 K3s 服务。

- [k3sup](https://github.com/alexellis/k3sup) is a light-weight utility to get from zero to KUBECONFIG with k3s on any local or remote VM.

两个工具都能较大的简化多节点集群的创建/管理工作，使用下来各有优缺点。

### 比较AutoK3s和k3sup

- 文档

	- AutoK3s

		K3s官方团队提供的工具，有完善的中文文档

	- k3sup

		社区大牛alexellis个人提供的工具，只有英文文档

- UI

	- AutoK3s

		执行`autok3s serve`启动UI服务，访问地址`http://127.0.0.1:8080/`

	- k3sup

		不支持

- 网络加速

	- K3s

		支持配置私有镜像。参考[中文文档](https://docs.rancher.cn/docs/k3s/installation/private-registry/_index/#mirrors)
		[英文文档](https://docs.k3s.io/installation/private-registry)

	- AutoK3s

		参数可以选择国内加速站点，安装非常快速。参考[中文文档](https://docs.rancher.cn/docs/k3s/autok3s/native/_index#进阶使用)
		[英文文档](https://github.com/cnrancher/autok3s/blob/master/docs/i18n/en_us/native/README.md#setting-up-private-registry)

		通过UI创建集群时，可以在这里配置： Create => K3s Options => Registry

	- k3sup

		虽然K3s支持国内加速，但是k3sup没有参数能设置。所以安装速度看情况，有时候很慢，容易失败。有时候也比较快，一次成功。建议多次尝试

- 集群管理

	- AutoK3s

		支持增/删集群，删除集群时自动卸载K3s。集群配置和状态保存在`~/.autok3s/.db/`

	- k3sup

		不支持

- SSH密钥

	通过SSH密钥登陆节点时，是否支持免密操作

	- AutoK3s

		使用参数`--ssh-agent-auth`

		但是要求所有节点的用户名一样，否则登陆ssh执行失败。节点需要添加和master同名账号后才能执行成功

	- k3sup

		无需参数，自动识别本机的ssh-agent

		支持不同账号，使用参数`--server-user`

- kubeconfig

	- AutoK3s

		和执行机器本地环境的~/.kube/config互相独立，可以通过UI/CLI下载。例子：

		`autok3s kubectl config view --raw > ./download.config`

		合并集群和本地的kubeconfig需要手工处理：

		`KUBECONFIG=$HOME/.kube/config:./download.config kubectl config view --flatten > all-in-one.config && cp all-in-one.config $HOME/.kube/config`

	- k3sup

		本身支持合并集群和本地的kubeconfig。例子：

		`k3sup install --ip $IP --user $USER --merge --local-path $HOME/.kube/config --context my-k3s`

