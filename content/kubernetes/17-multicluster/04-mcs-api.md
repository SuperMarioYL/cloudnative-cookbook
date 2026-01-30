---
title: "Multi-Cluster Services API"
weight: 4
---
## 概述

Multi-Cluster Services (MCS) API 是 Kubernetes SIG Multicluster 定义的标准 API，用于实现跨集群的服务发现和负载均衡。它提供了一种声明式的方式来导出和导入服务，使得在多集群环境中实现服务互通变得标准化。本文深入解析 MCS API 的设计原理和实现方案。

## MCS API 设计

### 核心概念

```
┌─────────────────────────────────────────────────────────────────┐
│                      MCS API 概念模型                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    ClusterSet                            │   │
│   │            (一组协作的 Kubernetes 集群)                   │   │
│   │                                                          │   │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐            │   │
│   │  │ Cluster A │  │ Cluster B │  │ Cluster C │            │   │
│   │  │           │  │           │  │           │            │   │
│   │  │  ClusterID│  │  ClusterID│  │  ClusterID│            │   │
│   │  │  = "a"    │  │  = "b"    │  │  = "c"    │            │   │
│   │  └───────────┘  └───────────┘  └───────────┘            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                   服务导出/导入流程                       │   │
│   │                                                          │   │
│   │   Cluster A                        Cluster B             │   │
│   │   ┌──────────────────┐            ┌──────────────────┐  │   │
│   │   │    Service       │            │  ServiceImport   │  │   │
│   │   │    "nginx"       │            │    "nginx"       │  │   │
│   │   └────────┬─────────┘            └────────▲─────────┘  │   │
│   │            │                               │             │   │
│   │            ▼                               │             │   │
│   │   ┌──────────────────┐                     │             │   │
│   │   │  ServiceExport   │─────────────────────┘             │   │
│   │   │    "nginx"       │      (跨集群同步)                 │   │
│   │   └──────────────────┘                                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    DNS 命名规范                          │   │
│   │                                                          │   │
│   │   本集群服务:                                            │   │
│   │   <service>.<namespace>.svc.cluster.local                │   │
│   │                                                          │   │
│   │   跨集群服务 (ClusterSet):                               │   │
│   │   <service>.<namespace>.svc.clusterset.local             │   │
│   │                                                          │   │
│   │   指定集群服务:                                          │   │
│   │   <clusterid>.<service>.<namespace>.svc.clusterset.local │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### API 资源定义

```yaml
# ServiceExport - 导出服务到 ClusterSet
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: nginx
  namespace: default
# ServiceExport 没有 spec，仅作为标记
# 存在 ServiceExport 表示对应的 Service 应该被导出
status:
  conditions:
    - type: Valid
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:00Z"
      reason: ServiceValid
      message: "Service is valid for export"
    - type: Conflict
      status: "False"
      lastTransitionTime: "2024-01-15T10:00:00Z"
      reason: NoConflict
      message: "No conflict detected"
---
# ServiceImport - 导入 ClusterSet 中的服务
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: nginx
  namespace: default
spec:
  # 服务类型
  type: ClusterSetIP  # 或 Headless
  # 端口定义
  ports:
    - name: http
      port: 80
      protocol: TCP
    - name: https
      port: 443
      protocol: TCP
  # 会话亲和性
  sessionAffinity: None
  # sessionAffinityConfig:
  #   clientIP:
  #     timeoutSeconds: 10800
status:
  # 提供服务的集群列表
  clusters:
    - cluster: cluster-a
    - cluster: cluster-b
    - cluster: cluster-c
---
# EndpointSlice 注解 - 标识来源集群
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: nginx-cluster-a-abc123
  namespace: default
  labels:
    kubernetes.io/service-name: nginx
    multicluster.kubernetes.io/source-cluster: cluster-a
  annotations:
    multicluster.kubernetes.io/service-export-uid: "12345-abcde"
addressType: IPv4
ports:
  - name: http
    port: 80
    protocol: TCP
endpoints:
  - addresses:
      - "10.1.1.100"
      - "10.1.1.101"
    conditions:
      ready: true
    topology:
      kubernetes.io/hostname: node-1
