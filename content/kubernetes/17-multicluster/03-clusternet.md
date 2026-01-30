---
title: "Clusternet 多集群管理"
weight: 3
---
## 概述

Clusternet 是一个开源的多集群管理项目，采用 Hub-Agent 架构，特别适合边缘计算和混合云场景。它支持 Pull 模式，允许成员集群主动拉取配置，解决了防火墙和网络隔离问题。本文深入解析 Clusternet 的架构设计和核心功能。

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    Clusternet 架构                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Hub Cluster                           │   │
│   │                                                          │   │
│   │  ┌────────────────────────────────────────────────────┐  │   │
│   │  │            clusternet-hub                           │  │   │
│   │  │                                                     │  │   │
│   │  │  ┌─────────────┐  ┌─────────────┐                  │  │   │
│   │  │  │ Cluster     │  │ Manifest    │                  │  │   │
│   │  │  │ Registration│  │ Deployer    │                  │  │   │
│   │  │  └─────────────┘  └─────────────┘                  │  │   │
│   │  │  ┌─────────────┐  ┌─────────────┐                  │  │   │
│   │  │  │ Scheduling  │  │ Status      │                  │  │   │
│   │  │  │ Controller  │  │ Aggregation │                  │  │   │
│   │  │  └─────────────┘  └─────────────┘                  │  │   │
│   │  └────────────────────────────────────────────────────┘  │   │
│   │                          │                               │   │
│   │  ┌────────────────────────────────────────────────────┐  │   │
│   │  │          clusternet-scheduler                       │  │   │
│   │  │         (多集群调度决策)                             │  │   │
│   │  └────────────────────────────────────────────────────┘  │   │
│   │                          │                               │   │
│   │  ┌────────────────────────────────────────────────────┐  │   │
│   │  │        Kubernetes API Server                        │  │   │
│   │  │          (存储集群和资源状态)                        │  │   │
│   │  └────────────────────────────────────────────────────┘  │   │
│   └──────────────────────────┬──────────────────────────────┘   │
│                              │                                   │
│            ┌─────────────────┼─────────────────┐                │
│            │ WebSocket/gRPC  │                 │                 │
│            │  (Pull 模式)    │                 │                 │
│            ▼                 ▼                 ▼                 │
│   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐  │
│   │  Child Cluster  │ │  Child Cluster  │ │  Child Cluster  │  │
│   │       A         │ │       B         │ │       C         │  │
│   │                 │ │                 │ │                 │  │
│   │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │  │
│   │  │clusternet │  │ │  │clusternet │  │ │  │clusternet │  │  │
│   │  │  -agent   │  │ │  │  -agent   │  │ │  │  -agent   │  │  │
│   │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │  │
│   │                 │ │                 │ │                 │  │
│   │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │  │
│   │  │ Workloads │  │ │  │ Workloads │  │ │  │ Workloads │  │  │
│   │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │  │
│   └─────────────────┘ └─────────────────┘ └─────────────────┘  │
│                                                                  │
│   特点:                                                          │
│   • Pull 模式 - Agent 主动连接 Hub                              │
│   • 防火墙友好 - 只需出站连接                                   │
│   • 边缘友好 - 支持弱网络环境                                   │
│   • 轻量级 - 最小化资源占用                                     │
└─────────────────────────────────────────────────────────────────┘
```

### 组件职责

```yaml
# clusternet-hub 部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusternet-hub
  namespace: clusternet-system
spec:
  replicas: 3
  selector:
    matchLabels:
      app: clusternet-hub
  template:
    metadata:
      labels:
        app: clusternet-hub
    spec:
      serviceAccountName: clusternet-hub
      containers:
        - name: clusternet-hub
          image: ghcr.io/clusternet/clusternet-hub:v0.16.0
          command:
            - /clusternet-hub
            - --secure-port=6443
            - --feature-gates=SocketConnection=true,Deployer=true
            - --leader-elect=true
            - --leader-elect-resource-namespace=clusternet-system
            # 启用 WebSocket 接受 Agent 连接
            - --peer-token-file=/etc/clusternet/tokens/peer-token
            - --v=4
          ports:
            - containerPort: 6443
              name: https
            - containerPort: 8443
              name: websocket
          volumeMounts:
            - name: peer-token
              mountPath: /etc/clusternet/tokens
              readOnly: true
      volumes:
        - name: peer-token
          secret:
            secretName: clusternet-peer-token
