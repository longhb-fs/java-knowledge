# Day 25: Kubernetes Storage

## Mục tiêu học tập
- Hiểu Kubernetes storage architecture
- Cấu hình PersistentVolumes và PersistentVolumeClaims
- Sử dụng StorageClasses cho dynamic provisioning
- Deploy stateful applications với StatefulSets
- Implement backup và recovery strategies

## 1. Kubernetes Storage Concepts

### 1.1 Storage Architecture

```
Kubernetes Storage Architecture
═══════════════════════════════

┌─────────────────────────────────────────────────────────────────────┐
│                          POD                                         │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                      CONTAINERS                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │   │
│  │  │   App       │  │   App       │  │  Sidecar    │         │   │
│  │  │ Container 1 │  │ Container 2 │  │  Container  │         │   │
│  │  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘         │   │
│  │         │                │                │                  │   │
│  │         └────────────────┼────────────────┘                  │   │
│  │                          │                                   │   │
│  │                    ┌─────┴─────┐                            │   │
│  │                    │  VOLUMES  │                            │   │
│  │                    └─────┬─────┘                            │   │
│  └──────────────────────────│──────────────────────────────────┘   │
│                              │                                       │
└──────────────────────────────│───────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                 PERSISTENT VOLUME CLAIM (PVC)                        │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  - Storage request from pod                                  │   │
│  │  - Namespace scoped                                          │   │
│  │  - Binds to PV                                               │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               │ Binds
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   PERSISTENT VOLUME (PV)                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  - Cluster resource (cluster scoped)                         │   │
│  │  - Provisioned by admin or dynamically                       │   │
│  │  - Has lifecycle independent of pod                          │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               │ Backed by
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      STORAGE BACKEND                                 │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐       │
│  │   AWS     │  │   GCP     │  │   Azure   │  │   NFS     │       │
│  │   EBS     │  │   PD      │  │   Disk    │  │   Server  │       │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘       │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Volume Types

```
Volume Types Overview
═════════════════════

┌───────────────────┬─────────────────────────────────────────────────┐
│     Volume Type   │              Description                        │
├───────────────────┼─────────────────────────────────────────────────┤
│ emptyDir          │ Temporary, deleted when pod removed             │
├───────────────────┼─────────────────────────────────────────────────┤
│ hostPath          │ Mount host filesystem path (testing only)       │
├───────────────────┼─────────────────────────────────────────────────┤
│ configMap         │ Mount ConfigMap as files                        │
├───────────────────┼─────────────────────────────────────────────────┤
│ secret            │ Mount Secret as files                           │
├───────────────────┼─────────────────────────────────────────────────┤
│ persistentVolume  │ Persistent storage via PVC                      │
│ Claim             │                                                 │
├───────────────────┼─────────────────────────────────────────────────┤
│ nfs               │ Network File System share                       │
├───────────────────┼─────────────────────────────────────────────────┤
│ awsElasticBlock   │ AWS EBS volume                                  │
│ Store             │                                                 │
├───────────────────┼─────────────────────────────────────────────────┤
│ gcePersistentDisk │ GCP Persistent Disk                             │
├───────────────────┼─────────────────────────────────────────────────┤
│ azureDisk         │ Azure Managed Disk                              │
└───────────────────┴─────────────────────────────────────────────────┘
```

## 2. Persistent Volumes and Claims

### 2.1 PersistentVolume (Static Provisioning)

```yaml
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
  labels:
    app: mysql
    type: local-storage
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce       # RWO: Single node read-write
  persistentVolumeReclaimPolicy: Retain  # Retain, Delete, Recycle
  storageClassName: local-storage
  # Local volume example
  local:
    path: /mnt/data/mysql
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - worker-node-1
---
# AWS EBS volume example
apiVersion: v1
kind: PersistentVolume
metadata:
  name: ebs-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: gp3
  awsElasticBlockStore:
    volumeID: vol-0123456789abcdef0
    fsType: ext4
---
# NFS volume example
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany        # RWX: Multiple nodes read-write
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: 10.0.0.100
    path: /exports/data
