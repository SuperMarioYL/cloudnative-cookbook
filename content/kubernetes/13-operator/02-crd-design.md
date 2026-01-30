---
title: "CRD 设计"
weight: 2
---
## 概述

CustomResourceDefinition（CRD）是 Kubernetes 的扩展机制，允许用户定义新的资源类型。良好的 CRD 设计是 Operator 成功的基础。本章介绍 CRD 的设计原则和最佳实践。

## CRD 结构

### 完整定义

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # 名称格式: <plural>.<group>
  name: databases.example.com
  # 标签
  labels:
    app.kubernetes.io/name: database-operator
  # 注解
  annotations:
    controller-gen.kubebuilder.io/version: v0.14.0
spec:
  # API 组
  group: example.com

  # 资源名称
  names:
    kind: Database
    listKind: DatabaseList
    plural: databases
    singular: database
    shortNames:
      - db
      - dbs
    categories:
      - all
      - database

  # 作用域: Namespaced 或 Cluster
  scope: Namespaced

  # 版本列表
  versions:
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          # ... schema 定义
      subresources:
        status: {}
        scale:
          specReplicasPath: .spec.replicas
          statusReplicasPath: .status.readyReplicas
      additionalPrinterColumns:
        # ... 打印列
```

### 资源名称设计

```yaml
names:
  # Kind: PascalCase，单数
  kind: Database

  # ListKind: Kind + List
  listKind: DatabaseList

  # Plural: 小写，复数，用于 API 路径
  # GET /apis/example.com/v1/namespaces/default/databases
  plural: databases

  # Singular: 小写，单数，用于 kubectl
  singular: database

  # ShortNames: 简短别名
  shortNames:
    - db

  # Categories: 资源分类
  # kubectl get all 会包含此资源
  categories:
    - all
```

## Schema 设计

### 基本结构

```yaml
schema:
  openAPIV3Schema:
    type: object
    required:
      - spec
    properties:
      apiVersion:
        type: string
      kind:
        type: string
      metadata:
        type: object
      spec:
        type: object
        # spec 属性定义
      status:
        type: object
        # status 属性定义
```

### Spec 设计

```yaml
spec:
  type: object
  required:
    - engine
    - version
  properties:
    # 字符串字段
    engine:
      type: string
      description: Database engine type
      enum:
        - mysql
        - postgresql
        - mongodb
      default: postgresql

    # 字符串带模式验证
    version:
      type: string
      description: Database version
      pattern: "^[0-9]+\\.[0-9]+\\.[0-9]+$"

    # 整数字段
    replicas:
      type: integer
      description: Number of replicas
      minimum: 1
      maximum: 10
      default: 1

    # 布尔字段
    highAvailability:
      type: boolean
      description: Enable high availability mode
      default: false

    # 资源数量（Quantity）
    storage:
      type: string
      description: Storage size
      pattern: "^[0-9]+[KMGTPE]i?$"
      default: "10Gi"

    # 嵌套对象
    resources:
      type: object
      properties:
        requests:
          type: object
          properties:
            cpu:
              type: string
            memory:
              type: string
        limits:
          type: object
          properties:
            cpu:
              type: string
            memory:
              type: string

    # 数组字段
    users:
      type: array
      items:
        type: object
        required:
          - name
        properties:
          name:
            type: string
          privileges:
            type: array
            items:
              type: string
              enum:
                - read
                - write
                - admin

    # 引用字段
    secretRef:
      type: object
      required:
        - name
      properties:
        name:
          type: string
        key:
          type: string
          default: password

    # 可选的嵌套对象（使用 x-kubernetes-preserve-unknown-fields）
    config:
      type: object
      x-kubernetes-preserve-unknown-fields: true
      description: Additional database configuration
```

### Status 设计

```yaml
status:
  type: object
  properties:
    # 阶段
    phase:
      type: string
      enum:
        - Pending
        - Creating
        - Running
        - Updating
        - Failed
        - Terminating

    # 副本状态
    replicas:
      type: integer
    readyReplicas:
      type: integer
    updatedReplicas:
      type: integer

    # 观察到的代
    observedGeneration:
      type: integer
      format: int64

    # 连接信息
    endpoint:
      type: string
    port:
      type: integer

    # 条件列表
    conditions:
      type: array
      items:
        type: object
        required:
          - type
          - status
        properties:
          type:
            type: string
          status:
            type: string
            enum:
              - "True"
              - "False"
              - "Unknown"
          reason:
            type: string
          message:
            type: string
          lastTransitionTime:
            type: string
            format: date-time
          lastUpdateTime:
            type: string
            format: date-time

    # 最后操作
    lastOperation:
      type: object
      properties:
        type:
          type: string
          enum:
            - Create
            - Update
            - Delete
            - Backup
            - Restore
        state:
          type: string
          enum:
            - Processing
            - Succeeded
            - Failed
        description:
          type: string
        lastUpdateTime:
          type: string
          format: date-time
