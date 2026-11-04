# 05. Deployment, ReplicaSet

[Index](README.md) | Prev: [04](04-labels-selectors-annotations.md) | Next: [06. Other workloads](06-statefulset-daemonset-job-cronjob.md)

## Hierarchy

```
Deployment  (rollout strategy, history, rollback)
   └── ReplicaSet  (N copies eppudu undali)
          └── Pods
```

- **ReplicaSet**: "ee template tho N pods eppudu run avvali" ani chusukuntundi. Anthe. Version change, rollout logic ledu.
- **Deployment**: ReplicaSets ni manage chesthundi. Template maarithe kotha ReplicaSet create chesi, old daanni slowly scale down chesthundi. Ade rolling update.

ReplicaSet ni direct ga create cheyyam. Eppudu Deployment vadu.

## Template lo `metadata.name` enduku undadu

Template okka pod kosam kaadu, **N pods ki blueprint**. Prathi pod ki unique name kavali, kabatti controller generate chesthundi:

```
Deployment:  web
ReplicaSet:  web-7c5ddbdf54           # deployment name + pod-template-hash
Pods:        web-7c5ddbdf54-x8k2p     # replicaset name + random 5 chars
             web-7c5ddbdf54-q9m4t
```

- `7c5ddbdf54` = pod template content hash. Template (image, env, etc.) maarithe hash maruthundi, kotha ReplicaSet create avthundi.
- Deployment ee hash ni `pod-template-hash` label ga pods ki, ReplicaSet selector ki add chesthundi. Ala old/new pods mix avvavu.
- Template lo `name` rasina error radhu, ignore avthundi.

Pods ni identify cheyyadaniki names kaadu, **labels** vadatham. Anduke template lo label mandatory, name kaadu.

Compare: standalone `kind: Pod` lo `metadata.name` mandatory, endukante adi okkate pod.

```bash
kubectl get rs,pods -l app=web --show-labels   # pod-template-hash label chudu
```

## Rollout ela trigger avthundi

**Pod template (`spec.template`) lo edaina marithe matrame** kotha rollout:
- image, env, resources, labels in template, annotations in template, volumes...

`replicas` marchithe rollout kaadu, just scale. Same ReplicaSet lo pods penchuthundi.

ConfigMap content marchithe pods restart avvavu (template maaraledu kabatti). Restart kavali ante:

```bash
kubectl rollout restart deployment/web   # template lo restartedAt annotation add chesi rollout trigger
```

## Strategies

### RollingUpdate (default)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%          # desired kante ekkuva extra pods entha varaku create cheyyochu
      maxUnavailable: 25%    # rollout lo entha mandi unavailable undochu
  minReadySeconds: 10        # pod Ready ayyaka inni seconds stable unte ne "available"
  progressDeadlineSeconds: 600   # inthalo progress lekapothe rollout "failed" ani mark
```

`replicas: 4`, `maxSurge: 1`, `maxUnavailable: 0` ante:
1. 1 kotha pod create (total 5).
2. Kotha pod **Ready** ayyaka 1 old pod delete (total 4).
3. Repeat.

Zero-downtime ki `maxUnavailable: 0` + correct **readinessProbe** must. Probe lekapothe container start avvagane Ready ani Kubernetes anukuntundi, app inka ready kaakapoyina traffic vasthundi.

### Recreate

```yaml
strategy:
  type: Recreate
```

Anni old pods delete, tarvata kotha pods. Downtime untundi. Rendu versions okesari run avvakudadu (DB schema conflict, single-writer volume) ante ide.

### Blue/green, canary

Deployment native ga support cheyyadu. Options:
- Rendu Deployments (`web-blue`, `web-green`), Service selector switch cheyyadam.
- Argo Rollouts, Flagger.
- Istio/Gateway API weight-based traffic split ([21](21-service-mesh-istio.md)).

## Rollout commands

```bash
kubectl set image deployment/web web=nginx:1.28        # image update (container name=image)
kubectl annotate deployment/web kubernetes.io/change-cause="nginx 1.28" --overwrite
kubectl rollout status deployment/web                  # complete ayye varaku wait
kubectl rollout history deployment/web
kubectl rollout history deployment/web --revision=2    # aa revision template details
kubectl rollout undo deployment/web                    # previous version ki
kubectl rollout undo deployment/web --to-revision=1
kubectl rollout pause deployment/web                   # multiple changes chesi okesari rollout
kubectl rollout resume deployment/web
kubectl scale deployment/web --replicas=5
```

Real projects lo `kubectl set image` kante YAML lo image marchi `kubectl apply` (leda GitOps) better. Lekapothe Git lo unna YAML, cluster lo unnadi match avvavu.

`revisionHistoryLimit` (default 10): entha old ReplicaSets unchali. Avi `replicas: 0` tho untayi, rollback ki use avthayi.

## Rollout stuck aithe

```bash
kubectl rollout status deployment/web     # "Waiting for deployment ... 1 of 3 updated replicas are available"
kubectl get rs -l app=web                 # kotha RS lo pods Ready avvatledu
kubectl describe deployment web           # Conditions: Progressing=False, ProgressDeadlineExceeded
kubectl get pods -l app=web               # kotha pods status (ImagePullBackOff? CrashLoop? not Ready?)
```

Common causes: wrong image tag, readiness probe fail, resources saripodam (`Pending`), config missing.
Fix cheyyi leda `kubectl rollout undo`.

## Deployment useful fields summary

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 3
  revisionHistoryLimit: 5
  minReadySeconds: 5
  progressDeadlineSeconds: 300
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 30
      containers:
      - name: web
        image: nginx:1.27
        ports:
        - containerPort: 80
        resources:
          requests: { cpu: 100m, memory: 128Mi }
          limits: { memory: 256Mi }
        readinessProbe:
          httpGet: { path: /, port: 80 }
          periodSeconds: 5
```

## Interview one-liners

- Deployment vs ReplicaSet: RS count maintain chesthundi; Deployment RS lanu manage chesi rolling update, rollback isthundi.
- Rollback ela pani chesthundi: old RS (replicas 0 tho unchinadi) ni malli scale up chesthundi.
- Zero downtime rollout ki em kavali: readinessProbe, `maxUnavailable: 0`, graceful shutdown (preStop + SIGTERM handling), PDB.
- ConfigMap marchithe pods restart avthaya: avvavu. `rollout restart` leda template lo config hash annotation.

Lab: [labs/README.md#lab-03](labs/README.md#lab-03-rolling-update-and-rollback)
