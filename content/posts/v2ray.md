---
title: "安全上网工具V2Ray的安装配置"
subtitle: ""
date: 2022-10-26T07:56:03+08:00
lastmod: 2022-10-26T07:56:03+08:00
draft: false
author: "afteraincc"
authorLink: "afterain.cc"
description: ""
license: ""
images: []

tags: ['V2Ray', '科学上网']
categories: ['技术']

featuredImage: "/img/posts/V2Ray_logo.png"
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

- 下载

	`wget https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh`

- 安装

	`sudo ./install-release.sh`

<!--more-->

- 启动服务

	`systemctl enable v2ray && systemctl start v2ray`

### 配置

WS + TLS这种更安全的方式，依赖web服务器和HTTPS的证书。

- 域名

	域名服务商购买，并创建一个`A`记录类型，解析到自己的主机IP地址

- 证书

	个人使用时，证书可以不签名，配置客户端时勾选`transport setting`->`TLS`->`TLS allowInsecure`

	执行`openssl`命令：

	`openssl req -x509 -subj "/C=CN/ST=x/L=x/O=x/OU=x/CN=x" -nodes -days 3650 -newkey rsa:2048 -keyout x.key -out x.crt`

	重要参数说明：

	- 证书类型 `-x509`

	- 证书过期时间 `-days 3650` 天

	- 指定私钥输出文件  `-keyout x.key`

	- 指定证书输出文件 `-out x.crt`

	- 证书参数

		`/C`国家 `/ST`省或州 `/L`城市 `/O`组织单位 `/OU`组织部门 `CN`通用名（一般填写域名）

- nginx

	关键参数：

	- `server_name`：`x`是占位符，需要修改为自己的域名

	- `ssl_certificate`和`ssl_certificate_key`：`x`是占位符，x.key/x.crt是前面步骤中的`openssl`生成

	配置文件路径`/etc/nginx/sites-enabled/x.conf`，`x`是占位符，可以是任意有效文件名（建议用自己的域名），完整内容如下：

	```
server {
    listen       443 ssl;
    server_name  x;

    ssl_certificate      /etc/ssl/certs/x.crt;
    ssl_certificate_key  /etc/ssl/private/x.key;

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    location /v2ray {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:7879;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
    }
}
	```

- v2ray

	`id`可以使用`uuidgen`生成，配置文件`/usr/local/etc/v2ray/config.json`，完整内容如下（注：端口7879需要和nginx配置中一致）：

	```
	{
    "log": {
        "access": "/var/log/v2ray/access.log",
        "error": "/var/log/v2ray/error.log",
        "loglevel": "none"
    },
    "inbounds": [{
            "port": 7879,
            "protocol": "vmess",
            "settings": {
                "clients": [{
                        "id": "4BB35D9A-F164-4237-8FB9-4D3460F1556D",
                        "level": 0,
                        "alterId": 0
                    }
                ]
            },
            "streamSettings": {
                "network": "ws",
                "wsSettings": {
                    "path": "/v2ray"
                }
            }
        }
    ],
    "outbounds": [{
            "protocol": "freedom"
        }
    ]
	}
	```

### 网络加速

在`高延迟，高带宽` 的网络环境下，可以开启BBR来加速网络。

- 开启

	```
	echo "net.core.default_qdisc=fq" | sudo tee --append /etc/sysctl.conf
	echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee --append /etc/sysctl.conf
	sudo sysctl -p
	```

-  确认

	`lsmod | grep bbr`

	输出`tcp_bbr 20480 1`表示成功

### EveryAsCode自动化脚本

- 前置依赖


	- 参数

		- <domain>

			 域名，必选参数

		- [client_id]

			client_id，可选参数。默认空时会自动生成一个新的UUID作为client id

		例子：

		`sudo ./v2ray-ws+tls-install.sh v2ray.yourdomain.com 4E2D399A-5029-4440-ACA1-7E297FD80EC3`

	- Toolchain

		apt，nginx，openssl，sysctl，systemctl，unzip

		默认nginx，unzip未安装，脚本执行时会自动安装


- 脚本`v2ray-ws+tls-install.sh`

