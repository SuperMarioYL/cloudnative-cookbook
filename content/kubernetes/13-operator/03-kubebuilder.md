---
title: "Kubebuilder"
weight: 3
---
## 概述

Kubebuilder 是 Kubernetes SIG API Machinery 维护的官方脚手架工具，用于构建 Kubernetes API 和控制器。它提供了标准化的项目结构、代码生成和最佳实践。

## 项目初始化

### 安装 Kubebuilder

```bash
# 下载安装
curl -L -o kubebuilder "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder
sudo mv kubebuilder /usr/local/bin/

# 验证安装
kubebuilder version
```

### 初始化项目

```bash
# 创建项目目录
mkdir database-operator && cd database-operator

# 初始化项目
kubebuilder init \
    --domain example.com \
    --repo github.com/example/database-operator \
    --owner "Example Inc."

# 带组件配置初始化
kubebuilder init \
    --domain example.com \
    --repo github.com/example/database-operator \
    --component-config
```

### 目录结构

```
database-operator/
├── Dockerfile                 # 容器镜像构建
├── Makefile                   # 构建和部署命令
├── PROJECT                    # 项目元数据
├── README.md
├── api/                       # API 定义
│   └── v1/
│       ├── database_types.go  # 类型定义
│       ├── database_webhook.go # Webhook 定义
│       ├── groupversion_info.go
│       └── zz_generated.deepcopy.go
├── bin/                       # 构建输出
├── cmd/
│   └── main.go                # 入口文件
├── config/
│   ├── certmanager/           # cert-manager 配置
│   ├── crd/                   # CRD 清单
│   │   ├── bases/
│   │   └── patches/
│   ├── default/               # 默认配置
│   ├── manager/               # 控制器部署
│   ├── prometheus/            # Prometheus 监控
│   ├── rbac/                  # RBAC 配置
│   ├── samples/               # 示例 CR
│   └── webhook/               # Webhook 配置
├── go.mod
├── go.sum
├── hack/                      # 脚本工具
│   └── boilerplate.go.txt
└── internal/
    └── controller/
        ├── database_controller.go  # 控制器实现
        └── suite_test.go           # 测试套件
```

## API 创建

### 创建 API

```bash
# 创建 API 和控制器
kubebuilder create api \
    --group database \
    --version v1 \
    --kind Database

# 只创建 API（不创建控制器）
kubebuilder create api \
    --group database \
    --version v1 \
    --kind Database \
    --controller=false

# 只创建控制器（不创建 API）
kubebuilder create api \
    --group database \
    --version v1 \
    --kind Database \
    --resource=false
```

### 类型定义

