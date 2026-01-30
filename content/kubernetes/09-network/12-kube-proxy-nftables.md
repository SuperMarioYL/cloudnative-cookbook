---
title: "kube-proxy nftables 模式"
weight: 12
---
## 概述

nftables 是 Linux 内核中 iptables 的现代替代品，提供了更简洁的语法、更好的性能和统一的框架。Kubernetes 1.29 引入了 kube-proxy nftables 模式作为 Beta 功能，旨在利用 nftables 的优势提供更高效的 Service 实现。

## nftables 基础

### nftables vs iptables

```
┌─────────────────────────────────────────────────────────────────┐
│                 nftables vs iptables 对比                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  架构:                                                           │
│  ├── iptables: 独立工具 (iptables, ip6tables, arptables, etc)   │
│  └── nftables: 统一框架 (nft 单一命令)                           │
│                                                                  │
│  语法:                                                           │
│  ├── iptables: 命令行参数风格                                    │
│  └── nftables: 类编程语言语法                                    │
│                                                                  │
│  性能:                                                           │
│  ├── iptables: 规则串行匹配                                      │
│  └── nftables: 支持集合、映射等优化结构                          │
│                                                                  │
│  原子更新:                                                       │
│  ├── iptables: 需要 iptables-restore                            │
│  └── nftables: 原生支持原子事务                                  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### nftables 结构

```
nftables 层次结构:

Table (表)
├── Chain (链)
│   ├── Rule (规则)
│   └── Rule
├── Chain
│   └── Rule
├── Set (集合)
├── Map (映射)
└── Flowtable (流表)
```

### 基本语法

```bash
# 创建表
nft add table ip filter

# 创建链
nft add chain ip filter input { type filter hook input priority 0 \; }

# 添加规则
nft add rule ip filter input tcp dport 80 accept

# 创建集合
nft add set ip filter allowed_ips { type ipv4_addr \; }
nft add element ip filter allowed_ips { 192.168.1.0/24, 10.0.0.0/8 }

# 使用集合
nft add rule ip filter input ip saddr @allowed_ips accept

# 创建映射
nft add map ip nat port_map { type inet_service : ipv4_addr . inet_service \; }
```

## kube-proxy nftables 实现

### 启用 nftables 模式

```yaml
# kube-proxy 配置
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: nftables
nftables:
  # 同步周期
  syncPeriod: 30s
  # 最小同步间隔
  minSyncPeriod: 1s
  # 伪装位
  masqueradeBit: 14
```

```bash
# 或通过命令行启动
kube-proxy --proxy-mode=nftables
```

### 内核要求

```bash
# 检查内核版本（需要 5.13+）
uname -r

# 检查 nftables 模块
lsmod | grep nf_tables

# 加载必要模块
modprobe nf_tables
modprobe nft_chain_nat
modprobe nft_masq
```

## 规则结构

### 表和链

```bash
# kube-proxy 创建的 nftables 结构
table ip kube-proxy {
    # ClusterIP 服务链
    chain services {
        type nat hook prerouting priority dstnat; policy accept;
        # ...
    }

    chain output {
        type nat hook output priority dstnat; policy accept;
        # ...
    }

    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        # ...
    }

    # 过滤规则
    chain firewall-check {
        type filter hook forward priority 0; policy accept;
        # ...
    }

    # 集合定义
    set cluster-ips {
        type ipv4_addr . inet_proto . inet_service
        elements = { ... }
    }

    set nodeport-ips {
        type ipv4_addr
        elements = { ... }
    }

    # 映射定义
    map service-endpoints {
        type ipv4_addr . inet_proto . inet_service : verdict
        elements = { ... }
    }
}
```

### ClusterIP 实现

```bash
# nftables 规则示例
table ip kube-proxy {
    # 服务 IP 集合
    set cluster-ips {
        type ipv4_addr . inet_proto . inet_service
        elements = {
            10.96.0.100 . tcp . 80,
            10.96.0.1 . tcp . 443,
        }
    }

    # 服务到端点的映射
    map svc-my-service {
        type ipv4_addr . inet_proto . inet_service : verdict
        flags interval
        elements = {
            10.96.0.100 . tcp . 80 : jump endpoints-my-service
        }
    }

    # 端点链
    chain endpoints-my-service {
        # 负载均衡到多个端点
        numgen random mod 2 vmap {
            0 : goto endpoint-aaa,
            1 : goto endpoint-bbb
        }
    }

    chain endpoint-aaa {
        # DNAT 到第一个端点
        meta mark set mark or 0x4000
        dnat to 10.244.1.2:8080
    }

    chain endpoint-bbb {
        # DNAT 到第二个端点
        meta mark set mark or 0x4000
        dnat to 10.244.2.3:8080
    }

    # 主服务链
    chain services {
        type nat hook prerouting priority dstnat; policy accept;
        ip daddr . meta l4proto . th dport vmap @svc-my-service
    }
}
```

### NodePort 实现

```bash
table ip kube-proxy {
    # NodePort 端口集合
    set nodeports-tcp {
        type inet_service
        elements = { 30080, 30443 }
    }

    # NodePort 链
    chain nodeports {
        tcp dport @nodeports-tcp jump nodeport-dispatch
    }

    chain nodeport-dispatch {
        tcp dport 30080 jump svc-my-service-nodeport
    }

    chain svc-my-service-nodeport {
        # 标记需要 MASQUERADE
        meta mark set mark or 0x4000
        # 跳转到端点选择
        jump endpoints-my-service
    }
}
```

### MASQUERADE 实现

```bash
table ip kube-proxy {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        # 检查标记并执行 MASQUERADE
        meta mark & 0x4000 == 0x4000 masquerade
    }

    chain mark-for-masquerade {
        meta mark set mark or 0x4000
    }
}
```

## Proxier 实现

### 数据结构

```go
// pkg/proxy/nftables/proxier.go

