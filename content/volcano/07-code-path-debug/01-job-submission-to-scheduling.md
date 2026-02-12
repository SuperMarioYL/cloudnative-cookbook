---
title: "Job 提交到调度的端到端链路"
weight: 1
---

## 1. 概述

本文档从 DEBUG 视角完整追踪一个 Volcano Job 从 `kubectl apply` 提交到最终 Pod 被 Bind 到 Node 的全链路过程。理解这条链路对于定位 Job 卡住、调度失败、Pod 无法绑定等问题至关重要。

### 1.1 端到端链路总览

```mermaid
flowchart TD
    subgraph phase1["阶段一 - 提交与校验"]
        A["kubectl apply -f job.yaml"] --> B["API Server"]
        B --> C["Webhook Validate"]
        C --> D["Webhook Mutate"]
        D --> E["Job 持久化到 etcd"]
    end

    subgraph phase2["阶段二 - Controller 响应"]
        E --> F["Job Controller Watch"]
        F --> G["initiateJob"]
        G --> H["创建 PodGroup"]
        G --> I["创建 Pods"]
    end

    subgraph phase3["阶段三 - Cache 感知"]
        H --> J["PodGroup Informer"]
        I --> K["Pod Informer"]
        J --> L["addJob / addPodGroup"]
        K --> M["addPod / addTask"]
        L --> N["SchedulerCache 更新"]
        M --> N
    end

    subgraph phase4["阶段四 - 调度周期"]
        N --> O["runOnce 触发"]
        O --> P["OpenSession / Snapshot"]
        P --> Q["Actions 执行"]
        Q --> R["Allocate Action"]
    end

    subgraph phase5["阶段五 - 绑定与同步"]
        R --> S["stmt.Commit / allocate"]
        S --> T["AddBindTask"]
        T --> U["BindFlowChannel"]
        U --> V["processBindTask"]
        V --> W["executePreBind"]
        W --> X["kubeclient.Bind"]
    end

    style phase1 fill:#e3f2fd,stroke:#1565c0
    style phase2 fill:#e8f5e9,stroke:#2e7d32
    style phase3 fill:#fff3e0,stroke:#e65100
    style phase4 fill:#f3e5f5,stroke:#6a1b9a
    style phase5 fill:#fce4ec,stroke:#c62828
```

### 1.2 关键时间节点

| 阶段 | 关键事件 | 预期耗时 | 超时预警 |
|------|---------|---------|---------|
| Webhook | Validate + Mutate | < 100ms | > 500ms |
| Controller | initiateJob + createPodGroup | < 1s | > 5s |
| Cache | Informer 同步 | < 2s | > 10s |
| 调度 | runOnce 一个周期 | < 5s | > 30s |
| 绑定 | processBindTask | < 1s | > 5s |

---

## 2. 阶段一 - kubectl 提交与 Webhook 校验

### 2.1 提交流程

当用户执行 `kubectl apply -f job.yaml` 时，请求首先到达 Kubernetes API Server。API Server 在持久化之前，会调用 Volcano 注册的 Admission Webhook 进行校验和变更。

```mermaid
sequenceDiagram
    participant User as kubectl
    participant API as API Server
    participant WH as Webhook Manager
    participant ETCD as etcd

    User->>API: POST /apis/batch.volcano.sh/v1alpha1/jobs
    API->>WH: ValidatingWebhookConfiguration
    WH->>WH: validateJob(job)
    WH-->>API: Allowed / Denied

    API->>WH: MutatingWebhookConfiguration
    WH->>WH: mutateJob(job)
    WH-->>API: Patched Job

    API->>ETCD: 持久化 Job
    ETCD-->>API: 确认写入
    API-->>User: 201 Created
```

### 2.2 Webhook 校验源码

Webhook 的入口在 `pkg/webhooks/` 目录下。Job 相关的 Webhook Handler 包括 Validate 和 Mutate 两类。

**关键校验逻辑：**

