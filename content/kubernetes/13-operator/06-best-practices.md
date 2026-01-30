---
title: "Operator 最佳实践"
weight: 6
---
## 概述

构建生产级 Operator 需要遵循一系列最佳实践。本章总结了 Operator 开发中的关键设计原则和实践经验。

## 幂等性设计

### 基本原则

```go
// 幂等性：多次执行 Reconcile 结果相同

// 好的做法：使用 CreateOrUpdate
func (r *Reconciler) reconcileConfigMap(ctx context.Context, db *v1.Database) error {
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name + "-config",
            Namespace: db.Namespace,
        },
    }

    // CreateOrUpdate 确保幂等
    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, cm, func() error {
        // 设置所有者引用
        if err := controllerutil.SetControllerReference(db, cm, r.Scheme); err != nil {
            return err
        }
        // 设置期望的 Data
        cm.Data = map[string]string{
            "config.yaml": generateConfig(db),
        }
        return nil
    })
    return err
}

// 不好的做法：先检查后创建
func (r *Reconciler) badReconcileConfigMap(ctx context.Context, db *v1.Database) error {
    cm := &corev1.ConfigMap{}
    err := r.Get(ctx, types.NamespacedName{Name: db.Name + "-config", Namespace: db.Namespace}, cm)
    if errors.IsNotFound(err) {
        // 可能在检查和创建之间发生并发问题
        return r.Create(ctx, newConfigMap(db))
    }
    // 更新逻辑...
    return err
}
```

### 状态比较

```go
// 只在必要时更新资源
func (r *Reconciler) reconcileStatefulSet(ctx context.Context, db *v1.Database) error {
    sts := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
        },
    }

    op, err := controllerutil.CreateOrUpdate(ctx, r.Client, sts, func() error {
        if err := controllerutil.SetControllerReference(db, sts, r.Scheme); err != nil {
            return err
        }

        // 只设置需要管理的字段
        desired := r.buildStatefulSet(db)

        // Spec 字段
        sts.Spec.Replicas = desired.Spec.Replicas
        sts.Spec.Template = desired.Spec.Template
        // 注意：某些字段如 Selector 是不可变的

        return nil
    })

    if err != nil {
        return err
    }

    if op != controllerutil.OperationResultNone {
        log.Info("StatefulSet reconciled", "operation", op)
    }

    return nil
}
```

## 状态管理

### Conditions 最佳实践

```go
import (
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

const (
    // 标准条件类型
    ConditionTypeReady       = "Ready"
    ConditionTypeProgressing = "Progressing"
    ConditionTypeDegraded    = "Degraded"
    ConditionTypeAvailable   = "Available"
)

// 设置条件
func (r *Reconciler) setCondition(db *v1.Database, condType string, status metav1.ConditionStatus, reason, message string) {
    condition := metav1.Condition{
        Type:               condType,
        Status:             status,
        ObservedGeneration: db.Generation,
        LastTransitionTime: metav1.Now(),
        Reason:             reason,
        Message:            message,
    }
    meta.SetStatusCondition(&db.Status.Conditions, condition)
}

// 检查条件
func isReady(db *v1.Database) bool {
    return meta.IsStatusConditionTrue(db.Status.Conditions, ConditionTypeReady)
}

// 完整的状态更新
func (r *Reconciler) updateStatus(ctx context.Context, db *v1.Database) error {
    // 获取最新版本
    latest := &v1.Database{}
    if err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, latest); err != nil {
        return err
    }

    // 检查 StatefulSet 状态
    sts := &appsv1.StatefulSet{}
    if err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, sts); err != nil {
        if errors.IsNotFound(err) {
            latest.Status.Phase = v1.DatabasePhasePending
            r.setCondition(latest, ConditionTypeReady, metav1.ConditionFalse, "StatefulSetNotFound", "Waiting for StatefulSet to be created")
        } else {
            return err
        }
    } else {
        latest.Status.Replicas = sts.Status.Replicas
        latest.Status.ReadyReplicas = sts.Status.ReadyReplicas

        if sts.Status.ReadyReplicas == *sts.Spec.Replicas {
            latest.Status.Phase = v1.DatabasePhaseRunning
            r.setCondition(latest, ConditionTypeReady, metav1.ConditionTrue, "AllReplicasReady", "All replicas are ready")
        } else {
            latest.Status.Phase = v1.DatabasePhaseCreating
            r.setCondition(latest, ConditionTypeReady, metav1.ConditionFalse, "ReplicasNotReady",
                fmt.Sprintf("Waiting for replicas: %d/%d ready", sts.Status.ReadyReplicas, *sts.Spec.Replicas))
            r.setCondition(latest, ConditionTypeProgressing, metav1.ConditionTrue, "Reconciling", "Reconciliation in progress")
        }
    }

    // 设置 ObservedGeneration
    latest.Status.ObservedGeneration = latest.Generation

    return r.Status().Update(ctx, latest)
}
```

