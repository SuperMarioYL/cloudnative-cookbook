---
title: "DynamicClient 详解"
weight: 4
---
## 概述

DynamicClient 是 client-go 提供的非类型化客户端，可以访问任意 Kubernetes 资源，包括 CRD（Custom Resource Definition）。与 Clientset 不同，DynamicClient 不需要预先生成的类型代码，使用 `unstructured.Unstructured` 作为通用数据结构。

## DynamicClient 结构

### 接口定义

```go
// staging/src/k8s.io/client-go/dynamic/interface.go

type Interface interface {
    Resource(resource schema.GroupVersionResource) NamespaceableResourceInterface
}

type NamespaceableResourceInterface interface {
    // 命名空间级操作
    Namespace(string) ResourceInterface
    // 集群级操作
    ResourceInterface
}

type ResourceInterface interface {
    Create(ctx context.Context, obj *unstructured.Unstructured, options metav1.CreateOptions, subresources ...string) (*unstructured.Unstructured, error)
    Update(ctx context.Context, obj *unstructured.Unstructured, options metav1.UpdateOptions, subresources ...string) (*unstructured.Unstructured, error)
    UpdateStatus(ctx context.Context, obj *unstructured.Unstructured, options metav1.UpdateOptions) (*unstructured.Unstructured, error)
    Delete(ctx context.Context, name string, options metav1.DeleteOptions, subresources ...string) error
    DeleteCollection(ctx context.Context, options metav1.DeleteOptions, listOptions metav1.ListOptions) error
    Get(ctx context.Context, name string, options metav1.GetOptions, subresources ...string) (*unstructured.Unstructured, error)
    List(ctx context.Context, opts metav1.ListOptions) (*unstructured.UnstructuredList, error)
    Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
    Patch(ctx context.Context, name string, pt types.PatchType, data []byte, options metav1.PatchOptions, subresources ...string) (*unstructured.Unstructured, error)
    Apply(ctx context.Context, name string, obj *unstructured.Unstructured, options metav1.ApplyOptions, subresources ...string) (*unstructured.Unstructured, error)
    ApplyStatus(ctx context.Context, name string, obj *unstructured.Unstructured, options metav1.ApplyOptions) (*unstructured.Unstructured, error)
}
```

### Unstructured 类型

```go
// staging/src/k8s.io/apimachinery/pkg/apis/meta/v1/unstructured/unstructured.go

type Unstructured struct {
    // Object 是底层数据结构
    Object map[string]interface{}
}

// 访问方法
func (u *Unstructured) GetAPIVersion() string
func (u *Unstructured) SetAPIVersion(version string)
func (u *Unstructured) GetKind() string
func (u *Unstructured) SetKind(kind string)
func (u *Unstructured) GetNamespace() string
func (u *Unstructured) SetNamespace(namespace string)
func (u *Unstructured) GetName() string
func (u *Unstructured) SetName(name string)
func (u *Unstructured) GetLabels() map[string]string
func (u *Unstructured) SetLabels(labels map[string]string)
func (u *Unstructured) GetAnnotations() map[string]string
func (u *Unstructured) SetAnnotations(annotations map[string]string)
func (u *Unstructured) GetResourceVersion() string
func (u *Unstructured) SetResourceVersion(version string)
func (u *Unstructured) GetUID() types.UID
// ...
```

## 创建 DynamicClient

```go
import (
    "k8s.io/client-go/dynamic"
    "k8s.io/client-go/rest"
    "k8s.io/client-go/tools/clientcmd"
)

func newDynamicClient() (dynamic.Interface, error) {
    // 从 kubeconfig 加载配置
    config, err := clientcmd.BuildConfigFromFlags("", os.Getenv("HOME")+"/.kube/config")
    if err != nil {
        // 尝试 in-cluster 配置
        config, err = rest.InClusterConfig()
        if err != nil {
            return nil, err
        }
    }

    return dynamic.NewForConfig(config)
}
```

## GVR 定义