- Job 名称合法性检查
- Task 模板校验（至少一个 Task）
- MinAvailable 合理性（不能超过总 replicas）
- Queue 是否存在
- Plugin 配置合法性

### 2.3 调试断点建议

| 断点位置 | 用途 |
|---------|------|
| Webhook validateJob 入口 | 查看收到的 Job 对象完整字段 |
| Webhook mutateJob 入口 | 查看 Mutate 前的原始 Job |
| Webhook mutateJob 出口 | 对比 Mutate 后的变更内容 |

**klog 日志过滤模式：**

```bash
# 查看 Webhook 相关日志
kubectl logs -n volcano-system <webhook-pod> | grep -E "validate|mutate|admission"
```

---

## 3. 阶段二 - Controller 响应

### 3.1 Job Controller Watch 机制

Job Controller 通过 Informer 机制 Watch `batch.volcano.sh/v1alpha1` 的 Job 资源。当新 Job 被创建时，Controller 的 AddFunc 回调被触发。

```mermaid
flowchart LR
    subgraph informer["Informer 机制"]
        A["API Server"] -->|"Watch"| B["Job Informer"]
        B -->|"AddFunc"| C["enqueueJob"]
        C --> D["workqueue"]
    end

    subgraph worker["Worker 处理"]
        D --> E["processNextWorkItem"]
        E --> F["syncJob"]
        F --> G{"isInitiated?"}
        G -->|"No"| H["initiateJob"]
        G -->|"Yes"| I["initOnJobUpdate"]
    end

    subgraph init["初始化流程"]
        H --> J["initJobStatus"]
        J --> K["pluginOnJobAdd"]
        K --> L["createJobIOIfNotExist"]
        L --> M["createOrUpdatePodGroup"]
        M --> N["创建 Pods"]
    end

    style informer fill:#e8f5e9,stroke:#2e7d32
    style worker fill:#fff3e0,stroke:#e65100
    style init fill:#f3e5f5,stroke:#6a1b9a
```

### 3.2 initiateJob 核心逻辑

源码位于 `pkg/controllers/job/job_controller_actions.go` 第 285-314 行：

```go
// pkg/controllers/job/job_controller_actions.go:285
func (cc *jobcontroller) initiateJob(job *batch.Job) (*batch.Job, error) {
    klog.V(3).Infof("Starting to initiate Job <%s/%s>", job.Namespace, job.Name)
    jobInstance, err := cc.initJobStatus(job)
    if err != nil {
        cc.recorder.Event(job, v1.EventTypeWarning, string(batch.JobStatusError),
            fmt.Sprintf("Failed to initialize job status, err: %v", err))
        return nil, err
    }

    if err := cc.pluginOnJobAdd(jobInstance); err != nil {
        // ...
    }

    newJob, err := cc.createJobIOIfNotExist(jobInstance)
    // ...

    if err := cc.createOrUpdatePodGroup(newJob); err != nil {
        // ...
    }

    return newJob, nil
}
```

### 3.3 syncJob - 创建 Pod 的核心逻辑

源码位于 `pkg/controllers/job/job_controller_actions.go` 第 343 行：

```go
// pkg/controllers/job/job_controller_actions.go:343
func (cc *jobcontroller) syncJob(jobInfo *apis.JobInfo, updateStatus state.UpdateStatusFn) error {
    job := jobInfo.Job
    klog.V(3).Infof("Starting to sync up Job <%s/%s>, current version %d",
        job.Namespace, job.Name, job.Status.Version)

    // ... 初始化检查 ...

    if !isInitiated(job) {
        if job, err = cc.initiateJob(job); err != nil {
            return err
        }
    }
    // ... 创建 Pod 并同步状态 ...
}
```

### 3.4 PodGroup 创建与 Pod 创建的关系

