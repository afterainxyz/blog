---
title: "安装配置lightsail"
subtitle: ""
date: 2022-10-26T07:33:05+08:00
lastmod: 2022-10-26T07:33:05+08:00
draft: false
author: "afteraincc"
authorLink: "afterain.cc"
description: ""
license: ""
images: []

tags: ['AWS', '科学上网']
categories: ['技术']

featuredImage: "/img/posts/amazon-lightsail.svg"
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

- 选择区域

	Tokyo, Zone A (ap-northeast-1a)

- 选择系统

	Platform： Linux

	blueprint： OS Only => Debian 11.4

<!--more-->

- 选择配置/费用

	配置：1vCPU/512MB

	流量：1TB流量

		注意流量的规则较为复杂：流量=输入+输出
		所以在代理方式使用场景下，相当于实际512GB流量
		折算为7x24小时不间断使用带宽=512*1000/30/86400*8～=1.58Mbps，个人使用基本够用

		超出流量后：
			输入的免费（相当于lightsail从外部下载的内容）
			输出的额外收费（相当于代理从lightsail下载的内容）
			所以超出的流量实际相当于半价

	费用：3.5$/月


### 配置防火墙

开放端口： 22/80/443

默认已经开放22和80端口，只需要新增443端口即可


### SSH登陆

系统的SSH默认配置不允许密码登陆，需要先[下载私钥](https://lightsail.aws.amazon.com/ls/webapp/account/keys)

默认账号`admin`，ssh admin@xxx

### 其他

根据实际需要安装代理等

### EveryAsCode自动化脚本

lightsail支持[CLI](https://docs.aws.amazon.com/cli/latest/reference/lightsail/index.html)和[API](https://docs.aws.amazon.com/lightsail/2016-11-28/api-reference/Welcome.html)

- 前置依赖

	- 用户

		从安全考虑，为自动化创建lightsail实例[新建专用账号](https://us-east-1.console.aws.amazon.com/iamv2/home#/users)

		-  权限

			[创建专用policy](https://us-east-1.console.aws.amazon.com/iamv2/home#/policies)，只赋予必要权限。这样即使不小心泄漏了用户的AK，也不会造成太大的危害

			创建实例和配置防火墙权限：

			```
			lightsail:CreateInstances
			lightsail:OpenInstancePublicPorts
			```

			查询系统（blueprint）和配置（bundle）权限：

			```
			lightsail:GetBlueprints
			lightsail:GetBundles
			```
	- SSH Key

		如果不使用默认SSH Key，可以[创建一个](https://lightsail.aws.amazon.com/ls/webapp/account/keys)

	- Toolchain

		- [AWS CLI](https://docs.aws.amazon.com/zh_tw/cli/latest/userguide/cli-chap-welcome.html)

			执行`aws configure`配置`Access Key ID`，`Secret Access Key`和`Default region name`（ap-northeast-1）

- 脚本`create-lightsail.sh`

```
#!/bin/bash
INSTANCE_NAME='debian-11-v2ray'

ret=$(aws lightsail get-instance --no-cli-pager \
      --instance-name $INSTANCE_NAME | grep 'NotFoundException')
if [ ! -z $ret ]; then
    aws lightsail create-instances --no-cli-pager \
      --instance-names $INSTANCE_NAME \
      --availability-zone ap-northeast-1a \
      --blueprint-id debian_11 \
      --bundle-id nano_2_0 \
      --ip-address-type ipv4 \
      --key-pair-name macbook-pro
    if [ $? -ne 0 ]; then
        echo -e "\033[31mcreate instance error\033[0m"
        exit 1
    fi
else
    echo -e "\033[32m$INSTANCE_NAME is exist\033[0m"
fi

aws lightsail open-instance-public-ports --no-cli-pager \
  --instance-name $INSTANCE_NAME \
  --port-info fromPort=443,protocol=TCP,toPort=443
if [ $? -ne 0 ]; then
    echo -e "\033[31mopen public port 443 error\033[0m"
    exit 1
fi

echo -n -e "\033[32mpublic ip:\033[0m"
aws lightsail get-instance --no-cli-pager \
  --instance-name $INSTANCE_NAME \
  | grep publicIpAddress | awk '{print $2}' | awk -F ',' '{print $1}'
```