```

## Submariner 实现

### 架构概述

```
┌─────────────────────────────────────────────────────────────────┐
│                    Submariner 架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Broker Cluster                        │   │
│   │                                                          │   │
│   │  ┌────────────────────────────────────────────────────┐  │   │
│   │  │              Broker API Server                      │  │   │
│   │  │   (存储 ClusterSet 元数据和 ServiceImport)          │  │   │
│   │  └────────────────────────────────────────────────────┘  │   │
│   └──────────────────────────┬──────────────────────────────┘   │
│                              │                                   │
│            ┌─────────────────┼─────────────────┐                │
│            │                 │                 │                 │
│            ▼                 ▼                 ▼                 │
│   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐  │
│   │    Cluster A    │ │    Cluster B    │ │    Cluster C    │  │
│   │                 │ │                 │ │                 │  │
│   │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │  │
│   │ │ submariner- │ │ │ │ submariner- │ │ │ │ submariner- │ │  │
│   │ │  gateway    │◀┼─┼▶│  gateway    │◀┼─┼▶│  gateway    │ │  │
│   │ │  (IPsec/    │ │ │ │  (IPsec/    │ │ │ │  (IPsec/    │ │  │
│   │ │  WireGuard) │ │ │ │  WireGuard) │ │ │ │  WireGuard) │ │  │
│   │ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │  │
│   │                 │ │                 │ │                 │  │
│   │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │  │
│   │ │ submariner- │ │ │ │ submariner- │ │ │ │ submariner- │ │  │
│   │ │   route     │ │ │ │   route     │ │ │ │   route     │ │  │
│   │ │   agent     │ │ │ │   agent     │ │ │ │   agent     │ │  │
│   │ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │  │
│   │                 │ │                 │ │                 │  │
│   │ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │  │
│   │ │ lighthouse- │ │ │ │ lighthouse- │ │ │ │ lighthouse- │ │  │
│   │ │   agent     │ │ │ │   agent     │ │ │ │   agent     │ │  │
│   │ │  (DNS 和    │ │ │ │  (DNS 和    │ │ │ │  (DNS 和    │ │  │
│   │ │   MCS 控制) │ │ │ │   MCS 控制) │ │ │ │   MCS 控制) │ │  │
│   │ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │  │
│   └─────────────────┘ └─────────────────┘ └─────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 部署 Submariner

```bash
# 安装 subctl CLI
curl -Ls https://get.submariner.io | VERSION=0.16.0 bash

# 部署 Broker
subctl deploy-broker --kubeconfig broker-cluster.kubeconfig

# 加入集群 A
subctl join --kubeconfig cluster-a.kubeconfig broker-info.subm \
  --clusterid cluster-a \
  --natt=false \
  --cable-driver wireguard

# 加入集群 B
subctl join --kubeconfig cluster-b.kubeconfig broker-info.subm \
  --clusterid cluster-b \
  --natt=false \
  --cable-driver wireguard

# 验证连接
subctl show connections
subctl show endpoints
subctl diagnose all
```

```yaml
# Submariner 配置
apiVersion: submariner.io/v1alpha1
kind: Submariner
metadata:
  name: submariner
  namespace: submariner-operator
spec:
  # 集群标识
  clusterID: cluster-a
  # 集群网络配置
  clusterCIDR: "10.244.0.0/16"
  serviceCIDR: "10.96.0.0/12"
  # GlobalNet (如果 CIDR 重叠)
  globalCIDR: "242.1.0.0/16"
  # 隧道驱动
  cableDriver: wireguard  # 或 libreswan, vxlan
  # Broker 配置
  broker: k8s
  brokerK8sApiServer: "https://broker.example.com:6443"
  brokerK8sRemoteNamespace: submariner-k8s-broker
  # NAT 穿透
  natEnabled: true
  # ServiceDiscovery
  serviceDiscoveryEnabled: true
  # Lighthouse (DNS)
  lightHouseEnabled: true
  # 调试
  debug: false
---
# Gateway 配置
apiVersion: submariner.io/v1alpha1
kind: Gateway
metadata:
  name: cluster-a-gateway
  namespace: submariner-operator
spec:
  # 网关节点选择
  nodeSelector:
    submariner.io/gateway: "true"
  # 公网 IP (NAT 场景)
  publicIP: "203.0.113.10"
  # 连接配置
  connections:
    - endpoint: cluster-b
      status: connected
      latency: 10ms
```