```mermaid
flowchart TD
    A["syncJob 入口"] --> B{"Job 是否已初始化?"}
    B -->|"No"| C["initiateJob"]
    C --> D["initJobStatus"]
    D --> E["pluginOnJobAdd"]
    E --> F["createJobIOIfNotExist"]
    F --> G["createOrUpdatePodGroup"]
    G --> H["PodGroup 创建成功"]

    B -->|"Yes"| I["initOnJobUpdate"]
    I --> J["pluginOnJobUpdate"]
    J --> K["createOrUpdatePodGroup"]

    H --> L{"PodGroup Phase?"}
    L -->|"Pending"| M["等待 Scheduler Enqueue"]
    L -->|"Inqueue/Running"| N["syncTask - 创建 Pod"]

    N --> O["遍历 Job.Spec.Tasks"]
    O --> P["为每个 Task 创建 Pod"]
    P --> Q["设置 OwnerReference 指向 Job"]
    Q --> R["设置 annotation 关联 PodGroup"]

    style C fill:#e8f5e9,stroke:#2e7d32
    style G fill:#fff3e0,stroke:#e65100
    style N fill:#f3e5f5,stroke:#6a1b9a
```

### 3.5 调试断点建议

| 断点位置 | 文件路径 | 用途 |
|---------|---------|------|
| `syncJob` 入口 | `pkg/controllers/job/job_controller_actions.go:343` | 追踪 Job 处理的完整流程 |
| `initiateJob` 入口 | `pkg/controllers/job/job_controller_actions.go:285` | 追踪首次初始化 |
| `createOrUpdatePodGroup` | `pkg/controllers/job/job_controller_actions.go` | 确认 PodGroup 创建 |

**klog 日志过滤模式：**

```bash
# 查看 Job Controller 日志
kubectl logs -n volcano-system <controller-pod> | grep -E "Starting to sync|initiate|PodGroup"
```

---

## 4. 阶段三 - Cache 感知

### 4.1 Informer 事件处理流程

Scheduler Cache 通过多个 Informer 监听 Kubernetes 资源变化，并维护内部状态。

```mermaid
flowchart TD
    subgraph informers["Informer 层"]
        A["Pod Informer"] -->|"AddFunc"| B["AddPod"]
        A -->|"UpdateFunc"| C["UpdatePod"]
        A -->|"DeleteFunc"| D["DeletePod"]

        E["PodGroup Informer"] -->|"AddFunc"| F["AddPodGroupV1beta1"]
        E -->|"UpdateFunc"| G["UpdatePodGroupV1beta1"]

        H["Node Informer"] -->|"AddFunc"| I["AddNode"]
        H -->|"UpdateFunc"| J["UpdateNode"]

        K["Queue Informer"] -->|"AddFunc"| L["AddQueueV1beta1"]
    end

    subgraph cache_ops["Cache 操作"]
        B --> M["addPod"]
        M --> N["NewTaskInfo"]
        N --> O["addTask"]
        O --> P{"NodeName 非空?"}
        P -->|"Yes"| Q["node.AddTask"]
        P -->|"No"| R["仅添加到 Job"]
        O --> S["getOrCreateJob"]
        S --> T["job.AddTaskInfo"]
    end

    subgraph cache_state["Cache 状态"]
        Q --> U["SchedulerCache.Nodes"]
        T --> V["SchedulerCache.Jobs"]
        F --> W["SchedulerCache.Jobs PodGroup"]
        L --> X["SchedulerCache.Queues"]
    end

    style informers fill:#e3f2fd,stroke:#1565c0
    style cache_ops fill:#e8f5e9,stroke:#2e7d32
    style cache_state fill:#fff3e0,stroke:#e65100
```

### 4.2 addTask 核心逻辑

源码位于 `pkg/scheduler/cache/event_handlers.go` 第 220-243 行：

```go
// pkg/scheduler/cache/event_handlers.go:220
func (sc *SchedulerCache) addTask(pi *schedulingapi.TaskInfo) error {
    if len(pi.NodeName) != 0 {
        if _, found := sc.Nodes[pi.NodeName]; !found {
            sc.Nodes[pi.NodeName] = schedulingapi.NewNodeInfo(nil)
            sc.Nodes[pi.NodeName].Name = pi.NodeName
        }

        node := sc.Nodes[pi.NodeName]
        if !isTerminated(pi.Status) {
            if err := node.AddTask(pi); err != nil {
                return err
            }
        }
    }

    job := sc.getOrCreateJob(pi)
    if job != nil {
        job.AddTaskInfo(pi)
    }

    return nil
}
```

