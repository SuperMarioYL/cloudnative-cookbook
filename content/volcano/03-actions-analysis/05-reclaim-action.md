---
title: "Reclaim Action 详解"
weight: 5
---

## 概述

Reclaim 是 Volcano 调度流水线中负责**跨队列资源回收**的 Action。当某个队列的 Job 因资源不足而处于饥饿状态（Starving）时，Reclaim 会从**其他队列**中运行的低优先级 Task 中回收资源，将其腾让给饥饿 Job。Reclaim 是实现队列间公平共享（Fair-Share）的关键机制，与 Proportion Plugin 配合，确保每个队列能获得其应得（Deserved）的资源份额。

> **源码参考**：`pkg/scheduler/actions/reclaim/reclaim.go`

---

## 设计意图

### 为什么需要 Reclaim

在多队列环境下，队列的资源使用存在动态波动。当某个队列暂时空闲时，其他队列可以借用这部分资源；但当空闲队列有新 Job 提交时，需要将借出的资源回收回来。Reclaim 正是处理这种跨队列资源归还的核心机制。

```mermaid
flowchart LR
    subgraph before["资源借用阶段"]
        direction TB
        qA1["Queue A\nDeserved: 40%\nUsed: 20%"]
        qB1["Queue B\nDeserved: 60%\nUsed: 80%"]
        note1["Queue B 借用了\nQueue A 的空闲资源"]
    end

    subgraph after["Reclaim 回收阶段"]
        direction TB
        qA2["Queue A\nDeserved: 40%\nUsed: 40%"]
        qB2["Queue B\nDeserved: 60%\nUsed: 60%"]
        note2["Queue A 提交新 Job\nReclaim 回收资源"]
    end

    before -->|"Queue A 新 Job 到达"| after

    style qA1 fill:#fff3e0
    style qB1 fill:#c8e6c9
    style qA2 fill:#c8e6c9
    style qB2 fill:#c8e6c9
```

**核心价值**：

- **公平性保障**：确保队列能够拿回被借用的资源，不因资源借出而饿死
- **与 Proportion 联动**：Proportion Plugin 计算每个队列的 Deserved 份额，Reclaim 负责在运行时执行回收
- **精确回收**：只回收标记为 `Reclaimable` 的队列中的 Task，避免干扰不可回收的高优先级工作负载

---

## Action 结构体

```go
type Action struct {
    enablePredicateErrorCache bool  // 默认: true, 缓存 Predicate 失败结果
}
```

| 字段 | 默认值 | 说明 |
|------|--------|------|
| `enablePredicateErrorCache` | `true` | 启用 Predicate 错误缓存，避免对相似 Task 重复计算节点过滤 |

配置通过 `parseArguments()` 从 Session 配置中读取 `enablePredicateErrorCache` 参数。

---

## 整体执行流程

Reclaim 的执行分为两个阶段：收集饥饿 Job，然后按队列优先级依次执行资源回收。

```mermaid
flowchart TB
    start(["Execute() 开始"]) --> parse["解析参数\nparseArguments()"]
    parse --> init["初始化优先级队列\nqueues, queueMap\npreemptorsMap, preemptorTasks"]
    init --> phase1["Phase 1 - 收集饥饿 Job"]

    phase1 --> scan{"遍历所有 Job"}
    scan -->|"Pending"| skip_p["跳过"]
    scan -->|"JobValid 失败"| skip_v["跳过"]
    scan -->|"Queue 不存在"| skip_q["跳过并报错"]
    scan -->|"通过校验"| add_q["加入队列映射\n创建 Queue 条目"]
    add_q --> starving{"JobStarving?"}
    starving -->|"是"| collect["加入 preemptorsMap\n收集 Pending 非 Gated Tasks"]
    starving -->|"否"| scan
    collect --> scan
    skip_p --> scan
    skip_v --> scan
    skip_q --> scan

    scan -->|"遍历完成"| phase2["Phase 2 - 队列级回收"]
    phase2 --> q_loop{"Queue 队列非空?"}

    q_loop -->|"是"| pop_q["弹出最高优先级 Queue"]
    pop_q --> overused{"ssn.Overused(queue)?"}
    overused -->|"是"| q_loop
    overused -->|"否"| j_loop{"该 Queue 有饥饿 Job?"}

    j_loop -->|"是"| pop_j["弹出最高优先级 Job\n创建 Statement"]
    pop_j --> t_loop{"Job 仍饥饿且有 Task?"}

    t_loop -->|"是"| pop_t["弹出最高优先级 Task"]
    pop_t --> policy{"PreemptionPolicy\n!= Never?"}
    policy -->|"否"| t_loop
    policy -->|"是"| preemptive{"ssn.Preemptive\n(queue, task)?"}
    preemptive -->|"否"| t_loop
    preemptive -->|"是"| prepred["ssn.PrePredicateFn(task)"]
    prepred -->|"失败"| t_loop
    prepred -->|"通过"| reclaim["reclaimForTask()"]
    reclaim --> t_loop

    t_loop -->|"否"| pipelined{"ssn.JobPipelined?"}
    pipelined -->|"是"| commit["stmt.Commit()"]
    pipelined -->|"否"| discard["stmt.Discard()"]
    commit --> push_q["Queue 放回队列\n（如果还有 Job）"]
    discard --> push_q
    push_q --> q_loop

    j_loop -->|"否"| q_loop
    q_loop -->|"否"| done(["Execute() 结束"])

    style start fill:#e1f5fe
    style done fill:#e8f5e9
    style commit fill:#c8e6c9
    style discard fill:#ffcdd2
    style phase1 fill:#e3f2fd
    style phase2 fill:#e3f2fd
```

