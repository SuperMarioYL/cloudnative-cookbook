---
title: "多集群概念"
weight: 1
---
## 概述

随着 Kubernetes 的广泛采用，单集群架构已无法满足企业级应用的需求。多集群架构通过将工作负载分布在多个集群中，提供更高的可用性、更好的隔离性和更灵活的资源利用。本文深入解析多集群的核心概念、架构模式和技术挑战。

## 多集群动机

### 为什么需要多集群

```
┌─────────────────────────────────────────────────────────────────┐
│                      多集群动机分析                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    高可用性 (HA)                         │   │
│   │                                                          │   │
│   │   • 避免单点故障 - 集群级故障隔离                        │   │
│   │   • 跨区域容灾 - 地理冗余                               │   │
│   │   • 升级隔离 - 滚动升级不影响所有用户                   │   │
│   │   • 灾难恢复 - 快速切换到备用集群                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    地理分布 (Locality)                   │   │
│   │                                                          │   │
│   │   • 低延迟 - 靠近用户部署                               │   │
│   │   • 数据主权 - 满足数据本地化要求                       │   │
│   │   • 边缘计算 - 边缘节点本地处理                         │   │
│   │   • 多云部署 - 避免云厂商锁定                           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    隔离需求 (Isolation)                  │   │
│   │                                                          │   │
│   │   • 环境隔离 - 开发/测试/生产分离                       │   │
│   │   • 租户隔离 - 不同客户独立集群                         │   │
│   │   • 安全边界 - 敏感工作负载隔离                         │   │
│   │   • 合规要求 - 满足行业监管                             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    规模扩展 (Scale)                      │   │
│   │                                                          │   │
│   │   • 突破单集群限制 - 节点数、Pod 数上限                 │   │
│   │   • 资源池化 - 统一管理多个集群资源                     │   │
│   │   • 弹性扩展 - 按需扩展到新集群                         │   │
│   │   • 成本优化 - 利用不同云的价格优势                     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 单集群 vs 多集群

| 维度 | 单集群 | 多集群 |
|------|--------|--------|
| **故障域** | 单点故障影响全部 | 故障隔离到单个集群 |
| **扩展性** | 受限于集群上限 | 水平扩展无上限 |
| **延迟** | 单区域 | 多区域就近访问 |
| **运维复杂度** | 较低 | 较高 |
| **一致性** | 强一致 | 最终一致 |
| **成本** | 相对较低 | 管理开销较高 |

## 架构模式

### 独立集群模式

```
┌─────────────────────────────────────────────────────────────────┐
│                      独立集群模式                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│   │   Cluster A     │  │   Cluster B     │  │   Cluster C     │ │
│   │   (开发环境)     │  │   (测试环境)     │  │   (生产环境)     │ │
│   │                 │  │                 │  │                 │ │
│   │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │ │
│   │  │  Control  │  │  │  │  Control  │  │  │  │  Control  │  │ │
│   │  │  Plane    │  │  │  │  Plane    │  │  │  │  Plane    │  │ │
│   │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │ │
│   │                 │  │                 │  │                 │ │
│   │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │ │
│   │  │  Workers  │  │  │  │  Workers  │  │  │  │  Workers  │  │ │
│   │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │ │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│            │                   │                   │            │
│            └───────────────────┼───────────────────┘            │
│                                │                                │
│                    ┌───────────────────────┐                    │
│                    │    统一运维平台        │                    │
│                    │  (Rancher/Lens/...)   │                    │
│                    └───────────────────────┘                    │
│                                                                  │
│   特点:                                                          │
│   • 集群完全独立，无直接通信                                     │
│   • 通过外部工具统一管理                                         │
│   • 适合环境隔离、租户隔离场景                                   │
│   • 运维简单，无复杂依赖                                         │
└─────────────────────────────────────────────────────────────────┘
```

### 联邦集群模式

```
┌─────────────────────────────────────────────────────────────────┐
│                      联邦集群模式                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌───────────────────────┐                    │
│                    │    Federation         │                    │
│                    │    Control Plane      │                    │
│                    │                       │                    │
│                    │  ┌─────────────────┐  │                    │
│                    │  │ Federation API  │  │                    │
│                    │  │    Server       │  │                    │
│                    │  └─────────────────┘  │                    │
│                    │  ┌─────────────────┐  │                    │
│                    │  │ Federation      │  │                    │
│                    │  │ Controllers     │  │                    │
│                    │  └─────────────────┘  │                    │
│                    └───────────┬───────────┘                    │
│                                │                                │
│            ┌───────────────────┼───────────────────┐            │
│            │                   │                   │            │
│            ▼                   ▼                   ▼            │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│   │  Member Cluster │  │  Member Cluster │  │  Member Cluster │ │
│   │       A         │  │       B         │  │       C         │ │
│   │                 │  │                 │  │                 │ │
│   │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │ │
│   │  │ Workloads │  │  │  │ Workloads │  │  │  │ Workloads │  │ │
│   │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │ │
│   └─────────────────┘  └─────────────────┘  └─────────────────┘ │
│                                                                  │
│   特点:                                                          │
│   • 中央控制平面管理多个成员集群                                 │
│   • 统一 API 分发资源到成员集群                                  │
│   • 支持跨集群调度和服务发现                                     │
│   • Karmada/Clusternet 等实现                                   │
└─────────────────────────────────────────────────────────────────┘
```

### Hub-Spoke 模式

```
┌─────────────────────────────────────────────────────────────────┐
│                      Hub-Spoke 模式                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                    ┌───────────────────────┐                    │
│                    │      Hub Cluster      │                    │
│                    │                       │                    │
│                    │  ┌─────────────────┐  │                    │
│                    │  │   Cluster       │  │                    │
│                    │  │   Registry      │  │                    │
│                    │  └─────────────────┘  │                    │
│                    │  ┌─────────────────┐  │                    │
│                    │  │   Policy        │  │                    │
│                    │  │   Engine        │  │                    │
│                    │  └─────────────────┘  │                    │
│                    │  ┌─────────────────┐  │                    │
│                    │  │   Work          │  │                    │
│                    │  │   Distribution  │  │                    │
│                    │  └─────────────────┘  │                    │
│                    └───────────┬───────────┘                    │
│                                │                                │
│         ┌──────────────────────┼──────────────────────┐         │
│         │                      │                      │         │
│         ▼                      ▼                      ▼         │
│  ┌────────────┐         ┌────────────┐         ┌────────────┐  │
│  │   Spoke    │         │   Spoke    │         │   Spoke    │  │
│  │ Cluster 1  │         │ Cluster 2  │         │ Cluster N  │  │
│  │            │         │            │         │            │  │
│  │ ┌────────┐ │         │ ┌────────┐ │         │ ┌────────┐ │  │
│  │ │ Agent  │ │         │ │ Agent  │ │         │ │ Agent  │ │  │
│  │ └────────┘ │         │ └────────┘ │         │ └────────┘ │  │
│  └────────────┘         └────────────┘         └────────────┘  │
│                                                                  │
│   特点:                                                          │
│   • Hub 集群作为管理中心                                        │
│   • Spoke 集群运行 Agent 上报状态                               │
│   • Pull 模式，Spoke 主动拉取配置                               │
│   • OCM (Open Cluster Management) 采用此模式                    │
└─────────────────────────────────────────────────────────────────┘
```

### 服务网格多集群

```
┌─────────────────────────────────────────────────────────────────┐
│                   服务网格多集群模式                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                  Istio Control Plane                     │   │
│   │   ┌───────────┐  ┌───────────┐  ┌───────────────────┐   │   │
│   │   │  Istiod   │  │  Istiod   │  │  Shared Root CA   │   │   │
│   │   │(Primary)  │  │(Primary)  │  │                   │   │   │
│   │   └───────────┘  └───────────┘  └───────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌──────────────────────┐    ┌──────────────────────┐          │
│   │     Cluster A        │    │     Cluster B        │          │
│   │                      │    │                      │          │
│   │  ┌────────────────┐  │    │  ┌────────────────┐  │          │
│   │  │   Service A    │  │    │  │   Service B    │  │          │
│   │  │  ┌──────────┐  │  │    │  │  ┌──────────┐  │  │          │
│   │  │  │  Envoy   │◀─┼──┼────┼──┼─▶│  Envoy   │  │  │          │
│   │  │  │ (Sidecar)│  │  │    │  │  │ (Sidecar)│  │  │          │
│   │  │  └──────────┘  │  │    │  │  └──────────┘  │  │          │
│   │  └────────────────┘  │    │  └────────────────┘  │          │
│   │                      │    │                      │          │
│   │  ┌────────────────┐  │    │  ┌────────────────┐  │          │
│   │  │ East-West GW   │◀─┼────┼─▶│ East-West GW   │  │          │
│   │  └────────────────┘  │    │  └────────────────┘  │          │
│   └──────────────────────┘    └──────────────────────┘          │
│                                                                  │
│   特点:                                                          │
│   • 统一服务网格控制平面                                        │
│   • 跨集群服务发现和负载均衡                                    │
│   • mTLS 保证跨集群通信安全                                     │
│   • 透明的流量管理                                              │
└─────────────────────────────────────────────────────────────────┘
```

## 核心挑战

### 网络连通性

```yaml
# 跨集群网络方案对比
apiVersion: v1
kind: ConfigMap
metadata:
  name: multicluster-network-options
