---
title: "RESTClient 详解"
weight: 2
---
## 概述

RESTClient 是 client-go 中最底层的客户端，直接封装了与 Kubernetes API Server 的 HTTP 交互。其他高层客户端（如 Clientset、DynamicClient）都是基于 RESTClient 构建的。理解 RESTClient 有助于深入理解 client-go 的工作原理。

## RESTClient 结构

### 核心组件

```go
// staging/src/k8s.io/client-go/rest/client.go

type RESTClient struct {
    // HTTP 客户端
    Client *http.Client

    // API 基础 URL
    base *url.URL

    // API 版本路径（如 /api 或 /apis）
    versionedAPIPath string

    // 内容配置
    content ClientContentConfig

    // 限速器
    rateLimiter flowcontrol.RateLimiter

    // 警告处理器
    warningHandler WarningHandler

    // 创建后端请求的函数
    createBackoffMgr func() BackoffManager
}
```

### 内容配置

```go
// ClientContentConfig 内容配置
type ClientContentConfig struct {
    // 接受的内容类型
    AcceptContentTypes string

    // 请求的内容类型
    ContentType string

    // 组版本
    GroupVersion schema.GroupVersion

    // 序列化器
    Negotiator runtime.ClientNegotiator
}
```

## 创建 RESTClient

### 从配置创建

```go
import (
    "k8s.io/client-go/rest"
    "k8s.io/client-go/kubernetes/scheme"
    corev1 "k8s.io/api/core/v1"
)

// 获取基础配置
config, err := rest.InClusterConfig()
if err != nil {
    // 从 kubeconfig 加载
    config, err = clientcmd.BuildConfigFromFlags("", os.Getenv("HOME")+"/.kube/config")
    if err != nil {
        panic(err)
    }
}

// 配置 RESTClient
config.APIPath = "/api"
config.GroupVersion = &corev1.SchemeGroupVersion
config.NegotiatedSerializer = scheme.Codecs.WithoutConversion()

// 创建 RESTClient
restClient, err := rest.RESTClientFor(config)
if err != nil {
    panic(err)
}
```

### 配置选项详解

```go
// 完整配置示例
config := &rest.Config{
    // API Server 地址
    Host: "https://kubernetes.default.svc",

    // TLS 配置
    TLSClientConfig: rest.TLSClientConfig{
        // CA 证书
        CAFile: "/var/run/secrets/kubernetes.io/serviceaccount/ca.crt",
        // 或直接提供数据
        CAData: caCertData,

        // 客户端证书
        CertFile: "/path/to/client.crt",
        CertData: clientCertData,

        // 客户端私钥
        KeyFile: "/path/to/client.key",
        KeyData: clientKeyData,

        // 跳过证书验证（不推荐）
        Insecure: false,
    },

    // Bearer Token 认证
    BearerToken:     "my-token",
    BearerTokenFile: "/var/run/secrets/kubernetes.io/serviceaccount/token",

    // 用户名密码认证（基本认证）
    Username: "admin",
    Password: "password",

    // 用户代理
    UserAgent: "my-client/1.0",

    // 超时设置
    Timeout: 30 * time.Second,

    // QPS 和 Burst 限制
    QPS:   50,    // 每秒请求数
    Burst: 100,   // 突发请求数

    // 内容类型
    ContentType: "application/json",
    // AcceptContentTypes: "application/json",
}
```

## 请求构建

### Request 对象

```go
// staging/src/k8s.io/client-go/rest/request.go

type Request struct {
    c *RESTClient

    // 警告处理
    warningHandler WarningHandler

    // 请求方法
    verb string

    // 路径组件
    pathPrefix string
    subpath    string
    params     url.Values
    headers    http.Header

    // 资源信息
    namespace    string
    namespaceSet bool
    resource     string
    resourceName string
    subresource  string

    // 请求体
    body io.Reader

    // 超时
    timeout time.Duration
}
```

### 链式调用

```go
// RESTClient 使用链式调用构建请求

// GET 请求
result := &corev1.PodList{}
err := restClient.Get().                          // HTTP 方法
    Namespace("default").                          // 命名空间
    Resource("pods").                              // 资源类型
    VersionedParams(&metav1.ListOptions{          // 查询参数
        LabelSelector: "app=nginx",
        Limit:         100,
    }, scheme.ParameterCodec).
    Timeout(30 * time.Second).                    // 超时
    Do(context.TODO()).                           // 执行请求
    Into(result)                                  // 解析响应

// POST 请求
pod := &corev1.Pod{
    ObjectMeta: metav1.ObjectMeta{
        Name:      "my-pod",
        Namespace: "default",
    },
    Spec: corev1.PodSpec{
        Containers: []corev1.Container{
            {Name: "nginx", Image: "nginx:latest"},
        },
    },
}

result := &corev1.Pod{}
err := restClient.Post().
    Namespace("default").
    Resource("pods").
    Body(pod).
    Do(context.TODO()).
    Into(result)
```

