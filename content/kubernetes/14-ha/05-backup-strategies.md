---
title: "备份策略"
weight: 5
---
## 概述

完善的备份策略是 Kubernetes 集群可靠性的基石。本文深入解析 Kubernetes 集群各层面的备份方法，包括 etcd 数据备份、Kubernetes 资源备份、持久卷备份，以及如何设计自动化备份方案和验证机制。

## etcd 备份

### 快照备份

etcd 快照是集群状态的完整备份，是最重要的备份数据。

```bash
#!/bin/bash
# etcd-snapshot-backup.sh

set -e

# 配置
BACKUP_DIR="/var/backups/etcd"
RETENTION_DAYS=30
ETCD_ENDPOINTS="https://127.0.0.1:2379"
ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"

# 创建备份目录
mkdir -p "$BACKUP_DIR"

# 生成备份文件名
TIMESTAMP=$(date +%Y%m%d%H%M%S)
SNAPSHOT_FILE="${BACKUP_DIR}/snapshot-${TIMESTAMP}.db"

echo "Creating etcd snapshot..."

# 创建快照
etcdctl snapshot save "$SNAPSHOT_FILE" \
    --endpoints="$ETCD_ENDPOINTS" \
    --cacert="$ETCD_CACERT" \
    --cert="$ETCD_CERT" \
    --key="$ETCD_KEY"

# 验证快照
echo "Verifying snapshot..."
etcdutl snapshot status "$SNAPSHOT_FILE" --write-out=table

# 压缩快照
echo "Compressing snapshot..."
gzip "$SNAPSHOT_FILE"
SNAPSHOT_FILE="${SNAPSHOT_FILE}.gz"

# 计算校验和
sha256sum "$SNAPSHOT_FILE" > "${SNAPSHOT_FILE}.sha256"

# 创建软链接指向最新备份
ln -sf "$SNAPSHOT_FILE" "${BACKUP_DIR}/snapshot-latest.db.gz"

# 清理过期备份
echo "Cleaning up old backups..."
find "$BACKUP_DIR" -name "snapshot-*.db.gz" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "snapshot-*.db.gz.sha256" -mtime +$RETENTION_DAYS -delete

# 显示备份信息
BACKUP_SIZE=$(du -h "$SNAPSHOT_FILE" | cut -f1)
echo "Backup completed:"
echo "  File: $SNAPSHOT_FILE"
echo "  Size: $BACKUP_SIZE"
echo "  Checksum: $(cat ${SNAPSHOT_FILE}.sha256 | cut -d' ' -f1)"
```

### 增量备份与 WAL 归档

```bash
#!/bin/bash
# etcd-wal-archive.sh

# WAL 归档用于实现更细粒度的恢复点

ETCD_DATA_DIR="/var/lib/etcd"
WAL_ARCHIVE_DIR="/var/backups/etcd/wal"
MAX_WAL_ARCHIVES=100

mkdir -p "$WAL_ARCHIVE_DIR"

# 复制 WAL 文件（在维护窗口执行）
archive_wal() {
    TIMESTAMP=$(date +%Y%m%d%H%M%S)
    WAL_SNAPSHOT_DIR="${WAL_ARCHIVE_DIR}/wal-${TIMESTAMP}"

    # 复制 WAL 目录
    cp -r "${ETCD_DATA_DIR}/member/wal" "$WAL_SNAPSHOT_DIR"

    # 压缩
    tar -czf "${WAL_SNAPSHOT_DIR}.tar.gz" -C "$WAL_ARCHIVE_DIR" "wal-${TIMESTAMP}"
    rm -rf "$WAL_SNAPSHOT_DIR"

    echo "WAL archived: ${WAL_SNAPSHOT_DIR}.tar.gz"
}

# 清理旧归档
cleanup_old_archives() {
    cd "$WAL_ARCHIVE_DIR"
    ls -t wal-*.tar.gz 2>/dev/null | tail -n +$((MAX_WAL_ARCHIVES + 1)) | xargs -r rm -f
}

archive_wal
cleanup_old_archives
```

### 远程备份上传

