# 19. GitOps, Argo CD

[Index](README.md) | Prev: [18](18-helm-kustomize.md) | Next: [20. Observability](20-observability.md)

Prerequisites: [17 controllers](17-controllers-crds-operators.md), [18 Helm/Kustomize](18-helm-kustomize.md)

## Problem with `kubectl apply` from CI

```
Developer -> CI pipeline -> kubectl apply (cluster credentials CI lo) -> cluster
```

- CI ki cluster admin credentials kavali (security risk).
- Evaraina `kubectl edit` chesthe cluster, Git veru avthayi (drift). Evariki teliyadu.
- "Prod lo ippudu em run avthondi?" ante cluster lo chudali, Git lo kaadu.

## GitOps principles

1. **Declarative**: desired state antha YAML lo.
2. **Versioned in Git**: Git ne single source of truth. History, review (PR), rollback (`git revert`).
3. **Pulled automatically**: cluster lopala agent Git nundi **pull** chesthundi. CI cluster ki push cheyyadu.
4. **Continuously reconciled**: drift aithe agent fix chesthundi.

Ide controller reconcile loop ([17](17-controllers-crds-operators.md)), desired state Git lo undi anthe.

```
Developer -> PR -> merge to Git (manifests repo)
                         ^
                         | pull + compare (every ~3 min / webhook)
                  Argo CD (cluster lo) -> apply -> cluster
```

CI pani: test, image build, ECR push, **manifests repo lo image tag update** (PR leda commit). Deploy Argo CD chesthundi.

## Argo CD

Install:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd get pods
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
kubectl -n argocd port-forward svc/argocd-server 8443:443     # https://localhost:8443, user admin
```

Production lo Helm chart tho install, SSO, RBAC configure cheyyi. EKS lo **EKS Capabilities** ga managed Argo CD option kuda undi.

### Application CRD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<you>/k8s-manifests.git
    targetRevision: main
    path: apps/web/overlays/dev          # Kustomize overlay (Helm chart path kuda)
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true          # Git nundi delete chesina resources cluster nundi kuda delete
      selfHeal: true       # cluster lo manual change chesthe Git state ki revert
    syncOptions:
    - CreateNamespace=true
```

### Status meanings

| Status | Ardham |
|---|---|
| `Synced` | Cluster == Git |
| `OutOfSync` | Difference undi (kotha commit leda manual change) |
| `Healthy` | Resources healthy (Deployment available, etc.) |
| `Progressing` | Rollout jaruguthondi |
| `Degraded` | Edo fail (CrashLoop, etc.) |

```bash
argocd app list
argocd app get web-dev
argocd app diff web-dev
argocd app sync web-dev
argocd app history web-dev
argocd app rollback web-dev <id>
```

## Repo structure (common)

```
k8s-manifests/
├── apps/
│   └── web/
│       ├── base/
│       └── overlays/{dev,qa,prod}/
├── platform/              # cluster add-ons: ingress, cert-manager, external-secrets
└── argocd/
    ├── root-app.yaml      # "app of apps": migatha Applications ni create chese Application
    └── apps/*.yaml
```

- **App of apps**: okka root Application, adi folder lo unna Application YAMLs ni sync chesthundi. Kotha app add = Git lo file add.
- **ApplicationSet**: template + generators (Git directories, clusters list) tho chala Applications auto generate. Multi-cluster/multi-env ki.

## Sync waves, hooks

Order kavalante (CRDs mundu, tarvata CRs; DB migration Job mundu, tarvata app):

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"        # takkuva number first
```

```yaml
metadata:
  annotations:
    argocd.argoproj.io/hook: PreSync          # sync mundu run (migration Job)
    argocd.argoproj.io/hook-delete-policy: HookSucceeded
```

## HPA tho conflict

Git lo `replicas: 3`, HPA 7 ki scale chesindi. selfHeal 3 ki revert chesthundi, fight.
Fix: manifest nundi `replicas` teesey, leda Argo CD `ignoreDifferences` lo `/spec/replicas`.

## Secrets in GitOps

Plain Secrets Git lo pettalemu. Options: External Secrets Operator (Git lo ExternalSecret matrame), Sealed Secrets, SOPS + Argo CD plugin. ([09](09-configmap-secret.md))

## Flux (alternative)

Flux kuda GitOps tool (CNCF). Argo CD ki UI strong, Flux lightweight, Kubernetes-native CRDs (`GitRepository`, `Kustomization`, `HelmRelease`). Concepts same.

## Progressive delivery

**Argo Rollouts**: Deployment ki badulu `Rollout` CRD. Canary steps (10% -> 50% -> 100%), metrics analysis (Prometheus error rate high aithe auto rollback), blue/green. Istio/ALB/Gateway API tho traffic split.

## Interview one-liners

- GitOps ante: Git source of truth, cluster agent pull + continuous reconcile.
- Push CI/CD vs GitOps pull: cluster credentials CI lo undavu, drift detection/self-heal.
- selfHeal vs prune: selfHeal manual changes revert; prune Git nundi removed resources delete.
- Rollback GitOps lo: `git revert` (best) leda Argo CD history rollback (auto-sync off cheyyali lekapothe Git malli apply).

Lab: [labs/README.md#lab-17](labs/README.md#lab-17-argo-cd-optional)
