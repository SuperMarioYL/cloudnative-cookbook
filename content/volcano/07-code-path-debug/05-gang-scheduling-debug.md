---
title: "Gang 调度调试指南"
weight: 5
---

## 1. 概述

Gang 调度（也称 All-or-Nothing 调度）是 Volcano 调度器的核心特性之一。它确保一个 Job 的所有（或至少 MinAvailable 个）Task 要么全部被调度，要么全部不被调度。这种语义对于 MPI、分布式训练等需要所有 Worker 同时就绪才能启动的场景至关重要。

### 1.1 核心思想

```mermaid
flowchart LR
    subgraph gang_success["Gang 调度成功 - 全部分配"]
        J1_T1["Task 1 - Allocated"] --- J1_T2["Task 2 - Allocated"]
        J1_T2 --- J1_T3["Task 3 - Allocated"]
        J1_T3 --- J1_T4["Task 4 - Allocated"]
    end

    subgraph gang_fail["Gang 调度失败 - 全部回滚"]
        J2_T1["Task 1 - Allocated"] --- J2_T2["Task 2 - Allocated"]
        J2_T2 --- J2_T3["Task 3 - Pending"]
        J2_T3 -.-|"MinAvailable=4 不满足"| J2_T4["Task 4 - Pending"]
        J2_T1 -.->|"Discard 回滚"| J2_T1_R["Task 1 - Pending"]
        J2_T2 -.->|"Discard 回滚"| J2_T2_R["Task 2 - Pending"]
    end
```

### 1.2 关键概念

| 概念 | 说明 | 来源 |
|------|------|------|
| **MinAvailable** | Job 需要的最小 Task 数量 | `PodGroup.Spec.MinMember` |
| **ReadyTaskNum** | 已就绪的 Task 数量（Bound + Binding + Running + Allocated） | `job_info.go:847` |
| **WaitingTaskNum** | 等待中的 Task 数量（Pipelined 状态） | `job_info.go:859` |
| **IsReady** | `ReadyTaskNum + PendingBestEffortTaskNum >= MinAvailable` | `job_info.go:1172` |
| **IsPipelined** | `WaitingTaskNum + ReadyTaskNum + PendingBestEffortTaskNum >= MinAvailable` | `job_info.go:1176` |
| **IsStarving** | `WaitingTaskNum + ReadyTaskNum < MinAvailable` | `job_info.go:1180` |

---

## 2. Gang Plugin 架构

### 2.1 Plugin 注册的钩子函数

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 57-213 行

Gang Plugin 在 `OnSessionOpen` 中注册了以下钩子函数：

```mermaid
graph TD
    subgraph gang_plugin["Gang Plugin 注册的钩子函数"]
        A["validJobFn"] -->|"验证 Job 是否合法"| A1["CheckTaskValid + CheckSubJobValid + ValidTaskNum"]
        B["preemptableFn"] -->|"抢占 Victim 筛选"| B1["保护 MinAvailable 不被破坏"]
        C["jobOrderFn"] -->|"Job 排序"| C1["未就绪 Job 优先调度"]
        D["subJobOrderFn"] -->|"SubJob 排序"| D1["未就绪 SubJob 优先"]
        E["jobReadyFn"] -->|"判断 Job 是否就绪"| E1["CheckTaskReady + CheckSubJobReady + IsReady"]
        F["subJobReadyFn"] -->|"判断 SubJob 是否就绪"| F1["SubJobInfo.IsReady()"]
        G["pipelinedFn"] -->|"判断 Job 是否 Pipelined"| G1["CheckTaskPipelined + CheckSubJobPipelined + IsPipelined"]
        H["subJobPipelinedFn"] -->|"判断 SubJob 是否 Pipelined"| H1["SubJobInfo.IsPipelined()"]
        I["jobStarvingFn"] -->|"判断 Job 是否饥饿"| I1["IsStarving()"]
    end
```

### 2.2 Plugin 注册代码

```go
// pkg/scheduler/plugins/gang/gang.go 第 57 行
func (gp *gangPlugin) OnSessionOpen(ssn *framework.Session) {
    ssn.AddJobValidFn(gp.Name(), validJobFn)           // 第 95 行
    ssn.AddReclaimableFn(gp.Name(), preemptableFn)      // 第 122 行
    ssn.AddPreemptableFn(gp.Name(), preemptableFn)      // 第 123 行
    ssn.AddJobOrderFn(gp.Name(), jobOrderFn)             // 第 149 行
    ssn.AddSubJobOrderFn(gp.Name(), subJobOrderFn)       // 第 175 行
    ssn.AddJobReadyFn(gp.Name(), jobReadyFn)             // 第 177 行
    ssn.AddSubJobReadyFn(gp.Name(), subJobReadyFn)       // 第 185 行
    ssn.AddJobPipelinedFn(gp.Name(), pipelinedFn)        // 第 197 行
    ssn.AddSubJobPipelinedFn(gp.Name(), subJobPipelinedFn)  // 第 199 行
    ssn.AddJobStarvingFns(gp.Name(), jobStarvingFn)      // 第 212 行
}
```

---

## 3. MinAvailable 验证链路

### 3.1 从 PodGroup 到 Job 的 MinAvailable 传递

