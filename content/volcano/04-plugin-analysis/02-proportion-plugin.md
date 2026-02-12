---
title: "Proportion Plugin 详解"
weight: 2
---

## 概述

Proportion 是 Volcano 调度器中**最核心的队列管理插件**，负责实现队列级别的**加权公平共享**（Weighted Fair-Share）资源分配。它决定了集群中每个 Queue 应该获得多少资源（deserved），以及在调度过程中如何控制资源的分配、回收和抢占。

几乎所有涉及队列资源管控的调度决策都经过 Proportion 插件 -- 它注册了多达 **11 个扩展点**，覆盖了入队、分配、抢占、回收和模拟等全部关键路径。

> **源码参考**：`pkg/scheduler/plugins/proportion/proportion.go`（557 行）

```mermaid
flowchart LR
    cluster["集群总资源\ntotalResource"] --> proportion["Proportion Plugin\n加权公平分配"]
    proportion --> q1["Queue-A\ndeserved=40%"]
    proportion --> q2["Queue-B\ndeserved=35%"]
    proportion --> q3["Queue-C\ndeserved=25%"]

    style proportion fill:#e3f2fd
    style q1 fill:#c8e6c9
    style q2 fill:#c8e6c9
    style q3 fill:#c8e6c9
```

**核心职责**：Deserved 计算（加权迭代公平分配）、队列排序（Priority + share）、分配控制（不超 deserved）、回收判定（超额队列可回收）、入队准入（容量预检）、抢占模拟（CycleState 深拷贝）。

---

## Plugin 结构体与队列属性

### queueAttr - 队列运行时属性

每个参与调度的 Queue 在 Proportion 中维护一份 `queueAttr`，它是整个插件的数据核心：

```go
type queueAttr struct {
    queueID        api.QueueID
    name           string
    weight         int32       // 队列权重，决定资源分配比例
    share          float64     // 主导资源使用率
    deserved       *api.Resource   // 应得资源量（算法计算结果）
    allocated      *api.Resource   // 已分配资源量（实际使用中）
    request        *api.Resource   // 队列中所有 Job 的资源需求总和
    elastic        *api.Resource   // 弹性资源 = sum(job.allocated - job.minAvailable)
    inqueue        *api.Resource   // 已入队 Job 的资源需求
    capability     *api.Resource   // 队列容量上限（用户配置）
    realCapability *api.Resource   // 实际容量上限（受 guarantee 修正）
    guarantee      *api.Resource   // 最低保障资源（用户配置）
}
```

```mermaid
flowchart TB
    subgraph qa["queueAttr 字段关系图"]
        direction TB
        weight["weight\n队列权重"] -->|"影响"| deserved["deserved\n应得资源"]
        guarantee["guarantee\n最低保障"] -->|"下界约束"| deserved
        capability["capability\n容量上限"] -->|"计算"| realCap["realCapability\n实际上限"]
        totalRes["totalResource"] -->|"计算"| realCap
        realCap -->|"上界约束"| deserved
        request["request\n需求总和"] -->|"上界约束"| deserved
        deserved -->|"对比"| allocated["allocated\n已分配"]
        allocated -->|"计算"| share["share\n主导资源使用率"]
        deserved -->|"计算"| share
        elastic["elastic\n弹性资源"] -->|"入队扣减"| inqueue["inqueue\n入队资源"]
    end

    style deserved fill:#e3f2fd
    style share fill:#fff3e0
    style realCap fill:#fce4ec
```