### 服务导出导入

```yaml
# 在 Cluster A 创建服务
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
spec:
  selector:
    app: nginx
  ports:
    - name: http
      port: 80
      targetPort: 8080
---
# 导出服务
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: nginx
  namespace: default
# 创建 ServiceExport 后，Lighthouse 会自动同步到 Broker
# 其他集群的 Lighthouse 会创建对应的 ServiceImport
---
# 在 Cluster B 查看导入的服务
# kubectl get serviceimport -n default
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: nginx
  namespace: default
spec:
  type: ClusterSetIP
  ports:
    - name: http
      port: 80
      protocol: TCP
status:
  clusters:
    - cluster: cluster-a
```

### DNS 解析

```yaml
# CoreDNS 配置 (Lighthouse 自动注入)
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

        # Lighthouse 插件处理 clusterset.local
        lighthouse clusterset.local {
            fallthrough
        }

        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }

        forward . /etc/resolv.conf

        cache 30
        loop
        reload
        loadbalance
    }
---
# DNS 查询示例
# 在 Cluster B 中访问 Cluster A 的服务

# ClusterSet 域名 (自动负载均衡到所有提供者)
nslookup nginx.default.svc.clusterset.local
# 返回: 242.1.0.100 (GlobalNet VIP)

# 指定集群
nslookup cluster-a.nginx.default.svc.clusterset.local
# 返回: 242.1.0.100 (Cluster A 的 GlobalNet IP)

# 本地服务 (如果本地也有同名服务)
nslookup nginx.default.svc.cluster.local
# 返回: 10.96.0.100 (本地 ClusterIP)
```

## Cilium Cluster Mesh

### 架构概述

```
┌─────────────────────────────────────────────────────────────────┐
│                  Cilium Cluster Mesh 架构                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              Cluster Mesh API Server                     │   │
│   │           (每个集群运行，etcd 存储服务信息)               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌──────────────────────┐    ┌──────────────────────┐         │
│   │      Cluster A       │    │      Cluster B       │         │
│   │                      │    │                      │         │
│   │  ┌────────────────┐  │    │  ┌────────────────┐  │         │
│   │  │ Cluster Mesh   │◀─┼────┼─▶│ Cluster Mesh   │  │         │
│   │  │  API Server    │  │    │  │  API Server    │  │         │
│   │  │   (etcd)       │  │    │  │   (etcd)       │  │         │
│   │  └────────────────┘  │    │  └────────────────┘  │         │
│   │         ▲            │    │         ▲            │         │
│   │         │            │    │         │            │         │
│   │  ┌──────┴───────┐    │    │  ┌──────┴───────┐    │         │
│   │  │    Cilium    │    │    │  │    Cilium    │    │         │
│   │  │    Agent     │    │    │  │    Agent     │    │         │
│   │  │   (每节点)   │    │    │  │   (每节点)   │    │         │
│   │  └──────────────┘    │    │  └──────────────┘    │         │
│   │         │            │    │         │            │         │
│   │         ▼            │    │         ▼            │         │
│   │  ┌──────────────┐    │    │  ┌──────────────┐    │         │
│   │  │   eBPF       │    │    │  │   eBPF       │    │         │
│   │  │  Datapath    │◀───┼────┼──│  Datapath    │    │         │
│   │  │              │    │    │  │              │    │         │
│   │  └──────────────┘    │    │  └──────────────┘    │         │
│   │                      │    │                      │         │
│   │  Pod CIDR:           │    │  Pod CIDR:           │         │
│   │  10.0.0.0/16         │    │  10.1.0.0/16         │         │
│   └──────────────────────┘    └──────────────────────┘         │
│                                                                  │
│   特点:                                                          │
│   • 原生 Pod-to-Pod 通信 (无隧道开销)                           │
│   • eBPF 数据平面，高性能                                       │
│   • 全局服务发现                                                │
│   • 统一网络策略                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 部署配置

```yaml
# Cilium 安装配置 (Helm values)
apiVersion: v1
kind: ConfigMap
metadata:
  name: cilium-config