```mermaid
flowchart TD
    PG["PodGroup.Spec.MinMember"] -->|"Controller 同步"| JOB_MIN["JobInfo.MinAvailable"]
    PG_TASKS["PodGroup.Spec.Tasks[].MinAvailable"] -->|"per-Task 最小数量"| TASK_MIN["JobInfo.TaskMinAvailable"]
    PG_TASKS -->|"总和"| TASK_MIN_TOTAL["JobInfo.TaskMinAvailableTotal"]
    JOB_MIN -->|"validJobFn 检查"| VALID["ValidTaskNum >= MinAvailable"]
    TASK_MIN -->|"CheckTaskValid 检查"| TASK_VALID["每个 Task 的有效数量 >= Task.MinAvailable"]
```

### 3.2 validJobFn 详细流程

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 58-93 行

`validJobFn` 是 Gang Plugin 注册的 Job 合法性验证函数，在 Preempt 和 Allocate Action 的入口处被调用。

```go
// gang.go 第 58-93 行
validJobFn := func(obj interface{}) *api.ValidateResult {
    job := obj.(*api.JobInfo)

    // 检查 1: 每个 Task 规格的有效 Pod 数量是否满足 Task 级别的 minAvailable
    if valid := job.CheckTaskValid(); !valid {
        return &api.ValidateResult{
            Pass:    false,
            Reason:  v1beta1.NotEnoughPodsOfTaskReason,
            Message: "Not enough valid pods of each task for gang-scheduling",
        }
    }

    // 检查 2: SubGroup 策略下，每个 SubGroup 的有效 Pod 数量
    if valid := job.CheckSubJobValid(); !valid {
        return &api.ValidateResult{
            Pass:    false,
            Reason:  v1beta1.NotEnoughPodsOfTaskReason,
            Message: "Not enough valid subGroups of each task for gang-scheduling",
        }
    }

    // 检查 3: 整体有效 Task 数量是否满足 Job 级别的 MinAvailable
    vtn := job.ValidTaskNum()
    if vtn < job.MinAvailable {
        return &api.ValidateResult{
            Pass:   false,
            Reason: v1beta1.NotEnoughPodsReason,
            Message: fmt.Sprintf("Not enough valid tasks for gang-scheduling, valid: %d, min: %d",
                vtn, job.MinAvailable),
        }
    }
    return nil
}
```

```mermaid
flowchart TD
    START["validJobFn(job)"] --> CHECK1["CheckTaskValid()"]
    CHECK1 --> C1_DETAIL{"MinAvailable < TaskMinAvailableTotal?"}
    C1_DETAIL -->|"是"| C1_SKIP["跳过 Task 级别检查"]
    C1_DETAIL -->|"否"| C1_CHECK["遍历每个 Task 规格"]
    C1_CHECK --> C1_VALID{"每个 Task 的 ValidNum >= TaskMinAvailable?"}
    C1_VALID -->|"否"| FAIL1["返回 NotEnoughPodsOfTaskReason"]
    C1_VALID -->|"是"| CHECK2["CheckSubJobValid()"]

    C1_SKIP --> CHECK2
    CHECK2 --> C2_VALID{"每个 SubGroup 有效?"}
    C2_VALID -->|"否"| FAIL2["返回 NotEnoughPodsOfTaskReason"]
    C2_VALID -->|"是"| CHECK3["ValidTaskNum()"]
    CHECK3 --> C3_VALID{"ValidTaskNum >= MinAvailable?"}
    C3_VALID -->|"否"| FAIL3["返回 NotEnoughPodsReason"]
    C3_VALID -->|"是"| PASS["返回 nil - 验证通过"]
```

### 3.3 CheckTaskValid 详解

> 源码: `pkg/scheduler/api/job_info.go` 第 995-1024 行

```go
func (ji *JobInfo) CheckTaskValid() bool {
    // 如果 MinAvailable < TaskMinAvailableTotal，跳过此检查
    if ji.MinAvailable < ji.TaskMinAvailableTotal {
        return true
    }
    // 检查每个 Task 规格
    for taskSpec, minNum := range ji.TaskMinAvailable {
        validNum := int32(0)
        for _, task := range ji.Tasks {
            if task.Role == taskSpec {
                // 有效状态: AllocatedStatus || Succeeded || Pipelined || Pending || Waiting
                if AllocatedStatus(task.Status) || task.Status == Succeeded ||
                    task.Status == Pipelined || task.Status == Pending || task.Status == Waiting {
                    validNum++
                }
            }
        }
        if validNum < minNum {
            return false   // 该 Task 规格的有效 Pod 不足
        }
    }
    return true
}
```

### 3.4 ValidTaskNum 详解

> 源码: `pkg/scheduler/api/job_info.go` 第 1100-1112 行

```go
func (ji *JobInfo) ValidTaskNum() int32 {
    occupied := 0
    for status, tasks := range ji.TaskStatusIndex {
        if AllocatedStatus(status) ||
            status == Succeeded ||
            status == Pipelined ||
            status == Pending ||
            status == Waiting {
            occupied += len(tasks)
        }
    }
    return int32(occupied)
}
```

### 3.5 调试方法 - MinAvailable 验证

```bash
# 设置 Delve 断点
break pkg/scheduler/plugins/gang/gang.go:67   # CheckTaskValid
break pkg/scheduler/plugins/gang/gang.go:75   # CheckSubJobValid
break pkg/scheduler/plugins/gang/gang.go:83   # ValidTaskNum 比较

# 条件断点 - 仅命中特定 Job
condition 1 job.Name == "my-training-job"

# 打印关键信息
print job.MinAvailable
print job.ValidTaskNum()
print job.TaskMinAvailable
print job.TaskMinAvailableTotal
```

