---
title: "抢占流程调试指南"
weight: 4
---

## 1. 概述

Volcano 调度器中的资源抢占机制包含两个核心 Action - **Preempt** 和 **Reclaim**，它们虽然都涉及驱逐已运行的 Pod 来释放资源，但触发场景和适用范围有本质区别。

### 1.1 Preempt vs Reclaim 对比

| 维度 | Preempt | Reclaim |
|------|---------|---------|
| **作用范围** | 队列内部 (intra-queue) | 跨队列 (inter-queue) |
| **触发条件** | 高优先级 Job 抢占低优先级 Job | 资源不足的队列从超用队列回收资源 |
| **目标** | 同一队列内的低优先级 Task | 其他队列中超出 deserved 配额的 Task |
| **源码位置** | `pkg/scheduler/actions/preempt/preempt.go` | `pkg/scheduler/actions/reclaim/reclaim.go` |
| **Plugin 钩子** | `PreemptableFn` | `ReclaimableFn` |
| **配额检查** | `Allocatable` (队列内配额) | `Overused` + `Preemptive` |

```mermaid
graph LR
    subgraph preempt_scope["Preempt - 队列内抢占"]
        A["高优先级 Job A"] -->|"抢占"| B["低优先级 Job B"]
        A -->|"同一队列"| B
    end
    subgraph reclaim_scope["Reclaim - 跨队列回收"]
        C["Queue-1 饥饿 Job"] -->|"回收资源"| D["Queue-2 超用 Job"]
        C -->|"跨队列"| D
    end
```

### 1.2 关键数据结构

在调试抢占流程前，需要理解以下核心概念：

- **Preemptor** - 发起抢占的 Task（Pending 状态，需要资源）
- **Preemptee / Victim** - 被抢占的 Task（Running 状态，将被驱逐）
- **Statement** - 事务对象，记录一系列操作（Evict、Pipeline），支持 Commit 和 Discard
- **Starving** - Job 处于饥饿状态，即 `WaitingTaskNum + ReadyTaskNum < MinAvailable`

---

## 2. Preempt Action 完整流程

### 2.1 Execute 方法入口

> 源码: `pkg/scheduler/actions/preempt/preempt.go` 第 103-280 行

Preempt Action 的 `Execute()` 方法是整个抢占流程的入口。它分为两个阶段：

1. **Phase 1** - 队列间 Job 抢占（同一队列内，高优先级 Job 抢占低优先级 Job）
2. **Phase 2** - Job 内部 Task 抢占（同一 Job 内，高优先级 Task 抢占低优先级 Task）

```mermaid
flowchart TD
    START["Execute() 入口"] --> PARSE["parseArguments() 解析配置"]
    PARSE --> SCAN["遍历 ssn.Jobs 构建 preemptorsMap"]
    SCAN --> CHECK_PENDING{"job.IsPending()?"}
    CHECK_PENDING -->|"是"| SKIP1["跳过"]
    CHECK_PENDING -->|"否"| CHECK_VALID{"ssn.JobValid(job)?"}
    CHECK_VALID -->|"不通过"| SKIP2["跳过"]
    CHECK_VALID -->|"通过"| CHECK_STARVING{"ssn.JobStarving(job)?"}
    CHECK_STARVING -->|"不饥饿"| SKIP3["跳过"]
    CHECK_STARVING -->|"饥饿"| CHECK_NET{"ContainsNetworkTopology()?"}
    CHECK_NET -->|"是"| SKIP4["跳过 - 不支持网络拓扑抢占"]
    CHECK_NET -->|"否"| ADD_PREEMPTOR["加入 preemptorsMap"]

    ADD_PREEMPTOR --> PHASE1["Phase 1 - 队列间 Job 抢占"]
    PHASE1 --> PHASE1_LOOP["遍历 queues"]
    PHASE1_LOOP --> POP_JOB["Pop 最高优先级 preemptorJob"]
    POP_JOB --> STMT_NEW["创建 Statement"]
    STMT_NEW --> INNER_LOOP["内层循环 - 逐 Task 抢占"]
    INNER_LOOP --> CHECK_STILL_STARVING{"JobStarving?"}
    CHECK_STILL_STARVING -->|"否"| CHECK_PIPELINED
    CHECK_STILL_STARVING -->|"是"| POP_TASK["Pop preemptor Task"]
    POP_TASK --> CALL_PREEMPT["调用 pmpt.preempt()"]
    CALL_PREEMPT --> INNER_LOOP

    CHECK_PIPELINED{"ssn.JobPipelined?"}
    CHECK_PIPELINED -->|"是"| COMMIT["stmt.Commit()"]
    CHECK_PIPELINED -->|"否"| DISCARD["stmt.Discard()"]

    COMMIT --> PHASE2["Phase 2 - Job 内 Task 抢占"]
    DISCARD --> PHASE1_LOOP

    PHASE2 --> PHASE2_LOOP["遍历 underRequest jobs"]
    PHASE2_LOOP --> TASK_PREEMPT["Task 间抢占"]
    TASK_PREEMPT --> END["结束"]
```

### 2.2 关键代码片段

**Phase 1 核心逻辑 - 队列间 Job 抢占：**

