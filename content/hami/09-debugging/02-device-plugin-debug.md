---
title: "Device Plugin 调试指南"
weight: 2
---


## 概述

HAMi Device Plugin 是运行在每个 GPU 节点上的 DaemonSet 组件，负责向 Kubelet 注册 GPU 设备、处理 Allocate 请求并维护节点注解。本文档系统性地介绍 Device Plugin 相关问题的调试方法和常见故障排查流程。

**核心源码路径**:
- `pkg/device-plugin/nvidiadevice/nvinternal/plugin/server.go` - gRPC 服务与 Allocate 逻辑
- `pkg/device-plugin/nvidiadevice/nvinternal/plugin/register.go` - 设备注册与节点注解
- `pkg/device/nvidia/device.go` - 设备定义与常量

---

## 1. Device Plugin 日志分析

### 1.1 关键日志级别

HAMi Device Plugin 使用 `klog` 日志库，日志级别通过 `-v` 参数控制:

| 级别 | 含义 | 典型内容 |
|------|------|----------|
| `-v=0` | Error/Fatal | 致命错误、panic |
| `-v=1` | Info | 启动、注册、分配的关键事件 |
| `-v=3` | Warning | 非致命警告（如 MIG 降级） |
| `-v=4` | Debug 基础 | 指标采集细节、注解详情 |
| `-v=5` | Debug 详细 | 逐设备信息、UUID 匹配等 |

### 1.2 正常启动日志序列

一个健康的 Device Plugin 启动应包含以下关键日志（按时间顺序）:

```
# 1. 配置加载
I Initializing metrics for scheduler
I reading config= ... resourceName ... configfile=

# 2. gRPC 服务启动
I Starting GRPC server for 'nvidia.com/gpu'
I Starting to serve 'nvidia.com/gpu' on /var/lib/kubelet/device-plugins/nvidia-gpu.sock

# 3. Kubelet 注册
I Registered device plugin for 'nvidia.com/gpu' with Kubelet

# 4. 设备注册到节点注解
I start working on the devices
I nvml registered device id=0, memory=81920, type=NVIDIA-A100, numa=0
I patch node with the following annos ...
I Successfully registered annotation. Next check in 30s seconds...

# 5. 健康检查启动
I Starting WatchAndRegister
```

### 1.3 异常日志模式

**NVML 初始化失败**:
```
E nvml Init err: ...
```
- 原因: NVIDIA 驱动未安装或版本不兼容
- 排查: 检查 `nvidia-smi` 是否可用

**设备注册失败**:
```
E get node error ...
E patch node error ...
```
- 原因: RBAC 权限不足或节点名不匹配
- 排查: 检查 ServiceAccount 权限和 `NODE_NAME` 环境变量

**Allocate 失败**:
```
E device number not matched
E invalid allocation request for 'nvidia.com/gpu': unknown device: ...
```
- 原因: 调度器分配的设备数与实际请求不一致
- 排查: 检查 Pod 注解和调度器日志

---

## 2. 节点注解检查

### 2.1 注解格式

Device Plugin 通过 `WatchAndRegister()` 每 30 秒更新一次节点注解。关键注解包括:

| 注解 Key | 说明 |
|----------|------|
| `hami.io/node-nvidia-register` | 设备注册信息（编码格式） |
| `hami.io/node-handshake` | 节点握手时间戳 |
| `hami.io/node-nvidia-score` | GPU 拓扑评分（可选） |
| `hami.io/mutex.lock` | 节点分配互斥锁 |

### 2.2 检查注解的方法

```bash
# 查看所有 HAMi 相关注解
kubectl get node <node-name> -o json | jq '.metadata.annotations | to_entries[] | select(.key | startswith("hami"))'

# 查看设备注册信息
kubectl get node <node-name> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}'

# 查看握手状态
kubectl get node <node-name> -o jsonpath='{.metadata.annotations.hami\.io/node-handshake}'

# 查看节点锁状态
kubectl get node <node-name> -o jsonpath='{.metadata.annotations.hami\.io/mutex\.lock}'
```

### 2.3 注解内容解析

`hami.io/node-nvidia-register` 包含编码后的设备列表，解码后每个设备包含:

- **ID**: GPU UUID
- **Index**: GPU 索引号
- **Count**: 设备分片数量（`DeviceSplitCount`）
- **Devmem**: 注册显存（MB），可能经过 `DeviceMemoryScaling` 调整
- **Devcore**: 设备核心百分比（默认 100，可经过 `DeviceCoreScaling` 调整）
- **Type**: 设备型号（如 `NVIDIA-A100-SXM4-80GB`）
- **Numa**: NUMA 节点号
- **Mode**: 运行模式（`hami-core` / `mig` / `mps`）
- **Health**: 健康状态

---

## 3. 设备健康检查流程

Device Plugin 通过 `rm.CheckHealth()` 持续监控 GPU 设备的健康状态。

### 3.1 健康检查机制

```go
// ListAndWatch 接收健康状态变化
func (plugin *NvidiaDevicePlugin) ListAndWatch(...) error {
    s.Send(&kubeletdevicepluginv1beta1.ListAndWatchResponse{
        Devices: plugin.apiDevices(),
    })
    for {
        select {
        case <-plugin.stop:
            return nil
        case d := <-plugin.health:
            d.Health = kubeletdevicepluginv1beta1.Unhealthy
            s.Send(&kubeletdevicepluginv1beta1.ListAndWatchResponse{
                Devices: plugin.apiDevices(),
            })
        }
    }
}
```

### 3.2 健康检查触发条件

当 GPU 出现以下问题时，健康检查会报告 Unhealthy:

- NVML 检测到 GPU 硬件错误（ECC 错误、温度过高等）
- GPU 驱动异常
- 设备从系统中消失

### 3.3 健康状态排查

```bash
# 检查 GPU 硬件状态
nvidia-smi -q -d HEALTH

# 检查 ECC 错误计数
nvidia-smi -q -d ECC

# 查看 kubelet 日志中的设备插件事件
journalctl -u kubelet | grep -i "device.*plugin\|nvidia"

# 检查 device plugin socket 是否存在
ls -la /var/lib/kubelet/device-plugins/nvidia-gpu.sock
```

---

## 4. 常见问题排查

### 4.1 GPU 设备未被发现

**症状**: 节点注解中不包含 GPU 设备，或设备数量不正确。

```mermaid
flowchart TD
    START["GPU 设备未被发现"] --> CHECK1{"nvidia-smi 是否正常?"}

    CHECK1 -->|"否"| FIX1["检查 NVIDIA 驱动安装"]
    FIX1 --> FIX1A["确认驱动版本兼容性"]
    FIX1A --> FIX1B["重新安装或升级驱动"]

    CHECK1 -->|"是"| CHECK2{"Device Plugin Pod 是否运行?"}

    CHECK2 -->|"否"| FIX2["检查 DaemonSet 状态"]
    FIX2 --> FIX2A{"镜像拉取成功?"}
    FIX2A -->|"否"| FIX2B["检查镜像仓库和拉取策略"]
    FIX2A -->|"是"| FIX2C{"容器启动报错?"}
    FIX2C -->|"是"| FIX2D["检查 HOOK_PATH 和 NODE_NAME 环境变量"]

    CHECK2 -->|"是"| CHECK3{"Device Plugin 日志是否有错误?"}

    CHECK3 -->|"有 nvml Init err"| FIX3["NVIDIA 驱动容器内不可见"]
    FIX3 --> FIX3A["检查 device plugin 容器
是否挂载了 /dev/nvidia*"]
    FIX3A --> FIX3B["检查容器 securityContext
是否有足够权限"]

    CHECK3 -->|"有 nvml GetDeviceCount err"| FIX4["NVML 返回设备数为 0"]
    FIX4 --> FIX4A["检查 NVIDIA_VISIBLE_DEVICES 环境变量"]
    FIX4A --> FIX4B["确认 nvidia-container-runtime 配置"]

    CHECK3 -->|"无错误但设备数少"| CHECK4{"是否配置了 FilterDevice?"}
    CHECK4 -->|"是"| FIX5["检查 config.json 中的
FilterDevice UUID/Index 配置"]
    CHECK4 -->|"否"| FIX6["检查 GPU 健康状态
nvidia-smi -q -d HEALTH"]

    CHECK3 -->|"有 patch node error"| FIX7["节点注解写入失败"]
    FIX7 --> FIX7A["检查 RBAC 权限
ServiceAccount 是否有节点 patch 权限"]
    FIX7A --> FIX7B["检查 NODE_NAME 环境变量
是否与实际节点名匹配"]

    style START fill:#ffcdd2
    style FIX1B fill:#c8e6c9
    style FIX2B fill:#c8e6c9
    style FIX2D fill:#c8e6c9
    style FIX3B fill:#c8e6c9
    style FIX4B fill:#c8e6c9
    style FIX5 fill:#c8e6c9
    style FIX6 fill:#c8e6c9
    style FIX7B fill:#c8e6c9
```

