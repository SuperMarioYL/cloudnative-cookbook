---
title: "CRI - 架构与接口设计"
weight: 01
---


CRI (Container Runtime Interface) 是 Kubernetes 定义的容器运行时接口标准。containerd 通过内置的 CRI 插件原生支持 Kubernetes，成为 Kubernetes 集群中最常用的容器运行时。

## CRI 概述

### 什么是 CRI

CRI 是 Kubernetes 1.5 引入的接口标准，将 kubelet 与底层容器运行时解耦：

```mermaid
flowchart TB
    subgraph k8s["Kubernetes 组件"]
        kubelet["kubelet"]
    end

    subgraph cri["CRI 接口层"]
        grpc["gRPC Protocol"]
        runtime_svc["RuntimeService"]
        image_svc["ImageService"]
    end

    subgraph runtimes["容器运行时"]
        containerd["containerd"]
        crio["CRI-O"]
        dockershim["dockershim<br/>(已废弃)"]
    end

    kubelet --> grpc
    grpc --> runtime_svc
    grpc --> image_svc
    runtime_svc --> runtimes
    image_svc --> runtimes
```

### CRI 设计目标

1. **解耦**：kubelet 不依赖特定运行时实现
2. **标准化**：统一的 gRPC 接口定义
3. **可扩展**：支持多种运行时实现
4. **Pod 优先**：以 Pod 为核心抽象

## CRI 服务接口

### 两大核心服务

```mermaid
flowchart LR
    subgraph cri_api["CRI API"]
        subgraph runtime["RuntimeService"]
            pod_ops["PodSandbox 操作"]
            container_ops["Container 操作"]
            exec_ops["Exec/Attach/PortForward"]
            status_ops["Status/Version"]
        end

        subgraph image["ImageService"]
            pull["PullImage"]
            list_img["ListImages"]
            remove_img["RemoveImage"]
            img_status["ImageStatus"]
        end
    end

    kubelet["kubelet"] --> runtime
    kubelet --> image
```

### RuntimeService 接口

```protobuf
// api/services/runtime/v1/api.proto

service RuntimeService {
    // PodSandbox 操作
    rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse);
    rpc StopPodSandbox(StopPodSandboxRequest) returns (StopPodSandboxResponse);
    rpc RemovePodSandbox(RemovePodSandboxRequest) returns (RemovePodSandboxResponse);
    rpc PodSandboxStatus(PodSandboxStatusRequest) returns (PodSandboxStatusResponse);
    rpc ListPodSandbox(ListPodSandboxRequest) returns (ListPodSandboxResponse);

    // Container 操作
    rpc CreateContainer(CreateContainerRequest) returns (CreateContainerResponse);
    rpc StartContainer(StartContainerRequest) returns (StartContainerResponse);
    rpc StopContainer(StopContainerRequest) returns (StopContainerResponse);
    rpc RemoveContainer(RemoveContainerRequest) returns (RemoveContainerResponse);
    rpc ListContainers(ListContainersRequest) returns (ListContainersResponse);
    rpc ContainerStatus(ContainerStatusRequest) returns (ContainerStatusResponse);

    // 交互操作
    rpc ExecSync(ExecSyncRequest) returns (ExecSyncResponse);
    rpc Exec(ExecRequest) returns (ExecResponse);
    rpc Attach(AttachRequest) returns (AttachResponse);
    rpc PortForward(PortForwardRequest) returns (PortForwardResponse);

    // 状态查询
    rpc ContainerStats(ContainerStatsRequest) returns (ContainerStatsResponse);
    rpc ListContainerStats(ListContainerStatsRequest) returns (ListContainerStatsResponse);
    rpc UpdateRuntimeConfig(UpdateRuntimeConfigRequest) returns (UpdateRuntimeConfigResponse);
    rpc Status(StatusRequest) returns (StatusResponse);
}
```

### ImageService 接口

```protobuf
service ImageService {
    // 镜像列表
    rpc ListImages(ListImagesRequest) returns (ListImagesResponse);

    // 镜像状态
    rpc ImageStatus(ImageStatusRequest) returns (ImageStatusResponse);

    // 拉取镜像
    rpc PullImage(PullImageRequest) returns (PullImageResponse);

    // 删除镜像
    rpc RemoveImage(RemoveImageRequest) returns (RemoveImageResponse);

    // 镜像文件系统信息
    rpc ImageFsInfo(ImageFsInfoRequest) returns (ImageFsInfoResponse);
}
```

