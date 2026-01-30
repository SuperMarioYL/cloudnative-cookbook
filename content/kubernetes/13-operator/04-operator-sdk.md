---
title: "Operator SDK"
weight: 4
---
## 概述

Operator SDK 是 Red Hat 维护的 Operator 开发框架，基于 Kubebuilder 构建，提供了额外的企业特性，包括 OLM 集成、Scorecard 测试、Ansible/Helm Operator 支持等。

## 安装与初始化

### 安装 Operator SDK

```bash
# macOS
brew install operator-sdk

# Linux
export ARCH=$(case $(uname -m) in x86_64) echo -n amd64 ;; aarch64) echo -n arm64 ;; *) echo -n $(uname -m) ;; esac)
export OS=$(uname | awk '{print tolower($0)}')
export OPERATOR_SDK_DL_URL=https://github.com/operator-framework/operator-sdk/releases/download/v1.34.0
curl -LO ${OPERATOR_SDK_DL_URL}/operator-sdk_${OS}_${ARCH}
chmod +x operator-sdk_${OS}_${ARCH}
sudo mv operator-sdk_${OS}_${ARCH} /usr/local/bin/operator-sdk

# 验证
operator-sdk version
```

### 项目初始化

```bash
# Go 项目（与 Kubebuilder 兼容）
operator-sdk init \
    --domain example.com \
    --repo github.com/example/database-operator

# Ansible 项目
operator-sdk init \
    --plugins=ansible \
    --domain example.com

# Helm 项目
operator-sdk init \
    --plugins=helm \
    --domain example.com
```

## Go Operator

### 创建 API

```bash
# 创建 API 和控制器
operator-sdk create api \
    --group database \
    --version v1 \
    --kind Database \
    --resource \
    --controller

# 创建 Webhook
operator-sdk create webhook \
    --group database \
    --version v1 \
    --kind Database \
    --defaulting \
    --programmatic-validation
```

### 与 Kubebuilder 差异

```go
// Operator SDK 提供额外的工具和注解

// +operator-sdk:csv:customresourcedefinitions:displayName="Database"
// +operator-sdk:csv:customresourcedefinitions:resources={{StatefulSet,v1,""},{Service,v1,""}}

// Database is the Schema for the databases API
type Database struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   DatabaseSpec   `json:"spec,omitempty"`
    Status DatabaseStatus `json:"status,omitempty"`
}
```

## Ansible Operator

### 项目结构

```
ansible-operator/
├── Dockerfile
├── Makefile
├── PROJECT
├── config/
│   ├── crd/
│   ├── default/
│   ├── manager/
│   ├── prometheus/
│   ├── rbac/
│   └── samples/
├── molecule/                  # 测试
│   ├── default/
│   └── kind/
├── playbooks/                 # Ansible playbooks
├── roles/
│   └── database/              # Ansible role
│       ├── defaults/
│       │   └── main.yml
│       ├── files/
│       ├── handlers/
│       │   └── main.yml
│       ├── meta/
│       │   └── main.yml
│       ├── tasks/
│       │   └── main.yml
│       ├── templates/
│       └── vars/
│           └── main.yml
├── requirements.yml           # Ansible 依赖
└── watches.yaml               # CR 到 Role 映射
```

### watches.yaml

```yaml
# watches.yaml - 定义 CR 与 Ansible Role/Playbook 的映射
---
- version: v1
  group: database.example.com
  kind: Database
  role: database
  # 或使用 playbook
  # playbook: playbooks/database.yml

  # 可选配置
  reconcilePeriod: 0s
  manageStatus: true
  watchDependentResources: true
  watchClusterScopedResources: false

  # 覆盖默认值
  overrideValues:
    key: value

  # 选择器
  selector:
    matchLabels:
      managed: "true"
```

### Ansible Role

