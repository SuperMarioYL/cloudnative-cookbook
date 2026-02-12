---
title: "HAMi 在 CNCF 生态中的定位"
weight: 3
---


![CNCF Sandbox](https://img.shields.io/badge/CNCF-Sandbox-blue?style=flat-square&logo=cncf)
![Kubernetes](https://img.shields.io/badge/Kubernetes-Native-326CE5?style=flat-square&logo=kubernetes)
![License](https://img.shields.io/badge/License-Apache_2.0-green?style=flat-square)

> 本文档阐述 HAMi 在 CNCF (Cloud Native Computing Foundation) 生态系统中的定位，分析其与 Kubernetes 调度框架的集成方式，以及与 Prometheus、Volcano 等其他 CNCF 项目的协作关系。

---

## 目录

- [1. HAMi 在 CNCF Landscape 中的位置](#1-hami-在-cncf-landscape-中的位置)
- [2. 与 Kubernetes 调度框架的集成](#2-与-kubernetes-调度框架的集成)
- [3. 与其他 CNCF 项目的协作](#3-与其他-cncf-项目的协作)
- [4. 生态系统全景图](#4-生态系统全景图)

---

## 1. HAMi 在 CNCF Landscape 中的位置

### 1.1 CNCF 项目层级

HAMi 于 2023 年被 CNCF 接纳为 **Sandbox** 级别项目，位于 CNCF Landscape 的 **Runtime** 领域下的 **Cloud Native Hardware/Device** 子类别。

```mermaid
flowchart TB
    subgraph cncf["CNCF 项目层级"]
        direction TB
        graduated["Graduated (毕业)"]
        incubating["Incubating (孵化)"]
        sandbox["Sandbox (沙箱)"]
    end

    subgraph graduated_projects["毕业项目"]
        k8s["Kubernetes"]
        prometheus_g["Prometheus"]
        containerd_g["containerd"]
    end

    subgraph incubating_projects["孵化项目"]
        volcano_i["Volcano"]
    end

    subgraph sandbox_projects["沙箱项目"]
        hami["HAMi"]
    end

    graduated --> graduated_projects
    incubating --> incubating_projects
    sandbox --> sandbox_projects

    style hami fill:#4CAF50,color:#fff,stroke:#388E3C,stroke-width:2px
    style sandbox fill:#FFF9C4
    style incubating fill:#BBDEFB
    style graduated fill:#C8E6C9
```

### 1.2 CNCF Landscape 分类定位

在 CNCF Landscape 的技术分类中，HAMi 属于以下类别：

| 分类层级 | 定位 | 说明 |
|:---------|:-----|:-----|
| 大类 | Runtime | 运行时层，管理计算资源 |
| 子类 | Cloud Native Hardware/Device | 云原生硬件与设备管理 |
| 关键词 | GPU Virtualization, Device Sharing | GPU 虚拟化、设备共享 |

### 1.3 HAMi 解决的 CNCF 生态问题

在 CNCF 的云原生技术栈中，HAMi 填补了**异构加速器虚拟化与共享**这一关键缺口。

```mermaid
flowchart LR
    subgraph gap["HAMi 填补的技术缺口"]
        direction TB
        native["Kubernetes 原生能力"]
        native --> n1["CPU/Memory - 精细调度"]
        native --> n2["Storage - CSI 标准"]
        native --> n3["Network - CNI 标准"]
        native --> n4["GPU - 仅整卡分配"]

        n4 -->|"HAMi 增强"| enhanced["HAMi 补充能力"]
        enhanced --> e1["GPU 细粒度共享"]
        enhanced --> e2["显存/算力隔离"]
        enhanced --> e3["多厂商统一管理"]
        enhanced --> e4["可观测性集成"]
    end

    style n4 fill:#FFCDD2
    style enhanced fill:#C8E6C9
```

---

## 2. 与 Kubernetes 调度框架的集成

### 2.1 Kubernetes 调度扩展机制

HAMi 通过 Kubernetes 官方提供的 **Scheduler Extender** 机制与默认调度器集成，无需修改 Kubernetes 核心代码。

```mermaid
flowchart TB
    subgraph k8s_scheduler["kube-scheduler 调度流程"]
        direction TB
        queue["调度队列"]
        prefilter["PreFilter"]
        filter_phase["Filter (过滤)"]
        score_phase["Score (评分)"]
        reserve["Reserve (预留)"]
        bind_phase["Bind (绑定)"]
    end

    subgraph extender_points["HAMi 扩展点"]
        ext_filter["Extender Filter
POST /filter"]
        ext_bind["Extender Bind
POST /bind"]
    end

    subgraph webhook_point["HAMi Webhook"]
        mut_webhook["Mutating Webhook
POST /webhook"]
    end

    subgraph device_plugin_point["HAMi Device Plugin"]
        dp_api["Device Plugin API
gRPC Allocate"]
    end

    queue --> prefilter
    prefilter --> filter_phase
    filter_phase -->|"HTTP 调用"| ext_filter
    ext_filter -->|"返回过滤结果"| filter_phase
    filter_phase --> score_phase
    score_phase --> reserve
    reserve --> bind_phase
    bind_phase -->|"HTTP 调用"| ext_bind
    ext_bind -->|"返回绑定结果"| bind_phase

    mut_webhook -.->|"Pod 创建时拦截
注入 schedulerName"| queue
    dp_api -.->|"Pod 启动时
注入 libvgpu.so"| bind_phase

    style ext_filter fill:#2196F3,color:#fff
    style ext_bind fill:#2196F3,color:#fff
    style mut_webhook fill:#FF9800,color:#fff
    style dp_api fill:#4CAF50,color:#fff
```

### 2.2 HAMi 使用的 Kubernetes 扩展点

| 扩展机制 | K8s 标准接口 | HAMi 实现 | 注入时机 |
|:---------|:-----------|:---------|:---------|
| Scheduler Extender | `v1beta1.Extender` (HTTP) | `scheduler-extender` | Pod 调度时 |
| Mutating Admission Webhook | `admissionregistration.k8s.io/v1` | `webhook.go` | Pod 创建时 |
| Device Plugin | `k8s.io/kubelet/pkg/apis/deviceplugin/v1beta1` (gRPC) | `device-plugin` | Pod 启动时 |
| Extended Resource | `node.status.capacity` | 虚拟化 GPU 资源 | 设备注册时 |
| Node Annotation | `node.metadata.annotations` | 设备信息编码 | 持续更新 |
| Lease | `coordination.k8s.io/v1` | Leader Election | 持续续约 |

### 2.3 与 Scheduling Framework 的对比

Kubernetes 1.19+ 提供了 Scheduling Framework (调度框架) 作为 Scheduler Extender 的替代方案。HAMi 目前使用 Extender 机制。

| 对比维度 | Scheduler Extender (HAMi 当前) | Scheduling Framework |
|:---------|:---------------------------|:--------------------|
| 部署方式 | 独立进程，HTTP 通信 | 编译到调度器二进制中 |
| 性能 | HTTP 序列化开销 | 内存调用，零序列化 |
| 扩展点 | Filter + Bind | PreFilter/Filter/Score/Reserve/Bind 等 20+ 扩展点 |
| 维护成本 | 独立部署，版本解耦 | 需与 kube-scheduler 版本绑定 |
| 多调度器 | 支持 | 支持 |
| 成熟度 | 稳定 | GA (Kubernetes 1.19+) |

> **设计选择**：HAMi 选择 Scheduler Extender 的主要原因是**版本解耦** -- HAMi 支持 Kubernetes 1.18+ 的广泛版本范围，而 Scheduling Framework 需要与特定 kube-scheduler 版本绑定编译。Extender 模式允许 HAMi 独立发布版本，降低用户升级成本。

---

## 3. 与其他 CNCF 项目的协作

### 3.1 Prometheus - 可观测性

HAMi 原生支持 Prometheus 指标格式，通过 vGPU Monitor 组件暴露 GPU 虚拟化层的监控数据。

```mermaid
flowchart LR
    subgraph hami_monitor["HAMi vGPU Monitor"]
        metrics_endpoint[":9394/metrics
Prometheus HTTP 端点"]
        shm_reader["共享内存读取器
读取 libvgpu.so 写入的数据"]
    end

    subgraph prometheus_stack["Prometheus 生态"]
        prometheus["Prometheus Server
指标采集与存储"]
        alertmanager["Alertmanager
告警管理"]
        grafana["Grafana
可视化面板"]
        servicemonitor["ServiceMonitor
(Prometheus Operator)"]
    end

    subgraph dcgm_stack["NVIDIA 原生监控"]
        dcgm["DCGM Exporter
物理 GPU 指标"]
    end

    shm_reader --> metrics_endpoint
    prometheus -->|"Scrape"| metrics_endpoint
    prometheus -->|"Scrape"| dcgm
    prometheus --> alertmanager
    prometheus --> grafana
    servicemonitor -->|"自动发现"| metrics_endpoint

    style metrics_endpoint fill:#E65100,color:#fff
```

**HAMi 暴露的核心指标：**

| 指标 | 类型 | 说明 |
|:-----|:-----|:-----|
| `hami_vgpu_memory_used_bytes` | Gauge | 容器实际 GPU 显存使用量 |
| `hami_vgpu_memory_limit_bytes` | Gauge | 容器 GPU 显存限额 |
| `hami_vgpu_core_used_percent` | Gauge | 容器 SM 使用率 |
| `hami_vgpu_core_limit_percent` | Gauge | 容器 SM 限额 |

**与 DCGM Exporter 的互补关系：**

| 监控层面 | DCGM Exporter | HAMi Monitor |
|:---------|:-------------|:-------------|
| 物理 GPU 温度 | 支持 | 不提供 |
| 物理 GPU 功耗 | 支持 | 不提供 |
| 物理 GPU 利用率 | 支持 | 不提供 |
| 容器级虚拟显存使用 | 不提供 | 支持 |
| 容器级虚拟算力使用 | 不提供 | 支持 |
| 容器级资源限额 | 不提供 | 支持 |

### 3.2 Volcano - 批处理调度

HAMi 与 Volcano 的集成提供了"GPU 虚拟化 + 批处理调度"的完整能力。

```mermaid
flowchart TB
    subgraph capability["能力矩阵"]
        direction LR
        subgraph hami_cap["HAMi 能力"]
            h1["GPU 细粒度共享"]
            h2["显存隔离"]
            h3["算力隔离"]
            h4["多厂商支持"]
        end

        subgraph volcano_cap["Volcano 能力"]
            v1["Gang Scheduling
组调度"]
            v2["Fair Scheduling
公平调度"]
            v3["Queue Management
队列管理"]
            v4["Job Lifecycle
作业生命周期"]
        end

        subgraph combined["组合能力"]
            c1["细粒度 GPU 共享
+ 组调度"]
            c2["多租户 GPU 配额
+ 公平调度"]
            c3["异构加速器管理
+ 批处理优化"]
        end

        hami_cap --> combined
        volcano_cap --> combined
    end
```

详细的集成配置请参考 [HAMi + Volcano 集成指南](../10-operations/02-volcano-integration/)。

### 3.3 containerd / CRI-O - 容器运行时

HAMi 的数据平面 (libvgpu.so) 通过 `LD_PRELOAD` 机制注入容器进程。该机制依赖容器运行时正确传递环境变量和 Volume 挂载。

| 容器运行时 | 兼容性 | 说明 |
|:----------|:-------|:-----|
| containerd | 完全兼容 | 推荐运行时 |
| CRI-O | 完全兼容 | |
| Docker (dockershim) | 兼容 (K8s < 1.24) | K8s 1.24+ 已移除 dockershim |

### 3.4 Helm - 包管理

HAMi 使用 Helm v3 Chart 作为标准部署方式，遵循 Helm Chart 最佳实践。

| Helm 特性 | HAMi 使用情况 |
|:----------|:-------------|
| Chart 版本管理 | Chart v2.8.0，与 appVersion 同步 |
| values.yaml 参数化 | 完整的参数化配置 |
| 子 Chart 依赖 | hami-dra (条件依赖) |
| Hooks | pre-install/post-install Job (证书生成) |
| RBAC 模板 | 完整的 ServiceAccount/Role/ClusterRole |

### 3.5 OpenTelemetry - 可观测性 (规划中)

HAMi 计划在未来版本中集成 OpenTelemetry，提供分布式追踪能力，覆盖从 Pod 调度到 GPU 分配的完整链路。

---

## 4. 生态系统全景图

### 4.1 CNCF 生态定位图

```mermaid
flowchart TB
    subgraph app_layer["应用层"]
        ai_framework["AI 框架
PyTorch / TensorFlow / MindSpore"]
        inference_engine["推理引擎
vLLM / Triton / TensorRT"]
    end

    subgraph orchestration["编排与调度层"]
        k8s_core["Kubernetes
(Graduated)"]
        volcano_orch["Volcano
(Incubating)
批处理调度"]
    end

    subgraph device_mgmt["设备管理层"]
        hami_mgmt["HAMi
(Sandbox)
GPU 虚拟化与共享"]
        device_plugin_api["Device Plugin API
(K8s 标准)"]
        dra_api["DRA API
(K8s 1.30+ Alpha)"]
    end

    subgraph runtime_layer["运行时层"]
        containerd_rt["containerd
(Graduated)"]
        nvidia_toolkit["nvidia-container-toolkit
容器 GPU 支持"]
        hamicore["HAMi-core
libvgpu.so
CUDA API Hook"]
    end

    subgraph observability_layer["可观测性层"]
        prometheus_obs["Prometheus
(Graduated)
指标采集"]
        grafana_obs["Grafana
可视化"]
        otel["OpenTelemetry
(Incubating)
分布式追踪"]
    end

    subgraph hardware["硬件层"]
        nvidia_hw["NVIDIA GPU
A100/H100/V100"]
        ascend_hw["华为 Ascend NPU
910/310"]
        cambricon_hw["寒武纪 MLU
思元系列"]
        other_hw["其他加速器
AMD/海光/天数智芯..."]
    end

    ai_framework --> k8s_core
    inference_engine --> k8s_core
    k8s_core --> volcano_orch
    k8s_core --> hami_mgmt
    hami_mgmt --> device_plugin_api
    hami_mgmt --> dra_api
    device_plugin_api --> containerd_rt
    containerd_rt --> nvidia_toolkit
    nvidia_toolkit --> hamicore
    hamicore --> nvidia_hw
    hami_mgmt --> ascend_hw
    hami_mgmt --> cambricon_hw
    hami_mgmt --> other_hw

    hami_mgmt -->|"指标暴露"| prometheus_obs
    prometheus_obs --> grafana_obs
    hami_mgmt -.->|"规划中"| otel

    style hami_mgmt fill:#4CAF50,color:#fff,stroke:#388E3C,stroke-width:3px
    style hamicore fill:#F44336,color:#fff
    style k8s_core fill:#326CE5,color:#fff
    style volcano_orch fill:#1976D2,color:#fff
    style prometheus_obs fill:#E65100,color:#fff
```

### 4.2 HAMi 在 AI 基础设施栈中的角色

```mermaid
flowchart LR
    subgraph stack["AI 基础设施技术栈"]
        direction TB
        l5["L5 - AI 应用层
LLM / CV / NLP / ASR"]
        l4["L4 - AI 框架层
PyTorch / TF / MindSpore / PaddlePaddle"]
        l3["L3 - 编排调度层
Kubernetes + Volcano + HAMi"]
        l2["L2 - 容器运行时层
containerd + HAMi-core (libvgpu.so)"]
        l1["L1 - 硬件驱动层
NVIDIA Driver / Ascend Driver / MLU Driver"]
        l0["L0 - 硬件层
GPU / NPU / MLU / DCU"]
    end

    l5 --> l4
    l4 --> l3
    l3 --> l2
    l2 --> l1
    l1 --> l0

    style l3 fill:#4CAF50,color:#fff,stroke:#388E3C,stroke-width:2px
    style l2 fill:#F44336,color:#fff
```

### 4.3 与同类 CNCF 项目的差异化

| 项目 | CNCF 级别 | 核心领域 | 与 HAMi 关系 |
|:-----|:---------|:---------|:------------|
| Kubernetes | Graduated | 容器编排 | HAMi 的运行平台 |
| Volcano | Incubating | 批处理调度 | 互补集成 |
| Prometheus | Graduated | 监控告警 | HAMi 指标消费者 |
| containerd | Graduated | 容器运行时 | HAMi 容器底座 |
| KubeVirt | Incubating | VM 管理 | 无直接关系 |
| KubeEdge | Incubating | 边缘计算 | 潜在集成 (边缘 GPU) |
| Akri | Sandbox | 边缘设备发现 | 类似领域 (设备管理) |

---

> **文档版本：** v1.0
>
> **适用 HAMi 版本：** v2.x
>
> **最后更新：** 2025-05