| 字段 | 来源 | 含义 |
|------|------|------|
| `weight` | Queue Spec | 调度权重，数值越大分配的资源比例越高 |
| `deserved` | 算法计算 | 队列应得的资源上限，由迭代算法每轮计算 |
| `allocated` | 实时追踪 | 队列中所有 Running Pod 实际占用的资源 |
| `request` | Job 聚合 | 队列中所有 Job（Allocated + Pending Task）的资源需求总和 |
| `elastic` | Job 聚合 | 各 Job 的弹性资源之和，即 `job.allocated - job.minAvailable` |
| `inqueue` | 入队追踪 | 已处于 Inqueue 或 Running 状态 Job 的最小资源需求 |
| `capability` | Queue Spec | 用户配置的队列资源硬上限 |
| `realCapability` | 算法计算 | 考虑 guarantee 后的实际上限，`<= capability` |
| `guarantee` | Queue Spec | 用户配置的队列最低保障资源 |
| `share` | 算法计算 | 主导资源使用率 = max(allocated[r]/deserved[r]) |

---

## realCapability 计算

`realCapability` 是对用户配置 `capability` 的修正，确保 guarantee 机制正确生效。

**公式**：`realCapability = min(capability, totalResource - totalGuarantee + guarantee)`

`totalResource - totalGuarantee` 是扣除所有队列保障资源后的**可竞争资源**。当前队列可用上限 = 可竞争资源 + 自己的保障资源，再与 `capability` 取较小值。

```mermaid
flowchart LR
    total["totalResource\n100 CPU"] --> sub["减去 totalGuarantee\n100 - 30 = 70 CPU"]
    sub --> add["加上自身 guarantee\n70 + 10 = 80 CPU"]
    add --> min_op{"min()"}
    cap["capability\n60 CPU"] --> min_op
    min_op --> result["realCapability\n60 CPU"]

    style total fill:#e3f2fd
    style result fill:#c8e6c9
    style cap fill:#fff3e0
```

**数值示例**（集群 100 CPU，totalGuarantee = 30 CPU）：

| Queue | guarantee | capability | realCapability |
|-------|-----------|------------|----------------|
| A | 10 CPU | 60 CPU | min(60, 80) = **60 CPU** |
| B | 10 CPU | 无限制 | **80 CPU** |
| C | 10 CPU | 50 CPU | min(50, 80) = **50 CPU** |

> 当 Queue 未设置 `capability` 时，`realCapability` 直接等于 `totalResource - totalGuarantee + guarantee`。

---

## Deserved 计算算法

这是 Proportion 插件的**核心算法**，采用迭代式加权公平分配。每轮迭代将剩余资源按权重比例分配给尚未满足的队列，直到所有队列都被满足或资源耗尽。

### 算法流程图

```mermaid
flowchart TB
    start(["开始"]) --> init["remaining = totalResource\nmeet = 空集"]
    init --> calc_w["计算 totalWeight\n= 未满足队列权重之和"]
    calc_w --> check_w{"totalWeight == 0?"}
    check_w -->|"是"| done(["结束"])
    check_w -->|"否"| save_old["oldRemaining = remaining"]

    save_old --> loop_q["遍历每个未满足的队列"]
    loop_q --> step1["Step 1 - 按权重加份额\ndeserved += remaining * w/W"]
    step1 --> step2["Step 2 - 上界约束\ndeserved = min(deserved, realCapability)"]
    step2 --> step3["Step 3 - 需求约束\ndeserved = min(deserved, request)"]
    step3 --> step4["Step 4 - 保障约束\ndeserved = max(deserved, guarantee)"]
    step4 --> check_meet{"request <= deserved\nOR deserved 未变化?"}
    check_meet -->|"是"| mark["标记为已满足"]
    check_meet -->|"否"| track["记录 increased / decreased"]
    mark --> track
    track --> next_q{"还有下一个队列?"}
    next_q -->|"是"| loop_q
    next_q -->|"否"| update_r["remaining = ExceededPart(\nremaining + decreased, increased)"]
    update_r --> check_r{"remaining 为空\nOR remaining == oldRemaining?"}
    check_r -->|"是"| done
    check_r -->|"否"| calc_w

    style start fill:#e1f5fe
    style done fill:#e8f5e9
    style step1 fill:#e3f2fd
    style step4 fill:#fff3e0
```

每轮四步处理：**加权分配 -> 容量上界截断 -> 需求上界截断 -> 保障下界提升**。满足条件的队列退出迭代，其未消耗的资源在下一轮重新分配。

