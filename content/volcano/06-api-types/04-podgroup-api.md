---
title: "PodGroup API 完整参考"
weight: 4
---

> 基于 `scheduling.volcano.sh/v1beta1` 版本的 PodGroup CRD 完整 API 参考文档。
>
> 源码路径: `staging/src/volcano.sh/apis/pkg/apis/scheduling/v1beta1/types.go`

---

## 1. 概述

PodGroup 是 Volcano 调度系统中的核心资源对象，用于将一组相关联的 Pod 组织在一起，实现 **Gang Scheduling**（协同调度）语义。PodGroup 确保要么所有必需的 Pod 同时获得资源开始运行，要么全部不启动，避免资源死锁和浪费。

### 1.1 CRD 基本信息

| 属性 | 值 |
|------|------|
| **API Group** | `scheduling.volcano.sh` |
| **API Version** | `v1beta1` |
| **Kind** | `PodGroup` |
| **Scope** | Namespaced |
| **Short Names** | `pg`, `podgroup-v1beta1` |
| **Subresource** | `status` |
| **PrintColumns** | STATUS, minMember, RUNNINGS, AGE, QUEUE(priority=1) |

### 1.2 核心功能定位

```mermaid
graph TD
    subgraph core["PodGroup 核心功能"]
        A["Gang Scheduling<br/>协同调度"]
        B["资源预检<br/>MinResources"]
        C["队列关联<br/>Queue Binding"]
        D["优先级管理<br/>PriorityClass"]
        E["网络拓扑感知<br/>NetworkTopology"]
        F["子组策略<br/>SubGroupPolicy"]
    end

    A --> A1["保证最小成员数<br/>同时获得资源"]
    B --> B1["调度前检查集群<br/>资源是否充足"]
    C --> C1["通过 Queue 实现<br/>多租户资源隔离"]
    D --> D1["基于 PriorityClass<br/>决定调度顺序"]
    E --> E1["配合 HyperNode<br/>实现拓扑亲和"]
    F --> F1["Pod 分组级别的<br/>Gang 调度"]
```

---

## 2. PodGroup 资源结构

```mermaid
classDiagram
    class PodGroup {
        +TypeMeta
        +ObjectMeta metadata
        +PodGroupSpec spec
        +PodGroupStatus status
    }

    class PodGroupSpec {
        +int32 minMember
        +map~string,int32~ minTaskMember
        +string queue
        +string priorityClassName
        +*ResourceList minResources
        +*NetworkTopologySpec networkTopology
        +[]SubGroupPolicySpec subGroupPolicy
    }

    class PodGroupStatus {
        +PodGroupPhase phase
        +[]PodGroupCondition conditions
        +int32 running
        +int32 succeeded
        +int32 failed
    }

    class PodGroupCondition {
        +PodGroupConditionType type
        +ConditionStatus status
        +string transitionID
        +Time lastTransitionTime
        +string reason
        +string message
    }

    class SubGroupPolicySpec {
        +string name
        +*NetworkTopologySpec networkTopology
        +*int32 subGroupSize
        +*int32 minSubGroups
        +*LabelSelector labelSelector
        +[]string matchLabelKeys
    }

    class NetworkTopologySpec {
        +NetworkTopologyMode mode
        +*int highestTierAllowed
        +string highestTierName
    }

    PodGroup --> PodGroupSpec
    PodGroup --> PodGroupStatus
    PodGroupStatus --> PodGroupCondition
    PodGroupSpec --> SubGroupPolicySpec
    PodGroupSpec --> NetworkTopologySpec
    SubGroupPolicySpec --> NetworkTopologySpec
```

---

## 3. PodGroupSpec 完整字段

### 3.1 MinMember

| 属性 | 值 |
|------|------|
| **类型** | `int32` |
| **必填** | 否 |
| **验证规则** | `minimum=0` |

定义 PodGroup 中的最小成员数。调度器在 Gang Scheduling 时，只有当集群资源足以同时启动至少 `minMember` 个 Pod 时，才会开始调度。典型场景：一个分布式训练任务需要 4 Worker + 1 PS，则 `minMember` 设为 5。

### 3.2 MinTaskMember

| 属性 | 值 |
|------|------|
| **类型** | `map[string]int32` |
| **必填** | 否 |
| **说明** | 建议使用 SubGroupPolicy 替代 |