### 4.3 AddPod 事件处理

源码位于 `pkg/scheduler/cache/event_handlers.go` 第 391-408 行：

```go
// pkg/scheduler/cache/event_handlers.go:391
func (sc *SchedulerCache) AddPod(obj interface{}) {
    pod, ok := obj.(*v1.Pod)
    if !ok {
        klog.Errorf("Cannot convert to *v1.Pod: %v", obj)
        return
    }

    sc.Mutex.Lock()
    defer sc.Mutex.Unlock()

    err := sc.addPod(pod)
    if err != nil {
        klog.Errorf("Failed to add pod <%s/%s> into cache: %v",
            pod.Namespace, pod.Name, err)
        return
    }
    klog.V(3).Infof("Added pod <%s/%v> into cache.", pod.Namespace, pod.Name)
}
```

### 4.4 getOrCreateJob - Job 与 Task 的关联

源码位于 `pkg/scheduler/cache/event_handlers.go` 第 70-84 行：

```go
// pkg/scheduler/cache/event_handlers.go:70
func (sc *SchedulerCache) getOrCreateJob(pi *schedulingapi.TaskInfo) *schedulingapi.JobInfo {
    if len(pi.Job) == 0 {
        if !slices.Contains(sc.schedulerNames, pi.Pod.Spec.SchedulerName) {
            klog.V(4).Infof("Pod %s/%s is not scheduled by %s, skip creating PodGroup and Job for it in cache.",
                pi.Pod.Namespace, pi.Pod.Name, strings.Join(sc.schedulerNames, ","))
        }
        return nil
    }

    if _, found := sc.Jobs[pi.Job]; !found {
        sc.Jobs[pi.Job] = schedulingapi.NewJobInfo(pi.Job)
    }

    return sc.Jobs[pi.Job]
}
```

### 4.5 调试断点建议

| 断点位置 | 文件路径 | 用途 |
|---------|---------|------|
| `AddPod` | `pkg/scheduler/cache/event_handlers.go:391` | 确认 Pod 被 Cache 捕获 |
| `addTask` | `pkg/scheduler/cache/event_handlers.go:220` | 追踪 Task 添加到 Node/Job |
| `getOrCreateJob` | `pkg/scheduler/cache/event_handlers.go:70` | 确认 Job 在 Cache 中创建 |

**klog 日志过滤模式：**

```bash
# 查看 Cache 事件（V3 级别）
kubectl logs -n volcano-system <scheduler-pod> -v 3 | grep -E "Added pod|Added node|addPodGroup"
```

---

## 5. 阶段四 - 调度周期

### 5.1 调度循环入口

Scheduler 的核心循环位于 `pkg/scheduler/scheduler.go`。`Run()` 方法启动后，会按 `schedulePeriod` 周期性调用 `runOnce()` 方法。

源码 `pkg/scheduler/scheduler.go` 第 91-103 行：

```go
// pkg/scheduler/scheduler.go:91
func (pc *Scheduler) Run(stopCh <-chan struct{}) {
    pc.loadSchedulerConf()
    go pc.watchSchedulerConf(stopCh)
    pc.cache.SetMetricsConf(pc.metricsConf)
    pc.cache.Run(stopCh)
    klog.V(2).Infof("Scheduler completes Initialization and start to run")
    go wait.Until(pc.runOnce, pc.schedulePeriod, stopCh)
    // ...
}
```

### 5.2 runOnce 核心流程

