# Project 3: Stateful Data That Survives Failure

This Minikube lab runs a three-instance PostgreSQL cluster managed by
CloudNativePG (CNPG), stores PostgreSQL backups in in-cluster MinIO, tests a
point-in-time restore, and uses a hand-written Redis StatefulSet to explore
stable pod identities and per-pod storage.

> **Scope:** The Redis StatefulSet below gives each Redis pod its own persistent
> data directory. It does not configure Redis replication or Redis Cluster, so
> the three Redis pods are independent cache instances, not a highly available
> Redis cluster.

## Architecture

```mermaid
flowchart LR
    Client[Applications / psql]
    RW["orders-db-rw<br/>primary only"]
    RO["orders-db-ro<br/>replicas only"]
    R["orders-db-r<br/>any instance"]

    subgraph Data["data namespace"]
        subgraph CNPG["CloudNativePG-managed PostgreSQL cluster"]
            Primary["PostgreSQL primary"]
            Replica1["PostgreSQL replica"]
            Replica2["PostgreSQL replica"]
            PrimaryPVC[(Primary PVC)]
            ReplicaPVC1[(Replica PVC)]
            ReplicaPVC2[(Replica PVC)]
        end
        Scheduled["ScheduledBackup"]
        CNPGBackup["Base backups + WAL"]
        MinIO["MinIO S3-compatible storage"]
        DumpJob["Nightly pg_dump CronJob"]
        DumpPVC[(Logical dump PVC)]
        Seed["One-off seed Job"]
        RedisHeadless["redis headless Service"]
        Redis0["redis-0"]
        Redis1["redis-1"]
        Redis2["redis-2"]
        RedisPVC0[(redis-0 PVC)]
        RedisPVC1[(redis-1 PVC)]
        RedisPVC2[(redis-2 PVC)]
    end
    Operator["CNPG operator"]

    Client --> RW
    Client --> RO
    Client --> R
    RW --> Primary
    RO --> Replica1
    RO --> Replica2
    R --> Primary
    R --> Replica1
    R --> Replica2
    Operator -. watches Cluster CR .-> CNPG
    Seed --> RW
    Scheduled --> CNPGBackup --> MinIO
    DumpJob --> RW
    DumpJob --> DumpPVC
    RedisHeadless --> Redis0
    RedisHeadless --> Redis1
    RedisHeadless --> Redis2
    Redis0 --> RedisPVC0
    Redis1 --> RedisPVC1
    Redis2 --> RedisPVC2
```

![Project 3 architecture diagram: PostgreSQL, Redis, backups, and restore](architecture.png)

CNPG continuously manages PostgreSQL instances and failover. Scheduled base
backups and archived WAL go to MinIO for physical recovery/PITR; the separate
nightly `pg_dump` CronJob creates logical SQL dumps on a PVC.

## Prerequisites

- Minikube, `kubectl`, Helm, and `psql` (optional; database client pods also
  work)
- A Minikube driver with persistent-volume support and enough resources for
  MinIO, three PostgreSQL instances, and three Redis pods (for example 4 CPUs
  and 8 GiB RAM)

Start the cluster and create the project namespace:

```bash
minikube start --cpus=4 --memory=8192
kubectl create namespace data
```

Check that a default StorageClass exists before creating PVCs:

```bash
kubectl get storageclass
```

## 1. Install the CloudNativePG operator

Install the operator with Helm:

```bash
helm repo add cnpg https://cloudnative-pg.github.io/charts
helm repo update
helm upgrade --install cnpg cnpg/cloudnative-pg \
  --namespace cnpg-system --create-namespace
kubectl get deployments -n cnpg-system
kubectl wait --for=condition=Available deployment --all \
  -n cnpg-system --timeout=180s
kubectl get crd clusters.postgresql.cnpg.io
```

Use the operator's documentation for the installed version when adapting the
backup API; CNPG's backup configuration can change between releases.

## 2. Run MinIO and create a bucket

Create a local lab credential Secret. Replace these example values for any
shared or persistent environment; do not commit real credentials.

```bash
kubectl create secret generic minio-root \
  -n data \
  --from-literal=MINIO_ROOT_USER=minioadmin \
  --from-literal=MINIO_ROOT_PASSWORD='change-this-local-password'
kubectl create secret generic minio-s3-credentials \
  -n data \
  --from-literal=ACCESS_KEY_ID=minioadmin \
  --from-literal=ACCESS_SECRET_KEY='change-this-local-password'
```

Save as `minio.yaml` and apply:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: minio-data
  namespace: data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio
  namespace: data
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
        - name: minio
          image: quay.io/minio/minio:latest
          args:
            - server
            - /data
            - --console-address
            - ":9001"
          envFrom:
            - secretRef:
                name: minio-root
          ports:
            - name: s3
              containerPort: 9000
            - name: console
              containerPort: 9001
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: minio-data
---
apiVersion: v1
kind: Service
metadata:
  name: minio
  namespace: data
