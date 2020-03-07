---
title: 跨平台flutter框架与go构建音乐聊天应用
tags: [跨平台, flutter, dart, golang, app]
date: 2020-03-07 16:18:36
---

## 前言

flutter是谷歌2018年发布的一款跨平台移动UI框架，完全开源免费。flutter采用dart语言开发，这是一款融合了javascript和java特点的开发语言，支持静态编译和动态编译，开发时热重载。通过dart语言开发flutter应用，我们可以快速构建高质量应用。

而golang也是谷歌发布的一款开源编程语言，它的特点是简单，可靠并且非常高效，go原生支持协程，能够支持更大的并发量。由于它简单并且强大的功能，go在当下越来越流行。知乎，bilibili包括腾讯等大量公司都在使用。

##环境搭建

首先要构建应用，我们首先要搭建开发环境，包括dart环境，flutter框架还有安卓模拟器帮助我们调试应用，以及后端开发用到的golang开发环境。

### flutter

首先我们要先下载sdk包，这里以macos为例

1，去官网下载[flutter SDK](https://flutter.dev/docs/development/tools/sdk/releases?tab=macos#macos "flutter sdk下载链接")

2，解压安装包到自定义安装目录，这里我放置在用户主目录下

```bash
cd ~/soft/flutter
```

3，添加flutter环境变量

```bash
export PATH=`pwd`/flutter/bin:$PATH
```

到这里，flutter sdk包已经成功安装，然后需要检查有哪些依赖需要安装，可以通过命令检查：

```bash
flutter doctor
```

根据信息一步步解决依赖。

### 平台设置

我们要选择是用安卓平台调试还是ios平台调试，选择其中的一个就可以，这里我选择的是安卓平台，使用安卓平台调试，一种是使用安卓模拟器，还有就是使用真机，通过usb连接电脑，就可以在手机上运行了，安卓模拟器运行非常吃资源，使用真机调试会比较流畅，这里看自己选择。

### 编辑器配置

编辑器对编程开发效率影响很大，一个好用的编辑器就尤为重要了，开发flutter应用，我们可以使用vscode或intellig idea，这两种编辑器功能都非常强大，不过各有特点，vscode相对来说更为轻便一点，但是功能没有idea全面。使用intellig idea的话如果电脑性能不佳会比较卡。这里我使用的是intellig idea。就以intellig来介绍。

#### 安装扩展

安装flutter还有dart扩展：

{% asset_img intellig_flutter_ext.png 依赖注入 %}

#### 选择dart sdk

创建flutter应用时会提示选择dart sdk，前面我们下载的flutter sdk包里已经包含有dart sdk，目录位置：

```bash
你的目录/flutter/bin/cache/dart-sdk
```

### 安装golang

安装golang就非常简单，我们直接在官网下载golang安装包，

这里以我为例，解压到/usr/local，像下面：

```bash
tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
```

然后定义环境变量：

```bash
export PATH=$PATH:/usr/local/go/bin
```

## 应用demo

{% asset_img example_1.png 音乐列表 %}

{% asset_img example_2.png 聊天列表 %}

{% asset_img example_3.png 聊天详情 %}


## 项目

[flutter应用](https://github.com/baifei2014/music-app "音乐聊天app")

[go应用](https://github.com/baifei2014/music-internal "音乐聊天后端")