源码 `pkg/scheduler/scheduler.go` 第 107-135 行：

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

    conf.EnabledActionMap = make(map[string]bool)
    for _, action := range actions {
        conf.EnabledActionMap[action.Name()] = true
    }

    ssn := framework.OpenSession(pc.cache, plugins, configurations)
    defer func() {
        framework.CloseSession(ssn)
        metrics.UpdateE2eDuration(metrics.Duration(scheduleStartTime))
    }()

    for _, action := range actions {
        actionStartTime := time.Now()
        action.Execute(ssn)
        metrics.UpdateActionDuration(action.Name(), metrics.Duration(actionStartTime))
    }
}
```

### 5.3 调度周期时序图

```mermaid
sequenceDiagram
    participant S as Scheduler
    participant C as Cache
    participant SS as Session
    participant A as Actions
    participant P as Plugins

    S->>S: runOnce() 开始
    S->>S: Lock - 复制 actions/plugins/configurations
    S->>S: 构建 EnabledActionMap

    S->>C: Snapshot()
    C->>C: Lock - 深拷贝 Jobs/Nodes/Queues
    C-->>SS: ClusterInfo 快照

    S->>SS: OpenSession(cache, plugins, configs)
    SS->>SS: 初始化 Session 数据结构
    SS->>SS: 加载 HyperNode 拓扑
    loop 每个 Plugin
        SS->>P: plugin.OnSessionOpen(ssn)
        P->>SS: 注册 extension points
    end

    loop 每个 Action (enqueue/allocate/backfill/...)
        S->>A: action.Execute(ssn)
        A->>SS: 调用 Plugin hooks
    end

    S->>SS: CloseSession(ssn)
    SS->>SS: JobUpdater.UpdateAll()
    SS->>SS: updateQueueStatus()
    loop 每个 Plugin
        SS->>P: plugin.OnSessionClose(ssn)
    end
```

### 5.4 Action 执行顺序

默认的 Action 执行顺序在 `pkg/scheduler/actions/factory.go` 中注册：

```go
// pkg/scheduler/actions/factory.go:33
func init() {
    framework.RegisterAction(reclaim.New())
    framework.RegisterAction(allocate.New())
    framework.RegisterAction(backfill.New())
    framework.RegisterAction(preempt.New())
    framework.RegisterAction(enqueue.New())
    framework.RegisterAction(shuffle.New())
}
```

实际执行顺序由配置文件决定，默认为：`enqueue` -> `allocate` -> `backfill` -> `reclaim` -> `preempt` -> `shuffle`。

### 5.5 调试断点建议

| 断点位置 | 文件路径 | 用途 |
|---------|---------|------|
| `runOnce` | `pkg/scheduler/scheduler.go:107` | 调度周期入口 |
| `OpenSession` | `pkg/scheduler/framework/framework.go:34` | Session 创建 |
| `Snapshot` | `pkg/scheduler/cache/cache.go:1424` | 快照创建 |
| `action.Execute` | `pkg/scheduler/scheduler.go:132` | 每个 Action 执行 |

---

## 6. 阶段五 - 绑定与状态同步

### 6.1 从 Allocate 到 Bind 的完整链路

```mermaid
flowchart TD
    A["Allocate Action"] --> B["stmt.Allocate(task, node)"]
    B --> C["更新 Session 中 task 状态为 Allocated"]
    C --> D["node.AddTask - 更新 Node 资源"]
    D --> E["触发 AllocateFunc 回调"]

    E --> F{"JobReady?"}
    F -->|"Yes"| G["stmt.Commit()"]
    F -->|"No"| H["stmt.Discard()"]

    G --> I["遍历 operations"]
    I --> J["stmt.allocate(task)"]
    J --> K["ssn.CreateBindContext(task)"]
    K --> L["cache.AddBindTask(bindContext)"]
    L --> M["更新 task 状态为 Binding"]
    M --> N["task 写入 BindFlowChannel"]

    N --> O["processBindTask"]
    O --> P{"达到 batchNum?"}
    P -->|"Yes"| Q["BindTask"]
    P -->|"No / Channel 空"| Q

    Q --> R["executePreBinds"]
    R --> S["Bind - kubeclient.CoreV1.Pods.Bind"]
    S --> T["Pod 绑定到 Node"]

    style A fill:#e3f2fd,stroke:#1565c0
    style G fill:#e8f5e9,stroke:#2e7d32
    style Q fill:#fff3e0,stroke:#e65100
    style T fill:#fce4ec,stroke:#c62828