### 4.2 注解格式错误

**症状**: 调度器无法正确读取节点设备信息，Pod 调度失败。

**排查步骤**:

1. **检查注解内容完整性**:
```bash
kubectl get node <node-name> -o jsonpath='{.metadata.annotations.hami\.io/node-nvidia-register}' | python3 -m json.tool
```

2. **验证设备信息字段**:
- `ID` 必须是完整的 GPU UUID（如 `GPU-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`）
- `Count` 必须大于 0
- `Devmem` 必须大于 0（单位 MB）
- `Type` 必须以 `NVIDIA` 或 `NVIDIA-` 开头

3. **检查 DeviceMemoryScaling 配置**:
```bash
# 如果 DeviceMemoryScaling > 1，registeredmem 会被放大
# 这在 MIG 模式下不会生效
kubectl get configmap hami-scheduler-device -n kube-system -o yaml
```

4. **检查模型名称格式**:
```
# 正确: NVIDIA-A100-SXM4-80GB
# 正确: NVIDIA A100-SXM4-80GB (会自动加前缀 NVIDIA-)
# 错误: A100 (缺少 NVIDIA 前缀 - 代码会自动修正)
```

### 4.3 握手超时

**症状**: Pod 长时间处于 Pending 状态，调度器日志显示 "handshake timeout"。

**背景**: HAMi 使用基于节点注解的握手机制来确保设备信息的一致性。调度器在分配设备前会检查节点的握手时间戳是否在有效期内。

**排查决策树**:

```mermaid
flowchart TD
    START["握手超时 - Pod Pending"] --> CHECK1{"检查 Device Plugin Pod 状态"}

    CHECK1 -->|"CrashLoopBackOff"| FIX1["查看 Device Plugin 容器日志"]
    FIX1 --> FIX1A["修复启动错误后重启 Pod"]

    CHECK1 -->|"Running"| CHECK2{"检查 WatchAndRegister 日志"}

    CHECK2 -->|"有 patch node error"| FIX2["节点注解更新失败"]
    FIX2 --> FIX2A["检查 RBAC 权限"]
    FIX2A --> FIX2B["检查 API Server 连通性"]

    CHECK2 -->|"Successfully registered"| CHECK3{"检查调度器日志"}

    CHECK3 -->|"handshake time expired"| FIX3["Device Plugin 和调度器时钟不同步"]
    FIX3 --> FIX3A["检查节点和 Master 的 NTP 同步状态"]
    FIX3A --> FIX3B["确保时钟偏差在 30 秒以内"]

    CHECK3 -->|"node not found"| FIX4["调度器未发现节点设备"]
    FIX4 --> FIX4A["确认调度器版本与 Device Plugin 版本一致"]
    FIX4A --> FIX4B["检查注解 key 是否匹配
hami.io/node-nvidia-register"]

    CHECK3 -->|"device decode error"| FIX5["设备信息解码失败"]
    FIX5 --> FIX5A["Device Plugin 和调度器版本不一致"]
    FIX5A --> FIX5B["统一升级 HAMi 组件"]

    CHECK2 -->|"无相关日志输出"| CHECK4{"WatchAndRegister 循环是否启动?"}
    CHECK4 -->|"否"| FIX6["Device Plugin Start() 可能未完成"]
    FIX6 --> FIX6A["检查 gRPC 和 Kubelet 注册日志"]
    CHECK4 -->|"是"| FIX7["增加日志级别 -v=4 重新排查"]

    style START fill:#ffcdd2
    style FIX1A fill:#c8e6c9
    style FIX2B fill:#c8e6c9
    style FIX3B fill:#c8e6c9
    style FIX4B fill:#c8e6c9
    style FIX5B fill:#c8e6c9
    style FIX6A fill:#c8e6c9
    style FIX7 fill:#c8e6c9
```

