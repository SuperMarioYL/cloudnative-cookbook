---
title: "Kustomize"
weight: 5
---
## 概述

Kustomize 是 Kubernetes 原生的配置管理工具，内置于 kubectl 中。它采用声明式方式，通过 overlay 模式管理不同环境的配置差异，无需模板语法。本文深入解析 Kustomize 的核心概念和实践方法。

## 核心概念

### 基本结构

```
project/
├── base/                      # 基础配置
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── configmap.yaml
└── overlays/                  # 环境覆盖层
    ├── development/
    │   ├── kustomization.yaml
    │   └── replica-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── namespace.yaml
    └── production/
        ├── kustomization.yaml
        ├── replica-patch.yaml
        ├── hpa.yaml
        └── resources-patch.yaml
```

### Base 配置

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 元数据
metadata:
  name: web-app-base

# 资源文件
resources:
  - deployment.yaml
  - service.yaml
  - configmap.yaml
  - serviceaccount.yaml

# 通用标签
commonLabels:
  app.kubernetes.io/name: web-app
  app.kubernetes.io/component: backend

# 通用注解
commonAnnotations:
  app.kubernetes.io/managed-by: kustomize

# 镜像
images:
  - name: web-app
    newName: myregistry/web-app
    newTag: latest
```

```yaml
# base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      serviceAccountName: web-app
      containers:
        - name: web-app
          image: web-app  # 将被 kustomization 中的 images 替换
          ports:
            - containerPort: 8080
          envFrom:
            - configMapRef:
                name: web-app-config
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
```

```yaml
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web-app
  ports:
    - port: 80
      targetPort: 8080
```

```yaml
# base/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
data:
  LOG_LEVEL: "info"
  ENVIRONMENT: "default"
```

## Overlay 覆盖层

### Development Overlay

```yaml
# overlays/development/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 命名空间
namespace: development

# 引用 base
resources:
  - ../../base

# 名称前缀/后缀
namePrefix: dev-

# 标签
commonLabels:
  environment: development

# 镜像覆盖
images:
  - name: web-app
    newName: myregistry/web-app
    newTag: dev-latest

# 补丁
patches:
  - path: replica-patch.yaml
  - path: resources-patch.yaml

# ConfigMap 生成器
configMapGenerator:
  - name: web-app-config
    behavior: merge
    literals:
      - LOG_LEVEL=debug
      - ENVIRONMENT=development
      - DEBUG=true
```

```yaml
# overlays/development/replica-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 1
```

```yaml
# overlays/development/resources-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
        - name: web-app
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
```

### Production Overlay

```yaml
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base
  - hpa.yaml
  - pdb.yaml
  - ingress.yaml

namePrefix: prod-

commonLabels:
  environment: production

commonAnnotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8080"

images:
  - name: web-app
    newName: myregistry/web-app
    newTag: v1.2.0

# 副本补丁
replicas:
  - name: web-app
    count: 5

# Strategic Merge Patch
patches:
  - path: deployment-patch.yaml

# JSON 6902 Patch
patchesJson6902:
  - target:
      group: apps
      version: v1
      kind: Deployment
      name: web-app
    patch: |-
      - op: add
        path: /spec/template/spec/affinity
        value:
          podAntiAffinity:
            preferredDuringSchedulingIgnoredDuringExecution:
              - weight: 100
                podAffinityTerm:
                  labelSelector:
                    matchLabels:
                      app: web-app
                  topologyKey: kubernetes.io/hostname

# ConfigMap 生成器
configMapGenerator:
  - name: web-app-config
    behavior: merge
    literals:
      - LOG_LEVEL=warn
      - ENVIRONMENT=production

# Secret 生成器
secretGenerator:
  - name: web-app-secrets
    files:
      - secrets/api-key.txt
    literals:
      - DB_PASSWORD=secret123
```

```yaml
# overlays/production/deployment-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  template:
    spec:
      containers:
        - name: web-app
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
            limits:
              cpu: "2"
              memory: "2Gi"
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: web-app-secrets
                  key: DB_PASSWORD
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web-app
```

```yaml
# overlays/production/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 5
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```yaml
# overlays/production/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-app
spec:
  minAvailable: 3
  selector:
    matchLabels:
      app: web-app
```

## 高级功能

### 组件 (Components)

```yaml
# components/monitoring/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1alpha1
kind: Component

# 添加监控相关配置
patches:
  - patch: |-
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: web-app
      spec:
        template:
          metadata:
            annotations:
              prometheus.io/scrape: "true"
              prometheus.io/port: "8080"
              prometheus.io/path: "/metrics"
          spec:
            containers:
              - name: web-app
                ports:
                  - name: metrics
                    containerPort: 8080

resources:
  - servicemonitor.yaml
```

```yaml
# components/monitoring/servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web-app
spec:
  selector:
    matchLabels:
      app: web-app
  endpoints:
    - port: http
      path: /metrics
      interval: 30s
```

