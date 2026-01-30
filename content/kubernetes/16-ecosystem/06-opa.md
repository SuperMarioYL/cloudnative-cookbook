---
title: "OPA 策略引擎"
weight: 6
---
## 概述

Open Policy Agent (OPA) 是一个通用的策略引擎，可以在整个技术栈中统一策略实施。在 Kubernetes 中，OPA 通过 Gatekeeper 项目提供准入控制策略，实现"策略即代码"的理念。本文深入解析 OPA 的核心概念和在 Kubernetes 中的实践应用。

## OPA 架构

### 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         OPA 架构                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      决策请求                             │  │
│  │   ┌─────────┐    ┌─────────┐    ┌─────────┐              │  │
│  │   │ K8s API │    │ 微服务  │    │   CI/CD │              │  │
│  │   │ Server  │    │   网关  │    │   系统  │              │  │
│  │   └────┬────┘    └────┬────┘    └────┬────┘              │  │
│  │        │              │              │                    │  │
│  │        └──────────────┼──────────────┘                    │  │
│  │                       ▼                                   │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    OPA 引擎                               │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │                   查询接口                          │  │  │
│  │  │    REST API    │    Go SDK    │    WASM           │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                          │                                │  │
│  │                          ▼                                │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │                  策略评估器                          │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │  │  │
│  │  │  │  Rego    │  │  编译器  │  │   虚拟机 (Topdown)│ │  │  │
│  │  │  │  解析器  │──│          │──│                  │ │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────────────┘ │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                          │                                │  │
│  │                          ▼                                │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │                    数据存储                          │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────────────┐ │  │  │
│  │  │  │  策略    │  │  外部    │  │      缓存        │ │  │  │
│  │  │  │ (Rego)   │  │  数据    │  │                  │ │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────────────┘ │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      决策结果                             │  │
│  │            Allow / Deny + 原因说明                        │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Gatekeeper 架构

```yaml
# Gatekeeper 组件架构
apiVersion: v1
kind: Namespace
metadata:
  name: gatekeeper-system
---
# Gatekeeper 控制器部署
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gatekeeper-controller-manager
  namespace: gatekeeper-system
spec:
  replicas: 3
  selector:
    matchLabels:
      control-plane: controller-manager
      gatekeeper.sh/operation: webhook
  template:
    metadata:
      labels:
        control-plane: controller-manager
        gatekeeper.sh/operation: webhook
    spec:
      containers:
        - name: manager
          image: openpolicyagent/gatekeeper:v3.15.0
          args:
            - --port=8443
            - --logtostderr
            - --exempt-namespace=gatekeeper-system
            - --operation=webhook
            - --operation=mutation-webhook
            - --disable-opa-builtin={http.send}
          ports:
            - containerPort: 8443
              name: webhook-server
              protocol: TCP
            - containerPort: 8888
              name: metrics
              protocol: TCP
            - containerPort: 9090
              name: healthz
              protocol: TCP
          resources:
            limits:
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 256Mi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
          volumeMounts:
            - mountPath: /certs
              name: cert
              readOnly: true
      volumes:
        - name: cert
          secret:
            defaultMode: 420
            secretName: gatekeeper-webhook-server-cert
---
# Gatekeeper 审计控制器
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gatekeeper-audit
  namespace: gatekeeper-system
spec:
  replicas: 1
  selector:
    matchLabels:
      control-plane: audit-controller
      gatekeeper.sh/operation: audit
  template:
    metadata:
      labels:
        control-plane: audit-controller
        gatekeeper.sh/operation: audit
    spec:
      containers:
        - name: manager
          image: openpolicyagent/gatekeeper:v3.15.0
          args:
            - --audit-interval=60
            - --audit-from-cache
            - --audit-chunk-size=500
            - --audit-match-kind-only
            - --operation=audit
            - --constraint-violations-limit=20
          resources:
            limits:
              memory: 512Mi
            requests:
              cpu: 100m
              memory: 256Mi
```

### Gatekeeper 工作流程