### 迭代示例

集群 **100 CPU**，三个队列（无 guarantee，realCapability = 100）：

| Queue | weight | request |
|-------|--------|---------|
| A | 2 | 80 CPU |
| B | 3 | 15 CPU |
| C | 5 | 200 CPU |

**第 1 轮**：remaining = 100, totalWeight = 10

| Queue | 加权份额 | req 约束后 | 满足? |
|-------|----------|-----------|-------|
| A | 20 | 20 | 否 |
| B | 30 | **15** | 是 |
| C | 50 | 50 | 否 |

increased = 85, remaining = ExceededPart(100, 85) = **15**

**第 2 轮**：remaining = 15, totalWeight = 7（仅 A=2, C=5）

| Queue | 加权份额 | 累加后 deserved |
|-------|----------|----------------|
| A | 15*2/7 = 4.29 | **24.29** |
| C | 15*5/7 = 10.71 | **60.71** |

remaining = 0，算法结束。最终：**A=24.29, B=15, C=60.71**。B 未使用的 15 CPU 按权重重新分配给了 A 和 C。

**关键设计点**：

1. **迭代收敛**：每轮至少有一个队列被标记为 meet 或 remaining 不变，保证算法必定终止
2. **按维度独立计算**：`MinDimensionResource` 和 `ExceededPart` 都是逐维度（CPU/Memory/GPU）操作，保证每种资源独立满足约束
3. **guarantee 是下界**：即使队列没有任何 request，deserved 也不会低于 guarantee
4. **剩余资源回收**：当一个队列被 request 或 capability 截断后，多余的资源通过 decreased 回流到 remaining 池，在下一轮迭代中重新分配

---

## Share 计算与队列排序

Share 代表队列的**主导资源使用率**：`share = max(allocated[r] / deserved[r])`，取所有资源维度中使用率最高的值。边界处理：`deserved=0 && allocated=0` 时 share=0；`deserved=0 && allocated>0` 时 share=1。

- **share < 1**：队列未用满额度，可继续分配
- **share >= 1**：队列已满或超额

**QueueOrderFn** 采用两级排序：

```mermaid
flowchart TB
    compare(["比较两个 Queue"]) --> prio{"Priority 不同?"}
    prio -->|"是"| by_prio["Priority 高的排前面"]
    prio -->|"否"| share_cmp{"share 比较"}
    share_cmp -->|"lv.share < rv.share"| lv_first["左队列排前面\n（使用率低优先调度）"]
    share_cmp -->|"lv.share > rv.share"| rv_first["右队列排前面"]
    share_cmp -->|"相等"| equal["顺序不变"]

    style compare fill:#e1f5fe
    style by_prio fill:#c8e6c9
    style lv_first fill:#c8e6c9
```

**在同等优先级下，资源使用率越低的队列越优先得到资源**，这是公平调度的核心保障。

---

## 注册的扩展点详解

### OverusedFn 与 AllocatableFn - 分配路径控制

**OverusedFn**（L300-312）：当 `deserved <= allocated` 时返回 true，Allocate Action 会跳过该队列中的所有 Job。

```go
overused := attr.deserved.LessEqual(attr.allocated, api.Zero)
```

**AllocatableFn**（L314-333）：检查两个条件 -- 队列处于 Open 状态，且 `allocated + task.Resreq <= deserved`：

```go
futureUsed := attr.allocated.Clone().Add(candidate.Resreq)
allocatable, _ := futureUsed.LessEqualWithDimensionAndResourcesName(attr.deserved, candidate.Resreq)
```

关键细节：`LessEqualWithDimensionAndResourcesName` 只检查 Task 请求的资源维度，避免不相关维度阻塞调度（如 GPU Task 不会因 Memory 超额被拒绝）。

### ReclaimableFn 与 PreemptiveFn - 回收/抢占路径

