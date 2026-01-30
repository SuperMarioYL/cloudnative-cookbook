---
title: "Karmada 多集群管理"
weight: 2
---
## 概述

Karmada (Kubernetes Armada) 是 CNCF 孵化项目，提供开放的多云多集群 Kubernetes 管理能力。它继承了 Kubernetes 原生 API，使用户可以像管理单集群一样管理多个集群。本文深入解析 Karmada 的架构设计和核心功能。

## 架构设计

### 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      Karmada 架构                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                 Karmada Control Plane                    │   │
│   │                                                          │   │
│   │  ┌────────────────────────────────────────────────────┐  │   │
│   │  │              Karmada API Server                     │  │   │
│   │  │         (扩展的 Kubernetes API Server)              │  │   │
│   │  └────────────────────────────────────────────────────┘  │   │
│   │                          │                               │   │
│   │  ┌────────────────────────────────────────────────────┐  │   │
│   │  │                   etcd                              │  │   │
│   │  │             (Karmada 状态存储)                       │  │   │
│   │  └────────────────────────────────────────────────────┘  │   │
│   │                          │                               │   │
│   │  ┌────────────────────────────────────────────────────┐  │   │
│   │  │            Karmada Controllers                      │  │   │
│   │  │  ┌──────────────┐  ┌──────────────┐               │  │   │
│   │  │  │ Propagation  │  │  Binding     │               │  │   │
│   │  │  │ Controller   │  │  Controller  │               │  │   │
│   │  │  └──────────────┘  └──────────────┘               │  │   │
│   │  │  ┌──────────────┐  ┌──────────────┐               │  │   │
│   │  │  │ Execution    │  │  Status      │               │  │   │
│   │  │  │ Controller   │  │  Controller  │               │  │   │
│   │  │  └──────────────┘  └──────────────┘               │  │   │
│   │  └────────────────────────────────────────────────────┘  │   │
│   │                          │                               │   │
│   │  ┌────────────────────────────────────────────────────┐  │   │
│   │  │           Karmada Scheduler                         │  │   │
│   │  │         (多集群资源调度决策)                         │  │   │
│   │  └────────────────────────────────────────────────────┘  │   │
│   └──────────────────────────┬──────────────────────────────┘   │
│                              │                                   │
│            ┌─────────────────┼─────────────────┐                │
│            │                 │                 │                 │
│            ▼                 ▼                 ▼                 │
│   ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐  │
│   │  Member Cluster │ │  Member Cluster │ │  Member Cluster │  │
│   │       A         │ │       B         │ │       C         │  │
│   │                 │ │                 │ │                 │  │
│   │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │  │
│   │  │ Karmada   │  │ │  │ Karmada   │  │ │  │ Karmada   │  │  │
│   │  │  Agent    │  │ │  │  Agent    │  │ │  │  Agent    │  │  │
│   │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │  │
│   │                 │ │                 │ │                 │  │
│   │  ┌───────────┐  │ │  ┌───────────┐  │ │  ┌───────────┐  │  │
│   │  │ Workloads │  │ │  │ Workloads │  │ │  │ Workloads │  │  │
│   │  └───────────┘  │ │  └───────────┘  │ │  └───────────┘  │  │
│   └─────────────────┘ └─────────────────┘ └─────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 核心组件