---
# clusternet-scheduler 部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusternet-scheduler
  namespace: clusternet-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: clusternet-scheduler
  template:
    spec:
      containers:
        - name: clusternet-scheduler
          image: ghcr.io/clusternet/clusternet-scheduler:v0.16.0
          command:
            - /clusternet-scheduler
            - --leader-elect=true
            - --leader-elect-resource-namespace=clusternet-system
            - --feature-gates=Predictor=true
            - --v=4
---
# clusternet-agent 部署 (在子集群)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusternet-agent
  namespace: clusternet-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: clusternet-agent
  template:
    spec:
      serviceAccountName: clusternet-agent
      containers:
        - name: clusternet-agent
          image: ghcr.io/clusternet/clusternet-agent:v0.16.0
          command:
            - /clusternet-agent
            # Hub 地址
            - --parent-url=https://hub.example.com:6443
            # 集群注册名称
            - --cluster-reg-name=child-cluster-1
            # 集群注册命名空间
            - --cluster-reg-namespace=clusternet-system
            # 使用 WebSocket 连接
            - --feature-gates=SocketConnection=true
            # 集群标签
            - --cluster-labels=region=us-west,environment=production
            - --v=4
          env:
            - name: PARENT_CLUSTER_TOKEN
              valueFrom:
                secretKeyRef:
                  name: clusternet-agent-token
                  key: token
```

### 集群注册流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   Clusternet 集群注册流程                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────┐                                            │
│  │ Child Cluster   │                                            │
│  │ (clusternet-    │                                            │
│  │  agent)         │                                            │
│  └────────┬────────┘                                            │
│           │ 1. 启动并连接 Hub                                   │
│           │    (WebSocket/gRPC)                                 │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    Hub Cluster                           │    │
│  │                                                          │    │
│  │  ┌────────────────────────────────────────────────────┐  │    │
│  │  │          Connection Handler                         │  │    │
│  │  │  • 验证 Agent Token                                 │  │    │
│  │  │  • 建立双向通道                                     │  │    │
│  │  └──────────────────────┬─────────────────────────────┘  │    │
│  │                         │ 2. 验证通过                     │    │
│  │                         ▼                                 │    │
│  │  ┌────────────────────────────────────────────────────┐  │    │
│  │  │        Cluster Registration Controller              │  │    │
│  │  │                                                     │  │    │
│  │  │  • 创建 ClusterRegistrationRequest                  │  │    │
│  │  │  • 自动批准或人工审批                               │  │    │
│  │  │  • 创建 ManagedCluster                              │  │    │
│  │  └──────────────────────┬─────────────────────────────┘  │    │
│  │                         │ 3. 创建集群资源                 │    │
│  │                         ▼                                 │    │
│  │  ┌────────────────────────────────────────────────────┐  │    │
│  │  │               资源创建                               │  │    │
│  │  │                                                     │  │    │
│  │  │  • ClusterRegistrationRequest (审批请求)            │  │    │
│  │  │  • ManagedCluster (集群元数据)                      │  │    │
│  │  │  • Namespace (clusternet-xxxxx)                     │  │    │
│  │  │  • Secret (集群访问凭证)                            │  │    │
│  │  └────────────────────────────────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
│           │ 4. 定期心跳和状态上报                               │
│           ▼                                                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                  状态同步                                │    │
│  │                                                          │    │
│  │  • Agent 上报节点信息、资源使用率                       │    │
│  │  • Hub 更新 ManagedCluster.status                       │    │
│  │  • Agent 拉取待部署的资源                               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 核心资源

### ClusterRegistrationRequest

```yaml
# 集群注册请求 (由 Agent 自动创建)
apiVersion: clusters.clusternet.io/v1beta1
kind: ClusterRegistrationRequest
metadata:
  name: clusternet-xxxx
  labels:
    clusters.clusternet.io/cluster-id: "12345-abcde"
spec:
  # 集群 ID
  clusterId: "12345-abcde"
  # 集群名称
  clusterName: child-cluster-1
  # 集群类型
  clusterType: EdgeCluster
  # 集群标签
  clusterLabels:
    region: us-west
    environment: production
    provider: aws
  # 同步模式
  syncMode: Push  # 或 Pull, Dual
