---
title: 'Paas扩展'
date: '2025-03-18T08:39:43+08:00'
lastmod: '2025-03-18T08:39:43+08:00'

draft: false

tags: ["homelab", "PaaS"]
author: "afterain"
---
### 领域知识

[homelab PaaS]({{< ref "homelab-paas" >}})仅提供了K8S + GitOps基础能力，不包含各种PaaS服务。

在安装SaaS软件时，会依赖一些基础的服务（例如存储服务、数据库服务等），就需要对PaaS进行扩展。

### 工具链

#### 选择

- 存储

	有一个双盘NAS，能提供基础的NFS存储服务。可以使用K8S的[持久卷](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)功能来管理空间的分配。

- 配置安全

	K8S的[Secret](https://kubernetes.io/zh-cn/docs/concepts/configuration/secret/)是一种包含少量敏感信息例如密码、令牌或密钥的对象， 使用 Secret 意味着你不需要在应用程序代码中包含机密数据。

	但是使用GitOps时，会遇到一个问题：[I can manage all my K8s config in git, except Secrets](https://github.com/bitnami-labs/sealed-secrets)，常见的几个方案：
	
	- [Sealed-Secrets](https://github.com/bitnami-labs/sealed-secrets)

		利用非对称加密算法对 Secret 对象进行加密，使用的时候在集群内自动进行解密，这样就可以将加密后的密钥安全地存储在 Git 仓库中。

	- [External-Secrets](https://external-secrets.io/latest/)

		需要外部密钥管理服务的支持，例如 AWS Secrets Manager、Google Secrets Manager、Azure Key Vault 等。

	- [Vault](https://vault.com/)

		HashiCorp 开源的一款密钥管理工具，要将它和 ArgoCD 结合使用需要额外的插件。

#### 前提条件

[homelab PaaS GitOps]({{< ref "homelab-paas-gitops" >}})

### 配置

#### NFS持久卷的使用

- PV

	- capacity：指定分配给K8S各种应用的存储最大空间
	
	- nfs(server/path)：指定NFS服务的地址和路径

	- accessModes：NFS可以多节点同时挂载读写访问

	- 例子

        ```
        	apiVersion: v1
        	kind: PersistentVolume
        	metadata:
        	  name: nas-nfs
        	spec:
        	  capacity:
        	    storage: 200Gi
        	  accessModes:
        	    - ReadWriteMany
        	  nfs:
        	    server: 192.168.0.xxx
        	    path: "/volume1/nfs"
        	  mountOptions:
        	    - nfsvers=4
        	  persistentVolumeReclaimPolicy: Retain
        ```

- PVC

	需要使用存储的应用可以先创建一个声明，参数requests是指需要申请的存储空间大小

    ```
    	apiVersion: v1
    	kind: PersistentVolumeClaim
    	metadata:
    	  name: nas-nfs
    	spec:
    	  accessModes:
    	    - ReadWriteMany
    	  storageClassName: ""
    	  resources:
    	    requests:
    	      storage: 10Gi
    	  volumeName: nas-nfs-for-xxx
    ```

#### Sealed-Secrets的安装配置

ArgoCD支持Helm的应用安装，Sealed-Secrets官方提供了[Helm Chart](https://github.com/bitnami-labs/sealed-secrets/tree/main/helm/sealed-secrets)

- destination：`kubernetes.default.svc`表示安装的K8S集群是ArgoCD所在集群

- syncPolicy：配置为自动同步方式`automated`

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sealed-secrets
spec:
  project: default
  source:
    chart: sealed-secrets
    repoURL: https://bitnami-labs.github.io/sealed-secrets
    targetRevision: 2.16.2
    helm:
      releaseName: sealed-secrets
      values: |
        fullnameOverride: sealed-secrets-controller
  destination:
    server: "https://kubernetes.default.svc"
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 自动化

有了GitOps基础，只需要把Yaml配置文件提交到git仓库，就能自动化安装配置