## containerd CRI 架构

### 整体架构

```mermaid
flowchart TB
    subgraph external["外部组件"]
        kubelet["kubelet"]
    end

    subgraph containerd_cri["containerd CRI 插件"]
        cri_server["CRI Server"]

        subgraph services["服务层"]
            runtime_service["RuntimeService"]
            image_service["ImageService"]
        end

        subgraph controllers["控制器层"]
            sandbox_ctrl["Sandbox Controller"]
            container_ctrl["Container Controller"]
        end

        subgraph helpers["辅助组件"]
            cni_mgr["CNI Manager"]
            streaming["Streaming Server"]
            image_mgr["Image Manager"]
        end
    end

    subgraph containerd_core["containerd 核心"]
        tasks["Tasks Service"]
        images["Images Service"]
        containers["Containers Service"]
        snapshots["Snapshots Service"]
        content["Content Service"]
    end

    subgraph external_runtime["外部运行时"]
        shim["containerd-shim"]
        runc["runc"]
        cni["CNI Plugins"]
    end

    kubelet --> |"gRPC"| cri_server
    cri_server --> services
    runtime_service --> controllers
    image_service --> image_mgr
    sandbox_ctrl --> cni_mgr
    sandbox_ctrl --> tasks
    container_ctrl --> tasks
    image_mgr --> images
    image_mgr --> content
    tasks --> shim --> runc
    cni_mgr --> cni
```

### CRI 插件注册

```go
// plugins/cri/cri.go

func init() {
    plugin.Register(&plugin.Registration{
        Type:   plugins.GRPCPlugin,
        ID:     "cri",
        Config: &config.PluginConfig{},
        Requires: []plugin.Type{
            plugins.EventPlugin,
            plugins.ServicePlugin,
            plugins.LeasePlugin,
            plugins.SandboxControllerPlugin,
            plugins.NFDPlugin,
            plugins.WarningPlugin,
        },
        InitFn: initCRIService,
    })
}

func initCRIService(ic *plugin.InitContext) (interface{}, error) {
    // 获取依赖的插件
    // 创建 CRI 服务实例
    // 注册 gRPC 服务
}
```

### CRI Server 实现

```go
// internal/cri/server/service.go

// CRIService 实现 CRI 接口
type CRIService struct {
    // 配置
    config criconfig.Config

    // 核心服务
    client *containerd.Client

    // Sandbox 控制器
    sandboxControllers map[string]sandbox.Controller

    // 镜像服务
    imageService imageservice.Service

    // CNI 网络管理
    netPlugin cni.CNI

    // Streaming 服务器
    streamServer streaming.Server

    // 事件监控
    eventMonitor *eventMonitor

    // 运行时处理器
    runtimeHandlers map[string]RuntimeHandler
}

// RuntimeService 方法由 CRIService 实现
var _ runtime.RuntimeServiceServer = &CRIService{}

// ImageService 方法由 CRIService 实现
var _ runtime.ImageServiceServer = &CRIService{}
```

## Pod 与 Sandbox 映射

### Kubernetes Pod 到 containerd

```mermaid
flowchart TB
    subgraph k8s_pod["Kubernetes Pod"]
        pod_spec["PodSpec"]
        container1["Container 1"]
        container2["Container 2"]
        init_container["Init Container"]
    end

    subgraph containerd_mapping["containerd 映射"]
        sandbox["Sandbox<br/>(pause 容器)"]
        task1["Task 1"]
        task2["Task 2"]
        init_task["Init Task"]
    end

    subgraph resources["共享资源"]
        netns["Network Namespace"]
        ipc["IPC Namespace"]
        uts["UTS Namespace"]
        volumes["Volumes"]
    end

    pod_spec --> sandbox
    container1 --> task1
    container2 --> task2
    init_container --> init_task

    sandbox --> resources
    task1 -.-> |"共享"| resources
    task2 -.-> |"共享"| resources
    init_task -.-> |"共享"| resources
```

### Sandbox 元数据结构