data:
  options: |
    ## 网络连通方案

    ### 1. VPN/隧道
    - WireGuard
    - IPsec
    - GRE 隧道
    优点: 简单、安全
    缺点: 性能开销、MTU 问题

    ### 2. 扁平网络
    - Cilium Cluster Mesh
    - Calico Federation
    - Submariner
    优点: 原生 Pod-to-Pod 通信
    缺点: IP 规划复杂、需要非重叠 CIDR

    ### 3. 服务网关
    - Istio Multi-cluster
    - Linkerd Multi-cluster
    - Consul Connect
    优点: 应用层负载均衡、安全
    缺点: 需要服务网格

    ### 4. DNS 联邦
    - CoreDNS 多集群
    - ExternalDNS
    优点: 简单、与网络解耦
    缺点: 仅服务发现，不解决网络问题
---
# Submariner 部署示例
apiVersion: submariner.io/v1alpha1
kind: Broker
metadata:
  name: submariner-broker
  namespace: submariner-k8s-broker
spec:
  globalnetEnabled: true
  globalnetCIDRRange: "242.0.0.0/8"
---
apiVersion: submariner.io/v1alpha1
kind: Submariner
metadata:
  name: submariner
  namespace: submariner-operator
spec:
  broker: k8s
  brokerK8sApiServer: "https://broker-api.example.com:6443"
  brokerK8sApiServerToken: "<token>"
  brokerK8sRemoteNamespace: submariner-k8s-broker
  brokerK8sCA: "<ca-data>"
  ceIPSecDebug: false
  ceIPSecPSK: "<psk>"
  clusterCIDR: "10.244.0.0/16"
  clusterID: "cluster-a"
  colorCodes: "blue"
  debug: false
  globalCIDR: "242.1.0.0/16"
  namespace: submariner-operator
  natEnabled: true
  repository: quay.io/submariner
  serviceCIDR: "10.96.0.0/12"
  serviceDiscoveryEnabled: true
  version: "0.16.0"