```go
// pkg/scheduler/actions/preempt/preempt.go 第 161-227 行
for _, queue := range queues {
    for {
        preemptors := preemptorsMap[queue.UID]
        if preemptors == nil || preemptors.Empty() {
            break
        }
        preemptorJob := preemptors.Pop().(*api.JobInfo)
        stmt := framework.NewStatement(ssn)

        for {
            if !ssn.JobStarving(preemptorJob) {
                break  // Job 不再饥饿，停止抢占
            }
            preemptor := preemptorTasks[preemptorJob.UID].Pop().(*api.TaskInfo)
            assigned, err = pmpt.preempt(ssn, stmt, preemptor, filter, ph)
        }

        // 关键决策点：Commit 还是 Discard
        if ssn.JobPipelined(preemptorJob) {
            stmt.Commit()    // Job 已获得足够资源，提交操作
        } else {
            stmt.Discard()   // Job 未获得足够资源，回滚所有操作
            continue
        }
    }
}
```

**Phase 1 的过滤函数 - 只抢占同队列内的其他 Job 的 Task：**

```go
// pkg/scheduler/actions/preempt/preempt.go 第 191-209 行
filter := func(task *api.TaskInfo) bool {
    if !api.PreemptableStatus(task.Status) { return false }      // 只抢占运行中的 Task
    if preemptor.BestEffort && !task.BestEffort { return false }  // BestEffort 不能抢占非 BestEffort
    if !task.Preemptable { return false }                         // 跳过不可抢占的 Pod
    job, found := ssn.Jobs[task.Job]
    if !found { return false }
    return job.Queue == preemptorJob.Queue && preemptor.Job != task.Job  // 同队列、不同 Job
}
```

**Phase 2 的过滤函数 - 只抢占同 Job 内的 Task：**

```go
// pkg/scheduler/actions/preempt/preempt.go 第 251-267 行
filter := func(task *api.TaskInfo) bool {
    if !api.PreemptableStatus(task.Status) { return false }
    if preemptor.BestEffort && !task.BestEffort { return false }
    if !task.Preemptable { return false }
    return preemptor.Job == task.Job  // 同 Job
}
```

### 2.3 调试断点建议

| 断点位置 | 文件与行号 | 调试目的 |
|----------|-----------|---------|
| Job 饥饿检测 | `preempt.go:134` | 确认 Job 是否被识别为饥饿 |
| preempt 方法入口 | `preempt.go:284` | 追踪单个 Task 的抢占过程 |
| Commit/Discard 决策 | `preempt.go:216-221` | 确认 Pipeline 判断结果 |
| normalPreempt 节点选择 | `preempt.go:340` | 追踪节点遍历和 Victim 选择 |

---

## 3. 抢占触发条件检测

### 3.1 JobStarvingFns - 判断 Job 是否饥饿

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 207-212 行

Gang Plugin 注册了 `jobStarvingFn`，用于判断 Job 是否处于饥饿状态。饥饿的定义是：当前已就绪和等待中的 Task 数量不足以满足 MinAvailable 要求。

```go
// pkg/scheduler/plugins/gang/gang.go 第 207-212 行
jobStarvingFn := func(obj interface{}) bool {
    ji := obj.(*api.JobInfo)
    return ji.IsStarving()
}

// pkg/scheduler/api/job_info.go 第 1180-1182 行
func (ji *JobInfo) IsStarving() bool {
    return ji.WaitingTaskNum()+ji.ReadyTaskNum() < ji.MinAvailable
}
```

```mermaid
flowchart TD
    START["JobStarving(job) 调用"] --> TIER["遍历 Tiers"]
    TIER --> PLUGIN["查找 EnabledJobStarving 的 Plugin"]
    PLUGIN --> GANG["Gang Plugin - jobStarvingFn"]
    GANG --> IS_STARVING{"IsStarving()"}
    IS_STARVING --> CALC["WaitingTaskNum + ReadyTaskNum < MinAvailable"]
    CALC -->|"条件成立"| TRUE["返回 true - Job 饥饿"]
    CALC -->|"条件不成立"| FALSE["返回 false - Job 不饥饿"]

    subgraph waiting_calc["WaitingTaskNum 计算"]
        W1["Pipelined 状态的 Task 数量"]
    end
    subgraph ready_calc["ReadyTaskNum 计算"]
        R1["Bound + Binding + Running + Allocated 状态的 Task 数量"]
    end
```

### 3.2 Session.JobStarving 的 Tier 遍历机制

> 源码: `pkg/scheduler/framework/session_plugins.go` 第 483-507 行

```go
func (ssn *Session) JobStarving(obj interface{}) bool {
    var hasFound bool
    for _, tier := range ssn.Tiers {
        for _, plugin := range tier.Plugins {
            if !isEnabled(plugin.EnabledJobStarving) { continue }
            jrf, found := ssn.jobStarvingFns[plugin.Name]
            if !found { continue }
            hasFound = true
            if !jrf(obj) {
                return false  // 任何一个 plugin 说不饥饿，就不饥饿
            }
        }
        if hasFound {
            return true  // 当前 tier 有 plugin 且全部说饥饿，返回 true
        }
    }
    return false  // 没有任何 plugin 注册，默认不饥饿
}
```

