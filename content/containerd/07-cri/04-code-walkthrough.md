---
title: "CRI - 代码走读"
weight: 04
---


本章深入 containerd CRI 模块的源码实现，通过代码分析理解 CRI 请求的完整处理流程。

## 代码结构

### 目录布局

```
containerd/
├── api/
│   └── services/
│       └── sandbox/
│           └── v1/                    # Sandbox gRPC API
├── internal/
│   └── cri/
│       ├── config/                    # CRI 配置
│       │   └── config.go
│       ├── server/                    # CRI 服务核心实现
│       │   ├── service.go             # CRI Service 入口
│       │   ├── podsandbox/            # Pod Sandbox 相关
│       │   │   ├── controller.go      # Sandbox Controller
│       │   │   ├── sandbox_run.go     # RunPodSandbox
│       │   │   ├── sandbox_stop.go    # StopPodSandbox
│       │   │   └── sandbox_remove.go  # RemovePodSandbox
│       │   ├── container/             # Container 相关
│       │   │   ├── create.go          # CreateContainer
│       │   │   ├── start.go           # StartContainer
│       │   │   └── stop.go            # StopContainer
│       │   ├── images/                # Image 相关
│       │   │   ├── pull.go            # PullImage
│       │   │   └── service.go         # ImageService
│       │   └── streaming/             # Exec/Attach/PortForward
│       │       └── server.go
│       └── store/                     # 状态存储
│           ├── sandbox/               # Sandbox Store
│           │   └── sandbox.go
│           └── container/             # Container Store
│               └── container.go
├── plugins/
│   └── cri/
│       └── cri.go                     # CRI 插件注册
└── core/
    └── sandbox/
        ├── controller.go              # Sandbox Controller 接口
        └── shim_controller.go         # Shim Controller 实现
```

## CRI 插件初始化

### 入口文件

```go
// plugins/cri/cri.go

package cri

import (
    "github.com/containerd/containerd/v2/plugins"
    "github.com/containerd/containerd/v2/internal/cri/config"
    "github.com/containerd/containerd/v2/internal/cri/server"
)

func init() {
    // 注册 CRI gRPC 插件
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
        InitFn: func(ic *plugin.InitContext) (interface{}, error) {
            return initCRIService(ic)
        },
    })
}
```

### 服务初始化

```go
// plugins/cri/cri.go

func initCRIService(ic *plugin.InitContext) (interface{}, error) {
    ctx := ic.Context

    // 1. 加载配置
    cfg := ic.Config.(*config.PluginConfig)

    // 2. 获取 containerd 客户端
    client, err := containerd.New(
        "",
        containerd.WithDefaultNamespace(cfg.ContainerdConfig.Namespace),
        containerd.WithInMemoryServices(ic),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create containerd client: %w", err)
    }

    // 3. 获取依赖的插件
    // 事件服务
    ep, err := ic.GetByID(plugins.EventPlugin, "exchange")
    // Sandbox 控制器
    sbControllers, err := getSandboxControllers(ic)

    // 4. 创建 CRI 服务
    criService, err := server.NewCRIService(
        cfg,
        client,
        getNRICallback(ic),
        server.WithSandboxControllers(sbControllers),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create CRI service: %w", err)
    }

    // 5. 启动服务
    go func() {
        if err := criService.Run(ctx); err != nil {
            log.G(ctx).WithError(err).Fatal("CRI service failed")
        }
    }()

    return criService, nil
}
```

## CRI Service 核心实现

### Service 结构

```go
// internal/cri/server/service.go

// CRIService 实现 RuntimeService 和 ImageService
type CRIService struct {
    // 配置
    config config.Config

    // containerd 客户端
    client *containerd.Client

    // Sandbox 控制器
    sandboxControllers map[string]sandbox.Controller

    // Sandbox 存储
    sandboxStore *sandboxstore.Store

    // 容器存储
    containerStore *containerstore.Store

    // 镜像服务
    imageService images.ImageService

    // CNI 网络管理
    netPlugin cni.CNI

    // Streaming 服务器 (Exec/Attach)
    streamServer streaming.Server

    // 运行时处理器
    runtimeHandlers map[string]RuntimeHandler

    // 事件监控
    eventMonitor *eventMonitor
}

// 确保实现了 CRI 接口
var _ runtime.RuntimeServiceServer = (*CRIService)(nil)
var _ runtime.ImageServiceServer = (*CRIService)(nil)
```