### 4.4 Allocate 失败

**症状**: Pod 调度成功但容器创建失败，事件中显示 Allocate 错误。

**常见原因与解决方案**:

| 错误信息 | 原因 | 解决方案 |
|----------|------|----------|
| `device number not matched` | 调度器分配的设备数与 kubelet 请求数不一致 | 检查 Pod 注解中的设备分配信息 |
| `invalid allocation request ... unknown device` | 请求了不存在的设备 ID | 检查节点上的 GPU 是否健康 |
| `request ... too large` | MIG 模式下请求超过 1 个共享设备 | 调整 Pod 资源请求 |
| `failed to get pending pod` | 无法从 API Server 获取 Pending Pod | 检查 API Server 连通性和 RBAC |
| `failed to decode pod devices` | Pod 注解中设备信息格式错误 | 检查调度器版本兼容性 |

### 4.5 libvgpu.so 加载失败

**症状**: 容器内 GPU 使用不受限制，共享内存文件未创建。

**排查步骤**:

```bash
# 1. 检查容器内是否有 ld.so.preload
kubectl exec -it <pod-name> -- cat /etc/ld.so.preload

# 2. 检查 libvgpu.so 是否挂载成功
kubectl exec -it <pod-name> -- ls -la /usr/local/vgpu/libvgpu.so

# 3. 检查 CUDA 相关环境变量
kubectl exec -it <pod-name> -- env | grep CUDA

# 4. 检查 CUDA_DISABLE_CONTROL 是否被设置
kubectl exec -it <pod-name> -- env | grep CUDA_DISABLE_CONTROL

# 5. 检查共享内存缓存目录
kubectl exec -it <pod-name> -- ls -la /usr/local/vgpu/vgpu/
```

---

## 5. 关键调试命令汇总

### 5.1 Device Plugin 状态

```bash
# 查看 Device Plugin DaemonSet 状态
kubectl get ds -n kube-system | grep hami

# 查看 Device Plugin Pod 日志
kubectl logs -n kube-system -l app=hami-device-plugin --tail=100

# 查看详细日志（增加日志级别）
kubectl logs -n kube-system <pod-name> -c device-plugin -- -v=5

# 查看 device plugin socket
kubectl exec -n kube-system <pod-name> -- ls -la /var/lib/kubelet/device-plugins/
```

### 5.2 Kubelet 侧调试

```bash
# 查看 kubelet 日志中的设备插件事件
journalctl -u kubelet | grep -i "device\|plugin\|nvidia\|allocat"

# 查看 kubelet 设备插件注册目录
ls -la /var/lib/kubelet/device-plugins/

# 检查 kubelet 资源分配状态
kubectl get node <node-name> -o json | jq '.status.allocatable'
```

### 5.3 GPU 硬件调试

```bash
# GPU 完整状态
nvidia-smi -q

# GPU 进程信息
nvidia-smi pmon -s u -d 1

# GPU ECC 错误
nvidia-smi -q -d ECC

# GPU 温度和功耗
nvidia-smi -q -d POWER,TEMPERATURE

# MIG 实例状态
nvidia-smi mig -lgi
nvidia-smi mig -lci
```

### 5.4 节点注解操作

```bash
# 清除节点锁（紧急情况）
kubectl annotate node <node-name> hami.io/mutex.lock-

# 手动触发设备重新注册（删除 Device Plugin Pod）
kubectl delete pod -n kube-system -l app=hami-device-plugin --field-selector spec.nodeName=<node-name>

# 查看所有 HAMi 相关注解
kubectl get node <node-name> -o json | jq '[.metadata.annotations | to_entries[] | select(.key | contains("hami") or contains("nvidia"))]'
```

---

## 6. 完整调试决策树