```
┌─────────────────────────────────────────────────────────────────┐
│                   Gatekeeper 准入控制流程                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────┐                                               │
│   │   kubectl   │                                               │
│   │   apply     │                                               │
│   └──────┬──────┘                                               │
│          │                                                       │
│          ▼                                                       │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    API Server                            │  │
│   │  ┌───────────────────────────────────────────────────┐  │  │
│   │  │              Admission Webhook                     │  │  │
│   │  │                                                    │  │  │
│   │  │  ┌────────────────┐    ┌────────────────────────┐ │  │  │
│   │  │  │  Mutating      │───▶│  Validating            │ │  │  │
│   │  │  │  Webhook       │    │  Webhook               │ │  │  │
│   │  │  │ (gatekeeper)   │    │  (gatekeeper)          │ │  │  │
│   │  │  └────────────────┘    └────────────────────────┘ │  │  │
│   │  └───────────────────────────────────────────────────┘  │  │
│   └──────────────────────┬──────────────────────────────────┘  │
│                          │                                      │
│                          ▼                                      │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                  Gatekeeper Controller                   │  │
│   │                                                          │  │
│   │  ┌──────────────────────────────────────────────────┐   │  │
│   │  │              请求处理流程                          │   │  │
│   │  │                                                    │   │  │
│   │  │  1. 接收 AdmissionReview 请求                     │   │  │
│   │  │              │                                     │   │  │
│   │  │              ▼                                     │   │  │
│   │  │  2. 匹配 Constraint (按 Kind/Namespace)           │   │  │
│   │  │              │                                     │   │  │
│   │  │              ▼                                     │   │  │
│   │  │  3. 获取对应的 ConstraintTemplate                 │   │  │
│   │  │              │                                     │   │  │
│   │  │              ▼                                     │   │  │
│   │  │  4. 执行 Rego 策略评估                            │   │  │
│   │  │              │                                     │   │  │
│   │  │              ▼                                     │   │  │
│   │  │  5. 返回 Allow/Deny 结果                          │   │  │
│   │  └──────────────────────────────────────────────────┘   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                     结果处理                             │  │
│   │                                                          │  │
│   │  Allow ──▶ 资源创建/更新成功                            │  │
│   │                                                          │  │
│   │  Deny  ──▶ 返回错误信息给用户                           │  │
│   │            "admission webhook denied the request:        │  │
│   │             [policy-name] violation message"             │  │
│   └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

## Rego 语言

### 基础语法

```rego
# 策略包声明
package kubernetes.admission

# 导入库
import future.keywords.in
import future.keywords.if
import future.keywords.contains

# 常量定义
default allow := false

# 规则定义 - 拒绝没有标签的 Pod
deny contains msg if {
    input.request.kind.kind == "Pod"
    pod := input.request.object
    not pod.metadata.labels
    msg := "Pod must have labels"
}

# 规则定义 - 拒绝使用 latest 标签的镜像
deny contains msg if {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    endswith(container.image, ":latest")
    msg := sprintf("Container '%s' uses 'latest' tag", [container.name])
}

# 带参数的规则
deny contains msg if {
    input.request.kind.kind == "Pod"
    container := input.request.object.spec.containers[_]
    not container_resource_limits_set(container)
    msg := sprintf("Container '%s' must set resource limits", [container.name])
}

# 辅助函数
container_resource_limits_set(container) if {
    container.resources.limits.cpu
    container.resources.limits.memory
}

# 条件组合
deny contains msg if {
    input.request.kind.kind == "Deployment"
    deployment := input.request.object
    deployment.spec.replicas < 2
    not is_allowed_namespace(deployment.metadata.namespace)
    msg := "Production deployments must have at least 2 replicas"
}

is_allowed_namespace(ns) if {
    ns in {"kube-system", "gatekeeper-system"}
}
```

### 数据查询

```rego
package kubernetes.policies

import future.keywords.in
import future.keywords.if
import future.keywords.contains

# 访问输入数据
# input.request.object - 请求的资源对象
# input.request.oldObject - 更新时的旧对象
# input.request.operation - CREATE/UPDATE/DELETE
# input.request.userInfo - 请求者信息

# 遍历数组
deny contains msg if {
    pod := input.request.object
    container := pod.spec.containers[i]
    container.securityContext.privileged == true
    msg := sprintf("Container %d (%s) cannot be privileged", [i, container.name])
}