### 常用 GVR

```go
import "k8s.io/apimachinery/pkg/runtime/schema"

// 核心资源 GVR
var (
    podsGVR = schema.GroupVersionResource{
        Group:    "",
        Version:  "v1",
        Resource: "pods",
    }

    servicesGVR = schema.GroupVersionResource{
        Group:    "",
        Version:  "v1",
        Resource: "services",
    }

    configmapsGVR = schema.GroupVersionResource{
        Group:    "",
        Version:  "v1",
        Resource: "configmaps",
    }

    secretsGVR = schema.GroupVersionResource{
        Group:    "",
        Version:  "v1",
        Resource: "secrets",
    }
)

// apps 组资源 GVR
var (
    deploymentsGVR = schema.GroupVersionResource{
        Group:    "apps",
        Version:  "v1",
        Resource: "deployments",
    }

    statefulsetsGVR = schema.GroupVersionResource{
        Group:    "apps",
        Version:  "v1",
        Resource: "statefulsets",
    }

    daemonsetsGVR = schema.GroupVersionResource{
        Group:    "apps",
        Version:  "v1",
        Resource: "daemonsets",
    }
)

// CRD 资源 GVR 示例
var (
    myResourceGVR = schema.GroupVersionResource{
        Group:    "mygroup.example.com",
        Version:  "v1",
        Resource: "myresources",
    }
)
```

### 动态获取 GVR

```go
import (
    "k8s.io/client-go/discovery"
    "k8s.io/client-go/restmapper"
)

func getGVR(discoveryClient discovery.DiscoveryInterface, gvk schema.GroupVersionKind) (schema.GroupVersionResource, error) {
    // 获取 API 资源映射
    apiGroupResources, err := restmapper.GetAPIGroupResources(discoveryClient)
    if err != nil {
        return schema.GroupVersionResource{}, err
    }

    mapper := restmapper.NewDiscoveryRESTMapper(apiGroupResources)

    // 从 GVK 获取 GVR
    mapping, err := mapper.RESTMapping(gvk.GroupKind(), gvk.Version)
    if err != nil {
        return schema.GroupVersionResource{}, err
    }

    return mapping.Resource, nil
}

// 使用示例
gvk := schema.GroupVersionKind{
    Group:   "apps",
    Version: "v1",
    Kind:    "Deployment",
}
gvr, err := getGVR(discoveryClient, gvk)
// gvr: apps/v1/deployments
```

## CRUD 操作

### Create

```go
func createResource(client dynamic.Interface, gvr schema.GroupVersionResource, namespace string, obj *unstructured.Unstructured) (*unstructured.Unstructured, error) {
    if namespace != "" {
        return client.Resource(gvr).Namespace(namespace).Create(
            context.TODO(),
            obj,
            metav1.CreateOptions{},
        )
    }
    return client.Resource(gvr).Create(
        context.TODO(),
        obj,
        metav1.CreateOptions{},
    )
}

// 创建 Deployment 示例
func createDeployment(client dynamic.Interface) (*unstructured.Unstructured, error) {
    deployment := &unstructured.Unstructured{
        Object: map[string]interface{}{
            "apiVersion": "apps/v1",
            "kind":       "Deployment",
            "metadata": map[string]interface{}{
                "name":      "nginx-deployment",
                "namespace": "default",
            },
            "spec": map[string]interface{}{
                "replicas": int64(3),
                "selector": map[string]interface{}{
                    "matchLabels": map[string]interface{}{
                        "app": "nginx",
                    },
                },
                "template": map[string]interface{}{
                    "metadata": map[string]interface{}{
                        "labels": map[string]interface{}{
                            "app": "nginx",
                        },
                    },
                    "spec": map[string]interface{}{
                        "containers": []interface{}{
                            map[string]interface{}{
                                "name":  "nginx",
                                "image": "nginx:1.21",
                                "ports": []interface{}{
                                    map[string]interface{}{
                                        "containerPort": int64(80),
                                    },
                                },
                            },
                        },
                    },
                },
            },
        },
    }

    return client.Resource(deploymentsGVR).Namespace("default").Create(
        context.TODO(),
        deployment,
        metav1.CreateOptions{},
    )
}
```