```
#!/bin/bash
function log_info() {
    echo -e "\033[32m$@\033[0m"
}

function log_warn() {
    echo -e "\033[33m$@\033[0m"
}

function log_error() {
    echo -e "\033[31m$@\033[0m"
}

function main() {
    local domain=$1
    local client_id=$2
    if [ -z $client_id ]; then
        client_id=$(uuidgen)
    fi

    log_info 'installing v2ray'
    systemctl is-enabled v2ray 2>/dev/null
    if [ $? -ne 0 ]; then
        wget https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh -o install-release.sh
        if [ $? -ne 0 ]; then
          log_error '  download v2ray error'
          return 1
        fi

        if ! type $cmd >/dev/null 2>&1; then
            apt install unzip -y
            if [ $? -ne 0 ]; then
              log_error '  install unzip error'
              return 1
            fi
        fi
        chmod +x install-release.sh && ./install-release.sh
        if [ $? -ne 0 ]; then
          log_error '  install v2ray error'
          return 1
        fi
    else
        log_warn '  v2ray has installed'
    fi

    log_info 'enabling v2ray service'
    systemctl enable v2ray
    if [ $? -ne 0 ]; then
        log_error '  enable v2ray service error'
        return 1
    fi

    log_info 'configuring ws+tls'
    log_info '  generating openssl cert'
    openssl req -x509 -subj '/C=CN/ST=x/L=x/O=x/OU=x/CN=x' -nodes -days 3650 -newkey rsa:2048 -keyout $domain.key -out $domain.crt
    if [ $? -ne 0 ]; then
        log_error '  generate ssl certificate error'
        return 1
    fi
    cp v2r.afterain.cc.crt /etc/ssl/certs/v2r.afterain.cc.crt
    cp v2r.afterain.cc.key /etc/ssl/private/v2r.afterain.cc.key

    log_info '  installing nginx'
    if ! type $cmd >/dev/null 2>&1; then
        apt install nginx -y
        if [ $? -ne 0 ]; then
            log_error '  install nginx error'
            return 1
        fi
    fi
    log_info '  configuring nginx'
    cat <<EOF | tee /etc/nginx/sites-enabled/$domain.conf 2>&1 1>/dev/null
server {
    listen       443 ssl;
    server_name  $domain;

    ssl_certificate      /etc/ssl/certs/$domain.crt;
    ssl_certificate_key  /etc/ssl/private/$domain.key;

    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  5m;

    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers  on;

    location /v2ray {
        proxy_redirect off;
        proxy_pass http://127.0.0.1:7879;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host \$http_host;
    }
}
EOF

    log_info 'configuring v2ray'
    cat <<EOF | tee /usr/local/etc/v2ray/config.json 2>&1 1>/dev/null
{
    "log": {
        "access": "/var/log/v2ray/access.log",
        "error": "/var/log/v2ray/error.log",
        "loglevel": "none"
    },
    "inbounds": [{
            "port": 7879,
            "protocol": "vmess",
            "settings": {
                "clients": [{
                        "id": "$client_id",
                        "level": 0,
                        "alterId": 0
                    }
                ]
            },
            "streamSettings": {
                "network": "ws",
                "wsSettings": {
                    "path": "/v2ray"
                }
            }
        }
    ],
    "outbounds": [{
            "protocol": "freedom"
        }
    ]
}
EOF

    log_info 'starting bbr'
    grep 'net.core.default_qdisc=fq' /etc/sysctl.conf
    if [ $? -ne 0 ]; then
        echo 'net.core.default_qdisc=fq' | tee --append /etc/sysctl.conf
    fi
    grep 'net.ipv4.tcp_congestion_control=bbr' /etc/sysctl.conf
    if [ $? -ne 0 ]; then
        echo 'net.ipv4.tcp_congestion_control=bbr' | tee --append /etc/sysctl.conf
    fi
    sysctl -p 2>&1 1>/dev/null
    if [ $? -ne 0 ]; then
        log_error '  start bbr error'
        return 1
    fi

    log_info 'restarting service'
    log_info '  restarting nginx service'
    systemctl restart nginx
    if [ $? -ne 0 ]; then
        log_error '  restart nginx error'
        return 1
    fi

    log_info '  restarting v2ray service'
    systemctl restart v2ray
    if [ $? -ne 0 ]; then
        log_error '  restart v2ray error'
        return 1
    fi

    log_info 'v2ray connfigure info:'
    log_info "  address: $domain"
    log_info "  port: 443"
    log_info "  client id: $client_id"
    log_info "  transport->network: ws"
    log_info "  transport->path: /v2ray"
    log_info "  transport->tls: Use TLS"
    log_info "  transport->tls->allowInsecure: true"
    log_info "  transport->tls->allowInsecure: true"
    log_info "  transport->tls->serverName: $domain"
}

if [ -n "$BASH_SOURCE" -a "$BASH_SOURCE" == "$0" ]; then
    if [ "$(id -u)" != "0" ]; then
        log_error "please run as root"
        exit 1
    fi
    if [ $# -eq 0 ]; then
        log_info "$(basename $0) <domain> [client_id]"
        exit 1
    fi

    apt update
    main $@
fi
```
