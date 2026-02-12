---
title: "调度周期 runOnce() 完整追踪"
weight: 2
---

## 1. 概述

`runOnce()` 是 Volcano Scheduler 的心跳方法。每个调度周期由 `wait.Until` 按 `schedulePeriod` 间隔触发一次 `runOnce()`，完成一轮完整的调度决策。理解 `runOnce()` 的每个子阶段对于性能调优和故障排查至关重要。

本文档从 DEBUG 视角逐步拆解 `runOnce()` 的执行过程，包括锁获取、快照创建、Session 打开、Action 执行、Session 关闭等核心阶段。

### 1.1 runOnce 生命周期概览

```mermaid
flowchart LR
    subgraph cycle["runOnce 调度周期"]
        direction TB
        A["获取锁 - 复制配置"] --> B["构建 EnabledActionMap"]
        B --> C["OpenSession"]
        C --> D["执行 Actions"]
        D --> E["CloseSession"]
        E --> F["记录 Metrics"]
    end

    subgraph opensession["OpenSession 子阶段"]
        direction TB
        C1["cache.Snapshot()"] --> C2["加载 HyperNode 拓扑"]
        C2 --> C3["实例化 Plugins"]
        C3 --> C4["OnSessionOpen 回调"]
        C4 --> C5["InitCycleState"]
    end

    subgraph closesession["CloseSession 子阶段"]
        direction TB
        E1["Plugin.OnSessionClose"] --> E2["JobUpdater.UpdateAll"]
        E2 --> E3["updateQueueStatus"]
        E3 --> E4["cache.OnSessionClose"]
    end

    C --> opensession
    E --> closesession

    style cycle fill:#e3f2fd,stroke:#1565c0
    style opensession fill:#e8f5e9,stroke:#2e7d32
    style closesession fill:#fce4ec,stroke:#c62828
```

---

## 2. 调度周期全景图

### 2.1 完整时序图

```mermaid
sequenceDiagram
    participant Timer as wait.Until Timer
    participant Sched as Scheduler
    participant Cache as SchedulerCache
    participant Ssn as Session
    participant Plugin as Plugins
    participant Action as Actions
    participant JU as JobUpdater
    participant QS as QueueStatus

    Timer->>Sched: 触发 runOnce()
    Note over Sched: scheduleStartTime = time.Now()

    rect rgb(227, 242, 253)
        Note over Sched: 阶段一 - 配置加载
        Sched->>Sched: mutex.Lock()
        Sched->>Sched: 复制 actions/plugins/configurations
        Sched->>Sched: mutex.Unlock()
        Sched->>Sched: 构建 EnabledActionMap
    end

    rect rgb(232, 245, 233)
        Note over Sched,Plugin: 阶段二 - OpenSession
        Sched->>Cache: OnSessionOpen()
        Sched->>Cache: Snapshot()
        Cache->>Cache: Lock + 深拷贝 Jobs/Nodes/Queues
        Cache->>Cache: 并行 cloneJob
        Cache-->>Ssn: ClusterInfo 快照

        Ssn->>Ssn: 初始化数据结构
        Ssn->>Ssn: 加载 HyperNode 拓扑
        Ssn->>Ssn: addClusterTopHyperNode
        Ssn->>Ssn: GenerateNodeMapAndSlice

        loop 每个 Tier 的每个 Plugin
            Ssn->>Plugin: GetPluginBuilder(name)
            Plugin->>Plugin: plugin = pb(arguments)
            Ssn->>Plugin: plugin.OnSessionOpen(ssn)
            Plugin->>Ssn: 注册 FnMap (OrderFn, PredicateFn...)
        end

        Ssn->>Ssn: InitCycleState()
    end

    rect rgb(255, 243, 224)
        Note over Sched,Action: 阶段三 - Action 执行
        loop 每个 Action
            Note over Sched: actionStartTime = time.Now()
            Sched->>Action: action.Execute(ssn)
            Action->>Ssn: 调用 Plugin Hooks
            Ssn-->>Action: 返回结果
            Sched->>Sched: UpdateActionDuration
        end
    end

    rect rgb(252, 228, 236)
        Note over Sched,QS: 阶段四 - CloseSession
        loop 每个 Plugin
            Ssn->>Plugin: plugin.OnSessionClose(ssn)
        end

        Ssn->>JU: NewJobUpdater(ssn)
        JU->>JU: UpdateAll() - 同步 PodGroup 状态
        Ssn->>QS: updateQueueStatus(ssn)
        QS->>QS: 计算各 Queue 的 allocated 资源

        Ssn->>Ssn: 清空 Session 字段
        Ssn->>Cache: OnSessionClose()
    end

    Note over Sched: UpdateE2eDuration
```