```

### 6.2 Statement Commit 逻辑

源码位于 `pkg/scheduler/framework/statement.go` 第 418-439 行：

```go
// pkg/scheduler/framework/statement.go:418
func (s *Statement) Commit() {
    klog.V(3).Info("Committing operations ...")
    for _, op := range s.operations {
        op.task.ClearLastTxContext()
        switch op.name {
        case Evict:
            err := s.evict(op.task, op.reason)
            // ...
        case Pipeline:
            s.pipeline(op.task)
        case Allocate:
            err := s.allocate(op.task)
            if err != nil {
                if e := s.unallocate(op.task); e != nil {
                    klog.Errorf("Failed to unallocate task <%v/%v>: %v.",
                        op.task.Namespace, op.task.Name, e)
                }
            }
        }
    }
}
```

### 6.3 AddBindTask 逻辑

源码位于 `pkg/scheduler/cache/cache.go` 第 1286-1328 行：

```go
// pkg/scheduler/cache/cache.go:1286
func (sc *SchedulerCache) AddBindTask(bindContext *BindContext) error {
    klog.V(5).Infof("add bind task %v/%v",
        bindContext.TaskInfo.Namespace, bindContext.TaskInfo.Name)
    sc.Mutex.Lock()
    defer sc.Mutex.Unlock()
    job, task, err := sc.findJobAndTask(bindContext.TaskInfo)
    // ...
    originalStatus := task.Status
    if err := job.UpdateTaskStatus(task, schedulingapi.Binding); err != nil {
        return err
    }
    // ...
    if err := node.AddTask(task); err != nil {
        // rollback ...
    }
    sc.BindFlowChannel <- bindContext
    return nil
}
```

### 6.4 processBindTask 批量绑定

源码位于 `pkg/scheduler/cache/cache.go` 第 1330-1354 行：

```go
// pkg/scheduler/cache/cache.go:1330
func (sc *SchedulerCache) processBindTask() {
    for {
        select {
        case bindContext, ok := <-sc.BindFlowChannel:
            if !ok {
                return
            }
            sc.bindCache = append(sc.bindCache, bindContext)
            if len(sc.bindCache) == sc.batchNum {
                sc.BindTask()
            }
        default:
        }
        if len(sc.BindFlowChannel) == 0 {
            break
        }
    }
    if len(sc.bindCache) == 0 {
        return
    }
    sc.BindTask()
}
```

### 6.5 DefaultBinder.Bind

源码位于 `pkg/scheduler/cache/cache.go` 第 218-239 行：

```go
// pkg/scheduler/cache/cache.go:218
func (db *DefaultBinder) Bind(kubeClient kubernetes.Interface, tasks []*schedulingapi.TaskInfo) map[schedulingapi.TaskID]string {
    errMsg := make(map[schedulingapi.TaskID]string)
    for _, task := range tasks {
        p := task.Pod
        if err := db.kubeclient.CoreV1().Pods(p.Namespace).Bind(context.TODO(),
            &v1.Binding{
                ObjectMeta: metav1.ObjectMeta{
                    Namespace: p.Namespace, Name: p.Name, UID: p.UID,
                    Annotations: p.Annotations,
                },
                Target: v1.ObjectReference{Kind: "Node", Name: task.NodeName},
            },
            metav1.CreateOptions{}); err != nil {
            klog.Errorf("Failed to bind pod <%v/%v> to node %s : %#v",
                p.Namespace, p.Name, task.NodeName, err)
            errMsg[task.UID] = err.Error()
        } else {
            metrics.UpdateTaskScheduleDuration(metrics.Duration(p.CreationTimestamp.Time))
        }
    }
    return errMsg
}
```

### 6.6 调试断点建议

| 断点位置 | 文件路径 | 用途 |
|---------|---------|------|
| `Statement.Commit` | `pkg/scheduler/framework/statement.go:418` | 确认 Commit 触发 |
| `Statement.allocate` | `pkg/scheduler/framework/statement.go:334` | 追踪 Bind 上下文创建 |
| `AddBindTask` | `pkg/scheduler/cache/cache.go:1286` | 追踪 BindFlowChannel 写入 |
| `processBindTask` | `pkg/scheduler/cache/cache.go:1330` | 追踪批量绑定 |
| `DefaultBinder.Bind` | `pkg/scheduler/cache/cache.go:218` | 确认 kubeclient.Bind 调用 |

---

## 7. 调试技巧

### 7.1 klog 日志级别说明

Volcano 使用 klog 进行日志记录，不同级别的日志提供不同粒度的信息：

| klog 级别 | 内容类型 | 使用场景 |
|----------|---------|---------|
| V(2) | 组件启动、配置加载 | 确认启动状态 |
| V(3) | 核心操作日志（Pod 添加、Job 同步、Session 开关） | 日常调试 |
| V(4) | 详细操作日志（调度周期开始/结束、跳过原因） | 定位卡住问题 |
| V(5) | 极详细日志（每个 Node 的资源、HyperNode 详情） | 深度排查 |

### 7.2 推荐的日志级别设置

```yaml
# volcano-scheduler deployment
spec:
  containers:
  - name: volcano-scheduler
    args:
    - --logtostderr
    - --v=4                    # 推荐 debug 时使用 V4
    - -2>&1
