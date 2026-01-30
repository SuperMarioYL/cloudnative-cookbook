---
title: "Clientset 详解"
weight: 3
---
## 概述

Clientset 是 client-go 中最常用的客户端类型，为 Kubernetes 内置资源提供类型安全的访问接口。每个 API 组和版本都有对应的客户端接口，使用起来直观且有编译时类型检查。

## Clientset 结构

### 接口定义

```go
// staging/src/k8s.io/client-go/kubernetes/clientset.go

type Interface interface {
    Discovery() discovery.DiscoveryInterface

    // 核心 API 组
    CoreV1() corev1.CoreV1Interface

    // apps API 组
    AppsV1() appsv1.AppsV1Interface

    // batch API 组
    BatchV1() batchv1.BatchV1Interface

    // networking API 组
    NetworkingV1() networkingv1.NetworkingV1Interface

    // rbac API 组
    RbacV1() rbacv1.RbacV1Interface

    // storage API 组
    StorageV1() storagev1.StorageV1Interface

    // ... 更多 API 组
}

// Clientset 实现
type Clientset struct {
    *discovery.DiscoveryClient
    coreV1       *corev1.CoreV1Client
    appsV1       *appsv1.AppsV1Client
    batchV1      *batchv1.BatchV1Client
    // ... 更多客户端
}
```

### 资源客户端接口

```go
// staging/src/k8s.io/client-go/kubernetes/typed/core/v1/core_client.go

type CoreV1Interface interface {
    RESTClient() rest.Interface
    ConfigMapsGetter
    EndpointsGetter
    EventsGetter
    NamespacesGetter
    NodesGetter
    PersistentVolumesGetter
    PersistentVolumeClaimsGetter
    PodsGetter
    SecretsGetter
    ServicesGetter
    ServiceAccountsGetter
    // ...
}

// PodsGetter 接口
type PodsGetter interface {
    Pods(namespace string) PodInterface
}

// PodInterface 接口
type PodInterface interface {
    Create(ctx context.Context, pod *v1.Pod, opts metav1.CreateOptions) (*v1.Pod, error)
    Update(ctx context.Context, pod *v1.Pod, opts metav1.UpdateOptions) (*v1.Pod, error)
    UpdateStatus(ctx context.Context, pod *v1.Pod, opts metav1.UpdateOptions) (*v1.Pod, error)
    Delete(ctx context.Context, name string, opts metav1.DeleteOptions) error
    DeleteCollection(ctx context.Context, opts metav1.DeleteOptions, listOpts metav1.ListOptions) error
    Get(ctx context.Context, name string, opts metav1.GetOptions) (*v1.Pod, error)
    List(ctx context.Context, opts metav1.ListOptions) (*v1.PodList, error)
    Watch(ctx context.Context, opts metav1.ListOptions) (watch.Interface, error)
    Patch(ctx context.Context, name string, pt types.PatchType, data []byte, opts metav1.PatchOptions, subresources ...string) (*v1.Pod, error)
    // 子资源
    GetLogs(name string, opts *v1.PodLogOptions) *rest.Request
    Evict(ctx context.Context, eviction *policyv1.Eviction) error
}
```

## 创建 Clientset

### 从 kubeconfig 创建

```go
import (
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/tools/clientcmd"
)

func newClientset() (*kubernetes.Clientset, error) {
    // 从默认位置加载 kubeconfig
    kubeconfig := filepath.Join(os.Getenv("HOME"), ".kube", "config")

    config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
    if err != nil {
        return nil, err
    }

    // 可选：调整配置
    config.QPS = 100
    config.Burst = 200

    return kubernetes.NewForConfig(config)
}
```

### 从集群内创建

```go
import (
    "k8s.io/client-go/kubernetes"
    "k8s.io/client-go/rest"
)

func newInClusterClientset() (*kubernetes.Clientset, error) {
    config, err := rest.InClusterConfig()
    if err != nil {
        return nil, err
    }

    return kubernetes.NewForConfig(config)
}
```

### 通用创建函数

```go
func getClientset() (*kubernetes.Clientset, error) {
    var config *rest.Config
    var err error

    // 优先尝试 in-cluster 配置
    config, err = rest.InClusterConfig()
    if err != nil {
        // 回退到 kubeconfig
        kubeconfig := os.Getenv("KUBECONFIG")
        if kubeconfig == "" {
            kubeconfig = filepath.Join(os.Getenv("HOME"), ".kube", "config")
        }
        config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
        if err != nil {
            return nil, err
        }
    }

    return kubernetes.NewForConfig(config)
}
```

## CRUD 操作

### Create