### 2.2 各阶段耗时分布

```mermaid
pie title "runOnce 各阶段典型耗时占比"
    "配置加载与锁获取" : 1
    "Snapshot 创建" : 15
    "Plugin OnSessionOpen" : 10
    "Action 执行" : 60
    "CloseSession" : 14
```

---

## 3. 阶段一 - 配置加载与锁获取

### 3.1 源码分析

源码位于 `pkg/scheduler/scheduler.go` 第 107-122 行：

```go
// pkg/scheduler/scheduler.go:107
func (pc *Scheduler) runOnce() {
    klog.V(4).Infof("Start scheduling ...")
    scheduleStartTime := time.Now()
    defer klog.V(4).Infof("End scheduling ...")

    pc.mutex.Lock()
    actions := pc.actions
    plugins := pc.plugins
    configurations := pc.configurations
    pc.mutex.Unlock()

    // Load ConfigMap to check which action is enabled.
    conf.EnabledActionMap = make(map[string]bool)
    for _, action := range actions {
        conf.EnabledActionMap[action.Name()] = true
    }
    // ...
}
```

### 3.2 关键细节

**为什么需要复制配置？**

Scheduler 支持配置文件热加载（通过 `fsnotify` 文件监听）。`watchSchedulerConf` 在后台 goroutine 中运行，当配置文件变更时会调用 `loadSchedulerConf()` 更新 `pc.actions/plugins/configurations`。为了避免在调度周期中配置被修改，这里通过 mutex 做了快照拷贝。

**EnabledActionMap 的作用：**

`conf.EnabledActionMap` 是一个全局 Map，用于让某些 Plugin/Action 知道当前哪些 Action 被启用。例如，Allocate Action 的 `buildAllocateContext` 中会检查 enqueue Action 是否启用：

```go
// pkg/scheduler/actions/allocate/allocate.go:152
if conf.EnabledActionMap["enqueue"] {
    klog.V(4).Infof("Job <%s/%s> Queue <%s> skip allocate, reason: job status is pending.",
        job.Namespace, job.Name, job.Queue)
    continue
}
```

### 3.3 调试建议

| 检查项 | 方法 |
|-------|------|
| 配置是否正确加载 | 查看 V(2) 日志 `Finished loading scheduler config` |
| 热加载是否生效 | 修改配置后查看 `watch event` 日志 |
| 锁竞争 | 分析 pprof mutex profile |

---

## 4. 阶段二 - OpenSession 详解

### 4.1 OpenSession 入口

源码位于 `pkg/scheduler/framework/framework.go` 第 34-58 行：

```go
// pkg/scheduler/framework/framework.go:34
func OpenSession(cache cache.Cache, tiers []conf.Tier, configurations []conf.Configuration) *Session {
    ssn := openSession(cache)
    ssn.Tiers = tiers
    ssn.Configurations = configurations
    ssn.NodeMap = GenerateNodeMapAndSlice(ssn.Nodes)
    ssn.PodLister = NewPodLister(ssn)

    for _, tier := range tiers {
        for _, plugin := range tier.Plugins {
            if pb, found := GetPluginBuilder(plugin.Name); !found {
                klog.Errorf("Failed to get plugin %s.", plugin.Name)
            } else {
                plugin := pb(plugin.Arguments)
                ssn.plugins[plugin.Name()] = plugin
                onSessionOpenStart := time.Now()
                plugin.OnSessionOpen(ssn)
                metrics.UpdatePluginDuration(plugin.Name(), metrics.OnSessionOpen,
                    metrics.Duration(onSessionOpenStart))
            }
        }
    }

    ssn.InitCycleState()
    return ssn
}
```