```go
// api/v1/database_types.go

package v1

import (
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// DatabaseSpec defines the desired state of Database
type DatabaseSpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // Engine is the database engine type
    // +kubebuilder:validation:Enum=mysql;postgresql;mongodb
    // +kubebuilder:default=postgresql
    Engine string `json:"engine"`

    // Version is the database version
    // +kubebuilder:validation:Required
    Version string `json:"version"`

    // Replicas is the number of database instances
    // +kubebuilder:validation:Minimum=1
    // +kubebuilder:validation:Maximum=10
    // +kubebuilder:default=1
    Replicas int32 `json:"replicas,omitempty"`

    // Storage defines the storage configuration
    Storage StorageSpec `json:"storage"`

    // Resources defines the resource requirements
    // +optional
    Resources *ResourceRequirements `json:"resources,omitempty"`

    // BackupSchedule defines the backup schedule in cron format
    // +optional
    BackupSchedule string `json:"backupSchedule,omitempty"`
}

// StorageSpec defines storage configuration
type StorageSpec struct {
    // Size is the storage size
    // +kubebuilder:validation:Pattern=`^[0-9]+[KMGTPE]i$`
    Size string `json:"size"`

    // StorageClass is the storage class name
    // +optional
    StorageClass string `json:"storageClass,omitempty"`
}

// ResourceRequirements defines resource requirements
type ResourceRequirements struct {
    // CPU is the CPU request/limit
    // +optional
    CPU string `json:"cpu,omitempty"`

    // Memory is the memory request/limit
    // +optional
    Memory string `json:"memory,omitempty"`
}

// DatabaseStatus defines the observed state of Database
type DatabaseStatus struct {
    // INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // Phase represents the current phase of the database
    Phase DatabasePhase `json:"phase,omitempty"`

    // Replicas is the current number of replicas
    Replicas int32 `json:"replicas,omitempty"`

    // ReadyReplicas is the number of ready replicas
    ReadyReplicas int32 `json:"readyReplicas,omitempty"`

    // Endpoint is the connection endpoint
    Endpoint string `json:"endpoint,omitempty"`

    // ObservedGeneration is the most recent generation observed
    ObservedGeneration int64 `json:"observedGeneration,omitempty"`

    // Conditions represent the latest available observations
    // +patchMergeKey=type
    // +patchStrategy=merge
    Conditions []metav1.Condition `json:"conditions,omitempty" patchStrategy:"merge" patchMergeKey:"type"`
}

// DatabasePhase represents the phase of a database
// +kubebuilder:validation:Enum=Pending;Creating;Running;Updating;Failed;Terminating
type DatabasePhase string

const (
    DatabasePhasePending     DatabasePhase = "Pending"
    DatabasePhaseCreating    DatabasePhase = "Creating"
    DatabasePhaseRunning     DatabasePhase = "Running"
    DatabasePhaseUpdating    DatabasePhase = "Updating"
    DatabasePhaseFailed      DatabasePhase = "Failed"
    DatabasePhaseTerminating DatabasePhase = "Terminating"
)

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas
// +kubebuilder:printcolumn:name="Engine",type=string,JSONPath=`.spec.engine`
// +kubebuilder:printcolumn:name="Version",type=string,JSONPath=`.spec.version`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
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

## 控制器实现

### 控制器结构

```go
// internal/controller/database_controller.go

package controller

import (
    "context"
    "fmt"
    "time"

    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    "k8s.io/apimachinery/pkg/api/meta"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/runtime"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
    "sigs.k8s.io/controller-runtime/pkg/log"

    databasev1 "github.com/example/database-operator/api/v1"
)

const (
    databaseFinalizer = "database.example.com/finalizer"
)

// DatabaseReconciler reconciles a Database object
type DatabaseReconciler struct {
    client.Client
    Scheme *runtime.Scheme
}

// +kubebuilder:rbac:groups=database.example.com,resources=databases,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=database.example.com,resources=databases/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=database.example.com,resources=databases/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=configmaps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=secrets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=persistentvolumeclaims,verbs=get;list;watch;create;update;patch;delete

// Reconcile is part of the main kubernetes reconciliation loop
func (r *DatabaseReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Reconciling Database")

    // Fetch the Database instance
    database := &databasev1.Database{}
    if err := r.Get(ctx, req.NamespacedName, database); err != nil {
        if errors.IsNotFound(err) {
            logger.Info("Database resource not found, ignoring")
            return ctrl.Result{}, nil
        }
        logger.Error(err, "Failed to get Database")
        return ctrl.Result{}, err
    }

    // Handle deletion
    if !database.DeletionTimestamp.IsZero() {
        return r.reconcileDelete(ctx, database)
    }

    // Add finalizer if not present
    if !controllerutil.ContainsFinalizer(database, databaseFinalizer) {
        controllerutil.AddFinalizer(database, databaseFinalizer)
        if err := r.Update(ctx, database); err != nil {
            return ctrl.Result{}, err
        }
        return ctrl.Result{Requeue: true}, nil
    }

    // Reconcile resources
    result, err := r.reconcileResources(ctx, database)
    if err != nil {
        r.setCondition(database, "Ready", metav1.ConditionFalse, "ReconcileFailed", err.Error())
        if updateErr := r.Status().Update(ctx, database); updateErr != nil {
            logger.Error(updateErr, "Failed to update status")
        }
        return result, err
    }

    // Update status
    if err := r.updateStatus(ctx, database); err != nil {
        logger.Error(err, "Failed to update status")
        return ctrl.Result{}, err
    }

    logger.Info("Successfully reconciled Database")
    return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
}

