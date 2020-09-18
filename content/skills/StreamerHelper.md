---
title: "StreamerHelper"
date: 2020-09-18T00:05:05+08:00
draft: false
indent: true
indentFirstParagraph: true
comments: true
toc: false
tags: []
---



<img src="https://camo.githubusercontent.com/657a163e660f32aae67fce0af8cbdef47251f106/68747470733a2f2f73312e617831782e636f6d2f323032302f30372f32322f55624b4370712e706e67" style="zoom:20%;" />

## Introduction

`StreamerHelper`是一个开源项目，可以实现虎牙、斗鱼、AfreecaTV等主流直播平台主播开播时自动录像，并适时上传。Github地址：[https://github.com/ZhangMingZhao1/StreamerHelper](https://github.com/ZhangMingZhao1/StreamerHelper)。

`StreamerHelper`使用`Node.js`开发，同时集成了直播录制和视频上传的功能，软件部署后，后台实时批量监测各个平台主播是否在线，并使用`FFmpeg`录制直播保存为视频文件，停播后投稿到b站。

## Project Tree

```
StreamerHelper
├─ src
│  ├─ app.ts
│  ├─ engine
│  │  ├─ getStreamUrl.ts
│  │  ├─ liveStreamStatus.ts
│  │  ├─ message.ts
│  │  ├─ RoomStatus.ts
│  │  └─ website
│  │     ├─ afreecatv.ts
│  │     ├─ bilibili.ts
│  │     ├─ cc.ts
│  │     ├─ ...
│  ├─ type
│  │  ├─ StreamInfo.ts
│  │  └─ VideoPart.ts
│  ├─ uploader
│  │  ├─ caller.ts
│  │  ├─ example.js
│  │  ├─ index.js
│  │  └─ uploadStatus.ts
│  └─ util
│     ├─ crypt.js
│     └─ utils.ts
└─ templates
   └─ info.json
```

`./src/`目录下存放项目源码，`src/app.ts`为程序入口文件，并且管理下载和上传的业务。

`./src/engine/`目录下为直播流获取及下载模块。

`./src/engine/website/`目录下为获取直播流的模块，以插件的方式集成，统一使用`axios`作为请求库，方便扩展。

`./src/uploader/`目录下为文件上传/投稿模块，部分代码使用`js`编写，方便直接调用上传。

`./src/type/`目录下定义了一系列接口。

`./src/util/`目录下封装了一些工具方法，如加盐、解析`info.json`等。

`./templates/info.json`保存用于投稿的信息，所需录制主播信息。

日志文件会自动创建，在`./logs/`下。

## How does it work?

程序启动之后，首先`app.ts`里有一个大的定时器`const timer = setInterval(F, 12000);`，每`12s`遍历一遍`info.json`

里的主播信息，并调用`getStreamUrl()`检测主播是否在线，如果检测到在线，会实例化一个`Recorder`，同时将直播流链接传入构造器，启动下载线程，同时，会有一个监听器监听直播流断开事件。

`Recorder`类的代码在`message.ts`里，实例化后，会调用子进程启动`FFmpeg`来以原码率、原音视频编码的格式下载传入的直播流（占用极小，实际测试中占用不到CPU的`1%`）。直播流断开后，触发`streamDiscon`事件，事件触发后，程序会根据直播流是否为正常断开，来选择是否上传。

项目以异步的方式下载多个直播流，并且由统一的事件触发器处理直播流断开的情况。当用户向进程发送`SIGINT`信号后，程序会移除监听`streamDiscn`事件，并等待`FFmpeg`子进程结束后退出。

下载的录播文件会暂时存放于`./download/`下，并在上传成功后自动删除。

## Troubles & Solutions

在获取直播源后的地址可以用`PotPlayer`播放器正常播放，但无法使用`FFmpeg`下载，并且返回了`403 Forbidden`错误，多次测试后发现，`PotPlayer`在播放直播流时默认携带了一系列`Headers`信息，之后在`FFmpeg`启动参数中加入`Headers`信息，可以成功下载。

项目经历过一次比较大的结构调整，主要是将下载器包装成一个`Recorder`类，可以通过实例化来异步地单独下载每一个直播流，并统一由`app.ts`管理，这么做的好处是解下载模块与上传模块的耦合，降低复杂度。

当需要强制使一个`FFmpeg`子进程退出时，可以通过向子进程的`stdin`的可写流写入`q`来通知`FFmpeg`主动退出。

上传模块中，为了避免频繁登录触发验证码，使用了长期有效的`access_token`来避免验证。

## 未来计划

- [ ] 支持`Twitch`。
- [ ] 支持`docker`部署。
- [ ] 爬虫定时区间，节省服务器流量...
- [ ] 重启后同时检测本地是否有上传失败的视频文件，并上传。