### 3.3 PreemptiveFn - 判断 Queue 是否有权回收资源

> 源码: `pkg/scheduler/plugins/proportion/proportion.go` 第 357-361 行

在 Reclaim Action 中，还需要通过 `Preemptive` 检查确认当前队列是否有权执行回收操作。Proportion Plugin 的实现实际上检查的是队列资源分配是否未超过 deserved 配额。

```go
// proportion.go 第 357-361 行
ssn.AddPreemptiveFn(pp.Name(), func(obj interface{}, candidate interface{}) bool {
    queue := obj.(*api.QueueInfo)
    task := candidate.(*api.TaskInfo)
    return queueAllocatable(queue, task)  // 检查 allocated + task.Resreq <= deserved
})
```

### 3.4 调试技巧 - 追踪饥饿判断

设置日志级别 `-v=4` 可以观察到以下关键日志：

```bash
# 确认 Job 是否被判断为饥饿
klog.V(4).Infof("Job <%s/%s> Queue <%s> skip preemption, reason: ...")  # 被跳过的 Job
klog.V(3).Infof("No preemptors in Queue <%s>, break.", queue.Name)       # 没有饥饿 Job 的队列
```

**断点调试方法：**

```
// 在 Delve 中设置断点
dlv debug volcano.sh/volcano/cmd/scheduler
(dlv) break pkg/scheduler/api/job_info.go:1180   // IsStarving
(dlv) break pkg/scheduler/actions/preempt/preempt.go:134  // JobStarving 检查点
(dlv) condition 2 job.Name == "target-job"   // 条件断点
```

---

## 4. Victim 选择流程

### 4.1 整体 Victim 选择架构

```mermaid
flowchart TD
    subgraph preempt_method["pmpt.preempt() 方法"]
        A["检查 Task 是否有权抢占"] --> B["PrePredicateFn 预过滤"]
        B --> C["过滤 Unschedulable 节点"]
        C --> D["PredicateNodes 节点筛选"]
        D --> E{"enableTopologyAwarePreemption?"}
        E -->|"是"| F["topologyAwarePreempt()"]
        E -->|"否"| G["normalPreempt()"]
    end

    subgraph normal_preempt["normalPreempt() 方法"]
        G --> H["PrioritizeNodes 节点评分"]
        H --> I["SortNodes 节点排序"]
        I --> J["遍历节点"]
        J --> K["收集节点上的 preemptees"]
        K --> L["ssn.Preemptable() 筛选 Victims"]
        L --> M["BuildVictimsPriorityQueue 构建优先队列"]
        M --> N["逐个 Evict Victim"]
        N --> O{"资源满足?"}
        O -->|"是"| P["Pipeline preemptor 到节点"]
        O -->|"否"| Q["尝试下一个节点"]
    end
```

### 4.2 PreemptableFn 调用 - Tiered Intersection 模式

> 源码: `pkg/scheduler/framework/session_plugins.go` 第 261-308 行

`Session.Preemptable()` 方法使用了 **Tiered Intersection** 模式来聚合多个 Plugin 的 Victim 选择结果：

```go
func (ssn *Session) Preemptable(preemptor *api.TaskInfo, preemptees []*api.TaskInfo) []*api.TaskInfo {
    var victims []*api.TaskInfo
    for _, tier := range ssn.Tiers {
        for _, plugin := range tier.Plugins {
            candidates, abstain := pf(preemptor, preemptees)
            if abstain == 0 { continue }          // Plugin 弃权
            if len(candidates) == 0 {
                victims = nil; break              // Plugin 明确拒绝所有候选
            }
            if victims == nil {
                victims = candidates              // 第一个投票的 plugin，初始化 victims
            } else {
                // 取交集
                intersection := intersect(victims, candidates)
                victims = intersection
            }
        }
        if victims != nil { return victims }      // 当前 tier 有结果，立即返回
    }
    return victims
}
```

```mermaid
flowchart TD
    START["Preemptable(preemptor, preemptees)"] --> TIER1["Tier 1"]

    subgraph tier1_process["Tier 1 处理"]
        TIER1 --> P1["Plugin A - preemptableFn"]
        P1 --> P1_RESULT{"返回结果?"}
        P1_RESULT -->|"弃权 (abstain=0)"| P2["Plugin B - preemptableFn"]
        P1_RESULT -->|"candidates=空"| TIER1_REJECT["victims=nil, break"]
        P1_RESULT -->|"有 candidates"| INIT_VICTIMS["victims = candidates"]
        INIT_VICTIMS --> P2

        P2 --> P2_RESULT{"返回结果?"}
        P2_RESULT -->|"弃权"| TIER1_CHECK
        P2_RESULT -->|"candidates=空"| TIER1_REJECT2["victims=nil, break"]
        P2_RESULT -->|"有 candidates"| INTERSECT["victims = intersect(victims, candidates)"]
        INTERSECT --> TIER1_CHECK
    end

    TIER1_CHECK{"victims != nil?"}
    TIER1_CHECK -->|"是"| RETURN["返回 victims"]
    TIER1_CHECK -->|"否"| TIER2["Tier 2 处理"]
    TIER1_REJECT --> TIER1_CHECK
    TIER1_REJECT2 --> TIER1_CHECK
    TIER2 --> RETURN2["同样的逻辑处理..."]
```

