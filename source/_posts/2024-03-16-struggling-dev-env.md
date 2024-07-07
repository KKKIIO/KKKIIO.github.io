---
layout: post
title: "开发环境夹缝求生"
date: 2024-03-16 17:33:00 +0800
tags: [work]
---

我最近工作的后端项目很难在本地运行。

<!--more-->

首先是程序依赖的服务和中间件太多，管理很繁琐。

其次是软件反盗版设计只考虑生产环境，本地环境缺少证书和几个 Linux 特有文件就无法运行。

好消息是近几年的远程开发工具发展很快，有了一些高效的解决方案。

## Devcontainers

Development Containers(Devcontainers) 是一个配置开发环境的协议。
它使用 Docker 容器技术，告诉 IDE 如何生成一个开发专用的容器，并在容器里“远程开发”。

Devcontainers 继承了几个容器虚拟化的优点：

### 优点一：随时随地可重建

在项目下加一个 `.devcontainer` 目录，就可以在任意支持 Docker 的机器上生成一样的开发环境。
不用再手动教新人如何配置本地环境。

![VSCode Devcontainers](https://microsoft.github.io/vscode-remote-release/images/remote-containers-readme.gif)

### 优点二：统一工具链

如果你用过 Protobuf ，你一定遇到过同事生成的文件跟你不一样，导致 git merge 时产生冲突。

Devcontainers 可以在 `Dockerfile` 里统一安装、配置工具，避免各自发挥导致工具冲突的问题。

### 优点三：近似生产环境

Devcontainers 可以统一使用 Linux 容器，兼容项目里假定系统是 Linux 的代码。

### 优点四：依赖服务齐全

调试的功能需要数据库、缓存、消息队列？还要其他团队的服务？
全都可以在一个 `docker-compose.yaml` 里配置。

之前想尝试加集成测试，但碍于在本地启动程序依赖的服务太多。
现在打开 VSCode 就能访问所有服务，也顺利加上了集成测试。

### 缺点：对现有部署方案复用程度不高

Devcontainers 通常与生产环境部署方案独立。

虽然能复用镜像，但复用不了服务的编排和配置。
这意味着同事们的工作成果不一定跟你的开发环境兼容。

## Kubernetes Remote

被动跟随的效率太低，主动推广开 Devcontainers 的难度也不小，有没有更大程度复用现有编排的方案呢？

[Okteto - Kubernetes Remote](https://www.okteto.com/blog/remote-kubernetes-development/) 让你可以在一个 Kubernetes 集群上开发。
借助 devops 同事做好的开发环境 Kubernetes 集群，你可以享受到完整的服务依赖。

Okteto 远程开发的[方案](https://www.okteto.com/docs/reference/okteto-cli/#up)是替换一个 Kubernetes deployment ，继承它的 configmap, secrets 等配置，替换掉 container，
然后在新的开发容器里启一个 SSH Server。 完成这些，你就有了一个可以访问各种中间件依赖的 SSH 远程服务器。

![okteto dev env](https://www.okteto.com/docs/assets/images/sync-development-arch-39b04674ed6df21af0a79cd9cc269387.png)

更好的是，很多现代 IDE 支持远程到 SSH 服务器开发。
你可以继续用 [VsCode](https://code.visualstudio.com/docs/remote/ssh) 或者 [JetBrains IDE](https://www.jetbrains.com/remote-development/gateway/) ，享受丝滑的开发、调试体验。
Okteto 会自动在你本地和远程服务器之间同步文件，不用担心改动丢失。

![](https://code.visualstudio.com/assets/docs/remote/ssh/architecture-ssh.png)

### 适合调试

这个方案偏 hack ，会遇到一些开发操作与 Kubernetes 编排不兼容的情况。
例如 configmap 挂载的文件是只读的，改配置需要改编排。

实践上看，至少是一个方便的远程调试方案。