```

## 验证规则

### OpenAPI 验证

```yaml
spec:
  type: object
  properties:
    # 必填字段
    name:
      type: string
      minLength: 1
      maxLength: 63
      pattern: "^[a-z0-9]([-a-z0-9]*[a-z0-9])?$"

    # 数值范围
    replicas:
      type: integer
      minimum: 1
      maximum: 100
      multipleOf: 1

    # 字符串枚举
    tier:
      type: string
      enum:
        - basic
        - standard
        - premium

    # 数组长度
    nodes:
      type: array
      minItems: 1
      maxItems: 10
      items:
        type: string

    # 互斥字段
    oneOf:
      - required: [configMapRef]
      - required: [secretRef]
```

### CEL 验证（Kubernetes 1.25+）

```yaml
schema:
  openAPIV3Schema:
    type: object
    properties:
      spec:
        type: object
        x-kubernetes-validations:
          # 字段级验证
          - rule: "self.replicas >= 1"
            message: "replicas must be at least 1"

          # 跨字段验证
          - rule: "self.minReplicas <= self.maxReplicas"
            message: "minReplicas must not exceed maxReplicas"

          # 条件验证
          - rule: "!self.highAvailability || self.replicas >= 3"
            message: "HA mode requires at least 3 replicas"

          # 字符串验证
          - rule: "self.name.matches('^[a-z][a-z0-9-]*$')"
            message: "name must start with lowercase letter"

        properties:
          replicas:
            type: integer
            x-kubernetes-validations:
              - rule: "self >= 1 && self <= 100"
                message: "replicas must be between 1 and 100"

          name:
            type: string
            x-kubernetes-validations:
              - rule: "size(self) <= 63"
                message: "name cannot exceed 63 characters"
```

### 不可变字段

```yaml
spec:
  type: object
  properties:
    # 使用 CEL 标记不可变字段
    engine:
      type: string
      x-kubernetes-validations:
        - rule: "self == oldSelf"
          message: "engine is immutable"
```

## 版本管理

### 多版本定义

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
  names:
    kind: Database
    plural: databases
  scope: Namespaced
  versions:
    # v1 版本（存储版本）
    - name: v1
      served: true
      storage: true
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                engine:
                  type: string
                replicas:
                  type: integer

    # v1beta1 版本（旧版本，仍可访问）
    - name: v1beta1
      served: true
      storage: false
      deprecated: true
      deprecationWarning: "v1beta1 is deprecated, use v1 instead"
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                type:  # 旧字段名
                  type: string
                size:  # 旧字段名
                  type: integer
```

### 版本转换 Webhook

```go
// Conversion Hub - v1 作为内部版本
func (r *Database) Hub() {}

// ConvertTo - v1beta1 转换到 v1
func (r *DatabaseV1beta1) ConvertTo(dstRaw conversion.Hub) error {
    dst := dstRaw.(*Database)

    // 复制基本字段
    dst.ObjectMeta = r.ObjectMeta

    // 字段映射
    dst.Spec.Engine = r.Spec.Type      // type -> engine
    dst.Spec.Replicas = r.Spec.Size    // size -> replicas

    // 状态复制
    dst.Status = DatabaseStatus(r.Status)

    return nil
}

// ConvertFrom - v1 转换到 v1beta1
func (r *DatabaseV1beta1) ConvertFrom(srcRaw conversion.Hub) error {
    src := srcRaw.(*Database)

    // 复制基本字段
    r.ObjectMeta = src.ObjectMeta

    // 字段映射
    r.Spec.Type = src.Spec.Engine      // engine -> type
    r.Spec.Size = src.Spec.Replicas    // replicas -> size

    // 状态复制
    r.Status = DatabaseV1beta1Status(src.Status)

    return nil
}
```

### 存储版本迁移