### 4.3 各 Plugin 的 PreemptableFn 实现

#### Gang Plugin

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 97-119 行

Gang Plugin 保护满足 MinAvailable 的 Job 不被抢占：

```go
preemptableFn := func(preemptor *api.TaskInfo, preemptees []*api.TaskInfo) ([]*api.TaskInfo, int) {
    var victims []*api.TaskInfo
    jobOccupiedMap := map[api.JobID]int32{}

    for _, preemptee := range preemptees {
        job := ssn.Jobs[preemptee.Job]
        if _, found := jobOccupiedMap[job.UID]; !found {
            jobOccupiedMap[job.UID] = job.ReadyTaskNum()
        }
        if jobOccupiedMap[job.UID] > job.MinAvailable {
            jobOccupiedMap[job.UID]--
            victims = append(victims, preemptee)    // 可以被抢占
        }
        // 否则跳过 - 不能破坏 Gang 约束
    }
    return victims, util.Permit
}
```

#### DRF Plugin

> 源码: `pkg/scheduler/plugins/drf/drf.go` 第 222-253 行

DRF Plugin 基于 Dominant Resource Fairness 份额比较：

```go
preemptableFn := func(preemptor *api.TaskInfo, preemptees []*api.TaskInfo) ([]*api.TaskInfo, int) {
    latt := drf.jobAttrs[preemptor.Job]
    lalloc := latt.allocated.Clone().Add(preemptor.Resreq)
    _, ls := drf.calculateShare(lalloc, drf.totalResource)  // preemptor 的 DRF 份额

    for _, preemptee := range preemptees {
        ralloc := allocations[preemptee.Job].Sub(preemptee.Resreq)
        _, rs := drf.calculateShare(ralloc, drf.totalResource)  // preemptee 移除后的 DRF 份额

        if ls < rs || math.Abs(ls-rs) <= shareDelta {
            victims = append(victims, preemptee)  // preemptor 份额 <= preemptee，允许抢占
        }
    }
    return victims, util.Permit
}
```

### 4.4 BuildVictimsPriorityQueue - Victim 排序

> 源码: `pkg/scheduler/framework/session_plugins.go` 第 1093-1138 行

Victim 排序优先队列确保先驱逐低优先级的 Task：

```go
victimsQueue := util.NewPriorityQueue(func(l, r interface{}) bool {
    lv := l.(*api.TaskInfo)
    rv := r.(*api.TaskInfo)
    if lv.Job == rv.Job {
        return !ssn.TaskOrderFn(l, r)        // 同 Job：反向 TaskOrder（低优先级在前）
    }
    // 不同 Job：反向 JobOrder
    if lvJob.Queue != rvJob.Queue {
        return ssn.VictimQueueOrderFn(...)    // 不同队列：使用 VictimQueueOrder
    }
    return !ssn.JobOrderFn(lvJob, rvJob)      // 同队列：反向 JobOrder
})
```

---

## 5. 模拟执行与验证（拓扑感知抢占）

### 5.1 拓扑感知抢占流程

> 源码: `pkg/scheduler/actions/preempt/preempt.go` 第 458-497 行

当启用 `enableTopologyAwarePreemption` 时，抢占使用更精细的模拟执行流程：

```mermaid
flowchart TD
    START["topologyAwarePreempt()"] --> FIND["findCandidates()"]
    FIND --> OFFSET["GetOffsetAndNumCandidates() - 随机偏移"]
    OFFSET --> DRY_RUN["DryRunPreemption() - 并行模拟"]

    subgraph dry_run_detail["DryRunPreemption 并行处理"]
        DRY_RUN --> PARALLEL["ParallelizeUntil() 并行检查节点"]
        PARALLEL --> CHECK_NODE["checkNode() 逐节点检查"]
        CHECK_NODE --> SELECT_VICTIMS_ON_NODE["SelectVictimsOnNode()"]
    end

    SELECT_VICTIMS_ON_NODE --> COLLECT_CANDIDATES["收集候选节点"]
    COLLECT_CANDIDATES --> SELECT_BEST["SelectCandidate() - 选择最佳候选"]
    SELECT_BEST --> PREPARE["prepareCandidate() - 执行驱逐"]
    PREPARE --> PIPELINE_TASK["Pipeline preemptor 到选中节点"]
```

### 5.2 SelectVictimsOnNode 详细流程

> 源码: `pkg/scheduler/actions/preempt/preempt.go` 第 691-825 行

这是拓扑感知抢占的核心方法，使用模拟函数进行验证：