```

### 7.3 关键日志模式

```bash
# 1. 追踪某个 Job 的完整生命周期
kubectl logs -n volcano-system <scheduler-pod> | grep "job-name"

# 2. 查看调度周期
kubectl logs -n volcano-system <scheduler-pod> | grep -E "Start scheduling|End scheduling"

# 3. 查看 Session 开关
kubectl logs -n volcano-system <scheduler-pod> | grep -E "Open Session|Close Session"

# 4. 查看 Pod 绑定
kubectl logs -n volcano-system <scheduler-pod> | grep -E "Bindqing Task|bind task"

# 5. 查看 Allocate 决策
kubectl logs -n volcano-system <scheduler-pod> | grep -E "Allocate|allocat"
```

### 7.4 Delve 远程调试配置

```bash
# 1. 使用 dlv 启动 scheduler
dlv exec ./vc-scheduler -- \
  --scheduler-conf=/etc/volcano/scheduler.conf \
  --logtostderr --v=4

# 2. 关键断点设置
break pkg/scheduler/scheduler.go:107        # runOnce 入口
break pkg/scheduler/framework/framework.go:34  # OpenSession
break pkg/scheduler/actions/allocate/allocate.go:119  # Allocate.Execute
break pkg/scheduler/framework/statement.go:418  # Commit
break pkg/scheduler/cache/cache.go:1286     # AddBindTask
```

---

## 8. 常见问题定位

### 8.1 问题定位决策树

```mermaid
flowchart TD
    A["Job 创建后未被调度"] --> B{"Job 是否在 etcd 中?"}
    B -->|"No"| C["检查 Webhook 是否拒绝"]
    B -->|"Yes"| D{"PodGroup 是否创建?"}

    D -->|"No"| E["检查 Job Controller 日志"]
    E --> E1["确认 Controller 是否 Watch 到 Job"]
    E --> E2["检查 initiateJob 是否报错"]

    D -->|"Yes"| F{"PodGroup Phase?"}
    F -->|"Pending"| G["检查 Enqueue Action"]
    G --> G1["确认 Queue 资源是否充足"]
    G --> G2["确认 MinAvailable 是否满足"]

    F -->|"Inqueue"| H{"Pod 是否创建?"}
    H -->|"No"| I["检查 Job Controller syncJob"]
    H -->|"Yes"| J{"Cache 是否感知?"}

    J -->|"No"| K["检查 Scheduler 的 schedulerName 配置"]
    K --> K1["确认 Pod.Spec.SchedulerName 匹配"]

    J -->|"Yes"| L{"调度周期是否处理?"}
    L -->|"No"| M["检查 Job 是否进入 Allocate"]
    L -->|"Yes"| N{"分配结果?"}

    N -->|"所有 Node 被过滤"| O["检查 Predicate 失败原因"]
    N -->|"Queue Overused"| P["检查 Queue 配额与当前使用量"]
    N -->|"Gang 约束不满足"| Q["检查 MinAvailable 与可用资源"]

    style A fill:#fce4ec,stroke:#c62828
    style C fill:#fff3e0,stroke:#e65100
    style O fill:#fff3e0,stroke:#e65100
    style P fill:#fff3e0,stroke:#e65100
    style Q fill:#fff3e0,stroke:#e65100