---

## 4. Job 就绪判断

### 4.1 jobReadyFn

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 177-183 行

```go
ssn.AddJobReadyFn(gp.Name(), func(obj interface{}) bool {
    ji := obj.(*api.JobInfo)
    if ji.CheckTaskReady() && ji.CheckSubJobReady() && ji.IsReady() {
        return true
    }
    return false
})
```

```mermaid
flowchart TD
    START["jobReadyFn(job)"] --> CHECK1["CheckTaskReady()"]

    subgraph check_task_ready["CheckTaskReady 详解"]
        CHECK1 --> CTR_SKIP{"MinAvailable < TaskMinAvailableTotal?"}
        CTR_SKIP -->|"是"| CTR_PASS["跳过 Task 级别检查"]
        CTR_SKIP -->|"否"| CTR_CHECK["遍历每个 Task 规格"]
        CTR_CHECK --> CTR_VALID{"AllocatedTaskNum >= TaskMinAvailable?"}
        CTR_VALID -->|"否"| FAIL1["返回 false"]
    end

    CTR_PASS --> CHECK2["CheckSubJobReady()"]
    CTR_VALID -->|"是"| CHECK2

    subgraph check_subjob_ready["CheckSubJobReady 详解"]
        CHECK2 --> CSR_CHECK["检查每个 SubGroup"]
        CSR_CHECK --> CSR_VALID{"SubJob.IsReady()?"}
        CSR_VALID -->|"否"| FAIL2["返回 false"]
    end

    CSR_VALID -->|"是"| CHECK3["IsReady()"]

    subgraph is_ready["IsReady 详解"]
        CHECK3 --> IR_CALC["ReadyTaskNum + PendingBestEffortTaskNum"]
        IR_CALC --> IR_COMPARE{">= MinAvailable?"}
        IR_COMPARE -->|"否"| FAIL3["返回 false"]
        IR_COMPARE -->|"是"| PASS["返回 true - Job 就绪"]
    end
```

### 4.2 CheckTaskReady 详解

> 源码: `pkg/scheduler/api/job_info.go` 第 1027-1039 行

```go
func (ji *JobInfo) CheckTaskReady() bool {
    if ji.MinAvailable < ji.TaskMinAvailableTotal {
        return true  // 当 Job MinAvailable 小于所有 Task MinAvailable 之和时跳过
    }
    occupiedMap := ji.getJobAllocatedRoles()  // 获取每个 Role 的已分配数量
    for taskSpec, minNum := range ji.TaskMinAvailable {
        if occupiedMap[taskSpec] < minNum {
            return false  // 该 Role 的已分配数量不足
        }
    }
    return true
}
```

### 4.3 IsReady 详解

> 源码: `pkg/scheduler/api/job_info.go` 第 1172-1174 行

```go
func (ji *JobInfo) IsReady() bool {
    return ji.ReadyTaskNum()+ji.PendingBestEffortTaskNum() >= ji.MinAvailable
}
```

**ReadyTaskNum 统计的状态：**

```mermaid
graph LR
    subgraph ready_states["ReadyTaskNum 统计的状态"]
        S1["Bound"]
        S2["Binding"]
        S3["Running"]
        S4["Allocated"]
    end
    subgraph not_ready["不计入 ReadyTaskNum"]
        S5["Pending"]
        S6["Pipelined"]
        S7["Releasing"]
        S8["Succeeded"]
        S9["Failed"]
    end
```

### 4.4 Session.JobReady 的调用链

> 源码: `pkg/scheduler/framework/session_plugins.go` 第 429-447 行

```go
func (ssn *Session) JobReady(obj interface{}) bool {
    for _, tier := range ssn.Tiers {
        for _, plugin := range tier.Plugins {
            if !isEnabled(plugin.EnabledJobReady) { continue }
            jrf, found := ssn.jobReadyFns[plugin.Name]
            if !found { continue }
            if !jrf(obj) {
                return false  // 任何一个 plugin 说不就绪，就不就绪
            }
        }
    }
    return true  // 所有注册的 plugin 都说就绪（或没有 plugin 注册）
}
```

---

## 5. Pipelined 机制

### 5.1 Pipelined 状态的含义

Pipelined 是一个中间状态，表示 Task 已经被"预分配"到节点但尚未真正绑定。这个状态在 Preempt Action 中特别重要，因为抢占需要先驱逐 Victim 才能释放资源。

### 5.2 pipelinedFn

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 190-197 行

```go
pipelinedFn := func(obj interface{}) int {
    ji := obj.(*api.JobInfo)
    if ji.CheckTaskPipelined() && ji.CheckSubJobPipelined() && ji.IsPipelined() {
        return util.Permit   // 返回正值表示 Pipelined
    }
    return util.Reject       // 返回负值表示未 Pipelined
}
```

### 5.3 Pipelined vs Ready 的区别

```mermaid
graph TD
    subgraph pipelined_check["IsPipelined 检查"]
        P1["WaitingTaskNum (Pipelined 状态)"]
        P2["ReadyTaskNum (Bound/Binding/Running/Allocated)"]
        P3["PendingBestEffortTaskNum"]
        P1 --> P_SUM["三者之和 >= MinAvailable"]
    end

    subgraph ready_check["IsReady 检查"]
        R1["ReadyTaskNum (Bound/Binding/Running/Allocated)"]
        R2["PendingBestEffortTaskNum"]
        R1 --> R_SUM["两者之和 >= MinAvailable"]
    end

    P_SUM -->|"更宽松"| PIPELINE_RESULT["允许 Pipeline 中的 Task 计入"]
    R_SUM -->|"更严格"| READY_RESULT["只计算真正分配的 Task"]
```