### Get

```go
func getResource(client dynamic.Interface, gvr schema.GroupVersionResource, namespace, name string) (*unstructured.Unstructured, error) {
    if namespace != "" {
        return client.Resource(gvr).Namespace(namespace).Get(
            context.TODO(),
            name,
            metav1.GetOptions{},
        )
    }
    return client.Resource(gvr).Get(
        context.TODO(),
        name,
        metav1.GetOptions{},
    )
}

// 使用示例
deployment, err := getResource(client, deploymentsGVR, "default", "nginx-deployment")
if err != nil {
    return err
}

// 访问字段
name, found, err := unstructured.NestedString(deployment.Object, "metadata", "name")
replicas, found, err := unstructured.NestedInt64(deployment.Object, "spec", "replicas")
```

### List

```go
func listResources(client dynamic.Interface, gvr schema.GroupVersionResource, namespace string, labelSelector string) (*unstructured.UnstructuredList, error) {
    opts := metav1.ListOptions{}
    if labelSelector != "" {
        opts.LabelSelector = labelSelector
    }

    if namespace != "" {
        return client.Resource(gvr).Namespace(namespace).List(context.TODO(), opts)
    }
    return client.Resource(gvr).List(context.TODO(), opts)
}

// 使用示例
pods, err := listResources(client, podsGVR, "default", "app=nginx")
if err != nil {
    return err
}

for _, pod := range pods.Items {
    name := pod.GetName()
    namespace := pod.GetNamespace()
    fmt.Printf("Pod: %s/%s\n", namespace, name)
}
```

### Update

```go
func updateResource(client dynamic.Interface, gvr schema.GroupVersionResource, namespace string, obj *unstructured.Unstructured) (*unstructured.Unstructured, error) {
    if namespace != "" {
        return client.Resource(gvr).Namespace(namespace).Update(
            context.TODO(),
            obj,
            metav1.UpdateOptions{},
        )
    }
    return client.Resource(gvr).Update(
        context.TODO(),
        obj,
        metav1.UpdateOptions{},
    )
}

// 更新带重试
func updateResourceWithRetry(client dynamic.Interface, gvr schema.GroupVersionResource, namespace, name string, updateFn func(*unstructured.Unstructured) error) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        // 获取最新版本
        obj, err := client.Resource(gvr).Namespace(namespace).Get(
            context.TODO(),
            name,
            metav1.GetOptions{},
        )
        if err != nil {
            return err
        }

        // 应用修改
        if err := updateFn(obj); err != nil {
            return err
        }

        // 提交更新
        _, err = client.Resource(gvr).Namespace(namespace).Update(
            context.TODO(),
            obj,
            metav1.UpdateOptions{},
        )
        return err
    })
}

// 使用示例
err := updateResourceWithRetry(client, deploymentsGVR, "default", "nginx-deployment", func(obj *unstructured.Unstructured) error {
    return unstructured.SetNestedField(obj.Object, int64(5), "spec", "replicas")
})
```

### Patch

```go
func patchResource(client dynamic.Interface, gvr schema.GroupVersionResource, namespace, name string, patchType types.PatchType, patchData []byte) (*unstructured.Unstructured, error) {
    if namespace != "" {
        return client.Resource(gvr).Namespace(namespace).Patch(
            context.TODO(),
            name,
            patchType,
            patchData,
            metav1.PatchOptions{},
        )
    }
    return client.Resource(gvr).Patch(
        context.TODO(),
        name,
        patchType,
        patchData,
        metav1.PatchOptions{},
    )
}

// JSON Patch 示例
func patchDeploymentReplicas(client dynamic.Interface, namespace, name string, replicas int64) (*unstructured.Unstructured, error) {
    patchData := []byte(fmt.Sprintf(`[{"op": "replace", "path": "/spec/replicas", "value": %d}]`, replicas))
    return patchResource(client, deploymentsGVR, namespace, name, types.JSONPatchType, patchData)
}

// Merge Patch 示例
func patchResourceLabels(client dynamic.Interface, gvr schema.GroupVersionResource, namespace, name string, labels map[string]string) (*unstructured.Unstructured, error) {
    patch := map[string]interface{}{
        "metadata": map[string]interface{}{
            "labels": labels,
        },
    }
    patchData, _ := json.Marshal(patch)
    return patchResource(client, gvr, namespace, name, types.MergePatchType, patchData)
}
```

