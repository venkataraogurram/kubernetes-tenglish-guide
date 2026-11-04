# 14. Autoscaling: HPA, VPA, Cluster Autoscaler, Karpenter

[Index](README.md) | Prev: [13](13-scheduling-affinity-taints.md) | Next: [15. Networking](15-networking-deep-dive.md)

Prerequisites: [11 requests/limits](11-resources-qos-probes.md), [13 scheduling](13-scheduling-affinity-taints.md)

## 3 levels of scaling

```
Pod count     -> HPA (Horizontal Pod Autoscaler)     "ekkuva copies"
Pod size      -> VPA (Vertical Pod Autoscaler)       "pedda pods"
Node count    -> Cluster Autoscaler / Karpenter      "ekkuva machines"
```

Chain ela pani chesthundi:

```
Traffic perigindi -> CPU up -> HPA replicas 3 -> 8
  -> kotha pods ki nodes lo space ledu -> Pending
  -> Karpenter/Auto Mode Pending pods chusi kotha node launch
  -> pods schedule
Traffic thaggindi -> HPA replicas down -> nodes khaali
  -> Karpenter consolidation: underused nodes remove
```

## HPA

### Prerequisite: metrics-server

```bash
kubectl top pods      # pani chesthe metrics-server undi
```

EKS Auto Mode clusters lo metrics API available. Standard EKS lo metrics-server add-on install cheyyali.

### Example

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70        # REQUESTS lo 70%
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300 # 5 min stable ga thakkuva unte ne scale down
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
```

Quick create:

```bash
kubectl autoscale deployment web --cpu-percent=70 --min=2 --max=10
kubectl get hpa web -w
kubectl describe hpa web
```

### Formula

```
desiredReplicas = ceil( currentReplicas * currentMetric / targetMetric )
```

3 pods, average CPU 140% of request, target 70% -> ceil(3 * 140/70) = 6.

### HPA mistakes

- **Pod ki CPU requests lekapothe** utilization calculate cheyyalemu: `TARGETS <unknown>/70%`. Requests must.
- Deployment lo `replicas: 3` + HPA: GitOps/`kubectl apply` prathi sari replicas 3 ki reset. HPA vadithe manifest nundi `replicas` teesey.
- Memory-based HPA: chala apps memory release cheyyavu, scale down avvadu.
- Scale-up time = metrics delay (~15-60s) + pod start + node launch (Karpenter ~1 min). Spiky traffic ki minReplicas sariga pettu.

### Custom / external metrics

Requests per second, queue length lanti metrics ki:
- **KEDA**: SQS queue depth, Kafka lag, Prometheus query, cron batti scale. Zero ki kuda scale.
- Prometheus Adapter: Prometheus metrics ni HPA ki.

```yaml
# KEDA example: SQS messages batti
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker
spec:
  scaleTargetRef:
    name: worker
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/<account>/jobs
      queueLength: "5"          # prathi pod ki 5 messages
      awsRegion: us-east-1
    authenticationRef:
      name: keda-aws            # TriggerAuthentication, Pod Identity tho
```

## VPA

Pods actual usage chusi **requests recommend/set** chesthundi. Separate install (CRDs + components).

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: web
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web
  updatePolicy:
    updateMode: "Off"       # recommendations matrame. "Recreate"/"InPlaceOrRecreate" auto apply
```

```bash
kubectl describe vpa web     # Recommendation: Target cpu 120m memory 300Mi
```

- Same metric (CPU) meeda HPA + VPA rendu vaddu, fight chesthayi.
- Best use: `Off` mode lo right-sizing recommendations teesukuni manifests lo requests update.
- Kubernetes kotha versions lo **in-place pod resize** (restart lekunda resources marchadam) undi, VPA `InPlaceOrRecreate` daanni vadukuntundi.

## Node autoscaling

### Cluster Autoscaler (old way)

- Node groups (ASG) meeda pani chesthundi. Pending pods unte ASG desired penchuthundi.
- Prathi node group lo fixed instance type(s). Chala node groups manage cheyyali.

### Karpenter (EKS Auto Mode lopala idi)

- Node groups levu. **Pending pods requirements** (CPU, memory, arch, zone, spot/on-demand) chusi **best fit EC2 instance** direct ga launch.
- **Consolidation**: underused nodes ni teesesi pods ni chinna/takkuva nodes ki move, cost save.
- **Drift / expiry**: config maarithe leda `expireAfter` daatithe nodes replace.

NodePool (mee Auto Mode cluster lo `general-purpose`, `system` untayi):

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: spot-arm
spec:
  template:
    spec:
      nodeClassRef:
        group: eks.amazonaws.com
        kind: NodeClass
        name: default
      requirements:
      - key: karpenter.sh/capacity-type
        operator: In
        values: ["spot", "on-demand"]
      - key: kubernetes.io/arch
        operator: In
        values: ["arm64"]
      - key: eks.amazonaws.com/instance-category
        operator: In
        values: ["c", "m", "r"]
      expireAfter: 336h
  limits:
    cpu: "100"                       # ee pool total max 100 vCPU
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 1m
```

```bash
kubectl get nodepools
kubectl get nodeclaims             # Karpenter launch chesina nodes
kubectl describe nodeclaim <name>  # e instance type, enduku
```

Consolidation valla pods move avthayi. Anduke **PDB + replicas >= 2 + graceful shutdown** important ([13](13-scheduling-affinity-taints.md)).
`karpenter.sh/do-not-disrupt: "true"` pod annotation tho konni pods ni consolidation nundi protect cheyyochu (long batch jobs).

## Cluster Autoscaler vs Karpenter

| | Cluster Autoscaler | Karpenter |
|---|---|---|
| Unit | ASG node groups | Direct EC2 (NodeClaims) |
| Instance types | Group ki fixed | Pods ki best fit, chala types |
| Speed | Slower (ASG) | Faster |
| Consolidation | Basic scale-down | Bin-packing, replace with cheaper |
| EKS Auto Mode | Kaadu | Built-in, managed |

## Interview one-liners

- HPA ela calculate chesthundi: current/target metric ratio x replicas; requests batti utilization.
- HPA `<unknown>`: metrics-server ledu leda pods ki requests levu.
- HPA + VPA together: same metric meeda vaddu.
- Karpenter vs CA: Karpenter groupless, pod-driven instance selection, consolidation.
- Pending pods unna Karpenter node launch cheyyakapothe: NodePool requirements/limits match avvatledu, `kubectl describe nodeclaim` / events chudu.

Lab: [labs/README.md#lab-12](labs/README.md#lab-12-hpa-autoscaling)
