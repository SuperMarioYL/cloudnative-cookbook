---
title: "Allocate Action 详解"
weight: 2
---

## 概述

Allocate 是 Volcano 调度器最核心、最复杂的 Action，负责为 Inqueue 状态 Job 的 Pending Task 分配节点。它实现了完整的 Gang Scheduling 语义、HyperNode 拓扑感知分配、节点梯度优选以及 Statement 事务控制。

> **源码参考**：`pkg/scheduler/actions/allocate/allocate.go`

## 核心结构

### Action 结构体

```go
type Action struct {
    session                   *framework.Session
    enablePredicateErrorCache bool      // 启用 Predicate 错误缓存
    recorder                  *Recorder // HyperNode 决策记录器
}
```

### 配置参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `enablePredicateErrorCache` | `true` | 缓存 Predicate 失败结果，避免重复计算 |

---

## 整体执行流程

```mermaid
flowchart TB
    start(["Execute() 开始"]) --> ctx["构建 AllocateContext\n创建 Queue/Job/Task 优先级队列\n分离 HardTopology/SoftTopology Job"]
    ctx --> q_loop{"Queue 队列非空?"}

    q_loop -->|"是"| pop_q["弹出最高优先级 Queue"]
    pop_q --> overused{"ssn.Overused(queue)?"}
    overused -->|"是"| q_loop
    overused -->|"否"| j_loop{"该 Queue 有 Job?"}

    j_loop -->|"是"| pop_j["弹出最高优先级 Job"]
    pop_j --> stmt["创建 Statement"]
    stmt --> topo{"有 HardTopology?"}

    topo -->|"是"| hard["allocateForJob()\nHyperNode 感知分配"]
    topo -->|"否"| soft["allocateResourcesForTasks()\n标准分配"]

    hard --> ready{"Job Ready\n或 Pipelined?"}
    soft --> ready
    ready -->|"是"| commit["Statement.Commit()\n提交分配"]
    ready -->|"否"| discard["Statement.Discard()\n回滚"]

    commit --> record["更新 HyperNode 决策\n记录 Metrics"]
    discard --> j_loop
    record --> push_q["Queue 放回队列"]
    push_q --> q_loop

    j_loop -->|"否"| q_loop
    q_loop -->|"否"| done(["Execute() 结束"])

    style start fill:#e1f5fe
    style done fill:#e8f5e9
    style commit fill:#c8e6c9
    style discard fill:#ffcdd2
```

---

## 核心分配循环

### allocateResourcesForTasks

这是 Allocate 最核心的函数，实现了单个 SubJob 内 Task 的逐个分配。

```mermaid
flowchart TB
    start["开始分配 Tasks"] --> t_loop{"有 Pending Task?"}
    t_loop -->|"是"| pop["弹出最高优先级 Task"]
    pop --> allocatable{"ssn.Allocatable\n(queue, task)?"}
    allocatable -->|"否"| t_loop

    allocatable -->|"是"| gated{"task.SchGated?"}
    gated -->|"是"| t_loop
    gated -->|"否"| pre_pred["ssn.PrePredicateFn(task)"]
    pre_pred -->|"失败"| t_loop

    pre_pred -->|"通过"| nominated{"有 NominatedNode?"}
    nominated -->|"是"| try_nom["优先尝试 NominatedNode"]
    try_nom --> nom_ok{"Predicate 通过?"}
    nom_ok -->|"是"| score["节点打分"]
    nom_ok -->|"否"| full["全节点 Predicate"]

    nominated -->|"否"| full
    full --> has_nodes{"有可用节点?"}
    has_nodes -->|"否"| fit_err["记录 FitError"]
    fit_err --> gang{"需要继续分配?"}
    gang -->|"否（Gang 模式）"| break_loop["break"]
    gang -->|"是"| t_loop

    has_nodes -->|"是"| score
    score --> best["选择最优节点"]
    best --> do_alloc["allocateResourcesForTask()"]
    do_alloc --> sub_ready{"SubJobReady?"}
    sub_ready -->|"是"| break_loop
    sub_ready -->|"否"| t_loop

    t_loop -->|"否"| end_loop["分配结束"]
    break_loop --> end_loop

    style start fill:#e1f5fe
    style do_alloc fill:#c8e6c9
    style fit_err fill:#fff3e0
    style break_loop fill:#ffcdd2
```

### 节点选择：双梯度优选

Allocate 将 Predicate 通过的节点分为两个梯度：

```mermaid
flowchart TB
    nodes["Predicate 通过的节点"] --> classify{"分类"}
    classify --> idle["第一梯度\nIdle 资源充足"]
    classify --> future["第二梯度\nFutureIdle 资源充足"]
    classify --> neither["不满足\n（跳过）"]

    idle --> score1["优先打分\nBatchNodeOrderFn"]
    score1 --> best1{"找到最优节点?"}
    best1 -->|"是"| result["返回最优节点"]
    best1 -->|"否"| future

    future --> score2["次优打分\nBatchNodeOrderFn"]
    score2 --> best2{"找到最优节点?"}
    best2 -->|"是"| result
    best2 -->|"否"| fail["无可用节点"]

    style idle fill:#c8e6c9
    style future fill:#fff9c4
    style result fill:#c8e6c9
```