```yaml
# roles/database/tasks/main.yml
---
- name: Create ConfigMap
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: "{{ ansible_operator_meta.name }}-config"
        namespace: "{{ ansible_operator_meta.namespace }}"
      data:
        config.yaml: |
          engine: {{ engine }}
          version: {{ version }}

- name: Create Secret
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        name: "{{ ansible_operator_meta.name }}-credentials"
        namespace: "{{ ansible_operator_meta.namespace }}"
      type: Opaque
      stringData:
        username: admin
        password: "{{ lookup('password', '/dev/null length=16 chars=ascii_letters,digits') }}"
  when: not secret_exists

- name: Create StatefulSet
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: apps/v1
      kind: StatefulSet
      metadata:
        name: "{{ ansible_operator_meta.name }}"
        namespace: "{{ ansible_operator_meta.namespace }}"
        labels:
          app: database
          instance: "{{ ansible_operator_meta.name }}"
      spec:
        replicas: "{{ replicas | default(1) }}"
        serviceName: "{{ ansible_operator_meta.name }}"
        selector:
          matchLabels:
            app: database
            instance: "{{ ansible_operator_meta.name }}"
        template:
          metadata:
            labels:
              app: database
              instance: "{{ ansible_operator_meta.name }}"
          spec:
            containers:
              - name: database
                image: "{{ image }}:{{ version }}"
                ports:
                  - containerPort: "{{ port }}"
                env:
                  - name: DB_PASSWORD
                    valueFrom:
                      secretKeyRef:
                        name: "{{ ansible_operator_meta.name }}-credentials"
                        key: password
                volumeMounts:
                  - name: data
                    mountPath: /var/lib/database
        volumeClaimTemplates:
          - metadata:
              name: data
            spec:
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: "{{ storage_size | default('10Gi') }}"

- name: Create Service
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Service
      metadata:
        name: "{{ ansible_operator_meta.name }}"
        namespace: "{{ ansible_operator_meta.namespace }}"
      spec:
        clusterIP: None
        selector:
          app: database
          instance: "{{ ansible_operator_meta.name }}"
        ports:
          - port: "{{ port }}"
            targetPort: "{{ port }}"

- name: Update status
  operator_sdk.util.k8s_status:
    api_version: database.example.com/v1
    kind: Database
    name: "{{ ansible_operator_meta.name }}"
    namespace: "{{ ansible_operator_meta.namespace }}"
    status:
      phase: Running
      endpoint: "{{ ansible_operator_meta.name }}.{{ ansible_operator_meta.namespace }}.svc:{{ port }}"
```

### 默认变量

```yaml
# roles/database/defaults/main.yml
---
engine: postgresql
version: "14.5"
replicas: 1
storage_size: "10Gi"

# 引擎配置
images:
  mysql: mysql
  postgresql: postgres
  mongodb: mongo

ports:
  mysql: 3306
  postgresql: 5432
  mongodb: 27017

image: "{{ images[engine] }}"
port: "{{ ports[engine] }}"
```

## Helm Operator

### 项目结构

```
helm-operator/
├── Dockerfile
├── Makefile
├── PROJECT
├── config/
│   ├── crd/
│   ├── default/
│   ├── manager/
│   ├── prometheus/
│   ├── rbac/
│   └── samples/
├── helm-charts/
│   └── database/             # Helm chart
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── _helpers.tpl
│           ├── configmap.yaml
│           ├── secret.yaml
│           ├── service.yaml
│           └── statefulset.yaml
└── watches.yaml
```

### watches.yaml

```yaml
# watches.yaml
---
- group: database.example.com
  version: v1
  kind: Database
  chart: helm-charts/database

  # 覆盖默认值
  overrideValues:
    image.tag: $RELATED_IMAGE_DATABASE

  # 选择器
  selector:
    matchLabels:
      managed: "true"

  # 设置 reconcile 周期
  reconcilePeriod: 10m
```

### Helm Chart

