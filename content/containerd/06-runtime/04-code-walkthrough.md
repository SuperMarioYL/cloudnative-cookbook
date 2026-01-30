---
title: "Runtime 代码走读"
weight: 04
---


本章深入分析 Runtime 模块的源码实现，帮助你理解容器运行时的内部工作机制。

## 代码结构

```
containerd/
├── core/runtime/
│   ├── runtime.go          # 运行时接口定义
│   ├── task.go             # Task 接口定义
│   └── v2/
│       ├── shim_manager.go # Shim 管理器
│       ├── shim.go         # Shim 客户端
│       └── bundle.go       # Bundle 管理
├── cmd/containerd-shim-runc-v2/
│   ├── main.go             # Shim 入口
│   ├── service.go          # TTRPC 服务
│   ├── process/
│   │   ├── init.go         # Init 进程
│   │   └── exec.go         # Exec 进程
│   └── runc/
│       └── container.go    # runc 封装
└── plugins/services/tasks/
    └── service.go          # gRPC 服务
```

## ShimManager 分析

### 核心结构

```go
// core/runtime/v2/shim_manager.go

type ShimManager struct {
    root           string                 // 持久化目录
    state          string                 // 运行时目录
    containerdAddr string                 // containerd 地址
    shims          sync.Map               // Shim 实例缓存
    events         *exchange.Exchange     // 事件发布器
}

func NewShimManager(ctx context.Context, config *ManagerConfig) (*ShimManager, error) {
    m := &ShimManager{
        root:           config.Root,
        state:          config.State,
        containerdAddr: config.Address,
        events:         config.Events,
    }

    // 恢复已存在的 Shim
    if err := m.loadExistingShims(ctx); err != nil {
        return nil, err
    }

    return m, nil
}
```

### Start 方法

```go
func (m *ShimManager) Start(ctx context.Context, id string, opts runtime.CreateOpts) (shim.Shim, error) {
    // 1. 创建 Bundle
    bundle, err := NewBundle(ctx, m.root, m.state, id, opts.Spec)
    if err != nil {
        return nil, fmt.Errorf("create bundle: %w", err)
    }

    // 2. 确定 Shim 二进制路径
    shimPath, err := m.resolveRuntimePath(opts.Runtime)
    if err != nil {
        return nil, err
    }

    // 3. 启动 Shim 进程
    params, err := m.startShim(ctx, bundle, id, shimPath, opts)
    if err != nil {
        bundle.Delete()
        return nil, err
    }

    // 4. 连接到 Shim
    conn, err := m.connect(ctx, params)
    if err != nil {
        return nil, err
    }

    // 5. 创建 Shim 客户端
    s := &shim{
        bundle: bundle,
        client: NewTaskClient(conn),
    }

    m.shims.Store(id, s)
    return s, nil
}
```

### startShim 方法

```go
func (m *ShimManager) startShim(ctx context.Context, bundle *Bundle, id, shimPath string, opts runtime.CreateOpts) (*BootstrapParams, error) {
    // 构建启动命令
    cmd := exec.Command(shimPath,
        "-namespace", opts.Namespace,
        "-id", id,
        "-address", m.containerdAddr,
        "start",
    )

    // 设置工作目录
    cmd.Dir = bundle.Path
    cmd.Env = append(os.Environ(), "GOMAXPROCS=2")

    // 创建输出管道
    stdout, err := cmd.StdoutPipe()
    if err != nil {
        return nil, err
    }

    // 启动进程
    if err := cmd.Start(); err != nil {
        return nil, err
    }

    // 读取 bootstrap 信息
    var params BootstrapParams
    if err := json.NewDecoder(stdout).Decode(&params); err != nil {
        cmd.Process.Kill()
        cmd.Wait()
        return nil, err
    }

    // 等待 Shim 启动完成
    cmd.Wait()

    return &params, nil
}
```

## Shim 客户端

### shim 结构

