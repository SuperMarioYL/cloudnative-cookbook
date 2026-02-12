---
title: "Action 流水线"
weight: 4
---

## 概述

Action 流水线是 Volcano 调度器每个调度周期的执行引擎。在 `runOnce()` 中，调度器按配置顺序依次执行每个 Action，每个 Action 在 Session 快照上进行特定类型的调度决策。默认的 Action 执行顺序为：

```
enqueue → allocate → backfill → reclaim → preempt → shuffle
```

每个 Action 实现不同的调度策略：入队控制、资源分配、空隙填充、资源回收、抢占调度、随机重调度。

## Action 接口

> **源码参考**：`pkg/scheduler/framework/interface.go`

```go
type Action interface {
    Name() string           // Action 唯一名称
    Initialize()            // 初始化（一次性）
    Execute(ssn *Session)   // 执行调度逻辑
    UnInitialize()          // 清理
}
```

所有 Action 在 `pkg/scheduler/actions/factory.go` 中注册：

```go
func init() {
    framework.RegisterAction(enqueue.New())
    framework.RegisterAction(allocate.New())
    framework.RegisterAction(backfill.New())
    framework.RegisterAction(preempt.New())
    framework.RegisterAction(reclaim.New())
    framework.RegisterAction(shuffle.New())
}
```

---

## Action 执行流程总览

```mermaid
flowchart TB
    subgraph pipeline["Action 流水线"]
        direction TB
        enqueue["Enqueue\nPending → Inqueue\n控制哪些 Job 进入调度"]
        allocate["Allocate\n核心分配\n为 Task 分配 Node"]
        backfill["Backfill\n空隙填充\n填充 BestEffort 任务"]
        reclaim["Reclaim\n跨队列回收\n从其他队列回收资源"]
        preempt["Preempt\n队列内抢占\n高优先级抢占低优先级"]
        shuffle["Shuffle\n重新调度\n选择性驱逐进行优化"]

        enqueue --> allocate --> backfill --> reclaim --> preempt --> shuffle
    end

    style enqueue fill:#e3f2fd
    style allocate fill:#c8e6c9
    style backfill fill:#fff9c4
    style reclaim fill:#ffe0b2
    style preempt fill:#ffcdd2
    style shuffle fill:#e1bee7
```

---

## Enqueue Action

> **源码参考**：`pkg/scheduler/actions/enqueue/enqueue.go`

### 职责

控制 Job 从 `Pending` 状态转为 `Inqueue` 状态。只有 Inqueue 的 Job 才会进入后续的 Allocate 流程。Enqueue 通过检查队列资源和 Plugin 投票来决定哪些 Job 可以入队。

### 算法流程

```mermaid
flowchart TB
    start(["Enqueue 开始"]) --> build["构建 Queue 优先级队列"]
    build --> loop{"还有 Queue?"}
    loop -->|"是"| pop_q["弹出最高优先级 Queue"]
    pop_q --> build_jobs["构建该 Queue 的 Pending Job 优先级队列"]
    build_jobs --> job_loop{"还有 Pending Job?"}
    job_loop -->|"是"| pop_j["弹出最高优先级 Job"]
    pop_j --> check_min{"设置了 minResources?"}

    check_min -->|"否"| enqueue_ok["设置 Job Phase = Inqueue"]
    check_min -->|"是"| vote{"ssn.JobEnqueueable(job)"}
    vote -->|"Permit"| enqueue_ok
    vote -->|"Reject"| job_loop

    enqueue_ok --> notify["ssn.JobEnqueued(job)\n通知 Plugin"]
    notify --> job_loop

    job_loop -->|"否"| push_q["将 Queue 放回优先级队列"]
    push_q --> loop
    loop -->|"否"| done(["Enqueue 结束"])

    style start fill:#e1f5fe
    style done fill:#e8f5e9
    style enqueue_ok fill:#c8e6c9
```

### 关键机制

- **Queue 排序**：通过 `ssn.QueueOrderFn()` 确定队列优先级
- **Job 排序**：通过 `ssn.JobOrderFn()` 确定 Job 优先级
- **入队投票**：`ssn.JobEnqueueable(job)` 使用 VoteFn 模式，Plugin 可以 Permit/Reject/Abstain
- **通知机制**：`ssn.JobEnqueued(job)` 通知所有 Plugin 某个 Job 已入队（如 Proportion Plugin 用此更新队列已分配资源）