# 遍历对象
deny contains msg if {
    pod := input.request.object
    label_key := object.keys(pod.metadata.labels)[_]
    startswith(label_key, "internal.")
    msg := sprintf("Label key '%s' uses reserved prefix 'internal.'", [label_key])
}

# 集合操作
required_labels := {"app", "version", "owner"}

missing_labels contains label if {
    pod := input.request.object
    label := required_labels[_]
    not pod.metadata.labels[label]
}

deny contains msg if {
    count(missing_labels) > 0
    msg := sprintf("Pod is missing required labels: %v", [missing_labels])
}

# 字符串操作
deny contains msg if {
    container := input.request.object.spec.containers[_]
    image := container.image

    # 分割镜像名称
    parts := split(image, "/")
    registry := parts[0]

    # 检查注册表
    not startswith(registry, "approved-registry.com")
    msg := sprintf("Image '%s' must come from approved registry", [image])
}

# 正则表达式
deny contains msg if {
    pod := input.request.object
    name := pod.metadata.name
    not regex.match("^[a-z][a-z0-9-]*[a-z0-9]$", name)
    msg := sprintf("Pod name '%s' does not match naming convention", [name])
}

# JSON Path 风格访问
deny contains msg if {
    pod := input.request.object
    volume := pod.spec.volumes[_]
    volume.hostPath
    not volume.hostPath.type in {"Directory", "File"}
    msg := sprintf("HostPath volume '%s' must specify type", [volume.name])
}
```

### 外部数据

```rego
package kubernetes.policies

import future.keywords.if
import future.keywords.contains

# 使用外部数据 (通过 data 关键字)
# data.inventory - Gatekeeper 缓存的集群资源
# data.parameters - Constraint 中定义的参数

# 检查 Ingress 主机名唯一性
deny contains msg if {
    input.request.kind.kind == "Ingress"
    host := input.request.object.spec.rules[_].host

    # 查询现有 Ingress
    existing := data.inventory.namespace[ns][kins].Ingress[name]
    existing.spec.rules[_].host == host

    # 排除自身
    not same_ingress(existing)

    msg := sprintf("Host '%s' already used by Ingress %s/%s", [host, ns, name])
}

same_ingress(existing) if {
    existing.metadata.namespace == input.request.object.metadata.namespace
    existing.metadata.name == input.request.object.metadata.name
}

# 检查资源配额
deny contains msg if {
    input.request.kind.kind == "Pod"
    pod := input.request.object

    # 获取命名空间的资源配额
    quota := data.inventory.namespace[pod.metadata.namespace]["v1"].ResourceQuota["default-quota"]

    # 检查是否超出限制
    requested_cpu := sum_container_cpu(pod)
    to_number(quota.status.hard.cpu) < to_number(quota.status.used.cpu) + requested_cpu

    msg := "Namespace CPU quota would be exceeded"
}

sum_container_cpu(pod) := total if {
    total := sum([parse_cpu(c.resources.requests.cpu) | c := pod.spec.containers[_]])
}