```go
// internal/cri/store/sandbox/metadata.go

// Metadata 是 Sandbox 的元数据
type Metadata struct {
    // ID 是 Sandbox 的唯一标识
    ID string

    // Name 是 Sandbox 名称
    Name string

    // Config 是创建时的配置
    Config *runtime.PodSandboxConfig

    // RuntimeHandler 运行时处理器名称
    RuntimeHandler string

    // CNIResult CNI 网络配置结果
    CNIResult *cni.Result

    // ProcessLabel SELinux 进程标签
    ProcessLabel string

    // NetNS 网络命名空间路径
    NetNSPath string

    // State 当前状态
    State StateType
}

// StateType Sandbox 状态
type StateType int

const (
    StateUnknown StateType = iota
    StateReady
    StateNotReady
)
```

## 容器状态管理

### 容器生命周期

```mermaid
stateDiagram-v2
    [*] --> Created: CreateContainer()
    Created --> Running: StartContainer()
    Running --> Exited: 进程退出
    Running --> Exited: StopContainer()
    Exited --> [*]: RemoveContainer()

    note right of Created
        容器已创建
        rootfs 已准备
        未启动进程
    end note

    note right of Running
        主进程运行中
        可执行 Exec
    end note

    note right of Exited
        进程已退出
        保留日志和元数据
    end note
```

### 容器元数据

```go
// internal/cri/store/container/metadata.go

// Metadata 容器元数据
type Metadata struct {
    // ID 容器唯一标识
    ID string

    // Name 容器名称
    Name string

    // SandboxID 所属 Sandbox
    SandboxID string

    // Config 容器配置
    Config *runtime.ContainerConfig

    // ImageRef 镜像引用
    ImageRef string

    // LogPath 日志路径
    LogPath string

    // StopSignal 停止信号
    StopSignal string
}

// Status 容器状态
type Status struct {
    // Pid 主进程 PID
    Pid uint32

    // CreatedAt 创建时间
    CreatedAt int64

    // StartedAt 启动时间
    StartedAt int64

    // FinishedAt 结束时间
    FinishedAt int64

    // ExitCode 退出码
    ExitCode int32

    // Reason 退出原因
    Reason string

    // Message 额外信息
    Message string
}
```

## 请求处理流程

### RunPodSandbox 流程

```mermaid
sequenceDiagram
    participant kubelet
    participant CRI as CRI Server
    participant SandboxCtrl as Sandbox Controller
    participant CNI as CNI Manager
    participant Shim
    participant runc

    kubelet->>CRI: RunPodSandbox(config)

    CRI->>CRI: 生成 Sandbox ID
    CRI->>CRI: 准备 Sandbox 元数据

    CRI->>SandboxCtrl: Create(id, config)
    SandboxCtrl->>Shim: 启动 Shim 进程
    Shim->>runc: 创建 pause 容器
    runc-->>Shim: 容器已创建
    Shim-->>SandboxCtrl: Sandbox 已创建

    CRI->>CNI: Setup(podName, podNS, id)
    CNI->>CNI: 创建 veth pair
    CNI->>CNI: 配置 IP 地址
    CNI-->>CRI: CNI Result

    CRI->>CRI: 保存 Sandbox 状态

    CRI-->>kubelet: RunPodSandboxResponse(id)
```

### CreateContainer 流程

```mermaid
sequenceDiagram
    participant kubelet
    participant CRI as CRI Server
    participant ImageSvc as Image Service
    participant ContainerCtrl as Container Controller
    participant Snapshots
    participant Shim

    kubelet->>CRI: CreateContainer(sandbox_id, config)

    CRI->>CRI: 验证 Sandbox 存在

    CRI->>ImageSvc: GetImage(image_ref)
    ImageSvc-->>CRI: Image 信息

    CRI->>Snapshots: Prepare(container_id, image_chain_id)
    Snapshots-->>CRI: Mount 信息

    CRI->>CRI: 生成 OCI Spec

    CRI->>ContainerCtrl: Create(spec, mounts)
    ContainerCtrl->>Shim: CreateTask (不启动)
    Shim-->>ContainerCtrl: Task 已创建
    ContainerCtrl-->>CRI: Container 已创建

    CRI->>CRI: 保存 Container 状态

    CRI-->>kubelet: CreateContainerResponse(id)
```

## Streaming 服务