func (r *DatabaseReconciler) reconcileResources(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    // Update phase
    db.Status.Phase = databasev1.DatabasePhaseCreating

    // Reconcile ConfigMap
    if err := r.reconcileConfigMap(ctx, db); err != nil {
        return ctrl.Result{}, fmt.Errorf("failed to reconcile ConfigMap: %w", err)
    }

    // Reconcile Secret
    if err := r.reconcileSecret(ctx, db); err != nil {
        return ctrl.Result{}, fmt.Errorf("failed to reconcile Secret: %w", err)
    }

    // Reconcile Service
    if err := r.reconcileService(ctx, db); err != nil {
        return ctrl.Result{}, fmt.Errorf("failed to reconcile Service: %w", err)
    }

    // Reconcile StatefulSet
    if err := r.reconcileStatefulSet(ctx, db); err != nil {
        return ctrl.Result{}, fmt.Errorf("failed to reconcile StatefulSet: %w", err)
    }

    return ctrl.Result{}, nil
}

func (r *DatabaseReconciler) reconcileConfigMap(ctx context.Context, db *databasev1.Database) error {
    cm := &corev1.ConfigMap{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name + "-config",
            Namespace: db.Namespace,
        },
    }

    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, cm, func() error {
        // Set owner reference
        if err := controllerutil.SetControllerReference(db, cm, r.Scheme); err != nil {
            return err
        }

        // Set data
        cm.Data = map[string]string{
            "database.conf": r.generateConfig(db),
        }

        return nil
    })

    return err
}

func (r *DatabaseReconciler) reconcileSecret(ctx context.Context, db *databasev1.Database) error {
    secret := &corev1.Secret{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name + "-credentials",
            Namespace: db.Namespace,
        },
    }

    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, secret, func() error {
        if err := controllerutil.SetControllerReference(db, secret, r.Scheme); err != nil {
            return err
        }

        // Only set password if not exists
        if secret.Data == nil {
            secret.Data = map[string][]byte{
                "username": []byte("admin"),
                "password": []byte(generatePassword()),
            }
        }

        return nil
    })

    return err
}

func (r *DatabaseReconciler) reconcileService(ctx context.Context, db *databasev1.Database) error {
    svc := &corev1.Service{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
        },
    }

    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, svc, func() error {
        if err := controllerutil.SetControllerReference(db, svc, r.Scheme); err != nil {
            return err
        }

        svc.Spec.Selector = map[string]string{
            "app.kubernetes.io/name":     "database",
            "app.kubernetes.io/instance": db.Name,
        }
        svc.Spec.Ports = []corev1.ServicePort{
            {
                Name:     "database",
                Port:     getDefaultPort(db.Spec.Engine),
                Protocol: corev1.ProtocolTCP,
            },
        }
        svc.Spec.ClusterIP = corev1.ClusterIPNone // Headless service

        return nil
    })

    return err
}

