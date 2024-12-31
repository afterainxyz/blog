---
title: "FFmpeg如何添加一个Codec"
subtitle: ""
date: 2022-10-26T08:27:45+08:00
lastmod: 2022-10-26T08:27:45+08:00
draft: false
author: "afteraincc"
authorLink: "afterain.cc"
description: ""
license: ""
images: []

tags: ['FFmpeg']
categories: ['技术']

featuredImage: "/img/posts/FFmpeg-Logo.svg"
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

**支持FFmpeg 4.2版本**

用decoder做示例：

<!--more-->

- libavcodec/allcodecs.c

	`extern AVCodec ff_h264_xxx_decoder;`

- libavcodec/Makefile

	`OBJS-$(CONFIG_H264_XXX_DECODER)            += xxx.o yyy.o`

- libavcodec/xxx.c  yyy.c

	代码实现

- 执行configure

	新增一个codec（无论是decoder/encoder），必须执行一次。执行完成后，`ffbuild/config.mak`会自动添加`CONFIG_H264_XXX_DECODER=yes`，`libavcodec/codec_list.c`会自动添加`&ff_h264_xxx_decoder,`，编译出来的ffmpeg才会有对应的codec。

	**以上只是结果，未深入分析configure脚本如果实现**

- 编译结果

	make后执行`./ffmpeg -codecs |grep 264`查询结果是否包含新的codec（AVCodec结构体对应的字段`name`）

### 如何验证/测试

- 建议

	ffmpeg本身静态编译（仅指静态链接libavxxx库，不含例如libx264等其它库），方便测试（不需要每次make install以及设置LD\_LIBRARY\_PATH）

	例子（关键参数`--pkg-config-flags="--static"`）：
	`./configure --prefix=./zy_build --pkg-config-flags="--static" --extra-libs="-lpthread -lm -ldl -static-libgcc" --enable-gpl --disable-libaom --enable-libx264 --enable-libx265 --enable-nonfree --disable-shared --enable-static --disable-ffplay --disable-ffprobe --disable-doc`

- 如何验证解码器

	- 指定使用自定义的解码器（dummy_h264_dec是解码器名，根据实际情况填写）

		`./ffmpeg -vcodec dummy_h264_dec -i video.264 -f rawvideo -an test.yuv -y`

	- video.264可以用标准版的ffmpeg+libx264生成

		`ffmpeg -ss 0 -t 1 -i video.mp4 -vcodec libx264 -pix_fmt yuv420p -an video.264 -y`

	- 播放yuv文件（1104x622是分辨率，根据实际情况填写）

		`ffplay -f rawvideo -video_size 1104x622 -pixel_format yuv420p test.yuv`

- 如何验证编码器

	- 指定使用自定义的编码器（1104x622是分辨率/dummy_h264_enc是编码器名，根据实际情况填写）

		`./ffmpeg -s  1104x622 -r 25 -pix_fmt yuv420p -i video.yuv  -an -vcodec dummy_h264_enc test.264 -y`

	- video.yuv可以用标准版的ffmpeg生成

		`ffmpeg -ss 0 -t 1 -i video.mp4 -vcodec rawvideo -pix_fmt yuv420p -an video.yuv -y`

	- 播放264文件

		`ffplay test.264`

### 实现Codec

