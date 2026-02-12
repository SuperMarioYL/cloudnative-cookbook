---
title: "调度器配置参考"
weight: 8
---

## 概述

Volcano 调度器通过 YAML 配置文件定义 Action 执行顺序、Plugin Tier 层级、Plugin 参数以及 Metrics 配置。配置文件支持热加载：修改后无需重启调度器，下一个调度周期自动使用新配置。

本文提供完整的配置结构、所有可用参数和典型场景配置模板。

## 配置结构

> **源码参考**：`pkg/scheduler/conf/scheduler_conf.go`

```go
type SchedulerConfiguration struct {
    Actions              string            `yaml:"actions"`
    Tiers                []Tier            `yaml:"tiers"`
    Configurations       []Configuration   `yaml:"configurations"`
    MetricsConfiguration map[string]string `yaml:"metrics"`
}
```

### 完整配置示例

```yaml
actions: "enqueue, allocate, backfill, reclaim, preempt, shuffle"

tiers:
  - plugins:
      - name: gang
        enableJobOrder: true
        enableJobReady: true
        enableJobPipelined: true
      - name: priority
        enableJobOrder: true
        enableTaskOrder: true
  - plugins:
      - name: drf
        enableJobOrder: true
        enablePreemptable: true
      - name: proportion
        enableQueueOrder: true
        enableReclaimable: true
        enabledOverused: true
        enabledAllocatable: true
        enableJobEnqueued: true
  - plugins:
      - name: predicates
        enablePredicate: true
      - name: nodeorder
        enableNodeOrder: true
        arguments:
          nodeaffinity.weight: 10
          podaffinity.weight: 10
          tainttoleration.weight: 10
      - name: binpack
        enableNodeOrder: true
        arguments:
          binpack.weight: 5
          binpack.cpu: 1
          binpack.memory: 1

configurations:
  - name: allocate
    arguments:
      enablePredicateErrorCache: true

metrics:
  type: "metrics-server"
  interval: "30s"
```

---

## Actions 配置

### 格式

```yaml
actions: "action1, action2, action3"
```

逗号分隔的 Action 名称列表，定义了每个调度周期的执行顺序。

### 可用 Action

| Action | 说明 |
|--------|------|
| `enqueue` | 控制 Job 从 Pending → Inqueue 的入队 |
| `allocate` | 核心分配，为 Task 分配节点 |
| `backfill` | 填充 BestEffort 类型的 Task |
| `reclaim` | 跨队列资源回收 |
| `preempt` | 队列内优先级抢占 |
| `shuffle` | 选择性驱逐以优化布局 |

### 推荐配置

| 场景 | 配置 |
|------|------|
| 最小化 | `"enqueue, allocate"` |
| 基础调度 | `"enqueue, allocate, backfill"` |
| 多队列 | `"enqueue, allocate, backfill, reclaim"` |
| 完整功能 | `"enqueue, allocate, backfill, reclaim, preempt, shuffle"` |

---

## Tiers 配置

### 结构

```yaml
tiers:
  - plugins:           # Tier 1（最高优先级）
      - name: plugin_a
        arguments: {}
  - plugins:           # Tier 2
      - name: plugin_b
```

Tier 按数组顺序排列，第一个 Tier 优先级最高。同一 Tier 内的 Plugin 按数组顺序执行。

### PluginOption 完整字段

> **源码参考**：`pkg/scheduler/conf/scheduler_conf.go::PluginOption`