### 事件记录

```go
import (
    "k8s.io/client-go/tools/record"
    corev1 "k8s.io/api/core/v1"
)

type DatabaseReconciler struct {
    client.Client
    Scheme   *runtime.Scheme
    Recorder record.EventRecorder
}

// 在 Manager 中设置
func SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&v1.Database{}).
        Complete(&DatabaseReconciler{
            Client:   mgr.GetClient(),
            Scheme:   mgr.GetScheme(),
            Recorder: mgr.GetEventRecorderFor("database-controller"),
        })
}

// 使用事件
func (r *DatabaseReconciler) recordEvent(db *v1.Database, eventType, reason, message string) {
    r.Recorder.Event(db, eventType, reason, message)
}

func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // ...

    // 记录正常事件
    r.Recorder.Event(db, corev1.EventTypeNormal, "Created", "Database resources created successfully")

    // 记录警告事件
    r.Recorder.Eventf(db, corev1.EventTypeWarning, "ScalingFailed", "Failed to scale: %v", err)

    // 带注解的事件
    r.Recorder.AnnotatedEventf(db, map[string]string{
        "operator-version": "1.0.0",
    }, corev1.EventTypeNormal, "Reconciled", "Database reconciled")

    return ctrl.Result{}, nil
}
```

## Finalizer 模式

### 正确实现

```go
const finalizerName = "database.example.com/finalizer"

func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    db := &v1.Database{}
    if err := r.Get(ctx, req.NamespacedName, db); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // 检查是否正在删除
    if !db.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, db)
    }

    // 确保 Finalizer 存在
    if !controllerutil.ContainsFinalizer(db, finalizerName) {
        log.Info("Adding finalizer")
        controllerutil.AddFinalizer(db, finalizerName)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
        // 返回 Requeue 以使用更新后的对象继续
        return ctrl.Result{Requeue: true}, nil
    }

    // 正常协调逻辑
    return r.reconcileNormal(ctx, db)
}

func (r *Reconciler) reconcileDelete(ctx context.Context, db *v1.Database) (ctrl.Result, error) {
    log := log.FromContext(ctx)

    if !controllerutil.ContainsFinalizer(db, finalizerName) {
        // Finalizer 已移除，无需处理
        return ctrl.Result{}, nil
    }

    log.Info("Performing cleanup")

    // 1. 清理外部资源
    if err := r.cleanupExternalResources(ctx, db); err != nil {
        // 如果清理失败，返回错误以触发重试
        return ctrl.Result{}, fmt.Errorf("failed to cleanup external resources: %w", err)
    }

    // 2. 等待子资源删除（可选）
    if hasOwnedResources, err := r.hasOwnedResources(ctx, db); err != nil {
        return ctrl.Result{}, err
    } else if hasOwnedResources {
        log.Info("Waiting for owned resources to be deleted")
        return ctrl.Result{RequeueAfter: 5 * time.Second}, nil
    }

    // 3. 移除 Finalizer
    log.Info("Removing finalizer")
    controllerutil.RemoveFinalizer(db, finalizerName)
    if err := r.Update(ctx, db); err != nil {
        return ctrl.Result{}, err
    }

    return ctrl.Result{}, nil
}

func (r *Reconciler) cleanupExternalResources(ctx context.Context, db *v1.Database) error {
    // 清理云资源、DNS 记录等外部资源
    return nil
}

func (r *Reconciler) hasOwnedResources(ctx context.Context, db *v1.Database) (bool, error) {
    // 检查是否还有子资源
    stsList := &appsv1.StatefulSetList{}
    if err := r.List(ctx, stsList, client.InNamespace(db.Namespace), client.MatchingFields{
        "metadata.ownerReferences.uid": string(db.UID),
    }); err != nil {
        return false, err
    }
    return len(stsList.Items) > 0, nil
}
```

