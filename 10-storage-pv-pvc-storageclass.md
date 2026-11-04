# 10. Storage: PV, PVC, StorageClass

[Index](README.md) | Prev: [09](09-configmap-secret.md) | Next: [11. Resources, probes](11-resources-qos-probes.md)

## Problem

Container filesystem temporary. Container restart aithe lopala rasina files poyayi.
Database, uploads lanti data ki pod life kante ekkuva kalam unde storage kavali.

## Volume types (quick)

| Type | Lifetime | Use |
|---|---|---|
| `emptyDir` | Pod unnantha varaku | Containers madhya temp files share, cache |
| `hostPath` | Node meeda | Node agents matrame. Apps ki vaddu (security risk, node marithe data poyindi) |
| `configMap` / `secret` | Object tho | Config files |
| `persistentVolumeClaim` | Pod kante ekkuva | **Real persistent data** |
| `projected` | Pod tho | Multiple sources (SA token, configmap, secret) okka dir lo |

## 3 objects ardham chesuko

```
Pod --uses--> PVC ("naaku 10Gi RWO disk kavali")
                |
                v  bind
              PV  (actual disk: EBS vol-0abc...)
                ^
                | dynamic ga create chesindi
          StorageClass ("gp3 EBS, encrypted, ila create cheyyi")
```

- **PersistentVolume (PV)**: actual storage piece. Cluster-level object.
- **PersistentVolumeClaim (PVC)**: storage kosam request. Namespace-level. Pod PVC ni reference chesthundi.
- **StorageClass**: PV ni **dynamic** ga ela create cheyyalo template. PVC create chesthe provisioner (CSI driver) automatic ga disk create chesi PV create chesthundi.

Analogy: StorageClass = menu, PVC = order, PV = vachina plate.

## EKS Auto Mode: StorageClass nuvve create cheyyali

EKS Auto Mode StorageClass create cheyyadu. Cluster lo `gp2` (`kubernetes.io/aws-ebs` in-tree provisioner) StorageClass kanipinchina, adi Auto Mode storage vadadu. Auto Mode ki provisioner `ebs.csi.eks.amazonaws.com` undali:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: auto-ebs-sc
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"   # PVC lo class ivvakapothe idi
provisioner: ebs.csi.eks.amazonaws.com
volumeBindingMode: WaitForFirstConsumer
allowedTopologies:
- matchLabelExpressions:
  - key: eks.amazonaws.com/compute-type
    values: ["auto"]
parameters:
  type: gp3
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
```

`gp2` ni kuda default ga mark chesi unte rendu defaults avthayi. Check:

```bash
kubectl get storageclass     # (default) ani e class pakkana undo chudu
```

Source: [EKS docs: Create a storage class](https://docs.aws.amazon.com/eks/latest/userguide/create-storage-class.html)

## PVC + Pod

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: auto-ebs-sc
  resources:
    requests:
      storage: 5Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: writer
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "date >> /data/log.txt; cat /data/log.txt; sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: data
```

Pod delete chesi malli create chesthe `/data/log.txt` lo old lines untayi. Ade persistence.

## Important fields

### accessModes

| Mode | Ardham | EBS | EFS |
|---|---|---|---|
| `ReadWriteOnce` (RWO) | Okka **node** read-write | Yes | Yes |
| `ReadOnlyMany` (ROX) | Chala nodes read-only | No | Yes |
| `ReadWriteMany` (RWX) | Chala nodes read-write | **No** | Yes |
| `ReadWriteOncePod` | Okka **pod** matrame | Yes | Yes |

EBS block disk, okka EC2 ki matrame attach. Chala pods different nodes nundi share cheyyali ante **EFS** (EFS CSI driver, Auto Mode lo built-in kaadu, add-on install cheyyali).

### volumeBindingMode

- `Immediate`: PVC create avvagane disk create. Pod vere AZ lo schedule aithe attach avvadu.
- `WaitForFirstConsumer`: pod schedule ayyaka, aa node AZ lo disk create. **EBS ki idi vadu.**

### reclaimPolicy

- `Delete` (default dynamic): PVC delete chesthe disk kuda delete. **Data poyindi.**
- `Retain`: PVC delete chesina disk undi, manual ga cleanup/recover.

Production DB data ki `Retain` + snapshots alochinchu.

### Expansion

`allowVolumeExpansion: true` unte PVC `storage` penchi apply chesthe disk grow avthundi. Shrink cheyyalemu.

## PV / PVC states

```bash
kubectl get pvc
kubectl get pv
kubectl describe pvc data
```

| Status | Ardham |
|---|---|
| `Pending` | Inka bind avvaledu. WaitForFirstConsumer lo pod schedule ayye varaku normal |
| `Bound` | PV tho bind ayyindi |
| `Released` | PVC delete, PV `Retain` tho undi, malli bind avvadu (manual cleanup) |
| `Failed` | Reclaim fail |

PVC `Pending` lo undipoyindi ante `describe pvc` events chudu: StorageClass ledu, provisioner tappu, IAM permissions, AZ mismatch.

## Snapshots

CSI `VolumeSnapshot` (snapshot CRDs + controller kavali):

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: data-snap
spec:
  volumeSnapshotClassName: <class>
  source:
    persistentVolumeClaimName: data
```

Cluster-level backup ki **Velero** / AWS Backup for EKS.

## Common mistakes

- Deployment `replicas: 3` + RWO EBS PVC: okka node meeda pods matrame start avthayi, migathavi `Multi-Attach error`. Per-pod storage ki StatefulSet vadu.
- Pod AZ marithe EBS follow avvadu. EBS AZ-bound.
- `Delete` reclaim tho PVC delete chesi data pogottukovadam.
- Auto Mode lo old `gp2` in-tree class vadi PVC `Pending` lo undipovadam.

## Interview one-liners

- PV vs PVC: PV actual storage, PVC user request; PVC PV ki bind avthundi.
- StorageClass enduku: dynamic provisioning, disk type/params define.
- WaitForFirstConsumer enduku: topology-aware, pod AZ lo ne disk create.
- RWX EBS tho possible aa: ledu (EBS Multi-Attach special case tappa); EFS vadu.

Lab: [labs/README.md#lab-08](labs/README.md#lab-08-persistent-storage-with-ebs)