spec:
  selector:
    app: minio
  ports:
    - name: s3
      port: 9000
      targetPort: s3
    - name: console
      port: 9001
      targetPort: console
```

```bash
kubectl apply -f minio.yaml
kubectl rollout status deployment/minio -n data
```

Create a bucket named `postgres-backups` using MinIO's `mc` client. The
following temporary pod accesses the MinIO Service from inside the cluster:

```bash
kubectl run minio-mc -n data --rm -it --restart=Never \
  --image=minio/mc:latest \
  --env="MC_HOST_local=http://minioadmin:change-this-local-password@minio.data.svc.cluster.local:9000" \
  -- mc mb --ignore-existing local/postgres-backups
```

Use the same example credentials in the Secret for this local exercise. The
Secret key names and the CNPG `s3Credentials` references must match exactly.

## 3. Create the PostgreSQL cluster

Create an application credential Secret and then apply the CNPG Cluster:

```bash
kubectl create secret generic orders-db-app \
  -n data \
  --from-literal=username=app \
  --from-literal=password='change-this-app-password'
```

Save as `postgres-cluster.yaml`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: orders-db
  namespace: data
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17
  bootstrap:
    initdb:
      database: orders
      owner: app
      secret:
        name: orders-db-app
  storage:
    size: 2Gi
  backup:
    barmanObjectStore:
      destinationPath: s3://postgres-backups/orders-db
      endpointURL: http://minio.data.svc.cluster.local:9000
      s3Credentials:
        accessKeyId:
          name: minio-s3-credentials
          key: ACCESS_KEY_ID
        secretAccessKey:
          name: minio-s3-credentials
          key: ACCESS_SECRET_KEY
      wal:
        compression: gzip
      data:
        compression: gzip
```

```bash
kubectl apply -f postgres-cluster.yaml
kubectl get cluster,pod,service -n data -w
```

Wait until the Cluster reports healthy and all three instances are ready:

```bash
kubectl get cluster orders-db -n data
kubectl get pods -n data -l cnpg.io/cluster=orders-db
```

CNPG exposes three useful Services:

- `orders-db-rw`: routes to the current primary; use it for writes and
  application connections that need automatic primary failover.
- `orders-db-ro`: routes to replica instances; use it for read-only workloads
  that can tolerate replication lag.
- `orders-db-r`: routes to any cluster instance; it is not a write-primary
  endpoint.

Inspect the Services and their endpoint membership:

```bash
kubectl get svc -n data -l cnpg.io/cluster=orders-db
kubectl get endpointslice -n data \
  -l kubernetes.io/service-name=orders-db-rw -o wide
kubectl get endpointslice -n data \
  -l kubernetes.io/service-name=orders-db-ro -o wide
```

## 4. Seed sample order data with a Job

Save as `seed-job.yaml`:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: seed-orders
  namespace: data
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: psql
          image: postgres:17
          command:
            - /bin/sh
            - -ec
            - |
              psql -v ON_ERROR_STOP=1 <<'SQL'
              CREATE TABLE IF NOT EXISTS orders (
                id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
                customer text NOT NULL,
                amount numeric(10, 2) NOT NULL,
                created_at timestamptz NOT NULL DEFAULT now()
              );
              INSERT INTO orders (customer, amount)
              VALUES ('sample customer', 42.50);
              SQL
          env:
            - name: PGHOST
              value: orders-db-rw
            - name: PGDATABASE
              value: orders
            - name: PGUSER
              valueFrom:
                secretKeyRef:
                  name: orders-db-app
                  key: username
            - name: PGPASSWORD
              valueFrom:
                secretKeyRef:
                  name: orders-db-app
                  key: password
```

```bash
kubectl apply -f seed-job.yaml
kubectl wait --for=condition=complete job/seed-orders -n data --timeout=180s
kubectl logs job/seed-orders -n data
```

## 5. Schedule physical backups to MinIO

CNPG archives WAL to the configured object store. A `ScheduledBackup` starts
periodic base backups; both the base backup and the WAL archive are needed for
point-in-time recovery. Save as `scheduled-backup.yaml`:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: ScheduledBackup
metadata:
  name: orders-db-daily
  namespace: data
spec:
  schedule: "0 0 2 * * *"
  backupOwnerReference: self
  cluster:
    name: orders-db
  method: barmanObjectStore
```

The schedule uses six fields (seconds, minutes, hours, day, month, weekday):
this example runs daily at 02:00. Apply it:

```bash
kubectl apply -f scheduled-backup.yaml
kubectl get scheduledbackup,backup -n data
```

For a lab, trigger an immediate backup instead of waiting for the schedule:

```bash
kubectl apply -f - <<'YAML'
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: orders-db-manual-001
  namespace: data
spec:
  method: barmanObjectStore
  cluster:
    name: orders-db
YAML
kubectl get backup -n data -w
```

Give each manual Backup a unique name. Check its status and the MinIO bucket
before attempting a restore. Do not treat a ScheduledBackup object alone as
proof that a backup completed.

## 6. Nightly logical dump to a PVC

The dump PVC is separate from the CNPG physical backups. A PVC inside the same
cluster is useful for the exercise but is not an off-cluster disaster-recovery
copy.

Save as `dump-pvc.yaml` and apply:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: orders-dumps
  namespace: data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

Save as `dump-cronjob.yaml`:

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: orders-db-dump
  namespace: data
spec:
  schedule: "0 2 * * *"
  timeZone: Etc/UTC
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: pg-dump
              image: postgres:17
              command:
                - /bin/sh
                - -ec
                - |
                  stamp="$(date -u +%Y%m%dT%H%M%SZ)"
                  pg_dump --format=custom \
                    --file="/backups/orders-${stamp}.dump"
                  find /backups -type f -name 'orders-*.dump' \
                    -mtime +7 -delete
              env:
                - name: PGHOST
                  value: orders-db-rw
                - name: PGDATABASE
                  value: orders
                - name: PGUSER
                  valueFrom:
                    secretKeyRef:
                      name: orders-db-app
                      key: username
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: orders-db-app
                      key: password
              volumeMounts:
                - name: backups
                  mountPath: /backups
          volumes:
            - name: backups
              persistentVolumeClaim:
                claimName: orders-dumps
```

Apply and run a test backup now:

```bash
kubectl apply -f dump-pvc.yaml
kubectl apply -f dump-cronjob.yaml
kubectl create job --from=cronjob/orders-db-dump orders-db-dump-test -n data
kubectl wait --for=condition=complete job/orders-db-dump-test \
  -n data --timeout=180s
kubectl logs job/orders-db-dump-test -n data
```

The CronJob uses `concurrencyPolicy: Forbid` to avoid overlapping dumps and
limits retained Job history. The script keeps the most recent seven days of
dump files on the PVC.

## 7. Restore to a new cluster and verify data

Record a target timestamp after the seed Job and a successful backup/WAL
archival. Use UTC RFC3339 time and choose a point at or after the seed
transaction but no later than the available archived WAL. Confirm the restore
API fields for your installed CNPG version.

Save as `orders-db-restore.yaml`, substituting the desired timestamp:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: orders-db-restore
  namespace: data
spec:
  instances: 3
  imageName: ghcr.io/cloudnative-pg/postgresql:17
  storage:
    size: 2Gi
  bootstrap:
    recovery:
      source: orders-db-backup
      recoveryTarget:
        targetTime: "REPLACE_WITH_UTC_TARGET_TIME"
  externalClusters:
    - name: orders-db-backup
      barmanObjectStore:
        destinationPath: s3://postgres-backups/orders-db
        endpointURL: http://minio.data.svc.cluster.local:9000
        serverName: orders-db
        s3Credentials:
          accessKeyId:
            name: minio-s3-credentials
            key: ACCESS_KEY_ID
          secretAccessKey:
            name: minio-s3-credentials
            key: ACCESS_SECRET_KEY
```

Replace the timestamp with a real UTC RFC3339 time from your own run; the
placeholder is not a valid target. Do not restore over the source cluster;
this creates a separate Cluster and separate PVCs.

```bash
kubectl apply -f orders-db-restore.yaml
kubectl get cluster,pods -n data -w
```

After the restored cluster is healthy, query its read/write Service to prove
the seeded schema and rows were recovered. CNPG's generated app Secret for the
new cluster is `orders-db-restore-app`:

```bash
kubectl run restore-check -n data --rm -i --restart=Never \
  --image=postgres:17 \
  --env="PGPASSWORD=$(kubectl get secret orders-db-restore-app -n data -o jsonpath='{.data.password}' | base64 -d)" \
  -- psql -h orders-db-restore-rw -U app -d orders \
  -c 'SELECT id, customer, amount FROM orders ORDER BY id;'
```

Record the restore target, the resulting Cluster status, and the query output
as evidence that the procedure worked.

## 8. Prove primary failover

First write a marker row through the stable `orders-db-rw` Service:

```bash
kubectl run failover-before -n data --rm -i --restart=Never \
  --image=postgres:17 \
  --env="PGPASSWORD=$(kubectl get secret orders-db-app -n data -o jsonpath='{.data.password}' | base64 -d)" \
  -- psql -h orders-db-rw -U app -d orders -v ON_ERROR_STOP=1 \
  -c "INSERT INTO orders (customer, amount) VALUES ('before failover', 1.00);"
```

Delete the primary pod and watch CNPG elect/promote a healthy instance:

```bash
kubectl get pods -n data -l cnpg.io/cluster=orders-db \
  -L cnpg.io/instanceRole -o wide
kubectl delete pod -n data \
  -l cnpg.io/cluster=orders-db,cnpg.io/instanceRole=primary
kubectl get pods,cluster -n data -w
```

Once the new primary is ready, write and read through `orders-db-rw` again.
This proves clients can continue to use the same Service name after failover:

```bash
kubectl run failover-after -n data --rm -i --restart=Never \
  --image=postgres:17 \
  --env="PGPASSWORD=$(kubectl get secret orders-db-app -n data -o jsonpath='{.data.password}' | base64 -d)" \
  -- psql -h orders-db-rw -U app -d orders -v ON_ERROR_STOP=1 \
  -c "INSERT INTO orders (customer, amount) VALUES ('after failover', 2.00);" \
  -c "SELECT id, customer, amount FROM orders ORDER BY id;"
```

Allow a short reconnection interval during promotion; an in-flight connection
may fail, but reconnecting clients should resume through `-rw`.

## 9. Redis StatefulSet with per-pod PVCs

Save as `redis.yaml`. `serviceName` points at a headless Service, and the
StatefulSet's default `OrderedReady` pod management creates `redis-0`, then
`redis-1`, then `redis-2`. Each pod receives its own PVC from the claim
template.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: redis
  namespace: data
spec:
  clusterIP: None
  selector:
    app: redis
  ports:
    - name: redis
      port: 6379
      targetPort: redis
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
  namespace: data
spec:
  serviceName: redis
  replicas: 3
  podManagementPolicy: OrderedReady
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
          image: redis:7.4
          command:
            - redis-server
            - --appendonly
            - "yes"
          ports:
            - name: redis
              containerPort: 6379
          volumeMounts:
            - name: redis-data
              mountPath: /data
  volumeClaimTemplates:
    - metadata:
        name: redis-data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
```

```bash
kubectl apply -f redis.yaml
kubectl rollout status statefulset/redis -n data
kubectl get pods,pvc -n data -l app=redis
```

The stable DNS name of Redis ordinal zero is
`redis-0.redis.data.svc.cluster.local`. Scale down and up to check pod identity
and PVC retention:

```bash
kubectl scale statefulset redis -n data --replicas=1
kubectl get pods,pvc -n data -l app=redis
kubectl scale statefulset redis -n data --replicas=3
kubectl rollout status statefulset/redis -n data
kubectl get pods,pvc -n data -l app=redis
```

After scaling back up, `redis-1` and `redis-2` should reattach their existing
PVCs. StatefulSet scale-down does not normally delete those claims; actual
retention also depends on the StatefulSet PVC retention policy and the
underlying StorageClass/PV reclaim policy.

## Break-it drills

- **Undersized PostgreSQL PVC:** In a disposable cluster, reduce the requested
  storage and generate data until the volume fills. Observe pod/database
  symptoms and PVC capacity; recover by expanding storage only if the
  StorageClass supports it, or rebuild the lab from a verified backup.
- **Delete a replica PVC:** In a disposable cluster, identify a non-primary
  instance and its PVC, delete that claim, then observe CNPG recreate the
  instance and resynchronize it. Never delete the primary's only data volume in
  a non-disposable environment.
- **Let CronJobs overlap:** In a disposable copy, change the schedule to every
  minute and `concurrencyPolicy: Allow`, then slow the dump command and observe
  concurrent Jobs. Restore `Forbid` after the experiment.
- **Redis storage:** Scale the StatefulSet down, inspect retained claims, then
  scale it up and verify the same ordinal-to-claim association.

## Cleanup

These commands delete the data namespace, including database data, Redis data,
and MinIO backup objects:

```bash
kubectl delete namespace data
helm uninstall cnpg -n cnpg-system
kubectl delete namespace cnpg-system
minikube stop
```

PV/PVC deletion and reclaim behavior depends on the cluster StorageClass. Check
the PV reclaim policy before cleanup if you need to preserve data.

## Learning outcomes

- StatefulSet identity, ordered startup, headless Services, and per-pod storage
- PVC retention and storage reclaim behavior
- CNPG's operator pattern, PostgreSQL roles, Services, and failover
- The difference between physical base/WAL backups and logical `pg_dump`
- Jobs, CronJobs, retry controls, concurrency, and history limits
- A tested restore and failover procedure rather than an unverified backup