status:
  # 审批状态
  result: Approved
  # 生成的 ManagedCluster 名称
  managedClusterName: clusternet-xxxx
  # 分配的命名空间
  dedicatedNamespace: clusternet-xxxx
---
# 手动批准注册请求
kubectl patch clusterregistrationrequest clusternet-xxxx \
  --type=merge \
  -p '{"status":{"result":"Approved"}}'
```

### ManagedCluster

```yaml
# 托管集群 (由 Hub 自动创建)
apiVersion: clusters.clusternet.io/v1beta1
kind: ManagedCluster
metadata:
  name: clusternet-v7wbf
  namespace: clusternet-system
  labels:
    clusters.clusternet.io/cluster-id: "12345-abcde"
    clusters.clusternet.io/cluster-name: child-cluster-1
    region: us-west
    environment: production
spec:
  # 集群 ID
  clusterId: "12345-abcde"
  # 同步模式
  syncMode: Push
  # 污点
  taints:
    - key: node.clusternet.io/unschedulable
      effect: NoSchedule
status:
  # 心跳时间
  lastObservedTime: "2024-01-15T10:30:00Z"
  # 集群健康状态
  healthz: true
  livez: true
  readyz: true
  # Kubernetes 版本
  k8sVersion: v1.28.0
  # 平台信息
  platform: linux/amd64
  # API Server 地址
  apiServerURL: https://10.0.0.100:6443
  # 节点统计
  nodeStatistics:
    readyNodes: 10
    notReadyNodes: 0
    unknownNodes: 0
    lostNodes: 0
  # 资源统计
  allocatable:
    cpu: "80"
    memory: "320Gi"
    pods: "1100"
  capacity:
    cpu: "100"
    memory: "400Gi"
    pods: "1100"
  # 集群 CIDR
  clusterCIDR: "10.244.0.0/16"
  serviceCIDR: "10.96.0.0/12"
  # 条件
  conditions:
    - type: Ready
      status: "True"
      lastTransitionTime: "2024-01-15T10:00:00Z"
      reason: ManagedClusterReady
      message: "Cluster is ready"
```

### Subscription

```yaml
# 订阅 - 定义分发什么资源
apiVersion: apps.clusternet.io/v1alpha1
kind: Subscription
metadata:
  name: nginx-subscription
  namespace: default
spec:
  # 订阅者 - 目标集群
  subscribers:
    # 按集群名称
    - clusterAffinity:
        matchLabels:
          clusters.clusternet.io/cluster-name: child-cluster-1
    # 按集群标签
    - clusterAffinity:
        matchExpressions:
          - key: region
            operator: In
            values:
              - us-west
              - us-east
          - key: environment
            operator: In
            values:
              - production

  # 调度策略
  schedulingStrategy: Replication  # 或 Dividing

  # 分发配置
  feeds:
    # 从 Helm Chart
    - apiVersion: apps.clusternet.io/v1alpha1
      kind: HelmChart
      name: nginx-chart
      namespace: default
    # 从原生资源
    - apiVersion: v1
      kind: ConfigMap
      name: nginx-config
      namespace: default
    # 从 Manifest (描述)
    - apiVersion: apps.clusternet.io/v1alpha1
      kind: Description
      name: nginx-desc
      namespace: default
---
# 使用 Dividing 策略的订阅
apiVersion: apps.clusternet.io/v1alpha1
kind: Subscription
metadata:
  name: nginx-divided
  namespace: default
spec:
  subscribers:
    - clusterAffinity:
        matchLabels:
          environment: production
      weight: 2  # 权重
    - clusterAffinity:
        matchLabels:
          environment: staging
      weight: 1

  schedulingStrategy: Dividing
  dividingScheduling:
    type: Static  # 或 Dynamic
    dynamicDividing:
      strategy: Spread  # 或 Binpack

  feeds:
    - apiVersion: apps.clusternet.io/v1alpha1
      kind: Description
      name: nginx-desc
```

### Feed 类型

```yaml
# HelmChart - Helm 方式部署
apiVersion: apps.clusternet.io/v1alpha1
kind: HelmChart
metadata:
  name: nginx-chart
  namespace: default