```mermaid
flowchart TD
    START["SelectVictimsOnNode()"] --> FILTER["过滤节点上的候选 Victim"]
    FILTER --> PREEMPTABLE["ssn.Preemptable() - Plugin 筛选"]
    PREEMPTABLE --> VALIDATE["ValidateVictims() 验证"]
    VALIDATE --> SORT["按 Pod Priority 降序排列"]
    SORT --> BUILD_QUEUE["BuildVictimsPriorityQueue()"]
    BUILD_QUEUE --> EVICT_LOOP["逐个模拟移除 Victim"]

    subgraph simulation["模拟移除循环"]
        EVICT_LOOP --> REMOVE["SimulateRemoveTaskFn - 模拟移除"]
        REMOVE --> NODE_REMOVE["nodeInfo.RemoveTask()"]
        NODE_REMOVE --> CHECK_FIT{"SimulateAllocatableFn + FutureIdle 满足?"}
        CHECK_FIT -->|"否"| EVICT_LOOP
        CHECK_FIT -->|"是"| PREDICATE_CHECK["SimulatePredicateFn - 模拟 Predicate"]
        PREDICATE_CHECK -->|"通过"| STOP_EVICT["停止移除"]
        PREDICATE_CHECK -->|"不通过"| EVICT_LOOP
    end

    STOP_EVICT --> REPRIEVE["reprievePod - 尝试恢复 Victim"]

    subgraph reprieve_loop["恢复尝试循环"]
        REPRIEVE --> ADD_BACK["SimulateAddTaskFn - 模拟添加回"]
        ADD_BACK --> STILL_FIT{"preemptor 仍可调度?"}
        STILL_FIT -->|"是"| KEEP["保留此 Victim - 不需要驱逐"]
        STILL_FIT -->|"否"| MUST_EVICT["确认此 Victim 必须驱逐"]
    end

    MUST_EVICT --> RETURN["返回最终 victims 列表"]
    KEEP --> RETURN
```

### 5.3 四个模拟函数说明

| 函数 | 用途 | 注册 Plugin |
|------|------|------------|
| `SimulateRemoveTaskFn` | 模拟从节点移除一个 Victim，更新模拟状态 | proportion, predicates |
| `SimulatePredicateFn` | 在模拟状态上运行 Predicate 检查 | predicates |
| `SimulateAllocatableFn` | 在模拟状态上检查队列配额是否可分配 | proportion |
| `SimulateAddTaskFn` | 模拟将 Victim 添加回节点（reprieve 阶段） | proportion, predicates |

```go
// 模拟移除 - proportion plugin 的实现
// pkg/scheduler/plugins/proportion/proportion.go 第 433-447 行
ssn.AddSimulateRemoveTaskFn(pp.Name(), func(ctx context.Context, cycleState fwk.CycleState,
    taskToSchedule *api.TaskInfo, taskToRemove *api.TaskInfo, nodeInfo *api.NodeInfo) error {
    state, _ := getProportionState(cycleState)
    job := ssn.Jobs[taskToRemove.Job]
    attr := state.queueAttrs[job.Queue]
    attr.allocated.Sub(taskToRemove.Resreq)   // 模拟减少队列已分配资源
    updateQueueAttrShare(attr)
    return nil
})
```

### 5.4 SelectCandidate - 最佳候选选择

> 源码: `pkg/scheduler/actions/preempt/preempt.go` 第 829-854 行

当存在多个候选节点时，按以下优先级选择：

```mermaid
flowchart TD
    START["SelectCandidate()"] --> CHECK_COUNT{"候选数量?"}
    CHECK_COUNT -->|"0"| NIL["返回 nil"]
    CHECK_COUNT -->|"1"| SINGLE["直接返回唯一候选"]
    CHECK_COUNT -->|">1"| SCORE["评分函数排序"]

    SCORE --> S1["1. 最小最高优先级 Victim"]
    S1 -->|"平局"| S2["2. 最小优先级总和"]
    S2 -->|"平局"| S3["3. 最少 Victim 数量"]
    S3 -->|"平局"| S4["4. 最新 Victim 启动时间"]
    S4 -->|"平局"| S5["5. 选择第一个节点"]
```

---

## 6. Reclaim Action 流程

### 6.1 Reclaim 整体流程

> 源码: `pkg/scheduler/actions/reclaim/reclaim.go` 第 56-173 行

Reclaim Action 的目标是从超用队列回收资源给不足队列。

```mermaid
flowchart TD
    START["Execute() 入口"] --> BUILD_QUEUES["构建队列优先队列和 preemptorsMap"]
    BUILD_QUEUES --> QUEUE_LOOP["遍历队列优先队列"]
    QUEUE_LOOP --> POP_QUEUE["Pop 最高优先级队列"]
    POP_QUEUE --> CHECK_OVERUSED{"ssn.Overused(queue)?"}
    CHECK_OVERUSED -->|"是"| SKIP_QUEUE["跳过 - 队列已超用"]
    CHECK_OVERUSED -->|"否"| JOB_LOOP["遍历该队列的饥饿 Job"]

    JOB_LOOP --> POP_JOB["Pop 饥饿 Job"]
    POP_JOB --> STMT_CREATE["创建 Statement"]
    STMT_CREATE --> TASK_LOOP["遍历待调度 Task"]

    TASK_LOOP --> CHECK_POLICY{"PreemptionPolicy == Never?"}
    CHECK_POLICY -->|"是"| NEXT_TASK["下一个 Task"]
    CHECK_POLICY -->|"否"| CHECK_PREEMPTIVE{"ssn.Preemptive(queue, task)?"}
    CHECK_PREEMPTIVE -->|"否"| NEXT_TASK
    CHECK_PREEMPTIVE -->|"是"| RECLAIM_TASK["reclaimForTask()"]

    RECLAIM_TASK --> PIPELINE_CHECK{"ssn.JobPipelined?"}
    PIPELINE_CHECK -->|"是"| COMMIT["stmt.Commit()"]
    PIPELINE_CHECK -->|"否"| DISCARD["stmt.Discard()"]
```