parse_cpu(cpu_str) := result if {
    endswith(cpu_str, "m")
    result := to_number(trim_suffix(cpu_str, "m"))
} else := result if {
    result := to_number(cpu_str) * 1000
}
```

## Gatekeeper 核心资源

### ConstraintTemplate

```yaml
# 镜像来源限制模板
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedrepos
  annotations:
    metadata.gatekeeper.sh/title: "Allowed Repositories"
    metadata.gatekeeper.sh/version: 1.0.0
    description: >-
      Requires container images to begin with a string from the specified list.
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRepos
      validation:
        # Schema for the `parameters` field
        openAPIV3Schema:
          type: object
          properties:
            repos:
              description: The list of prefixes a container image is allowed to have.
              type: array
              items:
                type: string
            exemptImages:
              description: Images to exempt from the policy
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sallowedrepos

        import future.keywords.in
        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          not is_exempt(container.image)
          satisfied := [good | repo := input.parameters.repos[_]; good := startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("container <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        violation contains {"msg": msg} if {
          container := input.review.object.spec.initContainers[_]
          not is_exempt(container.image)
          satisfied := [good | repo := input.parameters.repos[_]; good := startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("initContainer <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        violation contains {"msg": msg} if {
          container := input.review.object.spec.ephemeralContainers[_]
          not is_exempt(container.image)
          satisfied := [good | repo := input.parameters.repos[_]; good := startswith(container.image, repo)]
          not any(satisfied)
          msg := sprintf("ephemeralContainer <%v> has an invalid image repo <%v>, allowed repos are %v", [container.name, container.image, input.parameters.repos])
        }

        is_exempt(image) if {
          exempt := input.parameters.exemptImages[_]
          image == exempt
        }

        is_exempt(image) if {
          exempt := input.parameters.exemptImages[_]
          startswith(image, exempt)
        }
---
# 必需标签模板
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: object
                properties:
                  key:
                    type: string
                  allowedRegex:
                    type: string
                required:
                  - key
            message:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        import future.keywords.if
        import future.keywords.contains

        get_message(parameters, _default) := msg if {
          not parameters.message
          msg := _default
        }

        get_message(parameters, _) := parameters.message

        violation contains {"msg": msg} if {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_].key}
          missing := required - provided
          count(missing) > 0
          def_msg := sprintf("you must provide labels: %v", [missing])
          msg := get_message(input.parameters, def_msg)
        }

        violation contains {"msg": msg} if {
          label := input.parameters.labels[_]
          label.allowedRegex
          value := input.review.object.metadata.labels[label.key]
          not regex.match(label.allowedRegex, value)
          def_msg := sprintf("Label <%v: %v> does not satisfy allowed regex: %v", [label.key, value, label.allowedRegex])
          msg := get_message(input.parameters, def_msg)
        }
---
# 资源限制模板
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8scontainerlimits
spec:
  crd:
    spec:
      names:
        kind: K8sContainerLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            cpu:
              description: Maximum CPU limit
              type: string
            memory:
              description: Maximum memory limit
              type: string
            exemptImages:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8scontainerlimits

        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          not is_exempt(container.image)
          not container.resources.limits.cpu
          msg := sprintf("container <%v> does not have a CPU limit set", [container.name])
        }

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          not is_exempt(container.image)
          not container.resources.limits.memory
          msg := sprintf("container <%v> does not have a memory limit set", [container.name])
        }

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          not is_exempt(container.image)
          input.parameters.cpu
          cpu_limit := container.resources.limits.cpu
          cpu_limit_millicores := parse_cpu(cpu_limit)
          max_cpu_millicores := parse_cpu(input.parameters.cpu)
          cpu_limit_millicores > max_cpu_millicores
          msg := sprintf("container <%v> CPU limit <%v> exceeds maximum <%v>", [container.name, cpu_limit, input.parameters.cpu])
        }

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          not is_exempt(container.image)
          input.parameters.memory
          mem_limit := container.resources.limits.memory
          mem_limit_bytes := parse_memory(mem_limit)
          max_mem_bytes := parse_memory(input.parameters.memory)
          mem_limit_bytes > max_mem_bytes
          msg := sprintf("container <%v> memory limit <%v> exceeds maximum <%v>", [container.name, mem_limit, input.parameters.memory])
        }

        is_exempt(image) if {
          exempt := input.parameters.exemptImages[_]
          startswith(image, exempt)
        }

        parse_cpu(cpu) := millicores if {
          endswith(cpu, "m")
          millicores := to_number(trim_suffix(cpu, "m"))
        } else := millicores if {
          millicores := to_number(cpu) * 1000
        }

        parse_memory(mem) := bytes if {
          endswith(mem, "Ki")
          bytes := to_number(trim_suffix(mem, "Ki")) * 1024
        } else := bytes if {
          endswith(mem, "Mi")
          bytes := to_number(trim_suffix(mem, "Mi")) * 1024 * 1024
        } else := bytes if {
          endswith(mem, "Gi")
          bytes := to_number(trim_suffix(mem, "Gi")) * 1024 * 1024 * 1024
        } else := bytes if {
          bytes := to_number(mem)
        }
```

### Constraint

```yaml
# 应用镜像来源限制
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["Deployment", "StatefulSet", "DaemonSet"]
    namespaces:
      - production
      - staging
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
    labelSelector:
      matchExpressions:
        - key: policy-exempt
          operator: DoesNotExist
  parameters:
    repos:
      - "gcr.io/company/"
      - "docker.io/company/"
      - "quay.io/company/"
    exemptImages:
      - "gcr.io/kubebuilder/kube-rbac-proxy"