---

## Allocate Action

> **源码参考**：`pkg/scheduler/actions/allocate/allocate.go`

### 职责

Allocate 是最核心的 Action，负责为 Inqueue 状态的 Job 的 Pending Task 分配节点。它实现了完整的 Gang Scheduling 语义：要么 Job 的 minMember 个 Task 都能分配，要么一个都不分配。

### 算法流程

```mermaid
flowchart TB
    start(["Allocate 开始"]) --> ctx["构建 Allocate Context\n创建 Queue/Job/Task 优先级队列"]
    ctx --> q_loop{"还有 Queue?"}
    q_loop -->|"是"| pop_q["弹出 Queue"]
    pop_q --> overused{"ssn.Overused(queue)?"}
    overused -->|"是"| q_loop
    overused -->|"否"| j_loop{"还有 Job?"}

    j_loop -->|"是"| pop_j["弹出 Job"]
    pop_j --> topo{"有 Hard Topology?"}

    topo -->|"是"| hyper["allocateForJob\nHyperNode 感知分配"]
    topo -->|"否"| simple["allocateResourcesForTasks\n标准分配"]

    hyper --> commit{"Job Ready?"}
    simple --> commit
    commit -->|"是"| do_commit["Statement.Commit()\n提交分配"]
    commit -->|"否"| do_discard["Statement.Discard()\n回滚分配"]

    do_commit --> j_loop
    do_discard --> j_loop
    j_loop -->|"否"| q_loop
    q_loop -->|"否"| done(["Allocate 结束"])

    style start fill:#e1f5fe
    style done fill:#e8f5e9
    style do_commit fill:#c8e6c9
    style do_discard fill:#ffcdd2
```

### 核心分配循环：allocateResourcesForTasks

```mermaid
flowchart TB
    start["开始分配 Tasks"] --> t_loop{"还有 Pending Task?"}
    t_loop -->|"是"| pop_t["弹出最高优先级 Task"]
    pop_t --> alloc_check{"ssn.Allocatable\n(queue, task)?"}
    alloc_check -->|"否"| t_loop
    alloc_check -->|"是"| pre_pred["PrePredicateFn(task)\n预过滤"]
    pre_pred -->|"失败"| t_loop
    pre_pred -->|"通过"| predicate["PredicateNodes\n过滤所有节点"]
    predicate --> has_nodes{"有可用节点?"}
    has_nodes -->|"否"| cache_err["缓存 FitError"]
    cache_err --> gang_check{"Gang 模式?"}
    gang_check -->|"是"| break_loop["跳出循环\n（全有或全无）"]
    gang_check -->|"否"| t_loop

    has_nodes -->|"是"| score["prioritizeNodes\n节点打分"]
    score --> best["选择最优节点"]
    best --> do_alloc["allocateResourcesForTask\n执行分配"]
    do_alloc --> ready{"SubJobReady?"}
    ready -->|"是"| break_loop
    ready -->|"否"| t_loop

    t_loop -->|"否"| end_loop["分配结束"]
    break_loop --> end_loop

    style start fill:#e1f5fe
    style do_alloc fill:#c8e6c9
    style cache_err fill:#fff3e0
    style break_loop fill:#ffcdd2
```

### 节点选择过程

```mermaid
flowchart LR
    all["所有节点"] --> nominated{"有 NominatedNode?"}
    nominated -->|"是"| try_nom["优先尝试\nNominated 节点"]
    try_nom --> pred_nom{"Predicate 通过?"}
    pred_nom -->|"是"| use_nom["使用 Nominated 节点"]
    pred_nom -->|"否"| full_pred["全节点 Predicate"]

    nominated -->|"否"| full_pred
    full_pred --> pass_nodes["通过 Predicate 的节点列表"]
    pass_nodes --> node_order["NodeOrderFn 打分"]
    node_order --> batch_order["BatchNodeOrderFn 批量打分"]
    batch_order --> best_node["BestNodeFn\n或选择最高分"]
    best_node --> result["最优节点"]

    style result fill:#c8e6c9
    style use_nom fill:#c8e6c9
```