### 服务启动

```go
// internal/cri/server/service.go

func (c *CRIService) Run(ctx context.Context) error {
    log.G(ctx).Info("Start CRI service")

    // 1. 初始化 CNI 网络
    if err := c.initCNINetwork(ctx); err != nil {
        return fmt.Errorf("failed to init CNI network: %w", err)
    }

    // 2. 启动 Streaming 服务器
    if err := c.streamServer.Start(true); err != nil {
        return fmt.Errorf("failed to start streaming server: %w", err)
    }

    // 3. 启动事件监控
    go c.eventMonitor.Run(ctx)

    // 4. 恢复已有的 Sandbox 和容器
    if err := c.recover(ctx); err != nil {
        return fmt.Errorf("failed to recover: %w", err)
    }

    log.G(ctx).Info("CRI service started successfully")
    return nil
}
```

## RunPodSandbox 代码走读

### 入口函数

```go
// internal/cri/server/podsandbox/sandbox_run.go

// RunPodSandbox 创建并启动 Pod Sandbox
func (c *Controller) RunPodSandbox(
    ctx context.Context,
    r *runtime.RunPodSandboxRequest,
) (*runtime.RunPodSandboxResponse, error) {
    config := r.GetConfig()
    log.G(ctx).Infof("RunPodSandbox for %+v", config.GetMetadata())

    // 1. 生成唯一 ID
    id := util.GenerateID()

    // 2. 获取运行时处理器
    ociRuntime, err := c.getSandboxRuntime(config, r.GetRuntimeHandler())
    if err != nil {
        return nil, fmt.Errorf("failed to get sandbox runtime: %w", err)
    }

    // 3. 创建并启动 Sandbox
    sandbox, err := c.runPodSandbox(ctx, id, config, ociRuntime, r.GetRuntimeHandler())
    if err != nil {
        return nil, err
    }

    return &runtime.RunPodSandboxResponse{PodSandboxId: sandbox.ID}, nil
}
```

### 核心实现

```go
// internal/cri/server/podsandbox/sandbox_run.go

func (c *Controller) runPodSandbox(
    ctx context.Context,
    id string,
    config *runtime.PodSandboxConfig,
    ociRuntime criconfig.Runtime,
    runtimeHandler string,
) (*sandboxstore.Sandbox, error) {

    // 1. 创建 Sandbox 元数据
    metadata := sandboxstore.Metadata{
        ID:             id,
        Name:           config.GetMetadata().GetName(),
        Config:         config,
        RuntimeHandler: runtimeHandler,
    }

    // 2. 创建网络命名空间 (非 hostNetwork)
    var netNS *netns.NetNS
    if !hostNetwork(config) {
        netNS, err = netns.NewNetNS(c.config.NetworkPluginConfDir)
        if err != nil {
            return nil, fmt.Errorf("failed to create network namespace: %w", err)
        }
    }

    // 3. 准备 Sandbox 对象
    sandbox := sandboxstore.Sandbox{
        Metadata:  metadata,
        NetNS:     netNS,
        State:     sandboxstore.StateUnknown,
        CreatedAt: time.Now().UTC(),
    }

    // 4. 创建 Sandbox 容器
    // 关键调用：创建 pause 容器
    container, err := c.createSandboxContainer(ctx, &sandbox, ociRuntime)
    if err != nil {
        return nil, fmt.Errorf("failed to create sandbox container: %w", err)
    }

    // 5. 创建 Task (启动 Shim)
    task, err := container.NewTask(ctx,
        containerd.WithTaskAPIEndpoint(ociRuntime.SandboxMode),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create sandbox task: %w", err)
    }

    // 6. 启动 Task
    if err := task.Start(ctx); err != nil {
        return nil, fmt.Errorf("failed to start sandbox task: %w", err)
    }

    // 7. 设置网络
    if netNS != nil {
        result, err := c.setupPodNetwork(ctx, &sandbox)
        if err != nil {
            // 网络设置失败，清理并返回错误
            task.Kill(ctx, syscall.SIGKILL)
            return nil, fmt.Errorf("failed to setup pod network: %w", err)
        }
        sandbox.IP = result.Interfaces[0].IPConfigs[0].IP.String()
    }

    // 8. 更新状态并保存
    sandbox.State = sandboxstore.StateReady
    if err := c.sandboxStore.Add(sandbox); err != nil {
        return nil, err
    }

    return &sandbox, nil
}
```