spec:
  # Helm 仓库
  repository: https://charts.bitnami.com/bitnami
  # Chart 名称
  chart: nginx
  # Chart 版本
  chartVersion: 15.0.0
  # Release 名称
  targetNamespace: nginx-system
  # 自定义 Values
  values: |
    replicaCount: 3
    service:
      type: ClusterIP
    resources:
      limits:
        cpu: 500m
        memory: 512Mi
---
# Description - 原生资源描述
apiVersion: apps.clusternet.io/v1alpha1
kind: Description
metadata:
  name: nginx-desc
  namespace: default
spec:
  deployer: Deployer  # 或 Helm
  charts: []  # Helm 模式时使用
  raw:
    # 原生 Kubernetes 资源
    - apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: nginx
        namespace: default
      spec:
        replicas: 3
        selector:
          matchLabels:
            app: nginx
        template:
          metadata:
            labels:
              app: nginx
          spec:
            containers:
              - name: nginx
                image: nginx:1.25
                ports:
                  - containerPort: 80
    - apiVersion: v1
      kind: Service
      metadata:
        name: nginx-svc
        namespace: default
      spec:
        selector:
          app: nginx
        ports:
          - port: 80
            targetPort: 80
```

### Localization 和 Globalization

```yaml
# Localization - 集群级别定制
apiVersion: apps.clusternet.io/v1alpha1
kind: Localization
metadata:
  name: nginx-localization-us-west
  namespace: clusternet-xxxx  # 特定集群的命名空间
spec:
  # 优先级 (数字越大优先级越高)
  priority: 100
  # 目标资源
  feed:
    apiVersion: apps.clusternet.io/v1alpha1
    kind: Description
    name: nginx-desc
    namespace: default
  # 覆盖规则
  overrides:
    # JSON Patch
    - name: replicas-override
      type: JSONPatch
      value: |
        - op: replace
          path: /spec/replicas
          value: 5
    # Merge Patch
    - name: image-override
      type: MergePatch
      value: |
        spec:
          template:
            spec:
              containers:
              - name: nginx
                image: us-west-registry.example.com/nginx:1.25
                resources:
                  limits:
                    cpu: "1"
                    memory: "1Gi"
---
# Globalization - 全局覆盖
apiVersion: apps.clusternet.io/v1alpha1
kind: Globalization
metadata:
  name: add-monitoring-labels
  namespace: default
spec:
  # 优先级
  priority: 50
  # 目标资源
  feed:
    apiVersion: apps.clusternet.io/v1alpha1
    kind: Description
    name: nginx-desc
  # 集群选择器
  clusterAffinity:
    matchLabels:
      environment: production
  # 覆盖规则
  overrides:
    - name: add-labels
      type: MergePatch
      value: |
        metadata:
          labels:
            monitoring.example.com/enabled: "true"
            team: platform
```

## 调度策略

### Replication 策略

```yaml
# 复制策略 - 每个集群部署完整副本
apiVersion: apps.clusternet.io/v1alpha1
kind: Subscription
metadata:
  name: nginx-replicated
spec:
  subscribers:
    - clusterAffinity:
        matchLabels:
          environment: production

  schedulingStrategy: Replication
  # 每个匹配的集群都会收到完整的资源副本

  feeds:
    - apiVersion: apps.clusternet.io/v1alpha1
      kind: Description
      name: nginx-desc

# 资源在每个集群的状态
# Cluster A: Deployment nginx (replicas: 3)
# Cluster B: Deployment nginx (replicas: 3)
# Cluster C: Deployment nginx (replicas: 3)
```

### Dividing 策略

```yaml
# 分割策略 - 副本分散到多个集群
apiVersion: apps.clusternet.io/v1alpha1
kind: Subscription
metadata:
  name: nginx-divided
spec:
  subscribers:
    - clusterAffinity:
        matchLabels:
          clusters.clusternet.io/cluster-name: cluster-a
      weight: 3
    - clusterAffinity:
        matchLabels:
          clusters.clusternet.io/cluster-name: cluster-b
      weight: 2
    - clusterAffinity:
        matchLabels:
          clusters.clusternet.io/cluster-name: cluster-c
      weight: 1

  schedulingStrategy: Dividing
  dividingScheduling:
    type: Static
    # 假设 Deployment replicas=6
    # cluster-a: 3 副本 (6 * 3/6)
    # cluster-b: 2 副本 (6 * 2/6)
    # cluster-c: 1 副本 (6 * 1/6)

  feeds:
    - apiVersion: apps.clusternet.io/v1alpha1
      kind: Description
      name: nginx-desc