### 4.2 openSession - 核心初始化

源码位于 `pkg/scheduler/framework/session.go` 第 165-282 行。该函数完成以下关键步骤：

```mermaid
flowchart TD
    A["openSession(cache)"] --> B["cache.OnSessionOpen()"]
    B --> C["初始化 Session 数据结构"]
    C --> D["cache.Snapshot()"]

    D --> E["复制 Jobs 到 Session"]
    E --> F["保存 PodGroupOldState"]
    F --> G["构建 NodeList"]
    G --> H["加载 HyperNodes 快照"]

    H --> I["addClusterTopHyperNode"]
    I --> J["parseHyperNodesTiers"]
    J --> K{"HyperNodesReady?"}

    K -->|"Yes"| L["removeInvalidAllocatedHyperNode"]
    L --> M["recoverAllocatedHyperNode"]
    K -->|"No"| N["跳过 HyperNode 相关操作"]

    M --> O["复制 Nodes/Queues/NamespaceInfo"]
    N --> O
    O --> P["计算 TotalResource"]

    style A fill:#e3f2fd,stroke:#1565c0
    style D fill:#fff3e0,stroke:#e65100
    style I fill:#f3e5f5,stroke:#6a1b9a
    style P fill:#e8f5e9,stroke:#2e7d32
```

### 4.3 Snapshot 深度分析

Snapshot 是整个调度周期最关键的操作之一 - 它为调度器创建一个一致的集群状态视图。

源码位于 `pkg/scheduler/cache/cache.go` 第 1423-1533 行：

```go
// pkg/scheduler/cache/cache.go:1423
func (sc *SchedulerCache) Snapshot() *schedulingapi.ClusterInfo {
    sc.Mutex.Lock()
    defer sc.Mutex.Unlock()

    snapshot := &schedulingapi.ClusterInfo{
        Nodes:   make(map[string]*schedulingapi.NodeInfo),
        Jobs:    make(map[schedulingapi.JobID]*schedulingapi.JobInfo),
        Queues:  make(map[schedulingapi.QueueID]*schedulingapi.QueueInfo),
        // ... 其他字段初始化 ...
    }

    // 1. 复制 NodeList
    copy(snapshot.NodeList, sc.NodeList)

    // 2. 复制 Nodes（仅 Ready 节点）
    for _, value := range sc.Nodes {
        if !value.Ready() { continue }
        snapshot.Nodes[value.Name] = value.Clone()
    }

    // 3. 快照 HyperNodes（单独加锁）
    sc.HyperNodesInfo.Lock()
    snapshot.HyperNodes = sc.HyperNodesInfo.HyperNodes()
    // ...
    sc.HyperNodesInfo.Unlock()

    // 4. 复制 Queues
    for _, value := range sc.Queues {
        snapshot.Queues[value.UID] = value.Clone()
    }

    // 5. 并行克隆 Jobs
    var cloneJobLock sync.Mutex
    var wg sync.WaitGroup
    cloneJob := func(value *schedulingapi.JobInfo) {
        defer wg.Done()
        clonedJob := value.Clone()
        cloneJobLock.Lock()
        snapshot.Jobs[value.UID] = clonedJob
        cloneJobLock.Unlock()
    }
    for _, value := range sc.Jobs {
        if value.PodGroup == nil { continue }
        if _, found := snapshot.Queues[value.Queue]; !found { continue }
        wg.Add(1)
        go cloneJob(value)
    }
    wg.Wait()

    return snapshot
}
```

### 4.4 Snapshot 关键调试点