| 维度 | IsPipelined | IsReady |
|------|------------|---------|
| **计入 Pipelined 状态** | 是 | 否 |
| **计入 Allocated 状态** | 是 | 是 |
| **计入 Running 状态** | 是 | 是 |
| **使用场景** | Preempt/Reclaim 中的 Commit/Discard 决策 | Allocate Action 中的 Commit/Discard 决策 |
| **宽松程度** | 更宽松 | 更严格 |

### 5.4 CheckTaskPipelined 详解

> 源码: `pkg/scheduler/api/job_info.go` 第 1042-1070 行

```go
func (ji *JobInfo) CheckTaskPipelined() bool {
    if ji.MinAvailable < ji.TaskMinAvailableTotal {
        return true
    }
    occupiedMap := map[string]int32{}
    for status, tasks := range ji.TaskStatusIndex {
        if AllocatedStatus(status) || status == Pipelined {  // 关键：包含 Pipelined
            for _, task := range tasks {
                occupiedMap[task.Role]++
            }
        }
    }
    for taskSpec, minNum := range ji.TaskMinAvailable {
        if occupiedMap[taskSpec] < minNum {
            return false
        }
    }
    return true
}
```

### 5.5 调用时机

```mermaid
sequenceDiagram
    participant Preempt as Preempt Action
    participant Stmt as Statement
    participant Gang as Gang Plugin
    participant Session as Session

    Preempt->>Stmt: 创建 Statement
    loop 为每个 Pending Task 抢占
        Preempt->>Stmt: stmt.Evict(victim)
        Preempt->>Stmt: stmt.Pipeline(preemptor, node)
    end

    Preempt->>Session: ssn.JobPipelined(preemptorJob)
    Session->>Gang: pipelinedFn(job)
    Gang->>Gang: CheckTaskPipelined()
    Gang->>Gang: CheckSubJobPipelined()
    Gang->>Gang: IsPipelined()

    alt Job 已 Pipelined
        Gang-->>Session: Permit
        Session-->>Preempt: true
        Preempt->>Stmt: stmt.Commit()
        Note over Stmt: 执行所有 Evict 和 Pipeline 操作
    else Job 未 Pipelined
        Gang-->>Session: Reject
        Session-->>Preempt: false
        Preempt->>Stmt: stmt.Discard()
        Note over Stmt: 逆序回滚所有操作
    end
```

---

## 6. Commit/Discard 决策

### 6.1 Statement 模式

> 源码: `pkg/scheduler/framework/statement.go`

Statement 是 Volcano 调度器的事务抽象，在 Allocate、Preempt、Reclaim 等 Action 中用于实现原子性操作。

```mermaid
flowchart TD
    subgraph statement_lifecycle["Statement 生命周期"]
        CREATE["NewStatement(ssn)"] --> OPS["记录操作"]
        OPS --> EVICT["Evict(task) - 记录驱逐"]
        OPS --> PIPELINE["Pipeline(task, node) - 记录预分配"]
        OPS --> ALLOCATE["Allocate(task, node) - 记录分配"]

        EVICT --> DECISION{"Gang 检查"}
        PIPELINE --> DECISION
        ALLOCATE --> DECISION

        DECISION -->|"JobPipelined/JobReady"| COMMIT["Commit()"]
        DECISION -->|"!JobPipelined/!JobReady"| DISCARD["Discard()"]
    end

    subgraph commit_detail["Commit 操作"]
        COMMIT --> C_EVICT["evict() - 调用 cache.Evict 真正删除 Pod"]
        COMMIT --> C_PIPELINE["pipeline() - 标记 Task"]
        COMMIT --> C_ALLOCATE["allocate() - 调用 cache.AddBindTask 绑定 Pod"]
    end

    subgraph discard_detail["Discard 操作 - 逆序回滚"]
        DISCARD --> D_UNEVICT["unevict() - 恢复 Task 为 Running"]
        DISCARD --> D_UNPIPELINE["UnPipeline() - 恢复 Task 为 Pending"]
        DISCARD --> D_UNALLOCATE["unallocate() - 恢复 Task 为 Pending"]
    end
```

### 6.2 Commit 实现

> 源码: `pkg/scheduler/framework/statement.go` 第 418-440 行

```go
func (s *Statement) Commit() {
    klog.V(3).Info("Committing operations ...")
    for _, op := range s.operations {
        op.task.ClearLastTxContext()
        switch op.name {
        case Evict:
            err := s.evict(op.task, op.reason)    // 调用 cache.Evict -> 删除 Pod
        case Pipeline:
            s.pipeline(op.task)                    // 空操作 - Pipeline 状态已在记录时更新
        case Allocate:
            err := s.allocate(op.task)             // 调用 cache.AddBindTask -> 绑定 Pod
            if err != nil {
                s.unallocate(op.task)              // 失败时回滚
            }
        }
    }
}
```

### 6.3 Discard 实现

> 源码: `pkg/scheduler/framework/statement.go` 第 392-415 行