---
# 应用必需标签
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-team-label
spec:
  enforcementAction: warn  # 警告模式，不阻止
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
  parameters:
    labels:
      - key: team
      - key: cost-center
        allowedRegex: "^cc-[0-9]{6}$"
    message: "All namespaces must have 'team' and 'cost-center' (format: cc-XXXXXX) labels"
---
# 应用资源限制
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sContainerLimits
metadata:
  name: container-must-have-limits
spec:
  enforcementAction: dryrun  # 审计模式
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - production
  parameters:
    cpu: "4"
    memory: "8Gi"
    exemptImages:
      - "istio/proxyv2"
```

### Mutation

```yaml
# 启用 Mutation
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: gatekeeper-system
spec:
  match:
    - excludedNamespaces: ["kube-system", "gatekeeper-system"]
      processes: ["*"]
  mutation:
    enabled: true
---
# Assign - 添加或修改字段
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: add-default-resource-limits
spec:
  applyTo:
    - groups: [""]
      kinds: ["Pod"]
      versions: ["v1"]
  match:
    scope: Namespaced
    kinds:
      - apiGroups: ["*"]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
  location: "spec.containers[name:*].resources.limits.memory"
  parameters:
    assign:
      value: "256Mi"
    pathTests:
      - subPath: "spec.containers[name:*].resources.limits.memory"
        condition: MustNotExist
---
# AssignMetadata - 添加标签/注解
apiVersion: mutations.gatekeeper.sh/v1
kind: AssignMetadata
metadata:
  name: add-managed-by-label
spec:
  match:
    scope: Namespaced
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  location: "metadata.labels.managed-by"
  parameters:
    assign:
      value: "gatekeeper"
---
# ModifySet - 修改数组
apiVersion: mutations.gatekeeper.sh/v1
kind: ModifySet
metadata:
  name: add-image-pull-secret
spec:
  applyTo:
    - groups: [""]
      kinds: ["Pod"]
      versions: ["v1"]
  match:
    scope: Namespaced
    kinds:
      - apiGroups: ["*"]
        kinds: ["Pod"]
    namespaces:
      - production
  location: "spec.imagePullSecrets"
  parameters:
    operation: merge
    values:
      fromList:
        - name: registry-credentials
```

## 常见策略

### 安全策略

```yaml
# 禁止特权容器
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spspprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sPSPPrivilegedContainer
      validation:
        openAPIV3Schema:
          type: object
          properties:
            exemptImages:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspprivileged

        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg, "details": {}} if {
          c := input_containers[_]
          not is_exempt(c.image)
          c.securityContext.privileged
          msg := sprintf("Privileged container is not allowed: %v, securityContext: %v", [c.name, c.securityContext])
        }

        input_containers[c] if {
          c := input.review.object.spec.containers[_]
        }

        input_containers[c] if {
          c := input.review.object.spec.initContainers[_]
        }

        input_containers[c] if {
          c := input.review.object.spec.ephemeralContainers[_]
        }

        is_exempt(image) if {
          exempt := input.parameters.exemptImages[_]
          glob.match(exempt, [], image)
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    exemptImages:
      - "gcr.io/istio-release/proxyv2:*"
---
# 禁止 hostPath
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spsphostfilesystem
spec:
  crd:
    spec:
      names:
        kind: K8sPSPHostFilesystem
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedHostPaths:
              type: array
              items:
                type: object
                properties:
                  pathPrefix:
                    type: string
                  readOnly:
                    type: boolean
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spsphostfilesystem

        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg, "details": {}} if {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          not input_hostpath_allowed(volume.hostPath.path)
          msg := sprintf("HostPath volume %v is not allowed, pod: %v. Allowed path prefixes: %v", [volume.hostPath.path, input.review.object.metadata.name, input.parameters.allowedHostPaths])
        }

        input_hostpath_allowed(path) if {
          allowedPrefix := input.parameters.allowedHostPaths[_].pathPrefix
          startswith(path, allowedPrefix)
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPHostFilesystem
metadata:
  name: restrict-hostpath
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    allowedHostPaths:
      - pathPrefix: "/var/log"
        readOnly: true
      - pathPrefix: "/tmp"
---
# 要求只读根文件系统
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8spspreadonlyrootfilesystem
spec:
  crd:
    spec:
      names:
        kind: K8sPSPReadOnlyRootFilesystem
      validation:
        openAPIV3Schema:
          type: object
          properties:
            exemptImages:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8spspreadonlyrootfilesystem

        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg, "details": {}} if {
          c := input_containers[_]
          not is_exempt(c.image)
          not c.securityContext.readOnlyRootFilesystem == true
          msg := sprintf("Container %v must set securityContext.readOnlyRootFilesystem=true", [c.name])
        }

        input_containers[c] if {
          c := input.review.object.spec.containers[_]
        }

        input_containers[c] if {
          c := input.review.object.spec.initContainers[_]
        }

        is_exempt(image) if {
          exempt := input.parameters.exemptImages[_]
          glob.match(exempt, [], image)
        }