func (r *DatabaseReconciler) reconcileStatefulSet(ctx context.Context, db *databasev1.Database) error {
    sts := &appsv1.StatefulSet{
        ObjectMeta: metav1.ObjectMeta{
            Name:      db.Name,
            Namespace: db.Namespace,
        },
    }

    _, err := controllerutil.CreateOrUpdate(ctx, r.Client, sts, func() error {
        if err := controllerutil.SetControllerReference(db, sts, r.Scheme); err != nil {
            return err
        }

        labels := map[string]string{
            "app.kubernetes.io/name":     "database",
            "app.kubernetes.io/instance": db.Name,
        }

        sts.Spec.Replicas = &db.Spec.Replicas
        sts.Spec.ServiceName = db.Name
        sts.Spec.Selector = &metav1.LabelSelector{
            MatchLabels: labels,
        }
        sts.Spec.Template = corev1.PodTemplateSpec{
            ObjectMeta: metav1.ObjectMeta{
                Labels: labels,
            },
            Spec: corev1.PodSpec{
                Containers: []corev1.Container{
                    {
                        Name:  "database",
                        Image: getImage(db.Spec.Engine, db.Spec.Version),
                        Ports: []corev1.ContainerPort{
                            {
                                Name:          "database",
                                ContainerPort: getDefaultPort(db.Spec.Engine),
                            },
                        },
                        EnvFrom: []corev1.EnvFromSource{
                            {
                                SecretRef: &corev1.SecretEnvSource{
                                    LocalObjectReference: corev1.LocalObjectReference{
                                        Name: db.Name + "-credentials",
                                    },
                                },
                            },
                        },
                        VolumeMounts: []corev1.VolumeMount{
                            {
                                Name:      "data",
                                MountPath: "/var/lib/database",
                            },
                            {
                                Name:      "config",
                                MountPath: "/etc/database",
                            },
                        },
                    },
                },
                Volumes: []corev1.Volume{
                    {
                        Name: "config",
                        VolumeSource: corev1.VolumeSource{
                            ConfigMap: &corev1.ConfigMapVolumeSource{
                                LocalObjectReference: corev1.LocalObjectReference{
                                    Name: db.Name + "-config",
                                },
                            },
                        },
                    },
                },
            },
        }

        // Add VolumeClaimTemplate
        sts.Spec.VolumeClaimTemplates = []corev1.PersistentVolumeClaim{
            {
                ObjectMeta: metav1.ObjectMeta{
                    Name: "data",
                },
                Spec: corev1.PersistentVolumeClaimSpec{
                    AccessModes: []corev1.PersistentVolumeAccessMode{
                        corev1.ReadWriteOnce,
                    },
                    Resources: corev1.VolumeResourceRequirements{
                        Requests: corev1.ResourceList{
                            corev1.ResourceStorage: resource.MustParse(db.Spec.Storage.Size),
                        },
                    },
                },
            },
        }

        return nil
    })

    return err
}

func (r *DatabaseReconciler) reconcileDelete(ctx context.Context, db *databasev1.Database) (ctrl.Result, error) {
    logger := log.FromContext(ctx)
    logger.Info("Reconciling Database deletion")

    // Perform cleanup
    if controllerutil.ContainsFinalizer(db, databaseFinalizer) {
        // Run finalization logic
        if err := r.finalizeDatabase(ctx, db); err != nil {
            return ctrl.Result{}, err
        }

        // Remove finalizer
        controllerutil.RemoveFinalizer(db, databaseFinalizer)
        if err := r.Update(ctx, db); err != nil {
            return ctrl.Result{}, err
        }
    }

    return ctrl.Result{}, nil
}

func (r *DatabaseReconciler) finalizeDatabase(ctx context.Context, db *databasev1.Database) error {
    // Cleanup external resources if any
    // e.g., cloud provider resources, DNS records, etc.
    return nil
}