```yaml
# 使用组件
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

components:
  - ../../components/monitoring
  - ../../components/logging
```

### 转换器 (Transformers)

```yaml
# transformers/labels-transformer.yaml
apiVersion: builtin
kind: LabelTransformer
metadata:
  name: labels-transformer
labels:
  team: platform
  cost-center: "12345"
fieldSpecs:
  - path: metadata/labels
    create: true
  - path: spec/selector
    create: false
    kind: Deployment
  - path: spec/template/metadata/labels
    create: true
    kind: Deployment
```

```yaml
# kustomization.yaml
transformers:
  - transformers/labels-transformer.yaml
```

### 生成器

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# ConfigMap 生成器
configMapGenerator:
  # 从文件生成
  - name: app-config
    files:
      - config.json
      - settings.yaml
    options:
      disableNameSuffixHash: false

  # 从字面值生成
  - name: env-config
    literals:
      - LOG_LEVEL=info
      - MAX_CONNECTIONS=100

  # 从 env 文件生成
  - name: dotenv-config
    envs:
      - .env

# Secret 生成器
secretGenerator:
  - name: db-credentials
    files:
      - username=secrets/db-user.txt
      - password=secrets/db-pass.txt
    type: kubernetes.io/basic-auth

  - name: tls-secret
    files:
      - tls.crt=certs/server.crt
      - tls.key=certs/server.key
    type: kubernetes.io/tls

# 生成器选项
generatorOptions:
  disableNameSuffixHash: false
  labels:
    generated: "true"
  annotations:
    generated-by: kustomize
```

### 变量替换

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - deployment.yaml
  - service.yaml

# 定义变量
vars:
  - name: SERVICE_NAME
    objref:
      kind: Service
      name: web-app
      apiVersion: v1
    fieldref:
      fieldpath: metadata.name

# 替换变量的位置
replacements:
  - source:
      kind: Service
      name: web-app
      fieldPath: metadata.name
    targets:
      - select:
          kind: Deployment
          name: web-app
        fieldPaths:
          - spec.template.spec.containers.[name=web-app].env.[name=SERVICE_NAME].value
```

### Helm Chart 集成

```yaml
# kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# 使用 Helm Chart 作为资源
helmCharts:
  - name: nginx
    repo: https://charts.bitnami.com/bitnami
    version: 15.0.0
    releaseName: web-server
    namespace: production
    valuesFile: values.yaml
    includeCRDs: false
```

## 常用命令

```bash
# 构建并查看输出
kubectl kustomize overlays/production
kustomize build overlays/production

# 应用到集群
kubectl apply -k overlays/production
kustomize build overlays/production | kubectl apply -f -

# 查看差异
kubectl diff -k overlays/production

# 删除资源
kubectl delete -k overlays/production

# 编辑 kustomization
kustomize edit add resource deployment.yaml
kustomize edit set image web-app=myregistry/web-app:v2.0.0
kustomize edit add label app:web-app
kustomize edit add annotation team:platform

# 设置命名空间
kustomize edit set namespace production

# 设置名称前缀/后缀
kustomize edit set nameprefix prod-
kustomize edit set namesuffix -v2
```

## 最佳实践

### 目录结构建议

```
kubernetes/
├── base/                           # 基础配置
│   ├── kustomization.yaml
│   └── ...
├── components/                     # 可复用组件
│   ├── monitoring/
│   ├── logging/
│   └── security/
├── overlays/                       # 环境覆盖层
│   ├── development/
│   ├── staging/
│   └── production/
└── clusters/                       # 集群特定配置
    ├── us-west-1/
    └── eu-central-1/
```

### 配置检查清单

| 检查项 | 说明 |
|--------|------|
| Base 独立性 | Base 应该可以独立运行 |
| 最小化 Overlay | 只包含环境差异 |
| 使用组件 | 共享功能提取为 Component |
| 避免重复 | 使用 Strategic Merge Patch |
| 版本控制 | 所有配置纳入 Git |
| 验证配置 | 使用 `kubectl diff -k` 验证 |

### 与 GitOps 集成

```yaml
# Argo CD Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-app
  namespace: argocd
spec:
  source:
    repoURL: https://github.com/myorg/k8s-configs.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
```

## 总结

Kustomize 提供了一种简洁的方式管理 Kubernetes 配置：

1. **无模板**：直接使用 YAML，无需学习模板语法
2. **Overlay 模式**：清晰分离基础配置和环境差异
3. **内置支持**：kubectl 原生支持，无需额外安装
4. **可组合**：通过 Component 实现配置复用
5. **GitOps 友好**：声明式配置，易于版本控制

Kustomize 适合管理中等复杂度的 Kubernetes 配置，对于更复杂的场景可以考虑与 Helm 结合使用。