```yaml
# helm-charts/database/Chart.yaml
apiVersion: v2
name: database
description: A Helm chart for Database
version: 0.1.0
appVersion: "1.0.0"

# helm-charts/database/values.yaml
engine: postgresql
version: "14.5"

replicaCount: 1

image:
  repository: postgres
  pullPolicy: IfNotPresent
  tag: ""

storage:
  size: 10Gi
  storageClass: ""

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 100m
    memory: 256Mi
```

```yaml
# helm-charts/database/templates/statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: {{ include "database.fullname" . }}
  labels:
    {{- include "database.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  serviceName: {{ include "database.fullname" . }}
  selector:
    matchLabels:
      {{- include "database.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "database.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.version }}"
          ports:
            - containerPort: {{ include "database.port" . }}
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "database.fullname" . }}-credentials
                  key: password
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        {{- if .Values.storage.storageClass }}
        storageClassName: {{ .Values.storage.storageClass }}
        {{- end }}
        resources:
          requests:
            storage: {{ .Values.storage.size }}
```

## OLM 集成

### Bundle 格式

```
bundle/
├── manifests/
│   ├── database.example.com_databases.yaml    # CRD
│   └── database-operator.clusterserviceversion.yaml  # CSV
├── metadata/
│   └── annotations.yaml
├── tests/
│   └── scorecard/
│       └── config.yaml
└── Dockerfile
```

### 生成 Bundle

```bash
# 生成 Bundle 清单
make bundle IMG=example.com/database-operator:v1.0.0

# 构建 Bundle 镜像
make bundle-build bundle-push BUNDLE_IMG=example.com/database-operator-bundle:v1.0.0
```

### ClusterServiceVersion

```yaml
# bundle/manifests/database-operator.clusterserviceversion.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: database-operator.v1.0.0
  namespace: placeholder
  annotations:
    capabilities: Full Lifecycle
    categories: Database
    description: Manage databases on Kubernetes
    containerImage: example.com/database-operator:v1.0.0
    repository: https://github.com/example/database-operator
    support: Example Inc.
    alm-examples: |
      [
        {
          "apiVersion": "database.example.com/v1",
          "kind": "Database",
          "metadata": {
            "name": "example-database"
          },
          "spec": {
            "engine": "postgresql",
            "version": "14.5",
            "replicas": 1,
            "storage": {
              "size": "10Gi"
            }
          }
        }
      ]
spec:
  displayName: Database Operator
  description: |
    ## Overview
    Database Operator manages database instances on Kubernetes.

    ## Features
    - Automated deployment
    - High availability
    - Backup and restore
    - Monitoring integration

  version: 1.0.0
  replaces: database-operator.v0.9.0
  maturity: stable
  minKubeVersion: 1.25.0

  keywords:
    - database
    - postgresql
    - mysql
    - mongodb

  maintainers:
    - name: Example Inc.
      email: support@example.com

  provider:
    name: Example Inc.
    url: https://example.com

  links:
    - name: Documentation
      url: https://example.com/docs
    - name: Source Code
      url: https://github.com/example/database-operator

  icon:
    - base64data: |
        iVBORw0KGgoAAAANSUhEUgAA...
      mediatype: image/png

  installModes:
    - type: OwnNamespace
      supported: true
    - type: SingleNamespace
      supported: true
    - type: MultiNamespace
      supported: false
    - type: AllNamespaces
      supported: true

  install:
    strategy: deployment
    spec:
      clusterPermissions:
        - serviceAccountName: database-operator-controller-manager
          rules:
            - apiGroups: [""]
              resources: ["configmaps", "secrets", "services"]
              verbs: ["*"]
            - apiGroups: ["apps"]
              resources: ["statefulsets"]
              verbs: ["*"]
            - apiGroups: ["database.example.com"]
              resources: ["databases", "databases/status", "databases/finalizers"]
              verbs: ["*"]

      deployments:
        - name: database-operator-controller-manager
          spec:
            replicas: 1
            selector:
              matchLabels:
                control-plane: controller-manager
            template:
              metadata:
                labels:
                  control-plane: controller-manager
              spec:
                serviceAccountName: database-operator-controller-manager
                containers:
                  - name: manager
                    image: example.com/database-operator:v1.0.0
                    args:
                      - --leader-elect
                    resources:
                      limits:
                        cpu: 500m
                        memory: 128Mi
                      requests:
                        cpu: 10m
                        memory: 64Mi

  customresourcedefinitions:
    owned:
      - name: databases.database.example.com
        version: v1
        kind: Database
        displayName: Database
        description: Represents a database instance
        resources:
          - kind: StatefulSet
            version: v1
          - kind: Service
            version: v1
          - kind: ConfigMap
            version: v1
          - kind: Secret
            version: v1
        specDescriptors:
          - path: engine
            displayName: Engine
            description: Database engine type
            x-descriptors:
              - urn:alm:descriptor:com.tectonic.ui:select:mysql
              - urn:alm:descriptor:com.tectonic.ui:select:postgresql
              - urn:alm:descriptor:com.tectonic.ui:select:mongodb
          - path: version
            displayName: Version
            description: Database version
            x-descriptors:
              - urn:alm:descriptor:com.tectonic.ui:text
          - path: replicas
            displayName: Replicas
            description: Number of replicas
            x-descriptors:
              - urn:alm:descriptor:com.tectonic.ui:podCount
        statusDescriptors:
          - path: phase
            displayName: Phase
            description: Current phase
            x-descriptors:
              - urn:alm:descriptor:io.kubernetes.phase
          - path: endpoint
            displayName: Endpoint
            description: Connection endpoint
            x-descriptors:
              - urn:alm:descriptor:text
```

