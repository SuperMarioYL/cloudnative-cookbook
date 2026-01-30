# Kubernetes 原理深度解析

本文档系列深入剖析 Kubernetes 的核心原理与实现机制，基于 Kubernetes 1.32 版本源码，帮助读者从代码层面理解 K8s 的工作原理。

## 文档导航

### 核心组件

| 模块 | 内容概述 | 文章数 |
|------|----------|--------|
| [01-架构总览](./01-overview/) | K8s 整体架构、核心概念、API 设计规范、控制循环模式 | 4 |
| [02-代码调试](./02-debugging/) | 开发环境搭建、本地集群、Delve 调试、E2E 测试 | 7 |
| [03-API Server](./03-apiserver/) | 启动流程、请求生命周期、认证授权准入、存储层 | 8 |
| [04-etcd](./04-etcd/) | Raft 协议、MVCC、K8s 集成、备份恢复 | 7 |
| [05-Controller](./05-controller/) | Informer 机制、WorkQueue、各类控制器实现 | 11 |
| [06-Scheduler](./06-scheduler/) | 调度框架、Filter/Score 插件、抢占机制 | 9 |
| [07-Kubelet](./07-kubelet/) | Pod 生命周期、CRI、容器管理、探针、DRA | 12 |

### 子系统

| 模块 | 内容概述 | 文章数 |
|------|----------|--------|
| [08-存储系统](./08-storage/) | CSI 架构、PV/PVC、StorageClass、卷快照 | 10 |
| [09-网络系统](./09-network/) | CNI、Service、kube-proxy、Ingress、Gateway API | 12 |
| [10-安全机制](./10-security/) | RBAC、Pod Security、Secrets、审计、加密 | 8 |
| [11-性能调优](./11-performance/) | 各组件调优、大规模集群优化、监控诊断 | 8 |

### 开发与运维

| 模块 | 内容概述 | 文章数 |
|------|----------|--------|
| [12-client-go](./12-client-go/) | RESTClient、Clientset、Informer、Leader Election | 8 |
| [13-Operator](./13-operator/) | CRD 设计、Kubebuilder、Operator SDK、测试 | 6 |
| [14-高可用](./14-ha/) | 控制平面 HA、etcd HA、灾难恢复、备份策略 | 5 |
| [15-使用示例](./15-examples/) | 部署模式、有状态应用、微服务、CI/CD | 6 |
| [16-生态集成](./16-ecosystem/) | Prometheus、Istio、Helm、Argo、OPA | 6 |
| [17-多集群](./17-multicluster/) | Karmada、Clusternet、MCS API | 4 |

## 快速开始

### 推荐阅读顺序

**入门路径** (理解 K8s 核心机制):
1. [01-架构总览](./01-overview/) - 建立整体认知
2. [03-API Server](./03-apiserver/) - 理解集群入口
3. [05-Controller](./05-controller/) - 掌握控制循环
4. [06-Scheduler](./06-scheduler/) - 了解调度决策
5. [07-Kubelet](./07-kubelet/) - 深入节点管理

**开发者路径** (构建控制器/Operator):
1. [12-client-go](./12-client-go/) - 掌握客户端编程
2. [05-Controller](./05-controller/) - 理解控制器模式
3. [13-Operator](./13-operator/) - 学习 Operator 开发

**运维路径** (集群管理与调优):
1. [04-etcd](./04-etcd/) - 数据存储管理
2. [14-高可用](./14-ha/) - 高可用架构
3. [11-性能调优](./11-performance/) - 性能优化
4. [02-代码调试](./02-debugging/) - 问题排查

## 代码版本

本文档基于 Kubernetes 1.32 稳定版源码分析，代码路径参考：

```
kubernetes/
├── cmd/                    # 各组件入口
│   ├── kube-apiserver/
│   ├── kube-controller-manager/
│   ├── kube-scheduler/
│   ├── kubelet/
│   └── kube-proxy/
├── pkg/                    # 核心实现
│   ├── controller/         # 控制器
│   ├── scheduler/          # 调度器
│   ├── kubelet/            # Kubelet
│   ├── proxy/              # kube-proxy
│   └── volume/             # 存储
└── staging/src/k8s.io/     # 独立发布的库
    ├── client-go/          # Go 客户端
    ├── apimachinery/       # API 机制
    └── apiserver/          # API Server 框架
```

## 图表说明

文档中使用 Mermaid 绘制架构图、流程图、时序图等。如果图表无法正常显示，请确保：
- 使用支持 Mermaid 的 Markdown 渲染器
- GitHub 原生支持 Mermaid 图表

## 贡献指南

欢迎提交 Issue 和 PR 来完善本文档：
- 发现错误或过时内容
- 补充更多实现细节
- 改进图表和示例

## 许可证

本文档遵循 Apache License 2.0 许可证。
