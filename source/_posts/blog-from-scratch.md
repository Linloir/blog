---
title: 基于 IPv6 公网地址、NAS 和 MacMini 的私有部署博客方案
date: 2024-10-12 22:21:13
tags:
  - 瞎捣鼓
categories:
  - 技术
---

## 方案速览

简单来说，方案分为了几个主要的部分：

1. 根据 [在 NAS 上部署自己的 Gitea 服务，无需公网服务器](https://blog.linloir.cn/2024/10/13/host-git-at-home/) 方案打通外网到家用 NAS / MacMini 的链路
2. 采用 Git 仓库 main 分支存放源码 + Gitea Actions 编译至 publish 分支实现源码及制品存储
3. 使用 [caddy-git](https://github.com/greenpau/caddy-git) 插件实现拉取 Git 仓库 publish 分支并作为 fileserver 由 Caddy 反向代理

全方案的拓扑图如下

其中红色线条为 HTTP 流量，蓝色线条为 DDNS-GO 流量，紫色线条为本地或 v6 直连的 ssh TCP 流量

![blog_topology](/img/blog-from-scratch/blog_topology.png)

---

## 环境准备

在配博客之前，我是先配好了 Nas 上的 Gitea 服务，可以参考 [在 NAS 上部署自己的 Gitea 服务，无需公网服务器](https://blog.linloir.cn/2024/10/13/host-git-at-home/) 这一篇博客来准备基本的网络环境和 Gitea 服务。

(也就是说，我是先搭好了 Gitea，然后实在不知道能拿干点什么，才决定把博客迁移回来的。有点为了醋包饺子的感觉哈哈，不过现在博客全部内容都运行在自己本地感觉还是颇有成就感的)

## 仓库配置

待后面补充~