**分片模式下**进一步细分为 4 个梯度：
1. Idle 资源充足 + 本分片节点
2. Idle 资源充足 + 其他分片节点
3. FutureIdle 资源充足 + 本分片节点
4. FutureIdle 资源充足 + 其他分片节点

### 资源分配决策

```go
func allocateResourcesForTask(stmt, task, node, job) error {
    if task.InitResreq.LessEqual(node.Idle, Zero) {
        // 空闲资源充足 → 直接 Allocate
        stmt.Allocate(task, node)
    } else if task.InitResreq.LessEqual(node.FutureIdle(), Zero) {
        // 未来空闲资源充足 → Pipeline（等待释放）
        stmt.Pipeline(task, node.Name, false)
    }
}
```

| 资源状态 | 操作 | Task 状态 |
|---------|------|----------|
| `Idle >= request` | `Allocate` | Allocated |
| `FutureIdle >= request` | `Pipeline` | Pipelined |
| 两者都不足 | 跳过此节点 | 不变 |

---

## HyperNode 拓扑感知分配

### Hard Topology 模式

对于设置了 Hard Topology 的 Job，Allocate 使用 HyperNode 梯度搜索：

```mermaid
flowchart TB
    start["allocateForJob()"] --> gradient["HyperNodeGradientForJobFn\n获取 HyperNode 梯度组"]
    gradient --> g_loop{"遍历梯度组\n（从低 Tier 到高 Tier）"}

    g_loop -->|"下一梯度"| h_loop{"遍历该梯度的 HyperNode"}
    h_loop -->|"下一 HyperNode"| snapshot["SnapshotSubJobStatus()\n快照 SubJob 状态"]
    snapshot --> clone["克隆 JobWorksheet"]
    clone --> try_sub["为每个 SubJob\nallocateForSubJob()"]

    try_sub --> job_ready{"Job Ready\n或 Pipelined?"}
    job_ready -->|"是"| save["SaveOperations()\n保存方案"]
    job_ready -->|"否"| discard["Discard()"]

    save --> discard_dry["Discard()\ndry-run 回滚"]
    discard --> recover_snap["RecoverSubJobStatus()\n恢复 SubJob 状态"]
    discard_dry --> recover_snap
    recover_snap --> h_loop

    h_loop -->|"遍历完"| has_solution{"有成功方案?"}
    has_solution -->|"是"| select_best["HyperNodeOrderMapFn\n选择最优 HyperNode"]
    has_solution -->|"否"| g_loop

    select_best --> recover["RecoverOperations()\n恢复最优方案"]
    recover --> done["返回 Statement"]

    g_loop -->|"遍历完"| no_solution["返回 nil\n（分配失败）"]

    style save fill:#fff3e0
    style recover fill:#c8e6c9
    style no_solution fill:#ffcdd2
```

### SubJob 分配

每个 SubJob 独立进行 HyperNode 选择：

```mermaid
flowchart TB
    start["allocateForSubJob()"] --> gradient["HyperNodeGradientForSubJobFn\n获取 SubJob 梯度"]
    gradient --> g_loop{"遍历梯度"}
    g_loop --> h_loop{"遍历 HyperNode"}
    h_loop --> get_nodes["获取 HyperNode 下的节点"]
    get_nodes --> alloc["allocateResourcesForTasks()\n在这些节点上分配"]
    alloc --> sub_ready{"SubJob Ready\n或 Pipelined?"}
    sub_ready -->|"是"| save["SaveOperations\n保存方案"]
    save --> discard["Discard\n回滚继续尝试"]
    sub_ready -->|"否"| discard2["Discard"]
    discard --> h_loop
    discard2 --> h_loop
    h_loop -->|"完"| select["选择最优 HyperNode\n恢复方案"]
    select --> done["返回 Statement"]

    style save fill:#fff3e0
    style select fill:#c8e6c9
```

### Recorder 决策记录

> **源码参考**：`pkg/scheduler/actions/allocate/recorder.go`

`Recorder` 记录每个 Job/SubJob 选择的 HyperNode，用于在 Commit 后更新 Job 的 `AllocatedHyperNode` 字段。

```go
type Recorder struct {
    jobDecisions    map[api.JobID]string                          // Job → 选择的 HyperNode
    subJobDecisions map[api.JobID]map[string]map[api.SubJobID]string // Job → HyperNode → SubJob 决策
    subJobStatusSnapshot map[api.JobID]map[api.SubJobID]*SubJobStatus
}
```

---

## Gang Scheduling 实现

Allocate 中的 Gang Scheduling 通过 Statement 事务实现：