```

### 身份认证

```yaml
# 跨集群身份认证方案
apiVersion: v1
kind: ConfigMap
metadata:
  name: multicluster-auth-options
data:
  spiffe: |
    # SPIFFE (Secure Production Identity Framework For Everyone)
    # 统一工作负载身份

    ## Trust Domain 配置
    trust_domain: "example.org"

    ## SPIFFE ID 格式
    # spiffe://example.org/ns/<namespace>/sa/<service-account>

    ## 跨集群信任
    # 共享根 CA 或联邦信任

  service-account-token: |
    # ServiceAccount Token 跨集群使用

    ## TokenRequest API
    kubectl create token <sa-name> \
      --audience=https://cluster-b.example.com \
      --duration=3600s

    ## Bound Token 投影
    volumes:
    - name: token
      projected:
        sources:
        - serviceAccountToken:
            audience: https://cluster-b.example.com
            expirationSeconds: 3600
            path: token

  oidc: |
    # OIDC 联邦身份

    ## 配置 API Server 信任外部 OIDC
    --oidc-issuer-url=https://oidc.example.com
    --oidc-client-id=kubernetes
    --oidc-username-claim=email
    --oidc-groups-claim=groups
---
# SPIFFE/SPIRE 配置示例
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterSPIFFEID
metadata:
  name: default