**ReclaimableFn**（L278-298）：遍历候选被回收 Task，只从 `allocated > deserved` 的队列中回收。每回收一个 Task 更新模拟 allocated，确保不会过度回收到 deserved 以下：

```go
if !allocated.LessEqual(attr.deserved, api.Zero) {
    allocated.Sub(reclaimee.Resreq)
    victims = append(victims, reclaimee)
}
```

**PreemptiveFn**（L357-361）：复用 `queueAllocatable` 逻辑，只有当队列还有 deserved 额度时才允许通过抢占获取资源。

### JobEnqueueableFn - 入队准入判定

决定 Pending Job 能否进入 Inqueue 状态，是最复杂的扩展点之一。

```mermaid
flowchart TB
    start(["JobEnqueueableFn"]) --> check_open{"Queue 状态 == Open?"}
    check_open -->|"否"| reject1["Reject\n队列未开启"]
    check_open -->|"是"| check_cap{"realCapability\n已设置?"}
    check_cap -->|"否"| permit1["Permit\n无容量限制"]
    check_cap -->|"是"| check_min{"MinResources\n已设置?"}
    check_min -->|"否"| permit2["Permit\n无最小资源要求"]
    check_min -->|"是"| calc["计算 r =\nminReq + allocated + inqueue - elastic"]
    calc --> compare{"r <= realCapability?\n（按 minReq 维度检查）"}
    compare -->|"是"| permit3["Permit\n并将 minReq 加入 inqueue"]
    compare -->|"否"| reject2["Reject\n队列资源配额不足"]

    style start fill:#e1f5fe
    style permit1 fill:#c8e6c9
    style permit2 fill:#c8e6c9
    style permit3 fill:#c8e6c9
    style reject1 fill:#ffcdd2
    style reject2 fill:#ffcdd2
```

核心公式 `r = minReq + allocated + inqueue - elastic` 估算「如果这个 Job 入队，队列最终需要的最小资源总量」。减去 elastic 是因为已有 Job 的弹性部分可以收缩释放。

### EventHandler - 分配/释放事件追踪

每当 Allocate/Reclaim/Preempt 执行一个 Task 的分配或释放时，EventHandler 立刻更新对应队列的 `allocated` 和 `share`。这保证同一调度周期内后续决策基于最新状态。

```go
AllocateFunc:   attr.allocated.Add(event.Task.Resreq); pp.updateShare(attr)
DeallocateFunc: attr.allocated.Sub(event.Task.Resreq); pp.updateShare(attr)
```

### 模拟函数 - Simulate 系列

Preempt Action 需要**模拟**资源变化而不污染真实状态。Proportion 通过 CycleState 机制支持：

1. **PrePredicateFn**：将所有 queueAttr 深拷贝到 Task 的 CycleState
2. **SimulateAllocatableFn**：从 CycleState 克隆状态检查分配额度
3. **SimulateAddTaskFn / SimulateRemoveTaskFn**：在克隆状态上模拟 allocated 增减

每个待调度 Task 有独立的模拟空间，互不影响。这种设计使得 Preempt Action 可以安全地回答「如果抢占某些 Task，目标 Task 能否被调度」的问题。

---

## 资源流转全景图

```mermaid
sequenceDiagram
    participant User as 用户
    participant Enqueue as Enqueue Action
    participant Proportion as Proportion Plugin
    participant Allocate as Allocate Action
    participant Reclaim as Reclaim Action
    participant Node as 集群节点

    Note over Proportion: Session Open - 计算 deserved

    User->>Enqueue: 提交 Job (Pending)
    Enqueue->>Proportion: JobEnqueueableFn(job)
    Proportion-->>Enqueue: Permit / Reject
    Enqueue->>Enqueue: Job -> Inqueue

    Allocate->>Proportion: OverusedFn(queue)
    Proportion-->>Allocate: false (未超额)
    Allocate->>Proportion: AllocatableFn(queue, task)
    Proportion-->>Allocate: true (额度充足)
    Allocate->>Node: Bind Task to Node
    Allocate->>Proportion: EventHandler.AllocateFunc
    Note over Proportion: allocated += task.Resreq

    Reclaim->>Proportion: ReclaimableFn(reclaimer, victims)
    Proportion-->>Reclaim: 超额 Task 列表
    Reclaim->>Node: Evict victim Tasks
    Reclaim->>Proportion: EventHandler.DeallocateFunc
    Note over Proportion: allocated -= task.Resreq
```

