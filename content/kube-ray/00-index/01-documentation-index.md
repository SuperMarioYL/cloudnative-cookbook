---
title: "KubeRay 技术文档索引"
weight: 1
---

<p align="center">
  <img src="https://img.shields.io/badge/KubeRay-v1.3-blue?style=flat-square" alt="KubeRay" />
  <img src="https://img.shields.io/badge/Go-1.25-00ADD8?style=flat-square&logo=go" alt="Go" />
  <img src="https://img.shields.io/badge/License-Apache%202.0-green?style=flat-square" alt="License" />
  <img src="https://img.shields.io/badge/CNCF-Sandbox-blue?style=flat-square" alt="CNCF" />
  <img src="https://img.shields.io/badge/Docs-中文-red?style=flat-square" alt="Docs" />
</p>

---

## 项目简介

**KubeRay** 是 CNCF Sandbox 级别的开源项目，是 Ray 与 Kubernetes 之间的桥梁。它基于 controller-runtime 框架，通过 4 种 CRD（RayCluster、RayJob、RayService、RayCronJob）将 Ray 的分布式计算能力与 Kubernetes 的编排管理能力无缝对接，让用户以 Kubernetes 原生的声明式方式管理 Ray 工作负载。

> 本文档系列从源码层面深入剖析 KubeRay 的架构设计、核心原理与实现细节，涵盖 4 种 CRD 控制器、Pod 构建机制、高级特性、周边组件等全方位内容。

---

## 文档总览