```mermaid
flowchart TD
    subgraph snapshot_debug["Snapshot 调试要点"]
        A["Snapshot 入口"] --> B{"Node 数量是否正确?"}
        B -->|"不符"| C["检查 Node Ready 状态"]
        C --> C1["node.Ready() 依赖 NodeInfo 中的条件"]

        B -->|"正确"| D{"Job 数量是否正确?"}
        D -->|"缺少"| E["检查 PodGroup 是否为 nil"]
        E --> E1["PodGroup 未创建 / Informer 未同步"]
        D -->|"缺少"| F["检查 Queue 是否存在"]
        F --> F1["Job 的 Queue 不在 Cache 中"]

        D -->|"正确"| G{"HyperNodes 是否 Ready?"}
        G -->|"No"| H["HyperNode CRD 未创建或不完整"]
        G -->|"Yes"| I["Snapshot 正常"]
    end

    style A fill:#e3f2fd,stroke:#1565c0
    style C fill:#fce4ec,stroke:#c62828
    style E fill:#fce4ec,stroke:#c62828
    style I fill:#e8f5e9,stroke:#2e7d32
```

**Snapshot 日志关键模式：**

```bash
# V(3) - 查看 Snapshot 汇总
kubectl logs <scheduler> | grep "SnapShot for scheduling"
# 输出: SnapShot for scheduling jobNum=15, QueueNum=3, NodeNum=10

# V(4) - 查看 HyperNode 详情
kubectl logs <scheduler> -v 4 | grep "HyperNode snapShot"

# V(4) - 查看 Job 优先级
kubectl logs <scheduler> -v 4 | grep "The priority of job"
```

### 4.5 Plugin 实例化与 OnSessionOpen

Plugin 实例化发生在 `OpenSession` 中，按 Tier 顺序遍历。每个 Plugin 的 `OnSessionOpen` 方法会向 Session 注册各种 Hook 函数。

```mermaid
flowchart TD
    A["OpenSession"] --> B["遍历 Tiers"]
    B --> C["遍历 Tier 中的 Plugins"]
    C --> D["GetPluginBuilder(name)"]
    D --> E["pb(arguments) - 创建 Plugin 实例"]
    E --> F["plugin.OnSessionOpen(ssn)"]

    F --> G["注册 Hook 函数"]
    G --> G1["AddJobOrderFn"]
    G --> G2["AddQueueOrderFn"]
    G --> G3["AddPredicateFn"]
    G --> G4["AddNodeOrderFn"]
    G --> G5["AddJobReadyFn"]
    G --> G6["AddOverusedFn"]
    G --> G7["...其他 20+ Hook"]

    style A fill:#e3f2fd,stroke:#1565c0
    style F fill:#e8f5e9,stroke:#2e7d32
    style G fill:#fff3e0,stroke:#e65100
```

**常见 Plugin 注册的 Hook：**

| Plugin | 注册的 Hook | 用途 |
|--------|-----------|------|
| `gang` | JobOrderFn, JobReadyFn, JobPipelinedFn, JobValidFn | Gang Scheduling 约束 |
| `drf` | JobOrderFn, EventHandler, PreemptableFn | DRF 公平调度 |
| `proportion` | QueueOrderFn, OverusedFn, AllocatableFn, JobEnqueueableFn | Queue 配额管理 |
| `predicates` | PredicateFn, PrePredicateFn | K8s 兼容性过滤 |
| `nodeorder` | BatchNodeOrderFn | Node 评分 |
| `binpack` | NodeOrderFn | 资源紧凑分配 |

### 4.6 调试断点建议

| 断点位置 | 文件路径 | 用途 |
|---------|---------|------|
| `openSession` | `pkg/scheduler/framework/session.go:165` | Session 初始化 |
| `cache.Snapshot()` | `pkg/scheduler/cache/cache.go:1424` | 快照创建 |
| `plugin.OnSessionOpen` | `pkg/scheduler/framework/framework.go:49` | Plugin 初始化 |
| `AddPredicateFn` | `pkg/scheduler/framework/session_plugins.go` | 确认 Hook 注册 |

---

## 5. 阶段三 - Action 流水线执行

### 5.1 Action 执行循环

源码位于 `pkg/scheduler/scheduler.go` 第 130-134 行：