### 6.2 reclaimForTask 详细流程

> 源码: `pkg/scheduler/actions/reclaim/reclaim.go` 第 175-254 行

```go
func (ra *Action) reclaimForTask(ssn *framework.Session, stmt *framework.Statement,
    task *api.TaskInfo, job *api.JobInfo) {
    // 1. 筛选候选节点
    totalNodes := ssn.FilterOutUnschedulableAndUnresolvableNodesForTask(task)
    predicateNodes, _ := predicateHelper.PredicateNodes(task, totalNodes,
        ssn.PredicateForPreemptAction, ...)   // 注意：使用 PreemptAction 的 Predicate（更宽松）

    for _, n := range predicateNodes {
        // 2. 收集该节点上其他队列的 Running Task 作为候选 reclaimee
        var reclaimees []*api.TaskInfo
        for _, taskOnNode := range n.Tasks {
            if taskOnNode.Status != api.Running { continue }
            j := ssn.Jobs[taskOnNode.Job]
            if j.Queue != job.Queue {              // 必须是不同队列
                q := ssn.Queues[j.Queue]
                if q.Reclaimable() {               // 队列允许被回收
                    reclaimees = append(reclaimees, taskOnNode.Clone())
                }
            }
        }

        // 3. 调用 Plugin 筛选 Victims
        victims := ssn.Reclaimable(task, reclaimees)

        // 4. 逐个驱逐 Victim 直到资源满足
        for !victimsQueue.Empty() {
            reclaimee := victimsQueue.Pop().(*api.TaskInfo)
            stmt.Evict(reclaimee, "reclaim")
            availableResources.Add(reclaimee.Resreq)
            if resreq.LessEqual(availableResources, api.Zero) {
                break   // 资源已足够
            }
        }

        // 5. Pipeline Task 到节点
        if task.InitResreq.LessEqual(availableResources, api.Zero) {
            stmt.Pipeline(task, n.Name, evictionOccurred)
            break
        }
    }
}
```

### 6.3 Overused 判断

> 源码: `pkg/scheduler/plugins/proportion/proportion.go` 第 300-312 行

Proportion Plugin 的 `overusedFn` 通过比较 allocated 和 deserved 来判断队列是否超用：

```go
ssn.AddOverusedFn(pp.Name(), func(obj interface{}) bool {
    queue := obj.(*api.QueueInfo)
    attr := pp.queueOpts[queue.UID]
    overused := attr.deserved.LessEqual(attr.allocated, api.Zero)
    // overused = true 当 deserved <= allocated（队列已超用）
    return overused
})
```

### 6.4 ReclaimableFn - Proportion Plugin

> 源码: `pkg/scheduler/plugins/proportion/proportion.go` 第 278-298 行

```go
ssn.AddReclaimableFn(pp.Name(), func(reclaimer *api.TaskInfo, reclaimees []*api.TaskInfo) ([]*api.TaskInfo, int) {
    var victims []*api.TaskInfo
    allocations := map[api.QueueID]*api.Resource{}

    for _, reclaimee := range reclaimees {
        job := ssn.Jobs[reclaimee.Job]
        attr := pp.queueOpts[job.Queue]
        allocated := allocations[job.Queue]

        // 只有当队列 allocated > deserved 时才回收
        if !allocated.LessEqual(attr.deserved, api.Zero) {
            allocated.Sub(reclaimee.Resreq)
            victims = append(victims, reclaimee)
        }
    }
    return victims, util.Permit
})
```

```mermaid
graph TD
    subgraph proportion_reclaim["Proportion ReclaimableFn 逻辑"]
        A["遍历 reclaimees"] --> B{"allocated > deserved?"}
        B -->|"是 - 队列超用"| C["加入 victims"]
        B -->|"否 - 队列未超用"| D["跳过 - 不可回收"]
        C --> E["allocated -= reclaimee.Resreq"]
        E --> F{"继续检查下一个 reclaimee"}
    end
```

---

## 7. 拓扑感知抢占

### 7.1 HyperNode 层级抢占

当启用 `enableTopologyAwarePreemption=true` 时，Preempt Action 使用拓扑感知的抢占策略。

```mermaid
flowchart TD
    subgraph topology_preempt["拓扑感知抢占流程"]
        A["preempt() 方法"] --> B{"enableTopologyAwarePreemption?"}
        B -->|"是"| C["topologyAwarePreempt()"]
        B -->|"否"| D["normalPreempt()"]

        C --> E["findCandidates()"]
        E --> F["GetOffsetAndNumCandidates() 计算搜索范围"]
        F --> G["DryRunPreemption() 并行模拟"]
        G --> H["SelectCandidate() 选择最佳候选"]
        H --> I["prepareCandidate() 执行驱逐"]
        I --> J["Pipeline Task"]
    end
```