```bash
#!/bin/bash
# upload-backup-to-s3.sh

set -e

BACKUP_DIR="/var/backups/etcd"
S3_BUCKET="s3://my-cluster-backups/etcd"
AWS_REGION="us-west-2"

# 上传最新快照到 S3
upload_to_s3() {
    local snapshot_file="$1"
    local checksum_file="${snapshot_file}.sha256"

    echo "Uploading to S3..."

    # 上传快照文件
    aws s3 cp "$snapshot_file" "${S3_BUCKET}/" \
        --region "$AWS_REGION" \
        --storage-class STANDARD_IA

    # 上传校验和文件
    aws s3 cp "$checksum_file" "${S3_BUCKET}/" \
        --region "$AWS_REGION" \
        --storage-class STANDARD_IA

    echo "Upload completed: ${S3_BUCKET}/$(basename $snapshot_file)"
}

# 设置 S3 生命周期策略
setup_lifecycle() {
    cat > /tmp/lifecycle.json << 'EOF'
{
    "Rules": [
        {
            "ID": "etcd-backup-lifecycle",
            "Status": "Enabled",
            "Filter": {
                "Prefix": "etcd/"
            },
            "Transitions": [
                {
                    "Days": 30,
                    "StorageClass": "GLACIER"
                }
            ],
            "Expiration": {
                "Days": 365
            }
        }
    ]
}
EOF

    aws s3api put-bucket-lifecycle-configuration \
        --bucket "${S3_BUCKET#s3://}" \
        --lifecycle-configuration file:///tmp/lifecycle.json
}

# 主流程
LATEST_SNAPSHOT="${BACKUP_DIR}/snapshot-latest.db.gz"
if [ -f "$LATEST_SNAPSHOT" ]; then
    upload_to_s3 "$LATEST_SNAPSHOT"
else
    echo "Error: No snapshot found"
    exit 1
fi
```

### CronJob 自动备份

```yaml
# etcd-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  # 每小时执行一次
  schedule: "0 * * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3

  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 600
      template:
        spec:
          restartPolicy: OnFailure

          # 必须在 control plane 节点运行
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
            - key: node-role.kubernetes.io/control-plane
              effect: NoSchedule

          containers:
            - name: backup
              image: bitnami/etcd:3.5
              command:
                - /bin/bash
                - -c
                - |
                  set -e

                  TIMESTAMP=$(date +%Y%m%d%H%M%S)
                  BACKUP_FILE="/backup/snapshot-${TIMESTAMP}.db"

                  echo "Creating backup..."
                  etcdctl snapshot save "$BACKUP_FILE" \
                      --endpoints=${ETCD_ENDPOINTS} \
                      --cacert=/certs/ca.crt \
                      --cert=/certs/server.crt \
                      --key=/certs/server.key

                  echo "Verifying backup..."
                  etcdutl snapshot status "$BACKUP_FILE" --write-out=table

                  echo "Compressing..."
                  gzip "$BACKUP_FILE"

                  echo "Uploading to S3..."
                  aws s3 cp "${BACKUP_FILE}.gz" s3://${S3_BUCKET}/etcd/

                  echo "Cleanup old local backups..."
                  find /backup -name "snapshot-*.db.gz" -mtime +7 -delete

                  echo "Backup completed!"

              env:
                - name: ETCD_ENDPOINTS
                  value: "https://127.0.0.1:2379"
                - name: S3_BUCKET
                  valueFrom:
                    secretKeyRef:
                      name: backup-credentials
                      key: bucket
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: backup-credentials
                      key: access-key
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: backup-credentials
                      key: secret-key

              volumeMounts:
                - name: etcd-certs
                  mountPath: /certs
                  readOnly: true
                - name: backup-storage
                  mountPath: /backup

          volumes:
            - name: etcd-certs
              hostPath:
                path: /etc/kubernetes/pki/etcd
                type: Directory
            - name: backup-storage
              hostPath:
                path: /var/backups/etcd
                type: DirectoryOrCreate
```

## Kubernetes 资源备份

### Velero 部署与配置

```yaml
# velero-install.yaml

# 1. 安装 Velero
# helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts
# helm install velero vmware-tanzu/velero --namespace velero --create-namespace -f values.yaml

# values.yaml
configuration:
  backupStorageLocation:
    - name: default
      provider: aws
      bucket: my-cluster-backups
      prefix: velero
      config:
        region: us-west-2
        s3ForcePathStyle: "true"
        s3Url: https://s3.us-west-2.amazonaws.com

  volumeSnapshotLocation:
    - name: default
      provider: aws
      config:
        region: us-west-2

  defaultBackupStorageLocation: default
  defaultVolumeSnapshotLocations:
    - aws:default

  # 特性配置
  features: EnableCSI

credentials:
  useSecret: true
  secretContents:
    cloud: |
      [default]
      aws_access_key_id = AKIAIOSFODNN7EXAMPLE
      aws_secret_access_key = wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

initContainers:
  - name: velero-plugin-for-aws
    image: velero/velero-plugin-for-aws:v1.8.0
    volumeMounts:
      - mountPath: /target
        name: plugins

deployNodeAgent: true
nodeAgent:
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 1000m
      memory: 1024Mi

resources:
  requests:
    cpu: 500m
    memory: 128Mi
  limits:
    cpu: 1000m
    memory: 512Mi
```

