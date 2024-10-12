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

1. 公网访问使用了可行性较高的 **光猫桥接路由器拨号同时获取 IPv4 大内网和 IPv6 公网 /64 地址** 方案
2. v6 / v4 双栈访问采用了 **Cloudflare DNS Proxy** 通过 Cloudflare 的代理回源的方式实现
3. 80 / 443 端口封禁规避同样使用的是 **Cloudflare 回源指定端口** 的能力
4. 博客源代码使用 **Gitea 仓库存储 + Actions 编译自动化部署** 实现
5. Gitea 仓库 **直接在 Nas 上通过 Docker Compose 部署**
6. 博客公网访问采取 **Caddy 反向代理** 方案，通过 **caddy-git 插件实现 webhook 更新本地 git 仓库并用 fileserver 代理仓库目录** 实现网站内容更新，由于 Nas 性能羸弱，Caddy 服务部署在 MacMini 上
7. TLS 采用 **caddy-dns 插件实现，用 Cloudflare token 实现 dns 验证，Caddy 自动部署证书**
8. Gitea Actions **容器化部署在 MacMini 端，用本地链接访问 Nas 上的 Gitea 服务**，减少 Cloudflare 代理流量和延迟
9. 博客自动化部署 Actions 采用 **Main 分支 Checkout - 环境配置 - 文件编译、拷贝、暂存 - Publish 分支 Checkout - 文件覆盖 - Commit 提交 - 触发 Caddy-git 更新 Webhook** 链路

全方案的拓扑图如下

![blog_topology](blog_topology.png)

后面有空再慢慢翔实