### 创建 Sandbox 容器

```go
// internal/cri/server/podsandbox/sandbox_run.go

func (c *Controller) createSandboxContainer(
    ctx context.Context,
    sandbox *sandboxstore.Sandbox,
    ociRuntime criconfig.Runtime,
) (containerd.Container, error) {

    // 1. 确保 pause 镜像存在
    image, err := c.ensureSandboxImageExists(ctx, c.config.SandboxImage)
    if err != nil {
        return nil, err
    }

    // 2. 生成 OCI Spec
    spec, err := c.sandboxContainerSpec(
        sandbox.ID,
        sandbox.Config,
        image.Config,
        sandbox.NetNS.GetPath(),
        ociRuntime.PodAnnotations,
    )
    if err != nil {
        return nil, err
    }

    // 3. 准备容器选项
    opts := []containerd.NewContainerOpts{
        // Snapshotter
        containerd.WithSnapshotter(c.config.ContainerdConfig.Snapshotter),
        // 从镜像创建 Snapshot
        containerd.WithNewSnapshot(sandbox.ID, image.Image),
        // OCI Spec
        containerd.WithSpec(spec),
        // 标签
        containerd.WithContainerLabels(sandboxLabels(sandbox)),
        // 运行时
        containerd.WithRuntime(ociRuntime.Type, ociRuntime.Options),
        // 扩展数据 (存储 sandbox 元数据)
        containerd.WithContainerExtension(
            criSandboxExtensionKey,
            &sandbox.Metadata,
        ),
    }

    // 4. 创建容器
    return c.client.NewContainer(ctx, sandbox.ID, opts...)
}
```

## CreateContainer 代码走读

### 入口函数

```go
// internal/cri/server/container/create.go

func (c *criService) CreateContainer(
    ctx context.Context,
    r *runtime.CreateContainerRequest,
) (*runtime.CreateContainerResponse, error) {
    config := r.GetConfig()
    sandboxConfig := r.GetSandboxConfig()

    log.G(ctx).Infof("CreateContainer within sandbox %q for %+v",
        r.GetPodSandboxId(), config.GetMetadata())

    // 1. 获取 Sandbox
    sandbox, err := c.sandboxStore.Get(r.GetPodSandboxId())
    if err != nil {
        return nil, fmt.Errorf("sandbox %q not found: %w", r.GetPodSandboxId(), err)
    }

    // 2. 验证 Sandbox 状态
    if sandbox.State != sandboxstore.StateReady {
        return nil, fmt.Errorf("sandbox %q is not ready", sandbox.ID)
    }

    // 3. 创建容器
    id, err := c.createContainer(ctx, sandbox, config, sandboxConfig)
    if err != nil {
        return nil, err
    }

    return &runtime.CreateContainerResponse{ContainerId: id}, nil
}
```

### 核心实现