```go
func createDeployment(clientset *kubernetes.Clientset) (*appsv1.Deployment, error) {
    deployment := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "nginx-deployment",
            Namespace: "default",
            Labels: map[string]string{
                "app": "nginx",
            },
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: ptr.To(int32(3)),
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{
                    "app": "nginx",
                },
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{
                        "app": "nginx",
                    },
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{
                        {
                            Name:  "nginx",
                            Image: "nginx:1.21",
                            Ports: []corev1.ContainerPort{
                                {ContainerPort: 80},
                            },
                        },
                    },
                },
            },
        },
    }

    return clientset.AppsV1().Deployments("default").Create(
        context.TODO(),
        deployment,
        metav1.CreateOptions{},
    )
}
```

### Get

```go
func getDeployment(clientset *kubernetes.Clientset, namespace, name string) (*appsv1.Deployment, error) {
    return clientset.AppsV1().Deployments(namespace).Get(
        context.TODO(),
        name,
        metav1.GetOptions{},
    )
}

func getPod(clientset *kubernetes.Clientset, namespace, name string) (*corev1.Pod, error) {
    return clientset.CoreV1().Pods(namespace).Get(
        context.TODO(),
        name,
        metav1.GetOptions{},
    )
}
```

### List

```go
func listPods(clientset *kubernetes.Clientset, namespace string) (*corev1.PodList, error) {
    return clientset.CoreV1().Pods(namespace).List(
        context.TODO(),
        metav1.ListOptions{},
    )
}

// 带标签选择器
func listPodsByLabel(clientset *kubernetes.Clientset, namespace, labelSelector string) (*corev1.PodList, error) {
    return clientset.CoreV1().Pods(namespace).List(
        context.TODO(),
        metav1.ListOptions{
            LabelSelector: labelSelector, // "app=nginx,env=prod"
        },
    )
}

// 带字段选择器
func listPodsByField(clientset *kubernetes.Clientset, namespace, fieldSelector string) (*corev1.PodList, error) {
    return clientset.CoreV1().Pods(namespace).List(
        context.TODO(),
        metav1.ListOptions{
            FieldSelector: fieldSelector, // "status.phase=Running"
        },
    )
}

// 分页列表
func listPodsWithPagination(clientset *kubernetes.Clientset, namespace string, limit int64) ([]corev1.Pod, error) {
    var allPods []corev1.Pod
    continueToken := ""

    for {
        podList, err := clientset.CoreV1().Pods(namespace).List(
            context.TODO(),
            metav1.ListOptions{
                Limit:    limit,
                Continue: continueToken,
            },
        )
        if err != nil {
            return nil, err
        }

        allPods = append(allPods, podList.Items...)

        if podList.Continue == "" {
            break
        }
        continueToken = podList.Continue
    }

    return allPods, nil
}
```

### Update

```go
func updateDeployment(clientset *kubernetes.Clientset, deployment *appsv1.Deployment) (*appsv1.Deployment, error) {
    return clientset.AppsV1().Deployments(deployment.Namespace).Update(
        context.TODO(),
        deployment,
        metav1.UpdateOptions{},
    )
}

// 更新带重试
func updateDeploymentWithRetry(clientset *kubernetes.Clientset, namespace, name string, updateFn func(*appsv1.Deployment)) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        // 获取最新版本
        deployment, err := clientset.AppsV1().Deployments(namespace).Get(
            context.TODO(),
            name,
            metav1.GetOptions{},
        )
        if err != nil {
            return err
        }

        // 应用修改
        updateFn(deployment)

        // 提交更新
        _, err = clientset.AppsV1().Deployments(namespace).Update(
            context.TODO(),
            deployment,
            metav1.UpdateOptions{},
        )
        return err
    })
}

// 使用示例
err := updateDeploymentWithRetry(clientset, "default", "nginx", func(d *appsv1.Deployment) {
    d.Spec.Replicas = ptr.To(int32(5))
})
```

### Patch

```go
import "k8s.io/apimachinery/pkg/types"

// JSON Patch
func patchDeploymentReplicas(clientset *kubernetes.Clientset, namespace, name string, replicas int32) (*appsv1.Deployment, error) {
    patchData := []byte(fmt.Sprintf(`[{"op": "replace", "path": "/spec/replicas", "value": %d}]`, replicas))

    return clientset.AppsV1().Deployments(namespace).Patch(
        context.TODO(),
        name,
        types.JSONPatchType,
        patchData,
        metav1.PatchOptions{},
    )
}

// Strategic Merge Patch
func patchDeploymentLabels(clientset *kubernetes.Clientset, namespace, name string, labels map[string]string) (*appsv1.Deployment, error) {
    patch := map[string]interface{}{
        "metadata": map[string]interface{}{
            "labels": labels,
        },
    }
    patchData, _ := json.Marshal(patch)

    return clientset.AppsV1().Deployments(namespace).Patch(
        context.TODO(),
        name,
        types.StrategicMergePatchType,
        patchData,
        metav1.PatchOptions{},
    )
}

// Merge Patch
func patchPodAnnotations(clientset *kubernetes.Clientset, namespace, name string, annotations map[string]string) (*corev1.Pod, error) {
    patch := map[string]interface{}{
        "metadata": map[string]interface{}{
            "annotations": annotations,
        },
    }
    patchData, _ := json.Marshal(patch)

    return clientset.CoreV1().Pods(namespace).Patch(
        context.TODO(),
        name,
        types.MergePatchType,
        patchData,
        metav1.PatchOptions{},
    )
}
```