定义每个 Task 类型的最小 Pod 数量。key 为 Task 名称，value 为最小 Pod 数。`SubGroupPolicy` 覆盖了其所有能力，官方推荐使用 `SubGroupPolicy`，不建议并用。

### 3.3 Queue

| 属性 | 值 |
|------|------|
| **类型** | `string` |
| **默认值** | `"default"` |
| **验证规则** | `maxLength=253`，Pattern: `^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$` |

指定 PodGroup 所属的 Queue，用于多租户资源隔离和公平共享。如果 Queue 不存在，PodGroup 不会被调度。

### 3.4 PriorityClassName

| 属性 | 值 |
|------|------|
| **类型** | `string` |
| **验证规则** | `maxLength=253` |

指定优先级，支持 `system-node-critical` 和 `system-cluster-critical` 内置关键字，或自定义 PriorityClass 名称。影响调度顺序和抢占决策。

### 3.5 MinResources

| 属性 | 值 |
|------|------|
| **类型** | `*v1.ResourceList` |
| **必填** | 否 |

PodGroup 运行所需的最小资源总量。调度器在 `enqueue` 阶段检查集群可用资源是否满足，只有满足时才从 Pending 转入 Inqueue，起到资源预检作用。

### 3.6 NetworkTopology

| 属性 | 值 |
|------|------|
| **类型** | `*NetworkTopologySpec` |
| **必填** | 否 |

PodGroup 级别的网络拓扑约束，需配合 HyperNode CRD 使用。

```mermaid
graph TD
    subgraph nt["NetworkTopologySpec"]
        M["mode"]
        H["highestTierAllowed"]
        N["highestTierName"]
    end

    M -->|hard| HARD["硬约束 - Pod 必须<br/>位于同一拓扑域内"]
    M -->|soft| SOFT["软约束 - 优先同拓扑域<br/>允许跨域调度"]
    H --> TIER["指定允许跨越的最高<br/>拓扑层级（数字）"]
    N --> NAME["指定允许跨越的最高<br/>拓扑层级（名称）"]
    TIER -.->|互斥| NAME
```

| 字段 | 类型 | 默认值 | 验证规则 |
|------|------|--------|----------|
| `mode` | `NetworkTopologyMode` | `hard` | Enum: `hard`, `soft` |
| `highestTierAllowed` | `*int` | 无 | `minimum=0` |
| `highestTierName` | `string` | 无 | `maxLength=253` |

> `highestTierAllowed` 和 `highestTierName` 不能同时设置。

### 3.7 SubGroupPolicy

| 属性 | 值 |
|------|------|
| **类型** | `[]SubGroupPolicySpec` |
| **必填** | 否 |

