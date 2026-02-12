---
title: "Volcano 架构概览"
weight: 2
---

## 1. 整体架构

Volcano 是一个构建在 Kubernetes 之上的高性能批量计算调度引擎，专为 AI/ML、大数据、HPC 等高吞吐计算场景设计。在 Kubernetes 集群中，Volcano 以一组协作组件的形式部署，通过 CRD（Custom Resource Definition）扩展 Kubernetes 原生 API，提供 Job 生命周期管理、Gang Scheduling、队列管理、公平调度等核心能力。

Volcano 的整体架构由 **5 个核心组件** 构成，所有组件均通过 Kubernetes API Server 进行通信和协调：

| 组件 | 部署形态 | 核心职责 |
|------|---------|---------|
| **vc-scheduler** | Deployment | 批量调度器，执行 Action/Plugin 调度流水线 |
| **vc-controller-manager** | Deployment | 控制器管理器，管理 Job、Queue、PodGroup 等资源的生命周期 |
| **vc-webhook-manager** | Deployment | Admission Webhook，负责资源的校验（Validate）与变更（Mutate） |
| **vc-agent** | DaemonSet | 节点代理，负责混部场景下的资源监控与 cgroup 管理 |
| **vcctl** | CLI 工具 | 命令行客户端，提供 Job 管理操作接口 |

以下架构图展示了这 5 个组件在 Kubernetes 集群中的部署关系及其与 API Server 的交互模式：

```mermaid
flowchart TB
    subgraph user_space["User Space"]
        kubectl["kubectl / vcctl"]
    end

    subgraph control_plane["Kubernetes Control Plane"]
        apiserver["Kubernetes API Server"]
        etcd["etcd"]
    end

    subgraph volcano_system["Volcano System Components"]
        subgraph managers["Manager Deployments"]
            webhook["vc-webhook-manager"]
            controller["vc-controller-manager"]
            scheduler["vc-scheduler"]
        end
        subgraph node_level["Node-Level Agent"]
            agent["vc-agent (DaemonSet)"]
        end
    end

    subgraph worker_nodes["Worker Nodes"]
        node1["Node 1"]
        node2["Node 2"]
        node3["Node N"]
    end

    kubectl -->|"1. 提交 Job/Queue"| apiserver
    apiserver <-->|"持久化存储"| etcd

    webhook <-->|"2. Admit/Mutate"| apiserver
    controller <-->|"3. Watch & Reconcile"| apiserver
    scheduler <-->|"4. Watch & Bind"| apiserver

    agent -->|"上报节点指标"| apiserver
    agent -->|"cgroup 管理"| node1
    agent -->|"cgroup 管理"| node2
    agent -->|"cgroup 管理"| node3

    scheduler -.->|"调度决策"| node1
    scheduler -.->|"调度决策"| node2
    scheduler -.->|"调度决策"| node3

    style apiserver fill:#326CE5,color:#fff
    style etcd fill:#419EDA,color:#fff
    style webhook fill:#FF6B35,color:#fff
    style controller fill:#4ECDC4,color:#fff
    style scheduler fill:#9B59B6,color:#fff
    style agent fill:#F39C12,color:#fff
```

从架构图中可以看出，Volcano 遵循 Kubernetes 的 **声明式 API + 控制器** 设计范式。所有组件都不直接相互通信，而是以 API Server 作为唯一的数据交换中心。这种松耦合的架构设计使得各组件可以独立扩缩容、独立升级，同时保证了系统的高可用性。

---

## 2. 核心组件

### 2.1 vc-scheduler - 调度器

**入口文件**：`cmd/scheduler/main.go`

vc-scheduler 是 Volcano 的核心调度引擎，负责将待调度的 Pod（以 PodGroup 为调度单位）分配到合适的节点上运行。与 Kubernetes 默认调度器逐 Pod 调度不同，vc-scheduler 采用批量调度模式，在每个调度周期内同时处理多个 Job 和 PodGroup。

**核心包结构**：