```go
func (s *Statement) Discard() {
    klog.V(3).Info("Discarding operations ...")
    for i := len(s.operations) - 1; i >= 0; i-- {   // 关键：逆序回滚
        op := s.operations[i]
        op.task.GenerateLastTxContext()
        switch op.name {
        case Evict:
            s.unevict(op.task)       // 恢复 Task 状态为 Running
        case Pipeline:
            s.UnPipeline(op.task)    // 恢复 Task 状态为 Pending，从节点移除
        case Allocate:
            s.unallocate(op.task)    // 恢复 Task 状态为 Pending，从节点移除
        }
    }
}
```

### 6.4 关键调试点

```mermaid
flowchart TD
    subgraph allocate_action["Allocate Action 中的决策"]
        A1["为 Job 的 Tasks 分配节点"] --> A2["stmt.Allocate(task, node)"]
        A2 --> A3{"ssn.JobReady(job)?"}
        A3 -->|"是"| A4["stmt.Commit() - 提交绑定"]
        A3 -->|"否"| A5["stmt.Discard() - 回滚分配"]
    end

    subgraph preempt_action["Preempt Action 中的决策"]
        P1["驱逐 Victim + Pipeline preemptor"] --> P2["stmt.Evict() + stmt.Pipeline()"]
        P2 --> P3{"ssn.JobPipelined(job)?"}
        P3 -->|"是"| P4["stmt.Commit() - 提交驱逐"]
        P3 -->|"否"| P5["stmt.Discard() - 恢复 Victim"]
    end
```

---

## 7. Job 排序逻辑

### 7.1 jobOrderFn

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 125-148 行

Gang Plugin 的 Job 排序逻辑的核心设计思想是：**让未就绪的 Job 优先被调度**。

```go
jobOrderFn := func(l, r interface{}) int {
    lv := l.(*api.JobInfo)
    rv := r.(*api.JobInfo)

    lReady := lv.IsReady()
    rReady := rv.IsReady()

    if lReady && rReady { return 0 }  // 两个都就绪，不区分
    if lReady { return 1 }             // l 就绪，r 未就绪 -> r 优先（返回正值 = l 排后）
    if rReady { return -1 }            // r 就绪，l 未就绪 -> l 优先（返回负值 = l 排前）
    return 0                           // 两个都未就绪，不区分
}
```

```mermaid
graph TD
    subgraph job_order["Gang jobOrderFn 排序逻辑"]
        INPUT["比较 Job-L vs Job-R"] --> CHECK_BOTH{"都就绪?"}
        CHECK_BOTH -->|"是"| EQUAL["返回 0 - 相等"]
        CHECK_BOTH -->|"否"| CHECK_L{"L 就绪?"}
        CHECK_L -->|"是"| L_AFTER["返回 1 - L 排后"]
        CHECK_L -->|"否"| CHECK_R{"R 就绪?"}
        CHECK_R -->|"是"| R_AFTER["返回 -1 - L 排前"]
        CHECK_R -->|"否"| BOTH_UNREADY["返回 0 - 相等"]
    end
```

### 7.2 设计原理

为什么未就绪的 Job 要优先调度？

1. **资源利用效率** - 已就绪的 Job 已经获得了足够资源，不需要额外资源
2. **避免饥饿** - 优先分配给未就绪的 Job，有助于其尽快满足 MinAvailable
3. **Gang 语义保障** - 集中资源让未就绪 Job 尽快凑齐 Task，而不是分散资源

### 7.3 subJobOrderFn

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 151-174 行

SubJob 排序逻辑与 Job 排序完全一致：

```go
subJobOrderFn := func(l, r interface{}) int {
    lv := l.(*api.SubJobInfo)
    rv := r.(*api.SubJobInfo)
    lReady := lv.IsReady()
    rReady := rv.IsReady()
    // 同样的逻辑：未就绪的优先
    if lReady && rReady { return 0 }
    if lReady { return 1 }
    if rReady { return -1 }
    return 0
}
```

### 7.4 JobOrderFn 在 Session 中的调用

> 源码: `pkg/scheduler/framework/session_plugins.go` 第 661-684 行

```go
func (ssn *Session) JobOrderFn(l, r interface{}) bool {
    for _, tier := range ssn.Tiers {
        for _, plugin := range tier.Plugins {
            if !isEnabled(plugin.EnabledJobOrder) { continue }
            jof, found := ssn.jobOrderFns[plugin.Name]
            if !found { continue }
            if j := jof(l, r); j != 0 {
                return j < 0  // 负值 = l 排前 = 优先级更高
            }
        }
    }
    // 没有 plugin 决定的情况下，按创建时间排序
    lv := l.(*api.JobInfo)
    rv := r.(*api.JobInfo)
    if lv.CreationTimestamp.Equal(&rv.CreationTimestamp) {
        return lv.UID < rv.UID
    }
    return lv.CreationTimestamp.Before(&rv.CreationTimestamp)
}
```

---

## 8. SubGroup 策略

### 8.1 SubGroupPolicy 概述

SubGroup 策略允许将 Job 的 Task 分组，每个组可以有独立的 MinAvailable 要求。这在混合 Worker 类型（如 GPU Worker + CPU Worker）的场景中非常有用。

### 8.2 SubJob 就绪检查

> 源码: `pkg/scheduler/api/sub_job_info.go` 第 216-222 行

```go
func (sji *SubJobInfo) IsReady() bool {
    return sji.ReadyTaskNum()+sji.PendingBestEffortTaskNum() >= sji.MinAvailable
}

func (sji *SubJobInfo) IsPipelined() bool {
    return sji.WaitingTaskNum()+sji.ReadyTaskNum()+sji.PendingBestEffortTaskNum() >= sji.MinAvailable
}
```

