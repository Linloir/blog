---
title: Swift 学习笔记 - 配置 VS Code 开发 Flutter iOS Native 代码
date: 2025-05-15 09:53:53
tags:
  - Swift
  - iOS
  - Flutter
  - 编程语言
categories:
  - 技术
---

## 背景

这段时间学 Swift 就冲着一个 iOS 开发来的，这学完了必须给环境整上，虽然说 VS Code 和 XCode 搭配着用也没什么不好的，毕竟开发过程中再怎么也没法避免打开 XCode 的过程 (比如配置开发者签名之类的)，但是能够写码时全程不离开当前 IDE 终归还是让人感觉会舒服很多，遂研究如何在 VS Code 中配置 Swift 开发环境以及正常开发 Flutter/ios 项目

## 配置 Swift 插件

根据官方提供的文档 [Configuring VS Code for Swift Development](https://www.swift.org/documentation/articles/getting-started-with-vscode-swift.html)，可以在 VS Code 插件商店中下载 Swift 插件，即可开始编写一般的 Swift 项目了

{% note warning flat %}
在 Windows 上，Swift 插件可能会因为编码原因拒绝工作，例如出现错误 `Unable to parse output from 'swift package init --help'`，目前在本地未能解决，建议直接使用 ssh 远程 Mac 机器进行开发
{% endnote %}

安装完插件后，对于一般的普通 Swift 项目，插件看起来能够正常工作，包括 F5 运行等都可以正常使用。似乎编译器的依赖关系是通过 `Package.swfit` 解析的，正确配置 `Package.swift` 即可在一般项目里正常开发

## No such module 'Flutter'

创建完 Flutter 工程后，进入 `ios` 文件夹下 Flutter 创建的 swift 文件，会发现报错 `no such module 'Flutter'`，此时如果使用 XCode 打开同样会有此报错

由于是初始工程，没有添加任何使用了 native code 的插件，因此在 `ios` 文件夹下是**不会有 Podfile 文件的**，网上关于这个问题的解决方法大多是 `pod install`，对当前的问题没有任何效果，可以不用尝试了

误打误撞发现在 XCode 下选择 Product-Analyze 后，XCode 内报错消失，猜测是 XCode 成功找到了 Flutter 相关的库文件，然而 Swift 插件由于缺少 `Package.swift` 文件，依然无法找到 Flutter 模块

遂就此思路询问 Gemini，借助大模型强大的 DeepResearch 能力，得到了下述解决方案

### 安装 SweetPad

首先，在 VSCode 插件商店中搜索 SweetPad 插件并安装，安装完成后左侧工具栏会出现糖果图标

点击糖果图标进入 SweetPad 配置页面，大概率 SweetPad 已经自动识别了工程。如果没有，可能需要在 `.vscode/settings.json` 中手动添加配置:

```json
{
  "sweetpad.build.xcodeWorkspacePath": "ios/Runner.xcworkspace"
}
```

之后重新加载 VS Code

### 安装 xcode-build-server

使用 brew 安装 xcode-build-server:

```bash
brew install xcode-build-server
```

如果想的话，可以额外安装 xcbeautify (SweetPad 推荐):

```bash
brew install xcbeautify
```

### 生成 buildServer.json

打开命令面板 (或使用 `Ctrl` + `Shift` + `P` / `Command` + `Shift` + `P`)，搜索 `SweetPad`

选中 **SweetPad: Generate Build Server Config**

SweetPad 应该会提示选择对应的 scheme，例如 `Runner` 以及目标设备等

完成后，SweetPad 应该会生成例如 `buildServer.json` 或是 `compile_commands.json` 这样的文件，之后重新启动 VS Code 即可
