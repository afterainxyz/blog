---
title: "如何在K8S中运行HomeAssisant"
subtitle: ""
date: 2022-10-26T18:04:15+08:00
lastmod: 2022-10-26T18:04:15+08:00
draft: false
author: "afteraincc"
authorLink: "afterain.cc"
description: ""
license: ""
images: []

tags: ['Kubernetes', 'HomeAssistant']
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

### 简介

HomeAssiatant支持容器的运行方式，但是容器没有宕机重启的能力，如果出现异常退出，需要手工登录树莓派重启，操作比较麻烦。这时就可以利用kubernetes的能力了。

### 安装
<!--more-->

- 安装

	`kubectl apply -f home-assistant-deployment.yaml`

	`home-assistant-deployment.yaml`内容如下：

		apiVersion: apps/v1
		kind: Deployment
		metadata:
		  labels:
		    app.kubernetes.io/name: home
		  name: home-assistant
		spec:
		  replicas: 1
		  selector:
		    matchLabels:
		      app.kubernetes.io/name: home
		  template:
		    metadata:
		      labels:
		        app.kubernetes.io/name: home
		    spec:
		      hostNetwork: true
		      containers:
		      - image: ghcr.io/home-assistant/raspberrypi4-homeassistant:stable
		        name: home-assistant
		        ports:
		        - containerPort: 8123
		        securityContext:
		          privileged: true
		        volumeMounts:
		        - mountPath: /config
		          name: ha-config
		      volumes:
		      - name: ha-config
		        hostPath:
		          path: /home/pi/ha-config

- 验证

	等待一段时间（ha的镜像是在`ghcr.io`国外网站，下载需要较长时间），可以执行`kubectl get pod`查看pod `home-assistant-xxx`的输出信息，如果状态是`Running`就表示启动成功，例如：

		NAME                                 READY   STATUS        RESTARTS   AGE
		home-assistant-7b6f645b9b-fgbz7      0/1     Running       0          0m10s

### 远程访问

运行HomeAssiatant的Deployment时，指定了两个重要参数：

- 网络

	`hostNetwork: true`，使用的是主机网络，也就是可以直接使用树莓派IP:port来访问HA服务。

- 权限

	`privileged: true`，启动的HA服务有权限访问树莓派上的所有设备。这个非常重要，HA远程控制智能设备时需要访问wifi/bluetooth/usb等设备

打来浏览器访问树莓派的`8213`端口即可