spec:
  className: spire-spiffe-id
  spiffeIDTemplate: "spiffe://{{ .TrustDomain }}/ns/{{ .PodMeta.Namespace }}/sa/{{ .PodSpec.ServiceAccountName }}"
  podSelector:
    matchLabels: {}
  namespaceSelector:
    matchLabels: {}
---
# 跨集群信任配置
apiVersion: spire.spiffe.io/v1alpha1
kind: ClusterFederatedTrustDomain
metadata:
  name: cluster-b
spec:
  trustDomain: cluster-b.example.org
  bundleEndpointURL: https://spire-server.cluster-b.example.com:8443
  bundleEndpointProfile:
    type: https_spiffe
    endpointSPIFFEID: spiffe://cluster-b.example.org/spire/server
```

### 资源调度

```yaml
# 跨集群调度策略
apiVersion: v1
kind: ConfigMap
metadata:
  name: multicluster-scheduling-options
data:
  strategies: |
    ## 调度策略类型

    ### 1. 静态分配
    - 固定资源到特定集群
    - 简单但不灵活

    ### 2. 权重分配
    - 按比例分配到多个集群
    - 支持灰度发布

    ### 3. 动态调度
    - 基于集群资源使用率
    - 自动负载均衡

    ### 4. 地理感知
    - 基于用户位置就近调度
    - 满足数据主权要求

    ### 5. 成本优化
    - 考虑不同云的价格
    - Spot 实例利用

  considerations: |
    ## 调度考量因素

    - 集群可用资源
    - 网络延迟
    - 数据亲和性
    - 合规要求
    - 成本因素
    - 故障域分布
---
# Karmada 调度策略示例
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
        - cluster-a
        - cluster-b
        - cluster-c
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Weighted
      weightPreference:
        staticWeightList:
          - targetCluster:
              clusterNames:
                - cluster-a
            weight: 2
          - targetCluster:
              clusterNames:
                - cluster-b
            weight: 1
          - targetCluster:
              clusterNames:
                - cluster-c
            weight: 1
```

### 状态同步

```yaml
# 跨集群状态同步方案
apiVersion: v1
kind: ConfigMap
metadata:
  name: multicluster-state-sync
data:
  patterns: |
    ## 状态同步模式

    ### 1. Push 模式
    - 中央控制器推送到成员集群
    - 实时性好
    - 需要网络连通

    ### 2. Pull 模式
    - 成员集群主动拉取
    - 网络友好（防火墙穿越）
    - 可能有延迟

    ### 3. 事件驱动
    - 基于消息队列
    - 解耦、可靠
    - 复杂度较高

    ### 4. GitOps
    - Git 作为单一事实来源
    - 声明式、可审计
    - 适合配置同步

  conflict-resolution: |
    ## 冲突解决策略

    ### 最后写入优先 (Last Writer Wins)
    - 简单但可能丢失数据

    ### 向量时钟 (Vector Clocks)
    - 检测冲突
    - 需要手动解决

    ### 操作转换 (OT)
    - 自动合并
    - 复杂度高

    ### CRDT
    - 无冲突复制数据类型
    - 最终一致性