### 备份策略配置

```yaml
# backup-schedules.yaml

# 生产环境 - 每小时备份
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: production-hourly
  namespace: velero
spec:
  schedule: "0 * * * *"
  useOwnerReferencesInBackup: false
  template:
    includedNamespaces:
      - production
    includedResources:
      - '*'
    excludedResources:
      - events
      - events.events.k8s.io
    storageLocation: default
    volumeSnapshotLocations:
      - default
    ttl: 168h  # 7 天
    snapshotVolumes: true
    defaultVolumesToFsBackup: false

    # 资源过滤
    labelSelector:
      matchExpressions:
        - key: backup
          operator: NotIn
          values:
            - exclude

    # 排序依赖
    orderedResources:
      - secrets
      - configmaps
      - persistentvolumeclaims
      - deployments.apps
      - services
---
# 生产环境 - 每日完整备份
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: production-daily
  namespace: velero
spec:
  schedule: "0 2 * * *"
  template:
    includedNamespaces:
      - production
      - staging
    storageLocation: default
    volumeSnapshotLocations:
      - default
    ttl: 720h  # 30 天
    snapshotVolumes: true

    # 包含集群资源
    includeClusterResources: true
    includedClusterScopedResources:
      - storageclasses
      - persistentvolumes
      - clusterroles
      - clusterrolebindings
---
# 全集群备份 - 每周
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: full-cluster-weekly
  namespace: velero
spec:
  schedule: "0 3 * * 0"  # 每周日凌晨 3 点
  template:
    includedNamespaces:
      - '*'
    excludedNamespaces:
      - kube-system
      - velero
    storageLocation: default
    volumeSnapshotLocations:
      - default
    ttl: 2160h  # 90 天
    snapshotVolumes: true
    includeClusterResources: true
```

### 备份 Hooks

```yaml
# backup-with-hooks.yaml
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: database-backup
  namespace: velero
spec:
  includedNamespaces:
    - production
  labelSelector:
    matchLabels:
      app: database

  # 备份前后执行的 Hooks
  hooks:
    resources:
      # PostgreSQL 备份 Hook
      - name: postgres-backup
        includedNamespaces:
          - production
        labelSelector:
          matchLabels:
            app: postgresql
        pre:
          - exec:
              container: postgresql
              command:
                - /bin/bash
                - -c
                - |
                  # 创建一致性快照点
                  psql -U postgres -c "SELECT pg_start_backup('velero', true);"

                  # 执行逻辑备份（可选）
                  pg_dumpall -U postgres > /backup/pre-backup.sql
              onError: Fail
              timeout: 300s
        post:
          - exec:
              container: postgresql
              command:
                - /bin/bash
                - -c
                - |
                  # 结束备份模式
                  psql -U postgres -c "SELECT pg_stop_backup();"

                  # 清理临时文件
                  rm -f /backup/pre-backup.sql
              onError: Continue
              timeout: 60s

      # MySQL 备份 Hook
      - name: mysql-backup
        includedNamespaces:
          - production
        labelSelector:
          matchLabels:
            app: mysql
        pre:
          - exec:
              container: mysql
              command:
                - /bin/bash
                - -c
                - |
                  # 锁定表并刷新
                  mysql -u root -p${MYSQL_ROOT_PASSWORD} -e "FLUSH TABLES WITH READ LOCK;"
                  mysql -u root -p${MYSQL_ROOT_PASSWORD} -e "FLUSH LOGS;"
              onError: Fail
              timeout: 120s
        post:
          - exec:
              container: mysql
              command:
                - /bin/bash
                - -c
                - |
                  # 解锁表
                  mysql -u root -p${MYSQL_ROOT_PASSWORD} -e "UNLOCK TABLES;"
              onError: Continue
              timeout: 30s

      # Redis 备份 Hook
      - name: redis-backup
        includedNamespaces:
          - production
        labelSelector:
          matchLabels:
            app: redis
        pre:
          - exec:
              container: redis
              command:
                - redis-cli
                - BGSAVE
              onError: Continue
              timeout: 60s
```