```go
// core/runtime/v2/shim.go

type shim struct {
    bundle *Bundle
    client taskAPI.TaskClient
}

func (s *shim) Create(ctx context.Context, opts runtime.CreateOpts) (runtime.Task, error) {
    // 准备挂载点
    rootfs := make([]*types.Mount, len(opts.Rootfs))
    for i, m := range opts.Rootfs {
        rootfs[i] = &types.Mount{
            Type:    m.Type,
            Source:  m.Source,
            Options: m.Options,
        }
    }

    // 发送 Create 请求
    response, err := s.client.Create(ctx, &taskAPI.CreateTaskRequest{
        ID:       opts.TaskID,
        Bundle:   s.bundle.Path,
        Rootfs:   rootfs,
        Terminal: opts.Terminal,
        Stdin:    opts.IO.Stdin,
        Stdout:   opts.IO.Stdout,
        Stderr:   opts.IO.Stderr,
    })
    if err != nil {
        return nil, err
    }

    return &task{
        shim: s,
        id:   opts.TaskID,
        pid:  response.Pid,
    }, nil
}

func (s *shim) Start(ctx context.Context, id string) (uint32, error) {
    response, err := s.client.Start(ctx, &taskAPI.StartRequest{
        ID: id,
    })
    if err != nil {
        return 0, err
    }
    return response.Pid, nil
}
```

## containerd-shim-runc-v2 分析

### 入口点

```go
// cmd/containerd-shim-runc-v2/main.go

func main() {
    if len(os.Args) < 2 {
        fmt.Fprintf(os.Stderr, "usage: containerd-shim-runc-v2 <command>\n")
        os.Exit(1)
    }

    switch os.Args[1] {
    case "start":
        // 启动模式
        if err := shimStart(); err != nil {
            fmt.Fprintf(os.Stderr, "shim start: %v\n", err)
            os.Exit(1)
        }
    case "delete":
        // 删除模式
        if err := shimDelete(); err != nil {
            fmt.Fprintf(os.Stderr, "shim delete: %v\n", err)
            os.Exit(1)
        }
    default:
        // 运行模式（作为 TTRPC 服务器）
        if err := shimRun(); err != nil {
            fmt.Fprintf(os.Stderr, "shim run: %v\n", err)
            os.Exit(1)
        }
    }
}
```

### shimStart 实现

```go
func shimStart() error {
    // 解析参数
    ns := flag.String("namespace", "", "namespace")
    id := flag.String("id", "", "container id")
    address := flag.String("address", "", "containerd address")
    flag.Parse()

    // 创建 socket 路径
    socketPath := filepath.Join(os.Getenv("XDG_RUNTIME_DIR"), "containerd-shim", *ns, *id, "shim.sock")

    // Fork 新进程运行 Shim 服务
    cmd := exec.Command(os.Args[0],
        "-namespace", *ns,
        "-id", *id,
        "-address", *address,
        "-socket", socketPath,
    )

    cmd.SysProcAttr = &syscall.SysProcAttr{
        Setsid: true, // 创建新会话，脱离终端
    }

    if err := cmd.Start(); err != nil {
        return err
    }

    // 输出 bootstrap 信息
    params := BootstrapParams{
        Version:  2,
        Address:  "unix://" + socketPath,
        Protocol: "ttrpc",
    }

    return json.NewEncoder(os.Stdout).Encode(params)
}
```

### TTRPC 服务实现