data:
  values.yaml: |
    cluster:
      name: cluster-a
      id: 1  # 每个集群唯一 ID

    # Cluster Mesh 配置
    clustermesh:
      useAPIServer: true
      apiserver:
        replicas: 2
        service:
          type: LoadBalancer
        tls:
          auto:
            method: helm
            certValidityDuration: 1095  # 3 years

    # 启用 MCS API 支持
    mcs-api:
      enabled: true

    # IPAM 配置 (确保 CIDR 不重叠)
    ipam:
      mode: kubernetes
      operator:
        clusterPoolIPv4PodCIDR: "10.0.0.0/16"
        clusterPoolIPv4MaskSize: 24

    # 跨集群路由
    tunnel: disabled  # 使用原生路由
    autoDirectNodeRoutes: true
    ipv4NativeRoutingCIDR: "10.0.0.0/8"
---
# 安装 Cilium
helm install cilium cilium/cilium \
  --version 1.15.0 \
  --namespace kube-system \
  -f values.yaml

# 启用 Cluster Mesh
cilium clustermesh enable

# 连接集群
cilium clustermesh connect --destination-context cluster-b
```

### 服务导出

```yaml
# 使用 annotation 导出服务
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: default
  annotations:
    # 导出到 Cluster Mesh
    service.cilium.io/global: "true"
    # 共享模式: shared (负载均衡到所有集群) 或 remote (优先远程)
    service.cilium.io/shared: "true"
    # 亲和性: local (优先本地), remote (优先远程), none (无偏好)
    service.cilium.io/affinity: "local"
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 8080
---
# 使用 MCS API 导出
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: nginx
  namespace: default
---
# 查看全局服务状态
# cilium service list --clustermesh
```

### 网络策略

```yaml
# 跨集群网络策略
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-cross-cluster
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
            # 允许来自任何集群的 frontend
            io.cilium.k8s.policy.cluster: "*"
    - fromEndpoints:
        - matchLabels:
            app: frontend
            # 只允许来自 cluster-a 的 frontend
            io.cilium.k8s.policy.cluster: "cluster-a"
---
# 跨集群出口策略
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-egress-to-remote-db
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      app: backend
  egress:
    - toEndpoints:
        - matchLabels:
            app: database
            # 只允许访问 cluster-b 的数据库
            io.cilium.k8s.policy.cluster: "cluster-b"
      toPorts:
        - ports:
            - port: "5432"
              protocol: TCP
```

## Istio 多集群

### 主-从架构

```yaml
# Primary Cluster 配置
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-primary
spec:
  profile: default
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster-a
      network: network1
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: network1
        enabled: true
        k8s:
          env:
            - name: ISTIO_META_ROUTER_MODE
              value: "sni-dnat"
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: network1
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
---
# Remote Cluster 配置
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  name: istio-remote
spec:
  profile: remote
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster-b
      network: network1
      remotePilotAddress: istiod.istio-system.svc.cluster.local
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        enabled: true
        label:
          istio: eastwestgateway
          topology.istio.io/network: network1
```

### 跨集群流量管理

```yaml
# VirtualService - 跨集群路由
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews
  namespace: default
spec:
  hosts:
    - reviews
  http:
    - match:
        - headers:
            x-cluster:
              exact: cluster-b
      route:
        - destination:
            host: reviews
            subset: cluster-b
    - route:
        - destination:
            host: reviews
            subset: cluster-a
          weight: 80
        - destination:
            host: reviews
            subset: cluster-b
          weight: 20
---
# DestinationRule - 跨集群子集
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: reviews
  namespace: default
spec:
  host: reviews
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
  subsets:
    - name: cluster-a
      labels:
        topology.istio.io/cluster: cluster-a
    - name: cluster-b
      labels:
        topology.istio.io/cluster: cluster-b
---
# ServiceEntry - 导入外部集群服务
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: external-svc
  namespace: default
spec:
  hosts:
    - external.example.com
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: HTTPS
  resolution: DNS
  endpoints:
    - address: cluster-c-gateway.example.com
      ports:
        https: 15443
      labels:
        topology.istio.io/cluster: cluster-c