### Delete

```go
func deleteResource(client dynamic.Interface, gvr schema.GroupVersionResource, namespace, name string) error {
    if namespace != "" {
        return client.Resource(gvr).Namespace(namespace).Delete(
            context.TODO(),
            name,
            metav1.DeleteOptions{},
        )
    }
    return client.Resource(gvr).Delete(
        context.TODO(),
        name,
        metav1.DeleteOptions{},
    )
}

// 带传播策略的删除
func deleteResourceWithPropagation(client dynamic.Interface, gvr schema.GroupVersionResource, namespace, name string, policy metav1.DeletionPropagation) error {
    opts := metav1.DeleteOptions{
        PropagationPolicy: &policy,
    }

    if namespace != "" {
        return client.Resource(gvr).Namespace(namespace).Delete(context.TODO(), name, opts)
    }
    return client.Resource(gvr).Delete(context.TODO(), name, opts)
}
```

## CRD 操作

### 定义 CRD GVR

```go
// 假设有一个 CRD: myresources.mygroup.example.com
var myResourceGVR = schema.GroupVersionResource{
    Group:    "mygroup.example.com",
    Version:  "v1",
    Resource: "myresources",
}

// 创建 CRD 实例
func createMyResource(client dynamic.Interface) (*unstructured.Unstructured, error) {
    myResource := &unstructured.Unstructured{
        Object: map[string]interface{}{
            "apiVersion": "mygroup.example.com/v1",
            "kind":       "MyResource",
            "metadata": map[string]interface{}{
                "name":      "my-resource-1",
                "namespace": "default",
            },
            "spec": map[string]interface{}{
                "field1": "value1",
                "field2": int64(42),
                "nested": map[string]interface{}{
                    "key": "value",
                },
            },
        },
    }

    return client.Resource(myResourceGVR).Namespace("default").Create(
        context.TODO(),
        myResource,
        metav1.CreateOptions{},
    )
}
```

### Watch CRD

```go
func watchMyResources(client dynamic.Interface, namespace string) error {
    watcher, err := client.Resource(myResourceGVR).Namespace(namespace).Watch(
        context.TODO(),
        metav1.ListOptions{},
    )
    if err != nil {
        return err
    }
    defer watcher.Stop()

    for event := range watcher.ResultChan() {
        obj, ok := event.Object.(*unstructured.Unstructured)
        if !ok {
            continue
        }

        switch event.Type {
        case watch.Added:
            fmt.Printf("MyResource Added: %s\n", obj.GetName())
        case watch.Modified:
            fmt.Printf("MyResource Modified: %s\n", obj.GetName())
        case watch.Deleted:
            fmt.Printf("MyResource Deleted: %s\n", obj.GetName())
        }
    }

    return nil
}
```

## 类型转换

### Unstructured 到类型化对象

```go
import "k8s.io/apimachinery/pkg/runtime"

func unstructuredToDeployment(obj *unstructured.Unstructured) (*appsv1.Deployment, error) {
    deployment := &appsv1.Deployment{}
    err := runtime.DefaultUnstructuredConverter.FromUnstructured(obj.Object, deployment)
    return deployment, err
}

// 使用示例
obj, err := client.Resource(deploymentsGVR).Namespace("default").Get(
    context.TODO(),
    "nginx-deployment",
    metav1.GetOptions{},
)
if err != nil {
    return err
}

deployment, err := unstructuredToDeployment(obj)
if err != nil {
    return err
}

fmt.Printf("Replicas: %d\n", *deployment.Spec.Replicas)
```