### 发布到 Catalog

```bash
# 创建 Catalog 镜像
opm index add \
    --bundles example.com/database-operator-bundle:v1.0.0 \
    --tag example.com/database-operator-index:v1.0.0

# 推送 Catalog
docker push example.com/database-operator-index:v1.0.0

# 创建 CatalogSource
cat <<EOF | kubectl apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: database-operator-catalog
  namespace: olm
spec:
  sourceType: grpc
  image: example.com/database-operator-index:v1.0.0
  displayName: Database Operator
  publisher: Example Inc.
EOF
```

## Scorecard 测试

### 配置

```yaml
# bundle/tests/scorecard/config.yaml
apiVersion: scorecard.operatorframework.io/v1alpha3
kind: Configuration
metadata:
  name: config
stages:
  - parallel: true
    tests:
      - entrypoint:
          - scorecard-test
          - basic-check-spec
        image: quay.io/operator-framework/scorecard-test:latest
        labels:
          suite: basic
          test: basic-check-spec-test
        storage:
          spec:
            mountPath: {}

      - entrypoint:
          - scorecard-test
          - olm-bundle-validation
        image: quay.io/operator-framework/scorecard-test:latest
        labels:
          suite: olm
          test: olm-bundle-validation-test

      - entrypoint:
          - scorecard-test
          - olm-crds-have-validation
        image: quay.io/operator-framework/scorecard-test:latest
        labels:
          suite: olm
          test: olm-crds-have-validation-test

      - entrypoint:
          - scorecard-test
          - olm-crds-have-resources
        image: quay.io/operator-framework/scorecard-test:latest
        labels:
          suite: olm
          test: olm-crds-have-resources-test

      - entrypoint:
          - scorecard-test
          - olm-spec-descriptors
        image: quay.io/operator-framework/scorecard-test:latest
        labels:
          suite: olm
          test: olm-spec-descriptors-test

      - entrypoint:
          - scorecard-test
          - olm-status-descriptors
        image: quay.io/operator-framework/scorecard-test:latest
        labels:
          suite: olm
          test: olm-status-descriptors-test
```

### 运行测试

```bash
# 运行所有测试
operator-sdk scorecard bundle

# 运行特定测试
operator-sdk scorecard bundle --selector=suite=basic

# 输出 JSON
operator-sdk scorecard bundle --output=json

# 指定 kubeconfig
operator-sdk scorecard bundle --kubeconfig=/path/to/kubeconfig

# 等待超时
operator-sdk scorecard bundle --wait-time=300s
```

