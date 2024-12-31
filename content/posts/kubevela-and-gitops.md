---
title: "安装配置Kubevela和GitOps"
subtitle: ""
date: 2022-10-26T18:21:39+08:00
lastmod: 2022-10-26T18:21:39+08:00
draft: false
author: "afteraincc"
authorLink: "afterain.cc"
description: ""
license: ""
images: []

tags: ['KubeVela', 'Kubernetes', 'GitOps']
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

### 安装

KubeVela需要一个K8S集群作为管控平台，Mac的个人开发环境下，最方便的是直接安装`Docker Desktop`。安装完成后，可以把其他K8S集群加入KubeVela进行管理。

<!--more-->

**注意：不要在个人环境下安装给线上提供服务的KubeVela管控平台/集群，稳定性和数据备份等都不完善**

- KubeVela

	`vela install`

	注意：不支持arm架构（例如RaspberryPi），在执行到Job`kubevela-vela-core-admission-create`时，Pod会提示错误信息`standard_init_linux.go:228: exec user process caused: exec format error`

- dashboard

	`vela addon enable velaux serviceType=NodePort repo=acr.kubevela.net`

	- 如何访问

		执行返回结果`ENDPOINT`有端口信息，在浏览器访问`http://localhost:port`即可
	
		- 初始账号/密码

			执行`vela logs -n vela-system --name apiserver addon-velaux | grep "initialized admin username"`
			
			类似输出`apiserver-655b77f8fd-l9tz8 apiserver 2022-05-19T07:40:35.530349180Z {"level":"info","ts":1652946035.5300071,"caller":"usecase/user.go:109","msg":"initialized admin username and password: admin / o32v564hof\n"}`，内容中`admin`是账号，`o32v564hof`是密码。

		- 端口
		
			因为`Docker Desktop for Mac`的`docker0 bridge`是运行在一个Linux的虚拟机中（参考官方文档[Networking features](https://docs.docker.com/desktop/mac/networking/)），Host下无法直接访问容器（包括Kubernetes的Ingress）。但是可以通过`Port Mapping`的机制暴露到Host中，所以安装时使用`NodePort`作为serviceType。

### 配置

- fluxcd

	安装helm相关的应用需要依赖fluxcd这个addon，通过dashboard或命令行安装都可以

- GitOps

	GitOps的前提是需要一个git仓库，为了方便在没有网络/无法访问公司仓库时·测试/调试，个人开发环境可以安装一个独立/轻量的git仓库服务。

	- gitea

		执行`vela up -f gitea.yaml`安装。

		- 关键参数

			要能正常访问（原因见前面的dashboard的端口说明），Service需要配置nodePort

			```
          service:
            http:
              nodePort: 30000
              type: NodePort
			```		

	- 监听git仓库

		执行`vela up -f infra.yaml`安装。安装成功后，vela将自动监听git仓库变化自动同步K8S集群。

		- 关键参数

			- 仓库地址

				gitea容器的服务端口是3000，注意因为fluxcd和gitea不在同一个namespace下，域名使用了服务的完整地址（服务的域名规则参考K8S的域名解析相关文档）。

				```
      			# 将此处替换成你需要监听的 git 配置仓库地址
      			url: http://gitea-http.default.svc.cluster.local:3000/dev/docker-desktop.git
	  			  		```

	  			  	- 监听目录

	  			  		根据实际情况填写路径

	  			  		```
      			# 监听变动的路径，指向仓库中 infra 目录下的文件
      			path: ./infra
				```

	- 完整yaml配置如下：

		```
		## gitea.yaml
		kind: Application
		apiVersion: core.oam.dev/v1alpha2
		metadata:
		  name: gitea
		spec:
		  components:
		    - name: gitea
		      type: helm
		      properties:
		        repoType: helm
		        url: 'https://dl.gitea.io/charts/'
		        chart: gitea
		        version: 5.0.0
		        values:
		          gitea:
		            config:
		              cache:
		                ADAPTER: memory
		                ENABLED: true
		                HOST: '127.0.0.1:9090'
		                INTERVAL: 60
		          ingress:
		            enabled: true
		            hosts:
		              - host: xxx
		                paths:
		                  - path: /
		                    pathType: Prefix
		          memcached:
		            enabled: false
		          persistence.size: 1Gi
		          service:
		            http:
		              nodePort: 30000
		              type: NodePort
		```

		```
		## infra.yaml
		apiVersion: core.oam.dev/v1beta1
		kind: Application
		metadata:
		  name: infra
		spec:
		  components:
		  - name: infra-source
		    type: kustomize
		    properties:
		      repoType: git
		      # 将此处替换成你需要监听的 git 配置仓库地址
		      url: http://gitea-http.default.svc.cluster.local:3000/dev/docker-desktop.git
		      # 如果是私有仓库，还需要关联 git secret
		      #secretRef: git-secret
		      # 自动拉取配置的时间间隔，由于基础设施的变动性较小，此处设置为十分钟
		      pullInterval: 10m
		      git:
		        # 监听变动的分支
		        branch: master
		      # 监听变动的路径，指向仓库中 infra 目录下的文件
		      path: ./infra
		```

### 配置GitOps

- 创建git仓库

	打开gitea [http://localhost:3000](http://localhost:3000)，第一个注册的账号是管理员。
	
	新建一个git仓库，`签名信任模型`选择`默认信任模型`，这样可以匿名clone仓库，但是不能push代码。

- clone仓库

	`git clone http://localhost:30000/gitdev/docker-desktop.git`

- 备份集群相关代码

	仓库根目录创建`clusters`，并把之前的`gitea.yaml`和`infra.yaml`添加到目录并提交。

- 增加其他应用

	仓库根目录创建`infra`目录，然后根据需要（参考vela文档）增加即可。例如安装`prometheus`。

	因为是虚拟机环境，注意配置helm的变量如下，否则无法正常启动`prometheus-node-exporter`。

	```
    values:
      nodeExporter:
        hostRootfs: false
	```

- 完整代码

	```
	apiVersion: core.oam.dev/v1beta1
	kind: Application
	metadata:
	  name: prometheus
	spec:
	  components:
	  - name: prometheus
	    type: helm
	    properties:
	      repoType: "helm"
	      url: "https://prometheus-community.github.io/helm-charts"
	      chart: "prometheus"
	      version: "15.8.7"
	      values:
	        nodeExporter:
	          hostRootfs: false
	```

### 补充

- 如何修改自定义域名

	编辑 coredns 配置 `kubectl -n kube-system edit configmap coredns`
	
	在`data.Corefile`内容中增加：
	
	```
      hosts {
          10.10.10.10 example.com
          fallthrough
      }
	```

- 如何离线安装镜像

	有些镜像需要翻墙才能正常访问，可以先找一个能访问的PC下载，然后离线安装到设备中

	- 下载

		能访问的PC通过pull命令即可下载。如何是下载ARM平台的，需要设置`OS/ARCH`参数，例如：
		
		`docker pull --platform linux/arm/v7 k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.4.1`

	- 保存

		`docker save k8s.gcr.io/kube-state-metrics/kube-state-metrics:v2.4.1 -o kube-state-metrics-v2.4.1.tar.gz`

	- 安装

		gz包上传到无法访问的设备，执行：
		
		`docker import kube-state-metrics-v2.4.1.tar.gz`
		
		如果不是使用docker，而是cri，执行：
		
		`ctr image import kube-state-metrics-v2.4.1.tar.gz`