```

### 2.2 PersistentVolumeClaim

```yaml
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: production
  labels:
    app: mysql
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-storage
  resources:
    requests:
      storage: 20Gi
  selector:
    matchLabels:
      app: mysql
      type: local-storage
---
# Dynamic provisioning PVC (no selector needed)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3  # StorageClass handles provisioning
  resources:
    requests:
      storage: 50Gi
```

### 2.3 Using PVC in Pod

```yaml
# pod-with-pvc.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql
  namespace: production
spec:
  containers:
    - name: mysql
      image: mysql:8.0
      env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: root-password
      ports:
        - containerPort: 3306
      volumeMounts:
        - name: mysql-data
          mountPath: /var/lib/mysql
        - name: mysql-config
          mountPath: /etc/mysql/conf.d
          readOnly: true
      resources:
        requests:
          memory: "1Gi"
          cpu: "500m"
        limits:
          memory: "2Gi"
          cpu: "2000m"
  volumes:
    - name: mysql-data
      persistentVolumeClaim:
        claimName: mysql-pvc
    - name: mysql-config
      configMap:
        name: mysql-config
```

## 3. StorageClasses

### 3.1 AWS EBS StorageClass

```yaml
# storage-class-aws.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer  # Topology-aware
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  # kmsKeyId: "arn:aws:kms:..." # Optional: Custom KMS key
---
# High-performance SSD
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: io2
provisioner: ebs.csi.aws.com
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
parameters:
  type: io2
  iops: "10000"
  encrypted: "true"
```

### 3.2 GCP StorageClass

```yaml
# storage-class-gcp.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: standard-rwo
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
allowVolumeExpansion: true
parameters:
  type: pd-standard
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-rwo
provisioner: pd.csi.storage.gke.io
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Retain
parameters:
  type: pd-ssd
```

### 3.3 NFS StorageClass (with provisioner)

```yaml
# storage-class-nfs.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-client
provisioner: cluster.local/nfs-subdir-external-provisioner
reclaimPolicy: Delete
volumeBindingMode: Immediate
parameters:
  pathPattern: "${.PVC.namespace}/${.PVC.name}"
  onDelete: delete
mountOptions:
  - hard
  - nfsvers=4.1
```

## 4. StatefulSets

### 4.1 StatefulSet with VolumeClaimTemplates

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: production
spec:
  serviceName: mysql-headless  # Required for stable network IDs
  replicas: 3
  podManagementPolicy: OrderedReady  # or Parallel
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      terminationGracePeriodSeconds: 30
      initContainers:
        - name: init-mysql
          image: mysql:8.0
          command:
            - bash
            - "-c"
            - |
              set -ex
              # Generate server-id from pod ordinal
              [[ $(hostname) =~ -([0-9]+)$ ]] || exit 1
              ordinal=${BASH_REMATCH[1]}
              echo "[mysqld]" > /mnt/conf.d/server-id.cnf
              echo "server-id=$((100 + ordinal))" >> /mnt/conf.d/server-id.cnf
              # Copy appropriate config based on ordinal
              if [[ $ordinal -eq 0 ]]; then
                cp /mnt/config-map/primary.cnf /mnt/conf.d/
              else
                cp /mnt/config-map/replica.cnf /mnt/conf.d/
              fi
          volumeMounts:
            - name: conf
              mountPath: /mnt/conf.d
            - name: config-map
              mountPath: /mnt/config-map
      containers:
        - name: mysql
          image: mysql:8.0
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: root-password
          ports:
            - name: mysql
              containerPort: 3306
          volumeMounts:
            - name: data
              mountPath: /var/lib/mysql
            - name: conf
              mountPath: /etc/mysql/conf.d
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
          livenessProbe:
            exec:
              command:
                - mysqladmin
                - ping
                - -uroot
                - -p${MYSQL_ROOT_PASSWORD}
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            exec:
              command:
                - mysql
                - -h
                - "127.0.0.1"
                - -uroot
                - -p${MYSQL_ROOT_PASSWORD}
                - -e
                - "SELECT 1"
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: conf
          emptyDir: {}
        - name: config-map
          configMap:
            name: mysql-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 50Gi
---
# Headless service for StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: mysql-headless
  namespace: production
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
---
# Regular service for client connections
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: production
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
    - name: mysql
      port: 3306
```