- **`pkg/scheduler/scheduler.go`** - 调度器主循环。`Scheduler` 结构体持有 Cache 引用、Action 列表和 Plugin 配置，通过 `runOnce()` 方法驱动每个调度周期。
- **`pkg/scheduler/cache/`** - 集群状态缓存层。通过 Informer 机制 Watch API Server 中的 Pod、Node、PodGroup、Queue、HyperNode 等资源变更，维护一份集群状态的本地副本。对外提供 `Snapshot()` 接口生成不可变快照。
- **`pkg/scheduler/framework/`** - 调度框架核心。`Session` 是每个调度周期的上下文，包含快照数据（Jobs、Nodes、Queues）和所有注册的 Plugin 回调函数（PredicateFn、NodeOrderFn、PreemptableFn 等）。
- **`pkg/scheduler/actions/`** - 调度动作。每个 Action 实现一个调度阶段：
  - `enqueue` - 将满足条件的 Job 从 Pending 状态入队为 Inqueue
  - `allocate` - 为 Inqueue 的 Job 分配节点资源
  - `preempt` - 同一 Queue 内高优先级 Job 抢占低优先级 Job
  - `reclaim` - 跨 Queue 资源回收（当某 Queue 资源不足时从超额使用的 Queue 回收）
  - `backfill` - 回填调度，利用碎片资源调度小任务
  - `shuffle` - 打乱任务调度顺序，避免饥饿
- **`pkg/scheduler/plugins/`** - 调度算法插件，共 20+ 个，包括：
  - `gang` - Gang Scheduling 保证
  - `proportion` - Queue 资源配额分配
  - `drf` - Dominant Resource Fairness
  - `binpack` - 装箱调度
  - `nodeorder` - 节点打分排序
  - `predicates` - 节点过滤断言
  - `priority` - 任务优先级
  - `numaaware` - NUMA 拓扑感知
  - `network-topology-aware` - 网络拓扑感知（HyperNode）
  - `capacity` - 层级队列容量管理

**调度主循环**的核心逻辑如下：

```go
// pkg/scheduler/scheduler.go - runOnce()
func (pc *Scheduler) runOnce() {
    ssn := framework.OpenSession(pc.cache, plugins, configurations)
    defer framework.CloseSession(ssn)

    for _, action := range actions {
        action.Execute(ssn)
    }
}
```

每个调度周期：先通过 `OpenSession` 从 Cache 创建 Snapshot 并初始化 Session，然后依次执行配置的 Action 序列，最后在 `CloseSession` 时将调度决策刷回 Cache。

### 2.2 vc-controller-manager - 控制器管理器

**入口文件**：`cmd/controller-manager/main.go`

vc-controller-manager 是 Volcano 的资源生命周期管理中心，以多控制器模式运行，每个控制器负责一类 CRD 资源的协调（Reconcile）。

**包路径**：`pkg/controllers/`

管理的控制器列表：

| 控制器 | 包路径 | 职责 |
|-------|-------|------|
| Job Controller | `pkg/controllers/job/` | 管理 Volcano Job 生命周期，创建 PodGroup 和 Pod |
| PodGroup Controller | `pkg/controllers/podgroup/` | 为非 Volcano Job 的 workload 自动创建 PodGroup |
| Queue Controller | `pkg/controllers/queue/` | 管理 Queue 状态，计算资源配额 |
| JobFlow Controller | `pkg/controllers/jobflow/` | 管理 JobFlow 工作流编排 |
| JobTemplate Controller | `pkg/controllers/jobtemplate/` | 管理 JobTemplate 模板资源 |
| CronJob Controller | `pkg/controllers/cronjob/` | 定时任务管理 |
| HyperNode Controller | `pkg/controllers/hypernode/` | 管理网络拓扑 HyperNode 资源 |
| Sharding Controller | `pkg/controllers/sharding/` | 多调度器分片管理 |
| GC Controller | `pkg/controllers/garbagecollector/` | 资源垃圾回收 |

其中 **Job Controller** 是最核心的控制器，它以有限状态机（FSM）的模式管理 Volcano Job 的完整生命周期。当用户创建 Volcano Job 后，Job Controller 自动创建对应的 PodGroup 和所有 Task 对应的 Pod，并持续监控 Pod 状态以更新 Job 状态。

### 2.3 vc-webhook-manager - Webhook 管理器

**入口文件**：`cmd/webhook-manager/main.go`