```mermaid
flowchart TB
    subgraph session_open["Session Open 阶段"]
        direction TB
        s1["汇总集群 totalResource"] --> s2["计算 totalGuarantee"]
        s2 --> s3["遍历 Job 构建 queueAttr"]
        s3 --> s4["计算 realCapability"]
        s4 --> s5["迭代计算 deserved"]
        s5 --> s6["注册 11 个扩展点"]
    end

    subgraph runtime["调度运行阶段"]
        direction TB
        r1["QueueOrderFn\n队列排序"] --> r2["OverusedFn\n过滤超额队列"]
        r2 --> r3["AllocatableFn\n检查分配额度"]
        r3 --> r4["EventHandler\n实时更新 allocated/share"]
        r4 -->|"反馈"| r1
    end

    subgraph special["特殊路径"]
        direction TB
        sp1["JobEnqueueableFn\n入队准入"] --> sp2["ReclaimableFn\n回收判定"]
        sp2 --> sp3["PreemptiveFn + Simulate\n抢占准入与模拟"]
    end

    session_open --> runtime
    session_open --> special

    style session_open fill:#e3f2fd
    style runtime fill:#c8e6c9
    style special fill:#fff3e0
```

---

## 常见问题

### Q1 - Deserved 和 Allocated 的区别？

**Deserved** 是算法计算的「应得额度」，**Allocated** 是实际占用量。`allocated < deserved` 可继续分配；`allocated > deserved` 可能被回收。

### Q2 - weight 如何影响资源分配？

weight 决定迭代算法中每轮分配比例。A(weight=1) 和 B(weight=3) 无其他约束时，B 的 deserved 是 A 的 3 倍。但如果 B 的 request 很小，多余的 deserved 会在后续迭代中重新分配给 A。

### Q3 - guarantee 和 capability 同时设置时的行为？

`guarantee` 是 deserved 的下界，`capability`（通过 realCapability）是上界。当 `guarantee > capability` 时 guarantee 生效（Step 4 后于 Step 2），但这属于不合理配置。

### Q4 - 为什么 AllocatableFn 只检查 Task 请求的资源维度？

避免不相关维度干扰。例如 GPU 训练任务只请求 GPU 和少量 CPU，即使队列 Memory 维度已超过 deserved，也不应阻止该任务调度。

### Q5 - elastic 资源在入队检查中的作用？

elastic 代表已运行 Job 中可弹性收缩的部分。入队公式减去 elastic 意味着这部分资源可被释放给新 Job，避免过于保守地拒绝入队。

### Q6 - 模拟函数为什么需要深拷贝？

Preempt Action 评估抢占方案时的「假设」不能污染真实状态。CycleState 深拷贝机制确保每个待调度 Task 有独立模拟空间。

### Q7 - 迭代算法为什么能保证收敛？

每轮至少有一个队列加入 meet 集合，或 remaining 不变/为空。队列数量有限且 remaining 单调不增，算法必在有限轮次内终止。

---

## 下一步

- [Gang Plugin 详解](./01-gang-plugin.md) - All-or-Nothing 成组调度
- [DRF Plugin 详解](./03-drf-plugin.md) - Job 级别的 Dominant Resource Fairness
- [Capacity Plugin 详解](./04-capacity-plugin.md) - 层级队列的容量管理
- [Allocate Action 详解](../03-actions-analysis/02-allocate-action.md) - Proportion 扩展点如何被 Action 调用
- [Reclaim Action 详解](../03-actions-analysis/05-reclaim-action.md) - 资源回收流程