```go
// cmd/containerd-shim-runc-v2/service.go

type service struct {
    mu         sync.Mutex
    context    context.Context
    events     chan interface{}
    containers map[string]*runc.Container
    processes  map[string]process.Process
}

func (s *service) Create(ctx context.Context, req *taskAPI.CreateTaskRequest) (*taskAPI.CreateTaskResponse, error) {
    s.mu.Lock()
    defer s.mu.Unlock()

    // 创建 runc 容器
    container, err := runc.NewContainer(ctx, s.platform, &runc.CreateOpts{
        ID:       req.ID,
        Bundle:   req.Bundle,
        Rootfs:   req.Rootfs,
        Terminal: req.Terminal,
        Stdin:    req.Stdin,
        Stdout:   req.Stdout,
        Stderr:   req.Stderr,
    })
    if err != nil {
        return nil, err
    }

    s.containers[req.ID] = container

    // 发布事件
    s.events <- &eventstypes.TaskCreate{
        ContainerID: req.ID,
        Pid:         uint32(container.Pid()),
    }

    return &taskAPI.CreateTaskResponse{
        Pid: uint32(container.Pid()),
    }, nil
}

func (s *service) Start(ctx context.Context, req *taskAPI.StartRequest) (*taskAPI.StartResponse, error) {
    container := s.containers[req.ID]
    if container == nil {
        return nil, errdefs.ToGRPCf(errdefs.ErrNotFound, "container not found")
    }

    // 启动容器
    if err := container.Start(ctx); err != nil {
        return nil, err
    }

    // 发布事件
    s.events <- &eventstypes.TaskStart{
        ContainerID: req.ID,
        Pid:         uint32(container.Pid()),
    }

    return &taskAPI.StartResponse{
        Pid: uint32(container.Pid()),
    }, nil
}

func (s *service) Wait(ctx context.Context, req *taskAPI.WaitRequest) (*taskAPI.WaitResponse, error) {
    container := s.containers[req.ID]
    if container == nil {
        return nil, errdefs.ToGRPCf(errdefs.ErrNotFound, "container not found")
    }

    // 等待容器退出
    exit, err := container.Wait(ctx)
    if err != nil {
        return nil, err
    }

    // 发布事件
    s.events <- &eventstypes.TaskExit{
        ContainerID: req.ID,
        Pid:         uint32(container.Pid()),
        ExitStatus:  exit.Status,
        ExitedAt:    timestamppb.New(exit.Timestamp),
    }

    return &taskAPI.WaitResponse{
        ExitStatus: exit.Status,
        ExitedAt:   timestamppb.New(exit.Timestamp),
    }, nil
}
```

## runc 封装

### Container 结构

```go
// cmd/containerd-shim-runc-v2/runc/container.go

type Container struct {
    id       string
    bundle   string
    process  *process.Init
    runtime  *runc.Runc
    platform stdio.Platform
}

func NewContainer(ctx context.Context, platform stdio.Platform, opts *CreateOpts) (*Container, error) {
    // 创建 runc 实例
    r := &runc.Runc{
        Command:      "runc",
        Root:         "/run/containerd/runc",
        PdeathSignal: syscall.SIGKILL,
    }

    // 挂载 rootfs
    if err := mount.All(opts.Rootfs, filepath.Join(opts.Bundle, "rootfs")); err != nil {
        return nil, err
    }

    // 创建 init 进程
    p, err := process.NewInit(ctx, opts.Bundle, &process.CreateConfig{
        ID:       opts.ID,
        Runtime:  r,
        Stdin:    opts.Stdin,
        Stdout:   opts.Stdout,
        Stderr:   opts.Stderr,
        Terminal: opts.Terminal,
    })
    if err != nil {
        return nil, err
    }

    // 调用 runc create
    if err := p.Create(ctx); err != nil {
        return nil, err
    }

    return &Container{
        id:       opts.ID,
        bundle:   opts.Bundle,
        process:  p,
        runtime:  r,
        platform: platform,
    }, nil
}

func (c *Container) Start(ctx context.Context) error {
    return c.process.Start(ctx)
}

func (c *Container) Wait(ctx context.Context) (*Exit, error) {
    return c.process.Wait()
}
```

### Init 进程

```go
// cmd/containerd-shim-runc-v2/process/init.go

type Init struct {
    id       string
    bundle   string
    runtime  *runc.Runc
    pid      int
    status   int
    exitTime time.Time
    waitCh   chan struct{}
}

func (p *Init) Create(ctx context.Context) error {
    // 调用 runc create
    pid, err := p.runtime.Create(ctx, p.id, p.bundle, &runc.CreateOpts{
        IO:      p.io,
        NoPivot: p.noPivot,
    })
    if err != nil {
        return err
    }

    p.pid = pid
    return nil
}

func (p *Init) Start(ctx context.Context) error {
    // 调用 runc start
    return p.runtime.Start(ctx, p.id)
}

func (p *Init) Wait() (*Exit, error) {
    <-p.waitCh

    return &Exit{
        Status:    uint32(p.status),
        Timestamp: p.exitTime,
    }, nil
}
```