```yaml
# Karmada API Server 配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-apiserver
  namespace: karmada-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: karmada-apiserver
  template:
    metadata:
      labels:
        app: karmada-apiserver
    spec:
      containers:
        - name: karmada-apiserver
          image: registry.k8s.io/karmada/karmada-apiserver:v1.8.0
          command:
            - /bin/karmada-apiserver
            - --allow-privileged=true
            - --authorization-mode=Node,RBAC
            - --client-ca-file=/etc/karmada/pki/ca.crt
            - --enable-admission-plugins=NodeRestriction
            - --enable-bootstrap-token-auth=true
            - --etcd-cafile=/etc/karmada/pki/etcd-ca.crt
            - --etcd-certfile=/etc/karmada/pki/etcd-client.crt
            - --etcd-keyfile=/etc/karmada/pki/etcd-client.key
            - --etcd-servers=https://etcd-0.etcd.karmada-system.svc:2379
            - --bind-address=0.0.0.0
            - --secure-port=5443
            - --service-cluster-ip-range=10.96.0.0/12
            - --tls-cert-file=/etc/karmada/pki/apiserver.crt
            - --tls-private-key-file=/etc/karmada/pki/apiserver.key
          ports:
            - containerPort: 5443
              name: https
          volumeMounts:
            - mountPath: /etc/karmada/pki
              name: karmada-certs
              readOnly: true
      volumes:
        - name: karmada-certs
          secret:
            secretName: karmada-cert
---
# Karmada Controller Manager
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-controller-manager
  namespace: karmada-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: karmada-controller-manager
  template:
    spec:
      containers:
        - name: karmada-controller-manager
          image: registry.k8s.io/karmada/karmada-controller-manager:v1.8.0
          command:
            - /bin/karmada-controller-manager
            - --kubeconfig=/etc/karmada/karmada.kubeconfig
            - --bind-address=0.0.0.0
            - --cluster-status-update-frequency=10s
            - --secure-port=10357
            - --leader-elect=true
            - --leader-elect-resource-namespace=karmada-system
            - --v=4
          volumeMounts:
            - name: karmada-config
              mountPath: /etc/karmada
              readOnly: true
---
# Karmada Scheduler
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-scheduler
  namespace: karmada-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: karmada-scheduler
  template:
    spec:
      containers:
        - name: karmada-scheduler
          image: registry.k8s.io/karmada/karmada-scheduler:v1.8.0
          command:
            - /bin/karmada-scheduler
            - --kubeconfig=/etc/karmada/karmada.kubeconfig
            - --bind-address=0.0.0.0
            - --secure-port=10351
            - --leader-elect=true
            - --leader-elect-resource-namespace=karmada-system
            - --scheduler-estimator-timeout=3s
            - --enable-scheduler-estimator=true
            - --v=4
```

### 资源分发流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   Karmada 资源分发流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐                                                │
│  │    User     │                                                │
│  │  kubectl    │                                                │
│  └──────┬──────┘                                                │
│         │ 1. 创建资源 + PropagationPolicy                       │
│         ▼                                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Karmada API Server                          │    │
│  │                                                          │    │
│  │  资源存储:                                               │    │
│  │  • Deployment (模板)                                     │    │
│  │  • PropagationPolicy                                     │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │ 2. Watch 变化                          │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │           Propagation Controller                         │    │
│  │                                                          │    │
│  │  • 匹配 Resource 和 Policy                               │    │
│  │  • 创建 ResourceBinding                                  │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │ 3. 创建 ResourceBinding                │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Karmada Scheduler                           │    │
│  │                                                          │    │
│  │  • 根据 Policy 决定目标集群                              │    │
│  │  • 计算每个集群的副本数                                  │    │
│  │  • 更新 ResourceBinding.spec.clusters                    │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │ 4. 调度决策                            │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │             Binding Controller                           │    │
│  │                                                          │    │
│  │  • 为每个目标集群创建 Work 对象                          │    │
│  │  • Work 包含要分发的资源模板                             │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │ 5. 创建 Work                           │
│                         ▼                                        │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │            Execution Controller                          │    │
│  │                                                          │    │
│  │  • 读取 Work 中的资源模板                                │    │
│  │  • 应用 OverridePolicy 进行定制                          │    │
│  │  • 通过集群凭证创建资源到成员集群                        │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │ 6. 创建资源到成员集群                  │
│            ┌────────────┼────────────┐                          │
│            ▼            ▼            ▼                          │
│     ┌──────────┐ ┌──────────┐ ┌──────────┐                     │
│     │Cluster A │ │Cluster B │ │Cluster C │                     │
│     │Deployment│ │Deployment│ │Deployment│                     │
│     │(2副本)   │ │(3副本)   │ │(1副本)   │                     │
│     └──────────┘ └──────────┘ └──────────┘                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## 核心资源

### Cluster