### 7.2 候选节点数量控制

> 源码: `pkg/scheduler/actions/preempt/preempt.go` 第 564-590 行

```go
func (pmpt *Action) calculateNumCandidates(numNodes int) int {
    n := (numNodes * pmpt.minCandidateNodesPercentage) / 100  // 默认 10%
    if n < pmpt.minCandidateNodesAbsolute { n = pmpt.minCandidateNodesAbsolute }  // 最小 1
    if n > pmpt.maxCandidateNodesAbsolute { n = pmpt.maxCandidateNodesAbsolute }  // 最大 100
    if n > numNodes { n = numNodes }
    return n
}
```

配置参数：

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `minCandidateNodesPercentage` | 10 | 候选节点百分比 |
| `minCandidateNodesAbsolute` | 1 | 最小候选节点数 |
| `maxCandidateNodesAbsolute` | 100 | 最大候选节点数 |
| `topologyAwarePreemptWorkerNum` | 16 | 并行 worker 数量 |

### 7.3 DryRunPreemption 并行执行

> 源码: `pkg/scheduler/actions/preempt/preempt.go` 第 592-638 行

```go
func (pmpt *Action) DryRunPreemption(...) ([]*candidate, map[string]api.Status, error) {
    candidates := newCandidateList(numCandidates)
    ctx, cancel := context.WithCancel(context.Background())

    checkNode := func(i int) {
        nodeInfoCopy := potentialNodes[(offset+i)%len(potentialNodes)].Clone()
        stateCopy := state.Clone()

        victims, status := SelectVictimsOnNode(ctx, stateCopy, preemptor, ...)
        if status.IsSuccess() && len(victims) != 0 {
            candidates.add(&candidate{victims: victims, name: nodeInfoCopy.Name})
            if candidates.size() >= numCandidates {
                cancel()  // 找到足够候选，取消剩余检查
            }
        }
    }

    // 并行检查所有潜在节点
    workqueue.ParallelizeUntil(ctx, pmpt.topologyAwarePreemptWorkerNum,
        len(potentialNodes), checkNode)
    return candidates.get(), nodeStatuses, errs
}
```

---

## 8. 调试技巧

### 8.1 判断抢占是否触发

**检查日志级别 `-v=5`：**

```bash
# Preempt Action 入口和退出日志
"Enter Preempt ..."
"Leaving Preempt ..."

# Reclaim Action 入口和退出日志
"Enter Reclaim ..."
"Leaving Reclaim ..."
```

**确认饥饿 Job 检测：**

```bash
# 如果没有饥饿 Job，会看到：
"No preemptors in Queue <queue-name>, break."

# 如果 Job 被 Valid 检查拦截：
"Job <ns/name> Queue <queue> skip preemption, reason: ..."
```

### 8.2 追踪 Victim 选择过程

设置 `-v=3` 查看 Victim 选择细节：

```bash
# 节点遍历
"Considering Task <ns/name> on Node <node-name>."

# 抢占尝试
"Try to preempt Task <victim-ns/victim-name> for Task <preemptor-ns/preemptor-name>"

# 抢占结果
"Preempted <resource> for Task <ns/name> requested <resource>."
```

### 8.3 Delve 调试方法

```bash
# 编译 scheduler（禁用优化以便调试）
go build -gcflags="all=-N -l" -o _output/bin/vc-scheduler ./cmd/scheduler

# 启动 Delve
dlv exec _output/bin/vc-scheduler -- --scheduler-conf=/path/to/config

# 关键断点
break pkg/scheduler/actions/preempt/preempt.go:103     # Execute 入口
break pkg/scheduler/actions/preempt/preempt.go:284     # preempt 方法
break pkg/scheduler/actions/preempt/preempt.go:352     # Preemptable 调用
break pkg/scheduler/actions/preempt/preempt.go:375     # Allocatable 检查
break pkg/scheduler/actions/reclaim/reclaim.go:56      # Reclaim Execute 入口
break pkg/scheduler/actions/reclaim/reclaim.go:209     # Reclaimable 调用
break pkg/scheduler/framework/statement.go:418         # Commit
break pkg/scheduler/framework/statement.go:392         # Discard
```

### 8.4 常见抢占失败原因

```mermaid
flowchart TD
    FAIL["抢占失败"] --> CAUSE1["Job 未被识别为饥饿"]
    FAIL --> CAUSE2["无可抢占的 Victim"]
    FAIL --> CAUSE3["Victim 被 Gang 保护"]
    FAIL --> CAUSE4["DRF 份额不允许"]
    FAIL --> CAUSE5["队列配额不允许"]
    FAIL --> CAUSE6["Predicate 检查失败"]
    FAIL --> CAUSE7["PreemptionPolicy=Never"]

    CAUSE1 --> FIX1["检查 ReadyTaskNum + WaitingTaskNum vs MinAvailable"]
    CAUSE2 --> FIX2["确认节点上有 Running 且 Preemptable 的 Task"]
    CAUSE3 --> FIX3["Victim Job 的 ReadyTaskNum 必须 > MinAvailable"]
    CAUSE4 --> FIX4["preemptor DRF share 必须 <= preemptee DRF share"]
    CAUSE5 --> FIX5["检查 Proportion Plugin 的 deserved 和 allocated"]
    CAUSE6 --> FIX6["检查节点亲和性、资源容量等约束"]
    CAUSE7 --> FIX7["检查 Pod Spec 的 preemptionPolicy 字段"]
```