```go
// internal/cri/server/container/create.go

func (c *criService) createContainer(
    ctx context.Context,
    sandbox *sandboxstore.Sandbox,
    config *runtime.ContainerConfig,
    sandboxConfig *runtime.PodSandboxConfig,
) (string, error) {

    // 1. 生成容器 ID
    id := util.GenerateID()

    // 2. 获取镜像
    imageRef := config.GetImage().GetImage()
    image, err := c.localResolve(ctx, imageRef)
    if err != nil {
        return "", fmt.Errorf("failed to resolve image %q: %w", imageRef, err)
    }

    // 3. 创建容器元数据
    metadata := containerstore.Metadata{
        ID:        id,
        Name:      config.GetMetadata().GetName(),
        SandboxID: sandbox.ID,
        Config:    config,
        ImageRef:  image.ID,
        LogPath:   c.buildContainerLogPath(config, sandbox.Config),
    }

    // 4. 创建 Snapshot
    snapshotKey := id
    if err := c.createContainerSnapshot(ctx, snapshotKey, image); err != nil {
        return "", err
    }

    // 5. 生成 OCI Spec
    // 关键：设置容器加入 Sandbox 的 Namespace
    spec, err := c.containerSpec(
        id,
        sandbox.ID,
        config,
        sandboxConfig,
        image.Config,
        sandbox.NetNS.GetPath(),
    )
    if err != nil {
        return "", err
    }

    // 6. 创建 containerd 容器
    opts := []containerd.NewContainerOpts{
        containerd.WithSnapshotter(c.config.ContainerdConfig.Snapshotter),
        containerd.WithSnapshot(snapshotKey),
        containerd.WithSpec(spec),
        containerd.WithContainerLabels(containerLabels(config, sandbox.Config)),
        containerd.WithRuntime(sandbox.Runtime.Type, sandbox.Runtime.Options),
        containerd.WithContainerExtension(criContainerExtensionKey, &metadata),
    }

    container, err := c.client.NewContainer(ctx, id, opts...)
    if err != nil {
        return "", err
    }

    // 7. 保存容器状态
    containerObj := containerstore.Container{
        Metadata:  metadata,
        Container: container,
        Status:    containerstore.Status{State: runtime.ContainerState_CONTAINER_CREATED},
    }
    if err := c.containerStore.Add(containerObj); err != nil {
        container.Delete(ctx)
        return "", err
    }

    return id, nil
}
```

### 容器 Spec 生成

```go
// internal/cri/server/container/create.go

func (c *criService) containerSpec(
    id string,
    sandboxID string,
    config *runtime.ContainerConfig,
    sandboxConfig *runtime.PodSandboxConfig,
    imageConfig *imagespec.ImageConfig,
    sandboxNetNSPath string,
) (*oci.Spec, error) {

    // 获取 Sandbox 的 PID (用于加入 Namespace)
    sandboxTask, err := c.getSandboxTask(ctx, sandboxID)
    if err != nil {
        return nil, err
    }
    sandboxPid := sandboxTask.Pid()

    // 构建基础 Spec
    spec := &oci.Spec{
        Version: specs.Version,
        Process: &specs.Process{
            Args: getContainerCommand(config, imageConfig),
            Env:  getContainerEnv(config, imageConfig),
            Cwd:  getContainerWorkDir(config, imageConfig),
        },
        Root: &specs.Root{
            Path:     "rootfs",
            Readonly: config.GetLinux().GetSecurityContext().GetReadonlyRootfs(),
        },
    }

    // 关键：配置 Namespace 共享
    // 容器加入 Sandbox 的 Network、IPC、UTS Namespace
    spec.Linux = &specs.Linux{
        Namespaces: []specs.LinuxNamespace{
            {
                Type: specs.NetworkNamespace,
                Path: fmt.Sprintf("/proc/%d/ns/net", sandboxPid),
            },
            {
                Type: specs.IPCNamespace,
                Path: fmt.Sprintf("/proc/%d/ns/ipc", sandboxPid),
            },
            {
                Type: specs.UTSNamespace,
                Path: fmt.Sprintf("/proc/%d/ns/uts", sandboxPid),
            },
            // Mount 和 PID Namespace 通常是独立的
            {Type: specs.MountNamespace},
            {Type: specs.PIDNamespace},
        },
    }

    // 如果配置了 PID Namespace 共享
    if sharesPidNamespace(sandboxConfig) {
        spec.Linux.Namespaces = append(spec.Linux.Namespaces, specs.LinuxNamespace{
            Type: specs.PIDNamespace,
            Path: fmt.Sprintf("/proc/%d/ns/pid", sandboxPid),
        })
    }

    return spec, nil
}
```