### 备份验证

```bash
#!/bin/bash
# verify-backup.sh

# 验证 Velero 备份的完整性和可恢复性

BACKUP_NAME="$1"
TEST_NAMESPACE="backup-verify-$(date +%s)"

if [ -z "$BACKUP_NAME" ]; then
    echo "Usage: $0 <backup-name>"
    exit 1
fi

echo "Verifying backup: $BACKUP_NAME"

# 1. 检查备份状态
echo "Step 1: Checking backup status..."
BACKUP_STATUS=$(velero backup get "$BACKUP_NAME" -o jsonpath='{.status.phase}')
if [ "$BACKUP_STATUS" != "Completed" ]; then
    echo "Error: Backup status is $BACKUP_STATUS, expected Completed"
    exit 1
fi
echo "Backup status: $BACKUP_STATUS ✓"

# 2. 查看备份详情
echo ""
echo "Step 2: Backup details..."
velero backup describe "$BACKUP_NAME" --details

# 3. 检查备份内容
echo ""
echo "Step 3: Checking backup contents..."
BACKUP_ITEMS=$(velero backup get "$BACKUP_NAME" -o jsonpath='{.status.progress.itemsBackedUp}')
echo "Items backed up: $BACKUP_ITEMS"

# 4. 验证卷快照
echo ""
echo "Step 4: Verifying volume snapshots..."
velero backup get "$BACKUP_NAME" -o jsonpath='{.status.volumeSnapshotsCompleted}'
echo ""

# 5. 恢复测试（到隔离的命名空间）
echo ""
echo "Step 5: Restore test..."
kubectl create namespace "$TEST_NAMESPACE"

velero restore create "verify-${BACKUP_NAME}" \
    --from-backup "$BACKUP_NAME" \
    --namespace-mappings "*:${TEST_NAMESPACE}" \
    --restore-volumes=false \
    --wait

# 6. 检查恢复结果
echo ""
echo "Step 6: Checking restore results..."
RESTORE_STATUS=$(velero restore get "verify-${BACKUP_NAME}" -o jsonpath='{.status.phase}')
echo "Restore status: $RESTORE_STATUS"

if [ "$RESTORE_STATUS" = "Completed" ]; then
    echo ""
    echo "Restored resources:"
    kubectl get all -n "$TEST_NAMESPACE"
fi

# 7. 清理测试资源
echo ""
echo "Step 7: Cleaning up..."
kubectl delete namespace "$TEST_NAMESPACE" --wait=false
velero restore delete "verify-${BACKUP_NAME}" --confirm

# 8. 输出验证结果
echo ""
echo "========================================="
if [ "$RESTORE_STATUS" = "Completed" ]; then
    echo "✓ Backup verification PASSED"
    exit 0
else
    echo "✗ Backup verification FAILED"
    exit 1
fi
```

## 持久卷备份

### CSI 快照配置

```yaml
# volume-snapshot-class.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-ebs-snapclass
  labels:
    velero.io/csi-volumesnapshot-class: "true"  # Velero 集成
  annotations:
    snapshot.storage.kubernetes.io/is-default-class: "true"
driver: ebs.csi.aws.com
deletionPolicy: Retain
parameters:
  # AWS 特定参数
  # 无需额外参数
---
# GCP CSI 快照类
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-gcp-pd-snapclass
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: pd.csi.storage.gke.io
deletionPolicy: Retain
parameters:
  snapshot-type: STANDARD
---
# Azure CSI 快照类
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-azure-disk-snapclass
  labels:
    velero.io/csi-volumesnapshot-class: "true"
driver: disk.csi.azure.com
deletionPolicy: Retain
parameters:
  incremental: "true"
```

### 自动快照调度

