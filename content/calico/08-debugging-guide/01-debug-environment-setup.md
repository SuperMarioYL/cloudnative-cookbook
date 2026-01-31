---
title: "调试环境搭建"
weight: 01
---


## 概述

本章介绍如何搭建 Calico 调试环境，包括本地开发环境配置、日志级别调整、pprof 性能分析和健康检查端点的使用。

## 前置知识

- Kubernetes 基础操作
- Go 语言开发环境
- Docker/容器运行时
- 基本的 Linux 调试工具

## 本地开发环境

### 使用 Kind 搭建测试集群

```bash
# 1. 创建 Kind 配置
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true  # 禁用默认 CNI
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/16
nodes:
  - role: control-plane
  - role: worker
  - role: worker
EOF

# 2. 创建集群
kind create cluster --config kind-config.yaml --name calico-dev

# 3. 安装 Calico
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/custom-resources.yaml

# 4. 等待 Calico 就绪
kubectl wait --for=condition=Ready pods -l k8s-app=calico-node -n calico-system --timeout=120s
```

### 构建本地 Felix

```bash
# 1. 克隆仓库
git clone https://github.com/projectcalico/calico.git
cd calico

# 2. 构建 Felix
cd felix
make build

# 3. 构建 Docker 镜像
make image

# 4. 将镜像加载到 Kind
kind load docker-image calico/felix:latest --name calico-dev
```

### 本地运行 Felix

```bash
# 设置必要的环境变量
export FELIX_DATASTORETYPE=kubernetes
export FELIX_LOGSEVERITYSCREEN=Debug
export FELIX_PROMETHEUSMETRICSENABLED=true
export FELIX_HEALTHENABLED=true
export FELIX_HEALTHPORT=9099
export KUBECONFIG=/path/to/kubeconfig

# 运行 Felix
./bin/calico-felix
```

## 日志配置

### FelixConfiguration 日志设置

```yaml
apiVersion: projectcalico.org/v3
kind: FelixConfiguration
metadata:
  name: default
spec:
  # 屏幕日志级别
  logSeverityScreen: Debug  # Trace, Debug, Info, Warning, Error, Fatal

  # 文件日志级别
  logSeverityFile: Info
  logFilePath: /var/log/calico/felix.log

  # Syslog 日志级别
  logSeveritySys: Info

  # Debug 文件名正则（仅对 Debug 级别生效）
  logDebugFilenameRegex: ""
```

### 日志级别说明

| 级别 | 说明 | 使用场景 |
|------|------|----------|
| Trace | 最详细，包含所有细节 | 深度调试 |
| Debug | 调试信息 | 开发和问题排查 |
| Info | 正常运行信息 | 生产环境监控 |
| Warning | 警告信息 | 潜在问题提示 |
| Error | 错误信息 | 需要关注的问题 |
| Fatal | 致命错误 | 程序终止 |

### 动态调整日志级别

```bash
# 使用 calicoctl 修改
kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"logSeverityScreen":"Debug"}}'

# 验证修改
kubectl get felixconfiguration default -o yaml | grep logSeverity
```

### 组件特定日志

```bash
# 查看 Felix 日志
kubectl logs -n calico-system ds/calico-node -c calico-node -f

# 查看 Typha 日志
kubectl logs -n calico-system deploy/calico-typha -f

# 查看 kube-controllers 日志
kubectl logs -n calico-system deploy/calico-kube-controllers -f

# 过滤特定关键词
kubectl logs -n calico-system ds/calico-node -c calico-node | grep -i "policy\|error"
```

## 健康检查

### HealthAggregator 架构

```go
// 文件: libcalico-go/lib/health/health.go:57-76

// HealthReport 健康报告结构
type HealthReport struct {
    Live   bool    // 存活状态
    Ready  bool    // 就绪状态
    Detail string  // 详细信息
}

// HealthAggregator 聚合多个组件的健康状态
type HealthAggregator struct {
    mutex        *sync.Mutex
    lastReport   *HealthReport
    reporters    map[string]*reporterState  // 各组件状态
    httpServeMux *http.ServeMux
    httpServer   *http.Server
    everReady    bool
}
```

### 注册健康报告

```go
// 文件: libcalico-go/lib/health/health.go:177-188

// RegisterReporter 注册健康报告器
func (aggregator *HealthAggregator) RegisterReporter(name string, reports *HealthReport, timeout time.Duration) {
    aggregator.mutex.Lock()
    defer aggregator.mutex.Unlock()
    aggregator.reporters[name] = &reporterState{
        name:      name,
        reports:   *reports,
        timeout:   timeout,            // 超时时间
        latest:    HealthReport{Live: true},
        timestamp: time.Now(),
    }
}
```

