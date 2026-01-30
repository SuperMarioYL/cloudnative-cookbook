---
title: "API Server 调优"
weight: 2
---
## 概述

API Server 是 Kubernetes 集群的核心组件，所有组件和用户都通过它与集群交互。API Server 的性能直接影响整个集群的响应速度和稳定性。本章详细介绍 API Server 的性能调优策略。

## 请求处理优化

### 并发请求限制

```bash
# API Server 并发配置
kube-apiserver \
  # 只读请求最大并发数
  --max-requests-inflight=400 \
  # 修改请求最大并发数
  --max-mutating-requests-inflight=200
```

```
并发限制说明:

max-requests-inflight (只读请求)
├── 默认值: 400
├── 影响: GET, LIST, WATCH 等请求
└── 建议: 根据节点规模调整 (1000 节点可设为 800)

max-mutating-requests-inflight (修改请求)
├── 默认值: 200
├── 影响: CREATE, UPDATE, PATCH, DELETE 等请求
└── 建议: 通常为只读请求的 50%
```

### 优先级和公平性 (APF)

```yaml
# 启用 API 优先级和公平性
kube-apiserver \
  --enable-priority-and-fairness=true

# FlowSchema 示例 - 系统关键请求
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: system-critical
spec:
  priorityLevelConfiguration:
    name: system
  matchingPrecedence: 100
  rules:
    - subjects:
        - kind: User
          user:
            name: system:kube-controller-manager
        - kind: User
          user:
            name: system:kube-scheduler
      resourceRules:
        - verbs: ["*"]
          apiGroups: ["*"]
          resources: ["*"]
          namespaces: ["*"]
---
# PriorityLevelConfiguration 示例
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: system
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 30
    limitResponse:
      type: Queue
      queuing:
        queues: 64
        handSize: 8
        queueLengthLimit: 50
```

### 请求超时配置

```bash
kube-apiserver \
  # 请求超时
  --request-timeout=60s \
  # 最小请求超时
  --min-request-timeout=1800
```

## Watch 机制优化

### Watch 缓存配置

```bash
kube-apiserver \
  # Watch 缓存大小配置
  --watch-cache-sizes=pods#1000,nodes#500,services#500 \
  # 默认 Watch 缓存大小
  --default-watch-cache-size=100
```

```
Watch 缓存说明:

缓存结构:
├── 存储最近的资源变更事件
├── 客户端可以从缓存中恢复 Watch
└── 减少 etcd 访问压力

配置格式: resource#size
├── pods#1000: pods 资源缓存 1000 个事件
├── nodes#500: nodes 资源缓存 500 个事件
└── 根据资源访问频率调整
```

### Watch 连接管理

```go
// Watch 连接优化建议

// 1. 使用 ResourceVersion 避免全量同步
watchOptions := metav1.ListOptions{
    ResourceVersion: lastKnownRV,
    Watch:           true,
}

// 2. 处理 BOOKMARK 事件
switch event.Type {
case watch.Bookmark:
    // 更新 ResourceVersion
    lastKnownRV = event.Object.GetResourceVersion()
}

// 3. 使用 LabelSelector 减少数据量
watchOptions := metav1.ListOptions{
    LabelSelector: "app=myapp",
    Watch:         true,
}
```

### Bookmark 事件

```bash
# 启用 Bookmark（默认启用）
# Bookmark 帮助客户端跟踪 ResourceVersion 进度

# 发送间隔配置（API Server 源码）
# 默认每分钟发送一次 Bookmark
```

## etcd 交互优化

### etcd 客户端配置

```bash
kube-apiserver \
  # etcd 服务器列表
  --etcd-servers=https://etcd1:2379,https://etcd2:2379,https://etcd3:2379 \
  # etcd 请求超时
  --etcd-healthcheck-timeout=10s \
  # etcd 读取超时
  --etcd-readycheck-timeout=10s
```

### 压缩和存储

```bash
kube-apiserver \
  # 启用 etcd 压缩
  --etcd-compaction-interval=5m
```

### 存储版本

```yaml
# 内部版本与存储版本
# API Server 在内存中使用内部版本
# 存储到 etcd 时转换为存储版本

# 检查存储版本
kubectl get --raw /apis/apps/v1/deployments | jq '.apiVersion'
```

## 序列化优化

### Protobuf 支持

```go
// 客户端配置使用 Protobuf
config := rest.Config{
    Host: "https://kubernetes",
    ContentType: "application/vnd.kubernetes.protobuf",
    AcceptContentTypes: "application/vnd.kubernetes.protobuf,application/json",
}

// Protobuf 优势:
// - 更小的消息体积
// - 更快的序列化/反序列化
// - 减少网络传输
```

### 响应压缩

```bash
# API Server 默认启用 gzip 压缩
# 客户端通过 Accept-Encoding 头请求压缩
curl -H "Accept-Encoding: gzip" \
  https://api-server/api/v1/pods
```

## 认证授权优化

### 认证缓存

```bash
kube-apiserver \
  # Token 认证缓存 TTL
  --authentication-token-webhook-cache-ttl=2m
```

### RBAC 授权缓存

```go
// RBAC 授权器内部缓存
// 缓存授权决策减少重复计算

// 建议:
// 1. 避免过于细粒度的 RBAC 规则
// 2. 使用 ClusterRole 聚合减少规则数量
// 3. 合理使用 resourceNames 限制
```

### Webhook 优化