```yaml
# 注册成员集群
apiVersion: cluster.karmada.io/v1alpha1
kind: Cluster
metadata:
  name: member-cluster-1
spec:
  syncMode: Push  # Push 或 Pull 模式
  apiEndpoint: https://member-1.example.com:6443
  secretRef:
    namespace: karmada-system
    name: member-cluster-1-secret
  # 集群标签用于调度
  taints:
    - key: cluster.karmada.io/not-ready
      effect: NoSchedule
  # 集群提供者信息
  provider: "AWS"
  region: "us-west-2"
  zone: "us-west-2a"
---
# 集群访问凭证
apiVersion: v1
kind: Secret
metadata:
  name: member-cluster-1-secret
  namespace: karmada-system
type: Opaque
data:
  # kubeconfig 或 token
  kubeconfig: <base64-encoded-kubeconfig>
---
# Pull 模式下的 Agent 部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: karmada-agent
  namespace: karmada-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: karmada-agent
  template:
    spec:
      containers:
        - name: karmada-agent
          image: registry.k8s.io/karmada/karmada-agent:v1.8.0
          command:
            - /bin/karmada-agent
            - --karmada-kubeconfig=/etc/karmada/karmada.kubeconfig
            - --karmada-context=karmada
            - --cluster-name=member-cluster-1
            - --cluster-api-endpoint=https://member-1.example.com:6443
            - --cluster-provider=AWS
            - --cluster-region=us-west-2
            - --leader-elect=true
```

### PropagationPolicy

```yaml
# 基本分发策略
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-propagation
  namespace: default
spec:
  # 资源选择器
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
    - apiVersion: v1
      kind: Service
      name: nginx-svc
    - apiVersion: v1
      kind: ConfigMap
      labelSelector:
        matchLabels:
          app: nginx

  # 分发策略
  placement:
    # 集群亲和性
    clusterAffinity:
      clusterNames:
        - member-cluster-1
        - member-cluster-2

    # 集群容忍
    clusterTolerations:
      - key: cluster.karmada.io/not-ready
        operator: Exists
        effect: NoSchedule

    # 副本调度
    replicaScheduling:
      replicaSchedulingType: Divided  # Duplicated 或 Divided
      replicaDivisionPreference: Weighted
      weightPreference:
        staticWeightList:
          - targetCluster:
              clusterNames:
                - member-cluster-1
            weight: 2
          - targetCluster:
              clusterNames:
                - member-cluster-2
            weight: 1

    # 拓扑分布约束
    spreadConstraints:
      - maxGroups: 2
        minGroups: 1
        spreadByField: region
---
# 集群级别分发策略
apiVersion: policy.karmada.io/v1alpha1
kind: ClusterPropagationPolicy
metadata:
  name: cluster-nginx-propagation
spec:
  resourceSelectors:
    - apiVersion: v1
      kind: Namespace
      name: nginx-system
  placement:
    clusterAffinity:
      labelSelector:
        matchLabels:
          environment: production
---
# 动态调度策略
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: nginx-dynamic
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    # 基于标签选择集群
    clusterAffinity:
      labelSelector:
        matchExpressions:
          - key: environment
            operator: In
            values:
              - production
          - key: region
            operator: In
            values:
              - us-west
              - us-east

    # 动态权重调度
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Dynamic
      dynamicWeight: AvailableReplicas  # 基于可用副本数
```

### OverridePolicy

```yaml
# 基本覆盖策略
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: nginx-override
  namespace: default
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx

  overrideRules:
    # 针对特定集群的覆盖
    - targetCluster:
        clusterNames:
          - member-cluster-1
      overriders:
        # 镜像覆盖
        imageOverrider:
          - component: Registry
            operator: replace
            value: registry-1.example.com
          - component: Tag
            operator: replace
            value: v1.0.0-cluster1

        # 副本数覆盖
        replicasOverrider:
          - value: 3

        # 命令行参数覆盖
        argsOverrider:
          - containerName: nginx
            operator: add
            value:
              - "--config=/etc/nginx/cluster1.conf"

        # 资源覆盖
        resourcesOverrider:
          - containerName: nginx
            value:
              limits:
                cpu: "2"
                memory: "2Gi"
              requests:
                cpu: "500m"
                memory: "512Mi"

    # 针对另一个集群的覆盖
    - targetCluster:
        clusterNames:
          - member-cluster-2
      overriders:
        imageOverrider:
          - component: Registry
            operator: replace
            value: registry-2.example.com

        # 标签覆盖
        labelsOverrider:
          - operator: addIfAbsent
            value:
              cluster: member-cluster-2

        # 注解覆盖
        annotationsOverrider:
          - operator: addIfAbsent
            value:
              monitoring.example.com/enabled: "true"
---
# 使用 JSON Patch 进行复杂覆盖
apiVersion: policy.karmada.io/v1alpha1
kind: OverridePolicy
metadata:
  name: nginx-jsonpatch
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  overrideRules:
    - targetCluster:
        labelSelector:
          matchLabels:
            environment: production
      overriders:
        plaintext:
          - path: "/spec/template/spec/containers/0/env"
            operator: add
            value:
              - name: ENVIRONMENT
                value: production
              - name: LOG_LEVEL
                value: warn
          - path: "/spec/template/spec/affinity"
            operator: add
            value:
              podAntiAffinity:
                preferredDuringSchedulingIgnoredDuringExecution:
                  - weight: 100
                    podAffinityTerm:
                      labelSelector:
                        matchLabels:
                          app: nginx
                      topologyKey: kubernetes.io/hostname
```