### 自定义测试

```go
// 自定义 Scorecard 测试
package main

import (
    "context"
    "fmt"

    scapiv1alpha3 "github.com/operator-framework/api/pkg/apis/scorecard/v1alpha3"
    apimanifests "github.com/operator-framework/api/pkg/manifests"
)

func main() {
    result := scapiv1alpha3.TestResult{
        Name:   "custom-test",
        State:  scapiv1alpha3.PassState,
        Errors: []string{},
        Suggestions: []string{},
    }

    // 运行自定义测试逻辑
    bundle, err := apimanifests.GetBundleFromDir("/bundle")
    if err != nil {
        result.State = scapiv1alpha3.FailState
        result.Errors = append(result.Errors, err.Error())
    }

    // 验证 Bundle
    if bundle.CSV == nil {
        result.State = scapiv1alpha3.FailState
        result.Errors = append(result.Errors, "CSV not found")
    }

    // 输出结果
    fmt.Printf("%+v\n", result)
}
```

## 升级与迁移

### 版本升级

```bash
# 检查项目是否需要升级
operator-sdk pkgman-to-bundle --help

# 从 packagemanifest 迁移到 bundle
operator-sdk pkgman-to-bundle /path/to/packagemanifest --output-dir /path/to/bundle
```

### 项目布局迁移

```bash
# 检查当前布局版本
cat PROJECT

# 迁移到新布局
operator-sdk migration upgrade

# 手动迁移步骤
# 1. 更新 PROJECT 文件
# 2. 更新目录结构
# 3. 更新 Makefile
# 4. 更新依赖
```

## 最佳实践

### Operator 类型选择

| 类型 | 适用场景 | 优势 | 劣势 |
|-----|---------|-----|-----|
| Go | 复杂逻辑、高性能 | 完全控制、类型安全 | 学习曲线高 |
| Ansible | 运维自动化、现有 Playbook | 快速开发、易于维护 | 性能较低 |
| Helm | 已有 Helm Chart | 最快开发、易于迁移 | 灵活性有限 |

### CSV 描述符

```yaml
# 常用描述符
specDescriptors:
  # 文本输入
  - path: name
    x-descriptors:
      - urn:alm:descriptor:com.tectonic.ui:text

  # 数字输入
  - path: replicas
    x-descriptors:
      - urn:alm:descriptor:com.tectonic.ui:number

  # 下拉选择
  - path: engine
    x-descriptors:
      - urn:alm:descriptor:com.tectonic.ui:select:mysql
      - urn:alm:descriptor:com.tectonic.ui:select:postgresql

  # 布尔开关
  - path: enabled
    x-descriptors:
      - urn:alm:descriptor:com.tectonic.ui:booleanSwitch

  # 资源限制
  - path: resources
    x-descriptors:
      - urn:alm:descriptor:com.tectonic.ui:resourceRequirements

  # 密码
  - path: password
    x-descriptors:
      - urn:alm:descriptor:com.tectonic.ui:password

statusDescriptors:
  # 阶段
  - path: phase
    x-descriptors:
      - urn:alm:descriptor:io.kubernetes.phase

  # 条件
  - path: conditions
    x-descriptors:
      - urn:alm:descriptor:io.kubernetes.conditions

  # Pod 数量
  - path: replicas
    x-descriptors:
      - urn:alm:descriptor:com.tectonic.ui:podCount
```

## 总结

Operator SDK 核心要点：

**项目类型**
- Go Operator：完全控制
- Ansible Operator：运维友好
- Helm Operator：快速迁移

**OLM 集成**
- Bundle 格式
- ClusterServiceVersion
- Catalog 发布

**Scorecard 测试**
- 内置测试套件
- 自定义测试
- CI 集成

**迁移升级**
- 版本升级工具
- 布局迁移
- 向后兼容