```

## 方案对比

### 功能对比表

| 功能 | Submariner | Cilium Mesh | Istio |
|------|------------|-------------|-------|
| **网络连通** | IPsec/WireGuard 隧道 | 原生路由/BGP | East-West Gateway |
| **服务发现** | Lighthouse DNS | etcd 同步 | Pilot 同步 |
| **MCS API** | 完整支持 | 支持 | 部分支持 |
| **负载均衡** | Round-Robin | eBPF | Envoy |
| **网络策略** | 不支持跨集群 | 支持 | 支持 |
| **流量管理** | 基础 | 基础 | 丰富 |
| **CIDR 重叠** | GlobalNet 支持 | 不支持 | 支持 |
| **复杂度** | 中等 | 中等 | 高 |
| **性能开销** | 隧道开销 | 低 | Envoy 开销 |

### 选型建议

```
┌─────────────────────────────────────────────────────────────────┐
│                     MCS 实现方案选型                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│                      需要什么能力?                               │
│                           │                                      │
│         ┌─────────────────┼─────────────────┐                   │
│         │                 │                 │                    │
│         ▼                 ▼                 ▼                    │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐               │
│   │仅跨集群  │     │跨集群网络│     │完整服务  │               │
│   │服务发现  │     │+ 服务发现│     │网格能力  │               │
│   └────┬─────┘     └────┬─────┘     └────┬─────┘               │
│        │                │                │                      │
│        ▼                ▼                ▼                      │
│   ┌──────────┐     ┌──────────┐     ┌──────────┐               │
│   │Submariner│     │Cilium    │     │ Istio    │               │
│   │Lighthouse│     │Cluster   │     │Multi-    │               │
│   │(DNS only)│     │Mesh      │     │cluster   │               │
│   └──────────┘     └──────────┘     └──────────┘               │
│                                                                  │
│   其他考虑因素:                                                  │
│                                                                  │
│   • CIDR 重叠 ──▶ Submariner (GlobalNet)                        │
│   • 高性能要求 ──▶ Cilium (eBPF)                                │
│   • 复杂流量管理 ──▶ Istio                                      │
│   • 简单部署 ──▶ Submariner                                     │
│   • 已使用 Cilium ──▶ Cilium Cluster Mesh                       │
│   • 已使用 Istio ──▶ Istio Multi-cluster                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 最佳实践

### DNS 配置建议

```yaml
# CoreDNS 多集群配置
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

        # 本地集群服务
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
        }

        # ClusterSet 服务 (MCS API)
        # 根据使用的实现配置
        # Submariner: lighthouse
        # Cilium: 自动处理
        lighthouse clusterset.local {
            fallthrough
        }

        # 跨集群服务别名
        rewrite name substring .global.svc. .svc.clusterset.local. answer auto

        forward . /etc/resolv.conf
        cache 30
        reload
    }
```

### 故障排查

```bash
# Submariner 诊断
subctl diagnose all
subctl diagnose connections
subctl diagnose service-discovery

# 验证服务导出
kubectl get serviceexports -A
kubectl get serviceimports -A

# 验证 DNS
kubectl run -it --rm debug --image=busybox -- nslookup nginx.default.svc.clusterset.local

# Cilium 诊断
cilium clustermesh status
cilium service list --clustermesh
cilium connectivity test --multi-cluster

# 网络连通性测试
kubectl exec -it test-pod -- curl nginx.default.svc.clusterset.local

# 查看 EndpointSlice
kubectl get endpointslices -l multicluster.kubernetes.io/source-cluster
```

## 总结

MCS API 核心价值：

| 价值 | 说明 |
|------|------|
| **标准化** | Kubernetes SIG 定义的标准 API |
| **简单性** | 声明式服务导出/导入 |
| **灵活性** | 支持多种实现方案 |
| **DNS 集成** | clusterset.local 域名自动解析 |
| **渐进采用** | 不影响现有服务 |

关键注意事项：
- CIDR 规划要提前，避免重叠
- 选择适合的实现方案
- 测试跨集群网络延迟
- 配置适当的故障转移策略
- 监控跨集群服务状态