| 模块 | 目录 | 文件数 | 定位 |
|:-----|:-----|:------:|:-----|
| [1. 架构总览与入门](#模块-1---架构总览与入门) | `01-architecture-overview/` | 3 | 全景认知 + 初学者入门 |
| [2. RayCluster Controller](#模块-2---raycluster-controller-深度解析) | `02-raycluster-controller/` | 4 | 核心控制器深度解析 |
| [3. RayJob Controller](#模块-3---rayjob-controller-深度解析) | `03-rayjob-controller/` | 3 | 任务生命周期管理 |
| [4. RayService Controller](#模块-4---rayservice-controller-深度解析) | `04-rayservice-controller/` | 3 | 服务化 + 零停机升级 |
| [5. Pod 创建与资源管理](#模块-5---pod-创建与资源管理) | `05-pod-creation/` | 2 | Pod 模板构建底层机制 |
| [6. 高级特性与扩展](#模块-6---高级特性与扩展机制) | `06-advanced-features/` | 4 | Autoscaler / 批调度 / GCS FT / Webhook |
| [7. 周边组件](#模块-7---周边组件深度解析) | `07-peripheral-components/` | 4 | APIServer / History Server / kubectl / Helm |
| [8. CronJob 与可观测性](#模块-8---cronjob-与可观测性) | `08-cronjob-and-observability/` | 2 | 定时任务 + Metrics 监控 |

**总计: 8 个模块, 25 篇文档**

---

## 模块 1 - 架构总览与入门

| # | 文档 | 内容概要 |
|:-:|:-----|:---------|
| 1.1 | [KubeRay 是什么](../01-architecture-overview/01-what-is-kuberay.md) | Ray 与 K8s 的关系、核心组件概览、4 种 CRD、项目目录结构 |
| 1.2 | [系统架构全景](../01-architecture-overview/02-overall-architecture.md) | controller-runtime 框架、main.go 启动流程、Feature Gates、多命名空间 Watch |
| 1.3 | [CRD 类型体系与 API 设计](../01-architecture-overview/03-crd-api-design.md) | RayCluster/RayJob/RayService/RayCronJob 完整字段解析、kubebuilder 标记、XValidation |

## 模块 2 - RayCluster Controller 深度解析

| # | 文档 | 内容概要 |
|:-:|:-----|:---------|
| 2.1 | [调和循环核心流程](../02-raycluster-controller/01-reconciliation-loop.md) | Reconciler 结构体、9 个 reconcileFunc 链、GCS FT Finalizer、四层 Validation |
| 2.2 | [Pod 调和与生命周期](../02-raycluster-controller/02-pod-reconciliation.md) | reconcilePods 深度分析、Suspend 处理、Head/Worker Pod 调和、Multi-host Worker |
| 2.3 | [状态计算与条件管理](../02-raycluster-controller/03-status-calculation.md) | calculateStatus 完整分析、5 种 Condition、Metrics 暴露 |
| 2.4 | [Debug 指南](../02-raycluster-controller/04-debug-guide-raycluster.md) | 调试环境搭建、5 个典型场景代码链路追踪、Event 解读、断点推荐 |

## 模块 3 - RayJob Controller 深度解析

| # | 文档 | 内容概要 |
|:-:|:-----|:---------|
| 3.1 | [RayJob 生命周期管理](../03-rayjob-controller/01-rayjob-lifecycle.md) | JobDeploymentStatus 10 种状态、4 种 SubmissionMode、BackoffLimit 重试、Dashboard Client |
| 3.2 | [删除策略与清理机制](../03-rayjob-controller/02-deletion-strategy.md) | Legacy 模式、Rules 模式、DeletionPolicyType、TTL 延迟、XValidation CEL |
| 3.3 | [Debug 指南](../03-rayjob-controller/03-debug-guide-rayjob.md) | 5 个常见问题追踪、Dashboard HTTP Client、K8s Submitter Job、Event 解读 |

## 模块 4 - RayService Controller 深度解析

| # | 文档 | 内容概要 |
|:-:|:-----|:---------|
| 4.1 | [RayService 调和核心流程](../04-rayservice-controller/01-rayservice-reconciliation.md) | Reconciler 结构体、Reconcile 14 步、Active/Pending 双集群、ServeConfig LRU 缓存 |
| 4.2 | [零停机升级机制](../04-rayservice-controller/02-zero-downtime-upgrade.md) | 3 种 UpgradeStrategy、NewCluster 蓝绿部署、Incremental Upgrade Gateway API 流量迁移 |
| 4.3 | [Debug 指南](../04-rayservice-controller/03-debug-guide-rayservice.md) | 5 个常见问题追踪、HTTP Proxy Client、Serve 状态查询链路 |

## 模块 5 - Pod 创建与资源管理

| # | 文档 | 内容概要 |
|:-:|:-----|:---------|
| 5.1 | [Pod 模板构建](../05-pod-creation/01-pod-template-construction.md) | DefaultHeadPodTemplate / DefaultWorkerPodTemplate、GCS FT 注入、Ray 启动命令构建、GPU/TPU 映射 |
| 5.2 | [Service / Ingress / RBAC](../05-pod-creation/02-service-ingress-rbac.md) | Head/Serve/Headless Service、Ingress/Route、Autoscaler RBAC、Redis Cleanup Job |

## 模块 6 - 高级特性与扩展机制

| # | 文档 | 内容概要 |
|:-:|:-----|:---------|
| 6.1 | [Autoscaler 集成](../06-advanced-features/01-autoscaler-integration.md) | In-tree Autoscaling、V1/V2 Autoscaler、Scale Expectations、WorkersToDelete |
| 6.2 | [批调度器集成](../06-advanced-features/02-batch-scheduler-integration.md) | BatchScheduler 接口、Volcano/YuniKorn/Kai Scheduler/Scheduler Plugins |
| 6.3 | [GCS 容错与 Redis 清理](../06-advanced-features/03-gcs-fault-tolerance.md) | GCS FT 配置、Pod 环境变量注入、Redis Cleanup Finalizer、Head 恢复 |
| 6.4 | [Webhook 验证机制](../06-advanced-features/04-webhook-validation.md) | 三层验证链路（CRD XValidation / Webhook / Controller）、CEL 表达式 |

## 模块 7 - 周边组件深度解析

| # | 文档 | 内容概要 |
|:-:|:-----|:---------|
| 7.1 | [APIServer gRPC+REST 网关](../07-peripheral-components/01-apiserver.md) | 双协议架构、5 个 gRPC Service、ResourceManager、Model 转换 |
| 7.2 | [History Server 事件回放](../07-peripheral-components/02-history-server.md) | 四层架构、EventHandler、API 端点、Live/Dead 集群路由、存储后端 |
| 7.3 | [kubectl Ray 插件](../07-peripheral-components/03-kubectl-plugin.md) | 命令体系、Cobra + client-go 实现 |
| 7.4 | [Helm Chart 部署指南](../07-peripheral-components/04-helm-charts.md) | 3 套 Chart、关键配置项、高可用部署 |

## 模块 8 - CronJob 与可观测性

| # | 文档 | 内容概要 |
|:-:|:-----|:---------|
| 8.1 | [RayCronJob 定时任务](../08-cronjob-and-observability/01-raycronjob-controller.md) | Feature Gate、Cron Schedule 解析、RayJob 创建、与 K8s CronJob 对比 |
| 8.2 | [Metrics 与可观测性](../08-cronjob-and-observability/02-metrics-and-monitoring.md) | Prometheus Metrics、日志系统、K8s Events、监控最佳实践 |
