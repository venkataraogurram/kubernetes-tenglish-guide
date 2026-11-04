# 13. Scheduling: nodeSelector, affinity, taints, topology spread, PDB

[Index](README.md) | Prev: [12](12-namespaces-rbac-serviceaccounts.md) | Next: [14. Autoscaling](14-autoscaling-hpa-vpa-karpenter.md)

Prerequisites: [01 scheduler role](01-kubernetes-enduku-architecture.md), [04 labels](04-labels-selectors-annotations.md), [11 requests](11-resources-qos-probes.md)

## Scheduler ela decide chesthundi

Pending pod ki 2 phases:

1. **Filter**: e nodes ki ee pod fit avthundi?
   - Enough allocatable CPU/memory (requests batti, actual usage kaadu)
   - nodeSelector / nodeAffinity match
   - Taints tolerate chesthunda
   - Volume AZ match (EBS)
   - Ports free
2. **Score**: fit ayye nodes lo best edi? (spread, affinity preferences, image already undi, balanced usage)

Okka node kuda filter pass kaakapothe pod `Pending`, event lo reason:

```
0/3 nodes are available: 1 Insufficient cpu, 2 node(s) had untolerated taint {dedicated: gpu}.
```

Ee message chadavadam nerchuko, chala debugging ide.

Mental model: **Pod side rules** (nodeSelector, affinity, tolerations) "nenu ekkada vellali/vellochu". **Node side rules** (taints) "naa daggariki evaru raakudadu".

## 1. nodeSelector (simple)

```yaml
spec:
  nodeSelector:
    kubernetes.io/arch: arm64
    topology.kubernetes.io/zone: us-east-1a
```

Node labels exact match. Ee labels unna nodes meeda matrame.

```bash
kubectl get nodes --show-labels
kubectl get nodes -L kubernetes.io/arch,topology.kubernetes.io/zone,node.kubernetes.io/instance-type
```

Common well-known labels: `kubernetes.io/arch`, `kubernetes.io/os`, `topology.kubernetes.io/zone`, `topology.kubernetes.io/region`, `node.kubernetes.io/instance-type`, `karpenter.sh/capacity-type` (spot/on-demand).

## 2. Node affinity (flexible nodeSelector)

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:     # HARD rule
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/arch
            operator: In
            values: [arm64, amd64]
      preferredDuringSchedulingIgnoredDuringExecution:    # SOFT rule, weight tho
      - weight: 80
        preference:
          matchExpressions:
          - key: karpenter.sh/capacity-type
            operator: In
            values: [spot]
```

- `required...`: match kaakapothe schedule avvadu.
- `preferred...`: try chesthundi, kudarakapothe vere chota.
- `IgnoredDuringExecution`: pod run ayyaka node labels maarina pod move avvadu.
- Operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`.

Long name gurthu pettukune trick: **required/preferred + DuringScheduling + IgnoredDuringExecution**.

## 3. Pod affinity / anti-affinity (vere pods batti)

"Ee pod ni aa pods daggara pettu" leda "aa pods nundi dooram ga pettu".

```yaml
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web
          topologyKey: kubernetes.io/hostname     # same node lo rendu web pods vaddu
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: cache
        topologyKey: topology.kubernetes.io/zone  # cache unna zone lo ne web pettu
```

**topologyKey** ante "same" ante enti: `kubernetes.io/hostname` = same node, `topology.kubernetes.io/zone` = same AZ.

Required anti-affinity tho replicas > nodes aithe extra pods `Pending`. Mostly `preferred` vadu, leda topology spread better.

## 4. Topology spread constraints (HA ki best)

Pods ni zones/nodes across evenly spread.

```yaml
spec:
  topologySpreadConstraints:
  - maxSkew: 1                                  # zones madhya difference max 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule            # hard. ScheduleAnyway = soft
    labelSelector:
      matchLabels:
        app: web
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: web
```