```yaml
# volume-snapshot-schedule.yaml

# 使用 CronJob 创建定期快照
apiVersion: batch/v1
kind: CronJob
metadata:
  name: database-snapshot
  namespace: production
spec:
  schedule: "0 */6 * * *"  # 每 6 小时
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: snapshot-creator
          restartPolicy: OnFailure
          containers:
            - name: snapshot-creator
              image: bitnami/kubectl:1.28
              command:
                - /bin/bash
                - -c
                - |
                  set -e

                  TIMESTAMP=$(date +%Y%m%d%H%M%S)
                  PVC_NAME="database-pvc"
                  SNAPSHOT_NAME="database-snapshot-${TIMESTAMP}"

                  # 创建快照
                  cat <<EOF | kubectl apply -f -
                  apiVersion: snapshot.storage.k8s.io/v1
                  kind: VolumeSnapshot
                  metadata:
                    name: ${SNAPSHOT_NAME}
                    namespace: production
                    labels:
                      app: database
                      backup-type: scheduled
                  spec:
                    volumeSnapshotClassName: csi-aws-ebs-snapclass
                    source:
                      persistentVolumeClaimName: ${PVC_NAME}
                  EOF

                  # 等待快照就绪
                  echo "Waiting for snapshot to be ready..."
                  kubectl wait volumesnapshot/${SNAPSHOT_NAME} \
                      --for=jsonpath='{.status.readyToUse}'=true \
                      --timeout=300s \
                      -n production

                  echo "Snapshot created: ${SNAPSHOT_NAME}"

                  # 清理旧快照（保留最近 10 个）
                  echo "Cleaning up old snapshots..."
                  kubectl get volumesnapshot -n production \
                      -l app=database,backup-type=scheduled \
                      --sort-by=.metadata.creationTimestamp \
                      -o name | head -n -10 | xargs -r kubectl delete -n production

                  echo "Done!"
---
# ServiceAccount 和 RBAC
apiVersion: v1
kind: ServiceAccount
metadata:
  name: snapshot-creator
  namespace: production
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: snapshot-creator
  namespace: production
rules:
  - apiGroups: ["snapshot.storage.k8s.io"]
    resources: ["volumesnapshots"]
    verbs: ["create", "get", "list", "watch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: snapshot-creator
  namespace: production
subjects:
  - kind: ServiceAccount
    name: snapshot-creator
    namespace: production
roleRef:
  kind: Role
  name: snapshot-creator
  apiGroup: rbac.authorization.k8s.io
```

### 跨区域卷复制

```bash
#!/bin/bash
# cross-region-volume-backup.sh

# 将卷快照复制到另一个区域

SOURCE_REGION="us-west-2"
TARGET_REGION="us-east-1"
SOURCE_SNAPSHOT_ID="$1"

if [ -z "$SOURCE_SNAPSHOT_ID" ]; then
    echo "Usage: $0 <source-snapshot-id>"
    exit 1
fi

echo "Copying snapshot $SOURCE_SNAPSHOT_ID from $SOURCE_REGION to $TARGET_REGION"

# AWS EBS 快照跨区域复制
TARGET_SNAPSHOT_ID=$(aws ec2 copy-snapshot \
    --source-region "$SOURCE_REGION" \
    --source-snapshot-id "$SOURCE_SNAPSHOT_ID" \
    --destination-region "$TARGET_REGION" \
    --description "Cross-region backup of $SOURCE_SNAPSHOT_ID" \
    --query 'SnapshotId' \
    --output text)

echo "Created snapshot: $TARGET_SNAPSHOT_ID"

# 等待快照完成
echo "Waiting for snapshot to complete..."
aws ec2 wait snapshot-completed \
    --snapshot-ids "$TARGET_SNAPSHOT_ID" \
    --region "$TARGET_REGION"

echo "Snapshot copy completed!"

# 添加标签
aws ec2 create-tags \
    --resources "$TARGET_SNAPSHOT_ID" \
    --tags Key=SourceRegion,Value="$SOURCE_REGION" \
           Key=SourceSnapshotId,Value="$SOURCE_SNAPSHOT_ID" \
           Key=BackupType,Value="cross-region" \
    --region "$TARGET_REGION"
```

## 应用一致性备份

### 数据库备份策略

```mermaid
graph TB
    subgraph app_backup["应用一致性备份"]
        subgraph pre_backup["备份前准备"]
            quiesce["暂停应用写入"]
            flush["刷新缓存到磁盘"]
            lock["锁定表/创建检查点"]
        end

        subgraph backup_exec["执行备份"]
            snapshot["创建卷快照"]
            logical["逻辑备份（可选）"]
        end

        subgraph post_backup["备份后清理"]
            unlock["解锁/恢复写入"]
            verify["验证备份完整性"]
            notify["发送通知"]
        end
    end

    quiesce --> flush
    flush --> lock
    lock --> snapshot
    snapshot --> logical
    logical --> unlock
    unlock --> verify
    verify --> notify

    style pre_backup fill:#fff3e0
    style backup_exec fill:#e8f5e9
    style post_backup fill:#e3f2fd
```