```bash
kube-apiserver \
  # Webhook 超时
  --authentication-token-webhook-cache-ttl=2m \
  # 授权 Webhook 缓存
  --authorization-webhook-cache-authorized-ttl=5m \
  --authorization-webhook-cache-unauthorized-ttl=30s
```

## 高可用配置

### 多实例部署

```yaml
# API Server 多实例负载均衡
apiVersion: v1
kind: Service
metadata:
  name: kubernetes
  namespace: default
spec:
  type: ClusterIP
  ports:
    - port: 443
      targetPort: 6443
  selector:
    component: kube-apiserver
```

### 负载均衡策略

```
API Server 负载均衡:

L4 负载均衡 (推荐):
├── 基于 TCP 连接分发
├── 支持长连接 (Watch)
└── 实现: HAProxy, Nginx, 云 LB

配置示例 (HAProxy):
frontend kubernetes
  bind *:6443
  mode tcp
  default_backend kubernetes-backend

backend kubernetes-backend
  mode tcp
  balance roundrobin
  option tcp-check
  server api1 10.0.0.1:6443 check
  server api2 10.0.0.2:6443 check
  server api3 10.0.0.3:6443 check
```

## 监控指标

### 关键指标

```yaml
# API Server Prometheus 指标

# 请求延迟分布
- apiserver_request_duration_seconds_bucket
- apiserver_request_duration_seconds_sum
- apiserver_request_duration_seconds_count

# 当前并发请求
- apiserver_current_inflight_requests{request_kind="readOnly"}
- apiserver_current_inflight_requests{request_kind="mutating"}

# 请求计数
- apiserver_request_total{code, verb, resource}

# etcd 请求延迟
- etcd_request_duration_seconds_bucket

# Watch 计数
- apiserver_longrunning_requests{verb="WATCH"}

# APF 指标
- apiserver_flowcontrol_dispatched_requests_total
- apiserver_flowcontrol_rejected_requests_total
```

### 告警规则

```yaml
groups:
  - name: apiserver
    rules:
      # API 请求延迟过高
      - alert: APIServerLatencyHigh
        expr: |
          histogram_quantile(0.99,
            sum(rate(apiserver_request_duration_seconds_bucket{verb!~"WATCH|CONNECT"}[5m])) by (le, verb)
          ) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "API Server 请求延迟过高"

      # API 请求错误率高
      - alert: APIServerErrorsHigh
        expr: |
          sum(rate(apiserver_request_total{code=~"5.."}[5m])) /
          sum(rate(apiserver_request_total[5m])) > 0.01
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "API Server 错误率过高"

      # 并发请求接近限制
      - alert: APIServerInflightRequestsHigh
        expr: |
          apiserver_current_inflight_requests /
          on() group_left() kube_apiserver_max_inflight_requests > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "API Server 并发请求接近限制"

      # etcd 请求延迟高
      - alert: EtcdRequestLatencyHigh
        expr: |
          histogram_quantile(0.99, sum(rate(etcd_request_duration_seconds_bucket[5m])) by (le)) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "etcd 请求延迟过高"
```

## 性能调优参数汇总

### 推荐配置

```bash
# 小规模集群 (< 100 节点)
kube-apiserver \
  --max-requests-inflight=400 \
  --max-mutating-requests-inflight=200 \
  --watch-cache-sizes=pods#500,nodes#100

# 中规模集群 (100-1000 节点)
kube-apiserver \
  --max-requests-inflight=800 \
  --max-mutating-requests-inflight=400 \
  --watch-cache-sizes=pods#2000,nodes#500,services#500

# 大规模集群 (> 1000 节点)
kube-apiserver \
  --max-requests-inflight=1600 \
  --max-mutating-requests-inflight=800 \
  --watch-cache-sizes=pods#5000,nodes#1000,services#1000 \
  --enable-priority-and-fairness=true
```

### 资源配置

```yaml
# API Server Pod 资源配置
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
spec:
  containers:
    - name: kube-apiserver
      resources:
        requests:
          cpu: "2"
          memory: "4Gi"
        limits:
          cpu: "4"
          memory: "8Gi"
      # 根据集群规模调整
      # 大规模集群可能需要 8+ CPU, 16+ GB 内存
```

## 常见问题排查

### 高延迟排查

```bash
# 1. 检查请求延迟分布
kubectl get --raw /metrics | grep apiserver_request_duration

# 2. 检查 etcd 延迟
kubectl get --raw /metrics | grep etcd_request_duration

# 3. 检查并发请求
kubectl get --raw /metrics | grep inflight_requests

# 4. 使用 audit 日志分析慢请求
grep "responseComplete" /var/log/kubernetes/audit.log | \
  jq 'select(.stageTimestamp - .requestReceivedTimestamp > 1)'
```

### 连接问题排查

```bash
# 检查 Watch 连接数
kubectl get --raw /metrics | grep longrunning_requests

# 检查客户端连接
netstat -an | grep 6443 | wc -l

# 检查 API Server 日志
journalctl -u kube-apiserver | grep -i "connection"
```

## 总结

API Server 调优核心要点：

**并发控制**
- 合理设置 max-requests-inflight
- 启用 APF 进行流量整形
- 为关键组件配置优先级

**缓存优化**
- 配置 Watch 缓存大小
- 利用认证授权缓存
- 减少 etcd 访问

**序列化优化**
- 使用 Protobuf 协议
- 启用响应压缩
- 减少大对象传输

**监控告警**
- 监控请求延迟和错误率
- 监控并发请求数
- 监控 etcd 性能

**最佳实践**
- 多实例高可用部署
- 根据规模调整参数
- 定期性能测试和调优