```go
// pkg/scheduler/scheduler.go:130
for _, action := range actions {
    actionStartTime := time.Now()
    action.Execute(ssn)
    metrics.UpdateActionDuration(action.Name(), metrics.Duration(actionStartTime))
}
```

### 5.2 默认 Action 流水线

```mermaid
flowchart LR
    subgraph pipeline["Action Pipeline"]
        A["enqueue"] --> B["allocate"]
        B --> C["backfill"]
        C --> D["reclaim"]
        D --> E["preempt"]
        E --> F["shuffle"]
    end

    subgraph enqueue_desc["enqueue"]
        A1["将 Pending Job 转为 Inqueue"]
    end

    subgraph allocate_desc["allocate"]
        B1["核心资源分配"]
        B2["Queue 迭代 -> Job 迭代 -> Task 分配"]
    end

    subgraph backfill_desc["backfill"]
        C1["为 BestEffort Task 分配"]
    end

    subgraph reclaim_desc["reclaim"]
        D1["跨 Queue 资源回收"]
    end

    subgraph preempt_desc["preempt"]
        E1["同 Queue 内高优先级抢占"]
    end

    subgraph shuffle_desc["shuffle"]
        F1["驱逐不符合约束的 Task"]
    end

    A -.-> enqueue_desc
    B -.-> allocate_desc
    C -.-> backfill_desc
    D -.-> reclaim_desc
    E -.-> preempt_desc
    F -.-> shuffle_desc

    style pipeline fill:#e3f2fd,stroke:#1565c0
```

### 5.3 各 Action 的数据流

```mermaid
flowchart TD
    subgraph session_state["Session 状态"]
        Jobs["Jobs Map"]
        Nodes["Nodes Map"]
        Queues["Queues Map"]
    end

    subgraph enqueue_action["Enqueue Action"]
        EA["遍历 Pending Jobs"]
        EA --> EB["JobEnqueueableFn 检查"]
        EB --> EC["更新 PodGroup Phase 为 Inqueue"]
    end

    subgraph allocate_action["Allocate Action"]
        AA["QueueOrderFn 排序"]
        AA --> AB["JobOrderFn 排序"]
        AB --> AC["TaskOrderFn 排序"]
        AC --> AD["PredicateFn 过滤 Node"]
        AD --> AE["NodeOrderFn 评分"]
        AE --> AF["stmt.Allocate / Pipeline"]
    end

    subgraph reclaim_action["Reclaim Action"]
        RA["找到 Starving Queue"]
        RA --> RB["从 Overused Queue 回收"]
        RB --> RC["ReclaimableFn 检查"]
        RC --> RD["stmt.Evict + stmt.Pipeline"]
    end

    session_state --> enqueue_action
    enqueue_action --> allocate_action
    allocate_action --> reclaim_action

    style session_state fill:#e8f5e9,stroke:#2e7d32
    style allocate_action fill:#fff3e0,stroke:#e65100
```

### 5.4 Action 执行耗时监控

每个 Action 执行后都会记录 Prometheus Metric：

```go
metrics.UpdateActionDuration(action.Name(), metrics.Duration(actionStartTime))
```

**关键 Metric：**

| Metric | 标签 | 说明 |
|--------|------|------|
| `action_duration_seconds` | `action=enqueue` | Enqueue 耗时 |
| `action_duration_seconds` | `action=allocate` | Allocate 耗时（通常最长） |
| `action_duration_seconds` | `action=preempt` | Preempt 耗时 |

**调试建议：**

```bash
# 查看各 Action 耗时
curl http://localhost:8080/metrics | grep action_duration

# 对比各 Action 占比
curl http://localhost:8080/metrics | grep action_duration_seconds_sum
```

---

## 6. 阶段四 - CloseSession 详解

### 6.1 CloseSession 入口

源码位于 `pkg/scheduler/framework/framework.go` 第 61-70 行：

