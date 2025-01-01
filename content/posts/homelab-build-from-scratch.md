---
title: '从零构建homelab'
date: '2025-01-01T10:17:00+08:00'
lastmod: '2025-01-01T10:17:00+08:00'

draft: false

tags: ["homelab"]
author: "afterain"
---
## 为什么从零开始构建homelab

### 可以用来做什么

学习和实践各种技术，例如容器/IOT智能家居/AI大模型等。

### 为什么不使用现成的XXX服务

#### 保护隐私

互联网提供了很多方便易用的服务，但是在使用中也容易有数据和隐私泄露的问题。homelab则可以完全由自己掌控(例如视频监控)。

#### 响应延时

按照`数据就近计算`原则，家中设备产生的数据直接在局域网内部计算，不需要通过互联网走一遍，能降低整体的响应延时。

### 范围

包含最基础的IaaS，中间的PaaS，以及面向最终场景的SaaS。

#### IaaS

自动化安装操作系统和基础软件，提供计算/存储/网络的硬件基础设施。

#### PaaS

以kubernetes作为云原生的软件运行环境。

#### SaaS

用开源服务/软件支持各种场景。


### 基本原则

#### 从零开始

不做任何的前提假设，从机器的裸金属开始，一步一步搭建。

#### 最小依赖

仅使用必要的硬件/软件来满足需求。

#### 一切皆代码（Everything as Code）[1](https://octopus.com/blog/what-is-everything-as-code) [2](https://docs.aws.amazon.com/wellarchitected/latest/devops-guidance/everything-as-code.html)

避免复杂的手工操作流程，所有环节能自动化完成。
	
要实现EaC，有几个关键点：[领域知识](https://en.wikipedia.org/wiki/Domain_knowledge)，[工具链](https://en.wikipedia.org/wiki/Toolchain)+配置代码和自动化
		
- 领域知识

	每个问题都涉及对应领域的概念和[流程](https://en.wikipedia.org/wiki/Pipeline_\(computing\))，了解这些能更好的选择工具去解决问题。

- 工具链

	是多个工具的集合。在选择时，优先选择[声明式编程](https://en.wikipedia.org/wiki/Declarative_programming)风格的[面向终态](https://branislavjenco.github.io/desired-state-systems/)的工具（例如Kubernetes），这样遇到问题时可以反复执行而不用担心副作用。

- 配置代码

	保存在代码版本管理工具中（例如Git），能记录所有的变更，方便用diff工具比较不同版本之间的差异。

- 自动化

	只需要`一键执行`就能自动化完成所有工作。`一键执行`既可以是执行一个命令，也可以点击一个按钮，也可以是一次代码提交（例如GitOps）。