### 8.5 Preempt vs Allocate 的 Predicate 差异

Preempt Action 使用 `PredicateForPreemptAction` 而非 `PredicateFn`，关键区别在于对 `Unschedulable` 节点的处理更宽松：

```go
// preempt.go 第 302 行
predicateNodes, _ := predicateHelper.PredicateNodes(
    preemptor, allNodes,
    ssn.PredicateForPreemptAction,  // 更宽松：允许 Unschedulable 节点
    pmpt.enablePredicateErrorCache, ssn.NodesInShard)
```

在 Allocate Action 中使用的是 `ssn.PredicateFn`，它会拒绝标记为 `Unschedulable` 的节点。而在 Preempt 中，这些节点的 `Unschedulable` 状态可能是因为已有 Pod 占用了资源，驱逐后即可变为可调度状态。

### 8.6 Statement 事务追踪

Statement 的 Commit/Discard 是抢占流程中最关键的决策点。当调试时需要特别关注：

```go
// Commit 时执行的操作（statement.go 第 418-440 行）
func (s *Statement) Commit() {
    for _, op := range s.operations {
        switch op.name {
        case Evict:    s.evict(op.task, op.reason)   // 真正执行 Pod 驱逐
        case Pipeline: s.pipeline(op.task)            // 标记 Task 为 Pipelined
        case Allocate: s.allocate(op.task)            // 绑定 Task 到节点
        }
    }
}

// Discard 时执行的回滚操作（statement.go 第 392-415 行）
func (s *Statement) Discard() {
    for i := len(s.operations) - 1; i >= 0; i-- {   // 逆序回滚
        switch op.name {
        case Evict:    s.unevict(op.task)             // 恢复被驱逐的 Task
        case Pipeline: s.UnPipeline(op.task)           // 取消 Pipeline
        case Allocate: s.unallocate(op.task)           // 取消分配
        }
    }
}
```

---

## 9. 端到端调试案例

### 9.1 案例 - 为什么高优先级 Job 没有抢占成功

**场景描述：** Job-A（priority=100）在 Queue-1 中 Pending，Job-B（priority=1）在 Queue-1 中 Running，但 Job-A 始终无法抢占 Job-B。

**调试步骤：**

```mermaid
flowchart TD
    STEP1["Step 1 - 确认 Job-A 是否饥饿"] --> CHECK1{"IsStarving() == true?"}
    CHECK1 -->|"否"| FIX1["Job-A 已有足够资源，不需要抢占"]
    CHECK1 -->|"是"| STEP2["Step 2 - 确认 Job-A 是否通过 JobValid"]
    STEP2 --> CHECK2{"JobValid 通过?"}
    CHECK2 -->|"否"| FIX2["检查 MinAvailable 与 ValidTaskNum"]
    CHECK2 -->|"是"| STEP3["Step 3 - 确认 filter 函数"]
    STEP3 --> CHECK3{"filter 返回 true?"}
    CHECK3 -->|"否"| FIX3["检查 PreemptableStatus / BestEffort / Preemptable 标记"]
    CHECK3 -->|"是"| STEP4["Step 4 - 确认 Preemptable"]
    STEP4 --> CHECK4{"Preemptable 返回 victims?"}
    CHECK4 -->|"空"| FIX4["Gang/DRF Plugin 拒绝了抢占"]
    CHECK4 -->|"有"| STEP5["Step 5 - 检查 Predicate 和 Allocatable"]
    STEP5 --> CHECK5{"Pipeline 成功?"}
    CHECK5 -->|"否"| FIX5["节点资源或 Predicate 约束不满足"]
    CHECK5 -->|"是"| STEP6["Step 6 - 检查 JobPipelined"]
    STEP6 --> CHECK6{"JobPipelined == true?"}
    CHECK6 -->|"否"| FIX6["Gang 约束不满足 - Statement 被 Discard"]
    CHECK6 -->|"是"| SUCCESS["抢占应该成功 - 检查 Commit 日志"]
```

### 9.2 关键日志关键词

| 关键词 | 含义 |
|--------|------|
| `skip preemption, reason:` | Job 被跳过，不参与抢占 |
| `No preemptors in Queue` | 队列中没有饥饿的 Job |
| `No preemptor task in job` | Job 没有 Pending 的 Task |
| `Try to preempt Task` | 正在尝试抢占特定 Task |
| `Failed to preempt Task` | 抢占执行失败 |
| `Preempted <resource> for Task` | 成功驱逐了 Victim |
| `Committing operations` | Statement 正在提交 |
| `Discarding operations` | Statement 正在回滚 |
| `Queue is overused` | 队列已超用（Reclaim 中） |
| `cannot reclaim for task` | Preemptive 检查不通过 |