| YAML 字段 | 类型 | 默认值 | 说明 |
|-----------|------|--------|------|
| `name` | string | 必填 | Plugin 名称 |
| `enableJobOrder` | *bool | nil (启用) | 启用 JobOrderFn |
| `enableJobReady` | *bool | nil (启用) | 启用 JobReadyFn |
| `enableJobPipelined` | *bool | nil (启用) | 启用 JobPipelinedFn |
| `enableTaskOrder` | *bool | nil (启用) | 启用 TaskOrderFn |
| `enablePreemptable` | *bool | nil (启用) | 启用 PreemptableFn |
| `enableReclaimable` | *bool | nil (启用) | 启用 ReclaimableFn |
| `enablePreemptive` | *bool | nil (启用) | 启用 PreemptiveFn |
| `enableQueueOrder` | *bool | nil (启用) | 启用 QueueOrderFn |
| `EnabledClusterOrder` | *bool | nil (启用) | 启用 ClusterOrderFn |
| `enablePredicate` | *bool | nil (启用) | 启用 PredicateFn |
| `enableBestNode` | *bool | nil (启用) | 启用 BestNodeFn |
| `enableNodeOrder` | *bool | nil (启用) | 启用 NodeOrderFn |
| `enableTargetJob` | *bool | nil (启用) | 启用 TargetJobFn |
| `enableReservedNodes` | *bool | nil (启用) | 启用 ReservedNodesFn |
| `enableJobEnqueued` | *bool | nil (启用) | 启用 JobEnqueuedFn |
| `enabledVictim` | *bool | nil (启用) | 启用 VictimTasksFn |
| `enableJobStarving` | *bool | nil (启用) | 启用 JobStarvingFn |
| `enabledOverused` | *bool | nil (启用) | 启用 OverusedFn |
| `enabledAllocatable` | *bool | nil (启用) | 启用 AllocatableFn |
| `enabledHyperNodeOrder` | *bool | nil (启用) | 启用 HyperNodeOrderFn |
| `enabledSubJobReady` | *bool | nil (启用) | 启用 SubJobReadyFn |
| `enabledSubJobPipelined` | *bool | nil (启用) | 启用 SubJobPipelinedFn |
| `enabledSubJobOrder` | *bool | nil (启用) | 启用 SubJobOrderFn |
| `enabledHyperNodeGradient` | *bool | nil (启用) | 启用 HyperNodeGradientFn |
| `enableHierarchy` | *bool | nil (启用) | 启用层级队列共享 |
| `arguments` | map | {} | Plugin 特定参数 |

> **注意**：`nil`（未设置）等同于 `true`（启用）。只有显式设置为 `false` 才会禁用。

---

## Plugin 参数详解

### predicates

Kubernetes 原生 Predicate 兼容 Plugin。

```yaml
- name: predicates
  arguments:
    # 启用/禁用特定的 K8s Filter Plugin
    predicate.VolumeBindingEnabled: true
    predicate.GPUSharingEnable: true
    predicate.CachePredicate: true
```

### nodeorder

节点打分 Plugin，包装 K8s 原生 Score Plugin。

```yaml
- name: nodeorder
  arguments:
    # 各打分维度权重（0-100）
    nodeaffinity.weight: 10          # 节点亲和性
    podaffinity.weight: 10           # Pod 亲和性
    tainttoleration.weight: 10       # 污点容忍度
    leastrequested.weight: 1         # 最少请求优先
    mostrequested.weight: 0          # 最多请求优先
    balancedresource.weight: 1       # 资源均衡
    imagelocality.weight: 1          # 镜像本地性
    selectorspread.weight: 0         # 选择器分散
```

### binpack

装箱优化 Plugin，将 Task 聚集到已有负载的节点。

```yaml
- name: binpack
  arguments:
    binpack.weight: 10               # 总权重
    binpack.cpu: 1                   # CPU 维度权重
    binpack.memory: 1                # Memory 维度权重
    binpack.resources: "nvidia.com/gpu"  # 额外参与打分的资源
    binpack.resources.nvidia.com/gpu: 2  # GPU 维度权重
```

### proportion

队列公平共享 Plugin。

```yaml
- name: proportion
  # proportion 不接受额外参数
  # 资源分配由 Queue 的 weight/capability/guarantee 配置决定
```

### capacity

队列容量管理 Plugin（proportion 的增强版）。

```yaml
- name: capacity
  enableHierarchy: true              # 启用层级队列
```

### overcommit

资源超分配 Plugin，允许队列使用超出集群实际资源的配额。

```yaml
- name: overcommit
  arguments:
    overcommit-factor: 1.5           # 超分配因子（1.5 表示 150%）
```

### sla

SLA 约束 Plugin，Job 在指定时间内必须开始调度。

```yaml
- name: sla
  arguments:
    sla-waiting-time: "1h"           # 最大等待时间
```

### tdm

时分复用 Plugin，在指定时间段内允许特定类型的调度。

