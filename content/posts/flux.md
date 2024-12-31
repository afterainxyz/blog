---
title: "安装配置GitOps工具Flux"
subtitle: ""
date: 2022-10-27T08:56:08+08:00
lastmod: 2022-10-27T08:56:08+08:00
draft: true
author: "afteraincc"
authorLink: "afterain.cc"
description: ""
license: ""
images: []

tags: ['GitOps', 'Flux']
categories: ['技术']

featuredImage: "/img/posts/fluxcdio-ar21.svg"
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

[Flux](https://fluxcd.io)是一个持续和渐进式交付解决方案（continuous and progressive delivery solutions）。详细的安装和使用参考[官方文档](https://fluxcd.io/flux/)。

<!--more-->

### 安装

- 前提条件

	- Git仓库

		Flux支持多种Git仓库服务（例如GitHub，GitLab等）或者自建的Git Over SSH

	- Kubernetes集群

		kubectl可以正常访问集群。执行`flux check --pre`确认是否满足要求

- 安装Git仓库

	[使用 Helm 在 Kubernetes 云原生环境中安装 Gitea](https://docs.gitea.io/zh-cn/install-on-kubernetes/)。

	- 执行`helm install gitea -f values.yaml gitea/gitea`：
	
		```
		## values.yaml
	    gitea:
	      config:
	        database:
	          DB_TYPE: sqlite3
	      cache:
	        ENABLED: true
	        ADAPTER: memory
	        INTERVAL: 60
	
		ingress:
		  enabled: true
	    
	    persistence:
	      enabled: true
	      size: 1Gi
	    
	    memcached:
	      enabled: false
	    postgresql:
	      enabled: false
		```

	- [默认账号](https://gitea.com/gitea/helm-chart/#gitea)
	
		- username： gitea_admin 

		- password： r8sA8CPHD9!bt6d

	- 登陆链接：(端口转发的代理方式不能正常flux bootstrap，错误'generating source secret'=>'error ssh: handshake failed: EOF')
	
		-  开启端口转发 `kubectl --namespace default port-forward svc/gitea-http 3000:3000`

		- 访问 http://127.0.0.1:3000	
	
	- 创建仓库：demo-env，demo-app
	
	- 访问仓库：

	-	 前提
	
			开启端口转发 `kubectl --namespace default port-forward svc/gitea-http 3000:3000`
	
		- 集群外部访问域名： Gitea默认`git.example.com`

			由于域名涉及购买/解析等操作，本文档的实例并不指定域名，所以替换为： localhost:3000
		
			如果有自己域名，修改`values.yaml`，并把域名`git.yourdomain`解析到Kubernetes的Node IP才能正常访问
		
			```
			ingress:
			  enabled: true
			  hosts:
			    - host: git.yourdomain
			      paths:
			        - path: /
			          pathType: Prefix
			```
		
	- 例子
	
		`git clone http://localhost:3000/gitea_admin/demo-env.git`

- 安装Flux

	- 执行如下命令（由于镜像需要从海外下载，安装完成需要耐心等待一段时间）

		```
		flux bootstrap git --allow-insecure-http --url=http://localhost:3000/gitea_admin/hac-env.git --branch=main --path=cluster -u gitea_admin -p 'r8sA8CPHD9!bt6d'
		```

	- 几个关键参数说明：
	
		- --url
		
			Git仓库的地址，用于保存flux配置
	
		- --path
	
			在Git仓库的对应路径下会生成一个`flux-system`目录，保存flux本身的配置，以及后续应用的flux配置。
		
			如果有多个cluster，可以使用独立子目录区别，例如`--path=clusters/xxx`

### Flux配置

参考[GitOps流程](https://www.gitops.tech/#how-does-gitops-work)，Flux的配置分2个仓库。一个是`环境仓库（environment repository）`保存Flux本身的配置和部署相关配置，另一个是`应用仓库（application repository）`，保存应用的代码和配置。在安装Flux时，配置的Git仓库地址是`环境仓库`。


这两个仓库物理上可以是共享使用同一个Git仓库地址。需要注意的是，目录一定要区分开，应用相关的配置不要放在`bootstrap`安装时参数`--path`指定的目录。

- 下载代码

	Flux安装完成后，已经把Flux本身的系统配置提交到对应的Git仓库中了，下载代码后才能进行应用部署相关的配置。执行：`git clone ssh://git@xxx.xxx.xxx.xxx/$GIT_PATH/xxx.git`（注意这里的Git仓库地址是`环境仓库`，也就是Flux安装时设置的Git仓库地址）

- 添加应用

	每个应用由一个仓库（`GitRepository`或`HelmRepository`），和若干配置（`Kustomization`或`HelmRelease`）组成。多个文件可以组织到`--path`下的独立子目录，也可以都放在`--path`目录。一般建议一个应用一个目录，方便管理。

	在`--path`目录下新建文件`demo-app-source.yaml`，`demo-app.yaml`。应用并没有使用单独的Git仓库，而是复用Flux的`环境仓库`（通过sourceRef引用`flux-system`），使用一个独立目录`demo-app`。内容如下：

	```
	## demo-app-source.yaml
	apiVersion: source.toolkit.fluxcd.io/v1beta2
	kind: GitRepository
	metadata:
	  name: demo-app
	  namespace: flux-system
	spec:
	  interval: 30s
	  ref:
	    branch: main
	  url: http://gitea-http.default.svc.cluster.local:3000/gitea_admin/demo-app.git
	```

	```
	## demo-app.yaml
	apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
	kind: Kustomization
	metadata:
	  name: demo-app
	  namespace: flux-system
	spec:
	  interval: 5m0s
	  path: ./demo-app
	  prune: true
	  sourceRef:
	    kind: GitRepository
	    name: demo-app
	```
   
**虽然`kind`都是`Kustomization`，但是Flux的`Kustomization`和Kubernetes的`Kustomization`不是一个东西。Flux的apiVersion是`kustomize.toolkit.fluxcd.io/v1beta2`，Kubernetes的apiVersion是`kustomize.config.k8s.io/v1beta1`。**

**一个仓库还是多个仓库?一个目录还是多个目录?目录结构如何设计? 参考[详细说明文档](https://fluxcd.io/flux/guides/repository-structure/)**

### 应用配置

Flux支持两种配置方式`Kustomization`或`HelmRelease`。
	
在应用仓库的`demo-app`目录新建文件`kustomization.yaml`（详细参考[Kubernetes Kustomization](https://kustomize.io)文档），内容如下：
	
```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: default
resources:
- xxx.yaml
- yyy.yaml
```

其中xxx.yaml，yyy.yaml是具体应用的yaml配置
