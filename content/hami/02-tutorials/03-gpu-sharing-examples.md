---
title: "HAMi GPU 共享实战示例"
weight: 3
---


> 本文档提供 8 个典型的 GPU 共享使用场景，涵盖从基础共享到高级调度策略的各种配置方式。每个示例都包含完整的 YAML 规格、预期行为说明和验证方法。

---

## 示例总览

| 编号 | 场景 | 关键配置 | 适用场景 |
|------|------|----------|---------|
| 1 | 两个 Pod 共享一张 GPU | `gpumem` + `gpucores` | 开发测试、推理服务 |
| 2 | 仅显存隔离（不限制算力） | `gpumem`（不设 `gpucores`） | 推理服务、不敏感任务 |
| 3 | GPU 独占分配 | `gpucores: 100` | 训练任务、基准测试 |
| 4 | 百分比显存分配 | `gpumem-percentage` | 适配不同型号 GPU |
| 5 | 覆盖调度策略 | Pod Annotation | 特殊调度需求 |
| 6 | GPU 型号选择 | `use-gputype` / `nouse-gputype` | 混合 GPU 集群 |
| 7 | GPU UUID 绑定 | `use-gpuuuid` | 特定设备绑定 |
| 8 | NUMA 感知分配 | `numa-bind` | 高性能计算 |

---

## 示例 1: 两个 Pod 共享一张 GPU（各占 50% 资源）

### 场景描述

两个推理服务共享同一张 GPU，各分配 50% 的显存和算力。这是 HAMi 最常见的使用场景，适合推理服务等对 GPU 资源需求较低的工作负载。

### YAML 规格

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: inference-service-a
spec:
  containers:
  - name: model-server
    image: nvcr.io/nvidia/tritonserver:23.04-py3
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 8192       # 8GB 显存
        nvidia.com/gpucores: 50       # 50% 算力
---
apiVersion: v1
kind: Pod
metadata:
  name: inference-service-b
spec:
  containers:
  - name: model-server
    image: nvcr.io/nvidia/tritonserver:23.04-py3
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 8192       # 8GB 显存
        nvidia.com/gpucores: 50       # 50% 算力
```

### 预期行为

- 两个 Pod 被调度到同一张物理 GPU 上
- 每个 Pod 最多使用 8192MB 显存
- 每个 Pod 的 SM 利用率被限制在 50%
- 两个 Pod 的显存空间相互隔离，互不可见

### 验证命令

```bash
# 查看两个 Pod 分配的 GPU UUID（应相同）
kubectl get pod inference-service-a -o jsonpath='{.metadata.annotations.hami\.io/vgpu-devices-allocated}' && echo
kubectl get pod inference-service-b -o jsonpath='{.metadata.annotations.hami\.io/vgpu-devices-allocated}' && echo

# 在容器内验证显存限制
kubectl exec inference-service-a -- nvidia-smi
kubectl exec inference-service-b -- nvidia-smi

# 在宿主机上查看 GPU 使用情况
# ssh <gpu-node>
# nvidia-smi
```

---

## 示例 2: 仅显存隔离（不限制算力）

### 场景描述

当不需要算力隔离时，仅通过显存限制来实现 GPU 共享。不设置 `gpucores` 意味着 Pod 可以使用 GPU 的全部算力，适合推理类任务或对延迟敏感的场景。

### YAML 规格

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-only-isolation
spec:
  containers:
  - name: inference
    image: nvidia/cuda:11.0.3-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 4096       # 仅限制 4GB 显存
        # 不设置 nvidia.com/gpucores，算力不受限制
```

### 预期行为

- Pod 的显存被限制在 4096MB
- Pod 可以使用 GPU 100% 的算力（不受限制）
- 如果 Scheduler 的 `--default-cores` 为 `0`（默认值），则不会注入 `CUDA_DEVICE_SM_LIMIT`
- 多个仅隔离显存的 Pod 可以共享同一张 GPU，但算力上会互相竞争

### 验证命令

```bash
# 检查容器中的环境变量，确认没有 SM 限制
kubectl exec memory-only-isolation -- env | grep CUDA

# 预期输出:
# CUDA_DEVICE_MEMORY_LIMIT_0=4096M
# 不应有 CUDA_DEVICE_SM_LIMIT 或其值为 0
```

---

## 示例 3: GPU 独占分配

### 场景描述

某些工作负载（如模型训练、基准测试）需要独占整张 GPU。在 HAMi 中，设置 `gpucores: 100` 表示请求 100% 的算力，调度器会将该 GPU 视为已满，不再分配给其他 Pod。