func (r *DatabaseReconciler) updateStatus(ctx context.Context, db *databasev1.Database) error {
    // Get StatefulSet status
    sts := &appsv1.StatefulSet{}
    if err := r.Get(ctx, types.NamespacedName{Name: db.Name, Namespace: db.Namespace}, sts); err != nil {
        if errors.IsNotFound(err) {
            db.Status.Phase = databasev1.DatabasePhasePending
            db.Status.ReadyReplicas = 0
        } else {
            return err
        }
    } else {
        db.Status.Replicas = sts.Status.Replicas
        db.Status.ReadyReplicas = sts.Status.ReadyReplicas

        if sts.Status.ReadyReplicas == *sts.Spec.Replicas {
            db.Status.Phase = databasev1.DatabasePhaseRunning
            r.setCondition(db, "Ready", metav1.ConditionTrue, "AllReplicasReady", "All replicas are ready")
        } else {
            db.Status.Phase = databasev1.DatabasePhaseCreating
            r.setCondition(db, "Ready", metav1.ConditionFalse, "ReplicasNotReady", "Waiting for replicas to be ready")
        }
    }

    // Set endpoint
    db.Status.Endpoint = fmt.Sprintf("%s.%s.svc:%d", db.Name, db.Namespace, getDefaultPort(db.Spec.Engine))

    // Set observed generation
    db.Status.ObservedGeneration = db.Generation

    return r.Status().Update(ctx, db)
}

func (r *DatabaseReconciler) setCondition(db *databasev1.Database, condType string, status metav1.ConditionStatus, reason, message string) {
    meta.SetStatusCondition(&db.Status.Conditions, metav1.Condition{
        Type:               condType,
        Status:             status,
        Reason:             reason,
        Message:            message,
        ObservedGeneration: db.Generation,
    })
}

// SetupWithManager sets up the controller with the Manager.
func (r *DatabaseReconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&databasev1.Database{}).
        Owns(&appsv1.StatefulSet{}).
        Owns(&corev1.Service{}).
        Owns(&corev1.ConfigMap{}).
        Owns(&corev1.Secret{}).
        Complete(r)
}

// Helper functions
func getImage(engine, version string) string {
    images := map[string]string{
        "mysql":      "mysql",
        "postgresql": "postgres",
        "mongodb":    "mongo",
    }
    return fmt.Sprintf("%s:%s", images[engine], version)
}

func getDefaultPort(engine string) int32 {
    ports := map[string]int32{
        "mysql":      3306,
        "postgresql": 5432,
        "mongodb":    27017,
    }
    return ports[engine]
}

func (r *DatabaseReconciler) generateConfig(db *databasev1.Database) string {
    // Generate database configuration
    return fmt.Sprintf("# Configuration for %s %s\n", db.Spec.Engine, db.Spec.Version)
}

func generatePassword() string {
    // Generate random password
    return "changeme"
}
```

## Webhook 实现

### 创建 Webhook

```bash
# 创建 defaulting 和 validating webhook
kubebuilder create webhook \
    --group database \
    --version v1 \
    --kind Database \
    --defaulting \
    --programmatic-validation
```

### Webhook 代码

```go
// api/v1/database_webhook.go

package v1

import (
    "context"
    "fmt"

    "k8s.io/apimachinery/pkg/runtime"
    ctrl "sigs.k8s.io/controller-runtime"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/webhook"
    "sigs.k8s.io/controller-runtime/pkg/webhook/admission"
)

var databaselog = logf.Log.WithName("database-resource")

// SetupWebhookWithManager sets up the webhook with the Manager.
func (r *Database) SetupWebhookWithManager(mgr ctrl.Manager) error {
    return ctrl.NewWebhookManagedBy(mgr).
        For(r).
        Complete()
}

// +kubebuilder:webhook:path=/mutate-database-example-com-v1-database,mutating=true,failurePolicy=fail,sideEffects=None,groups=database.example.com,resources=databases,verbs=create;update,versions=v1,name=mdatabase.kb.io,admissionReviewVersions=v1

var _ webhook.Defaulter = &Database{}

// Default implements webhook.Defaulter
func (r *Database) Default() {
    databaselog.Info("default", "name", r.Name)

    // Set default replicas
    if r.Spec.Replicas == 0 {
        r.Spec.Replicas = 1
    }

    // Set default storage class
    if r.Spec.Storage.StorageClass == "" {
        r.Spec.Storage.StorageClass = "standard"
    }

    // Set labels
    if r.Labels == nil {
        r.Labels = make(map[string]string)
    }
    r.Labels["app.kubernetes.io/managed-by"] = "database-operator"
    r.Labels["app.kubernetes.io/name"] = "database"
    r.Labels["app.kubernetes.io/instance"] = r.Name
}