## 错误处理

### 错误分类

```go
// 错误类型定义
type ReconcileError struct {
    Err       error
    Retryable bool
    Delay     time.Duration
}

func (e *ReconcileError) Error() string {
    return e.Err.Error()
}

// 可重试错误
func NewRetryableError(err error, delay time.Duration) *ReconcileError {
    return &ReconcileError{
        Err:       err,
        Retryable: true,
        Delay:     delay,
    }
}

// 永久错误
func NewPermanentError(err error) *ReconcileError {
    return &ReconcileError{
        Err:       err,
        Retryable: false,
    }
}

// 错误处理
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    result, err := r.doReconcile(ctx, req)

    if err != nil {
        var reconcileErr *ReconcileError
        if errors.As(err, &reconcileErr) {
            if reconcileErr.Retryable {
                return ctrl.Result{RequeueAfter: reconcileErr.Delay}, nil
            }
            // 永久错误，不重试但记录
            log.Error(err, "Permanent error, not retrying")
            return ctrl.Result{}, nil
        }
        // 未知错误，使用默认重试
        return ctrl.Result{}, err
    }

    return result, nil
}
```

### 冲突处理

```go
import "k8s.io/client-go/util/retry"

// 使用重试处理冲突
func (r *Reconciler) updateWithRetry(ctx context.Context, db *v1.Database, updateFn func(*v1.Database) error) error {
    return retry.RetryOnConflict(retry.DefaultRetry, func() error {
        // 获取最新版本
        latest := &v1.Database{}
        if err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, latest); err != nil {
            return err
        }

        // 应用更新
        if err := updateFn(latest); err != nil {
            return err
        }

        // 提交更新
        return r.Update(ctx, latest)
    })
}

// 使用示例
func (r *Reconciler) addAnnotation(ctx context.Context, db *v1.Database) error {
    return r.updateWithRetry(ctx, db, func(d *v1.Database) error {
        if d.Annotations == nil {
            d.Annotations = make(map[string]string)
        }
        d.Annotations["last-reconciled"] = time.Now().Format(time.RFC3339)
        return nil
    })
}
```

## 资源管理

### OwnerReference

```go
// 设置 OwnerReference
func (r *Reconciler) setOwnerReference(owner, owned metav1.Object) error {
    return controllerutil.SetControllerReference(owner, owned, r.Scheme)
}

// 构建资源时设置
func (r *Reconciler) buildStatefulSet(db *v1.Database) *appsv1.StatefulSet {
    sts := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
            Labels:    r.buildLabels(db),
        },
        Spec: appsv1.StatefulSetSpec{
            // ...
        },
    }

    // 设置 OwnerReference
    _ = controllerutil.SetControllerReference(db, sts, r.Scheme)

    return sts
}
```

### 标签管理

```go
// 标准标签
func (r *Reconciler) buildLabels(db *v1.Database) map[string]string {
    return map[string]string{
        "app.kubernetes.io/name":       "database",
        "app.kubernetes.io/instance":   db.Name,
        "app.kubernetes.io/version":    db.Spec.Version,
        "app.kubernetes.io/component":  "database",
        "app.kubernetes.io/part-of":    "database-operator",
        "app.kubernetes.io/managed-by": "database-operator",
    }
}

// 选择器标签（不可变）
func (r *Reconciler) buildSelectorLabels(db *v1.Database) map[string]string {
    return map[string]string{
        "app.kubernetes.io/name":     "database",
        "app.kubernetes.io/instance": db.Name,
    }
}
```

## 性能优化

### 缓存配置

```go
// Manager 配置缓存
mgr, err := ctrl.NewManager(cfg, ctrl.Options{
    Cache: cache.Options{
        // 仅缓存特定命名空间
        DefaultNamespaces: map[string]cache.Config{
            "production": {},
        },
        // 特定资源的缓存配置
        ByObject: map[client.Object]cache.ByObject{
            &corev1.Secret{}: {
                // 只缓存特定标签的 Secret
                Label: labels.SelectorFromSet(labels.Set{"managed": "true"}),
            },
        },
    },
})
```

### 索引优化

