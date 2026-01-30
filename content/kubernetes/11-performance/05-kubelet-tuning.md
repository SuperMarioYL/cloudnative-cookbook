---
title: "Kubelet 调优"
weight: 5
---
## 概述

Kubelet 是运行在每个节点上的代理，负责 Pod 生命周期管理、容器运行时交互、资源管理等。Kubelet 的性能直接影响节点上 Pod 的启动速度和运行稳定性。

## Pod 同步优化

### 同步周期配置

```bash
# Kubelet 同步配置
kubelet \
  # Pod 同步周期
  --sync-frequency=1m \
  # 文件源检查周期
  --file-check-frequency=20s \
  # HTTP 源检查周期
  --http-check-frequency=20s
```

```yaml
# KubeletConfiguration 配置
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
syncFrequency: 1m
fileCheckFrequency: 20s
httpCheckFrequency: 20s
```

### PLEG 优化

```yaml
# PLEG (Pod Lifecycle Event Generator) 配置
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 重新列出周期
runtimeRequestTimeout: 2m

# PLEG 说明:
# - 监控容器状态变化
# - 默认 1 秒检查一次
# - 如果 PLEG 不健康，节点会变为 NotReady
```

### 并发控制

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 并发 Pod 同步数
maxPods: 110
# 并发镜像拉取
serializeImagePulls: false
maxParallelImagePulls: 5
```

## 容器运行时优化

### 镜像拉取优化

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 禁用串行镜像拉取
serializeImagePulls: false
# 最大并行镜像拉取数
maxParallelImagePulls: 5
# 镜像拉取进度截止时间
imageMinimumGCAge: 2m
# 镜像 GC 高水位（磁盘使用百分比）
imageGCHighThresholdPercent: 85
# 镜像 GC 低水位
imageGCLowThresholdPercent: 80
```

### 容器运行时配置

```yaml
# containerd 配置优化
# /etc/containerd/config.toml

version = 2

[plugins."io.containerd.grpc.v1.cri"]
  # 沙箱镜像
  sandbox_image = "registry.k8s.io/pause:3.9"

  [plugins."io.containerd.grpc.v1.cri".containerd]
    # 默认运行时
    default_runtime_name = "runc"

    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
      runtime_type = "io.containerd.runc.v2"

      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        # 启用 systemd cgroup
        SystemdCgroup = true

  [plugins."io.containerd.grpc.v1.cri".registry]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
      [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
        endpoint = ["https://mirror.example.com"]
```

### CRI 超时配置

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 运行时请求超时
runtimeRequestTimeout: 2m
```

## 资源管理优化

### 资源预留

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 系统预留资源
systemReserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "1Gi"
# Kubernetes 组件预留资源
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
  ephemeral-storage: "1Gi"
# 预留 cgroup
systemReservedCgroup: /system.slice
kubeReservedCgroup: /kubelet.slice
# 强制执行节点可分配
enforceNodeAllocatable:
  - pods
  - system-reserved
  - kube-reserved
```

### Cgroup 驱动

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 使用 systemd cgroup 驱动
cgroupDriver: systemd
# cgroup 根目录
cgroupRoot: /
# kubepods cgroup
cgroupsPerQOS: true
```

### QoS 管理

```yaml
# Pod QoS 级别计算
# Guaranteed: requests == limits (所有容器)
# Burstable: 至少有一个容器有 requests 或 limits
# BestEffort: 没有 requests 和 limits

# 建议配置:
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      resources:
        requests:
          cpu: "100m"
          memory: "128Mi"
        limits:
          cpu: "200m"
          memory: "256Mi"
```

## 驱逐管理优化

### 驱逐阈值配置

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 硬驱逐阈值（立即驱逐）
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
# 软驱逐阈值（等待宽限期）
evictionSoft:
  memory.available: "300Mi"
  nodefs.available: "15%"
# 软驱逐宽限期
evictionSoftGracePeriod:
  memory.available: "1m30s"
  nodefs.available: "1m30s"
# 驱逐压力转换周期
evictionPressureTransitionPeriod: 5m
# 最小回收量
evictionMinimumReclaim:
  memory.available: "500Mi"
  nodefs.available: "500Mi"
```

### 驱逐优先级

```
驱逐顺序:

1. 超过请求资源的 BestEffort Pod
2. 超过请求资源的 Burstable Pod
3. 未超过请求资源的 BestEffort Pod
4. 未超过请求资源的 Burstable Pod
5. Guaranteed Pod (最后驱逐)

在同一 QoS 级别内:
- 优先驱逐优先级低的 Pod
- 优先驱逐资源使用超出请求更多的 Pod
```

## 卷管理优化

### 卷统计配置

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 卷统计聚合周期
volumeStatsAggPeriod: 1m
```

### CSI 优化

```yaml
# CSI 驱动配置优化

# 1. 增加超时
# CSI 驱动 Deployment 中
args:
  - --timeout=120s

# 2. 并行卷操作
# Kubelet 默认并行处理卷挂载