### HyperNode 感知分配

对于设置了 Hard Topology 的 Job，Allocate 使用 HyperNode 梯度搜索：

```mermaid
flowchart TB
    start["allocateForJob"] --> gradient["HyperNodeGradientForJobFn\n获取 HyperNode 梯度组"]
    gradient --> g_loop{"遍历梯度组"}
    g_loop -->|"下一梯度"| h_loop{"遍历 HyperNode"}
    h_loop -->|"下一 HyperNode"| clone["克隆 JobWorksheet"]
    clone --> try["在此 HyperNode 上尝试分配"]
    try --> save{"分配成功?"}
    save -->|"是"| backup["SaveOperations\n保存方案"]
    save -->|"否"| h_loop
    backup --> discard["Discard()\ndry-run 回滚"]
    discard --> h_loop

    h_loop -->|"遍历完"| select["HyperNodeOrderMapFn\n选择最优 HyperNode"]
    select --> recover["RecoverOperations\n恢复最优方案"]
    recover --> commit["Commit"]

    g_loop -->|"遍历完"| done["结束"]

    style backup fill:#fff3e0
    style recover fill:#c8e6c9
```

### 关键特性

- **Predicate 缓存**：可以缓存节点不满足条件的错误，避免重复计算
- **Statement 事务**：每个 Job 的分配在 Statement 中执行，支持原子 Commit/Discard
- **Gang Scheduling**：如果某个 Task 无法分配，Gang 模式下整个 Job 的分配回滚

---

## Backfill Action

> **源码参考**：`pkg/scheduler/actions/backfill/backfill.go`

### 职责

在 Allocate 之后填充剩余的节点空隙。主要处理 BestEffort 类型的 Task（不参与 Gang 调度的任务）。

### 算法流程

```mermaid
flowchart TB
    start(["Backfill 开始"]) --> collect["收集 BestEffort Pending Tasks\n和 Pipelined BestEffort Tasks"]
    collect --> sort["按 Queue → Job → Task 排序"]
    sort --> t_loop{"还有 Task?"}
    t_loop -->|"是"| pop["弹出 Task"]
    pop --> pre["PrePredicateFn"]
    pre -->|"失败"| t_loop
    pre -->|"通过"| pred["PredicateForAllocateAction\n过滤节点"]
    pred --> score["BatchNodeOrderFn\n节点打分"]
    score --> best["选择最优节点"]
    best --> alloc["Allocate Task"]
    alloc --> t_loop
    t_loop -->|"否"| done(["Backfill 结束"])

    style start fill:#e1f5fe
    style done fill:#e8f5e9
    style alloc fill:#c8e6c9
```

### 特点

- **轻量级**：不涉及 Gang 语义，逐个 Task 独立分配
- **填充角色**：利用 Gang 调度后的剩余资源
- **低优先级**：只处理 BestEffort 任务

---

## Reclaim Action

> **源码参考**：`pkg/scheduler/actions/reclaim/reclaim.go`

### 职责

跨队列资源回收。当某个队列的 Job 处于饥饿状态时，从其他使用了超额资源的队列中回收资源。

### 算法流程

```mermaid
flowchart TB
    start(["Reclaim 开始"]) --> find["找到饥饿的 Job\nssn.JobStarving(job)"]
    find --> q_loop{"遍历队列"}
    q_loop -->|"下一队列"| j_loop{"遍历饥饿 Job"}
    j_loop -->|"下一 Job"| t_loop{"遍历 Pending Task"}

    t_loop -->|"下一 Task"| check_preempt{"PreemptionPolicy\n== PreemptNever?"}
    check_preempt -->|"是"| t_loop
    check_preempt -->|"否"| check_preemptive{"ssn.Preemptive\n(queue, task)?"}
    check_preemptive -->|"否"| t_loop
    check_preemptive -->|"是"| pred["PredicateForPreemptAction\n过滤节点"]
    pred --> n_loop{"遍历可用节点"}

    n_loop -->|"下一节点"| find_victims["找到该节点上\n可回收的 Task"]
    find_victims --> reclaimable["ssn.Reclaimable()\n交集投票"]
    reclaimable --> evict["按优先级驱逐受害者"]
    evict --> try_alloc{"尝试分配 Task"}
    try_alloc -->|"成功"| ready{"Job Ready?"}
    ready -->|"是"| commit["Commit"]
    ready -->|"否"| discard["Discard"]
    try_alloc -->|"失败"| n_loop

    n_loop -->|"遍历完"| t_loop
    t_loop -->|"遍历完"| j_loop
    j_loop -->|"遍历完"| q_loop
    q_loop -->|"遍历完"| done(["Reclaim 结束"])

    style start fill:#e1f5fe
    style done fill:#e8f5e9
    style commit fill:#c8e6c9
    style discard fill:#ffcdd2
```