---
# 动态分割
apiVersion: apps.clusternet.io/v1alpha1
kind: Subscription
metadata:
  name: nginx-dynamic-divided
spec:
  subscribers:
    - clusterAffinity:
        matchLabels:
          environment: production

  schedulingStrategy: Dividing
  dividingScheduling:
    type: Dynamic
    dynamicDividing:
      # Spread - 尽量分散
      # Binpack - 尽量聚合
      strategy: Spread
      # 基于集群可用资源动态计算
      minClusters: 2
      maxClusters: 5
      # 每个集群最小副本
      minReplicas: 1
      preferredClusters:
        - clusterAffinity:
            matchLabels:
              tier: premium
          weight: 2

  feeds:
    - apiVersion: apps.clusternet.io/v1alpha1
      kind: Description
      name: nginx-desc
```

### 基于 Predictor 的调度

```yaml
# 启用 Predictor 进行智能调度
apiVersion: apps.clusternet.io/v1alpha1
kind: Subscription
metadata:
  name: nginx-predicted
spec:
  subscribers:
    - clusterAffinity:
        matchLabels:
          environment: production

  schedulingStrategy: Dividing
  dividingScheduling:
    type: Dynamic
    dynamicDividing:
      strategy: Spread
      # 使用 Predictor 预测集群容量
      preferredClusters:
        - clusterAffinity:
            matchExpressions:
              - key: clusternet.io/predictor
                operator: Exists

  feeds:
    - apiVersion: apps.clusternet.io/v1alpha1
      kind: Description
      name: nginx-desc
---
# Predictor 配置 (在子集群部署)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusternet-predictor
  namespace: clusternet-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: clusternet-predictor
  template:
    spec:
      containers:
        - name: predictor
          image: ghcr.io/clusternet/clusternet-predictor:v0.16.0
          command:
            - /clusternet-predictor
            - --secure-port=8443
            - --v=4
```

## 状态聚合

### Base 资源状态

```yaml
# Base - 聚合的资源状态
apiVersion: apps.clusternet.io/v1alpha1
kind: Base
metadata:
  name: nginx-desc
  namespace: default
  # 由 Subscription 创建
  ownerReferences:
    - apiVersion: apps.clusternet.io/v1alpha1
      kind: Subscription
      name: nginx-subscription
spec:
  feeds:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
      namespace: default
status:
  # 聚合状态
  aggregatedStatus:
    # 按集群统计
    - clusterName: cluster-a
      feedStatusDetails:
        - apiVersion: apps/v1
          kind: Deployment
          name: nginx
          namespace: default
          exists: true
          observedStatus:
            replicas: 3
            readyReplicas: 3
            availableReplicas: 3
    - clusterName: cluster-b
      feedStatusDetails:
        - apiVersion: apps/v1
          kind: Deployment
          name: nginx
          namespace: default
          exists: true
          observedStatus:
            replicas: 2
            readyReplicas: 2
            availableReplicas: 2
  # 汇总
  conditions:
    - type: Scheduled
      status: "True"
    - type: Applied
      status: "True"
    - type: Healthy
      status: "True"
```

### Manifest 资源

```yaml
# Manifest - 分发到特定集群的资源
apiVersion: apps.clusternet.io/v1alpha1
kind: Manifest
metadata:
  name: nginx-deployment-cluster-a
  namespace: clusternet-xxxx  # 集群专属命名空间
  labels:
    apps.clusternet.io/feed: nginx-desc
spec:
  # 要部署的资源模板
  template:
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: nginx
      namespace: default
    spec:
      replicas: 3  # 已根据调度结果修改
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
            - name: nginx
              image: us-west-registry.example.com/nginx:1.25  # 已应用 Localization
status:
  # 部署状态
  observedStatus:
    replicas: 3
    readyReplicas: 3
    availableReplicas: 3
    conditions:
      - type: Available
        status: "True"
```

## 边缘场景支持

### 弱网络处理

```yaml
# Agent 配置 - 弱网络优化
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusternet-agent
  namespace: clusternet-system