### PostgreSQL 备份

```yaml
# postgres-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 2 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:15
              env:
                - name: PGHOST
                  value: "postgresql"
                - name: PGUSER
                  value: "postgres"
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgresql-secret
                      key: password
                - name: S3_BUCKET
                  value: "my-backups"
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: access-key
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: aws-credentials
                      key: secret-key

              command:
                - /bin/bash
                - -c
                - |
                  set -e

                  TIMESTAMP=$(date +%Y%m%d%H%M%S)
                  BACKUP_FILE="/tmp/backup-${TIMESTAMP}.sql.gz"

                  echo "Starting PostgreSQL backup..."

                  # 执行逻辑备份
                  pg_dumpall | gzip > "$BACKUP_FILE"

                  # 验证备份
                  if gzip -t "$BACKUP_FILE"; then
                      echo "Backup file is valid"
                  else
                      echo "Backup file is corrupted!"
                      exit 1
                  fi

                  # 上传到 S3
                  aws s3 cp "$BACKUP_FILE" "s3://${S3_BUCKET}/postgres/backup-${TIMESTAMP}.sql.gz"

                  # 清理本地文件
                  rm -f "$BACKUP_FILE"

                  echo "Backup completed successfully!"

              volumeMounts:
                - name: tmp
                  mountPath: /tmp

          volumes:
            - name: tmp
              emptyDir:
                sizeLimit: 10Gi
```

### MongoDB 备份

```yaml
# mongodb-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mongodb-backup
  namespace: production
spec:
  schedule: "0 3 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: mongo:6.0
              env:
                - name: MONGO_URI
                  valueFrom:
                    secretKeyRef:
                      name: mongodb-secret
                      key: uri

              command:
                - /bin/bash
                - -c
                - |
                  set -e

                  TIMESTAMP=$(date +%Y%m%d%H%M%S)
                  BACKUP_DIR="/backup/mongodb-${TIMESTAMP}"

                  echo "Starting MongoDB backup..."

                  # 使用 mongodump 进行一致性备份
                  mongodump \
                      --uri="$MONGO_URI" \
                      --out="$BACKUP_DIR" \
                      --oplog  # 包含 oplog 确保一致性

                  # 压缩备份
                  tar -czf "${BACKUP_DIR}.tar.gz" -C /backup "mongodb-${TIMESTAMP}"
                  rm -rf "$BACKUP_DIR"

                  # 上传到对象存储
                  # ...

                  echo "Backup completed!"

              volumeMounts:
                - name: backup-storage
                  mountPath: /backup

          volumes:
            - name: backup-storage
              persistentVolumeClaim:
                claimName: backup-pvc
```

## 备份监控与告警

### Prometheus 监控指标

```yaml
# backup-monitoring.yaml

# ServiceMonitor for Velero
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: velero
  namespace: velero
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: velero
  endpoints:
    - port: http-monitoring
      interval: 30s
---
# PrometheusRule for backup alerts
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backup-alerts
  namespace: monitoring
spec:
  groups:
    - name: backup.rules
      rules:
        # Velero 备份失败
        - alert: VeleroBackupFailed
          expr: |
            increase(velero_backup_failure_total[1h]) > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Velero backup failed"
            description: "Velero backup has failed in the last hour"

        # 备份超时
        - alert: VeleroBackupTooOld
          expr: |
            time() - velero_backup_last_successful_timestamp > 86400
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Velero backup is too old"
            description: "No successful backup in the last 24 hours"

        # etcd 快照失败
        - alert: EtcdBackupFailed
          expr: |
            etcd_backup_last_success_timestamp_seconds == 0
            or
            time() - etcd_backup_last_success_timestamp_seconds > 7200
          for: 10m
          labels:
            severity: critical
          annotations:
            summary: "etcd backup failed"
            description: "etcd backup has not succeeded in the last 2 hours"

        # 快照数量不足
        - alert: InsufficientBackups
          expr: |
            velero_backup_total < 7
          for: 1h
          labels:
            severity: warning
          annotations:
            summary: "Insufficient backups"
            description: "Less than 7 backups available"

        # 备份存储空间不足
        - alert: BackupStorageNearFull
          expr: |
            (backup_storage_bytes_used / backup_storage_bytes_total) > 0.85
          for: 30m
          labels:
            severity: warning
          annotations:
            summary: "Backup storage near full"
            description: "Backup storage is {{ $value | humanizePercentage }} full"
```