```

### 资源策略

```yaml
# Pod 资源请求
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
      validation:
        openAPIV3Schema:
          type: object
          properties:
            limits:
              type: array
              items:
                type: string
            requests:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          required := input.parameters.limits[_]
          not container.resources.limits[required]
          msg := sprintf("Container <%v> must set limits for <%v>", [container.name, required])
        }

        violation contains {"msg": msg} if {
          container := input.review.object.spec.containers[_]
          required := input.parameters.requests[_]
          not container.resources.requests[required]
          msg := sprintf("Container <%v> must set requests for <%v>", [container.name, required])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: container-must-have-resources
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - production
      - staging
  parameters:
    limits:
      - cpu
      - memory
    requests:
      - cpu
      - memory
---
# 副本数要求
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sminreplicas
spec:
  crd:
    spec:
      names:
        kind: K8sMinReplicas
      validation:
        openAPIV3Schema:
          type: object
          properties:
            minReplicas:
              type: integer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sminreplicas

        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg} if {
          input.review.object.spec.replicas < input.parameters.minReplicas
          msg := sprintf("Deployment must have at least %v replicas, got %v", [input.parameters.minReplicas, input.review.object.spec.replicas])
        }
---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sMinReplicas
metadata:
  name: deployment-min-replicas
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces:
      - production
  parameters:
    minReplicas: 2
```

### 网络策略

```yaml
# 禁止 LoadBalancer Service
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblocknodeport
spec:
  crd:
    spec:
      names:
        kind: K8sBlockNodePort
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sblocknodeport

        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg} if {
          input.review.kind.kind == "Service"
          input.review.object.spec.type == "NodePort"
          msg := "NodePort services are not allowed"
        }

        violation contains {"msg": msg} if {
          input.review.kind.kind == "Service"
          input.review.object.spec.type == "LoadBalancer"
          msg := "LoadBalancer services are not allowed"
        }
---
# 要求 NetworkPolicy
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequirenetworkpolicy
spec:
  crd:
    spec:
      names:
        kind: K8sRequireNetworkPolicy
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequirenetworkpolicy

        import future.keywords.if
        import future.keywords.contains

        violation contains {"msg": msg} if {
          input.review.kind.kind == "Pod"
          namespace := input.review.object.metadata.namespace

          # 检查命名空间是否有 NetworkPolicy
          not namespace_has_network_policy(namespace)
          msg := sprintf("Namespace %v must have at least one NetworkPolicy before deploying Pods", [namespace])
        }

        namespace_has_network_policy(namespace) if {
          data.inventory.namespace[namespace]["networking.k8s.io/v1"].NetworkPolicy[_]
        }
```

## 审计与合规

### 审计配置

```yaml
# 配置审计
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: gatekeeper-system
spec:
  sync:
    syncOnly:
      - group: ""
        version: "v1"
        kind: "Namespace"
      - group: ""
        version: "v1"
        kind: "Pod"
      - group: "apps"
        version: "v1"
        kind: "Deployment"
      - group: "networking.k8s.io"
        version: "v1"
        kind: "Ingress"
      - group: "networking.k8s.io"
        version: "v1"
        kind: "NetworkPolicy"
  validation:
    traces:
      - user: "*"
        kind:
          group: ""
          version: "v1"
          kind: "Pod"
        dump: "All"