### 受害者选择条件

一个 Task 可以被回收必须同时满足：
1. 状态为 `Running` 且标记为 `Preemptable`
2. 属于**不同队列**且该队列是 `Reclaimable`，或属于**同一队列**但优先级更低
3. 通过 `ssn.Reclaimable()` 交集投票

---

## Preempt Action

> **源码参考**：`pkg/scheduler/actions/preempt/preempt.go`

### 职责

队列内抢占。高优先级的饥饿 Job 可以抢占同队列中低优先级 Job 的 Task。

### 算法流程

```mermaid
flowchart TB
    start(["Preempt 开始"]) --> find["找到饥饿的 Job\n收集其 Pending Tasks"]
    find --> q_loop{"遍历队列"}
    q_loop -->|"下一队列"| pop_preemptor["弹出 Preemptor Job"]
    pop_preemptor --> starving{"Job 仍然饥饿?"}
    starving -->|"否"| q_loop
    starving -->|"是"| pop_task["弹出 Preemptor Task"]
    pop_task --> preempt_fn["preempt(ssn, stmt, task)"]

    subgraph preempt_detail["preempt 子过程"]
        direction TB
        find_nodes["找候选节点\n（有可抢占 Task 的节点）"]
        find_nodes --> for_node{"遍历候选节点"}
        for_node --> get_victims["ssn.Preemptable()\n获取受害者"]
        get_victims --> filter["过滤 PreemptNever\n和 Gang 约束"]
        filter --> evict_v["按优先级驱逐受害者"]
        evict_v --> try["尝试分配 Preemptor"]
        try -->|"成功"| success["返回 true"]
        try -->|"失败"| for_node
    end

    preempt_fn --> preempt_detail
    preempt_detail --> assigned{"分配成功?"}
    assigned -->|"是"| job_ready{"Job Ready?"}
    job_ready -->|"是"| commit["Commit"]
    job_ready -->|"否"| discard["Discard"]
    assigned -->|"否"| pop_task

    commit --> q_loop
    discard --> q_loop

    style start fill:#e1f5fe
    style commit fill:#c8e6c9
    style discard fill:#ffcdd2
    style success fill:#c8e6c9
```

### 拓扑感知抢占

Preempt 支持拓扑感知模式，通过配置启用：

```yaml
configurations:
  - name: preempt
    arguments:
      enableTopologyAwarePreemption: true
      topologyAwarePreemptWorkerNum: 16
      minCandidateNodesPercentage: 10
      maxCandidateNodesAbsolute: 100
```

拓扑感知抢占使用 Worker Pool 并行评估候选节点，减少抢占决策延迟。

---

## Shuffle Action

> **源码参考**：`pkg/scheduler/actions/shuffle/shuffle.go`

### 职责

Shuffle 是最简单的 Action，通过 Plugin 选择性驱逐部分运行中的 Task，使其重新调度到更优的节点。

### 算法流程

```mermaid
flowchart TB
    start(["Shuffle 开始"]) --> collect["收集所有 Running Tasks"]
    collect --> vote["ssn.VictimTasks(allTasks)\nPlugin 投票选择受害者"]
    vote --> loop{"还有受害者?"}
    loop -->|"是"| evict["ssn.Evict(victim, 'shuffle')"]
    evict --> loop
    loop -->|"否"| done(["Shuffle 结束"])

    style start fill:#e1f5fe
    style done fill:#e8f5e9
    style evict fill:#ffe0b2
```

### 使用场景

