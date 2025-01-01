---
title: 'homelab PaaS简介'
date: '2025-01-01T15:28:05+08:00'
lastmod: '2025-01-01T15:28:05+08:00'

draft: 

tags: ["homelab", "PaaS"]
author: "afterain"
---
## 什么是PaaS

PaaS是面向开发人员，围绕应用为中心的平台。提供了应用开发、部署、运行和管理所需的服务，而无需构建和维护。

### 有什么优势

包括：中间件(数据库/分布式缓存/消息队列等)、开发语言和工具(运行时/框架等)、运维(日志/监控等)、容器部署等等。

如果采用云原生，那么`容器即服务CaaS`的基础`Kubernetes`是较好的选择(能跨多云服务商、提供兼容性和一致性API)

### 解决用户什么问题

主要是实现降本增效和高可用性：

- 提高效率

	缩短开发周期，减少应用部署时间。不必担心底层基础架构的维护和更新，例如使用 DevOps 和持续交付（CD）等敏捷实践。

- 降低成本

	无需担心中间件服务的容量问题，服务按需使用，只需为实际用量付费。

- 高可用性

	面对复杂系统，构建弹性强且高可用的应用。

##  homelab期望一个什么样的PaaS

homelab主要是使用应用而不是开发应用，所以关注的主要软件的运行，需要有足够灵活性支持不同运行时/开发语言/框架/工具。

Kubernetes有两个核心理念：[声明式编程](https://en.wikipedia.org/wiki/Declarative_programming)和[面向终态](https://branislavjenco.github.io/desired-state-systems/)。软件的部署、运行和管理等工作主要就是编写声明式配置文件，然后结合GitOps，后续的工具链能直接满足`一切皆代码`原则。

根据`最小依赖`和`一切皆代码`原则，选择：Kubernetes + GitOps

## 工具链

- Kubernetes

	- [Docker](https://www.docker.com/products/kubernetes)

		Docker在Desktop版包含了Kubernetes，给开发人员提供了开箱即用的体验。

	- [OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift)

		OpenShift 是红帽的产品，采用了 Kubernetes 作为 OpenShift 中的编排技术。

	- [Rancher](https://rancher.com/kubernetes/)

		一个开源的 Kubernetes 管理平台，能够实现多 Kubernetes 集群的统一纳管。

	-  [K3s](https://k3s.io)

		Rancher推出的轻量级的 Kubernetes 发行版，专为在资源有限的环境中运行，每个服务器实例仅需 512MB RAM 以及 200MB 的磁盘空间。

- GitOps

	主要功能是监听Git Repositories变化和自动拉取变更，对比当前应用运行状态与期望运行状态的差异，执行自我修复和自我调整，达到期望的状态。

	- [Flux](https://fluxcd.io)

	- [ArgoCD](https://argo-cd.readthedocs.io/en/stable/)

## 目标

根据`最小依赖`和`一切皆代码`原则，选择的方案：Kubernetes + GitOps