### 健康端点

```go
// 文件: libcalico-go/lib/health/health.go:231-249

func NewHealthAggregator() *HealthAggregator {
    aggregator := &HealthAggregator{...}

    // /readiness 端点
    aggregator.httpServeMux.HandleFunc("/readiness", func(rsp http.ResponseWriter, req *http.Request) {
        summary := aggregator.Summary()
        genResponse(rsp, "ready", summary.Ready, summary.Detail)
    })

    // /liveness 端点
    aggregator.httpServeMux.HandleFunc("/liveness", func(rsp http.ResponseWriter, req *http.Request) {
        summary := aggregator.Summary()
        genResponse(rsp, "live", summary.Live, summary.Detail)
    })

    return aggregator
}
```

### 使用健康端点

```bash
# 启用健康检查
kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"healthEnabled":true,"healthPort":9099}}'

# 访问健康端点
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- \
  wget -qO- http://localhost:9099/liveness

kubectl exec -it -n calico-system ds/calico-node -c calico-node -- \
  wget -qO- http://localhost:9099/readiness
```

### 健康状态表格

```
+---------------------+---------+-----------+-----------+--------+
|      COMPONENT      | TIMEOUT | LIVENESS  | READINESS | DETAIL |
+---------------------+---------+-----------+-----------+--------+
| async-calc-graph    | 20s     | live      | ready     |        |
| dataplane-resync    | 90s     | live      | ready     |        |
| felix-startup       | none    | live      | ready     |        |
| int-dataplane       | 90s     | live      | ready     |        |
+---------------------+---------+-----------+-----------+--------+
```

## pprof 性能分析

### 启用 Debug 端口

```yaml
apiVersion: projectcalico.org/v3
kind: FelixConfiguration
metadata:
  name: default
spec:
  debugPort: 6060        # pprof 端口
  debugHost: localhost   # 绑定地址
```

### 端口转发

```bash
# 找到 calico-node Pod
POD=$(kubectl get pod -n calico-system -l k8s-app=calico-node -o jsonpath='{.items[0].metadata.name}')

# 端口转发
kubectl port-forward -n calico-system $POD 6060:6060 &
```

### CPU Profile

```bash
# 收集 30 秒 CPU Profile
curl -o cpu.prof http://localhost:6060/debug/pprof/profile?seconds=30

# 使用 go tool 分析
go tool pprof -http=:8080 cpu.prof

# 或者命令行分析
go tool pprof cpu.prof
(pprof) top 20
(pprof) list <function>
(pprof) web
```

### 内存 Profile

```bash
# 收集堆内存 Profile
curl -o heap.prof http://localhost:6060/debug/pprof/heap

# 分析
go tool pprof -http=:8080 heap.prof

# 查看分配统计
go tool pprof -alloc_space heap.prof

# 查看使用中的内存
go tool pprof -inuse_space heap.prof
```

### Goroutine Profile

```bash
# 查看 Goroutine 堆栈
curl http://localhost:6060/debug/pprof/goroutine?debug=2 | head -100

# 收集 Goroutine Profile
curl -o goroutine.prof http://localhost:6060/debug/pprof/goroutine

# 分析
go tool pprof goroutine.prof
```

### 阻塞 Profile

```bash
# 查看阻塞情况
curl -o block.prof http://localhost:6060/debug/pprof/block
go tool pprof block.prof
```

### 实时监控

```bash
# 查看运行时统计
curl http://localhost:6060/debug/pprof/

# 查看命令行参数
curl http://localhost:6060/debug/pprof/cmdline

# 查看符号表
curl http://localhost:6060/debug/pprof/symbol
```

## Prometheus 指标

### 启用 Prometheus

```yaml
apiVersion: projectcalico.org/v3
kind: FelixConfiguration
metadata:
  name: default
spec:
  prometheusMetricsEnabled: true
  prometheusMetricsHost: ""  # 绑定所有接口
  prometheusMetricsPort: 9091

  # 可选：启用 Go 和进程指标
  prometheusGoMetricsEnabled: true
  prometheusProcessMetricsEnabled: true
```

### 访问指标

```bash
# 端口转发
kubectl port-forward -n calico-system ds/calico-node 9091:9091

# 获取指标
curl http://localhost:9091/metrics | head -50
```

### 关键指标

```prometheus
# 数据平面同步状态
felix_resync_state

# 策略规则数量
felix_active_local_policies
felix_active_local_endpoints

# iptables 规则数量
felix_iptables_rules

# IP 集合成员数量
felix_ipset_members

# 数据平面错误
felix_int_dataplane_failures

# 计算图事件
felix_calc_graph_update_time_seconds
```