```go
// 添加索引
func SetupIndexes(mgr ctrl.Manager) error {
    // 按 spec.engine 索引 Database
    if err := mgr.GetFieldIndexer().IndexField(context.Background(), &v1.Database{}, "spec.engine", func(obj client.Object) []string {
        db := obj.(*v1.Database)
        return []string{db.Spec.Engine}
    }); err != nil {
        return err
    }

    // 按 OwnerReference 索引 StatefulSet
    if err := mgr.GetFieldIndexer().IndexField(context.Background(), &appsv1.StatefulSet{}, "metadata.ownerReferences.uid", func(obj client.Object) []string {
        sts := obj.(*appsv1.StatefulSet)
        var uids []string
        for _, ref := range sts.OwnerReferences {
            uids = append(uids, string(ref.UID))
        }
        return uids
    }); err != nil {
        return err
    }

    return nil
}

// 使用索引查询
func (r *Reconciler) listDatabasesByEngine(ctx context.Context, engine string) (*v1.DatabaseList, error) {
    dbList := &v1.DatabaseList{}
    err := r.List(ctx, dbList, client.MatchingFields{"spec.engine": engine})
    return dbList, err
}
```

### 限制并发

```go
// 控制器配置
func (r *Reconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&v1.Database{}).
        WithOptions(controller.Options{
            // 最大并发数
            MaxConcurrentReconciles: 5,
            // 自定义限速器
            RateLimiter: workqueue.NewMaxOfRateLimiter(
                workqueue.NewItemExponentialFailureRateLimiter(1*time.Second, 1000*time.Second),
                &workqueue.BucketRateLimiter{Limiter: rate.NewLimiter(rate.Limit(50), 300)},
            ),
        }).
        Complete(r)
}
```

## 可观测性

### 指标

```go
import (
    "github.com/prometheus/client_golang/prometheus"
    "sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
    reconcileTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "database_reconcile_total",
            Help: "Total number of reconciliations",
        },
        []string{"namespace", "name", "result"},
    )

    reconcileDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "database_reconcile_duration_seconds",
            Help:    "Duration of reconciliation",
            Buckets: prometheus.DefBuckets,
        },
        []string{"namespace", "name"},
    )

    databaseStatus = prometheus.NewGaugeVec(
        prometheus.GaugeOpts{
            Name: "database_status",
            Help: "Current status of database",
        },
        []string{"namespace", "name", "phase"},
    )
)

func init() {
    metrics.Registry.MustRegister(reconcileTotal, reconcileDuration, databaseStatus)
}

// 在 Reconcile 中使用
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    start := time.Now()
    defer func() {
        reconcileDuration.WithLabelValues(req.Namespace, req.Name).Observe(time.Since(start).Seconds())
    }()

    result, err := r.doReconcile(ctx, req)

    if err != nil {
        reconcileTotal.WithLabelValues(req.Namespace, req.Name, "error").Inc()
    } else {
        reconcileTotal.WithLabelValues(req.Namespace, req.Name, "success").Inc()
    }

    return result, err
}
```

### 日志

```go
import "sigs.k8s.io/controller-runtime/pkg/log"

func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 创建带上下文的 logger
    logger := log.FromContext(ctx).WithValues(
        "database", req.NamespacedName,
        "reconcileID", uuid.New().String(),
    )

    // 结构化日志
    logger.Info("Starting reconciliation")

    logger.V(1).Info("Debug info", "step", "fetching database")

    if err != nil {
        logger.Error(err, "Failed to reconcile", "reason", "network error")
    }

    logger.Info("Reconciliation completed", "duration", time.Since(start))

    return ctrl.Result{}, nil
}
```

## 总结

Operator 最佳实践核心要点：

**幂等性设计**
- 使用 CreateOrUpdate
- 只在必要时更新
- 比较状态变化

**状态管理**
- 使用标准 Conditions
- 记录有意义的事件
- 更新 ObservedGeneration

**Finalizer 模式**
- 正确添加和移除
- 清理外部资源
- 处理删除顺序

**错误处理**
- 区分可重试和永久错误
- 使用 retry 处理冲突
- 合理的重试策略

**资源管理**
- 设置 OwnerReference
- 使用标准标签
- 管理资源生命周期

**性能优化**
- 配置缓存范围
- 使用字段索引
- 限制并发数

**可观测性**
- Prometheus 指标
- 结构化日志
- 事件记录