### YAML 规格

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: training-job
spec:
  containers:
  - name: trainer
    image: nvcr.io/nvidia/pytorch:23.04-py3
    command: ["python", "train.py"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpucores: 100      # 100% 算力 = 独占
        # 不设置 gpumem，独占模式下自动获得全部显存
```

### 预期行为

- Pod 独占整张 GPU，其他 Pod 无法被调度到该 GPU 上
- 当 `gpucores=100` 且未指定 `gpumem` 时，Pod 自动获得该 GPU 的全部显存
- 如果 GPU 上已有其他 Pod 在运行（`Used > 0`），该独占请求会被拒绝，调度器会寻找其他空闲 GPU
- 这与直接使用 `nvidia.com/gpu: 1`（不加 HAMi 资源）的区别在于，HAMi 仍然会对资源进行追踪和监控

### 验证命令

```bash
# 确认 Pod 获得了整张 GPU
kubectl exec training-job -- nvidia-smi

# 尝试部署另一个请求同一张 GPU 的 Pod，应该会被调度到其他 GPU
kubectl get pod training-job -o jsonpath='{.metadata.annotations.hami\.io/vgpu-devices-allocated}' && echo
```

---

## 示例 4: 百分比显存分配

### 场景描述

使用 `gpumem-percentage` 按百分比分配显存，无需关心具体 GPU 型号的显存大小。例如在混合集群中，同一份 YAML 可以部署到 16GB 的 T4 或 80GB 的 A100 上，都能获得相应比例的显存。

### YAML 规格

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: percentage-alloc
spec:
  containers:
  - name: app
    image: nvidia/cuda:11.0.3-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem-percentage: 30    # 分配 30% 显存
        nvidia.com/gpucores: 40             # 分配 40% 算力
```

### 预期行为

| GPU 型号 | 总显存 | 实际分配显存（30%） |
|----------|--------|-------------------|
| Tesla T4 | 16GB | ~4.8GB |
| Tesla V100 | 32GB | ~9.6GB |
| A100 | 80GB | ~24GB |

- 显存分配量 = GPU 总显存 x `gpumem-percentage / 100`
- `gpumem-percentage` 和 `gpumem` 不应同时设置，如果同时设置了，`gpumem`（绝对值）优先生效
- 取值范围: 1-100

### 验证命令

```bash
# 检查实际分配的显存
kubectl exec percentage-alloc -- nvidia-smi

# 通过注解查看分配详情
kubectl get pod percentage-alloc -o jsonpath='{.metadata.annotations.hami\.io/vgpu-devices-allocated}' | python3 -m json.tool
```

---

## 示例 5: 通过 Pod Annotation 覆盖调度策略

### 场景描述

集群全局配置为 `spread` 策略（默认将 Pod 分散调度），但某些批量推理任务希望集中分配到同一张 GPU 以减少通信开销。可以通过 Pod Annotation 在 Pod 级别覆盖全局调度策略。

### YAML 规格

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: binpack-inference
  annotations:
    hami.io/node-scheduler-policy: "spread"     # 节点级: 分散到不同节点
    hami.io/gpu-scheduler-policy: "binpack"      # GPU 级: 集中到同一张 GPU
spec:
  containers:
  - name: inference
    image: nvidia/cuda:11.0.3-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 2048
        nvidia.com/gpucores: 25
```

### 可用的调度策略 Annotation

| Annotation | 可选值 | 说明 |
|-----------|--------|------|
| `hami.io/node-scheduler-policy` | `binpack` / `spread` | 覆盖全局节点级调度策略 |
| `hami.io/gpu-scheduler-policy` | `binpack` / `spread` / `topology` | 覆盖全局 GPU 级调度策略 |

> `topology` 策略是 GPU 调度策略的扩展选项，它会根据 GPU 间的拓扑互联评分选择最优的 GPU 组合，适用于多卡训练场景。

### 预期行为

- 此 Pod 的调度策略会覆盖全局默认策略
- 节点级使用 `spread`，Pod 会被分散到不同节点
- GPU 级使用 `binpack`，在选定的节点内会优先填满已有任务的 GPU
- 此策略仅影响当前 Pod，不影响其他 Pod 的调度

### 验证命令

```bash
# 部署多个使用 binpack 策略的 Pod，观察它们是否集中在同一张 GPU
for i in 1 2 3 4; do
  kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: binpack-test-$i
  annotations:
    hami.io/gpu-scheduler-policy: "binpack"
spec:
  containers:
  - name: app
    image: nvidia/cuda:11.0.3-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 2048
        nvidia.com/gpucores: 20
EOF
done

# 检查分配的 GPU UUID，binpack 策略下应集中在少数 GPU 上
for i in 1 2 3 4; do
  echo "Pod binpack-test-$i:"
  kubectl get pod binpack-test-$i -o jsonpath='{.metadata.annotations.hami\.io/vgpu-devices-allocated}' && echo
done
```

---

## 示例 6: GPU 型号选择

### 场景描述

在混合 GPU 集群中（如同时拥有 T4、V100、A100），某些任务只能运行在特定型号的 GPU 上。HAMi 支持通过 Annotation 指定目标 GPU 型号或排除不兼容的型号。

### YAML 规格

**场景 A: 指定目标 GPU 型号**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: a100-only-task
  annotations:
    nvidia.com/use-gputype: "A100"    # 仅调度到 A100 GPU
spec:
  containers:
  - name: training
    image: nvcr.io/nvidia/pytorch:23.04-py3
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 40960
        nvidia.com/gpucores: 50
```

**场景 B: 排除不兼容的 GPU 型号**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: exclude-old-gpu
  annotations:
    nvidia.com/nouse-gputype: "T4,P100"    # 排除 T4 和 P100
spec:
  containers:
  - name: inference
    image: nvidia/cuda:11.0.3-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 8192
        nvidia.com/gpucores: 30
```

**场景 C: 同时指定使用和排除**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: selective-gpu
  annotations:
    nvidia.com/use-gputype: "A100,V100"     # 允许 A100 或 V100
    nvidia.com/nouse-gputype: "A100-SXM"    # 但排除 A100-SXM 型号
spec:
  containers:
  - name: app
    image: nvidia/cuda:11.0.3-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 8192
```

### GPU 型号选择 Annotation

| Annotation | 格式 | 说明 |
|-----------|------|------|
| `nvidia.com/use-gputype` | 逗号分隔的型号列表 | 仅允许调度到指定型号的 GPU，使用模糊匹配（包含关系） |
| `nvidia.com/nouse-gputype` | 逗号分隔的型号列表 | 排除指定型号的 GPU，使用模糊匹配（包含关系） |

### 预期行为

- 型号匹配为大小写不敏感的子串包含匹配
- 例如 `use-gputype: "A100"` 会匹配 `A100-SXM4-80GB`、`A100-PCIE-40GB` 等所有包含 "A100" 的型号
- 当同时设置 `use-gputype` 和 `nouse-gputype` 时，必须通过两项检查：先匹配 use，再排除 nouse
- 如果没有满足条件的 GPU，Pod 将保持 Pending 状态

### 验证命令

```bash
# 查看集群中各节点的 GPU 型号
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"	"}{.metadata.annotations.hami\.io/node-nvidia-register}{"
"}{end}'

# 确认 Pod 调度到了正确的 GPU 型号
kubectl describe pod a100-only-task | grep -i "node:"
kubectl exec a100-only-task -- nvidia-smi -L
```

---

## 示例 7: GPU UUID 绑定

### 场景描述

在某些特殊场景下（如性能测试复现、固定设备绑定），需要将 Pod 绑定到特定 UUID 的 GPU 上。HAMi 支持通过 Annotation 指定允许或排除的 GPU UUID。

### YAML 规格

**场景 A: 绑定到指定 UUID 的 GPU**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: uuid-pinned-task
  annotations:
    nvidia.com/use-gpuuuid: "GPU-e3b5f6a0-1234-5678-abcd-ef1234567890"
spec:
  containers:
  - name: benchmark
    image: nvidia/cuda:11.0.3-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 8192
        nvidia.com/gpucores: 100
```

**场景 B: 排除指定 UUID 的 GPU**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: uuid-exclude-task
  annotations:
    nvidia.com/nouse-gpuuuid: "GPU-e3b5f6a0-1234-5678-abcd-ef1234567890,GPU-a1b2c3d4-5678-9012-efgh-ij3456789012"
spec:
  containers:
  - name: app
    image: nvidia/cuda:11.0.3-base-ubuntu20.04
    command: ["sleep", "infinity"]
    resources:
      limits:
        nvidia.com/gpu: 1
        nvidia.com/gpumem: 4096
```

### UUID 绑定 Annotation

| Annotation | 格式 | 说明 |
|-----------|------|------|
| `nvidia.com/use-gpuuuid` | 逗号分隔的 UUID 列表 | 仅允许调度到指定 UUID 的 GPU |
| `nvidia.com/nouse-gpuuuid` | 逗号分隔的 UUID 列表 | 排除指定 UUID 的 GPU |

### 预期行为

- UUID 匹配为精确匹配
- 如果指定的 GPU UUID 不存在于集群中，Pod 将保持 Pending 状态
- 可以与 GPU 型号选择 Annotation 组合使用
- 多个 UUID 使用英文逗号分隔

### 验证命令

```bash
# 获取集群中所有 GPU 的 UUID
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.annotations.hami\.io/node-nvidia-register}{"
"}{end}'

# 或者在 GPU 节点上直接查看
# nvidia-smi -L
# 输出示例:
# GPU 0: Tesla V100-SXM2-32GB (UUID: GPU-e3b5f6a0-1234-5678-abcd-ef1234567890)

# 验证 Pod 是否调度到了指定的 GPU
kubectl get pod uuid-pinned-task -o jsonpath='{.metadata.annotations.hami\.io/vgpu-devices-allocated}' && echo
```

---

## 示例 8: NUMA 感知分配

### 场景描述

在多 NUMA 节点的服务器上，GPU 设备与特定的 NUMA 节点关联。当工作负载对内存访问延迟敏感时（如高性能计算、大模型训练），启用 NUMA 感知分配可以确保分配到同一 NUMA 域的 GPU，减少跨 NUMA 内存访问带来的性能损失。

### YAML 规格

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: numa-aware-training
  annotations:
    nvidia.com/numa-bind: "true"    # 启用 NUMA 感知分配
spec:
  containers:
  - name: training
    image: nvcr.io/nvidia/pytorch:23.04-py3
    command: ["python", "train.py"]
    resources:
      limits:
        nvidia.com/gpu: 2             # 请求 2 张 GPU
        nvidia.com/gpumem: 32768      # 每张 GPU 32GB 显存
        nvidia.com/gpucores: 100      # 每张 GPU 100% 算力
```

### NUMA 感知 Annotation

| Annotation | 可选值 | 说明 |
|-----------|--------|------|
| `nvidia.com/numa-bind` | `true` / `false` | 是否启用 NUMA 感知分配，默认 `false` |

### 预期行为

- 启用 NUMA 感知后，调度器会确保请求的多张 GPU 全部位于同一个 NUMA 节点
- 如果某个 NUMA 节点无法满足所有 GPU 请求（例如 NUMA 0 只有 2 张空闲 GPU，但请求了 4 张），调度器会尝试其他 NUMA 节点
- 如果没有任何 NUMA 节点能满足请求，Pod 将保持 Pending 状态
- 当仅请求 1 张 GPU 时，NUMA 感知对结果没有影响
- 此特性依赖 Device Plugin 正确上报 GPU 的 NUMA 信息

### 典型的 NUMA 拓扑示例

```
NUMA 节点 0                    NUMA 节点 1
+----------+----------+       +----------+----------+
|  GPU 0   |  GPU 1   |       |  GPU 2   |  GPU 3   |
+----------+----------+       +----------+----------+
|      CPU 0-15       |       |      CPU 16-31      |
+---------------------+       +---------------------+
|     内存通道 0-5     |       |     内存通道 6-11    |
+---------------------+       +---------------------+
```

当请求 2 张 GPU 并启用 NUMA 感知时：
- 调度器会将 GPU 0 和 GPU 1 分配给 Pod（同一 NUMA 节点），而不会分配 GPU 0 和 GPU 2（跨 NUMA 节点）

### 验证命令

```bash
# 查看 GPU 的 NUMA 亲和性
# nvidia-smi topo -m
# 输出示例:
#         GPU0  GPU1  GPU2  GPU3  CPU Affinity  NUMA Affinity
# GPU0     X    NV2   SYS   SYS   0-15          0
# GPU1    NV2    X    SYS   SYS   0-15          0
# GPU2    SYS   SYS    X    NV2   16-31         1
# GPU3    SYS   SYS   NV2    X    16-31         1

# 验证 Pod 分配的 GPU 是否在同一 NUMA 节点
kubectl get pod numa-aware-training -o jsonpath='{.metadata.annotations.hami\.io/vgpu-devices-allocated}' && echo
```

---

## 综合对比

下表对比了各示例中使用的关键资源字段和 Annotation：

| 示例 | `nvidia.com/gpu` | `gpumem` | `gpumem-percentage` | `gpucores` | 关键 Annotation |
|------|-------------------|----------|---------------------|------------|----------------|
| 1. 共享 GPU | 1 | 8192 | - | 50 | - |
| 2. 仅显存隔离 | 1 | 4096 | - | - | - |
| 3. 独占分配 | 1 | - | - | 100 | - |
| 4. 百分比显存 | 1 | - | 30 | 40 | - |
| 5. 覆盖策略 | 1 | 2048 | - | 25 | `hami.io/gpu-scheduler-policy` |
| 6. 型号选择 | 1 | 8192 | - | 30 | `nvidia.com/use-gputype` |
| 7. UUID 绑定 | 1 | 8192 | - | 100 | `nvidia.com/use-gpuuuid` |
| 8. NUMA 感知 | 2 | 32768 | - | 100 | `nvidia.com/numa-bind` |

---

## 下一步

- [快速入门教程](./01-quick-start/) -- 如果您还没有安装 HAMi，从这里开始
- [配置指南](./02-configuration-guide/) -- 深入了解调度器和 Device Plugin 的配置参数