```yaml
- name: tdm
  arguments:
    tdm.revocable-zone.rz1: "0:00-12:00"  # 可回收区域时间段
```

### deviceshare

GPU 和设备共享 Plugin。

```yaml
- name: deviceshare
  arguments:
    deviceshare.GPUSharingEnable: true
    deviceshare.GPUNumberEnable: true
```

### network-topology-aware

网络拓扑感知 Plugin。

```yaml
- name: network-topology-aware
  enabledHyperNodeOrder: true
  enabledHyperNodeGradient: true
  arguments:
    network-topology-aware.weight: 10
```

---

## Configurations（Action 参数）

### 结构

```yaml
configurations:
  - name: action_name
    arguments:
      key: value
```

### allocate Action 参数

```yaml
configurations:
  - name: allocate
    arguments:
      enablePredicateErrorCache: true  # 启用 Predicate 错误缓存
```

### preempt Action 参数

```yaml
configurations:
  - name: preempt
    arguments:
      enableTopologyAwarePreemption: true       # 启用拓扑感知抢占
      topologyAwarePreemptWorkerNum: 16         # Worker 数量
      minCandidateNodesPercentage: 10           # 最小候选节点百分比
      minCandidateNodesAbsolute: 10             # 最小候选节点绝对数
      maxCandidateNodesAbsolute: 100            # 最大候选节点绝对数
```

---

## Metrics 配置

```yaml
metrics:
  type: "metrics-server"    # Metrics 后端类型
  interval: "30s"           # 采集间隔
```

| 参数 | 说明 |
|------|------|
| `type` | Metrics 数据源类型 |
| `interval` | 数据采集间隔 |

---

## 命令行参数

调度器启动时通过命令行参数进行全局配置：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `--schedule-period` | `1s` | 调度周期间隔 |
| `--scheduler-conf` | - | 配置文件路径 |
| `--scheduler-name` | `volcano` | 调度器名称 |
| `--default-queue` | `default` | 默认队列名 |
| `--node-selector` | - | 节点过滤标签（key=value） |
| `--node-worker-threads` | `20` | Node Worker 并发数 |
| `--listen-address` | `:8080` | Metrics/健康检查地址 |
| `--disable-default-scheduler-config` | `false` | 禁用默认配置 |
| `--enable-csi-storage` | `false` | 启用 CSI 存储感知 |
| `--ignored-csi-provisioners` | - | 忽略的 CSI Provisioner |

---

## 场景化配置模板

### 模板一：AI/ML 训练集群

```yaml
actions: "enqueue, allocate, backfill, reclaim, preempt"

tiers:
  - plugins:
      - name: gang
        enableJobOrder: true
        enableJobReady: true
        enableJobPipelined: true
        enablePreemptable: true
      - name: priority
        enableJobOrder: true
        enableTaskOrder: true
  - plugins:
      - name: drf
        enableJobOrder: true
        enablePreemptable: true
      - name: proportion
        enableQueueOrder: true
        enableReclaimable: true
        enabledOverused: true
        enabledAllocatable: true
        enableJobEnqueued: true
  - plugins:
      - name: predicates
        enablePredicate: true
      - name: nodeorder
        enableNodeOrder: true
        arguments:
          leastrequested.weight: 0
          mostrequested.weight: 0
          nodeaffinity.weight: 10
          podaffinity.weight: 10
          tainttoleration.weight: 10
      - name: binpack
        enableNodeOrder: true
        arguments:
          binpack.weight: 10
          binpack.cpu: 1
          binpack.memory: 1
          binpack.resources: "nvidia.com/gpu"
          binpack.resources.nvidia.com/gpu: 5
      - name: deviceshare
        arguments:
          deviceshare.GPUSharingEnable: true
```

**特点**：Gang 调度保证训练任务整体性，Binpack 优先 GPU 维度，DeviceShare 支持 GPU 共享。

### 模板二：多租户公平共享