# 3. 监控卷操作
# 关键指标:
# - kubelet_volume_stats_capacity_bytes
# - kubelet_volume_stats_used_bytes
# - storage_operation_duration_seconds
```

## 探针优化

### 探针配置最佳实践

```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
    - name: app
      # 存活探针
      livenessProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 15  # 足够的启动时间
        periodSeconds: 10        # 不要太频繁
        timeoutSeconds: 5
        failureThreshold: 3
      # 就绪探针
      readinessProbe:
        httpGet:
          path: /ready
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3
      # 启动探针（新应用推荐）
      startupProbe:
        httpGet:
          path: /healthz
          port: 8080
        initialDelaySeconds: 0
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 30  # 允许最多 150 秒启动
```

### 避免探针问题

```yaml
# 常见问题:

# 1. 探针超时过短
# 问题: 应用响应慢时被错误杀死
# 解决: 增加 timeoutSeconds

# 2. 没有 startupProbe
# 问题: 启动慢的应用被杀死
# 解决: 使用 startupProbe

# 3. 存活探针依赖外部服务
# 问题: 外部服务不可用时 Pod 被重启
# 解决: 存活探针只检查应用自身状态
```

## 日志和监控

### 日志配置

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 日志配置
logging:
  format: json
  verbosity: 2
# 容器日志
containerLogMaxSize: "100Mi"
containerLogMaxFiles: 5
```

### 关键监控指标

```yaml
# Kubelet Prometheus 指标

# Pod 启动延迟
- kubelet_pod_start_duration_seconds_bucket
- kubelet_pod_worker_duration_seconds_bucket

# 容器操作
- kubelet_container_operations_duration_seconds_bucket
- kubelet_container_operations_errors_total

# 运行时操作
- kubelet_runtime_operations_duration_seconds_bucket
- kubelet_runtime_operations_errors_total

# 卷操作
- kubelet_volume_stats_capacity_bytes
- kubelet_volume_stats_used_bytes
- storage_operation_duration_seconds_bucket

# PLEG
- kubelet_pleg_relist_duration_seconds_bucket
- kubelet_pleg_relist_interval_seconds_bucket

# 资源使用
- kubelet_node_status_capacity
- kubelet_node_status_allocatable
```

### 告警规则

```yaml
groups:
  - name: kubelet
    rules:
      # Pod 启动延迟高
      - alert: PodStartLatencyHigh
        expr: |
          histogram_quantile(0.99,
            sum(rate(kubelet_pod_start_duration_seconds_bucket[5m])) by (le)
          ) > 30
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Pod 启动延迟过高"

      # PLEG 不健康
      - alert: PLEGNotHealthy
        expr: |
          histogram_quantile(0.99,
            sum(rate(kubelet_pleg_relist_duration_seconds_bucket[5m])) by (le)
          ) > 10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "PLEG 不健康，可能导致节点 NotReady"

      # 容器运行时错误
      - alert: ContainerRuntimeErrors
        expr: |
          rate(kubelet_runtime_operations_errors_total[5m]) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "容器运行时错误率过高"

      # 磁盘压力
      - alert: NodeDiskPressure
        expr: |
          (kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes) < 0.15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "节点磁盘空间不足"
```

## 配置示例

### 通用节点配置

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
# 基本配置
address: 0.0.0.0
port: 10250
readOnlyPort: 0  # 禁用只读端口
# 认证
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
    cacheTTL: 2m
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m
    cacheUnauthorizedTTL: 30s
# 资源配置
maxPods: 110
cgroupDriver: systemd
systemReserved:
  cpu: "500m"
  memory: "1Gi"
kubeReserved:
  cpu: "500m"
  memory: "1Gi"
# 驱逐配置
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
# 镜像配置
serializeImagePulls: false
maxParallelImagePulls: 5
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
# 日志配置
containerLogMaxSize: "100Mi"
containerLogMaxFiles: 5
```

### 高密度节点配置

```yaml
# 用于运行大量小 Pod 的节点
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
maxPods: 250
# 增加并行处理能力
serializeImagePulls: false
maxParallelImagePulls: 10
# 增加资源预留
systemReserved:
  cpu: "1"
  memory: "2Gi"
kubeReserved:
  cpu: "1"
  memory: "2Gi"
# 更激进的 GC
imageGCHighThresholdPercent: 80
imageGCLowThresholdPercent: 70
```

## 故障排查

### 常见问题

```bash
# 1. PLEG 不健康
# 检查 PLEG 延迟
kubectl get --raw /metrics | grep pleg

# 检查容器运行时
crictl info
crictl ps -a

# 2. Pod 启动慢
# 检查 Pod 启动延迟
kubectl get --raw /metrics | grep pod_start

# 检查镜像拉取
journalctl -u kubelet | grep -i pull

# 3. 驱逐问题
# 检查驱逐事件
kubectl get events --field-selector reason=Evicted

# 检查节点状态
kubectl describe node <node> | grep -A 10 Conditions

# 4. 资源压力
# 检查节点资源
kubectl top node
kubectl describe node <node> | grep -A 5 "Allocated resources"
```

## 总结

Kubelet 调优核心要点：

**同步配置**
- 合理设置同步周期
- 控制并行度
- 优化 PLEG

**资源管理**
- 配置资源预留
- 使用 systemd cgroup 驱动
- 设置合理的驱逐阈值

**容器运行时**
- 并行镜像拉取
- 配置镜像 GC
- 优化 CRI 超时

**监控告警**
- 监控 Pod 启动延迟
- 监控 PLEG 健康
- 监控资源使用