## StartContainer 代码走读

```go
// internal/cri/server/container/start.go

func (c *criService) StartContainer(
    ctx context.Context,
    r *runtime.StartContainerRequest,
) (*runtime.StartContainerResponse, error) {

    // 1. 获取容器
    container, err := c.containerStore.Get(r.GetContainerId())
    if err != nil {
        return nil, err
    }

    // 2. 验证状态
    if container.Status.State != runtime.ContainerState_CONTAINER_CREATED {
        return nil, fmt.Errorf("container %q is not in created state", container.ID)
    }

    // 3. 创建 Task
    // 这会通过 Shim 创建容器进程
    task, err := container.Container.NewTask(ctx,
        cio.NewCreator(cio.WithStdio),
    )
    if err != nil {
        return nil, fmt.Errorf("failed to create task: %w", err)
    }

    // 4. 启动 Task
    if err := task.Start(ctx); err != nil {
        task.Delete(ctx)
        return nil, fmt.Errorf("failed to start task: %w", err)
    }

    // 5. 更新容器状态
    container.Status.State = runtime.ContainerState_CONTAINER_RUNNING
    container.Status.Pid = task.Pid()
    container.Status.StartedAt = time.Now().UnixNano()
    c.containerStore.Update(container)

    // 6. 启动事件监控
    go c.waitContainerExit(ctx, container, task)

    return &runtime.StartContainerResponse{}, nil
}

// 等待容器退出
func (c *criService) waitContainerExit(
    ctx context.Context,
    container *containerstore.Container,
    task containerd.Task,
) {
    exitCh, err := task.Wait(ctx)
    if err != nil {
        log.G(ctx).WithError(err).Error("Failed to wait task")
        return
    }

    // 阻塞等待退出
    exitStatus := <-exitCh

    // 更新容器状态
    container.Status.State = runtime.ContainerState_CONTAINER_EXITED
    container.Status.FinishedAt = time.Now().UnixNano()
    container.Status.ExitCode = int32(exitStatus.ExitCode())
    c.containerStore.Update(container)

    log.G(ctx).Infof("Container %q exited with code %d", container.ID, exitStatus.ExitCode())
}
```

## Exec 代码走读

### Exec 请求处理

```go
// internal/cri/server/container_exec.go

func (c *criService) Exec(
    ctx context.Context,
    r *runtime.ExecRequest,
) (*runtime.ExecResponse, error) {

    // 1. 获取容器
    container, err := c.containerStore.Get(r.GetContainerId())
    if err != nil {
        return nil, err
    }

    // 2. 验证容器状态
    if container.Status.State != runtime.ContainerState_CONTAINER_RUNNING {
        return nil, fmt.Errorf("container %q is not running", container.ID)
    }

    // 3. 生成 Exec URL
    // Streaming Server 会处理实际的 exec 操作
    return c.streamServer.GetExec(r)
}
```

### Streaming Server

```go
// internal/cri/server/streaming/server.go

type streamServer struct {
    config  streaming.Config
    runtime streaming.Runtime
}

// GetExec 生成 Exec 的 URL
func (s *streamServer) GetExec(r *runtime.ExecRequest) (*runtime.ExecResponse, error) {
    // 生成 Token
    token, err := s.genToken(execToken, r)
    if err != nil {
        return nil, err
    }

    // 返回 Streaming URL
    // 格式: https://<streaming-addr>/exec/<token>
    return &runtime.ExecResponse{
        Url: s.buildURL("exec", token),
    }, nil
}

// ServeExec 处理实际的 Exec 请求
func (s *streamServer) ServeExec(
    w http.ResponseWriter,
    r *http.Request,
    req *runtime.ExecRequest,
) {
    // 1. 升级到 WebSocket/SPDY
    streamOpts := &remotecommand.Options{
        Stdin:  req.Stdin,
        Stdout: req.Stdout,
        Stderr: req.Stderr,
        TTY:    req.Tty,
    }

    // 2. 执行 Exec
    ctx := r.Context()
    err := s.runtime.Exec(ctx, req.ContainerId, req.Cmd, streamOpts)
    if err != nil {
        http.Error(w, err.Error(), http.StatusInternalServerError)
    }
}
```