### 8.3 CheckSubJobValid 和 CheckSubJobReady

> 源码: `pkg/scheduler/api/job_info.go` 第 1114-1160 行

```go
func (ji *JobInfo) CheckSubJobValid() bool {
    subJobs := map[SubJobGID]int32{}
    for _, subJob := range ji.SubJobs {
        if _, ok := subJobs[subJob.GID]; !ok {
            subJobs[subJob.GID] = 0
        }
        // 统计每个 SubGroup 中有效的 SubJob 数量
        if subJob.ValidTaskNum() >= subJob.MinAvailable {
            subJobs[subJob.GID]++
        }
    }
    // 检查每个 SubGroup 是否满足 MinSubJobs
    // ...
}

func (ji *JobInfo) CheckSubJobReady() bool {
    if err := ji.checkSubJobCondition(func(subJob *SubJobInfo) bool {
        return subJob.IsReady()
    }); err != nil {
        return false
    }
    return true
}
```

### 8.4 Session 级 SubJobReady 和 SubJobPipelined

> 源码: `pkg/scheduler/framework/session_plugins.go` 第 370-426 行

```go
func (ssn *Session) SubJobReady(job *api.JobInfo, subJob *api.SubJobInfo) bool {
    if !job.ContainsSubJobPolicy() {
        return ssn.JobReady(job)  // 无 SubJob 策略时退化为 JobReady
    }
    for _, tier := range ssn.Tiers {
        for _, plugin := range tier.Plugins {
            if !isEnabled(plugin.EnabledSubJobReady) { continue }
            fn, found := ssn.subJobReadyFns[plugin.Name]
            if !found { continue }
            if !fn(subJob) { return false }
        }
    }
    return true
}
```

```mermaid
flowchart TD
    START["SubJobReady(job, subJob)"] --> CHECK_POLICY{"job.ContainsSubJobPolicy()?"}
    CHECK_POLICY -->|"否"| FALLBACK["退化为 JobReady(job)"]
    CHECK_POLICY -->|"是"| TIER_LOOP["遍历 Tiers"]
    TIER_LOOP --> PLUGIN_CHECK["调用各 Plugin 的 subJobReadyFn"]
    PLUGIN_CHECK --> GANG_CHECK["Gang Plugin - subJobReadyFn"]
    GANG_CHECK --> SUB_IS_READY{"SubJobInfo.IsReady()?"}
    SUB_IS_READY -->|"否"| NOT_READY["返回 false"]
    SUB_IS_READY -->|"是"| NEXT_PLUGIN["检查下一个 Plugin"]
    NEXT_PLUGIN --> READY["返回 true"]
```

---

## 9. OnSessionClose 状态更新

### 9.1 Session 关闭时的处理

> 源码: `pkg/scheduler/plugins/gang/gang.go` 第 215-283 行

当调度周期结束时，Gang Plugin 的 `OnSessionClose` 会更新不满足 MinAvailable 的 Job 的 PodGroup 状态。

```go
func (gp *gangPlugin) OnSessionClose(ssn *framework.Session) {
    var unreadyTaskCount int32
    var unScheduleJobCount int

    for _, job := range ssn.Jobs {
        if len(job.Tasks) == 0 { continue }

        if !job.IsReady() {
            // 计算不可调度的 Task 数量
            schedulableTaskNum := func() (num int32) {
                for _, task := range job.TaskStatusIndex[api.Pending] {
                    ctx := task.GetTransactionContext()
                    if task.LastTransaction != nil {
                        ctx = *task.LastTransaction
                    }
                    if api.AllocatedStatus(ctx.Status) {
                        num++
                    }
                }
                return num + job.ReadyTaskNum()
            }
            unreadyTaskCount = job.MinAvailable - schedulableTaskNum()

            msg := fmt.Sprintf("%v/%v tasks in gang unschedulable: %v",
                unreadyTaskCount, len(job.Tasks), job.FitError())

            // 更新 PodGroupCondition 为 Unschedulable
            jc := &scheduling.PodGroupCondition{
                Type:               scheduling.PodGroupUnschedulableType,
                Status:             v1.ConditionTrue,
                LastTransitionTime: metav1.Now(),
                TransitionID:       string(ssn.UID),
                Reason:             v1beta1.NotEnoughResourcesReason,
                Message:            msg,
            }
            ssn.UpdatePodGroupCondition(job, jc)
        } else {
            // Job 就绪 - 更新为 Scheduled
            jc := &scheduling.PodGroupCondition{
                Type:               scheduling.PodGroupScheduled,
                Status:             v1.ConditionTrue,
                Reason:             "tasks in gang are ready to be scheduled",
            }
            ssn.UpdatePodGroupCondition(job, jc)
        }
    }
}
```

```mermaid
flowchart TD
    START["OnSessionClose()"] --> LOOP["遍历 ssn.Jobs"]
    LOOP --> CHECK_TASKS{"len(job.Tasks) == 0?"}
    CHECK_TASKS -->|"是"| SKIP["跳过"]
    CHECK_TASKS -->|"否"| CHECK_READY{"job.IsReady()?"}

    CHECK_READY -->|"否 - 不满足 Gang"| CALC_UNREADY["计算 unreadyTaskCount"]
    CALC_UNREADY --> BUILD_MSG["构建失败消息"]
    BUILD_MSG --> UPDATE_UNSCHEDULABLE["更新 PodGroupCondition"]
    UPDATE_UNSCHEDULABLE --> SET_UNSCHEDULABLE["Type = PodGroupUnschedulableType"]
    SET_UNSCHEDULABLE --> METRICS1["RegisterJobRetries + UpdateUnscheduleTaskCount"]

    CHECK_READY -->|"是 - 满足 Gang"| UPDATE_SCHEDULED["更新 PodGroupCondition"]
    UPDATE_SCHEDULED --> SET_SCHEDULED["Type = PodGroupScheduled"]

    METRICS1 --> NEXT["下一个 Job"]
    SET_SCHEDULED --> NEXT
```