```

### 违规查看

```bash
# 查看所有 Constraint 状态
kubectl get constraints

# 查看特定 Constraint 的违规
kubectl get k8sallowedrepos allowed-repos -o yaml

# 输出示例
# status:
#   auditTimestamp: "2024-01-15T10:30:00Z"
#   totalViolations: 5
#   violations:
#   - enforcementAction: deny
#     group: ""
#     kind: Pod
#     message: 'container <nginx> has an invalid image repo <nginx:latest>'
#     name: test-pod
#     namespace: default
#     version: v1

# 获取所有违规
kubectl get constraints -o json | jq '.items[].status.violations[]'

# 按命名空间统计违规
kubectl get constraints -o json | \
  jq -r '.items[].status.violations[] | .namespace' | \
  sort | uniq -c | sort -rn
```

### 违规报告

```yaml
# 使用 CronJob 生成违规报告
apiVersion: batch/v1
kind: CronJob
metadata:
  name: gatekeeper-violation-report
  namespace: gatekeeper-system
spec:
  schedule: "0 8 * * *"  # 每天 8 点
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: gatekeeper-reporter
          containers:
            - name: reporter
              image: bitnami/kubectl:latest
              command:
                - /bin/bash
                - -c
                - |
                  # 收集违规信息
                  echo "=== Gatekeeper Violation Report ===" > /tmp/report.txt
                  echo "Generated: $(date)" >> /tmp/report.txt
                  echo "" >> /tmp/report.txt

                  # 按 Constraint 统计
                  echo "=== Violations by Constraint ===" >> /tmp/report.txt
                  for constraint in $(kubectl get constraints -o jsonpath='{.items[*].metadata.name}'); do
                    count=$(kubectl get constraints $constraint -o jsonpath='{.status.totalViolations}')
                    echo "$constraint: $count violations" >> /tmp/report.txt
                  done
                  echo "" >> /tmp/report.txt

                  # 详细违规列表
                  echo "=== Detailed Violations ===" >> /tmp/report.txt
                  kubectl get constraints -o json | \
                    jq -r '.items[] | select(.status.totalViolations > 0) |
                      "Constraint: \(.metadata.name)\n" +
                      (.status.violations[] | "  - \(.namespace)/\(.name): \(.message)\n")' \
                    >> /tmp/report.txt

                  # 发送报告 (示例使用 Slack)
                  cat /tmp/report.txt
                  # curl -X POST -H 'Content-type: application/json' \
                  #   --data "{\"text\": \"$(cat /tmp/report.txt)\"}" \
                  #   $SLACK_WEBHOOK_URL
          restartPolicy: OnFailure
---
# RBAC
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gatekeeper-reporter
  namespace: gatekeeper-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gatekeeper-reporter