- **负载均衡**：将 Task 从负载高的节点迁移到负载低的节点
- **节点整合**：将分散的 Task 聚集到少数节点，释放空闲节点
- **NUMA 优化**：将 Task 迁移到 NUMA 拓扑更优的节点
- **自定义策略**：通过 Plugin 实现任意驱逐策略

---

## Action 间的协作

```mermaid
sequenceDiagram
    participant E as "Enqueue"
    participant A as "Allocate"
    participant B as "Backfill"
    participant R as "Reclaim"
    participant P as "Preempt"
    participant S as "Shuffle"

    Note over E: Pending Job → Inqueue
    E->>A: Inqueue Jobs 进入分配

    Note over A: 核心分配<br/>Gang Scheduling
    A->>B: 剩余资源空隙

    Note over B: 填充 BestEffort Tasks
    B->>R: 仍有饥饿 Job

    Note over R: 跨队列回收资源
    R->>P: 队列内仍有饥饿 Job

    Note over P: 队列内高优抢占低优
    P->>S: 可能需要重新平衡

    Note over S: 选择性驱逐优化布局
```

### 资源流动模型

```mermaid
flowchart LR
    subgraph sources["资源来源"]
        idle["空闲资源\n（Allocate/Backfill 使用）"]
        other_q["其他队列超额资源\n（Reclaim 回收）"]
        low_pri["低优先级 Task\n（Preempt 抢占）"]
        shuffle_t["非最优 Task\n（Shuffle 驱逐）"]
    end

    subgraph targets["资源去向"]
        gang["Gang Job 的 minMember Tasks"]
        best["BestEffort Tasks"]
        starving["饥饿 Job 的 Tasks"]
    end

    idle --> gang
    idle --> best
    other_q --> starving
    low_pri --> starving
    shuffle_t -->|"重新调度"| gang

    style sources fill:#e3f2fd
    style targets fill:#e8f5e9
```

---

## 配置与定制

### Action 执行顺序

通过调度器配置文件指定 Action 执行顺序：

```yaml
actions: "enqueue, allocate, backfill, reclaim, preempt, shuffle"
```

可以根据场景调整顺序或移除不需要的 Action：

| 场景 | 推荐配置 |
|------|---------|
| 基础调度 | `enqueue, allocate, backfill` |
| 多队列公平共享 | `enqueue, allocate, backfill, reclaim` |
| 完整功能 | `enqueue, allocate, backfill, reclaim, preempt, shuffle` |
| 高优先级调度 | `enqueue, allocate, preempt, backfill` |

### 注册自定义 Action

```go
// 实现 Action 接口
type myAction struct{}
func (a *myAction) Name() string         { return "myaction" }
func (a *myAction) Initialize()          { }
func (a *myAction) Execute(ssn *Session) { /* 调度逻辑 */ }
func (a *myAction) UnInitialize()        { }

// 注册
framework.RegisterAction(&myAction{})
```

在配置中启用：
```yaml
actions: "enqueue, allocate, myaction, backfill"
```

---

## 常见问题

### Q: Action 的执行顺序重要吗？

非常重要。Enqueue 必须在 Allocate 之前（否则没有 Inqueue 的 Job）。Allocate 通常在 Reclaim/Preempt 之前（先尝试用空闲资源，不够再回收/抢占）。Shuffle 通常最后执行（在其他分配完成后优化布局）。

### Q: Allocate 失败的 Job 会重试吗？

当前周期失败的 Job 不会立即重试。它会保持 Inqueue 状态，在下一个调度周期重新尝试分配。如果连续多个周期都失败，PodGroup 的 Condition 会记录 `Unschedulable` 原因。

### Q: Reclaim 和 Preempt 的区别是什么？

- **Reclaim**：跨队列回收。队列 A 的 Job 从队列 B 回收超额使用的资源
- **Preempt**：队列内抢占。同一队列内高优先级 Job 抢占低优先级 Job 的资源

---

## 下一步

- [Plugin 系统](./05-plugin-system.md) -- Plugin 如何为 Action 提供决策支持
- [Statement 与绑定](./06-statement-and-binding.md) -- Action 中的事务机制
- [资源模型](./07-resource-model.md) -- 资源的多维度表示与计算