### Exec/Attach/PortForward

这些操作需要建立长连接，CRI 使用 Streaming Server 处理：

```mermaid
flowchart TB
    subgraph kubelet_side["kubelet 侧"]
        kubelet["kubelet"]
        api_server["API Server"]
    end

    subgraph cri_side["CRI 侧"]
        cri_server["CRI Server"]
        streaming["Streaming Server<br/>(HTTP/WebSocket)"]
    end

    subgraph container_side["容器侧"]
        shim["Shim"]
        container["Container Process"]
    end

    kubelet --> |"1. Exec Request"| cri_server
    cri_server --> |"2. 返回 URL"| kubelet
    kubelet --> |"3. 通知"| api_server
    api_server --> |"4. 连接"| streaming
    streaming --> |"5. TTRPC"| shim
    shim --> |"6. I/O"| container
```

### Streaming Server 实现

```go
// internal/cri/server/streaming.go

// StreamServer 处理流式请求
type StreamServer interface {
    // GetExec 获取 Exec URL
    GetExec(*runtimeapi.ExecRequest) (*runtimeapi.ExecResponse, error)

    // GetAttach 获取 Attach URL
    GetAttach(*runtimeapi.AttachRequest) (*runtimeapi.AttachResponse, error)

    // GetPortForward 获取 PortForward URL
    GetPortForward(*runtimeapi.PortForwardRequest) (*runtimeapi.PortForwardResponse, error)

    // Start 启动服务器
    Start(bool) error
}

// 执行 Exec 请求
func (s *streamServer) serveExec(req *runtimeapi.ExecRequest, stream io.ReadWriteCloser) error {
    // 1. 获取容器
    // 2. 创建 exec 进程
    // 3. 连接 I/O 流
    // 4. 等待完成
}
```

## RuntimeHandler 多运行时

### 运行时配置

containerd 支持配置多个运行时处理器：

```toml
# /etc/containerd/config.toml

[plugins."io.containerd.cri.v1.runtime"]
  [plugins."io.containerd.cri.v1.runtime".containerd]
    default_runtime_name = "runc"

    [plugins."io.containerd.cri.v1.runtime".containerd.runtimes]
      [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.runc]
        runtime_type = "io.containerd.runc.v2"

      [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.kata]
        runtime_type = "io.containerd.kata.v2"

      [plugins."io.containerd.cri.v1.runtime".containerd.runtimes.gvisor]
        runtime_type = "io.containerd.runsc.v1"
```

### RuntimeClass 使用

```yaml
# Kubernetes RuntimeClass
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata

---
# Pod 使用特定运行时
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  runtimeClassName: kata
  containers:
  - name: nginx
    image: nginx
```

### 运行时选择流程

```mermaid
flowchart TB
    subgraph request["RunPodSandbox 请求"]
        config["PodSandboxConfig"]
        runtime_handler["RuntimeHandler: kata"]
    end

    subgraph cri_processing["CRI 处理"]
        lookup["查找 RuntimeHandler"]
        get_config["获取运行时配置"]
        create_sandbox["创建 Sandbox"]
    end

    subgraph runtimes["可用运行时"]
        runc["runc<br/>(default)"]
        kata["kata"]
        gvisor["gvisor"]
    end

    request --> lookup
    lookup --> get_config
    get_config --> |"handler=kata"| kata
    kata --> create_sandbox
```

## 小结

containerd CRI 插件是 Kubernetes 与 containerd 之间的桥梁：

1. **标准接口**：实现 CRI RuntimeService 和 ImageService
2. **Pod 映射**：将 Pod 映射为 Sandbox + Containers
3. **多运行时**：支持 runc、kata、gvisor 等
4. **流式操作**：通过 Streaming Server 支持 Exec/Attach

理解 CRI 架构对于：
- Kubernetes 容器运行时调试
- 自定义运行时开发
- 性能优化和问题排查

下一节我们将深入学习 [Pod Sandbox 实现](./02-pod-sandbox.md)。

## 参考资料

- [CRI Specification](https://github.com/kubernetes/cri-api)
- [containerd CRI Plugin](https://github.com/containerd/containerd/tree/main/internal/cri)
- [Kubernetes Runtime Class](https://kubernetes.io/docs/concepts/containers/runtime-class/)