6 replicas, 3 AZs -> 2-2-2. Okka AZ down aithe 4 pods survive. Anti-affinity kante idi scale lo baaga pani chesthundi.
Karpenter/Auto Mode ee constraints chusi kavalsina AZ lo node launch chesthundi.

## 5. Taints and tolerations (node side)

**Taint** node meeda: "ee taint tolerate cheyyani pods raakudadu".
**Toleration** pod meeda: "aa taint naaku ok".

```bash
kubectl taint nodes node1 dedicated=gpu:NoSchedule
kubectl taint nodes node1 dedicated=gpu:NoSchedule-        # remove
kubectl describe node node1 | grep Taints
```

Effects:

| Effect | Ardham |
|---|---|
| `NoSchedule` | Kotha pods raavu (tolerate chesevi tappa). Unnavi untayi |
| `PreferNoSchedule` | Try to avoid (soft) |
| `NoExecute` | Kotha pods raavu + unna non-tolerating pods **evict** |

```yaml
spec:
  tolerations:
  - key: dedicated
    operator: Equal
    value: gpu
    effect: NoSchedule
```

**Common confusion**: toleration "aa node meeda **vellochu**" ani matrame, "vellali" ani kaadu.
Dedicated nodes ki rendu kavali: taint (others ni bayata unchadaniki) + nodeAffinity/nodeSelector (ee pods ni akkadiki pampadaniki).

Built-in taints: `node.kubernetes.io/not-ready`, `node.kubernetes.io/unreachable` (NoExecute, default toleration 300s: ade node down ayyaka 5 min eviction), `node.kubernetes.io/disk-pressure`, `memory-pressure`.

EKS Auto Mode `system` NodePool nodes ki `CriticalAddonsOnly` taint untundi. Mee app pods akkada schedule avvavu, `general-purpose` NodePool kotha nodes launch chesthundi.

## 6. PodDisruptionBudget (PDB)

**Voluntary disruptions** (node drain, upgrades, Karpenter consolidation) lo kanisam entha pods undali.

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  minAvailable: 2          # leda maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

- `kubectl drain` / Auto Mode node replacement PDB respect chesthayi. Violate ayithe wait chesthayi.
- Node crash (involuntary) ni PDB aapalemu.
- `replicas: 1` + `minAvailable: 1` = drain eppatiki complete avvadu. Node upgrades block. Common mistake.

## 7. PriorityClass, preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
value: 100000
---
spec:
  priorityClassName: high
```

Space lekapothe scheduler takkuva priority pods ni evict (preempt) chesi high priority pod ki chota isthundi. Critical system pods `system-cluster-critical`, `system-node-critical` vadatayi.

## Node maintenance

```bash
kubectl cordon node1        # kotha pods raavu (unschedulable)
kubectl drain node1 --ignore-daemonsets --delete-emptydir-data   # pods evict (PDB respect)
kubectl uncordon node1
```

Auto Mode lo nodes AWS manage chesthundi (built-in NodePools lo `expireAfter: 336h` = 14 days tarvata automatic replacement, max 21 days), kani concept same: PDBs + multiple replicas pettukunte disruption ledu.

## Decision table

| Need | Use |
|---|---|
| Specific node type/arch/zone ki | nodeSelector / nodeAffinity required |
| Prefer spot, kani on-demand ok | nodeAffinity preferred |
| Replicas different nodes/AZs lo | topologySpreadConstraints |
| Rendu apps daggara (latency) | podAffinity |
| GPU nodes ki GPU workloads matrame | taint + toleration + nodeAffinity |
| Upgrades lo downtime vaddu | PDB + replicas >= 2 |

## Interview one-liners

- Taint vs affinity: taint node repels pods; affinity pod attracts to nodes.
- Toleration unte aa node ki velthunda: guarantee ledu, allowed matrame.
- Pod Pending ki first step: `kubectl describe pod`, Events lo scheduler message.
- PDB enduku: voluntary disruptions lo minimum availability.

Lab: [labs/README.md#lab-11](labs/README.md#lab-11-scheduling-spread-and-pdb)