// +kubebuilder:webhook:path=/validate-database-example-com-v1-database,mutating=false,failurePolicy=fail,sideEffects=None,groups=database.example.com,resources=databases,verbs=create;update;delete,versions=v1,name=vdatabase.kb.io,admissionReviewVersions=v1

var _ webhook.CustomValidator = &Database{}

// ValidateCreate implements webhook.CustomValidator
func (r *Database) ValidateCreate(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    db := obj.(*Database)
    databaselog.Info("validate create", "name", db.Name)

    return db.validate()
}

// ValidateUpdate implements webhook.CustomValidator
func (r *Database) ValidateUpdate(ctx context.Context, oldObj, newObj runtime.Object) (admission.Warnings, error) {
    oldDB := oldObj.(*Database)
    newDB := newObj.(*Database)
    databaselog.Info("validate update", "name", newDB.Name)

    // Validate immutable fields
    if newDB.Spec.Engine != oldDB.Spec.Engine {
        return nil, fmt.Errorf("engine type is immutable")
    }

    return newDB.validate()
}

// ValidateDelete implements webhook.CustomValidator
func (r *Database) ValidateDelete(ctx context.Context, obj runtime.Object) (admission.Warnings, error) {
    db := obj.(*Database)
    databaselog.Info("validate delete", "name", db.Name)

    // Add deletion validation if needed
    return nil, nil
}

func (r *Database) validate() (admission.Warnings, error) {
    var warnings admission.Warnings

    // Validate engine
    validEngines := map[string]bool{
        "mysql":      true,
        "postgresql": true,
        "mongodb":    true,
    }
    if !validEngines[r.Spec.Engine] {
        return warnings, fmt.Errorf("invalid engine: %s", r.Spec.Engine)
    }

    // Validate replicas
    if r.Spec.Replicas < 1 || r.Spec.Replicas > 10 {
        return warnings, fmt.Errorf("replicas must be between 1 and 10")
    }

    // Warning for high replica count
    if r.Spec.Replicas > 5 {
        warnings = append(warnings, "high replica count may increase resource usage")
    }

    return warnings, nil
}
```

## 本地测试

### 运行控制器

```bash
# 生成 CRD 清单
make manifests

# 安装 CRD
make install

# 运行控制器（本地）
make run

# 创建示例资源
kubectl apply -f config/samples/
```

### 测试配置

```go
// internal/controller/suite_test.go

package controller

import (
    "context"
    "path/filepath"
    "testing"
    "time"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"

    "k8s.io/client-go/kubernetes/scheme"
    "k8s.io/client-go/rest"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"
    "sigs.k8s.io/controller-runtime/pkg/envtest"
    logf "sigs.k8s.io/controller-runtime/pkg/log"
    "sigs.k8s.io/controller-runtime/pkg/log/zap"

    databasev1 "github.com/example/database-operator/api/v1"
)

var (
    cfg       *rest.Config
    k8sClient client.Client
    testEnv   *envtest.Environment
    ctx       context.Context
    cancel    context.CancelFunc
)

func TestControllers(t *testing.T) {
    RegisterFailHandler(Fail)
    RunSpecs(t, "Controller Suite")
}