### ResourceBinding 和 Work

```yaml
# ResourceBinding (由控制器自动创建)
apiVersion: work.karmada.io/v1alpha2
kind: ResourceBinding
metadata:
  name: nginx-deployment
  namespace: default
spec:
  resource:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
    namespace: default
    resourceVersion: "12345"
  # 调度结果
  clusters:
    - name: member-cluster-1
      replicas: 2
    - name: member-cluster-2
      replicas: 1
status:
  conditions:
    - type: Scheduled
      status: "True"
    - type: FullyApplied
      status: "True"
  aggregatedStatus:
    - clusterName: member-cluster-1
      status:
        replicas: 2
        readyReplicas: 2
        availableReplicas: 2
    - clusterName: member-cluster-2
      status:
        replicas: 1
        readyReplicas: 1
        availableReplicas: 1
---
# Work (由 Binding Controller 创建)
apiVersion: work.karmada.io/v1alpha1
kind: Work
metadata:
  name: nginx-deployment-member-cluster-1
  namespace: karmada-es-member-cluster-1
spec:
  workload:
    manifests:
      - apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: nginx
          namespace: default
        spec:
          replicas: 2  # 已根据调度结果修改
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
                  image: registry-1.example.com/nginx:v1.0.0-cluster1  # 已应用 Override
status:
  conditions:
    - type: Applied
      status: "True"
  manifestStatuses:
    - identifier:
        group: apps
        kind: Deployment
        name: nginx
        namespace: default
        version: v1
      status:
        replicas: 2
        readyReplicas: 2
```

## 调度策略

### 副本调度模式

```yaml
# Duplicated 模式 - 每个集群完整副本
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: duplicated-policy
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
      replicaSchedulingType: Duplicated
      # 每个集群都会有完整的 replicas 数量
---
# Divided 模式 - 副本分散到多个集群
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: divided-policy
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
              clusterNames: [cluster-a]
            weight: 3
          - targetCluster:
              clusterNames: [cluster-b]
            weight: 2
          - targetCluster:
              clusterNames: [cluster-c]
            weight: 1
      # 假设 Deployment replicas=6
      # cluster-a: 3 副本 (6 * 3/6 = 3)
      # cluster-b: 2 副本 (6 * 2/6 = 2)
      # cluster-c: 1 副本 (6 * 1/6 = 1)
```

### 拓扑分布

```yaml
# 按区域分布
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: region-spread
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      labelSelector:
        matchLabels:
          environment: production
    spreadConstraints:
      - maxGroups: 3      # 最多分布到 3 个区域
        minGroups: 2      # 至少分布到 2 个区域
        spreadByField: region  # 按 region 字段分布
        spreadByLabel: ""
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Aggregated  # 尽量聚合到少数集群
---
# 按自定义标签分布
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: custom-spread
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    spreadConstraints:
      - spreadByLabel: topology.kubernetes.io/zone
        maxGroups: 3
        minGroups: 2
```

### 故障转移