type Proxier struct {
    // nftables 接口
    nft knftables.Interface

    // 服务和端点信息
    svcPortMap   proxy.ServicePortMap
    endpointsMap proxy.EndpointsMap

    // 同步控制
    syncRunner   *async.BoundedFrequencyRunner
    syncPeriod   time.Duration

    // 节点信息
    nodeIP   net.IP
    hostname string

    // 规则构建
    staleChains map[string]bool
}
```

### syncProxyRules

```go
// syncProxyRules 同步 nftables 规则
func (proxier *Proxier) syncProxyRules() {
    proxier.mu.Lock()
    defer proxier.mu.Unlock()

    // 1. 处理服务和端点变更
    serviceUpdateResult := proxier.svcPortMap.Update(proxier.serviceChanges)
    endpointUpdateResult := proxier.endpointsMap.Update(proxier.endpointsChanges)

    // 2. 创建 nftables 事务
    tx := proxier.nft.NewTransaction()

    // 3. 确保基础结构存在
    proxier.ensureBaseRules(tx)

    // 4. 构建服务规则
    activeServices := sets.NewString()
    for svcName, svc := range proxier.svcPortMap {
        svcInfo := svc.(*servicePortInfo)
        proxier.writeServiceRules(tx, svcName, svcInfo, activeServices)
    }

    // 5. 清理过时规则
    proxier.cleanupStaleRules(tx, activeServices)

    // 6. 提交事务
    if err := proxier.nft.Run(context.TODO(), tx); err != nil {
        klog.ErrorS(err, "Failed to sync nftables rules")
    }
}

// writeServiceRules 写入服务规则
func (proxier *Proxier) writeServiceRules(
    tx *knftables.Transaction,
    svcName proxy.ServicePortName,
    svcInfo *servicePortInfo,
    activeServices sets.String) {

    // 获取端点
    endpoints := proxier.endpointsMap[svcName]
    nEndpoints := len(endpoints)

    if nEndpoints == 0 {
        // 无端点，跳过或添加拒绝规则
        return
    }

    // 服务链名称
    svcChain := svcInfo.chainName()
    activeServices.Insert(svcChain)

    // 创建服务链
    tx.Add(&knftables.Chain{
        Name: svcChain,
    })

    // 创建端点链
    epChains := make([]string, nEndpoints)
    for i, ep := range endpoints {
        epInfo := ep.(*endpointInfo)
        epChain := epInfo.chainName()
        epChains[i] = epChain
        activeServices.Insert(epChain)

        // 创建端点链
        tx.Add(&knftables.Chain{
            Name: epChain,
        })

        // 标记 MASQUERADE
        if proxier.needsMasquerade(svcInfo, epInfo) {
            tx.Add(&knftables.Rule{
                Chain: epChain,
                Rule:  "meta mark set mark or 0x4000",
            })
        }

        // DNAT 规则
        tx.Add(&knftables.Rule{
            Chain: epChain,
            Rule: fmt.Sprintf("dnat to %s:%d",
                epInfo.ip, epInfo.port),
        })
    }

    // 负载均衡规则
    if nEndpoints == 1 {
        tx.Add(&knftables.Rule{
            Chain: svcChain,
            Rule:  fmt.Sprintf("jump %s", epChains[0]),
        })
    } else {
        // 使用 numgen 实现随机负载均衡
        vmap := make([]string, nEndpoints)
        for i, chain := range epChains {
            vmap[i] = fmt.Sprintf("%d : goto %s", i, chain)
        }
        tx.Add(&knftables.Rule{
            Chain: svcChain,
            Rule: fmt.Sprintf("numgen random mod %d vmap { %s }",
                nEndpoints, strings.Join(vmap, ", ")),
        })
    }

    // ClusterIP 规则
    tx.Add(&knftables.Element{
        Set: "cluster-ips",
        Key: fmt.Sprintf("%s . %s . %d",
            svcInfo.ClusterIP(), svcInfo.Protocol(), svcInfo.Port()),
    })

    tx.Add(&knftables.Element{
        Map:   "service-endpoints",
        Key:   fmt.Sprintf("%s . %s . %d", svcInfo.ClusterIP(), svcInfo.Protocol(), svcInfo.Port()),
        Value: fmt.Sprintf("jump %s", svcChain),
    })
}
```

## 性能优势

### 与 iptables 对比

| 特性 | iptables | nftables |
|------|----------|----------|
| 规则语法 | 命令行参数 | 类编程语法 |
| 集合支持 | 需要 ipset | 原生支持 |
| 映射支持 | 不支持 | 原生支持 |
| 原子更新 | iptables-restore | 原生事务 |
| 规则复杂度 | O(N) | O(1) 集合/映射 |
| 调试友好 | 较差 | 较好 |

### 规则数量对比

```
iptables 模式 (1000 Services, 每个 3 Endpoints):
- KUBE-SERVICES 链: ~1000 规则
- KUBE-SVC-* 链: ~3000 规则
- KUBE-SEP-* 链: ~3000 规则
- 总计: ~7000+ 规则