### 4.2 Redis Cluster StatefulSet

```yaml
# redis-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: production
spec:
  serviceName: redis-headless
  replicas: 6  # 3 masters + 3 replicas
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          command:
            - redis-server
          args:
            - /etc/redis/redis.conf
            - --cluster-enabled
            - "yes"
            - --cluster-config-file
            - /data/nodes.conf
          ports:
            - name: redis
              containerPort: 6379
            - name: cluster-bus
              containerPort: 16379
          volumeMounts:
            - name: data
              mountPath: /data
            - name: config
              mountPath: /etc/redis
          resources:
            requests:
              cpu: "100m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "1Gi"
          livenessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 5
            periodSeconds: 5
      volumes:
        - name: config
          configMap:
            name: redis-config
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: gp3
        resources:
          requests:
            storage: 10Gi
```

## 5. Volume Snapshots and Backup

### 5.1 VolumeSnapshot Configuration

```yaml
# volume-snapshot-class.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Retain
parameters:
  # Optional: Add tags to snapshots
  # tagSpecification_1: "backup=true"
---
# Create a snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-snapshot-20240115
  namespace: production
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: data-mysql-0
---
# Restore from snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-restored
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp3
  resources:
    requests:
      storage: 50Gi
  dataSource:
    name: mysql-snapshot-20240115
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

### 5.2 Backup CronJob

```yaml
# backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
  namespace: production
spec:
  schedule: "0 2 * * *"  # Every day at 2 AM
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 3
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: mysql:8.0
              env:
                - name: MYSQL_HOST
                  value: mysql-0.mysql-headless
                - name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: mysql-secret
                      key: root-password
                - name: BACKUP_DATE
                  value: "$(date +%Y%m%d_%H%M%S)"
              command:
                - /bin/bash
                - -c
                - |
                  set -e
                  BACKUP_FILE="/backup/mysql_backup_${BACKUP_DATE}.sql.gz"
                  echo "Starting backup to ${BACKUP_FILE}"
                  mysqldump -h ${MYSQL_HOST} -uroot -p${MYSQL_ROOT_PASSWORD} \
                    --all-databases \
                    --single-transaction \
                    --routines \
                    --triggers \
                    --events | gzip > ${BACKUP_FILE}
                  echo "Backup completed: $(ls -lh ${BACKUP_FILE})"

                  # Upload to S3
                  aws s3 cp ${BACKUP_FILE} s3://my-backup-bucket/mysql/

                  # Cleanup old local backups
                  find /backup -mtime +7 -delete
                  echo "Cleanup completed"
              volumeMounts:
                - name: backup-volume
                  mountPath: /backup
          volumes:
            - name: backup-volume
              persistentVolumeClaim:
                claimName: backup-pvc
```

### 5.3 Velero Backup Configuration

```yaml
# velero-backup-schedule.yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: production-daily-backup
  namespace: velero
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  template:
    includedNamespaces:
      - production
    excludedResources:
      - events
      - events.events.k8s.io
    snapshotVolumes: true
    storageLocation: default
    volumeSnapshotLocations:
      - default
    ttl: 720h  # 30 days retention
---
# Manual backup
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: production-manual-20240115
  namespace: velero
spec:
  includedNamespaces:
    - production
  labelSelector:
    matchLabels:
      backup: "true"
  snapshotVolumes: true
  storageLocation: default
  volumeSnapshotLocations:
    - default
```

## 6. Volume Expansion

### 6.1 Expanding PVC

```yaml
# Ensure StorageClass allows expansion
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-gp3
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true  # Required for expansion
parameters:
  type: gp3
```

```bash
# Expand PVC (patch the size)
kubectl patch pvc mysql-pvc -n production -p '{"spec":{"resources":{"requests":{"storage":"100Gi"}}}}'

# Verify expansion
kubectl get pvc mysql-pvc -n production -o yaml | grep -A 2 capacity