- FFmpeg的Codec框架

	- AVCodec

		AVCodec可以认为是FFmpeg编解码的类/结构定义，新增一个Codec（编码或解码）时，根据场景实现对应的回调函数即可。例子见下面`encoder/decoder`部分。

		- 通用字段

			`.name/.long_name/.type/.id`

	- `priv_data_size`字段

		AVCodec只是FFmpeg编解码的类/结构定义。Codec的具体实现是单独定义一个新的类/结构，用来封装/隔离FFmpeg运行时，一个Codec的多个实例的内部私有参数/变量。

		在回调函数中，有一个公共参数：AVCodecContext *avctx，`avctx->priv_data`就是Codec实现类/结构的实例指针。因为C语言实现的一些细节，FFmpeg使用`priv_data_size`字段来定义实现类/结构的大小（以便用来分配实例的内存），例子： `.priv_data_size = sizeof(XXXH264EncCtx),`

	- encoder

		最小函数集合：

		- `.init`，初始化
		- `close`，释放
		- `encode2`，编码

		示例：

		```
		typedef struct _DummyH264EncCtx {
		} DummyH264EncCtx;
		```

		```
		AVCodec ff_h264_dummy_encoder = {
		    .name           = "dummy_h264_enc",
		    .long_name      = NULL_IF_CONFIG_SMALL("Dummy H264 encoder"),
		    .type           = AVMEDIA_TYPE_VIDEO,
		    .id             = AV_CODEC_ID_H264,
		    .priv_data_size = sizeof(DummyH264EncCtx),
		    .init           = ff_dummy_h264_encode_init,
		    .encode2        = ff_dummy_h264_encode_frame,
		    .close          = ff_dummy_h264_encode_close,
		};
		```

	- decoder

		最小函数集合：

		- `.init`，初始化
		- `close`，释放
		- `decode`，解码

		示例：

		```
		typedef struct _DummyH264DecCtx {
		} DummyH264DecCtx;
		```

		```
		AVCodec ff_h264_dummy_decoder = {
		    .name           = "dummy_h264_dec",
		    .long_name      = NULL_IF_CONFIG_SMALL("Dummy H264 decoder"),
		    .type           = AVMEDIA_TYPE_VIDEO,
		    .id             = AV_CODEC_ID_H264,
		    .priv_data_size = sizeof(DummyH264DecCtx),
		    .init           = ff_dummy_h264_decode_init,
		    .close          = ff_dummy_h264_decode_close,
		    .decode         = ff_dummy_h264_decode_frame,
		};
		```

- 如何从FFmpeg命令行传递参数到Codec内部

	- 参考文档

		[FFmpeg官方文档-AVOption](http://www.ffmpeg.org/doxygen/4.1/group__avoptions.html)

		[FFmpeg源代码简单分析-AVOption](https://blog.csdn.net/leixiaohua1020/article/details/44268323)

	- AVClass和AVOption

		AVCodec的`priv_class`字段（类型是AVClass指针）结合FFmpeg的AVOption机制，能实现从FFmpeg命令行传递参数到Codec的内部实现类/结构。由于一些FFmpeg实现的细节，实际需要定义一个AVClass的类/结构，加上一个AVOption数组，实际的参数定义是在AVOption数组中实现。

		AVOption的每项定义一个参数（并关联到Codec实现类/结构的字段）。第一个参数是FFmpeg命令行的参数名称，例如`ffmpeg -i xxx -vcodec dummy_h264_enc -preset medium -dummy-params a=1:b=2 test.264`（详细实现参考下面例子）。第三个参数关联Codec实现类/结构的字段，例如`offsetof(DummyH264EncCtx, x)`（详细实现参考下面例子）。

		例子：

		```
		AVCodec ff_h264_dummy_encoder = {
			//xxx
		    .priv_class     = &dummy_h264_encoder_class,
		}
		```

		```
		static const AVClass dummy_h264_encoder_class = {
		    .class_name = "dummy_h264_enc",
		    .item_name  = av_default_item_name,
		    .option     = dummy_h264_encoder_options,
		    .version    = LIBAVUTIL_VERSION_INT,
		};
		```

		```
		#define OFFSET(x) offsetof(DummyH264EncCtx, x)
		#define VE AV_OPT_FLAG_VIDEO_PARAM | AV_OPT_FLAG_ENCODING_PARAM
		static const AVOption dummy_h264_encoder_options[] = {
		    { "preset", "Set the encoding preset", OFFSET(preset),        AV_OPT_TYPE_STRING, { .str = "medium" }, 0, 0, VE},
		    { "dummy-params",  "Override configuration using a :-separated list of key=value parameters", OFFSET(dummy_params), AV_OPT_TYPE_STRING, { 0 }, 0, 0, VE },
		    { NULL },
		};
		```


- 编译时增加依赖库

	执行./configure时，添加参数

	- 编译参数（例如头文件路径/变量定义 ）

		`--extra-cflags`

	- 链接参数（例如库文件路径/库）

		`--extra-ldflags` and `--extra-libs`