```yaml
# 故障转移策略
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: failover-policy
spec:
  resourceSelectors:
    - apiVersion: apps/v1
      kind: Deployment
      name: nginx
  placement:
    clusterAffinity:
      clusterNames:
        - cluster-a
        - cluster-b  # 备份集群
    replicaScheduling:
      replicaSchedulingType: Divided
      replicaDivisionPreference: Weighted
      weightPreference:
        staticWeightList:
          - targetCluster:
              clusterNames: [cluster-a]
            weight: 10  # 主集群
          - targetCluster:
              clusterNames: [cluster-b]
            weight: 0   # 正常情况不调度
  # 故障转移配置
  failover:
    # 当集群不可用时自动迁移
    applicationMigrationPolicy:
      decisionConditions:
        tolerationSeconds: 300  # 容忍 5 分钟
      purgePolicy: Gracefully
---
# 副本重调度
apiVersion: config.karmada.io/v1alpha1
kind: ReplicaSchedulingPolicy
metadata:
  name: replica-reschedule
spec:
  # 当集群资源不足时重新调度
  reschedulePolicy:
    enabled: true
    interval: 60s
    threshold:
      cpuUtilization: 80
      memoryUtilization: 80
```

## 多集群服务

### 服务导出导入

```yaml
# 在源集群导出服务
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: nginx-svc
  namespace: default
---
# 在 Karmada 控制平面查看导出的服务
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: nginx-svc
  namespace: default
spec:
  type: ClusterSetIP
  ports:
    - name: http
      port: 80
      protocol: TCP
status:
  clusters:
    - cluster: member-cluster-1
    - cluster: member-cluster-2
---
# 使用 PropagationPolicy 分发 ServiceExport
apiVersion: policy.karmada.io/v1alpha1
kind: PropagationPolicy
metadata:
  name: service-export-propagation
spec:
  resourceSelectors:
    - apiVersion: multicluster.x-k8s.io/v1alpha1
      kind: ServiceExport
      name: nginx-svc
  placement:
    clusterAffinity:
      clusterNames:
        - member-cluster-1
        - member-cluster-2
```

### 跨集群服务发现

```yaml
# 配置 Karmada 多集群服务
apiVersion: networking.karmada.io/v1alpha1
kind: MultiClusterService
metadata:
  name: nginx-mcs
  namespace: default
spec:
  # 服务类型
  types:
    - CrossCluster
  # 消费集群
  consumerClusters:
    - name: member-cluster-1
    - name: member-cluster-2
  # 提供集群
  providerClusters:
    - name: member-cluster-1
    - name: member-cluster-2
  # 端口配置
  ports:
    - name: http
      port: 80
      targetPort: 8080
---
# 配置跨集群 Ingress
apiVersion: networking.karmada.io/v1alpha1
kind: MultiClusterIngress
metadata:
  name: nginx-mci
  namespace: default
spec:
  template:
    spec:
      rules:
        - host: nginx.example.com
          http:
            paths:
              - path: /
                pathType: Prefix
                backend:
                  service:
                    name: nginx-mcs
                    port:
                      number: 80
```

## 状态聚合

### 聚合状态查看

```yaml
# 查看 Deployment 聚合状态
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: default
spec:
  replicas: 6
status:
  # Karmada 聚合的状态
  replicas: 6
  readyReplicas: 6
  availableReplicas: 6
  conditions:
    - type: Available
      status: "True"
---
# 查看详细的集群状态
# kubectl get resourcebinding nginx-deployment -o yaml
apiVersion: work.karmada.io/v1alpha2
kind: ResourceBinding
metadata:
  name: nginx-deployment
status:
  aggregatedStatus:
    - clusterName: member-cluster-1
      applied: true
      status:
        replicas: 3
        readyReplicas: 3
        availableReplicas: 3
        conditions:
          - type: Available
            status: "True"
    - clusterName: member-cluster-2
      applied: true
      status:
        replicas: 3
        readyReplicas: 3
        availableReplicas: 3
        conditions:
          - type: Available
            status: "True"
```

### 自定义状态聚合

