---
title: 'Frigate安装配置'
date: '2025-03-18T08:49:07+08:00'
lastmod: '2025-03-18T08:49:07+08:00'

draft: false

tags: ["homelab"]
author: "afterain"
---
### 简介

[Frigate](https://docs.frigate.video/) 是一个开源的视频监控系统，专为实时对象检测设计，通常用于家庭安全摄像头管理。有几个特点非常适合智能家居监控场景：

- 利用机器学习模型（如 YOLO）进行实时的物体检测，能够检测和识别摄像头捕获的运动物体（如人、车辆等）

- 支持硬件加速（如使用 GPU 或 Google Coral TPU），能够提高视频处理性能

- 它集成了Home Assistant

- 支持录制、事件检测和通知功能

更多特性参考[官方介绍]((https://docs.frigate.video/))

#### 前提条件

- [homelab PaaS]({{< ref "homelab-paas" >}})和[homelab PaaS扩展]({{< ref "homelab-paas-extension" >}})（提供了存储和密码保护方案）

- 网络摄像头若干

### 配置

在PaaS的K8S + GitOps基础上，只需要对应写Yaml配置文件即可

#### Frigate

详细参考[Frigate Chart文档](https://github.com/blakeblackshear/blakeshome-charts/tree/master/charts/frigate)

- nodeSelector

	可以指定使用K8S集群的一个具体节点，例如有硬件加速能力的节点。

- ingress

	配置Ingress域名，可以在浏览器直接输入域名访问（需要自行解决域名解析）

- persistence

	配置视频录像使用的存储。这里使用PaaS提供的NFS对应的Persistent Volume Claim名

- envFromSecrets

	配置密码来源使用Secret，避免直接填写密码的明文

- image

	默认镜像使用的ghcr仓库，国内访问比较慢，使用`ghcr.nju.edu.cn`作为镜像加速

- config

	[Frigate的配置](https://docs.frigate.video/guides/getting_started/#configuring-frigate)

	- mqtt：关闭MQTT

	- record：视频内容的录制。不需要录制所有时间，仅录制有motion变化的视频，最长保持30天。

	- ffmpeg：preset-rpi-64-h264 [树莓派的H264硬件解码](https://docs.frigate.video/configuration/hardware_acceleration/#raspberry-pi-34)

	- cameras

		萤石网络摄像头有2个视频流，主流分辨率、码率高用来存储（这样能存储更好的画质），子流分辨率用来做检测（在没有硬件加速的情况下，可以减轻CPU负载）

	- motion

		不需要检测的区域坐标（坐标可以使用Frigate的UI界面定位区域后获得）。例如视频画面中显示时间的位置，这样可以减少检测的范围（也可以避免误检测），进一步减少CPU消耗。

- 版本问题

	helm chart版本使用最新的tag 7.8.0，对应的app版本是0.14.1。但是官方文档是0.15.0版本的最新规范，使用旧版本会出现参数会提示错误`Extra Inputs Not Supported`。所以需要指定image tag = 0.15.0。

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frigate
spec:
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  source:
    chart: frigate
    repoURL: https://blakeblackshear.github.io/blakeshome-charts
    targetRevision: 7.8.0
    helm:
      releaseName: frigate
      values: |
        nodeSelector:
          kubernetes.io/hostname: raspberrypi
        ingress:
          enabled: true
          hosts:
            - host: frigate.homelab.afterain.xyz
              paths:
                - path: '/'
                  portName: http
        persistence:
          media:
            enabled: true
            existingClaim: nas-nfs
        envFromSecrets:
          - frigate-rtsp-password
        image:
          repository:
            ghcr.nju.edu.cn/blakeblackshear/frigate
          tag:
            0.15.0
        config: |
          mqtt:
            enabled: False
          record:
            enabled: True
            retain:
              days: 30
              mode: motion
            alerts:
              retain:
                days: 30
            detections:
              retain:
                days: 30
          snapshots:
            enabled: True
            retain:
              default: 30
          ffmpeg:
            hwaccel_args: preset-rpi-64-h264
          cameras:
            door:
              enabled: True
              ffmpeg:
                inputs:
                  - path: rtsp://admin:{FRIGATE_RTSP_PASSWORD_DOOR}@192.168.0.10/h264/ch1/main/av_stream
                    roles:
                      - record
                  - path: rtsp://admin:{FRIGATE_RTSP_PASSWORD_DOOR}@192.168.0.10/h264/ch1/sub/av_stream
                    roles:
                      - detect
              motion:
                mask:
                  - 0.801,0,1,0,1,1,0.813,1
                  - 0.003,0.01,0.304,0.005,0.27,0.998,0.003,0.995
  destination:
    server: "https://kubernetes.default.svc"
    namespace: default
```

#### 密码

配置文件

```
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  creationTimestamp: null
  name: frigate-rtsp-password
spec:
  encryptedData:
    FRIGATE_RTSP_PASSWORD_DOOR: placeholder
  template:
    metadata:
      creationTimestamp: null
      name: frigate-rtsp-password
```

- Secret名

	`frigate-rtsp-password`要和前面的`envFromSecrets`对应。

- 变量

	- `FRIGATE_RTSP_PASSWORD_DOOR`变量名必须是`FRIGATE_`开头。
	
	- 变量值随意填写，placeholder只是占位符。后面使用`Sealed-Secrets`加密会自动覆盖。

	- 变量可以设置若干个，每个变量都需要执行一次加密进行设置

- 加密

	执行`kubeseal`命令加密：
	
	```
	echo -n $secret_value | kubectl create secret generic example --dry-run=client --from-file=$secret_key=/dev/stdin -o json  | kubeseal -o yaml --merge-into $seal_secret_file
	```

	- $seal_secret_file是配置文件（例如`./frigate-password.yaml`）

	- $secret_value是加密变量名（例如`FRIGATE_RTSP_PASSWORD_DOOR`）

	- $secret_value是加密内容（例如`11223344`）

	配置文件中的变量值在加密后覆盖。

### 自动化

有了GitOps基础，只需要把Yaml配置文件提交到git仓库，就能自动化安装配置