---
# 使用 Liqo 的资源同步示例
apiVersion: discovery.liqo.io/v1alpha1
kind: ForeignCluster
metadata:
  name: cluster-b
spec:
  clusterIdentity:
    clusterID: cluster-b-id
    clusterName: cluster-b
  outgoingPeeringEnabled: true
  incomingPeeringEnabled: true
  insecureSkipTLSVerify: false
---
# ResourceOffer 表示可共享的资源
apiVersion: sharing.liqo.io/v1alpha1
kind: ResourceOffer
metadata:
  name: cluster-b-offer
spec:
  clusterId: cluster-b-id
  resourceQuota:
    hard:
      cpu: "10"
      memory: "20Gi"
      pods: "100"
  labels:
    region: us-west
    environment: production
```

## 多集群解决方案对比

### 方案对比表

| 方案 | 模式 | 网络要求 | 调度能力 | 服务发现 | 复杂度 | 适用场景 |
|------|------|----------|----------|----------|--------|----------|
| **Karmada** | Push | 需要 | 强 | 支持 | 中 | 多集群应用分发 |
| **Clusternet** | Push | 需要 | 中 | 部分 | 中 | 边缘计算 |
| **OCM** | Pull | 不需要 | 中 | 部分 | 中 | 混合云管理 |
| **Liqo** | Peer | 隧道 | 弱 | 支持 | 低 | 资源共享 |
| **Submariner** | - | 隧道 | 无 | 支持 | 中 | 跨集群网络 |
| **Istio** | Mesh | 需要 | 无 | 强 | 高 | 服务网格 |
| **Cilium** | Mesh | 需要 | 无 | 强 | 中 | 网络层方案 |

### 选型建议

```
┌─────────────────────────────────────────────────────────────────┐
│                      多集群方案选型决策树                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                     需要跨集群应用分发?                          │
│                            │                                     │
│               ┌────────────┴────────────┐                       │
│               │                         │                        │
│              是                        否                        │
│               │                         │                        │
│               ▼                         ▼                        │
│      网络可直接连通?              需要服务网格?                  │
│               │                         │                        │
│       ┌───────┴───────┐         ┌───────┴───────┐               │
│       │               │         │               │                │
│      是              否        是              否                │
│       │               │         │               │                │
│       ▼               ▼         ▼               ▼                │
│   ┌───────┐     ┌─────────┐  ┌──────┐     ┌─────────┐           │
│   │Karmada│     │  OCM    │  │ Istio│     │独立集群 │           │
│   │       │     │(Pull)   │  │Multi │     │+ Rancher│           │
│   └───────┘     └─────────┘  │Cluster│    └─────────┘           │
│                              └──────┘                            │
│                                                                  │
│   额外考虑:                                                      │
│                                                                  │
│   • 边缘计算场景 ──▶ Clusternet / KubeEdge                      │
│   • 资源共享场景 ──▶ Liqo                                       │
│   • 仅需跨集群网络 ──▶ Submariner / Cilium Cluster Mesh         │
│   • 多租户 SaaS ──▶ 独立集群 + 统一管理平台                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 最佳实践

### 集群命名规范

```yaml
# 集群命名约定
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-naming-convention
data:
  format: |
    # 格式: <环境>-<区域>-<用途>-<编号>

    ## 环境
    - prod: 生产
    - stag: 预发布
    - dev: 开发
    - test: 测试

    ## 区域
    - usw1: US West 1
    - use1: US East 1
    - euc1: EU Central 1
    - aps1: AP South 1
    - cnn1: China North 1

    ## 用途
    - gen: 通用
    - edge: 边缘
    - ml: 机器学习
    - data: 数据处理

    ## 示例
    - prod-usw1-gen-01
    - prod-euc1-edge-01
    - dev-use1-gen-01

  labels: |
    # 标准标签
    cluster.example.com/name: prod-usw1-gen-01
    cluster.example.com/environment: production
    cluster.example.com/region: us-west-1
    cluster.example.com/zone: us-west-1a
    cluster.example.com/provider: aws
    cluster.example.com/purpose: general
    topology.kubernetes.io/region: us-west-1
    topology.kubernetes.io/zone: us-west-1a
```