# For some storage classes, pod restart may be required
kubectl delete pod mysql-0 -n production
```

## 7. Storage Best Practices

### 7.1 Java Application with Persistent Storage

```yaml
# java-app-with-storage.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      securityContext:
        fsGroup: 1000  # Group for volume access
      containers:
        - name: order-service
          image: order-service:1.0.0
          securityContext:
            runAsUser: 1000
            runAsGroup: 1000
          volumeMounts:
            # Temporary storage (ephemeral)
            - name: tmp
              mountPath: /tmp
            # Logs (optional persistent)
            - name: logs
              mountPath: /app/logs
            # Config from ConfigMap
            - name: config
              mountPath: /app/config
              readOnly: true
            # Secrets
            - name: secrets
              mountPath: /app/secrets
              readOnly: true
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
              ephemeral-storage: "500Mi"  # Ephemeral storage request
            limits:
              memory: "1Gi"
              cpu: "1000m"
              ephemeral-storage: "1Gi"    # Ephemeral storage limit
      volumes:
        - name: tmp
          emptyDir:
            sizeLimit: 500Mi
        - name: logs
          emptyDir: {}  # Or PVC for persistent logs
        - name: config
          configMap:
            name: order-service-config
        - name: secrets
          secret:
            secretName: order-service-secrets
            defaultMode: 0400  # Read-only for owner
```

### 7.2 Shared Storage for Multiple Pods

```yaml
# shared-storage.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
  namespace: production
spec:
  accessModes:
    - ReadWriteMany  # RWX for shared access
  storageClassName: efs  # EFS, NFS, or similar RWX storage
  resources:
    requests:
      storage: 100Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: file-processor
  namespace: production
spec:
  replicas: 5  # Multiple pods share the volume
  selector:
    matchLabels:
      app: file-processor
  template:
    spec:
      containers:
        - name: processor
          image: file-processor:1.0.0
          volumeMounts:
            - name: shared
              mountPath: /data
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: shared-data
```

## 8. Hands-on Exercises

### Exercise 1: Deploy MySQL with Persistent Storage
Set up MySQL with StatefulSet and persistent storage.

```bash
# Requirements:
# 1. Create StorageClass
# 2. Create StatefulSet with 3 replicas
# 3. Configure volumeClaimTemplates
# 4. Test data persistence across pod restarts
# 5. Scale up and verify data
```

### Exercise 2: Implement Backup Strategy
Create automated backup solution.

```bash
# Requirements:
# 1. Create backup CronJob
# 2. Configure VolumeSnapshots
# 3. Test restore from snapshot
# 4. Set up retention policy
```

### Exercise 3: Volume Expansion
Practice expanding storage dynamically.

```bash
# Requirements:
# 1. Create expandable StorageClass
# 2. Provision PVC with 10Gi
# 3. Expand to 20Gi
# 4. Verify expansion without downtime
```

### Exercise 4: Shared Storage Setup
Configure shared storage for multiple pods.

```bash
# Requirements:
# 1. Set up NFS or EFS StorageClass
# 2. Create RWX PVC
# 3. Deploy multiple pods sharing the volume
# 4. Test concurrent read/write
```

## 9. Key Takeaways

### Storage Concepts
1. **PV/PVC**: Decouple storage from pods
2. **StorageClass**: Dynamic provisioning
3. **StatefulSet**: Stable storage identity
4. **VolumeSnapshot**: Point-in-time backups

### Access Modes
1. **ReadWriteOnce (RWO)**: Single node mount
2. **ReadOnlyMany (ROX)**: Many nodes read-only
3. **ReadWriteMany (RWX)**: Many nodes read-write

### Best Practices
1. **Use StorageClass**: Enable dynamic provisioning
2. **Set Resource Limits**: Include ephemeral storage
3. **Regular Backups**: Automate with CronJobs or Velero
4. **Volume Expansion**: Enable in StorageClass

### Security
1. **fsGroup**: Control volume ownership
2. **Read-only mounts**: For configs and secrets
3. **Encryption**: Enable at-rest encryption
4. **Least privilege**: Minimal permissions

---

## Navigation

[← Day 24: Kubernetes Networking](./day-24-kubernetes-networking.md) | [Day 26: Kubernetes Security →](./day-26-kubernetes-security.md)

[Back to Overview](./00-overview.md)
