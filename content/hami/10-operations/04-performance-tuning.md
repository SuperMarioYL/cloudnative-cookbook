---
title: "HAMi 性能调优指南"
weight: 4
---


![Performance](https://img.shields.io/badge/Performance-Tuning-orange?style=flat-square)
![HAMi](https://img.shields.io/badge/HAMi-v2.x-blue?style=flat-square)

> 本文档系统介绍 HAMi 各组件的性能调优方法，包括调度器吞吐量优化、显存超配因子调优、设备切分数选择，以及基于监控数据的动态调优方法论。

---

## 目录

- [1. 性能调优总览](#1-性能调优总览)
- [2. 调度器性能调优](#2-调度器性能调优)
- [3. 显存因子调优](#3-显存因子调优)
- [4. 设备切分数优化](#4-设备切分数优化)
- [5. 基于监控的调优方法](#5-基于监控的调优方法)
- [6. 调优场景速查表](#6-调优场景速查表)

---

## 1. 性能调优总览

HAMi 的性能瓶颈可能出现在三个层面：**调度层**（Scheduler Extender 的请求处理吞吐量）、**控制层**（Device Plugin 的设备注册与 Allocate 延迟）、**数据层**（libvgpu.so 的 CUDA API Hook 开销）。以下图表展示了完整的性能指标体系。

```mermaid
flowchart TB
    subgraph scheduler_perf["调度层性能"]
        qps["API Server QPS/Burst
与 K8s API 通信吞吐"]
        lock_timeout["节点锁超时
nodeLockExpire"]
        filter_latency["Filter 延迟
节点遍历 + 评分计算"]
        bind_latency["Bind 延迟
锁获取 + API 调用"]
    end

    subgraph control_perf["控制层性能"]
        split_count["设备切分数
deviceSplitCount"]
        mem_scaling["显存超配比
deviceMemoryScaling"]
        core_scaling["算力超配比
deviceCoreScaling"]
        register_interval["注册扫描间隔
15s 固定周期"]
    end

    subgraph data_perf["数据层性能"]
        hook_overhead["CUDA API Hook 开销
dlsym 拦截延迟"]
        shm_contention["共享内存竞争
信号量等待时间"]
        rate_limiter_perf["令牌桶精度
SM 利用率控制粒度"]
        log_level["日志级别
LIBCUDA_LOG_LEVEL"]
    end

    scheduler_perf -->|"影响"| schedule_time["Pod 调度延迟"]
    control_perf -->|"影响"| resource_efficiency["资源利用效率"]
    data_perf -->|"影响"| cuda_performance["CUDA 应用性能"]
```

---

## 2. 调度器性能调优

### 2.1 QPS/Burst 设置

Scheduler Extender 通过 Kubernetes client-go 与 API Server 通信。在大规模集群中，默认的 QPS/Burst 设置可能成为瓶颈。

| 参数 | 默认值 | 推荐值 (大规模集群) | 说明 |
|:-----|:------|:-------------------|:-----|
| `--kube-qps` | `5` | `50-100` | 每秒最大请求数 |
| `--kube-burst` | `10` | `100-200` | 突发请求数上限 |
| `--kube-timeout` | `15` | `30` | API 调用超时时间 (秒) |

```mermaid
flowchart LR
    subgraph small["小规模集群 (< 50 节点)"]
        s_qps["QPS: 5"]
        s_burst["Burst: 10"]
        s_timeout["Timeout: 15s"]
    end

    subgraph medium["中规模集群 (50-200 节点)"]
        m_qps["QPS: 20"]
        m_burst["Burst: 40"]
        m_timeout["Timeout: 20s"]
    end

    subgraph large["大规模集群 (200+ 节点)"]
        l_qps["QPS: 50-100"]
        l_burst["Burst: 100-200"]
        l_timeout["Timeout: 30s"]
    end
```

**配置方式：**

```yaml
# values.yaml
scheduler:
  extender:
    extraArgs:
      - --debug
      - -v=4
      - --kube-qps=50
      - --kube-burst=100
      - --kube-timeout=30
```

**调优依据：**

- 如果 Scheduler 日志中频繁出现 `rate limiter Wait returned error` 或 `client rate limiter: context deadline exceeded`，说明 QPS 不足
- 可通过 Scheduler 的 Prometheus 指标 `rest_client_requests_total` 观察实际 API 调用量
- API Server 端可通过 `apiserver_request_total` 观察是否触发限流 (429)

### 2.2 节点锁超时

Scheduler 在 Bind 阶段使用分布式节点锁防止并发调度冲突。锁超时时间影响调度的并发度和容错能力。

| 参数 | 默认值 | 范围建议 | 说明 |
|:-----|:------|:--------|:-----|
| `scheduler.nodeLockExpire` | `5m` | `2m - 10m` | 节点锁超时时间 |

```mermaid
flowchart TB
    subgraph lock_behavior["节点锁行为"]
        acquire["获取锁
Patch Node Annotation"]
        hold["持有锁
执行 Bind 操作"]
        release["释放锁
删除 Annotation"]
        expire["锁超时
自动过期释放"]
    end

    acquire --> hold
    hold -->|"正常路径"| release
    hold -->|"异常路径
(Scheduler 崩溃)"| expire

    subgraph tuning["调优权衡"]
        short["超时时间短 (< 3m)"]
        long["超时时间长 (> 5m)"]

        short -->|"优点"| s_pro["崩溃后快速恢复
减少节点阻塞时间"]
        short -->|"风险"| s_con["正常 Bind 可能超时
导致误释放锁"]

        long -->|"优点"| l_pro["容忍慢速 API 调用
降低误超时风险"]
        long -->|"风险"| l_con["崩溃后节点长时间锁定
新 Pod 调度阻塞"]
    end
```

**调优建议：**

- 网络延迟高的环境（如跨 AZ 部署）：适当增大超时时间至 `8m-10m`
- 高并发调度场景：保持默认 `5m` 或适当减小
- 如果日志中出现 `node lock expired, releasing` 告警，说明超时时间过短

### 2.3 调度器资源配额

```yaml
# values.yaml
scheduler:
  extender:
    resources:
      requests:
        cpu: 200m
        memory: 256Mi
      limits:
        cpu: 2000m
        memory: 2Gi
  kubeScheduler:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 1000m
        memory: 1Gi
```

| 集群规模 | CPU Request | CPU Limit | Memory Request | Memory Limit |
|:---------|:-----------|:----------|:--------------|:-------------|
| < 50 节点 | 100m | 500m | 128Mi | 512Mi |
| 50-200 节点 | 200m | 1000m | 256Mi | 1Gi |
| 200+ 节点 | 500m | 2000m | 512Mi | 2Gi |

---

## 3. 显存因子调优

### 3.1 deviceMemoryScaling 参数

`deviceMemoryScaling` 控制 Device Plugin 向 kubelet 报告的显存总量与物理显存的比例。

```
报告显存 = 物理显存 x deviceMemoryScaling
```

| 设置值 | 效果 | 适用场景 |
|:------|:-----|:---------|
| `0.5` | 报告 50% 物理显存 | 预留显存给系统/驱动 |
| `1.0` (默认) | 报告 100% 物理显存 | 标准场景 |
| `1.5` | 报告 150% 物理显存 | 轻度超配 |
| `2.0` | 报告 200% 物理显存 | 激进超配 |

### 3.2 超配原理与风险

```mermaid
flowchart TB
    subgraph physical["物理 GPU - A100 80GB"]
        real_mem["实际可用显存: 80GB"]
    end

    subgraph scaling1["deviceMemoryScaling = 1.0"]
        report1["kubelet 看到: 80GB"]
        alloc1["最多分配: 80GB (8 x 10GB)"]
        risk1["风险: 无"]
    end

    subgraph scaling2["deviceMemoryScaling = 2.0"]
        report2["kubelet 看到: 160GB"]
        alloc2["最多分配: 160GB (16 x 10GB)"]
        risk2["风险: 物理显存不足时 OOM"]
    end

    physical --> scaling1
    physical --> scaling2
```

### 3.3 超配调优策略

```mermaid
flowchart LR
    subgraph decision["超配决策流程"]
        A["分析工作负载特征"] --> B{"大部分 Pod
实际使用显存
远低于申请量?"}
        B -->|"是"| C["可适度超配
1.2 - 2.0"]
        B -->|"否"| D["不建议超配
保持 1.0"]

        C --> E{"是否有监控
实际显存使用?"}
        E -->|"是"| F["根据监控数据
精确设置比例"]
        E -->|"否"| G["先设置 1.5
观察一周后调整"]

        F --> H["超配比 = 申请总量 / 实际峰值使用量
取 90% 安全系数"]
    end
```

**超配公式：**

```
安全超配比 = (所有 Pod 申请显存总和 / 所有 Pod 实际显存峰值) x 0.9
```

**示例：**

```
10 个 Pod 各申请 8GB 显存 = 80GB
实际峰值使用: 各 Pod 约 4GB = 40GB
安全超配比 = (80 / 40) x 0.9 = 1.8
```

**配置方式：**

```yaml
# 全局设置
devicePlugin:
  deviceMemoryScaling: 1.5

# 或按节点差异化设置
devicePlugin:
  nodeConfiguration:
    config: |
      {
        "nodeconfig": [
          {
            "name": "inference-node-01",
            "devicememoryscaling": 2.0
          },
          {
            "name": "training-node-01",
            "devicememoryscaling": 1.0
          }
        ]
      }
```

---

## 4. 设备切分数优化

### 4.1 deviceSplitCount 参数

`deviceSplitCount` 定义每块物理 GPU 最大可被切分为多少个虚拟 GPU 实例。这个值直接影响 kubelet 看到的 `nvidia.com/gpu` 资源数量。

```
虚拟 GPU 资源数 = 物理 GPU 数量 x deviceSplitCount
```

### 4.2 切分数与资源粒度

| deviceSplitCount | 最小可分配显存 (A100 80GB) | 最大共享 Pod 数 | 适用场景 |
|:-----------------|:-------------------------|:---------------|:---------|
| 5 | 16GB | 5 | 大模型训练，需要较多显存 |
| 10 (默认) | 8GB | 10 | 通用场景 |
| 20 | 4GB | 20 | 推理服务，小模型 |
| 50 | 1.6GB | 50 | 开发调试，显存需求极小 |

### 4.3 切分数优化流程

```mermaid
flowchart TB
    A["收集工作负载特征"] --> B["统计 Pod 显存请求分布"]
    B --> C{"最小显存请求?"}
    C -->|">= 16GB"| D["deviceSplitCount: 5"]
    C -->|"8-16GB"| E["deviceSplitCount: 10"]
    C -->|"2-8GB"| F["deviceSplitCount: 20"]
    C -->|"< 2GB"| G["deviceSplitCount: 30-50"]

    D --> H["验证 - 检查是否有 Pod 因
nvidia.com/gpu 不足而 Pending"]
    E --> H
    F --> H
    G --> H

    H -->|"有 Pending"| I["增大 deviceSplitCount"]
    H -->|"无 Pending"| J["当前设置合理"]
```

### 4.4 切分数设置建议

```yaml
# 推理集群 - 大量小模型推理服务
devicePlugin:
  deviceSplitCount: 20
  deviceMemoryScaling: 1.5

# 训练集群 - 少量大模型训练
devicePlugin:
  deviceSplitCount: 5
  deviceMemoryScaling: 1.0

# 混合集群 - 按节点差异化
devicePlugin:
  deviceSplitCount: 10
  nodeConfiguration:
    config: |
      {
        "nodeconfig": [
          {
            "name": "inference-pool-*",
            "devicesplitcount": 20
          },
          {
            "name": "training-pool-*",
            "devicesplitcount": 5
          }
        ]
      }
```

### 4.5 切分数与调度性能的关系

```mermaid
flowchart LR
    subgraph impact["切分数对调度性能的影响"]
        direction TB
        high_split["高切分数 (50)"]
        low_split["低切分数 (5)"]

        high_split --> hs1["节点 Annotation 数据量增大"]
        high_split --> hs2["Filter 阶段遍历设备数增多"]
        high_split --> hs3["调度延迟略有增加"]
        high_split --> hs4["GPU 利用率提升"]

        low_split --> ls1["节点 Annotation 数据量小"]
        low_split --> ls2["Filter 遍历快速"]
        low_split --> ls3["调度延迟低"]
        low_split --> ls4["GPU 利用率可能较低"]
    end
```

> **经验值**：在 200+ 节点的大规模集群中，`deviceSplitCount` 超过 50 可能会导致 Node Annotation 过大，影响 API Server 性能。建议控制在 30 以内。

---

## 5. 基于监控的调优方法

### 5.1 关键监控指标

HAMi 通过 vGPU Monitor 暴露 Prometheus 指标，以下是调优时需要重点关注的指标。

| 指标名称 | 类型 | 说明 | 调优参考 |
|:---------|:-----|:-----|:---------|
| `hami_vgpu_memory_used_bytes` | Gauge | 容器实际 GPU 显存使用量 | 与 limit 对比评估超配空间 |
| `hami_vgpu_memory_limit_bytes` | Gauge | 容器 GPU 显存限额 | 资源分配量 |
| `hami_vgpu_core_used_percent` | Gauge | 容器实际 SM 使用率 | 算力隔离效果 |
| `hami_vgpu_core_limit_percent` | Gauge | 容器 SM 限额 | 算力分配量 |

### 5.2 监控驱动的调优流水线

```mermaid
flowchart TB
    subgraph collect["数据采集"]
        prometheus["Prometheus
采集 HAMi 指标"]
        node_exporter["Node Exporter
GPU 节点系统指标"]
        dcgm["DCGM Exporter
(可选) 物理 GPU 指标"]
    end

    subgraph analyze["数据分析"]
        mem_ratio["显存使用率分析
used / limit"]
        core_ratio["算力使用率分析
used / limit"]
        pending_pods["Pending Pod 分析
资源不足事件"]
    end

    subgraph tune["调优决策"]
        direction TB
        t1{"显存使用率
持续低于 50%?"}
        t2{"算力使用率
持续低于限额?"}
        t3{"有 Pending Pod
等待 GPU?"}

        t1 -->|"是"| a1["增大 deviceMemoryScaling
或减小 Pod 显存申请"]
        t1 -->|"否"| a4["保持或减小超配比"]

        t2 -->|"是"| a2["降低 gpucores 限制
或切换 gpuCorePolicy=disable"]

        t3 -->|"是"| a3["增大 deviceSplitCount
或增加 GPU 节点"]
    end

    collect --> analyze
    analyze --> tune
```

### 5.3 常用 PromQL 查询

#### 显存使用率分析

```promql
# 所有容器的显存使用率 (用于评估超配空间)
hami_vgpu_memory_used_bytes / hami_vgpu_memory_limit_bytes * 100

# 显存使用率低于 30% 的容器 (超配候选)
(hami_vgpu_memory_used_bytes / hami_vgpu_memory_limit_bytes * 100) < 30

# 显存使用率超过 90% 的容器 (OOM 风险)
(hami_vgpu_memory_used_bytes / hami_vgpu_memory_limit_bytes * 100) > 90
```

#### 算力使用率分析

```promql
# 算力使用率分布
histogram_quantile(0.95, hami_vgpu_core_used_percent)

# 算力限制效果评估
hami_vgpu_core_used_percent / hami_vgpu_core_limit_percent * 100
```

#### GPU 节点资源余量

```promql
# 节点可分配 GPU 数量
kube_node_status_allocatable{resource="nvidia_com_gpu"}

# 节点已使用 GPU 数量
sum by (node) (kube_pod_container_resource_requests{resource="nvidia_com_gpu"})
```

### 5.4 Grafana 告警规则示例

```yaml
# 告警: GPU 显存使用率超过 90%
- alert: HAMiGPUMemoryHigh
  expr: |
    (hami_vgpu_memory_used_bytes / hami_vgpu_memory_limit_bytes) > 0.9
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "GPU 容器显存使用率超过 90%"
    description: "Pod {{ $labels.pod }} 的 GPU 显存使用率为 {{ $value | humanizePercentage }}"

# 告警: GPU Pod Pending 超过 10 分钟
- alert: HAMiGPUPodPending
  expr: |
    kube_pod_status_phase{phase="Pending"} == 1
    and on(pod)
    kube_pod_container_resource_requests{resource="nvidia_com_gpu"} > 0
  for: 10m
  labels:
    severity: warning
  annotations:
    summary: "GPU Pod 长时间 Pending"
```

### 5.5 调优反馈闭环

```mermaid
flowchart LR
    subgraph loop["持续调优闭环"]
        monitor["监控
采集指标数据"]
        analyze_data["分析
识别瓶颈"]
        plan["规划
制定调优方案"]
        implement["实施
修改 Helm values"]
        verify["验证
对比调优前后指标"]
    end

    monitor --> analyze_data
    analyze_data --> plan
    plan --> implement
    implement --> verify
    verify -->|"持续观察"| monitor
```

---

## 6. 调优场景速查表

### 6.1 常见问题与调优方案

| 问题现象 | 可能原因 | 调优参数 | 建议值 |
|:---------|:---------|:--------|:------|
| GPU Pod 调度延迟高 | API Server QPS 不足 | `--kube-qps`, `--kube-burst` | 50, 100 |
| 节点被长时间锁定 | 锁超时过长 / Scheduler 异常 | `scheduler.nodeLockExpire` | `3m` |
| GPU 利用率低 | 切分数太小 | `devicePlugin.deviceSplitCount` | 增大至 20 |
| 显存 OOM 频繁 | 超配比过高 | `devicePlugin.deviceMemoryScaling` | 减小至 1.0 |
| 调度器 OOM Kill | 内存限制太小 | `scheduler.extender.resources.limits.memory` | 增大至 2Gi |
| Monitor 数据缺失 | Monitor 日志级别过低 | `devicePlugin.monitor.extraArgs` | `-v=4` |
| CUDA 应用性能下降 | 日志级别过高 | `devices.nvidia.libCudaLogLevel` | `0` 或 `1` |
| 算力隔离不精确 | 默认策略过于宽松 | `devices.nvidia.gpuCorePolicy` | `force` |

### 6.2 环境分级配置模板

```yaml
# === 开发环境 ===
devicePlugin:
  deviceSplitCount: 20
  deviceMemoryScaling: 2.0
scheduler:
  defaultSchedulerPolicy:
    nodeSchedulerPolicy: binpack
    gpuSchedulerPolicy: binpack
devices:
  nvidia:
    gpuCorePolicy: disable
    libCudaLogLevel: 3

# === 生产环境 ===
devicePlugin:
  deviceSplitCount: 10
  deviceMemoryScaling: 1.0
scheduler:
  leaderElect: true
  replicas: 2
  defaultSchedulerPolicy:
    nodeSchedulerPolicy: spread
    gpuSchedulerPolicy: spread
  extender:
    extraArgs:
      - --kube-qps=50
      - --kube-burst=100
devices:
  nvidia:
    gpuCorePolicy: force
    libCudaLogLevel: 0
```

---

> **文档版本：** v1.0
>
> **适用 HAMi 版本：** v2.x
>
> **最后更新：** 2025-05