## CRUD 操作

### Create 操作

```go
func createPod(restClient *rest.RESTClient) (*corev1.Pod, error) {
    pod := &corev1.Pod{
        ObjectMeta: metav1.ObjectMeta{
            Name:      "example-pod",
            Namespace: "default",
            Labels: map[string]string{
                "app": "example",
            },
        },
        Spec: corev1.PodSpec{
            Containers: []corev1.Container{
                {
                    Name:  "main",
                    Image: "nginx:1.21",
                    Ports: []corev1.ContainerPort{
                        {ContainerPort: 80},
                    },
                },
            },
        },
    }

    result := &corev1.Pod{}
    err := restClient.Post().
        Namespace(pod.Namespace).
        Resource("pods").
        Body(pod).
        Do(context.TODO()).
        Into(result)

    return result, err
}
```

### Get 操作

```go
func getPod(restClient *rest.RESTClient, namespace, name string) (*corev1.Pod, error) {
    result := &corev1.Pod{}
    err := restClient.Get().
        Namespace(namespace).
        Resource("pods").
        Name(name).
        Do(context.TODO()).
        Into(result)

    return result, err
}
```

### List 操作

```go
func listPods(restClient *rest.RESTClient, namespace string, labelSelector string) (*corev1.PodList, error) {
    result := &corev1.PodList{}
    err := restClient.Get().
        Namespace(namespace).
        Resource("pods").
        VersionedParams(&metav1.ListOptions{
            LabelSelector: labelSelector,
        }, scheme.ParameterCodec).
        Do(context.TODO()).
        Into(result)

    return result, err
}
```

### Update 操作

```go
func updatePod(restClient *rest.RESTClient, pod *corev1.Pod) (*corev1.Pod, error) {
    result := &corev1.Pod{}
    err := restClient.Put().
        Namespace(pod.Namespace).
        Resource("pods").
        Name(pod.Name).
        Body(pod).
        Do(context.TODO()).
        Into(result)

    return result, err
}
```

### Patch 操作

```go
import "k8s.io/apimachinery/pkg/types"

func patchPod(restClient *rest.RESTClient, namespace, name string, patchData []byte) (*corev1.Pod, error) {
    result := &corev1.Pod{}
    err := restClient.Patch(types.StrategicMergePatchType).
        Namespace(namespace).
        Resource("pods").
        Name(name).
        Body(patchData).
        Do(context.TODO()).
        Into(result)

    return result, err
}

// 使用示例
patchData := []byte(`{
    "metadata": {
        "labels": {
            "new-label": "new-value"
        }
    }
}`)
pod, err := patchPod(restClient, "default", "my-pod", patchData)
```

### Delete 操作

```go
func deletePod(restClient *rest.RESTClient, namespace, name string) error {
    return restClient.Delete().
        Namespace(namespace).
        Resource("pods").
        Name(name).
        Body(&metav1.DeleteOptions{}).
        Do(context.TODO()).
        Error()
}

// 带 GracePeriod 的删除
func deletePodGracefully(restClient *rest.RESTClient, namespace, name string, gracePeriod int64) error {
    return restClient.Delete().
        Namespace(namespace).
        Resource("pods").
        Name(name).
        Body(&metav1.DeleteOptions{
            GracePeriodSeconds: &gracePeriod,
        }).
        Do(context.TODO()).
        Error()
}
```

## Watch 操作

### 建立 Watch

```go
func watchPods(restClient *rest.RESTClient, namespace string) (watch.Interface, error) {
    return restClient.Get().
        Namespace(namespace).
        Resource("pods").
        VersionedParams(&metav1.ListOptions{
            Watch: true,
        }, scheme.ParameterCodec).
        Watch(context.TODO())
}

// 使用 Watch
watcher, err := watchPods(restClient, "default")
if err != nil {
    panic(err)
}
defer watcher.Stop()

for event := range watcher.ResultChan() {
    pod, ok := event.Object.(*corev1.Pod)
    if !ok {
        continue
    }

    switch event.Type {
    case watch.Added:
        fmt.Printf("Pod added: %s\n", pod.Name)
    case watch.Modified:
        fmt.Printf("Pod modified: %s\n", pod.Name)
    case watch.Deleted:
        fmt.Printf("Pod deleted: %s\n", pod.Name)
    case watch.Error:
        fmt.Printf("Watch error\n")
    }
}
```

### 带 ResourceVersion 的 Watch

```go
func watchPodsFromVersion(restClient *rest.RESTClient, namespace, resourceVersion string) (watch.Interface, error) {
    return restClient.Get().
        Namespace(namespace).
        Resource("pods").
        VersionedParams(&metav1.ListOptions{
            Watch:           true,
            ResourceVersion: resourceVersion,
        }, scheme.ParameterCodec).
        Watch(context.TODO())
}
```