### 9.2 schedulableTaskNum 的计算逻辑

这个内部函数计算的是"可调度的 Task 数量"，它不仅统计当前已分配的 Task，还考虑了**上一次事务上下文**中的状态：

```go
schedulableTaskNum := func() (num int32) {
    for _, task := range job.TaskStatusIndex[api.Pending] {
        ctx := task.GetTransactionContext()
        if task.LastTransaction != nil {
            ctx = *task.LastTransaction      // 使用上一次事务的状态
        }
        if api.AllocatedStatus(ctx.Status) {
            num++                            // Pending 但上次事务中被 Allocated 的 Task
        }
    }
    return num + job.ReadyTaskNum()
}
```

这种计算方式考虑了 Discard 回滚的情况：某些 Task 在本次调度周期中曾被分配但因为 Gang 不满足而被回滚。

---

## 10. 调试技巧

### 10.1 判断 Gang 为什么没有就绪

```mermaid
flowchart TD
    PROBLEM["Job Gang 未就绪"] --> STEP1["Step 1 - 检查 Job 状态"]
    STEP1 --> CHECK_VALID{"JobValid 通过?"}
    CHECK_VALID -->|"否"| FIX_VALID["检查 ValidTaskNum vs MinAvailable"]
    CHECK_VALID -->|"是"| STEP2["Step 2 - 检查 ReadyTaskNum"]

    STEP2 --> PRINT_NUMS["打印 ReadyTaskNum / WaitingTaskNum / MinAvailable"]
    PRINT_NUMS --> CHECK_READY{"ReadyTaskNum >= MinAvailable?"}
    CHECK_READY -->|"否"| STEP3["Step 3 - 为什么 Task 没有被分配"]

    STEP3 --> CAUSE1["节点资源不足"]
    STEP3 --> CAUSE2["Predicate 失败"]
    STEP3 --> CAUSE3["队列配额不足 (Allocatable 检查)"]
    STEP3 --> CAUSE4["Task 被 SchedulingGate 阻止"]

    CHECK_READY -->|"是"| STEP4["Step 4 - 检查 CheckTaskReady"]
    STEP4 --> TASK_READY_FAIL{"某个 Task 角色数量不足?"}
    TASK_READY_FAIL -->|"是"| FIX_TASK["增加该角色的 Pod 数量"]
    TASK_READY_FAIL -->|"否"| STEP5["Step 5 - 检查 CheckSubJobReady"]
    STEP5 --> SUBJOB_FAIL{"SubJob 未就绪?"}
    SUBJOB_FAIL -->|"是"| FIX_SUBJOB["检查 SubJob 的 MinAvailable 设置"]
```

### 10.2 MinAvailable 不满足的排查

**使用 kubectl 检查 PodGroup 状态：**

```bash
# 查看 PodGroup 状态
kubectl get podgroup -n <namespace> <podgroup-name> -o yaml

# 关注字段
spec:
  minMember: 4        # 需要的最小 Task 数
status:
  phase: Pending       # 当前阶段
  conditions:
  - type: Unschedulable
    status: "True"
    reason: NotEnoughResources
    message: "2/4 tasks in gang unschedulable: ..."

# 查看 Job 的 Pod 分布
kubectl get pods -n <namespace> -l volcano.sh/job-name=<job-name> -o wide
```

**使用 Delve 断点调试：**

```bash
# Gang 就绪检查
break pkg/scheduler/plugins/gang/gang.go:178   # jobReadyFn
break pkg/scheduler/api/job_info.go:1172        # IsReady
break pkg/scheduler/api/job_info.go:847         # ReadyTaskNum

# Pipeline 检查
break pkg/scheduler/plugins/gang/gang.go:191   # pipelinedFn
break pkg/scheduler/api/job_info.go:1176        # IsPipelined

# OnSessionClose 状态更新
break pkg/scheduler/plugins/gang/gang.go:223   # !job.IsReady() 分支
```

### 10.3 Commit/Discard 决策追踪

**关键日志：**

```bash
# 日志级别 -v=3

# Commit 成功
"Committing operations ..."

# Discard 回滚
"Discarding operations ..."

# Allocate Action 中
"After allocated Task <ns/name> to Node <node>: idle <...>, used <...>, releasing <...>"

# Pipeline 操作
"After pipelined Task <ns/name> to Node <node>: idle <...>, used <...>, releasing <...>"

# UnPipeline 回滚
"After unpipelined Task <ns/name> to Node <node>: idle <...>, used <...>, releasing <...>"
```

**Delve 调试 Statement 操作：**

```bash
break pkg/scheduler/framework/statement.go:418   # Commit
break pkg/scheduler/framework/statement.go:392   # Discard
break pkg/scheduler/framework/statement.go:72    # Evict
break pkg/scheduler/framework/statement.go:157   # Pipeline
break pkg/scheduler/framework/statement.go:263   # Allocate

# 在 Commit/Discard 断点处检查操作列表
print len(s.operations)
print s.operations[0].name
print s.operations[0].task.Name
```

