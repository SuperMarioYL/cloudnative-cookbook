---
title: "网络调优"
weight: 6
---
## 概述

Kubernetes 网络性能影响 Pod 间通信、Service 访问和外部流量处理。网络调优涉及 CNI 插件选择、kube-proxy 模式配置、DNS 优化等多个方面。

## CNI 插件优化

### 插件选择

```
┌─────────────────────────────────────────────────────────────────┐
│                    CNI 插件性能对比                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Calico:                                                         │
│  ├── 支持 BGP 和 VXLAN                                           │
│  ├── 高性能 eBPF 数据平面（可选）                                │
│  └── 丰富的网络策略支持                                          │
│                                                                  │
│  Cilium:                                                         │
│  ├── 原生 eBPF 实现                                              │
│  ├── 高性能 Service 代理                                         │
│  └── 深度可观测性                                                │
│                                                                  │
│  Flannel:                                                        │
│  ├── 简单易用                                                    │
│  ├── VXLAN 或 host-gw 模式                                       │
│  └── 适合小规模集群                                              │
│                                                                  │
│  Weave:                                                          │
│  ├── 自动网络发现                                                │
│  ├── 加密支持                                                    │
│  └── 中等性能                                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Calico 优化

```yaml
# Calico 配置优化
apiVersion: projectcalico.org/v3
kind: FelixConfiguration
metadata:
  name: default
spec:
  # 使用 eBPF 数据平面
  bpfEnabled: true
  # 绕过 iptables
  bpfDisableUnprivilegedBPF: false
  # eBPF 日志级别
  bpfLogLevel: "Off"
  # IP 转发
  ipForwarding: Enabled
  # 日志级别
  logSeverityScreen: Warning
  # 报告间隔
  reportingInterval: 30s
---
# IPAM 配置
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: default-ipv4-ippool
spec:
  cidr: 10.244.0.0/16
  # 使用 VXLAN
  vxlanMode: Always
  # 或使用 IPIP
  # ipipMode: Always
  natOutgoing: true
  blockSize: 26
```

### Cilium 优化

```yaml
# Cilium Helm values
cilium:
  # 使用 eBPF kube-proxy 替代
  kubeProxyReplacement: strict
  # eBPF 主机路由
  bpf:
    masquerade: true
    tproxy: true
  # 启用 Hubble 可观测性
  hubble:
    enabled: true
    relay:
      enabled: true
  # 性能调优
  tunnel: disabled  # 使用原生路由
  autoDirectNodeRoutes: true
  enableIPv4Masquerade: true
  # 负载均衡
  loadBalancer:
    mode: dsr  # Direct Server Return
```

## kube-proxy 优化

### 模式选择

```yaml
# kube-proxy 配置
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
# 模式选择
mode: ipvs  # iptables | ipvs | nftables

# 模式对比:
# iptables:
#   - 默认模式
#   - O(N) 复杂度
#   - 适合小规模 (< 1000 Services)
#
# ipvs:
#   - O(1) 查找复杂度
#   - 更多负载均衡算法
#   - 适合大规模集群
#
# nftables:
#   - 现代 Linux 防火墙
#   - 原子更新
#   - 需要较新内核
```

### IPVS 配置

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
ipvs:
  # 调度算法
  scheduler: rr  # rr | lc | dh | sh | sed | nq
  # 同步周期
  syncPeriod: 30s
  # 最小同步间隔
  minSyncPeriod: 1s
  # TCP 超时
  tcpTimeout: 0s
  tcpFinTimeout: 0s
  udpTimeout: 0s
  # 严格 ARP
  strictARP: true
  # 排除 CIDR
  excludeCIDRs: []
```

### iptables 配置

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: iptables
iptables:
  # 同步周期
  syncPeriod: 30s
  # 最小同步间隔
  minSyncPeriod: 1s
  # 伪装位
  masqueradeBit: 14
  # 本地主机掩码
  localhostNodePorts: true
```

### 连接跟踪优化

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
conntrack:
  # 最大连接数（每核）
  maxPerCore: 32768
  # 最小连接数
  min: 131072
  # TCP 超时
  tcpEstablishedTimeout: 24h
  tcpCloseWaitTimeout: 1h
```

```bash
# 系统级连接跟踪调优
# /etc/sysctl.conf

# 增加连接跟踪表大小
net.netfilter.nf_conntrack_max = 1048576
net.nf_conntrack_max = 1048576

# 减少超时时间
net.netfilter.nf_conntrack_tcp_timeout_established = 86400
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 3600
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
```

## DNS 优化

### CoreDNS 配置

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
          lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          fallthrough in-addr.arpa ip6.arpa
          # 增加 TTL
          ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf {
          # 增加缓存
          max_concurrent 1000
        }
        # 增大缓存
        cache 300 {
          success 9984
          denial 9984
        }
        loop
        reload
        loadbalance
    }
```

### 本地 DNS 缓存

```yaml
# 部署 NodeLocal DNSCache
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-local-dns
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: node-local-dns
  template:
    spec:
      hostNetwork: true
      dnsPolicy: Default
      containers:
        - name: node-cache
          image: registry.k8s.io/dns/k8s-dns-node-cache:1.22.23
          args:
            - -localip
            - "169.254.20.10"
            - -conf
            - /etc/Corefile
            - -upstreamsvc
            - kube-dns