var _ = BeforeSuite(func() {
    logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))

    ctx, cancel = context.WithCancel(context.TODO())

    By("bootstrapping test environment")
    testEnv = &envtest.Environment{
        CRDDirectoryPaths:     []string{filepath.Join("..", "..", "config", "crd", "bases")},
        ErrorIfCRDPathMissing: true,
    }

    var err error
    cfg, err = testEnv.Start()
    Expect(err).NotTo(HaveOccurred())
    Expect(cfg).NotTo(BeNil())

    err = databasev1.AddToScheme(scheme.Scheme)
    Expect(err).NotTo(HaveOccurred())

    k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
    Expect(err).NotTo(HaveOccurred())
    Expect(k8sClient).NotTo(BeNil())

    // Start controller
    mgr, err := ctrl.NewManager(cfg, ctrl.Options{
        Scheme: scheme.Scheme,
    })
    Expect(err).ToNot(HaveOccurred())

    err = (&DatabaseReconciler{
        Client: mgr.GetClient(),
        Scheme: mgr.GetScheme(),
    }).SetupWithManager(mgr)
    Expect(err).ToNot(HaveOccurred())

    go func() {
        defer GinkgoRecover()
        err = mgr.Start(ctx)
        Expect(err).ToNot(HaveOccurred())
    }()
})

var _ = AfterSuite(func() {
    cancel()
    By("tearing down the test environment")
    err := testEnv.Stop()
    Expect(err).NotTo(HaveOccurred())
})
```

### 控制器测试

```go
// internal/controller/database_controller_test.go

package controller

import (
    "time"

    . "github.com/onsi/ginkgo/v2"
    . "github.com/onsi/gomega"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"

    databasev1 "github.com/example/database-operator/api/v1"
)

var _ = Describe("Database Controller", func() {
    const (
        DatabaseName      = "test-database"
        DatabaseNamespace = "default"

        timeout  = time.Second * 30
        interval = time.Millisecond * 250
    )

    Context("When creating a Database", func() {
        It("Should create StatefulSet and Services", func() {
            By("Creating a new Database")
            database := &databasev1.Database{
                ObjectMeta: metav1.ObjectMeta{
                    Name:      DatabaseName,
                    Namespace: DatabaseNamespace,
                },
                Spec: databasev1.DatabaseSpec{
                    Engine:   "postgresql",
                    Version:  "14.5",
                    Replicas: 1,
                    Storage: databasev1.StorageSpec{
                        Size: "10Gi",
                    },
                },
            }
            Expect(k8sClient.Create(ctx, database)).Should(Succeed())

            By("Checking the Database status")
            Eventually(func() bool {
                db := &databasev1.Database{}
                err := k8sClient.Get(ctx, types.NamespacedName{
                    Name:      DatabaseName,
                    Namespace: DatabaseNamespace,
                }, db)
                if err != nil {
                    return false
                }
                return db.Status.Phase == databasev1.DatabasePhaseRunning
            }, timeout, interval).Should(BeTrue())

            By("Cleaning up")
            Expect(k8sClient.Delete(ctx, database)).Should(Succeed())
        })
    })
})
```

## 部署发布

### 构建镜像

```bash
# 构建并推送镜像
make docker-build docker-push IMG=example.com/database-operator:v1.0.0

# 多架构构建
make docker-buildx IMG=example.com/database-operator:v1.0.0
```

### 部署到集群

```bash
# 部署
make deploy IMG=example.com/database-operator:v1.0.0

# 卸载
make undeploy
```

### Kustomize 配置

```yaml
# config/default/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: database-operator-system

namePrefix: database-operator-

resources:
  - ../crd
  - ../rbac
  - ../manager

patches:
  - path: manager_config_patch.yaml

images:
  - name: controller
    newName: example.com/database-operator
    newTag: v1.0.0
```

## 总结

Kubebuilder 核心要点：

**项目初始化**
- `kubebuilder init` 创建项目
- `kubebuilder create api` 创建 API
- `kubebuilder create webhook` 创建 Webhook

**代码开发**
- 类型定义使用 kubebuilder 标记
- Reconciler 实现协调逻辑
- Webhook 实现验证和默认值

**测试**
- envtest 提供测试环境
- Ginkgo/Gomega 测试框架
- 单元测试和集成测试

**部署**
- make manifests 生成清单
- make docker-build 构建镜像
- make deploy 部署到集群