nftables 模式 (同等规模):
- 集合: 1 (包含 1000 元素)
- 映射: 1 (包含 1000 元素)
- 链: ~4000 (服务链 + 端点链)
- 总规则数减少，查找效率提升
```

### 基准测试

```bash
# 测试规则同步时间
time kube-proxy --proxy-mode=nftables --sync-period=0

# 测试连接建立延迟
wrk -t4 -c100 -d30s http://service:80

# 监控 CPU 使用
top -p $(pidof kube-proxy)
```

## 调试与监控

### 查看规则

```bash
# 查看所有规则
nft list ruleset

# 查看 kube-proxy 表
nft list table ip kube-proxy

# 查看特定链
nft list chain ip kube-proxy services

# 查看集合
nft list set ip kube-proxy cluster-ips

# 查看映射
nft list map ip kube-proxy service-endpoints
```

### 监控指标

```yaml
# Prometheus 指标
- kubeproxy_sync_proxy_rules_duration_seconds  # 同步耗时
- kubeproxy_sync_proxy_rules_last_timestamp    # 最后同步时间
- kubeproxy_nftables_sync_total               # nftables 同步次数
- kubeproxy_nftables_sync_errors_total        # 同步错误次数
```

### 常见问题

```bash
# 1. 检查 nftables 是否可用
nft --version

# 2. 检查内核模块
lsmod | grep nf_tables

# 3. 检查 kube-proxy 模式
kubectl get cm -n kube-system kube-proxy -o yaml | grep mode

# 4. 查看 kube-proxy 日志
kubectl logs -n kube-system -l k8s-app=kube-proxy | grep nftables

# 5. 手动测试规则
nft add table ip test
nft add chain ip test forward { type filter hook forward priority 0 \; }
nft delete table ip test
```

## 迁移指南

### 从 iptables 迁移

```yaml
# 1. 更新 kube-proxy 配置
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: nftables

# 2. 滚动更新 kube-proxy
kubectl rollout restart ds/kube-proxy -n kube-system

# 3. 验证
kubectl get pods -n kube-system -l k8s-app=kube-proxy -o wide
```

### 回退方案

```bash
# 如果 nftables 有问题，回退到 iptables
kubectl edit cm kube-proxy -n kube-system
# 将 mode 改为 iptables

kubectl rollout restart ds/kube-proxy -n kube-system
```

## 最佳实践

### 配置建议

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: nftables
nftables:
  syncPeriod: 30s
  minSyncPeriod: 1s
conntrack:
  maxPerCore: 32768
  min: 131072
```

### 监控建议

```yaml
# 监控 nftables 规则数
- record: kube_proxy:nftables_rules_total
  expr: count(nft_chain_rules)

# 监控同步延迟
- alert: KubeProxyNftablesSyncSlow
  expr: kubeproxy_sync_proxy_rules_duration_seconds > 5
  for: 5m
  labels:
    severity: warning
  annotations:
    summary: "kube-proxy nftables sync is slow"
```

## 总结

kube-proxy nftables 模式的优势：
- **现代化框架**：统一的 nftables 框架
- **更好的性能**：集合和映射支持 O(1) 查找
- **原子更新**：原生事务支持
- **简洁语法**：更易于理解和调试

适用场景：
- Linux 5.13+ 内核环境
- 大规模集群（1000+ Services）
- 对规则更新延迟敏感的场景
- 需要更好调试能力的环境

注意事项：
- 需要较新的内核版本
- 目前为 Beta 功能
- 某些 CNI 插件可能还不完全兼容