rules:
  - apiGroups: ["constraints.gatekeeper.sh"]
    resources: ["*"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gatekeeper-reporter
subjects:
  - kind: ServiceAccount
    name: gatekeeper-reporter
    namespace: gatekeeper-system
roleRef:
  kind: ClusterRole
  name: gatekeeper-reporter
  apiGroup: rbac.authorization.k8s.io
```

### Prometheus 监控

```yaml
# Gatekeeper 指标
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: gatekeeper
  namespace: gatekeeper-system
spec:
  selector:
    matchLabels:
      gatekeeper.sh/system: "yes"
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
---
# 告警规则
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: gatekeeper-alerts
  namespace: gatekeeper-system
spec:
  groups:
    - name: gatekeeper.rules
      rules:
        # Webhook 请求延迟
        - alert: GatekeeperWebhookLatencyHigh
          expr: |
            histogram_quantile(0.99,
              sum(rate(gatekeeper_validation_request_duration_seconds_bucket[5m])) by (le)
            ) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Gatekeeper webhook latency is high"
            description: "99th percentile latency is {{ $value }}s"

        # 违规数量增加
        - alert: GatekeeperViolationsIncreasing
          expr: |
            increase(gatekeeper_violations[1h]) > 10
          for: 0m
          labels:
            severity: warning
          annotations:
            summary: "Gatekeeper violations increasing"
            description: "{{ $value }} new violations in the last hour"

        # Webhook 可用性
        - alert: GatekeeperWebhookUnavailable
          expr: |
            sum(up{job="gatekeeper-webhook"}) < 1
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Gatekeeper webhook is unavailable"

        # 审计延迟
        - alert: GatekeeperAuditDurationHigh
          expr: |
            gatekeeper_audit_duration_seconds > 300
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Gatekeeper audit taking too long"
            description: "Audit duration is {{ $value }}s"
```

## 最佳实践

### 策略分类

```
┌─────────────────────────────────────────────────────────────────┐
│                      策略分类与实施建议                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    安全策略 (Critical)                   │   │
│   │                                                          │   │
│   │   • 禁止特权容器                enforcementAction: deny  │   │
│   │   • 禁止 hostPID/hostNetwork    enforcementAction: deny  │   │
│   │   • 镜像来源限制                enforcementAction: deny  │   │
│   │   • 禁止 root 用户              enforcementAction: deny  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    合规策略 (High)                       │   │
│   │                                                          │   │
│   │   • 必需标签                    enforcementAction: warn  │   │
│   │   • 资源限制                    enforcementAction: warn  │   │
│   │   • 副本数要求                  enforcementAction: warn  │   │
│   │   • PDB 要求                    enforcementAction: warn  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    最佳实践 (Medium)                     │   │
│   │                                                          │   │
│   │   • 镜像版本标签                enforcementAction: warn  │   │
│   │   • 健康检查配置                enforcementAction: dryrun│   │
│   │   • 资源命名规范                enforcementAction: dryrun│   │
│   │   • 注解完整性                  enforcementAction: dryrun│   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 渐进式部署

```yaml
# 阶段 1: 审计模式 - 只记录不阻止
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos-audit
spec:
  enforcementAction: dryrun  # 只审计
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
---
# 阶段 2: 警告模式 - 警告但不阻止
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos-warn
spec:
  enforcementAction: warn  # 警告
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
---
# 阶段 3: 强制模式 - 阻止违规
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRepos
metadata:
  name: allowed-repos-enforce
spec:
  enforcementAction: deny  # 强制
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

### 性能优化

```yaml
# 优化 Gatekeeper 配置
apiVersion: config.gatekeeper.sh/v1alpha1
kind: Config
metadata:
  name: config
  namespace: gatekeeper-system
spec:
  # 只同步必要的资源
  sync:
    syncOnly:
      - group: ""
        version: "v1"
        kind: "Namespace"
      - group: ""
        version: "v1"
        kind: "Pod"
  # 排除系统命名空间
  match:
    - excludedNamespaces:
        - kube-system
        - kube-public
        - gatekeeper-system
      processes:
        - "*"
  # 启用审计缓存
  readiness:
    statsEnabled: true
---
# 调整资源配置
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gatekeeper-controller-manager
  namespace: gatekeeper-system
spec:
  replicas: 3  # 高可用
  template:
    spec:
      containers:
        - name: manager
          resources:
            limits:
              cpu: "2"
              memory: 1Gi
            requests:
              cpu: 500m
              memory: 512Mi
          args:
            - --port=8443
            - --exempt-namespace=gatekeeper-system
            - --exempt-namespace=kube-system
            - --operation=webhook
            - --log-level=WARNING  # 减少日志
            - --disable-opa-builtin={http.send}  # 禁用不需要的内置函数
```

## 总结

OPA/Gatekeeper 策略引擎的核心价值：

| 特性 | 说明 |
|------|------|
| 策略即代码 | Rego 语言声明式策略，可版本控制 |
| 统一策略 | 跨平台、跨服务的统一策略引擎 |
| 准入控制 | 与 Kubernetes 原生准入机制集成 |
| 审计能力 | 持续审计现有资源的合规性 |
| 可扩展性 | 通过 ConstraintTemplate 自定义策略 |

关键实践要点：

1. **渐进部署**：从 dryrun → warn → deny 逐步实施
2. **策略分类**：按重要性分类，区分强制和建议策略
3. **性能优化**：限制同步资源、排除系统命名空间
4. **持续监控**：配置告警、定期审计违规报告
5. **版本管理**：策略代码纳入 Git 版本控制