vc-webhook-manager 实现了 Kubernetes Admission Webhook，在资源创建/更新请求到达 API Server 时进行拦截处理。

**包路径**：`pkg/webhooks/`

支持的 Webhook 类型：

| 资源类型 | 操作 | 说明 |
|---------|------|------|
| Jobs | Validate + Mutate | 校验 Job 配置合法性，自动填充默认值 |
| Queues | Validate + Mutate | 校验 Queue 配置，设置默认权重 |
| PodGroups | Validate + Mutate | 校验 PodGroup 参数 |
| Pods | Mutate | 为 Pod 注入调度器名称等标注 |
| CronJobs | Validate | 校验 CronJob Spec |
| HyperNodes | Validate | 校验 HyperNode 拓扑配置 |
| JobFlows | Validate | 校验 JobFlow DAG 配置 |

Webhook 管理器在资源进入 etcd 之前提供了一道安全屏障，确保所有 Volcano CRD 资源的配置正确性和完整性。

### 2.4 vc-agent - 节点代理

**入口文件**：`cmd/agent/main.go`

vc-agent 以 DaemonSet 形式部署在每个 Worker 节点上，主要服务于**混部（Colocation）** 场景。

**包路径**：`pkg/agent/`

核心能力：
- **资源超卖（Oversubscription）**：监控节点实际资源利用率，计算可超卖资源量并上报
- **cgroup 管理**：对混部场景下的离线任务进行 cgroup 级别的资源隔离和限制
- **健康检查**：节点级别的健康状态监控和上报
- **事件上报**：节点资源变更事件的采集和汇报

### 2.5 vcctl - CLI 工具

**入口文件**：`cmd/cli/`

vcctl 是 Volcano 提供的命令行管理工具，封装了对 Volcano CRD 的常用操作：

- `vsub` - 提交 Job
- `vjobs` - 查看 Job 列表和状态
- `vqueues` - 查看 Queue 列表和状态
- `vcancel` - 取消 Job
- `vsuspend` / `vresume` - 暂停/恢复 Job

---

## 3. 组件交互

Volcano 的各组件通过 Kubernetes API Server 进行间接通信，遵循 Kubernetes 的 Watch-Reconcile 模式。以下是各组件的交互关系：

1. **用户提交 Job**：用户通过 `kubectl` 或 `vcctl` 向 API Server 提交 Volcano Job
2. **Webhook 拦截**：vc-webhook-manager 拦截请求，执行 Validate（校验合法性）和 Mutate（注入默认值）
3. **Controller 协调**：vc-controller-manager 的 Job Controller Watch 到新 Job 后，创建对应的 PodGroup 和 Pods
4. **Scheduler 调度**：vc-scheduler 的 Cache 层 Watch 到新的 PodGroup 和 Pods，在下一个调度周期通过 Action/Plugin 流水线完成调度决策
5. **绑定执行**：Scheduler 将调度结果（Pod-Node Binding）写回 API Server，kubelet 拉起 Pod

以下序列图展示了一个 Volcano Job 从提交到 Pod 运行的完整端到端流程：

```mermaid
sequenceDiagram
    actor User as User
    participant kubectl as kubectl / vcctl
    participant API as API Server
    participant WH as vc-webhook-manager
    participant CM as vc-controller-manager
    participant Cache as Scheduler Cache
    participant Session as Scheduler Session
    participant Action as Actions (enqueue/allocate)
    participant Plugin as Plugins (gang/proportion/...)
    participant Kubelet as Kubelet

    User->>kubectl: 1. 编写 Job YAML 并提交
    kubectl->>API: 2. POST /apis/batch.volcano.sh/v1alpha1/jobs

    rect rgb(255, 245, 235)
        Note over API,WH: Admission 阶段
        API->>WH: 3. Mutating Webhook 调用
        WH-->>API: 4. 注入默认值 (schedulerName, queue, etc.)
        API->>WH: 5. Validating Webhook 调用
        WH-->>API: 6. 校验通过
    end

    API->>API: 7. 持久化 Job 到 etcd

    rect rgb(235, 245, 255)
        Note over API,CM: Controller 协调阶段
        API-->>CM: 8. Job Informer 通知新 Job
        CM->>API: 9. 创建 PodGroup
        CM->>API: 10. 创建 Pods (状态 Pending)
    end

    rect rgb(235, 255, 235)
        Note over API,Plugin: Scheduler 调度阶段
        API-->>Cache: 11. Informer 同步 PodGroup + Pods + Nodes
        Cache->>Session: 12. Snapshot() 生成不可变快照
        Session->>Action: 13. OpenSession 初始化
        Action->>Plugin: 14. Enqueue - 检查 Gang 约束
        Plugin-->>Action: 15. Job 满足 minAvailable - 入队
        Action->>Plugin: 16. Allocate - 节点过滤/打分
        Plugin-->>Action: 17. 返回最优节点列表
        Action->>Cache: 18. Bind Task 到目标 Node
    end

    Cache->>API: 19. 写入 Pod Binding
    API-->>Kubelet: 20. Pod 调度到目标节点
    Kubelet->>Kubelet: 21. 拉取镜像并启动容器
    Kubelet->>API: 22. 更新 Pod 状态为 Running
    API-->>CM: 23. Pod 状态更新通知
    CM->>API: 24. 更新 Job 状态为 Running
```