### 类型化对象到 Unstructured

```go
func deploymentToUnstructured(deployment *appsv1.Deployment) (*unstructured.Unstructured, error) {
    obj, err := runtime.DefaultUnstructuredConverter.ToUnstructured(deployment)
    if err != nil {
        return nil, err
    }
    return &unstructured.Unstructured{Object: obj}, nil
}
```

## 嵌套字段访问

### 读取嵌套字段

```go
// 读取字符串
name, found, err := unstructured.NestedString(obj.Object, "metadata", "name")

// 读取整数
replicas, found, err := unstructured.NestedInt64(obj.Object, "spec", "replicas")

// 读取布尔值
paused, found, err := unstructured.NestedBool(obj.Object, "spec", "paused")

// 读取字符串 Map
labels, found, err := unstructured.NestedStringMap(obj.Object, "metadata", "labels")

// 读取切片
containers, found, err := unstructured.NestedSlice(obj.Object, "spec", "template", "spec", "containers")

// 读取嵌套 Map
spec, found, err := unstructured.NestedMap(obj.Object, "spec")
```

### 设置嵌套字段

```go
// 设置字符串
err := unstructured.SetNestedField(obj.Object, "new-value", "metadata", "name")

// 设置整数
err := unstructured.SetNestedField(obj.Object, int64(5), "spec", "replicas")

// 设置字符串 Map
err := unstructured.SetNestedStringMap(obj.Object, map[string]string{"app": "nginx"}, "metadata", "labels")

// 设置切片
containers := []interface{}{
    map[string]interface{}{
        "name":  "nginx",
        "image": "nginx:1.21",
    },
}
err := unstructured.SetNestedSlice(obj.Object, containers, "spec", "template", "spec", "containers")
```

## Server-Side Apply

```go
func applyResource(client dynamic.Interface, gvr schema.GroupVersionResource, namespace string, obj *unstructured.Unstructured, fieldManager string) (*unstructured.Unstructured, error) {
    if namespace != "" {
        return client.Resource(gvr).Namespace(namespace).Apply(
            context.TODO(),
            obj.GetName(),
            obj,
            metav1.ApplyOptions{
                FieldManager: fieldManager,
                Force:        true,
            },
        )
    }
    return client.Resource(gvr).Apply(
        context.TODO(),
        obj.GetName(),
        obj,
        metav1.ApplyOptions{
            FieldManager: fieldManager,
            Force:        true,
        },
    )
}
```

## 与 Clientset 对比

```
DynamicClient vs Clientset:

DynamicClient:
├── 优点:
│   ├── 无需预生成代码
│   ├── 支持任意资源（包括 CRD）
│   └── 灵活的运行时发现
├── 缺点:
│   ├── 无编译时类型检查
│   ├── 字段访问繁琐
│   └── 性能略低

Clientset:
├── 优点:
│   ├── 类型安全
│   ├── IDE 自动补全
│   └── 编译时错误检查
├── 缺点:
│   ├── 需要预生成代码
│   ├── 不支持未知 CRD
│   └── 版本升级需重新生成

选择建议:
- 内置资源 → Clientset
- CRD 资源 → DynamicClient
- 通用工具 → DynamicClient
- 性能敏感 → Clientset
```

## 总结

DynamicClient 核心要点：

**适用场景**
- CRD 资源访问
- 通用 Kubernetes 工具
- 运行时资源发现

**GVR 定义**
- 手动定义常用 GVR
- 使用 DiscoveryClient 动态获取

**类型转换**
- Unstructured 嵌套字段访问
- 与类型化对象相互转换

**最佳实践**
- 使用 NestedXxx 函数访问字段
- 处理 found 返回值
- 考虑性能影响
