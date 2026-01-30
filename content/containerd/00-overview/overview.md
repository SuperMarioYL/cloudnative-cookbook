---
title: "Containerd 概述"
weight: 1
description: "Containerd 容器运行时介绍"
---

Containerd 是一个行业标准的容器运行时，专注于简洁、健壮和可移植性。它是 CNCF 毕业项目，被 Kubernetes、Docker 等广泛使用。

## 核心特性

- **OCI 兼容**: 完全支持 OCI 镜像和运行时规范
- **CRI 实现**: 原生支持 Kubernetes Container Runtime Interface
- **插件架构**: 灵活的插件系统支持扩展
- **快照系统**: 高效的镜像层管理

## 章节规划

- 架构设计
- CRI 实现原理
- 镜像管理
- 容器生命周期
- 快照与存储
- 网络配置