```

### 8.2 各阶段卡住的排查方法

#### Job 卡在 Pending

```bash
# 检查 PodGroup 状态
kubectl get podgroup -n <namespace> | grep <job-name>

# 检查 Queue 资源
kubectl get queue <queue-name> -o yaml

# 查看 Enqueue Action 日志
kubectl logs -n volcano-system <scheduler-pod> | grep -E "enqueue|Enqueue"
```

#### Job 卡在 Inqueue 但 Pod 未调度

```bash
# 检查 Pod 是否已创建
kubectl get pods -n <namespace> -l volcano.sh/job-name=<job-name>

# 检查 Scheduler Cache 是否感知
kubectl logs -n volcano-system <scheduler-pod> -v 3 | grep "Added pod"

# 检查 Allocate 是否处理该 Job
kubectl logs -n volcano-system <scheduler-pod> -v 3 | grep "Try to allocate"
```

#### Pod 分配了但未绑定

```bash
# 检查 Bind 相关日志
kubectl logs -n volcano-system <scheduler-pod> | grep -E "bind task|Bind|processBindTask"

# 检查 PreBind 是否失败
kubectl logs -n volcano-system <scheduler-pod> | grep "execute preBind"

# 检查 API Server 连通性
kubectl logs -n volcano-system <scheduler-pod> | grep "Failed to bind pod"
```

### 8.3 Cache Dumper 使用

当需要查看 Scheduler Cache 内部完整状态时，可以使用 Cache Dumper：

```bash
# 发送 SIGUSR2 信号触发 Cache Dump
kill -USR2 <scheduler-pid>

# Dump 文件位置由 --cache-dump-file-dir 参数指定
# 默认在 /tmp/ 目录下
```

### 8.4 Prometheus Metrics 关键指标

| Metric 名称 | 类型 | 说明 |
|------------|------|------|
| `e2e_scheduling_duration_seconds` | Histogram | 单次调度周期总耗时 |
| `action_duration_seconds` | Histogram | 每个 Action 的执行耗时 |
| `plugin_scheduling_duration_seconds` | Histogram | 每个 Plugin 的执行耗时 |
| `task_schedule_duration_seconds` | Histogram | Task 从创建到调度的延迟 |
| `e2e_job_scheduling_duration_seconds` | Gauge | Job 端到端调度延迟 |

```bash
# 查看 Prometheus metrics
kubectl port-forward -n volcano-system <scheduler-pod> 8080:8080
curl http://localhost:8080/metrics | grep -E "e2e_scheduling|action_duration"
```

---

## 9. 总结

Job 从提交到调度完成的端到端链路涉及 5 个核心阶段：

1. **Webhook 校验** - 确保 Job 定义合法
2. **Controller 响应** - 创建 PodGroup 和 Pods
3. **Cache 感知** - Informer 将资源变化同步到 Scheduler Cache
4. **调度周期** - runOnce 执行 Action Pipeline，Allocate 分配资源
5. **绑定同步** - 通过 BindFlowChannel 异步批量绑定 Pod 到 Node

理解这条完整链路，结合 klog 日志级别和断点设置，可以高效定位各阶段可能出现的问题。
