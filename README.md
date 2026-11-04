# Kubernetes Tenglish Guide

**Version:** v1.0

Kubernetes ni basics nundi advanced varaku **Tenglish** lo (Telugu + English technical terms) explain chese guide.
Prathi topic lo "idi enti", "enduku vadatham", "ela rayali", "common mistakes" untayi.
Labs anni **Amazon EKS Auto Mode** cluster meeda test chesi rasinavi.

## Ela chadavali

- Order lo chaduvu. Prathi file mundu file meeda depend avthundi.
- Prathi file chadivaka `labs/` lo aa topic lab run cheyyi. Chadavadam kante cheyyadam tho ne gurthu untundi.
- Advanced level lo stuck aithe, aa topic ki "Prerequisites" section chusi aa basic file malli chaduvu.

## Level 1: Basics (foundation)

| # | Topic | Em nerchukuntam |
|---|---|---|
| 01 | [Kubernetes enduku, architecture](01-kubernetes-enduku-architecture.md) | Control plane, nodes, desired state, reconcile loop |
| 02 | [Manifest easy ga rayadam](02-manifest-rayadam-ela.md) | Mandatory vs optional fields, generate/explain/validate workflow |
| 03 | [Pods](03-pods.md) | Pod lifecycle, multi-container, init containers, sidecars |
| 04 | [Labels, selectors, annotations](04-labels-selectors-annotations.md) | Resources ela connect avthayi |
| 05 | [Deployment, ReplicaSet](05-deployment-replicaset.md) | Rolling update, rollback, template lo name enduku undadu |
| 06 | [StatefulSet, DaemonSet, Job, CronJob](06-statefulset-daemonset-job-cronjob.md) | E workload ki e controller |

## Level 2: Intermediate

| # | Topic | Em nerchukuntam |
|---|---|---|
| 07 | [Services, DNS](07-services-dns.md) | ClusterIP, NodePort, LoadBalancer, headless, endpoints |
| 08 | [Ingress, Gateway API](08-ingress-gateway-api.md) | L7 routing, EKS Auto Mode ALB |
| 09 | [ConfigMap, Secret](09-configmap-secret.md) | Config ni image nundi veru cheyyadam |
| 10 | [Storage: PV, PVC, StorageClass](10-storage-pv-pvc-storageclass.md) | Dynamic provisioning, EBS, access modes |
| 11 | [Resources, QoS, probes](11-resources-qos-probes.md) | requests/limits, OOMKilled, readiness vs liveness |
| 12 | [Namespaces, RBAC, ServiceAccounts](12-namespaces-rbac-serviceaccounts.md) | Access control, Pod Identity / IRSA |

## Level 3: Advanced (ikkade chala mandi stuck avtharu)

| # | Topic | Em nerchukuntam |
|---|---|---|
| 13 | [Scheduling](13-scheduling-affinity-taints.md) | nodeSelector, affinity, taints/tolerations, topology spread, PDB |
| 14 | [Autoscaling](14-autoscaling-hpa-vpa-karpenter.md) | HPA, VPA, Cluster Autoscaler, Karpenter |
| 15 | [Networking deep dive](15-networking-deep-dive.md) | Pod-to-pod, kube-proxy, VPC CNI, NetworkPolicy |
| 16 | [Security](16-security.md) | securityContext, Pod Security Standards, image security |
| 17 | [Controllers, CRDs, Operators](17-controllers-crds-operators.md) | Kubernetes lopala ela pani chesthundi |
| 18 | [Helm, Kustomize](18-helm-kustomize.md) | Manifests ni package/reuse cheyyadam |
| 19 | [GitOps, Argo CD](19-gitops-argocd.md) | Git ne source of truth |
| 20 | [Observability](20-observability.md) | Logs, metrics, traces, events |
| 21 | [Service mesh, Istio](21-service-mesh-istio.md) | mTLS, traffic splitting, Envoy sidecars, ambient |
| 22 | [EKS specifics, Auto Mode](22-eks-auto-mode.md) | EKS lo AWS tho integration |
| 23 | [Troubleshooting playbook](23-troubleshooting-playbook.md) | CrashLoopBackOff, Pending, ImagePullBackOff, empty endpoints |
| 24 | [kubectl cheatsheet](24-kubectl-cheatsheet.md) | Daily commands, explanation tho |

## Labs

[labs/README.md](labs/README.md) lo step-by-step labs untayi. Anni `k8s-lab` namespace lo create avthayi, chivari lo okka command tho clean up cheyyochu.

## Learning path suggestion

```
Week 1: 01-06  (basics + labs 01-03)
Week 2: 07-12  (intermediate + labs 04-08)
Week 3: 13-17  (advanced core + labs 09-11)
Week 4: 18-24  (tooling, mesh, EKS, troubleshooting)
```

Advanced lo stuck avvadaniki main reason: basics lo **labels/selectors**, **controllers reconcile loop**, **networking** sariga clear kaakapovadam.
Ee moodu clear aithe advanced topics anni vaati meeda build ayyevi ani ardham avthundi.

See [CHANGELOG.md](CHANGELOG.md) for versions.