## gRPC 服务层

### Task Service

```go
// plugins/services/tasks/service.go

type service struct {
    runtimes map[string]runtime.PlatformRuntime
}

func (s *service) Create(ctx context.Context, req *api.CreateTaskRequest) (*api.CreateTaskResponse, error) {
    // 获取容器信息
    container, err := s.containers.Get(ctx, req.ContainerID)
    if err != nil {
        return nil, err
    }

    // 获取运行时
    rt := s.runtimes[container.Runtime.Name]
    if rt == nil {
        return nil, fmt.Errorf("unknown runtime %q", container.Runtime.Name)
    }

    // 创建 Task
    task, err := rt.Create(ctx, req.ContainerID, runtime.CreateOpts{
        Spec:     container.Spec,
        Rootfs:   fromProtoMounts(req.Rootfs),
        IO:       runtime.IO{
            Stdin:    req.Stdin,
            Stdout:   req.Stdout,
            Stderr:   req.Stderr,
            Terminal: req.Terminal,
        },
    })
    if err != nil {
        return nil, err
    }

    return &api.CreateTaskResponse{
        ContainerID: req.ContainerID,
        Pid:         task.Pid(),
    }, nil
}
```

## 调试技巧

### 使用 ctr 调试

```bash
# 创建并启动容器
ctr run -t docker.io/library/alpine:latest test-container /bin/sh

# 查看 Task 信息
ctr tasks ls

# 查看 Shim 进程
ps aux | grep containerd-shim

# 检查 Shim socket
ls -la /run/containerd/io.containerd.runtime.v2.task/default/test-container/
```

### 关键断点

| 文件 | 函数 | 用途 |
|------|------|------|
| `core/runtime/v2/shim_manager.go` | `Start()` | Shim 启动入口 |
| `core/runtime/v2/shim_manager.go` | `startShim()` | Shim 进程创建 |
| `cmd/containerd-shim-runc-v2/service.go` | `Create()` | 容器创建 |
| `cmd/containerd-shim-runc-v2/service.go` | `Start()` | 容器启动 |
| `cmd/containerd-shim-runc-v2/runc/container.go` | `NewContainer()` | runc 调用 |

### 日志调试

```bash
# containerd 日志
journalctl -u containerd -f

# Shim 日志
cat /run/containerd/io.containerd.runtime.v2.task/default/test-container/log.json
```

## 常见问题

### 问题 1: "shim not found"

```bash
# 检查 Shim 二进制
which containerd-shim-runc-v2

# 检查 PATH
echo $PATH

# 手动测试 Shim
containerd-shim-runc-v2 --help
```

### 问题 2: "OCI runtime create failed"

```bash
# 检查 runc
runc --version

# 检查 config.json
cat /run/containerd/.../config.json | jq .

# 手动运行 runc
cd /run/containerd/.../bundle
runc create test
```

### 问题 3: Shim 连接失败

```bash
# 检查 socket 文件
ls -la /run/containerd/.../shim.sock

# 检查 Shim 进程
pgrep -f "containerd-shim-runc-v2"
```

## 小结

Runtime 代码的关键路径：

1. **ShimManager.Start()**：启动 Shim 入口
2. **startShim()**：Fork Shim 进程
3. **Shim service.Create()**：创建容器
4. **runc.Container.Start()**：启动容器

调试建议：
1. 从 `ShimManager.Start()` 开始跟踪
2. 观察 Shim 进程的启动和通信
3. 跟踪 runc 的调用

下一章我们将学习 [CRI 模块](../07-cri/01-cri-architecture.md)。
