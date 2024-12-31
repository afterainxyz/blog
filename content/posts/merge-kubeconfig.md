---
title: "一键合并多个kubeconfig"
subtitle: ""
date: 2022-10-26T20:28:45+08:00
lastmod: 2022-10-26T20:28:45+08:00
draft: false
author: "afteraincc"
authorLink: "afterain.cc"
description: ""
license: ""
images: []

tags: ['Kubernetes']
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

执行`KUBECONFIG=~/.kube/config:./other.config kubectl config view --flatten >~/.kube/new_config && cp ~/.kube/new_config ~/.kube/config`

在环境变量`KUBECONFIG`中，多个文件使用`:`分隔

<!--more-->
