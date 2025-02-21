---
title: 'GitOps安装配置'
date: '2025-02-21T11:06:41+08:00'
lastmod: '2025-02-21T11:06:41+08:00'

draft: false

tags: ["homelab", "PaaS", "gitops"]
author: "afterain"
---
### 领域知识

[homelab PaaS简介]({{< ref "homelab-paas" >}})

### 工具链

#### 选择

- Kubernetes发行版

	- Docker

		Docker在Desktop版包含了Kubernetes，给开发人员提供了开箱即用的体验。

	- OpenShift

		OpenShift 是红帽的产品，采用了 Kubernetes 作为 OpenShift 中的编排技术。

	- Rancher

		一个开源的 Kubernetes 管理平台，能够实现多 Kubernetes 集群的统一纳管。

	- K3s

		Rancher推出的轻量级的 Kubernetes 发行版，专为在资源有限的环境中运行，每个服务器实例仅需 512MB RAM 以及 200MB 的磁盘空间。

	Homelab一般会有资源有限的设备（例如树莓派），所以选择K3s

- GitOps

	- Flux

	- ArgoCD

	- Gitea

		ArgoCD的前提是依赖一个Git仓库，在家庭场景中考虑隐私安全，需要先安装一个Git服务，然后创建仓库进行应用的配置管理。

	个人感觉ArgoCD简单一些，不代表其他的工具不好或不适合。

#### k3s

- 安装工具

	K3s的官方和开源社区都有提供简化的一键安装工具。两个都支持K3s的普通/高可用两种方式。

	- [AutoK3s](https://github.com/cnrancher/autok3s)

		用于简化 K3s 集群管理的轻量级工具，您可以使用 AutoK3s 在任何地方运行 K3s 服务。

	- [k3sup](https://github.com/alexellis/k3sup)

		is a light-weight utility to get from zero to KUBECONFIG with k3s on any local or remote VM.

	`k3sup`仅安装K3s本身，而`AutoK3s`可以在安装时指定[自动部署清单](https://docs.rancher.cn/docs/k3s/advanced/_index/#%E8%87%AA%E5%8A%A8%E9%83%A8%E7%BD%B2%E6%B8%85%E5%8D%95)（能同时部署GitOps），所以选择`AutoK3s`。

#### 前提条件

- [树莓派]({{< ref "homelab-iaas-netboot-raspberrypi" >}})

### 配置代码

#### AutoK3s

- 安装AutoK3s

	```
		curl -sS https://rancher-mirror.rancher.cn/autok3s/install.sh  | INSTALL_AUTOK3S_MIRROR=cn sh -
	```

#### K3s

- 配置GitOps

	详细参考[自动部署清单-Helm](https://docs.rancher.cn/docs/k3s/helm/_index)

	安装ArgoCD和Gitea，Helm配置文件 `gitops.yaml` 内容如下（host改成自己的域名）：

	```
		apiVersion: helm.cattle.io/v1
		kind: HelmChart
		metadata:
		  name: gitea
		  namespace: kube-system
		spec:
		  chart: https://dl.gitea.com/charts/gitea-10.6.0.tgz
		  version: 10.6.0
		  targetNamespace: default
		  valuesContent: |-
		    redis-cluster:
		      enabled: false
		    redis:
		      enabled: false
		    postgresql-ha:
		      enabled: false
		    postgresql:
		      enabled: false
		    gitea:
		      config:
		        database:
		          DB_TYPE: sqlite3
		        session:
		          PROVIDER: memory
		        cache:
		          ADAPTER: memory
		        queue:
		          TYPE: level
		    persistence:
		      enabled: true
		      size: 1Gi
		    ingress:
		      enabled: true
		      hosts:
		        - host: git.homelab.afterain.xyz
		          paths:
		            - path: /
		              pathType: Prefix
		---
		apiVersion: helm.cattle.io/v1
		kind: HelmChart
		metadata:
		  name: argo-cd
		  namespace: kube-system
		spec:
		  chart: https://github.com/argoproj/argo-helm/releases/download/argo-cd-7.8.3/argo-cd-7.8.3.tgz
		  version: 7.8.3
		  targetNamespace: default
		  valuesContent: |-
		    server:
		      ingress:
		        enabled: false
	```

- 配置镜像仓库

	详细参考[私有镜像](https://docs.k3s.io/installation/private-registry)

	`registries.yaml`内容如下：

	```
		mirrors:
		  "docker.io":
		    endpoint:
		      - "https://mirror.iscas.ac.cn"
		      - "https://docker.m.daocloud.io"
		      - "https://registry-1.docker.io"
	```

- kubeconfig

	- 查询kubeconfig内容

		```
			autok3s kubectl config view --raw=true  > homelab.yaml
		```

	- 合并到本地kubeconfig

		```
			KUBECONFIG=~/.kube/config:./homelab.yaml kubectl config view --flatten >~/.kube/new-config
			cp ~/.kube/new-config ~/.kube/config
		```


### 自动化

#### AutoK3s

详细参考[创建Native集群](https://docs.rancher.cn/docs/k3s/autok3s/native/_index)。安装命令如下：

```
	autok3s create \
	    --provider native \
	    --name homelab \
	    --k3s-channel stable \
	    --k3s-install-mirror INSTALL_K3S_MIRROR=cn \
	    --k3s-install-script  https://rancher-mirror.rancher.cn/k3s/k3s-install.sh \
	    --system-default-registry registry.cn-hangzhou.aliyuncs.com \
	    --ssh-user pi \
	    --ssh-agent-auth \
	    --manifests gitops.yaml \
	    --registry registries.yaml \
	    --master-ips <树莓派IP地址>
```

耐心等待一段时间即可，中间可通过命令查询进度

```
	autok3s kubectl get pod -A
```

如果需要卸载

```
	autok3s delete --provider native --name homelab
```

#### Gitea

- 浏览器访问

	参考`gitops.yaml`的`host`参数

- 默认账号

	- username： gitea_admin
	- password： r8sA8CPHD9!bt6d

#### ArgoCD

- 浏览器访问

`kubectl port-forward svc/argo-cd-argocd-server 8080:443`，访问`localhost:8080`

- 默认账号

	- username： admin
	- password：执行`kubectl get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d`获取

### 补充：GitOps使用

- 新建Git仓库

	登录Gitea，新建仓库：`homelab`，clone仓库后创建目录`saas`

- 新建Application

	可以通过CLI和UI两种方式添加

	- Git仓库地址

		ArgoCD和Gitea安装在同一个Kubernetes集群内，可以使用集群内部域名`gitea-http.default.svc.cluster.local`（因为是部署在同一个`namespace`下，可以简写为`gitea-http`）而不需要通过外部的域名/端口转发的方法访问。

		Git仓库地址例子： `http://gitea-http:3000/gitea_admin/homelab.git`
		
	- 例子

	```
		apiVersion: argoproj.io/v1alpha1
		kind: Application
		metadata:
		  name: homelab
		spec:
		  destination:
		    namespace: default
		    server: https://kubernetes.default.svc
		  source:
		  path: saas
		    repoURL: http://gitea-http:3000/gitea_admin/homelab.git
		    targetRevision: HEAD
		    directory:
		      recurse: true
		  sources: []
		  project: default
		  syncPolicy:
		    automated:
		      prune: true
		      selfHeal: true
	```