### 关键交互说明

**Admission 阶段**（步骤 3-6）：Webhook 管理器作为 API Server 的准入控制插件，在 Job 写入 etcd 之前完成校验和变更注入。Mutating Webhook 会自动为 Job 设置 `schedulerName: volcano`、默认 Queue 名称等字段；Validating Webhook 则校验 Task 配置、minAvailable 值等参数的合法性。

**Controller 协调阶段**（步骤 8-10）：Job Controller 通过 Informer Watch Job 资源变更。当新 Job 到达时，控制器首先创建与 Job 对应的 PodGroup（包含 minAvailable、Queue 等调度元信息），然后根据 Job Spec 中的 Tasks 定义创建对应数量的 Pod。此时 Pod 处于 Pending 状态，等待调度器分配。

**Scheduler 调度阶段**（步骤 11-18）：调度器的 Cache 层通过 Informer 持续同步集群中的 Pod、Node、PodGroup、Queue 等资源。每个调度周期开始时，Cache 生成一份不可变的 Snapshot，用于构建 Session。Session 中依次执行 Enqueue（入队判断）和 Allocate（资源分配）等 Action，每个 Action 内部调用注册的 Plugin 函数完成实际的调度逻辑。

---

## 4. 数据流

Volcano 调度器的数据流形成了一条从 API Server 出发、经过多层处理、最终回到 API Server 的闭环链路：

```
API Server → Informers → Cache → Snapshot → Session → Actions → Plugins → Bind → API Server
```

各层的职责如下：

1. **API Server** - Kubernetes 集群状态的唯一真实来源（Source of Truth）
2. **Informers** - 基于 Watch 机制的增量同步层，将 API Server 中的资源变更推送到 Cache
3. **Cache** - 集群状态的本地缓存，维护 Pod、Node、PodGroup、Queue、HyperNode 等资源的内存表示。Cache 通过 Event Handler 处理资源的 Add/Update/Delete 事件，保持与 API Server 的最终一致性
4. **Snapshot** - Cache 的深拷贝不可变快照。每个调度周期开始时由 `Cache.Snapshot()` 生成，确保调度过程中数据不被并发修改
5. **Session** - 调度周期的运行时上下文。在 `framework.OpenSession()` 中基于 Snapshot 初始化，包含所有 Jobs、Nodes、Queues 数据以及注册的 Plugin 回调函数
6. **Actions** - 调度阶段执行器。按配置顺序依次执行（通常为 enqueue → allocate → preempt → reclaim → backfill），每个 Action 操作 Session 中的数据
7. **Plugins** - 调度算法提供者。Action 执行过程中通过 Session 上注册的回调函数（PredicateFn、NodeOrderFn 等）调用 Plugin 逻辑
8. **Bind** - 调度结果持久化。将 Pod-to-Node 的绑定关系通过 Cache 写回 API Server

以下是 Cache 内部的事件驱动数据流详细展示：