spec:
  template:
    spec:
      containers:
        - name: clusternet-agent
          command:
            - /clusternet-agent
            - --parent-url=https://hub.example.com:6443
            # 心跳配置
            - --cluster-status-update-frequency=30s  # 降低更新频率
            - --cluster-lease-duration=60s
            # 重连配置
            - --reconnect-period=10s
            - --max-reconnect-period=5m
            # 缓存配置
            - --enable-cache=true
            - --cache-dir=/var/cache/clusternet
            # 压缩
            - --enable-compression=true
          volumeMounts:
            - name: cache
              mountPath: /var/cache/clusternet
      volumes:
        - name: cache
          emptyDir:
            sizeLimit: 1Gi
---
# Hub 配置 - 容忍断连
apiVersion: clusters.clusternet.io/v1beta1
kind: ManagedCluster
metadata:
  name: edge-cluster-1
spec:
  # 容忍较长的断连时间
  taints:
    - key: node.clusternet.io/unreachable
      effect: NoSchedule
      tolerationSeconds: 300  # 5 分钟容忍
```

### 离线部署

```yaml
# 预先缓存 Manifest
apiVersion: v1
kind: ConfigMap
metadata:
  name: clusternet-cache-config
  namespace: clusternet-system
data:
  cache-policy: |
    # 缓存策略
    cacheSize: 1Gi
    cacheDir: /var/cache/clusternet

    # 预取配置
    prefetch:
      enabled: true
      namespaces:
        - default
        - kube-system

    # 过期策略
    expiration:
      maxAge: 24h
      cleanupInterval: 1h
---
# Agent 使用本地缓存
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clusternet-agent
spec:
  template:
    spec:
      containers:
        - name: clusternet-agent
          command:
            - /clusternet-agent
            # 启用离线模式
            - --offline-cache-enabled=true
            - --offline-cache-dir=/var/cache/clusternet
            # 断网时使用缓存
            - --use-cache-when-disconnected=true
```

## 运维操作

### 常用命令

```bash
# 查看托管集群
kubectl get mcls -A
kubectl get managedclusters -A -o wide

# 查看集群注册请求
kubectl get clusterregistrationrequests

# 批准集群注册
kubectl patch clusterregistrationrequest <name> \
  --type=merge \
  -p '{"status":{"result":"Approved"}}'

# 查看订阅状态
kubectl get subscriptions -A
kubectl get sub nginx-subscription -o yaml

# 查看 Base 聚合状态
kubectl get base -A
kubectl get base nginx-desc -o yaml

# 查看分发到特定集群的 Manifest
kubectl get manifests -n clusternet-xxxx

# 查看 HelmChart
kubectl get helmcharts -A

# 查看 Localization/Globalization
kubectl get localizations -A
kubectl get globalizations -A

# 检查 Agent 连接状态
kubectl logs -n clusternet-system -l app=clusternet-hub | grep "cluster-a"

# 强制重新调度
kubectl annotate subscription nginx-subscription \
  apps.clusternet.io/reschedule="true" --overwrite
```

### 监控配置

```yaml
# Prometheus ServiceMonitor
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: clusternet-hub
  namespace: clusternet-system
spec:
  selector:
    matchLabels:
      app: clusternet-hub
  endpoints:
    - port: metrics
      interval: 30s
---
# 告警规则
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: clusternet-alerts
  namespace: clusternet-system
spec:
  groups:
    - name: clusternet.rules
      rules:
        # 集群离线
        - alert: ClusternetManagedClusterOffline
          expr: |
            time() - clusternet_managed_cluster_last_heartbeat_timestamp > 300
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Managed cluster {{ $labels.cluster }} is offline"

        # 资源分发失败
        - alert: ClusternetManifestApplyFailed
          expr: |
            clusternet_manifest_apply_failures_total > 0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Manifest apply failed for cluster {{ $labels.cluster }}"
```

## 总结

Clusternet 核心特性：

| 特性 | 说明 |
|------|------|
| **Pull 模式** | Agent 主动连接，防火墙友好 |
| **边缘支持** | 弱网络、离线场景优化 |
| **灵活分发** | Subscription + Feed 解耦 |
| **集群定制** | Localization/Globalization 覆盖 |
| **Helm 集成** | 原生支持 HelmChart 部署 |
| **轻量级** | 最小化资源占用 |

Clusternet 适用场景：
- 边缘计算多集群管理
- 防火墙后的集群接入
- 弱网络环境
- 大规模边缘节点管理
- 混合云/多云统一管理