```

### Pod DNS 配置

```yaml
apiVersion: v1
kind: Pod
spec:
  # DNS 策略
  dnsPolicy: ClusterFirst
  # 自定义 DNS 配置
  dnsConfig:
    # 减少 ndots 避免不必要的搜索
    options:
      - name: ndots
        value: "2"
      # 单查询超时
      - name: timeout
        value: "2"
      # 重试次数
      - name: attempts
        value: "2"
```

## Service 优化

### EndpointSlice 配置

```yaml
# 确保启用 EndpointSlice
# 默认在 1.21+ 启用

# 优势:
# - 分片存储减少大小
# - 增量更新
# - 更好的扩展性
```

### 拓扑感知路由

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    # 启用拓扑感知提示
    service.kubernetes.io/topology-mode: Auto
spec:
  selector:
    app: my-app
  ports:
    - port: 80
  # 内部流量策略
  internalTrafficPolicy: Local
```

### 会话亲和性

```yaml
apiVersion: v1
kind: Service
spec:
  # 会话亲和性
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800  # 3 小时
```

## 网络策略优化

### 策略设计

```yaml
# 高效的网络策略设计

# 使用命名空间级别的默认策略
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# 精确的 Pod 选择器
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-app
spec:
  podSelector:
    matchLabels:
      app: my-app
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: frontend
      ports:
        - port: 8080
```

### 性能考虑

```yaml
# 网络策略性能建议:

# 1. 避免过多策略
# - 每个 Pod 应用的策略数量影响性能
# - 使用合并策略而非多个小策略

# 2. 使用高效的选择器
# - 优先使用 namespaceSelector
# - 避免使用 ipBlock (需要更多规则)

# 3. 限制端口范围
# - 指定具体端口而非端口范围
```

## 监控指标

### 网络指标

```yaml
# Prometheus 网络指标

# kube-proxy 指标
- kubeproxy_sync_proxy_rules_duration_seconds
- kubeproxy_network_programming_duration_seconds
- kubeproxy_sync_proxy_rules_iptables_total

# CoreDNS 指标
- coredns_dns_request_duration_seconds
- coredns_dns_responses_total
- coredns_cache_hits_total
- coredns_cache_misses_total

# 节点网络
- node_network_receive_bytes_total
- node_network_transmit_bytes_total
- node_network_receive_packets_total
- node_network_transmit_packets_total
```

### 告警规则

```yaml
groups:
  - name: network
    rules:
      # kube-proxy 同步慢
      - alert: KubeProxySyncSlow
        expr: |
          histogram_quantile(0.99,
            sum(rate(kubeproxy_sync_proxy_rules_duration_seconds_bucket[5m])) by (le)
          ) > 1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "kube-proxy 规则同步过慢"

      # DNS 延迟高
      - alert: DNSLatencyHigh
        expr: |
          histogram_quantile(0.99,
            sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) by (le)
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "DNS 查询延迟过高"

      # DNS 错误率高
      - alert: DNSErrorsHigh
        expr: |
          sum(rate(coredns_dns_responses_total{rcode!="NOERROR"}[5m])) /
          sum(rate(coredns_dns_responses_total[5m])) > 0.01
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "DNS 错误率过高"
```

## 内核参数调优

### 网络相关 sysctl

```bash
# /etc/sysctl.d/99-kubernetes-network.conf

# 连接跟踪
net.netfilter.nf_conntrack_max = 1048576
net.nf_conntrack_max = 1048576

# TCP 调优
net.core.somaxconn = 32768
net.core.netdev_max_backlog = 16384
net.ipv4.tcp_max_syn_backlog = 8096
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_tw_reuse = 1

# IP 转发
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1

# 本地端口范围
net.ipv4.ip_local_port_range = 1024 65535

# TCP keepalive
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10

# 文件描述符
fs.file-max = 2097152
fs.inotify.max_user_watches = 524288
fs.inotify.max_user_instances = 512
```

## 配置示例

### 大规模集群网络配置

```yaml
# kube-proxy 配置
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs
ipvs:
  scheduler: rr
  syncPeriod: 30s
  minSyncPeriod: 1s
  strictARP: true
conntrack:
  maxPerCore: 65536
  min: 524288
---
# CoreDNS 配置 (高性能)
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
          pods insecure
          ttl 60
        }
        forward . /etc/resolv.conf
        cache 600 {
          success 19968
          denial 19968
        }
        loop
        reload
        loadbalance
    }
```

## 总结

网络调优核心要点：

**CNI 选择**
- 小规模: Flannel
- 中规模: Calico
- 大规模/高性能: Cilium

**kube-proxy 优化**
- 大规模使用 IPVS
- 调整连接跟踪参数
- 考虑 eBPF 替代方案

**DNS 优化**
- 增加 CoreDNS 缓存
- 部署 NodeLocal DNSCache
- 优化 ndots 配置

**系统调优**
- 调整内核网络参数
- 增加连接跟踪表大小
- 优化 TCP 参数