```mermaid
flowchart LR
    subgraph apiserver_layer["API Server"]
        api["Kubernetes API Server"]
    end

    subgraph informer_layer["Informer Layer"]
        pi["Pod Informer"]
        ni["Node Informer"]
        pgi["PodGroup Informer"]
        qi["Queue Informer"]
        hni["HyperNode Informer"]
    end

    subgraph cache_layer["Cache Layer"]
        eh["Event Handlers<br/>(Add/Update/Delete)"]
        store["In-Memory Store<br/>(Jobs, Nodes, Queues)"]
        snap["Snapshot()"]
    end

    subgraph session_layer["Session Layer"]
        session["Session Context"]
        subgraph action_pipeline["Action Pipeline"]
            enqueue["Enqueue"]
            allocate["Allocate"]
            preempt["Preempt"]
            reclaim["Reclaim"]
            backfill["Backfill"]
        end
    end

    subgraph plugin_layer["Plugin Layer"]
        gang["Gang"]
        proportion["Proportion"]
        drf["DRF"]
        predicates["Predicates"]
        nodeorder["NodeOrder"]
        binpack["Binpack"]
        others["..."]
    end

    subgraph bind_layer["Bind Layer"]
        bind["AddBindTask()"]
    end

    api --> pi & ni & pgi & qi & hni
    pi & ni & pgi & qi & hni --> eh
    eh --> store
    store --> snap
    snap --> session

    session --> enqueue --> allocate --> preempt --> reclaim --> backfill

    allocate <--> gang & proportion & drf & predicates & nodeorder & binpack & others
    preempt <--> gang & proportion & drf & predicates & nodeorder & binpack & others

    backfill --> bind
    allocate --> bind
    preempt --> bind
    reclaim --> bind
    bind -->|"Pod Binding"| api

    style api fill:#326CE5,color:#fff
    style session fill:#9B59B6,color:#fff
    style bind fill:#E74C3C,color:#fff
```

**数据一致性保证**：Cache 层的 Snapshot 机制是调度器数据一致性的关键。由于调度决策可能耗时较长，直接操作 Cache 中的实时数据会导致并发问题。通过在每个调度周期开始时创建 Snapshot 深拷贝，调度过程中所有的状态查询和修改都在这份隔离的快照上进行，不影响 Cache 接收新的 Informer 事件。调度结束后，最终的绑定结果通过 `AddBindTask` 写回 Cache 并提交到 API Server。

---

## 5. CRD 类型关系

Volcano 通过多组 CRD 扩展 Kubernetes API，每组 CRD 属于不同的 API Group，服务于不同的功能域。以下是所有 CRD 类型的分类和关系：

### 5.1 API Group 概览

| API Group | 版本 | 资源类型 | 用途 |
|-----------|------|---------|------|
| `batch.volcano.sh` | v1alpha1 | Job, CronJob | 批量作业定义与定时执行 |
| `scheduling.volcano.sh` | v1beta1 | PodGroup, Queue | 调度单元与队列管理 |
| `topology.volcano.sh` | v1alpha1 | HyperNode | 网络拓扑感知调度 |
| `flow.volcano.sh` | v1alpha1 | JobFlow, JobTemplate | 工作流编排 |
| `bus.volcano.sh` | v1alpha1 | Command | Job 控制命令传递 |

### 5.2 CRD 关系图

以下类图展示了 Volcano 各 CRD 类型之间的关联关系：