```go
// pkg/scheduler/framework/framework.go:61
func CloseSession(ssn *Session) {
    for _, plugin := range ssn.plugins {
        onSessionCloseStart := time.Now()
        plugin.OnSessionClose(ssn)
        metrics.UpdatePluginDuration(plugin.Name(), metrics.OnSessionClose,
            metrics.Duration(onSessionCloseStart))
    }

    closeSession(ssn)
    ssn.cache.OnSessionClose()
}
```

### 6.2 closeSession 核心逻辑

源码位于 `pkg/scheduler/framework/session.go` 第 597-617 行：

```go
// pkg/scheduler/framework/session.go:597
func closeSession(ssn *Session) {
    ju := NewJobUpdater(ssn)
    ju.UpdateAll()

    updateQueueStatus(ssn)

    ssn.Jobs = nil
    ssn.Nodes = nil
    ssn.RevocableNodes = nil
    ssn.plugins = nil
    ssn.eventHandlers = nil
    ssn.jobOrderFns = nil
    ssn.queueOrderFns = nil
    ssn.clusterOrderFns = nil
    ssn.NodeList = nil
    ssn.TotalResource = nil

    ssn.cache.OnSessionClose()

    klog.V(3).Infof("Close Session %v", ssn.UID)
}
```

### 6.3 CloseSession 流程图

```mermaid
flowchart TD
    A["CloseSession 入口"] --> B["Plugin OnSessionClose"]
    B --> B1["proportion - 清理 Queue 状态"]
    B --> B2["drf - 清理 DRF 状态"]
    B --> B3["gang - 清理 Gang 状态"]

    B --> C["closeSession(ssn)"]
    C --> D["NewJobUpdater(ssn)"]
    D --> E["JobUpdater.UpdateAll()"]

    E --> E1["遍历 DirtyJobs"]
    E1 --> E2{"PodGroup 状态变化?"}
    E2 -->|"Yes"| E3["cache.RecordJobStatusEvent"]
    E2 -->|"Yes"| E4["更新 PodGroup Annotations"]
    E2 -->|"No"| E5["跳过"]

    C --> F["updateQueueStatus(ssn)"]
    F --> F1["遍历所有 Queue"]
    F1 --> F2["计算 allocated 资源"]
    F2 --> F3["cache.UpdateQueueStatus"]

    C --> G["清空 Session 字段"]
    G --> H["cache.OnSessionClose()"]

    style A fill:#e3f2fd,stroke:#1565c0
    style E fill:#e8f5e9,stroke:#2e7d32
    style F fill:#fff3e0,stroke:#e65100
    style H fill:#fce4ec,stroke:#c62828
```

### 6.4 JobUpdater 详解

JobUpdater 负责将调度周期内的 Job 状态变化同步到 API Server。

```mermaid
flowchart TD
    A["JobUpdater.UpdateAll()"] --> B["遍历 ssn.Jobs"]
    B --> C{"Job 在 DirtyJobs 中?"}
    C -->|"No"| D["比较 PodGroup 新旧状态"]
    C -->|"Yes"| E["强制更新"]

    D --> F{"状态有变化?"}
    F -->|"Yes"| E
    F -->|"No"| G["跳过"]

    E --> H["jobStatus(ssn, job)"]
    H --> I["计算 Phase"]
    I --> I1["Running tasks + Unschedulable -> Unknown"]
    I --> I2["Scheduled >= MinMember -> Running"]
    I --> I3["所有 Scheduled 完成 -> Completed"]
    I --> I4["其他 -> Pending / Inqueue"]

    H --> J["更新 Running/Failed/Succeeded 计数"]
    J --> K["cache.RecordJobStatusEvent"]
    K --> L["更新 PodGroup annotations"]

    style A fill:#e3f2fd,stroke:#1565c0
    style H fill:#e8f5e9,stroke:#2e7d32
    style K fill:#fff3e0,stroke:#e65100
```

### 6.5 updateQueueStatus 详解

