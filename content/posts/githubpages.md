---
title: "使用GitHub免费搭建个人网站"
date: 2022-10-25T18:46:06+08:00
lastmod: 2022-10-25T18:46:06+08:00
draft: false
author: "afteraincc"
authorLink: "afterain.cc"

tags: ['建站']
categories: ['技术']

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


### 什么是GitHub Pages

[GitHub Pages](https://pages.github.com/)是通过git仓库来管理个人或项目网站。

优势

<!--more-->

- 文章不丢失

    使用git（分布式代码管理工具）管理文章内容，能安全可靠的保存文章，以及所有的历史变更记录。

- 免维护工作和无成本

    避免个人网站的各种维护工作和成本

    - 成本：购买服务器/带宽/存储等费用

    - 维护工作：https证书申请/轮换，网站安全，系统管理等等

- 自定义域名

    很多blog服务也能保证文章丢失，也不需要维护工作和成本，但是很少支持使用自己的自定义域名。这个能让网站看起来是一个独立的个人网站，和Github完全无关。

劣势

- 不支持动态网站

    如果只是用来写文章/记录各种心得，静态网站能完全满足需求。但是如果需要实验建站的各种前后端技术，就不太适合了。

### 如何使用GitHub Pages

关键步骤

- 注册github账号

- 创建git仓库

    注意仓库名必须是`username.github.io`形式，`username`是GitHub的用户名
    
- 编辑内容

    主页文件：index.html
    
    先通过git clone代码仓库，然后创建/编辑文件，最后在提交代码成功时，会自动发布到网站
    
- 自定义域名

    - 需要先在自己的域名提供商中配置：
    
        增加解析

        ```
        记录类型： CNAME
        主机记录： @ （直接解析主域名，例如github.io）
        记录值： username.github.io （username替换为GitHub的用户名）
        ```

        如果需要解析例如www.github.io，则在增加一条解析，其他参数相同，主机记录设置为： www 即可。

    - 详细文档以及GitHub本身的配置参考[GitHub Pages配置文档](https://docs.github.com/en/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages)

        基本过程是在对应的git仓库中设置，链接：https://github.com/usename/usename.github.io/settings/pages（username替换为GitHub的用户名）
        
        - 分支

            默认是main

        - 域名

            填写自己的域名

        - HTTPS

            勾选`Enforce HTTPS`，不能通过HTTP访问，强制设置为HTTPS访问。

### 静态网站工具

静态网站的工具非常多，例如GitHub官方支持的Jekyll，还有Hugo/Hexo/VuePress/Nuxt.js等。

每个工具的使用方法各有特色，但是基本原理都是有一个build步骤，把内容编译成html/js/css文件，并输出到一个public目录。然后把http服务的根目录设置为public目录即可访问。

### 静态网站工具如何结合GitHub Pages

GitHub支持[workflows](https://docs.github.com/en/actions/using-workflows/about-workflows)的功能，能在提交代码后自动执行指定的任务。只要把任务配置为静态网站工具的build就能自动发布内容了。

以[Hugo](https://gohugo.io)工具为例，详细参考[Host on GitHub](https://gohugo.io/hosting-and-deployment/hosting-on-github/)

- 设置任务

    先在git仓库的根目录的`.github/workflows`新建`gh-pages.yml`文件，内容如下：

    ```
    name: github pages

    on:
      push:
        branches:
          - main  # Set a branch to deploy
      pull_request:

    jobs:
      deploy:
        runs-on: ubuntu-22.04
        steps:
          - uses: actions/checkout@v3
            with:
              submodules: true  # Fetch Hugo themes (true OR recursive)
              fetch-depth: 0    # Fetch all history for .GitInfo and .Lastmod

          - name: Setup Hugo
            uses: peaceiris/actions-hugo@v2
            with:
              hugo-version: 'latest'
              extended: true

          - name: Build
            run: hugo --minify

          - name: Deploy
            uses: peaceiris/actions-gh-pages@v3
            if: github.ref == 'refs/heads/main'
            with:
              github_token: ${{ secrets.GITHUB_TOKEN }}
              publish_dir: ./public
    ```

- 修改`GitHub Pages配置`

    上面设置的任务中，实际是把`main`分支的内容，经过Hogo的build，把public下所有内容重新提交到`gh-pages`分支根目录下。所以只要把`GitHub Pages配置`中的分支参数修改为： gh-pages，即可以实现`提交代码自动发布到网站`的功能。