```mermaid
classDiagram
    direction TB

    namespace batch_volcano_sh {
        class Job {
            +JobSpec spec
            +JobStatus status
            +Tasks[] tasks
            +Queue queue
            +SchedulerName schedulerName
            +MinAvailable int32
            +Plugins map
        }

        class CronJob {
            +CronJobSpec spec
            +Schedule string
            +JobTemplate jobTemplate
        }
    }

    namespace scheduling_volcano_sh {
        class PodGroup {
            +PodGroupSpec spec
            +PodGroupStatus status
            +MinMember int32
            +Queue string
            +MinResources ResourceList
        }

        class Queue {
            +QueueSpec spec
            +QueueStatus status
            +Weight int32
            +Capability ResourceList
            +Guarantee Guarantee
            +Reclaimable bool
        }
    }

    namespace topology_volcano_sh {
        class HyperNode {
            +HyperNodeSpec spec
            +HyperNodeStatus status
            +Tier int
            +Members MemberSpec[]
        }
    }

    namespace flow_volcano_sh {
        class JobFlow {
            +JobFlowSpec spec
            +JobFlowStatus status
            +Flows Flow[]
            +DependsOn DependsOn
        }

        class JobTemplate {
            +JobTemplateSpec spec
            +JobSpec jobSpec
        }
    }

    namespace bus_volcano_sh {
        class Command {
            +TargetObject target
            +Action string
            +Message string
        }
    }

    Job "1" --> "1" PodGroup : "自动创建"
    Job "1" --> "N" Pod : "创建 Task Pods"
    Job "N" --> "1" Queue : "提交到 Queue"
    CronJob "1" --> "N" Job : "定时触发"
    PodGroup "N" --> "1" Queue : "归属 Queue"
    JobFlow "1" --> "N" JobTemplate : "引用模板"
    JobFlow "1" --> "N" Job : "编排 Job DAG"
    Command "1" --> "1" Job : "控制命令"
    HyperNode "1" --> "N" HyperNode : "层级拓扑 (parent-child)"
    HyperNode "1" --> "N" Node : "包含节点"

    class Pod {
        <<Kubernetes Native>>
        +PodSpec spec
        +schedulerName string
        +annotations map
    }

    class Node {
        <<Kubernetes Native>>
        +NodeSpec spec
        +Allocatable ResourceList
    }
```

### 5.3 核心 CRD 关系说明

**Job 与 PodGroup 的一对一关系**：当用户提交 Volcano Job 后，Job Controller 自动为其创建一个同名的 PodGroup。PodGroup 是调度器的基本调度单元，它定义了 Gang Scheduling 所需的最小就绪 Pod 数量（`minMember`）和队列归属。调度器以 PodGroup 而非单个 Pod 为粒度进行调度决策。

**Job 与 Queue 的多对一关系**：所有 Job 通过 `spec.queue` 字段指定其归属的 Queue。Queue 定义了资源配额（`capability`）、权重（`weight`）和资源保障（`guarantee`），调度器的 proportion 等插件基于 Queue 维度进行资源划分和公平调度。

**JobFlow 的 DAG 编排**：JobFlow 通过引用 JobTemplate 定义一组有依赖关系的 Job 执行图。每个 Flow 节点可以指定 `dependsOn` 来声明前置依赖，JobFlow Controller 根据依赖关系按序创建和管理 Job。

**HyperNode 的层级拓扑**：HyperNode 是 Volcano 的网络拓扑抽象，通过 `tier` 字段表示拓扑层级（如 Rack、Switch、DataCenter），通过 `members` 字段关联子 HyperNode 或 Node。调度器在 Session 初始化时构建完整的 HyperNode 拓扑树，用于网络拓扑感知调度 -- 将同一 Job 的 Pod 尽量调度到网络距离最近的节点上。

**Command 的控制传递**：Command 是 Volcano 的内部控制信号 CRD，属于 `bus.volcano.sh` API Group。当用户通过 vcctl 执行 `vsuspend`、`vresume` 等操作时，实际是创建一个 Command 对象指向目标 Job，Job Controller Watch 到 Command 后执行对应的状态转换。

---

## 6. 小结

Volcano 的架构设计体现了以下几个关键原则：

1. **松耦合通信**：所有组件通过 API Server 间接通信，无组件间直连依赖
2. **可扩展的调度流水线**：Action + Plugin 架构允许灵活组合调度策略，用户可自定义 Plugin 扩展调度算法
3. **批量调度优先**：以 PodGroup 而非 Pod 为调度单位，原生支持 Gang Scheduling
4. **多层缓存与快照隔离**：Informer → Cache → Snapshot 的多层数据流设计，兼顾实时性和一致性
5. **声明式资源管理**：通过 CRD 扩展 Kubernetes 原生 API，所有资源管理遵循 Kubernetes 的声明式设计模式

理解这些架构核心后，可以进一步深入各组件的实现细节。建议的阅读路径为：先从 vc-scheduler 的调度主循环入手（`pkg/scheduler/scheduler.go`），再深入 Action 和 Plugin 的执行逻辑，最后研究 Controller 和 Webhook 的实现。