```go
// 伪代码 - updateQueueStatus 核心逻辑
func updateQueueStatus(ssn *Session) {
    for _, queue := range ssn.Queues {
        allocated := api.EmptyResource()
        for _, job := range ssn.Jobs {
            if job.Queue != queue.UID { continue }
            for status, tasks := range job.TaskStatusIndex {
                if api.AllocatedStatus(status) {
                    for _, task := range tasks {
                        allocated.Add(task.Resreq)
                    }
                }
            }
        }
        // 更新 Queue 的 allocated 字段
        ssn.cache.UpdateQueueStatus(queue, allocated)
    }
}
```

### 6.6 调试断点建议

| 断点位置 | 文件路径 | 用途 |
|---------|---------|------|
| `CloseSession` | `pkg/scheduler/framework/framework.go:61` | CloseSession 入口 |
| `closeSession` | `pkg/scheduler/framework/session.go:597` | 核心清理逻辑 |
| `JobUpdater.UpdateAll` | `pkg/scheduler/framework/session.go` | Job 状态同步 |
| `updateQueueStatus` | `pkg/scheduler/framework/session.go` | Queue 状态更新 |

---

## 7. 性能分析

### 7.1 各阶段耗时瓶颈

```mermaid
flowchart TD
    subgraph bottlenecks["性能瓶颈分析"]
        A["Snapshot 锁竞争"] --> A1["Cache.Mutex 是全局锁"]
        A1 --> A2["Informer 回调也需要获取锁"]
        A1 --> A3["大集群下 Clone 耗时长"]

        B["Action 执行瓶颈"] --> B1["Allocate 是最耗时的 Action"]
        B1 --> B2["Predicate 逐 Node 过滤"]
        B1 --> B3["NodeOrder 评分计算"]
        B1 --> B4["HyperNode 拓扑感知分配"]

        C["CloseSession 开销"] --> C1["JobUpdater 逐 Job 更新"]
        C1 --> C2["API Server 调用延迟"]
        C --> C3["updateQueueStatus 遍历所有 Job"]
    end

    style A fill:#fce4ec,stroke:#c62828
    style B fill:#fff3e0,stroke:#e65100
    style C fill:#fff3e0,stroke:#e65100
```

### 7.2 Snapshot 锁竞争分析

Snapshot 期间需要持有 `SchedulerCache.Mutex` 全局锁，这意味着在 Snapshot 执行过程中，所有 Informer 的 Add/Update/Delete 回调都会被阻塞。

**优化建议：**

1. **并行 Job Clone**: 源码中已实现（`go cloneJob(value)`），但 `cloneJobLock` 仍是序列化点
2. **减少 Clone 深度**: 对于不变的字段考虑浅拷贝
3. **增加 schedulePeriod**: 减少 Snapshot 频率以降低锁竞争

**监控方法：**

```bash
# 查看 Snapshot 日志中 Job/Node/Queue 数量
kubectl logs <scheduler> -v 3 | grep "SnapShot for scheduling"

# 使用 pprof 分析锁竞争
go tool pprof http://localhost:8080/debug/pprof/mutex
```

### 7.3 Action 执行性能分析

```bash
# 获取各 Action 耗时的 Prometheus 数据
curl -s http://localhost:8080/metrics | grep action_duration_seconds

# 示例输出
# action_duration_seconds_sum{action="enqueue"} 0.001
# action_duration_seconds_sum{action="allocate"} 2.345
# action_duration_seconds_sum{action="backfill"} 0.012
# action_duration_seconds_sum{action="reclaim"} 0.089
# action_duration_seconds_sum{action="preempt"} 0.156
```

### 7.4 Plugin 耗时分析

每个 Plugin 的 `OnSessionOpen` 和 `OnSessionClose` 都有独立的 Metric：

```bash
# 查看 Plugin 耗时
curl -s http://localhost:8080/metrics | grep plugin_scheduling_duration_seconds

# 关注以下 Plugin
# - proportion (OnSessionOpen 中计算 Queue 资源)
# - predicates (可能涉及 API 调用)
# - capacity (层级 Queue 资源计算)
```

---

## 8. 调试工具

### 8.1 Prometheus Metrics 完整列表