```bash
# 查看存储版本
kubectl get crd databases.example.com -o jsonpath='{.status.storedVersions}'

# 触发迁移
kubectl get databases --all-namespaces -o yaml | kubectl apply -f -

# 更新存储版本
kubectl patch crd databases.example.com --type='json' \
  -p='[{"op": "replace", "path": "/status/storedVersions", "value":["v1"]}]'
```

## 子资源

### Status 子资源

```yaml
subresources:
  status: {}
```

```go
// 使用 Status 子资源更新
// 这样更新 status 不会触发 spec 的 webhook
func (r *Reconciler) updateStatus(ctx context.Context, db *examplev1.Database) error {
    db.Status.Phase = "Running"
    db.Status.ReadyReplicas = 3

    // 使用 Status 客户端
    return r.Status().Update(ctx, db)
}
```

### Scale 子资源

```yaml
subresources:
  scale:
    # spec 中副本数路径
    specReplicasPath: .spec.replicas
    # status 中副本数路径
    statusReplicasPath: .status.readyReplicas
    # 可选：标签选择器路径
    labelSelectorPath: .status.selector
```

```bash
# 使用 kubectl scale
kubectl scale database my-db --replicas=5

# HPA 可以使用 scale 子资源
kubectl autoscale database my-db --min=2 --max=10 --cpu-percent=80
```

## 打印列

### 自定义 kubectl 输出

```yaml
additionalPrinterColumns:
  # 字符串列
  - name: Engine
    type: string
    description: Database engine type
    jsonPath: .spec.engine

  # 整数列
  - name: Replicas
    type: integer
    description: Number of replicas
    jsonPath: .spec.replicas

  # 状态列
  - name: Ready
    type: string
    description: Ready replicas
    jsonPath: .status.readyReplicas

  # 阶段列
  - name: Phase
    type: string
    description: Current phase
    jsonPath: .status.phase

  # 布尔列
  - name: HA
    type: boolean
    description: High availability enabled
    jsonPath: .spec.highAvailability

  # 日期列
  - name: Age
    type: date
    jsonPath: .metadata.creationTimestamp

  # 优先级（低优先级在 -o wide 时显示）
  - name: Endpoint
    type: string
    jsonPath: .status.endpoint
    priority: 1
```

```bash
# 默认输出
kubectl get databases
# NAME    ENGINE      REPLICAS   READY   PHASE     HA      AGE
# my-db   postgresql  3          3       Running   true    5m

# 宽输出
kubectl get databases -o wide
# NAME    ENGINE      REPLICAS   READY   PHASE     HA      AGE    ENDPOINT
# my-db   postgresql  3          3       Running   true    5m     my-db.default.svc:5432
```

## Go Types 定义

### 类型定义

```go
// api/v1/database_types.go

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas
// +kubebuilder:printcolumn:name="Engine",type=string,JSONPath=`.spec.engine`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
// +kubebuilder:resource:shortName=db

// Database is the Schema for the databases API
type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DatabaseSpec   `json:"spec,omitempty"`
    Status DatabaseStatus `json:"status,omitempty"`
}

// DatabaseSpec defines the desired state of Database
type DatabaseSpec struct {
    // Engine specifies the database engine type
    // +kubebuilder:validation:Enum=mysql;postgresql;mongodb
    // +kubebuilder:default=postgresql
    Engine string `json:"engine"`

    // Version specifies the database version
    // +kubebuilder:validation:Pattern=`^[0-9]+\.[0-9]+\.[0-9]+$`
    Version string `json:"version"`

    // Replicas is the number of database instances
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    // +kubebuilder:default=1
    Replicas int32 `json:"replicas,omitempty"`

    // Storage is the storage size
    // +kubebuilder:validation:Pattern=`^[0-9]+[KMGTPE]i?$`
    // +kubebuilder:default="10Gi"
    Storage string `json:"storage,omitempty"`

    // HighAvailability enables HA mode
    // +kubebuilder:default=false
    HighAvailability bool `json:"highAvailability,omitempty"`

    // Resources defines the resource requirements
    // +optional
    Resources *ResourceRequirements `json:"resources,omitempty"`

    // Config is additional configuration
    // +kubebuilder:pruning:PreserveUnknownFields
    // +optional
    Config *runtime.RawExtension `json:"config,omitempty"`
}

// ResourceRequirements defines resource requirements
type ResourceRequirements struct {
    // Requests describes the minimum resources required
    // +optional
    Requests ResourceList `json:"requests,omitempty"`

    // Limits describes the maximum resources allowed
    // +optional
    Limits ResourceList `json:"limits,omitempty"`
}

// ResourceList is a map of resource names to quantities
type ResourceList map[string]resource.Quantity

// DatabaseStatus defines the observed state of Database
type DatabaseStatus struct {
    // Phase is the current phase of the database
    // +kubebuilder:validation:Enum=Pending;Creating;Running;Updating;Failed;Terminating
    Phase string `json:"phase,omitempty"`

    // Replicas is the total number of replicas
    Replicas int32 `json:"replicas,omitempty"`

    // ReadyReplicas is the number of ready replicas
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`

    // ObservedGeneration is the most recent generation observed
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Endpoint is the connection endpoint
    Endpoint string `json:"endpoint,omitempty"`

    // Conditions represent the latest available observations
    // +patchMergeKey=type
    // +patchStrategy=merge
    // +listType=map
    // +listMapKey=type
    Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`
}

// +kubebuilder:object:root=true

// DatabaseList contains a list of Database
type DatabaseList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []Database `json:"items"`
}

func init() {
    SchemeBuilder.Register(&Database{}, &DatabaseList{})
}
```