### 备份状态仪表盘

```yaml
# backup-dashboard-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backup-dashboard
  namespace: monitoring
  labels:
    grafana_dashboard: "1"
data:
  backup-dashboard.json: |
    {
      "title": "Kubernetes Backup Status",
      "panels": [
        {
          "title": "Backup Success Rate",
          "type": "gauge",
          "targets": [
            {
              "expr": "sum(velero_backup_success_total) / sum(velero_backup_total) * 100"
            }
          ]
        },
        {
          "title": "Last Successful Backup",
          "type": "stat",
          "targets": [
            {
              "expr": "time() - max(velero_backup_last_successful_timestamp)"
            }
          ]
        },
        {
          "title": "Backup History",
          "type": "table",
          "targets": [
            {
              "expr": "velero_backup_total"
            }
          ]
        },
        {
          "title": "etcd Backup Status",
          "type": "stat",
          "targets": [
            {
              "expr": "etcd_backup_last_success_timestamp_seconds > 0"
            }
          ]
        }
      ]
    }
```

## 备份策略最佳实践

### 策略设计清单

| 备份类型 | 频率 | 保留时间 | 存储位置 | 验证频率 |
|----------|------|----------|----------|----------|
| etcd 快照 | 每小时 | 7 天 | 本地 + 远程 | 每日 |
| Velero 备份 | 每小时 | 7 天 | 对象存储 | 每周 |
| 卷快照 | 每 6 小时 | 30 天 | 云提供商 | 每周 |
| 数据库逻辑备份 | 每日 | 90 天 | 对象存储 | 每月 |
| 全集群备份 | 每周 | 90 天 | 跨区域存储 | 每月 |

### 3-2-1 备份原则

```mermaid
graph TB
    subgraph backup_321["3-2-1 备份原则"]
        three["3 份副本"]
        two["2 种介质"]
        one["1 份异地"]
    end

    three --> copy1["生产数据"]
    three --> copy2["本地备份"]
    three --> copy3["远程备份"]

    two --> media1["本地磁盘/SSD"]
    two --> media2["对象存储/磁带"]

    one --> offsite["跨区域/跨云"]

    style three fill:#e8f5e9
    style two fill:#e3f2fd
    style one fill:#fff3e0
```

### 备份安全

```yaml
# backup-encryption.yaml

# 使用 KMS 加密备份
apiVersion: velero.io/v1
kind: BackupStorageLocation
metadata:
  name: encrypted-backup
  namespace: velero
spec:
  provider: aws
  objectStorage:
    bucket: my-encrypted-backups
    prefix: velero
  config:
    region: us-west-2
    # 启用服务器端加密
    serverSideEncryption: aws:kms
    kmsKeyId: arn:aws:kms:us-west-2:123456789:key/12345678-1234-1234-1234-123456789012
---
# S3 桶策略 - 强制加密
# {
#     "Version": "2012-10-17",
#     "Statement": [
#         {
#             "Sid": "DenyUnencryptedUploads",
#             "Effect": "Deny",
#             "Principal": "*",
#             "Action": "s3:PutObject",
#             "Resource": "arn:aws:s3:::my-encrypted-backups/*",
#             "Condition": {
#                 "StringNotEquals": {
#                     "s3:x-amz-server-side-encryption": "aws:kms"
#                 }
#             }
#         }
#     ]
# }
```

## 总结

完善的备份策略需要多层次、多维度的保护：

1. **etcd 备份**：定期快照，本地和远程双重保存，定期验证完整性
2. **资源备份**：使用 Velero 进行 Kubernetes 资源的声明式备份和恢复
3. **持久卷备份**：利用 CSI 快照实现应用一致性的卷备份
4. **应用备份**：针对数据库等有状态应用执行专门的逻辑备份
5. **监控告警**：建立完善的备份监控和告警机制，及时发现问题
6. **定期验证**：通过自动化测试验证备份的可恢复性

遵循 3-2-1 原则，结合自动化工具，能够构建可靠的备份体系，为灾难恢复提供坚实保障。