```mermaid
flowchart TD
    START["GPU Pod 问题"] --> SYMPTOM{"问题症状?"}

    SYMPTOM -->|"Pod Pending"| PEND1{"kubectl describe pod
查看 Events"}
    SYMPTOM -->|"Pod 运行但 GPU 不受限"| CTRL1{"检查 libvgpu.so"}
    SYMPTOM -->|"Pod 创建失败"| CREATE1{"查看 Allocate 错误"}

    PEND1 -->|"Insufficient nvidia.com/gpu"| PEND2{"节点是否有可用 GPU?"}
    PEND2 -->|"kubectl get node -o yaml
检查 allocatable"| PEND3{"allocatable 有 GPU?"}
    PEND3 -->|"否"| PEND4["Device Plugin 未注册设备
参见 4.1"]
    PEND3 -->|"是"| PEND5{"现有 Pod 占用了所有设备?"}
    PEND5 -->|"是"| PEND6["等待资源释放
或增加 DeviceSplitCount"]
    PEND5 -->|"否"| PEND7["检查调度器日志
可能是调度器未运行"]

    PEND1 -->|"handshake timeout"| HT["参见 4.3"]
    PEND1 -->|"没有相关 Events"| PEND8["检查调度器是否正常运行"]

    CTRL1 -->|"libvgpu.so 不存在"| CTRL2["检查 Device Plugin
Allocate 返回的 Mounts"]
    CTRL1 -->|"libvgpu.so 存在"| CTRL3{"ld.so.preload 内容?"}
    CTRL3 -->|"未包含 libvgpu.so"| CTRL4["检查 CUDA_DISABLE_CONTROL 环境变量"]
    CTRL3 -->|"包含 libvgpu.so"| CTRL5["检查 .cache 文件是否创建"]
    CTRL5 -->|"未创建"| CTRL6["容器未调用 cuInit
或 libvgpu.so 版本不兼容"]
    CTRL5 -->|"已创建"| CTRL7["Monitor 未正确读取
参见 vgpu-monitor.md"]

    CREATE1 -->|"device number not matched"| CRT2["调度器注解与 kubelet 请求不一致
检查 Pod 注解"]
    CREATE1 -->|"unknown device"| CRT3["设备 ID 无效
检查 GPU 健康状态"]
    CREATE1 -->|"failed to get pending pod"| CRT4["API Server 连通性
或 RBAC 权限问题"]

    style START fill:#e1f5fe
    style PEND4 fill:#fff9c4
    style PEND6 fill:#c8e6c9
    style PEND7 fill:#fff9c4
    style HT fill:#fff9c4
    style PEND8 fill:#fff9c4
    style CTRL2 fill:#fff9c4
    style CTRL4 fill:#fff9c4
    style CTRL6 fill:#fff9c4
    style CTRL7 fill:#fff9c4
    style CRT2 fill:#fff9c4
    style CRT3 fill:#fff9c4
    style CRT4 fill:#fff9c4
```

---

## 7. 日志分析速查表

| 日志关键词 | 严重程度 | 含义 | 处理建议 |
|-----------|----------|------|----------|
| `nvml Init err` | Fatal | NVIDIA 驱动不可用 | 检查驱动安装和设备挂载 |
| `nvml GetDeviceCount err` | Fatal | 无法枚举 GPU 设备 | 检查 NVIDIA_VISIBLE_DEVICES |
| `nvml get memory error` | Fatal | 获取显存信息失败 | 检查 GPU 硬件状态 |
| `GetMemoryInfo not supported` | Warning | 统一内存架构 GPU | 配置 `preConfiguredDeviceMemory` |
| `patch node error` | Error | 节点注解写入失败 | 检查 RBAC 和网络 |
| `device number not matched` | Error | 设备数不一致 | 检查 Pod 注解和调度器 |
| `Successfully registered annotation` | Info | 正常注册成功 | 无需处理 |
| `Retrying in 5s seconds` | Warning | 注册失败后重试 | 观察是否恢复 |
| `GRPC server has repeatedly crashed` | Fatal | gRPC 服务反复崩溃 | 检查 socket 权限和系统资源 |
| `MIG apply lock file detected` | Info | MIG 配置变更进行中 | 等待完成 |
