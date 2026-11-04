# 06. StatefulSet, DaemonSet, Job, CronJob

[Index](README.md) | Prev: [05](05-deployment-replicaset.md) | Next: [07. Services](07-services-dns.md)

## E workload ki e controller

| Need | Controller |
|---|---|
| Stateless app (web, API), any pod any node | **Deployment** |
| Stable identity + own disk (DB, Kafka, ZooKeeper, Elasticsearch) | **StatefulSet** |
| Prathi node meeda okka copy (log agent, monitoring agent, CNI) | **DaemonSet** |
| Oka pani chesi complete (migration, batch) | **Job** |
| Schedule prakaram Job | **CronJob** |

## StatefulSet

### Deployment tho difference

| | Deployment | StatefulSet |
|---|---|---|
| Pod names | Random: `web-7c5d-x8k2p` | Ordered, stable: `db-0`, `db-1`, `db-2` |
| Start/stop order | Anni okesari | Default ga order lo: 0, 1, 2 start; 2, 1, 0 stop |
| Storage | Anni pods same PVC share (unte) | Prathi pod ki **own PVC** (`volumeClaimTemplates`) |
| Pod restart aithe | Kotha name, kotha identity | **Same name, same disk** malli attach |
| DNS | Service IP matrame | Prathi pod ki stable DNS: `db-0.db.ns.svc.cluster.local` |

### Example

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  clusterIP: None              # headless Service: prathi pod ki DNS record (07 lo detail)
  selector:
    app: db
  ports:
  - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db              # headless service name, DNS ki
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres:16
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef: { name: db-secret, key: password }
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # prathi pod ki separate PVC: data-db-0, data-db-1, ...
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: auto-ebs-sc
      resources:
        requests:
          storage: 10Gi
```

### Important points

- StatefulSet delete chesina, scale down chesina **PVCs delete avvavu** (data safety). Manual ga delete cheyyali, leda `persistentVolumeClaimRetentionPolicy` set cheyyi.
- `podManagementPolicy: Parallel` pedithe order lekunda anni okesari start (speed kosam, app support chesthe).
- 3 replicas ichhi "PostgreSQL cluster" ayipoyindi anukovaddu. StatefulSet identity + storage matrame isthundi. Replication, failover app/operator chesukovali (CloudNativePG, etc.). Production DB ki EKS lo RDS/Aurora vaadatame simple.
- EBS volume oka AZ ki bind. `db-0` PVC `us-east-1a` lo unte `db-0` pod eppudu `us-east-1a` node meeda ne schedule avthundi.

## DaemonSet

Prathi node (leda selected nodes) meeda exact ga okka pod. Kotha node add aithe automatic ga daani meeda pod vasthundi.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-agent
spec:
  selector:
    matchLabels:
      app: node-agent
  template:
    metadata:
      labels:
        app: node-agent
    spec:
      tolerations:                     # control-plane/tainted nodes meeda kuda run avvali ante
      - operator: Exists
      containers:
      - name: agent
        image: busybox:1.36
        command: ["sh", "-c", "while true; do echo $(hostname) alive; sleep 60; done"]
        resources:
          requests: { cpu: 10m, memory: 16Mi }
```

Use cases: Fluent Bit (logs), node-exporter / CloudWatch agent (metrics), CNI, CSI node plugins, security agents.
Konni nodes meeda matrame kavali ante `nodeSelector` / affinity ([13](13-scheduling-affinity-taints.md)).
Update strategy: `RollingUpdate` (default, `maxUnavailable: 1`) leda `OnDelete`.

## Job

Pod run ayyi **success tho complete** avvali. Fail aithe retry.

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  completions: 1                 # entha successful pods kavali
  parallelism: 1                 # okesari entha run avvali
  backoffLimit: 3                # inni sarlu fail aithe Job Failed
  activeDeadlineSeconds: 600     # total time limit
  ttlSecondsAfterFinished: 3600  # complete ayyaka 1h ki auto-delete
  template:
    spec:
      restartPolicy: OnFailure   # Job lo Never leda OnFailure matrame
      containers:
      - name: migrate
        image: busybox:1.36
        command: ["sh", "-c", "echo migrating; sleep 5; echo done"]
```

Patterns:
- `completions: 10, parallelism: 3`: 10 tasks, 3 okesari.
- `completionMode: Indexed`: prathi pod ki `JOB_COMPLETION_INDEX` env (0..N-1), work divide cheyyadaniki.

```bash
kubectl get jobs
kubectl logs job/db-migrate
kubectl wait --for=condition=complete job/db-migrate --timeout=120s
```

## CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: nightly-report
spec:
  schedule: "0 2 * * *"            # min hour day month weekday -> roju 02:00
  timeZone: "Asia/Kolkata"         # ivvakapothe kube-controller-manager timezone (EKS lo UTC)
  concurrencyPolicy: Forbid        # previous run inka run avthunte kotha di skip. Allow / Replace kuda
  startingDeadlineSeconds: 300
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: report
            image: busybox:1.36
            command: ["sh", "-c", "date; echo report generated"]
```

Cron format quick ref:

```
*/5 * * * *    prathi 5 min
0 * * * *      prathi gantaku
0 9 * * 1-5    weekdays 9 AM
0 0 1 * *      prathi nela 1st
```

Manual ga test run:

```bash
kubectl create job manual-run --from=cronjob/nightly-report
```

## Interview one-liners

- StatefulSet enduku: stable network identity, stable storage per pod, ordered rollout.
- StatefulSet delete chesthe data: PVCs default ga undipothayi.
- DaemonSet scheduling: prathi eligible node ki okka pod; taints unte tolerations kavali.
- Job vs CronJob: Job one-time; CronJob schedule meeda Jobs create chesthundi.
- Job lo `restartPolicy: Always` enduku kudaradu: complete avvalsina pani, Always aithe eppatiki complete avvadu.

Lab: [labs/README.md#lab-04](labs/README.md#lab-04-job-cronjob-daemonset)