### Prometheus 抓取配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'calico-felix'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_k8s_app]
        regex: calico-node
        action: keep
      - source_labels: [__meta_kubernetes_pod_container_port_number]
        regex: "9091"
        action: keep
```

## 调试工具

### calicoctl

```bash
# 安装 calicoctl
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calicoctl.yaml

# 使用 kubectl exec
kubectl exec -it -n kube-system calicoctl -- /calicoctl get nodes

# 或者本地安装
curl -L https://github.com/projectcalico/calico/releases/download/v3.27.0/calicoctl-linux-amd64 -o calicoctl
chmod +x calicoctl
sudo mv calicoctl /usr/local/bin/
```

### 常用调试命令

```bash
# 查看节点状态
calicoctl node status

# 查看 IPAM 状态
calicoctl ipam show
calicoctl ipam show --show-blocks

# 查看工作负载端点
calicoctl get workloadendpoint -A -o wide

# 查看策略
calicoctl get networkpolicy -A -o yaml
calicoctl get globalnetworkpolicy -o yaml

# 查看 BGP 对等
calicoctl get bgppeer -o yaml
```

### 网络调试

```bash
# 在节点上查看 iptables 规则
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- \
  iptables-save | grep -i cali

# 查看 IP 集合
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- \
  ipset list | head -50

# 查看路由
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- \
  ip route

# 查看 BIRD 状态
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- \
  birdcl show protocols
```

## 超时配置覆盖

### HealthTimeoutOverrides

```yaml
apiVersion: projectcalico.org/v3
kind: FelixConfiguration
metadata:
  name: default
spec:
  # 覆盖健康检查超时
  healthTimeoutOverrides:
    - name: async-calc-graph
      timeout: 60s
    - name: int-dataplane
      timeout: 180s
    - name: dataplane-resync
      timeout: 180s
```

### 代码实现

```go
// 文件: libcalico-go/lib/health/health.go:37-55

func SetGlobalTimeoutOverrides(overrides map[string]time.Duration) {
    overridesCopy := map[string]time.Duration{}
    for k, v := range overrides {
        overridesCopy[k] = v
    }
    globalOverridesLock.Lock()
    defer globalOverridesLock.Unlock()
    globalTimeoutOverrides = overrides
}

func GlobalOverride(name string) *time.Duration {
    globalOverridesLock.Lock()
    defer globalOverridesLock.Unlock()
    override, ok := globalTimeoutOverrides[name]
    if ok {
        return &override
    }
    return nil
}
```

## 实验

### 实验 1：启用调试日志

```bash
# 1. 修改日志级别
kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"logSeverityScreen":"Debug"}}'

# 2. 查看日志
kubectl logs -n calico-system ds/calico-node -c calico-node -f | head -100

# 3. 恢复正常日志级别
kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"logSeverityScreen":"Info"}}'
```

### 实验 2：收集性能 Profile

```bash
# 1. 启用 debug 端口
kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"debugPort":6060}}'

# 2. 端口转发
POD=$(kubectl get pod -n calico-system -l k8s-app=calico-node -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward -n calico-system $POD 6060:6060 &

# 3. 收集 CPU Profile
curl -o cpu.prof http://localhost:6060/debug/pprof/profile?seconds=10

# 4. 分析
go tool pprof -http=:8080 cpu.prof
```

### 实验 3：检查健康状态

```bash
# 1. 启用健康检查
kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"healthEnabled":true,"healthPort":9099}}'

# 2. 检查存活状态
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- \
  wget -qO- http://localhost:9099/liveness

# 3. 检查就绪状态
kubectl exec -it -n calico-system ds/calico-node -c calico-node -- \
  wget -qO- http://localhost:9099/readiness
```

### 实验 4：Prometheus 指标

```bash
# 1. 启用 Prometheus
kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"prometheusMetricsEnabled":true,"prometheusMetricsPort":9091}}'

# 2. 端口转发
kubectl port-forward -n calico-system ds/calico-node 9091:9091 &

# 3. 获取指标
curl -s http://localhost:9091/metrics | grep felix_ | head -20
```

## 总结

Calico 提供了丰富的调试工具：

1. **日志系统** - 多级别、多输出的日志配置
2. **健康检查** - 组件级别的存活和就绪检查
3. **pprof** - CPU、内存、Goroutine 性能分析
4. **Prometheus** - 详细的运行时指标
5. **calicoctl** - 专用的管理和调试工具

## 参考资料

- [Calico 故障排除](https://docs.tigera.io/calico/latest/operations/troubleshoot/)
- [Go pprof 文档](https://pkg.go.dev/net/http/pprof)
- `libcalico-go/lib/health/health.go` - 健康检查实现
- `felix/daemon/daemon.go` - Felix 入口和配置