### 10.4 常见 Gang 调度问题

| 问题 | 可能原因 | 排查方法 |
|------|---------|---------|
| Job 始终 Pending | MinAvailable 设置过高 | 检查 `PodGroup.Spec.MinMember` 与实际可用资源 |
| 部分 Task Pending | 单个 Task Role 的 MinAvailable 不满足 | 检查 `CheckTaskReady` 中每个 Role 的分配情况 |
| Job 反复 Allocate/Discard | 资源刚好在边界 | 查看 `-v=3` 日志中的 Commit/Discard 循环 |
| SubJob 策略不生效 | SubGroupPolicy 配置问题 | 检查 `ContainsSubJobPolicy()` 返回值 |
| 抢占后 Gang 仍不满足 | Victim 数量不足以释放所需资源 | 检查 `Preemptable` 返回的 Victim 列表 |
| PodGroup 状态不更新 | `OnSessionClose` 未执行 | 确认 Gang Plugin 是否在配置中启用 |

### 10.5 完整调试案例

**场景：** 一个 TensorFlow 分布式训练 Job（minMember=8，包含 1 个 PS 和 7 个 Worker）始终无法就绪。

```mermaid
sequenceDiagram
    participant Scheduler as Scheduler Loop
    participant Allocate as Allocate Action
    participant Gang as Gang Plugin
    participant Statement as Statement

    Scheduler->>Allocate: Execute(ssn)
    Allocate->>Gang: JobValid(job) - 检查 Job 合法性
    Gang-->>Allocate: Pass (ValidTaskNum=8 >= MinAvailable=8)

    loop 为每个 Pending Task 分配
        Allocate->>Statement: stmt.Allocate(task, node)
        Note over Statement: 分配 PS-0 到 Node-1
        Allocate->>Statement: stmt.Allocate(task, node)
        Note over Statement: 分配 Worker-0 到 Worker-4 到 Node-2~6
    end

    Note over Allocate: 5 个 Worker 分配后无可用节点

    Allocate->>Gang: JobReady(job)
    Gang->>Gang: CheckTaskReady()
    Note over Gang: PS: 1/1 OK, Worker: 5/7 FAIL
    Gang-->>Allocate: false

    Allocate->>Statement: stmt.Discard()
    Note over Statement: 逆序回滚所有 6 个分配

    Scheduler->>Gang: OnSessionClose()
    Gang->>Gang: schedulableTaskNum() = 6
    Gang->>Gang: unreadyTaskCount = 8 - 6 = 2
    Note over Gang: 更新 PodGroup 状态为 Unschedulable
    Gang->>Gang: Message = "2/8 tasks in gang unschedulable"
```

**解决方案选择：**

1. 增加集群节点资源
2. 降低 `minMember` 为 6（允许 5 个 Worker 即可启动）
3. 减小 Worker Pod 的资源请求
4. 使用 Preempt Action 抢占低优先级 Job 释放资源

---

## 11. Gang 调度与其他 Action 的交互

### 11.1 与 Allocate Action 的交互

```mermaid
flowchart TD
    subgraph allocate_gang["Allocate Action + Gang Plugin"]
        A1["遍历队列和 Job"] --> A2["JobOrderFn 排序 - 未就绪优先"]
        A2 --> A3["为 Job 创建 Statement"]
        A3 --> A4["逐个分配 Task 到节点"]
        A4 --> A5{"ssn.JobReady(job)?"}
        A5 -->|"是"| A6["stmt.Commit() - 提交分配"]
        A5 -->|"否"| A7["stmt.Discard() - 回滚"]
        A7 -->|"下一个 Job"| A2
    end
```

### 11.2 与 Preempt Action 的交互

```mermaid
flowchart TD
    subgraph preempt_gang["Preempt Action + Gang Plugin"]
        P1["JobStarving 检查饥饿 Job"] --> P2["Gang Plugin - IsStarving()"]
        P2 --> P3["为饥饿 Job 寻找 Victim"]
        P3 --> P4["PreemptableFn - 保护 MinAvailable"]
        P4 --> P5["Evict Victim + Pipeline Preemptor"]
        P5 --> P6{"ssn.JobPipelined(job)?"}
        P6 -->|"是"| P7["stmt.Commit()"]
        P6 -->|"否"| P8["stmt.Discard() - 恢复 Victim"]
    end
```

### 11.3 与 Enqueue Action 的交互

Gang Plugin 本身不直接参与 Enqueue，但 `validJobFn` 会在后续 Action 中拦截不合法的 Job。如果一个 Job 的 `ValidTaskNum < MinAvailable`，它在 Allocate 和 Preempt 中都会被跳过。

### 11.4 调试日志级别建议

| 日志级别 | 信息内容 | 适用场景 |
|----------|---------|---------|
| `-v=3` | Commit/Discard 操作、Task 分配/Pipeline 结果 | 日常调试 |
| `-v=4` | Gang jobOrderFn 排序详情、PreemptableFn 结果 | Gang 行为分析 |
| `-v=5` | 进入/离开 Action 的标记日志 | 确认 Action 执行顺序 |

```bash
# 推荐的调试命令
./vc-scheduler \
    --scheduler-conf=/etc/volcano/scheduler.conf \
    --logtostderr \
    -v=4 \
    2>&1 | grep -E "(Gang|IsReady|IsPipelined|Commit|Discard|Starving)"
```