### Delete

```go
func deleteDeployment(clientset *kubernetes.Clientset, namespace, name string) error {
    return clientset.AppsV1().Deployments(namespace).Delete(
        context.TODO(),
        name,
        metav1.DeleteOptions{},
    )
}

// 带传播策略的删除
func deleteDeploymentWithPropagation(clientset *kubernetes.Clientset, namespace, name string) error {
    propagationPolicy := metav1.DeletePropagationForeground
    return clientset.AppsV1().Deployments(namespace).Delete(
        context.TODO(),
        name,
        metav1.DeleteOptions{
            PropagationPolicy: &propagationPolicy,
        },
    )
}

// 删除集合
func deletePodsWithLabel(clientset *kubernetes.Clientset, namespace, labelSelector string) error {
    return clientset.CoreV1().Pods(namespace).DeleteCollection(
        context.TODO(),
        metav1.DeleteOptions{},
        metav1.ListOptions{
            LabelSelector: labelSelector,
        },
    )
}
```

## Watch 操作

### 基本 Watch

```go
func watchPods(clientset *kubernetes.Clientset, namespace string) error {
    watcher, err := clientset.CoreV1().Pods(namespace).Watch(
        context.TODO(),
        metav1.ListOptions{},
    )
    if err != nil {
        return err
    }
    defer watcher.Stop()

    for event := range watcher.ResultChan() {
        pod, ok := event.Object.(*corev1.Pod)
        if !ok {
            continue
        }

        switch event.Type {
        case watch.Added:
            fmt.Printf("Pod Added: %s/%s\n", pod.Namespace, pod.Name)
        case watch.Modified:
            fmt.Printf("Pod Modified: %s/%s Phase: %s\n", pod.Namespace, pod.Name, pod.Status.Phase)
        case watch.Deleted:
            fmt.Printf("Pod Deleted: %s/%s\n", pod.Namespace, pod.Name)
        case watch.Error:
            fmt.Printf("Watch Error\n")
        }
    }

    return nil
}
```

### 带超时的 Watch

```go
func watchPodsWithTimeout(clientset *kubernetes.Clientset, namespace string, timeout time.Duration) error {
    timeoutSeconds := int64(timeout.Seconds())

    watcher, err := clientset.CoreV1().Pods(namespace).Watch(
        context.TODO(),
        metav1.ListOptions{
            TimeoutSeconds: &timeoutSeconds,
        },
    )
    if err != nil {
        return err
    }
    defer watcher.Stop()

    for event := range watcher.ResultChan() {
        // 处理事件...
        _ = event
    }

    return nil
}
```

## 子资源操作

### Status 子资源

```go
func updatePodStatus(clientset *kubernetes.Clientset, pod *corev1.Pod) (*corev1.Pod, error) {
    return clientset.CoreV1().Pods(pod.Namespace).UpdateStatus(
        context.TODO(),
        pod,
        metav1.UpdateOptions{},
    )
}

func updateDeploymentStatus(clientset *kubernetes.Clientset, deployment *appsv1.Deployment) (*appsv1.Deployment, error) {
    return clientset.AppsV1().Deployments(deployment.Namespace).UpdateStatus(
        context.TODO(),
        deployment,
        metav1.UpdateOptions{},
    )
}
```

### Scale 子资源

```go
func scaleDeployment(clientset *kubernetes.Clientset, namespace, name string, replicas int32) (*autoscalingv1.Scale, error) {
    // 获取当前 scale
    scale, err := clientset.AppsV1().Deployments(namespace).GetScale(
        context.TODO(),
        name,
        metav1.GetOptions{},
    )
    if err != nil {
        return nil, err
    }

    // 更新副本数
    scale.Spec.Replicas = replicas

    return clientset.AppsV1().Deployments(namespace).UpdateScale(
        context.TODO(),
        name,
        scale,
        metav1.UpdateOptions{},
    )
}
```

### Logs 子资源