### 阶段详解

#### Phase 1 - 收集饥饿 Job（lines 71-103）

遍历 Session 中的所有 Job，进行三级过滤：

1. **状态过滤**：跳过 Pending 状态的 Job（尚未入队，不参与回收）
2. **有效性校验**：`ssn.JobValid(job)` 检查 Job 是否有效
3. **饥饿检测**：`ssn.JobStarving(job)` 由 Gang Plugin 判断 Job 是否因资源不足无法满足最小成员数

对饥饿 Job，收集其所有 Pending 且未被 SchGated 的 Task 作为待调度候选。

#### Phase 2 - 队列级回收（lines 105-172）

按队列优先级循环处理。对每个队列中的每个饥饿 Job：

1. 创建 Statement 事务
2. 逐个处理 Pending Task，检查 PreemptionPolicy、Preemptive 权限、PrePredicate
3. 调用 `reclaimForTask()` 尝试从其他队列回收资源
4. 如果 Job 达到 Pipelined 状态则 Commit，否则 Discard

---

## reclaimForTask 详解

`reclaimForTask()` 是 Reclaim 的核心函数，负责在所有可行节点上寻找可回收的受害者（victims）并驱逐。

```mermaid
flowchart TB
    start["reclaimForTask()"] --> filter["FilterOutUnschedulableAndUnresolvable\n过滤不可调度节点"]
    filter --> pred["PredicateNodes()\n节点 Predicate 过滤"]
    pred --> shard["GetPredicatedNodeByShard()\n按分片整理并展平"]

    shard --> n_loop{"遍历可行节点"}
    n_loop -->|"下一个节点"| collect["收集该节点上的 reclaimees\n条件 - Running\n条件 - Preemptable\n条件 - 不同队列\n条件 - 队列 Reclaimable()"]

    collect --> has_victims{"有 reclaimees?"}
    has_victims -->|"否"| n_loop
    has_victims -->|"是"| reclaimable["ssn.Reclaimable(task, reclaimees)\n获取 victims"]

    reclaimable --> validate["ValidateVictims(task, node, victims)"]
    validate -->|"失败"| n_loop
    validate -->|"通过"| build_pq["BuildVictimsPriorityQueue\n构建受害者优先级队列"]

    build_pq --> init_res["availableResources = node.FutureIdle()\nreclaimed = EmptyResource()"]
    init_res --> evict_loop{"victimsQueue 非空?"}

    evict_loop -->|"是"| pop_v["弹出最低优先级 victim"]
    pop_v --> do_evict["stmt.Evict(victim, 'reclaim')"]
    do_evict -->|"成功"| add_res["reclaimed += victim.Resreq\navailableResources += victim.Resreq"]
    do_evict -->|"失败"| evict_loop
    add_res --> enough{"task.InitResreq\n<= availableResources?"}
    enough -->|"否"| evict_loop
    enough -->|"是"| pipeline["stmt.Pipeline(task, node)"]

    evict_loop -->|"空"| check_final{"资源足够?"}
    check_final -->|"是"| pipeline
    check_final -->|"否"| n_loop

    pipeline -->|"成功"| done["break - 完成"]
    pipeline -->|"失败"| rollback["stmt.UnPipeline(task)\n回滚"]
    rollback --> n_loop

    n_loop -->|"遍历完"| end_fn["返回 - 无法回收"]

    style start fill:#e1f5fe
    style pipeline fill:#c8e6c9
    style rollback fill:#ffcdd2
    style done fill:#e8f5e9
```