```yaml
actions: "enqueue, allocate, backfill, reclaim"

tiers:
  - plugins:
      - name: gang
      - name: priority
  - plugins:
      - name: proportion
        enableQueueOrder: true
        enableReclaimable: true
        enabledOverused: true
        enabledAllocatable: true
        enableJobEnqueued: true
        enablePreemptive: true
  - plugins:
      - name: predicates
      - name: nodeorder
        arguments:
          balancedresource.weight: 5
          leastrequested.weight: 3
          nodeaffinity.weight: 10
          tainttoleration.weight: 10
```

**特点**：Proportion 实现队列间公平共享，Reclaim 允许跨队列回收，NodeOrder 偏向资源均衡。

### 模板三：拓扑感知调度

```yaml
actions: "enqueue, allocate, backfill, preempt"

tiers:
  - plugins:
      - name: gang
        enabledSubJobReady: true
        enabledSubJobPipelined: true
        enabledSubJobOrder: true
      - name: priority
  - plugins:
      - name: proportion
      - name: network-topology-aware
        enabledHyperNodeOrder: true
        enabledHyperNodeGradient: true
        arguments:
          network-topology-aware.weight: 10
  - plugins:
      - name: predicates
      - name: nodeorder
      - name: binpack

configurations:
  - name: preempt
    arguments:
      enableTopologyAwarePreemption: true
      topologyAwarePreemptWorkerNum: 16
```

**特点**：HyperNode 拓扑感知，SubJob 支持，拓扑感知抢占。

### 模板四：简单批处理

```yaml
actions: "enqueue, allocate, backfill"

tiers:
  - plugins:
      - name: gang
      - name: priority
  - plugins:
      - name: proportion
  - plugins:
      - name: predicates
      - name: nodeorder
        arguments:
          leastrequested.weight: 5
          balancedresource.weight: 3
```

**特点**：最简配置，适合单队列简单批处理场景。

---

## 配置热加载

### 机制

```mermaid
flowchart LR
    edit["修改配置文件\n或 kubectl edit configmap"] --> fs["fsnotify\n检测文件变化"]
    fs --> load["loadSchedulerConf()\n重新解析 YAML"]
    load --> validate{"配置有效?"}
    validate -->|"是"| update["mutex.Lock()\n更新 actions/plugins\nmutex.Unlock()"]
    validate -->|"否"| keep["保持当前配置\n记录错误日志"]
    update --> next["下一个 runOnce()\n使用新配置"]

    style update fill:#c8e6c9
    style keep fill:#ffcdd2
```

### 使用 ConfigMap 管理配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: volcano-scheduler-configmap
  namespace: volcano-system
data:
  volcano-scheduler.conf: |
    actions: "enqueue, allocate, backfill, reclaim, preempt"
    tiers:
      - plugins:
          - name: gang
          - name: priority
      # ...
```

调度器 Deployment 挂载此 ConfigMap 并通过 `--scheduler-conf` 指定路径。修改 ConfigMap 后，文件变化会被自动检测并重新加载。

---

## 常见问题

### Q: 配置文件无效会导致调度器崩溃吗？

不会。无效配置会被忽略，调度器保持上一次有效配置继续运行，并在日志中记录错误。

### Q: 默认配置是什么？

如果没有提供配置文件且未禁用默认配置（`--disable-default-scheduler-config=false`），调度器使用代码中硬编码的默认配置：

```yaml
actions: "enqueue, allocate, backfill"
tiers:
  - plugins:
      - name: gang
      - name: priority
  - plugins:
      - name: drf
      - name: proportion
  - plugins:
      - name: predicates
      - name: nodeorder
      - name: conformance
```

### Q: 如何验证配置是否生效？

1. 检查调度器日志中是否有配置加载成功的日志
2. 查看调度器 Metrics 中 Action 执行的名称和耗时
3. 使用 `--v=4` 或更高日志级别查看详细的 Plugin 初始化信息

### Q: 如何选择 proportion 还是 capacity？

- **proportion**：适合扁平的多队列场景，基于权重公平共享
- **capacity**：适合层级队列场景，支持父子队列的资源继承和限制

两者不应同时启用。

---

## 下一步

- [调度器生命周期](./01-scheduler-lifecycle.md) -- 配置如何在调度循环中被使用
- [Plugin 系统](./05-plugin-system.md) -- 深入了解各 Plugin 的算法实现
- [Action 流水线](./04-action-pipeline.md) -- 理解 Action 参数的作用