### IP 规划

```yaml
# 多集群 CIDR 规划
apiVersion: v1
kind: ConfigMap
metadata:
  name: multicluster-cidr-planning
data:
  planning: |
    ## CIDR 规划原则

    1. Pod CIDR 不重叠（如需扁平网络）
    2. Service CIDR 可重叠（通过 DNS 发现）
    3. 预留足够扩展空间
    4. 便于识别和调试

    ## 示例规划

    ### Cluster A
    Pod CIDR:     10.0.0.0/16    (65,536 IPs)
    Service CIDR: 172.16.0.0/16
    Node CIDR:    192.168.0.0/24

    ### Cluster B
    Pod CIDR:     10.1.0.0/16    (65,536 IPs)
    Service CIDR: 172.16.0.0/16
    Node CIDR:    192.168.1.0/24

    ### Cluster C
    Pod CIDR:     10.2.0.0/16    (65,536 IPs)
    Service CIDR: 172.16.0.0/16
    Node CIDR:    192.168.2.0/24

    ### GlobalNet (Submariner)
    Global CIDR:  242.0.0.0/8
    - Cluster A:  242.1.0.0/16
    - Cluster B:  242.2.0.0/16
    - Cluster C:  242.3.0.0/16
```

### 监控与可观测

```yaml
# 多集群监控架构
apiVersion: v1
kind: ConfigMap
metadata:
  name: multicluster-observability
data:
  metrics: |
    ## 指标收集架构

    ### 联邦 Prometheus
    - 每个集群部署 Prometheus
    - 中央 Thanos 查询聚合
    - 对象存储持久化

    ### 关键指标
    - 集群健康状态
    - 跨集群延迟
    - 资源使用率
    - 同步状态

  logging: |
    ## 日志收集架构

    ### 每个集群
    - Fluentd/Fluent Bit 采集
    - 添加集群标签

    ### 中央存储
    - Elasticsearch/Loki
    - 按集群索引

  tracing: |
    ## 分布式追踪

    ### 方案
    - Jaeger/Zipkin
    - OpenTelemetry Collector

    ### 跨集群追踪
    - 统一 Trace ID 传递
    - Context Propagation
---
# Thanos 联邦查询配置
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus
  namespace: monitoring
spec:
  replicas: 2
  thanos:
    baseImage: quay.io/thanos/thanos
    version: v0.32.0
    objectStorageConfig:
      key: thanos.yaml
      name: thanos-objstore-config
  externalLabels:
    cluster: prod-usw1-gen-01
    region: us-west-1
---
# Thanos Query 配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: thanos-query
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: thanos-query
  template:
    spec:
      containers:
        - name: thanos-query
          image: quay.io/thanos/thanos:v0.32.0
          args:
            - query
            - --query.replica-label=prometheus_replica
            - --store=dnssrv+_grpc._tcp.thanos-sidecar.cluster-a.svc
            - --store=dnssrv+_grpc._tcp.thanos-sidecar.cluster-b.svc
            - --store=dnssrv+_grpc._tcp.thanos-sidecar.cluster-c.svc
            - --store=dnssrv+_grpc._tcp.thanos-store.monitoring.svc
```

## 总结

多集群架构的关键考量：

| 方面 | 要点 |
|------|------|
| **动机明确** | 确定是否真正需要多集群（HA、地理、隔离、规模） |
| **模式选择** | 根据需求选择适合的架构模式 |
| **网络规划** | 提前规划 CIDR，避免重叠 |
| **身份统一** | 跨集群身份认证和授权 |
| **运维体系** | 统一监控、日志、告警 |
| **渐进实施** | 从简单场景开始，逐步演进 |

多集群是解决特定问题的手段，不是目的。在引入多集群之前，应充分评估其带来的复杂性是否值得。