## 子资源操作

### Status 子资源

```go
func updatePodStatus(restClient *rest.RESTClient, pod *corev1.Pod) (*corev1.Pod, error) {
    result := &corev1.Pod{}
    err := restClient.Put().
        Namespace(pod.Namespace).
        Resource("pods").
        Name(pod.Name).
        SubResource("status").
        Body(pod).
        Do(context.TODO()).
        Into(result)

    return result, err
}
```

### Exec 子资源

```go
import (
    "k8s.io/client-go/tools/remotecommand"
)

func execInPod(config *rest.Config, restClient *rest.RESTClient, namespace, podName, containerName string, command []string) error {
    req := restClient.Post().
        Namespace(namespace).
        Resource("pods").
        Name(podName).
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

    return exec.Stream(remotecommand.StreamOptions{
        Stdin:  os.Stdin,
        Stdout: os.Stdout,
        Stderr: os.Stderr,
    })
}
```

### Log 子资源

```go
func getPodLogs(restClient *rest.RESTClient, namespace, podName, containerName string) (string, error) {
    req := restClient.Get().
        Namespace(namespace).
        Resource("pods").
        Name(podName).
        SubResource("log").
        VersionedParams(&corev1.PodLogOptions{
            Container: containerName,
            Follow:    false,
            TailLines: ptr.To(int64(100)),
        }, scheme.ParameterCodec)

    stream, err := req.Stream(context.TODO())
    if err != nil {
        return "", err
    }
    defer stream.Close()

    buf := new(bytes.Buffer)
    _, err = io.Copy(buf, stream)
    return buf.String(), err
}
```

## 错误处理

### 错误类型

```go
import "k8s.io/apimachinery/pkg/api/errors"

func handleError(err error) {
    if err == nil {
        return
    }

    // 检查错误类型
    switch {
    case errors.IsNotFound(err):
        fmt.Println("资源不存在")

    case errors.IsAlreadyExists(err):
        fmt.Println("资源已存在")

    case errors.IsConflict(err):
        fmt.Println("资源版本冲突，需要重新获取")

    case errors.IsForbidden(err):
        fmt.Println("权限不足")

    case errors.IsUnauthorized(err):
        fmt.Println("未认证")

    case errors.IsInvalid(err):
        fmt.Println("请求参数无效")

    case errors.IsTimeout(err):
        fmt.Println("请求超时")

    case errors.IsServerTimeout(err):
        fmt.Println("服务器超时")

    case errors.IsTooManyRequests(err):
        fmt.Println("请求过多，被限流")

    case errors.IsServiceUnavailable(err):
        fmt.Println("服务不可用")

    default:
        fmt.Printf("未知错误: %v\n", err)
    }
}
```

### 重试机制

```go
import "k8s.io/client-go/util/retry"

func updatePodWithRetry(restClient *rest.RESTClient, namespace, name string, updateFn func(*corev1.Pod)) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        // 获取最新版本
        pod := &corev1.Pod{}
        err := restClient.Get().
            Namespace(namespace).
            Resource("pods").
            Name(name).
            Do(context.TODO()).
            Into(pod)
        if err != nil {
            return err
        }

        // 应用更新
        updateFn(pod)

        // 提交更新
        return restClient.Put().
            Namespace(namespace).
            Resource("pods").
            Name(name).
            Body(pod).
            Do(context.TODO()).
            Error()
    })
}
```

## 高级用法

### 分页查询

```go
func listAllPods(restClient *rest.RESTClient, namespace string) ([]corev1.Pod, error) {
    var allPods []corev1.Pod
    continueToken := ""

    for {
        opts := metav1.ListOptions{
            Limit:    100,
            Continue: continueToken,
        }

        result := &corev1.PodList{}
        err := restClient.Get().
            Namespace(namespace).
            Resource("pods").
            VersionedParams(&opts, scheme.ParameterCodec).
            Do(context.TODO()).
            Into(result)
        if err != nil {
            return nil, err
        }

        allPods = append(allPods, result.Items...)

        if result.Continue == "" {
            break
        }
        continueToken = result.Continue
    }

    return allPods, nil
}
```

### 原始请求

```go
func rawRequest(restClient *rest.RESTClient) ([]byte, error) {
    return restClient.Get().
        AbsPath("/healthz").
        DoRaw(context.TODO())
}
```

## 总结

RESTClient 核心要点：

**创建配置**
- InClusterConfig 用于 Pod 内
- BuildConfigFromFlags 用于外部访问
- 配置 TLS、认证、超时等

**请求构建**
- 链式调用构建请求
- 支持所有 HTTP 方法
- 类型安全的参数编码

**CRUD 操作**
- Create/Get/List/Update/Delete
- Patch 支持多种类型
- 子资源访问

**错误处理**
- 类型化错误检查
- 重试机制
- 冲突处理