### 关键步骤说明

**节点过滤**：先通过 `FilterOutUnschedulableAndUnresolvableNodesForTask` 去除不可调度节点，再用 `PredicateNodes` 对剩余节点进行 Predicate 校验（复用 `PredicateForPreemptAction`），最后按分片整理。

**贪心驱逐**：在每个节点上，按优先级从低到高逐个驱逐 victim，直到释放的资源加上节点当前的 `FutureIdle()` 满足 Task 需求。这是一种贪心策略，尽可能少地驱逐 Task。

**Pipeline 与回滚**：驱逐足够资源后，通过 `stmt.Pipeline(task, node)` 将 Task 标记为 Pipelined。如果 Pipeline 失败，会尝试 `UnPipeline` 回滚。

---

## 跨队列受害者选择

Reclaim 与 Preempt 最大的区别在于受害者的选择范围。Reclaim 严格限制为跨队列回收，并且需要目标队列显式标记为可回收。

```mermaid
flowchart TB
    subgraph node["节点 Node-1 上运行的 Tasks"]
        t1["Task-A\nQueue: prod\nRunning"]
        t2["Task-B\nQueue: dev\nRunning"]
        t3["Task-C\nQueue: test\nRunning"]
        t4["Task-D\nQueue: prod\nRunning"]
        t5["Task-E\nQueue: dev\nCompleted"]
    end

    subgraph filter["受害者过滤条件"]
        f1["Status == Running ?"]
        f2["Preemptable == true ?"]
        f3["不同队列 ?"]
        f4["队列 Reclaimable() ?"]
    end

    subgraph request["请求方"]
        req["Starving Task-X\nQueue: prod"]
    end

    req --> f1
    t1 -->|"Running - 同队列"| f3
    t2 -->|"Running - 不同队列"| f3
    t3 -->|"Running - 不同队列"| f3
    t4 -->|"Running - 同队列"| f3
    t5 -->|"Completed"| f1

    f3 -->|"Task-A - 同队列排除"| excluded1["排除"]
    f3 -->|"Task-D - 同队列排除"| excluded2["排除"]
    f3 -->|"Task-B"| f4
    f3 -->|"Task-C"| f4

    f4 -->|"dev 队列可回收"| candidate1["候选 victim"]
    f4 -->|"test 队列不可回收"| excluded3["排除"]

    style candidate1 fill:#c8e6c9
    style excluded1 fill:#ffcdd2
    style excluded2 fill:#ffcdd2
    style excluded3 fill:#ffcdd2
    style req fill:#e3f2fd
```

### 过滤规则详解

以下是每个节点上收集 reclaimees 的四重过滤条件：

| 序号 | 条件 | 代码 | 说明 |
|------|------|------|------|
| 1 | `Status == Running` | `taskOnNode.Status != api.Running` | 只驱逐正在运行的 Task |
| 2 | `Preemptable == true` | `!taskOnNode.Preemptable` | Task 必须标记为可抢占 |
| 3 | 不同队列 | `j.Queue != job.Queue` | 只回收其他队列的 Task（核心区别） |
| 4 | 队列可回收 | `q.Reclaimable()` | 目标队列必须标记为可回收 |

通过四重过滤后，候选 victims 被传入 `ssn.Reclaimable(task, reclaimees)`，由 Plugin（如 Proportion）进一步筛选，返回最终的受害者列表。

---

## 与 Preempt Action 的对比

Reclaim 和 Preempt 都涉及驱逐已运行的 Task 来腾出资源，但它们的设计目标和工作范围截然不同：

| 维度 | Reclaim | Preempt |
|------|---------|---------|
| **目标范围** | 跨队列（不同队列的 Task） | 同队列内（相同队列的 Task） |
| **设计意图** | 回收被借用的资源，恢复公平份额 | 队列内高优先级 Job 抢占低优先级 Job |
| **Overused 检查** | 跳过 Overused 队列 | 不检查 Overused |
| **Preemptive 检查** | `ssn.Preemptive(queue, task)` 检查请求方权限 | 无此检查 |
| **受害者钩子** | `ssn.Reclaimable()` | `ssn.Preemptable()` |
| **队列可回收检查** | `queue.Reclaimable()` 过滤目标队列 | 无此过滤 |
| **NominatedNode** | 不使用，直接 Pipeline | 支持设置 NominatedNode |
| **拓扑感知模式** | 不支持（顺序遍历） | 支持 Worker Pool 并发 |
| **Pipeline 回滚** | 失败时 UnPipeline 回滚 | 失败时 UnPipeline 回滚 |