```go
func getPodLogs(clientset *kubernetes.Clientset, namespace, podName, containerName string) (string, error) {
    req := clientset.CoreV1().Pods(namespace).GetLogs(podName, &corev1.PodLogOptions{
        Container:  containerName,
        Follow:     false,
        TailLines:  ptr.To(int64(100)),
        Timestamps: true,
    })

    stream, err := req.Stream(context.TODO())
    if err != nil {
        return "", err
    }
    defer stream.Close()

    buf := new(bytes.Buffer)
    _, err = io.Copy(buf, stream)
    return buf.String(), err
}

// 流式日志
func streamPodLogs(clientset *kubernetes.Clientset, namespace, podName, containerName string) error {
    req := clientset.CoreV1().Pods(namespace).GetLogs(podName, &corev1.PodLogOptions{
        Container: containerName,
        Follow:    true,
    })

    stream, err := req.Stream(context.TODO())
    if err != nil {
        return err
    }
    defer stream.Close()

    _, err = io.Copy(os.Stdout, stream)
    return err
}
```

### Exec 子资源

```go
import (
    "k8s.io/client-go/tools/remotecommand"
)

func execInPod(clientset *kubernetes.Clientset, config *rest.Config, namespace, podName, containerName string, command []string) error {
    req := clientset.CoreV1().RESTClient().Post().
        Resource("pods").
        Name(podName).
        Namespace(namespace).
        SubResource("exec").
        VersionedParams(&corev1.PodExecOptions{
            Container: containerName,
            Command:   command,
            Stdin:     true,
            Stdout:    true,
            Stderr:    true,
            TTY:       false,
        }, scheme.ParameterCodec)

    exec, err := remotecommand.NewSPDYExecutor(config, "POST", req.URL())
    if err != nil {
        return err
    }

    return exec.StreamWithContext(context.TODO(), remotecommand.StreamOptions{
        Stdin:  os.Stdin,
        Stdout: os.Stdout,
        Stderr: os.Stderr,
    })
}
```

### Eviction 子资源

```go
func evictPod(clientset *kubernetes.Clientset, namespace, name string) error {
    eviction := &policyv1.Eviction{
        ObjectMeta: metav1.ObjectMeta{
            Name:      name,
            Namespace: namespace,
        },
        DeleteOptions: &metav1.DeleteOptions{},
    }

    return clientset.PolicyV1().Evictions(namespace).Evict(context.TODO(), eviction)
}
```

## 常见模式

### 等待资源就绪

```go
func waitForPodReady(clientset *kubernetes.Clientset, namespace, name string, timeout time.Duration) error {
    ctx, cancel := context.WithTimeout(context.Background(), timeout)
    defer cancel()

    watcher, err := clientset.CoreV1().Pods(namespace).Watch(ctx, metav1.ListOptions{
        FieldSelector: fmt.Sprintf("metadata.name=%s", name),
    })
    if err != nil {
        return err
    }
    defer watcher.Stop()

    for event := range watcher.ResultChan() {
        if event.Type == watch.Error {
            return fmt.Errorf("watch error")
        }

        pod, ok := event.Object.(*corev1.Pod)
        if !ok {
            continue
        }

        if isPodReady(pod) {
            return nil
        }
    }

    return fmt.Errorf("timeout waiting for pod ready")
}

func isPodReady(pod *corev1.Pod) bool {
    for _, condition := range pod.Status.Conditions {
        if condition.Type == corev1.PodReady && condition.Status == corev1.ConditionTrue {
            return true
        }
    }
    return false
}
```

### 批量操作

```go
func createMultipleResources(clientset *kubernetes.Clientset, namespace string) error {
    // 创建 ConfigMap
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "app-config",
            Namespace: namespace,
        },
        Data: map[string]string{
            "key": "value",
        },
    }
    _, err := clientset.CoreV1().ConfigMaps(namespace).Create(context.TODO(), cm, metav1.CreateOptions{})
    if err != nil && !errors.IsAlreadyExists(err) {
        return err
    }

    // 创建 Secret
    secret := &corev1.Secret{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "app-secret",
            Namespace: namespace,
        },
        StringData: map[string]string{
            "password": "secret",
        },
    }
    _, err = clientset.CoreV1().Secrets(namespace).Create(context.TODO(), secret, metav1.CreateOptions{})
    if err != nil && !errors.IsAlreadyExists(err) {
        return err
    }

    // 创建 Deployment
    // ...

    return nil
}
```

## 总结

Clientset 核心要点：

**创建方式**
- NewForConfig 从配置创建
- InClusterConfig 用于 Pod 内
- BuildConfigFromFlags 用于外部

**CRUD 操作**
- 类型安全的 API 调用
- 支持标签和字段选择器
- 分页查询大量资源

**子资源**
- Status 更新
- Scale 扩缩容
- Logs 日志获取
- Exec 命令执行

**最佳实践**
- 使用重试处理冲突
- 适当设置超时
- 处理常见错误类型