### 标记注解参考

| 标记 | 说明 |
|-----|------|
| `+kubebuilder:object:root=true` | 标记为根对象 |
| `+kubebuilder:subresource:status` | 启用 status 子资源 |
| `+kubebuilder:subresource:scale` | 启用 scale 子资源 |
| `+kubebuilder:printcolumn` | 添加打印列 |
| `+kubebuilder:resource` | 资源配置 |
| `+kubebuilder:validation:Enum` | 枚举验证 |
| `+kubebuilder:validation:Minimum` | 最小值验证 |
| `+kubebuilder:validation:Maximum` | 最大值验证 |
| `+kubebuilder:validation:Pattern` | 正则验证 |
| `+kubebuilder:default` | 默认值 |
| `+optional` | 可选字段 |
| `+kubebuilder:pruning:PreserveUnknownFields` | 保留未知字段 |

## 最佳实践

### 命名规范

```yaml
# API 组命名
# - 使用组织域名的子域
# - 避免使用 k8s.io、kubernetes.io
group: database.example.com

# Kind 命名
# - 使用 PascalCase
# - 使用具体名词
# - 避免 Manager、Controller 等后缀
kind: Database  # 好
kind: DatabaseManager  # 不好

# 字段命名
# - 使用 camelCase
# - 简洁但有意义
# - 与 Kubernetes 惯例一致
spec:
  replicas: 3     # 好
  replicaCount: 3  # 不好（与 K8s 不一致）
```

### 向后兼容

```go
// 1. 新字段应该有默认值
type Spec struct {
    // 新字段，有默认值
    // +kubebuilder:default=true
    NewFeature bool `json:"newFeature,omitempty"`
}

// 2. 废弃字段使用注解
type Spec struct {
    // Deprecated: use NewField instead
    // +optional
    OldField string `json:"oldField,omitempty"`

    // NewField replaces OldField
    NewField string `json:"newField,omitempty"`
}

// 3. 在 Webhook 中处理兼容性
func (r *Database) Default() {
    // 迁移旧字段到新字段
    if r.Spec.OldField != "" && r.Spec.NewField == "" {
        r.Spec.NewField = r.Spec.OldField
    }
}
```

### 文档化

```go
// DatabaseSpec defines the desired state of Database
// +kubebuilder:object:generate=true
type DatabaseSpec struct {
    // Engine specifies the database engine type.
    // Supported values are: mysql, postgresql, mongodb.
    // This field is immutable after creation.
    //
    // +kubebuilder:validation:Enum=mysql;postgresql;mongodb
    // +kubebuilder:validation:XValidation:rule="self == oldSelf",message="engine is immutable"
    Engine string `json:"engine"`

    // Version specifies the database version in semver format.
    // Example: "14.5.0" for PostgreSQL 14.5.0
    //
    // +kubebuilder:validation:Pattern=`^[0-9]+\.[0-9]+\.[0-9]+$`
    Version string `json:"version"`
}
```

## 总结

CRD 设计核心要点：

**结构设计**
- spec/status 分离
- 清晰的字段定义
- 合理的默认值

**验证规则**
- OpenAPI Schema 验证
- CEL 表达式验证
- 不可变字段保护

**版本管理**
- 多版本支持
- 转换 Webhook
- 存储版本迁移

**子资源**
- status 独立更新
- scale 支持扩缩容

**最佳实践**
- 命名规范
- 向后兼容
- 充分文档化