| Metric | 类型 | 说明 |
|--------|------|------|
| `e2e_scheduling_duration_seconds` | Histogram | runOnce 总耗时 |
| `action_duration_seconds` | Histogram | 单个 Action 耗时 |
| `plugin_scheduling_duration_seconds` | Histogram | Plugin OnSessionOpen/Close 耗时 |
| `task_schedule_duration_seconds` | Histogram | Task 从创建到 Bind 的延迟 |
| `schedule_attempts_total` | Counter | 调度尝试次数 |

### 8.2 klog V Level 建议

| 场景 | 推荐 V Level | 输出内容 |
|------|-------------|---------|
| 生产环境 | V(2) | 组件启动、配置变更 |
| 日常监控 | V(3) | Session 开关、Pod 增删、Queue 状态 |
| 问题排查 | V(4) | 调度周期详情、Job 跳过原因、优先级信息 |
| 深度调试 | V(5) | 每个 Node 的资源详情、HyperNode 拓扑、Predicate 详细结果 |

### 8.3 Cache Dumper 使用

```bash
# 启用 Cache Dumper
volcano-scheduler --enable-cache-dumper --cache-dump-file-dir=/tmp/volcano-dump

# 触发 Dump（发送 SIGUSR2 信号）
kill -USR2 $(pidof vc-scheduler)

# 查看 Dump 文件
ls /tmp/volcano-dump/
cat /tmp/volcano-dump/cache-dump-*.json
```

Cache Dumper 会输出 SchedulerCache 的完整状态，包括：

- 所有 Node 的资源使用情况（idle, used, allocatable）
- 所有 Job 的状态和 Task 分布
- 所有 Queue 的配额和使用量
- Namespace 信息

### 8.4 Delve 调试配置

```bash
# 设置调度周期相关断点
break pkg/scheduler/scheduler.go:107           # runOnce 入口
break pkg/scheduler/scheduler.go:124           # OpenSession 调用
break pkg/scheduler/scheduler.go:132           # Action 执行
break pkg/scheduler/framework/framework.go:34  # OpenSession 定义
break pkg/scheduler/framework/framework.go:61  # CloseSession 定义
break pkg/scheduler/cache/cache.go:1424        # Snapshot 入口

# 条件断点 - 仅在特定 Action 执行时停住
condition 3 action.Name() == "allocate"

# 查看 Session 中的 Job 数量
print len(ssn.Jobs)

# 查看 Session 中的 Node 数量
print len(ssn.Nodes)
```

### 8.5 常见问题排查表

| 症状 | 可能原因 | 排查方法 |
|------|---------|---------|
| 调度周期耗时过长 | Allocate Action 中 Node 过多 | 查看 `action_duration_seconds{action="allocate"}` |
| Snapshot 耗时异常 | Job 数量过多导致 Clone 慢 | 查看 `SnapShot for scheduling` 日志中的 jobNum |
| Session 中 Job 数量为 0 | PodGroup 为 nil 或 Queue 不存在 | 检查 V(4) 日志中的 skip 原因 |
| Plugin OnSessionOpen 超时 | Plugin 实现有性能问题 | 查看 `plugin_scheduling_duration_seconds` |
| 配置热加载不生效 | 文件监听未触发 | 查看 `watch event` 日志 |

---

## 9. 总结

`runOnce()` 的完整调度周期包含四个核心阶段：

1. **配置加载** - 通过 mutex 获取当前生效的 actions/plugins/configurations 的快照
2. **OpenSession** - 创建 Cache Snapshot，初始化 HyperNode 拓扑，实例化 Plugins 并注册 Hook 函数
3. **Action 执行** - 按序执行 enqueue/allocate/backfill/reclaim/preempt/shuffle，每个 Action 都有独立耗时记录
4. **CloseSession** - Plugin 清理，JobUpdater 同步 PodGroup 状态，updateQueueStatus 更新 Queue 资源

通过合理设置 klog 级别、监控 Prometheus Metrics、使用 Cache Dumper 和 Delve 调试，可以深入分析每个阶段的行为并快速定位性能瓶颈。