```mermaid
flowchart TB
    subgraph reclaim_scope["Reclaim - 跨队列"]
        direction LR
        rq1["Queue A\n(请求方)"] -->|"回收资源"| rq2["Queue B\n(被回收方)"]
        rq1 -->|"回收资源"| rq3["Queue C\n(被回收方)"]
    end

    subgraph preempt_scope["Preempt - 同队列内"]
        direction LR
        pj1["高优先级 Job"] -->|"抢占"| pj2["低优先级 Job"]
    end

    style rq1 fill:#e3f2fd
    style rq2 fill:#fff3e0
    style rq3 fill:#fff3e0
    style pj1 fill:#e3f2fd
    style pj2 fill:#fff3e0
```

---

## 调用的扩展点

| 扩展点 | 用途 | 典型 Plugin |
|--------|------|-------------|
| `QueueOrderFn` | Queue 优先级排序 | proportion, capacity |
| `JobOrderFn` | Job 优先级排序 | priority, gang |
| `TaskOrderFn` | Task 优先级排序 | priority |
| `JobValid` | Job 有效性校验 | gang |
| `JobStarving` | 判断 Job 是否资源饥饿 | gang |
| `Overused` | 判断 Queue 是否超用 | proportion, capacity |
| `Preemptive` | 判断 Queue/Task 是否有权回收 | proportion |
| `PrePredicateFn` | Task 预过滤 | predicates, numaaware |
| `FilterOutUnschedulableAndUnresolvableNodesForTask` | 过滤不可调度节点 | 内置 |
| `PredicateForPreemptAction` | 节点 Predicate（复用 Preempt 的） | predicates |
| `Reclaimable` | 筛选可回收受害者 | proportion, capacity |
| `BuildVictimsPriorityQueue` | 构建受害者优先级队列 | priority |
| `JobPipelined` | 判断 Job 是否达到 Pipeline 状态 | gang |
| `Evict`（via Statement） | 驱逐受害者 Task | 内置 |
| `Pipeline`（via Statement） | 将 Task 标记为 Pipelined | 内置 |

---

## 常见问题

### Q: Reclaim 和 Preempt 会冲突吗？

不会。两者作用范围不同：Reclaim 处理跨队列回收，Preempt 处理同队列内抢占。在调度流水线中，通常 Preempt 先执行（处理队列内优先级），Reclaim 后执行（处理队列间公平性）。两者使用不同的 Plugin Hook（`Preemptable` vs `Reclaimable`），互不干扰。

### Q: 什么决定了一个队列是否 Reclaimable？

`queue.Reclaimable()` 由 Queue 的配置决定。只有显式标记为可回收的队列中的 Task 才会被 Reclaim 驱逐。这允许管理员保护特定队列（如生产队列）不被回收。

### Q: 为什么 Reclaim 跳过 Overused 的队列？

Overused 意味着队列使用的资源已经超过了 Deserved 份额。如果一个队列本身就在超用，它不应该再去回收其他队列的资源。只有资源使用低于应得份额的队列才有资格发起回收。

### Q: Reclaim 的回收是精确到 Deserved 份额吗？

不是精确到份额的。Reclaim 是逐 Task 驱逐的贪心算法：对于每个 Pending Task，在节点上找到足够的 victims 释放资源即可。`ssn.Reclaimable()` 由 Proportion Plugin 实现，会综合考虑双方队列的 Deserved/Allocated 来决定哪些 Task 可以被回收。

### Q: Pipeline 失败后的 UnPipeline 回滚是如何工作的？

当 `stmt.Pipeline(task, node)` 失败时，Reclaim 会调用 `stmt.UnPipeline(task)` 尝试将 Task 状态恢复到 Pipeline 之前。如果回滚也失败，会记录错误日志但继续处理其他节点。已经通过 `stmt.Evict()` 驱逐的 victims 不会被回滚（它们已经在 Statement 中记录），最终由 Statement 的 Commit 或 Discard 统一处理。

---

## 下一步

- [Shuffle Action](./06-shuffle-action.md) -- 随机打散避免调度热点
- [Statement 与绑定](../02-scheduler-deep-dive/06-statement-and-binding.md) -- Commit/Discard 事务机制