```mermaid
sequenceDiagram
    participant A as "Allocate"
    participant St as "Statement"
    participant S as "Session"
    participant G as "Gang Plugin"

    A->>St: NewStatement(ssn)

    loop 每个 Pending Task
        A->>S: Allocatable(queue, task)
        A->>S: PredicateNodes(task, nodes)
        A->>S: PrioritizeNodes(task, nodes)
        A->>St: Allocate(task, bestNode)
    end

    A->>S: JobReady(job)
    S->>G: gang.JobReadyFn(job)
    G-->>S: allocated >= minMember?

    alt Job Ready
        A->>St: Commit()
        Note over St: 所有 Task 进入绑定流水线
    else Job Not Ready
        A->>St: Discard()
        Note over St: 回滚所有分配
    end
```

### NeedContinueAllocating

当某个 Task 无法找到可用节点时，需要决定是否继续尝试其他 Task：

```go
func (job *JobInfo) NeedContinueAllocating(subJobUID SubJobID) bool
```

- **Gang 模式（minMember = totalTasks）**：返回 `false`，立即停止（一个失败全部失败）
- **弹性 Gang（minMember < totalTasks）**：返回 `true`，继续尝试其他 Task
- **无 Gang（minMember = 1）**：返回 `true`，逐个独立分配

---

## Predicate 与错误缓存

### Predicate 流程

```go
func (alloc *Action) predicate(task *api.TaskInfo, node *api.NodeInfo) error {
    // 1. 资源检查：FutureIdle >= request
    if ok, resources := task.InitResreq.LessEqualWithResourcesName(node.FutureIdle(), api.Zero); !ok {
        return NewFitErr("InsufficientResources", resources)
    }
    // 2. K8s Predicate：亲和性、污点、端口等
    return ssn.PredicateForAllocateAction(task, node)
}
```

### 错误缓存机制

启用 `enablePredicateErrorCache` 后：

```mermaid
flowchart LR
    task["Task-1"] --> check{"节点 A 缓存\n有 FitError?"}
    check -->|"有"| skip["跳过节点 A"]
    check -->|"无"| pred["执行 Predicate"]
    pred -->|"失败"| cache["缓存 FitError"]
    pred -->|"成功"| use["使用节点 A"]

    task2["Task-2\n（同 Job）"] --> check2{"节点 A 缓存?"}
    check2 -->|"有"| skip2["直接跳过\n（复用缓存）"]

    style cache fill:#fff3e0
    style skip fill:#ffcdd2
    style skip2 fill:#ffcdd2
```

同一个 Job 内相似的 Task 可以复用之前 Task 的 Predicate 失败结果，大幅减少重复计算。

---

## 调用的扩展点

| 扩展点 | 用途 |
|--------|------|
| `QueueOrderFn` | Queue 排序 |
| `JobOrderFn` | Job 排序 |
| `TaskOrderFn` | Task 排序 |
| `Overused` | 判断 Queue 是否超用 |
| `Allocatable` | 判断 Queue 是否可分配 |
| `JobValid` | 验证 Job 有效性 |
| `PrePredicateFn` | Task 预过滤 |
| `PredicateForAllocateAction` | 节点 Predicate |
| `BatchNodeOrderFn` | 批量节点打分 |
| `NodeOrderMapFn` / `NodeOrderReduceFn` | 节点映射/聚合打分 |
| `BestNodeFn` | 最优节点选择 |
| `JobReady` | Job 就绪检查（Gang） |
| `JobPipelined` | Job Pipeline 检查 |
| `SubJobReady` / `SubJobPipelined` | SubJob 就绪检查 |
| `HyperNodeGradientForJobFn` | HyperNode 梯度（Job 级） |
| `HyperNodeGradientForSubJobFn` | HyperNode 梯度（SubJob 级） |
| `HyperNodeOrderMapFn` | HyperNode 打分 |

---

## 常见问题

### Q: Queue 被标记为 Overused 意味着什么？

Queue 的已分配资源超过了 Deserved 资源。Allocate 会跳过 Overused 的 Queue，不再为其中的 Job 分配新资源。这确保了队列间的公平共享。

### Q: NominatedNode 是什么？

NominatedNode 是上一次 Preempt Action 为 Task 推荐的节点。Allocate 会优先尝试 NominatedNode，如果它的 FutureIdle 资源满足需求且 Predicate 通过，就直接使用，避免重新搜索所有节点。

### Q: Hard Topology 和 Soft Topology 的区别？

- **Hard Topology**：Job 必须分配到满足拓扑约束的 HyperNode，否则分配失败
- **Soft Topology**：优先选择拓扑最优的 HyperNode，但不满足时可以降级到更高 Tier

---

## 下一步

- [Backfill Action](./03-backfill-action.md) -- Allocate 后的空隙填充
- [Preempt Action](./04-preempt-action.md) -- 队列内优先级抢占
- [Statement 与绑定](../02-scheduler-deep-dive/06-statement-and-binding.md) -- Commit/Discard 事务机制