为 PodGroup 内的 Pod 提供二级分组能力，支持子组级别的 Gang Scheduling 和网络拓扑亲和。详见 [第 4 节](#4-subgrouppolicyspec-详解)。

---

## 4. SubGroupPolicySpec 详解

### 4.1 字段定义

| 字段 | 类型 | 默认值 | 验证规则 | 说明 |
|------|------|--------|----------|------|
| `name` | `string` | 无 | `maxLength=253` | 子组策略名称 |
| `networkTopology` | `*NetworkTopologySpec` | 无 | - | 子组级别网络拓扑约束 |
| `subGroupSize` | `*int32` | `1` | `minimum=1` | 每个子亲和组的 Pod 数量 |
| `minSubGroups` | `*int32` | `0` | `minimum=0` | 触发调度所需的最小子组数 |
| `labelSelector` | `*metav1.LabelSelector` | 无 | - | Pod 匹配标签选择器 |
| `matchLabelKeys` | `[]string` | 无 | `listType=atomic` | 标签键分组过滤规则 |

### 4.2 工作原理

```mermaid
flowchart TB
    subgraph step1["Step 1 - Pod 匹配"]
        LS["LabelSelector<br/>筛选匹配的 Pod"]
    end
    subgraph step2["Step 2 - 精细分组"]
        MLK["MatchLabelKeys<br/>按标签值精细分组"]
    end
    subgraph step3["Step 3 - 子组划分"]
        SGS["SubGroupSize<br/>每个子组的 Pod 数量"]
    end
    subgraph step4["Step 4 - Gang 检查"]
        MSG["MinSubGroups<br/>满足最小子组数才调度"]
    end
    subgraph step5["Step 5 - 拓扑约束"]
        NT["NetworkTopology<br/>子组内 Pod 的拓扑约束"]
    end
    step1 --> step2 --> step3 --> step4 --> step5
    step5 --> RESULT["调度决策 - 满足所有约束的子组可以被调度"]
```

### 4.3 与 MinTaskMember 对比

| 能力 | MinTaskMember | SubGroupPolicy |
|------|:---:|:---:|
| 按 Task 设定最小 Pod 数 | 支持 | 支持 |
| 子组级别 Gang Scheduling | 不支持 | 支持 |
| 子组级别网络拓扑约束 | 不支持 | 支持 |
| 灵活的 Pod 分组规则 | 不支持 | 支持 |

---

## 5. PodGroupStatus

### 5.1 字段定义

| 字段 | 类型 | 验证规则 | 说明 |
|------|------|----------|------|
| `phase` | `PodGroupPhase` | Enum: Pending, Running, Unknown, Inqueue, Completed | 当前阶段 |
| `conditions` | `[]PodGroupCondition` | - | 状态条件列表 |
| `running` | `int32` | `minimum=0` | 正在运行的 Pod 数量 |
| `succeeded` | `int32` | `minimum=0` | 已成功完成的 Pod 数量 |
| `failed` | `int32` | `minimum=0` | 已失败的 Pod 数量 |

### 5.2 PodGroupCondition

| 字段 | 类型 | 说明 |
|------|------|------|
| `type` | `PodGroupConditionType` | `Unschedulable` 或 `Scheduled` |
| `status` | `v1.ConditionStatus` | True / False / Unknown |
| `transitionID` | `string` | 状态转换标识 |
| `lastTransitionTime` | `metav1.Time` | 最近转换时间 |
| `reason` | `string` | 变更原因（CamelCase） |
| `message` | `string` | 人类可读详细信息 |

### 5.3 常见 Reason 值

| Reason | 常量 | 触发场景 |
|--------|------|----------|
| `PodFailed` | `PodFailedReason` | PodGroup 中有 Pod 失败 |
| `PodDeleted` | `PodDeletedReason` | PodGroup 中有 Pod 被删除 |
| `NotEnoughResources` | `NotEnoughResourcesReason` | 集群资源不足 |
| `NotEnoughTasks` | `NotEnoughPodsReason` | Pod 总数不满足 `minMember` |
| `NotEnoughPodsOfTask` | `NotEnoughPodsOfTaskReason` | Task 的 Pod 数不满足 `minTaskMember` |

---

## 6. PodGroupPhase 枚举

### 6.1 阶段定义

| Phase | 值 | 说明 |
|-------|------|------|
| **Pending** | `"Pending"` | 系统已接受，但资源不足 |
| **Inqueue** | `"Inqueue"` | 资源预检通过，控制器可创建 Pod（Pending 与 Running 间的过渡态） |
| **Running** | `"Running"` | 至少 `minMember` 个 Pod 进入运行状态 |
| **Unknown** | `"Unknown"` | 部分 Pod 运行但其余无法调度，等待恢复 |
| **Completed** | `"Completed"` | 所有 Pod 都已完成 |

### 6.2 状态转换图

```mermaid
stateDiagram-v2
    [*] --> Pending : PodGroup 创建

    Pending --> Inqueue : 资源预检通过<br/>enqueue action
    Inqueue --> Pending : 资源不再满足<br/>或被踢出队列
    Inqueue --> Running : minMember 个 Pod<br/>进入 Running 状态
    Running --> Unknown : 部分 Pod 失败<br/>运行数不足 minMember
    Unknown --> Running : Pod 恢复<br/>重新满足 minMember
    Running --> Completed : 所有 Pod 完成
    Unknown --> Pending : Pod 大量失败<br/>需要重新调度
    Pending --> Completed : 作业取消或<br/>所有 Pod 直接完成
```

---

## 7. PodGroup 自动创建

用户提交 Volcano Job 时，Job Controller 自动创建对应的 PodGroup。

### 7.1 创建流程

```mermaid
sequenceDiagram
    participant User as 用户
    participant API as API Server
    participant JC as Job Controller
    participant PG as PodGroup

    User->>API: 提交 Volcano Job
    API->>JC: Job 创建事件
    JC->>JC: createOrUpdatePodGroup()

    alt PodGroup 不存在
        JC->>JC: generateRelatedPodGroupName()
        Note over JC: Name = "{job.Name}-{job.UID}"
        JC->>API: 创建 PodGroup
        API->>PG: PodGroup Created
    else PodGroup 已存在
        JC->>JC: shouldUpdateExistingPodGroup()
        JC->>API: 更新 PodGroup（如需要）
    end
```

### 7.2 命名规则

PodGroup 命名格式：`{job.Name}-{job.UID}`。例如 Job 名为 `training-job`，UID 为 `a1b2c3d4-...`，则 PodGroup 名为 `training-job-a1b2c3d4-...`。

> 兼容性：1.5 版本前 PodGroup 名称直接使用 Job 名称，Controller 查找时先尝试新格式，再回退旧格式。

### 7.3 字段映射

| PodGroup 字段 | 来源 |
|--------------|------|
| `metadata.namespace` | `job.Namespace` |
| `metadata.name` | `{job.Name}-{job.UID}` |
| `metadata.annotations` / `labels` | 继承 Job 注解和标签 |
| `metadata.ownerReferences` | Job 作为 Controller Owner |
| `spec.minMember` | `job.Spec.MinAvailable` |
| `spec.minTaskMember` | 遍历 `job.Spec.Tasks` 计算 |
| `spec.queue` | `job.Spec.Queue` |
| `spec.minResources` | 从 Job Tasks 计算资源总量 |
| `spec.priorityClassName` | `job.Spec.PriorityClassName` |
| `spec.networkTopology` | `job.Spec.NetworkTopology` |
| `spec.subGroupPolicy` | 从 `job.Spec.Tasks` 构建 |

---

## 8. 与 Job 的关系

### 8.1 关联机制

```mermaid
graph TB
    subgraph job["Volcano Job"]
        JM["metadata"]
        JS["spec.minAvailable"]
        JQ["spec.queue"]
        JT["spec.tasks"]
    end

    subgraph pg["PodGroup"]
        PM["metadata.ownerReferences"]
        PS["spec.minMember"]
        PQ["spec.queue"]
        PT["spec.minTaskMember"]
    end

    subgraph pod["Pod"]
        POD_A["annotation:<br/>scheduling.k8s.io/group-name"]
        POD_B["annotation:<br/>scheduling.volcano.sh/group-name"]
        POD_L["label:<br/>volcano.sh/job-name"]
    end

    JM -->|"OwnerReference"| PM
    JS -->|"映射"| PS
    JQ -->|"映射"| PQ
    JT -->|"计算"| PT
    pg -.->|"被 Pod 引用"| POD_A
    pg -.->|"被 Pod 引用"| POD_B
    job -.->|"标记 Pod"| POD_L
```

### 8.2 OwnerReferences

Job Controller 创建 PodGroup 时设置 OwnerReferences 指向 Job（`controller: true`），确保 Job 删除时 PodGroup 被自动清理。

### 8.3 关键注解和标签

**Pod 关联 PodGroup 的注解**：

| 注解 Key | 常量 |
|----------|------|
| `scheduling.k8s.io/group-name` | `KubeGroupNameAnnotationKey` |
| `scheduling.volcano.sh/group-name` | `VolcanoGroupNameAnnotationKey` |

**Pod 关联 Job 的标签**：

| 标签 Key | 常量 |
|----------|------|
| `volcano.sh/job-name` | `JobNameKey` |
| `volcano.sh/queue-name` | `QueueNameKey` |
| `volcano.sh/task-spec` | `TaskSpecKey` |

### 8.4 生命周期联动

```mermaid
sequenceDiagram
    participant J as Volcano Job
    participant PG as PodGroup
    participant S as Scheduler
    participant P as Pods

    Note over J,P: 创建阶段
    J->>PG: Job Controller 创建 PodGroup
    PG-->>S: Scheduler 感知新 PodGroup

    Note over J,P: 调度阶段
    S->>PG: enqueue - Pending to Inqueue
    J->>P: Job Controller 创建 Pods
    S->>P: allocate - 绑定 Pod 到节点
    PG-->>PG: Inqueue to Running

    Note over J,P: 运行与清理
    P-->>PG: Pod 完成/失败 - 更新计数
    J->>PG: Job 删除时级联清理 PodGroup
```

---

## 9. 独立创建场景与非 Volcano 工作负载

对于 Deployment、StatefulSet 等非 Volcano 原生工作负载，用户可手动创建 PodGroup 并通过注解关联 Pod：

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: my-deployment-pg
spec:
  minMember: 3
  queue: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    metadata:
      annotations:
        scheduling.volcano.sh/group-name: my-deployment-pg
    spec:
      schedulerName: volcano
      containers:
        - name: app
          image: my-app:latest
```

Volcano Webhook 也支持通过注解自动创建 PodGroup：

| 注解 Key | 说明 |
|----------|------|
| `scheduling.volcano.sh/group-min-member` | PodGroup 最小成员数 |
| `scheduling.volcano.sh/group-min-resources` | PodGroup 最小资源量 |
| `scheduling.volcano.sh/queue-name` | 所属 Queue 名称 |

---

## 10. 完整 YAML 示例

### 10.1 通过 Volcano Job 自动创建

```yaml
apiVersion: batch.volcano.sh/v1alpha1
kind: Job
metadata:
  name: distributed-training
spec:
  minAvailable: 5
  schedulerName: volcano
  queue: gpu-queue
  tasks:
    - name: ps
      replicas: 1
      template:
        spec:
          containers:
            - name: ps
              image: tensorflow/tensorflow:latest
              resources:
                requests:
                  cpu: "2"
                  memory: "4Gi"
    - name: worker
      replicas: 4
      template:
        spec:
          containers:
            - name: worker
              image: tensorflow/tensorflow:latest-gpu
              resources:
                requests:
                  cpu: "4"
                  memory: "8Gi"
                  nvidia.com/gpu: "1"
```

Job Controller 自动创建的 PodGroup：

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: distributed-training-<job-uid>
  ownerReferences:
    - apiVersion: batch.volcano.sh/v1alpha1
      kind: Job
      name: distributed-training
      controller: true
spec:
  minMember: 5
  minTaskMember:
    ps: 1
    worker: 4
  queue: gpu-queue
  minResources:
    cpu: "18"
    memory: "36Gi"
    nvidia.com/gpu: "4"
```

### 10.2 带 SubGroupPolicy 的 PodGroup

```yaml
apiVersion: scheduling.volcano.sh/v1beta1
kind: PodGroup
metadata:
  name: training-pg
  namespace: ml-team
spec:
  minMember: 32
  queue: gpu-queue
  minResources:
    cpu: "64"
    memory: "256Gi"
    nvidia.com/gpu: "32"
  subGroupPolicy:
    - name: "trainer-group"
      subGroupSize: 8
      minSubGroups: 3
      networkTopology:
        mode: hard
        highestTierAllowed: 1
      labelSelector:
        matchLabels:
          task: trainer
      matchLabelKeys:
        - "ring-id"
    - name: "evaluator-group"
      subGroupSize: 4
      minSubGroups: 1
      labelSelector:
        matchLabels:
          task: evaluator
```

---

## 11. 调度器集成

```mermaid
flowchart TB
    subgraph enqueue_phase["Enqueue Action"]
        E1["检查 MinResources"]
        E2["Pending -> Inqueue"]
    end
    subgraph allocate_phase["Allocate Action"]
        A1["按优先级排序 PodGroup"]
        A2["Gang 插件检查 MinMember"]
        A3["分配资源给 Pod"]
    end
    subgraph preempt_phase["Preempt Action"]
        P1["高优先级 PodGroup 抢占"]
    end
    subgraph reclaim_phase["Reclaim Action"]
        R1["Queue 间资源回收"]
    end
    enqueue_phase --> allocate_phase --> preempt_phase --> reclaim_phase
```

| 插件 | 与 PodGroup 的交互 |
|------|-------------------|
| `gang` | 检查 `minMember` / `minTaskMember` / `subGroupPolicy`，实现 Gang Scheduling |
| `proportion` | 基于 PodGroup 所属 Queue 的权重分配资源份额 |
| `priority` | 基于 `priorityClassName` 对 PodGroup 排序 |
| `sla` | 检查 `sla-waiting-time` 注解，超时降级 |
| `predicates` | 对 PodGroup 中每个 Pod 进行节点过滤 |

---

## 12. 常用操作命令

```bash
# 查看所有 PodGroup
kubectl get podgroup -A

# 查看特定 PodGroup 详情
kubectl describe podgroup <name> -n <namespace>

# 使用简写
kubectl get pg -A

# 查看带 QUEUE 列的详细输出
kubectl get pg -A -o wide

# 以 YAML 格式查看完整定义
kubectl get pg <name> -n <namespace> -o yaml
```