```yaml
# 资源解释器 - 自定义状态聚合
apiVersion: config.karmada.io/v1alpha1
kind: ResourceInterpreterCustomization
metadata:
  name: deployment-interpreter
spec:
  target:
    apiVersion: apps/v1
    kind: Deployment
  customizations:
    # 副本聚合
    replicaResource:
      luaScript: |
        function ReplicasCount(desiredObj)
          if desiredObj.spec.replicas then
            return desiredObj.spec.replicas
          end
          return 1
        end

    # 状态聚合
    statusAggregation:
      luaScript: |
        function AggregateStatus(desiredObj, statusItems)
          if statusItems == nil then
            return desiredObj
          end

          local totalReplicas = 0
          local totalReady = 0
          local totalAvailable = 0

          for _, item in ipairs(statusItems) do
            if item.status then
              totalReplicas = totalReplicas + (item.status.replicas or 0)
              totalReady = totalReady + (item.status.readyReplicas or 0)
              totalAvailable = totalAvailable + (item.status.availableReplicas or 0)
            end
          end

          desiredObj.status = desiredObj.status or {}
          desiredObj.status.replicas = totalReplicas
          desiredObj.status.readyReplicas = totalReady
          desiredObj.status.availableReplicas = totalAvailable

          return desiredObj
        end

    # 健康状态判断
    healthInterpretation:
      luaScript: |
        function InterpretHealth(observedObj)
          if observedObj.status == nil then
            return false
          end
          if observedObj.status.readyReplicas == observedObj.status.replicas then
            return true
          end
          return false
        end

    # 保留字段
    retention:
      luaScript: |
        function Retain(desiredObj, observedObj)
          if observedObj.spec.replicas then
            desiredObj.spec.replicas = observedObj.spec.replicas
          end
          return desiredObj
        end
```

## 最佳实践

### 生产部署清单

```yaml
# Karmada 高可用部署
apiVersion: v1
kind: ConfigMap
metadata:
  name: karmada-ha-config
data:
  deployment: |
    ## 控制平面高可用

    ### API Server
    - 3 副本
    - 负载均衡器前置
    - 独立 etcd 集群

    ### Controller Manager
    - 2 副本
    - Leader Election

    ### Scheduler
    - 2 副本
    - Leader Election

    ### etcd
    - 3 或 5 节点集群
    - SSD 存储
    - 定期备份

  network: |
    ## 网络配置

    ### 成员集群连接
    - 确保控制平面可访问成员集群 API
    - 考虑私有网络/VPN
    - 配置适当的超时

    ### 服务网络
    - 考虑使用 Submariner 实现跨集群网络
    - 或使用 Istio 多集群

  security: |
    ## 安全配置

    ### 证书管理
    - 定期轮换证书
    - 使用短期 Token

    ### RBAC
    - 最小权限原则
    - 按团队/项目分配权限

    ### 网络策略
    - 限制控制平面访问
    - 加密集群间通信
```

### 运维命令

```bash
# Karmada 常用命令

# 查看集群状态
kubectl --kubeconfig /etc/karmada/karmada.kubeconfig get clusters

# 查看资源分发状态
kubectl --kubeconfig /etc/karmada/karmada.kubeconfig get rb,work -A

# 查看 PropagationPolicy
kubectl --kubeconfig /etc/karmada/karmada.kubeconfig get pp,cpp -A

# 查看聚合状态
kubectl --kubeconfig /etc/karmada/karmada.kubeconfig get deploy nginx -o yaml

# 使用 karmadactl
karmadactl get clusters
karmadactl describe cluster member-cluster-1
karmadactl logs -l app=nginx --all-clusters
karmadactl exec -it nginx-xxx --cluster=member-cluster-1 -- sh

# 集群注册
karmadactl join member-cluster-3 \
  --cluster-kubeconfig=/path/to/member-cluster-3.kubeconfig \
  --karmada-context=karmada

# 集群注销
karmadactl unjoin member-cluster-3
```

## 总结

Karmada 核心能力：

| 能力 | 说明 |
|------|------|
| **API 兼容** | 原生 Kubernetes API，零学习成本 |
| **灵活调度** | 多种副本调度策略，支持故障转移 |
| **差异化配置** | OverridePolicy 实现集群定制 |
| **状态聚合** | 跨集群资源状态统一视图 |
| **多集群服务** | 跨集群服务发现和负载均衡 |
| **可扩展性** | 支持自定义资源解释器 |

Karmada 适用场景：
- 多云/混合云统一管理
- 跨地域应用部署
- 多集群灾备
- 大规模集群联邦