## 调试断点建议

### 关键断点位置

| 场景 | 文件 | 函数 |
|------|------|------|
| Pod 创建 | `internal/cri/server/podsandbox/sandbox_run.go` | `RunPodSandbox` |
| Sandbox 容器创建 | `internal/cri/server/podsandbox/sandbox_run.go` | `createSandboxContainer` |
| 网络设置 | `internal/cri/server/podsandbox/sandbox_run.go` | `setupPodNetwork` |
| 容器创建 | `internal/cri/server/container/create.go` | `CreateContainer` |
| 容器 Spec | `internal/cri/server/container/create.go` | `containerSpec` |
| 容器启动 | `internal/cri/server/container/start.go` | `StartContainer` |
| Exec | `internal/cri/server/container_exec.go` | `Exec` |

### Delve 调试示例

```bash
# 1. 编译带调试信息的 containerd
make GODEBUG=1 binaries

# 2. 启动 Delve
dlv attach $(pidof containerd)

# 3. 设置断点
(dlv) break internal/cri/server/podsandbox/sandbox_run.go:50
(dlv) break internal/cri/server/container/create.go:30

# 4. 继续执行
(dlv) continue

# 5. 触发操作 (在另一个终端)
crictl runp sandbox.json
crictl create <pod_id> container.json sandbox.json

# 6. 查看变量
(dlv) print config
(dlv) print sandbox
```

## 日志调试

### 启用 Debug 日志

```toml
# /etc/containerd/config.toml
[debug]
  level = "debug"
```

### 关键日志位置

```go
// 在代码中添加日志
import "github.com/containerd/log"

func (c *criService) CreateContainer(ctx context.Context, r *runtime.CreateContainerRequest) {
    log.G(ctx).WithFields(log.Fields{
        "sandbox_id":   r.GetPodSandboxId(),
        "container":    r.GetConfig().GetMetadata().GetName(),
    }).Debug("CreateContainer called")

    // ...

    log.G(ctx).WithFields(log.Fields{
        "container_id": id,
        "image":        imageRef,
    }).Info("Container created successfully")
}
```

### 使用 crictl 调试

```bash
# 查看 Pod 详情
crictl inspectp <pod_id>

# 查看容器详情
crictl inspect <container_id>

# 查看容器日志
crictl logs <container_id>

# 实时查看事件
crictl events
```

## 小结

通过代码走读，我们了解了 CRI 的核心实现：

1. **插件注册**：CRI 作为 gRPC 插件注册到 containerd
2. **Sandbox 管理**：通过 Sandbox Controller 管理 Pod 生命周期
3. **Namespace 共享**：容器通过 `/proc/<pid>/ns/` 加入 Sandbox
4. **Streaming**：Exec/Attach 通过独立的 HTTP 服务器处理

关键代码路径：
- Pod 创建: `RunPodSandbox` → `createSandboxContainer` → `setupPodNetwork`
- 容器创建: `CreateContainer` → `createContainerSnapshot` → `containerSpec`
- 容器启动: `StartContainer` → `NewTask` → `Start`

下一章我们将学习 [元数据与事件系统](../08-metadata-events/01-metadata-store.md)。

## 参考资料

- [containerd CRI Source](https://github.com/containerd/containerd/tree/main/internal/cri)
- [CRI API Definition](https://github.com/kubernetes/cri-api)
- [Delve Debugger](https://github.com/go-delve/delve)
