# 04. Labels, selectors, annotations

[Index](README.md) | Prev: [03](03-pods.md) | Next: [05. Deployment](05-deployment-replicaset.md)

## Enduku idi chala important

Kubernetes lo resources okadanni okati **name tho pattukovu, labels tho pattukuntayi**.
Deployment e pods naavi ani, Service e pods ki traffic pampali ani, NetworkPolicy e pods ki apply avvali ani, anni labels tho decide avthayi.
Advanced lo stuck ayye chala problems ki root cause label mismatch.

## Labels

Objects meeda key-value tags.

```yaml
metadata:
  labels:
    app: web
    tier: frontend
    env: dev
    version: v2
```

Rules:
- Key: optional prefix + name, e.g. `app.kubernetes.io/name`. Name 63 chars varaku.
- Value: 63 chars varaku, alphanumeric, `-`, `_`, `.`.
- Labels meeda query cheyyochu. Anduke identify cheyyadaniki vadatham.

### Recommended labels (standard)

```yaml
labels:
  app.kubernetes.io/name: web
  app.kubernetes.io/instance: web-prod
  app.kubernetes.io/version: "1.4.2"
  app.kubernetes.io/component: frontend
  app.kubernetes.io/part-of: shop
  app.kubernetes.io/managed-by: helm
```

Tools (Helm, Argo CD, Kiali, dashboards) ivi chusi automatic ga group chesthayi. Learning ki simple `app: web` chalu.

## Selectors

Labels batti objects ni select cheyyadam.

### Equality-based

```bash
kubectl get pods -l app=web
kubectl get pods -l app=web,env=dev        # AND
kubectl get pods -l env!=prod
```

### Set-based

```bash
kubectl get pods -l 'env in (dev,qa)'
kubectl get pods -l 'tier notin (backend)'
kubectl get pods -l 'version'              # ee key unna anni
kubectl get pods -l '!version'             # ee key leni anni
```

### Manifests lo

```yaml
# Service: simple map (equality matrame)
selector:
  app: web

# Deployment/ReplicaSet/Job/NetworkPolicy: matchLabels + matchExpressions
selector:
  matchLabels:
    app: web
  matchExpressions:
  - key: env
    operator: In            # In, NotIn, Exists, DoesNotExist
    values: [dev, qa]
```

## Label chain (gurthu pettuko)

```
Deployment.spec.selector.matchLabels   app: web  ─┐
Deployment.spec.template.metadata.labels app: web ┼── ee moodu match avvali
Service.spec.selector                   app: web  ─┘
```

Match avvakapothe em jarugutundi:

| Mismatch | Result |
|---|---|
| Deployment selector vs template labels | `apply` fail: `selector does not match template labels` |
| Service selector tappu | **Error radhu.** Service ki endpoints undavu, traffic ekkadiki vellatledu |
| NetworkPolicy podSelector tappu | Policy apply avvadu, silent ga |
| HPA scaleTargetRef tappu | HPA `unknown` metrics chupisthundi |

Service check:

```bash
kubectl get endpointslices -l kubernetes.io/service-name=web
kubectl get endpoints web          # deprecated API kani quick check ki ippatiki pani chesthundi
```

Empty aithe selector check cheyyi: `kubectl get pods -l app=web` emaina return chesthunda?

## Deployment selector immutable

Deployment create ayyaka `spec.selector` marchalemu. Marchali ante Deployment delete chesi malli create cheyyali.
Kabatti selector lo stable labels matrame pettu (`app: web`). `version` lanti marutu unde labels selector lo pettaku, template labels lo matrame pettu.

## Labels runtime lo marchadam

```bash
kubectl label pod web-abc123 env=dev
kubectl label pod web-abc123 env=qa --overwrite
kubectl label pod web-abc123 env-              # remove
kubectl get pods --show-labels
kubectl get pods -L app,env                    # columns ga chupisthundi
```

Debug trick: oka misbehaving pod nundi `app=web` label teesesthe, ReplicaSet daanni "naadi kaadu" ani kotha pod create chesthundi, Service traffic aapesthundi. Old pod alaage untundi, nuvvu leisure ga debug cheyyochu.

## Annotations

Labels laage key-value, kani **select cheyyadaniki kaadu**. Extra info / tool config ki.

```yaml
metadata:
  annotations:
    kubernetes.io/change-cause: "upgrade to nginx 1.27"
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    prometheus.io/scrape: "true"
```

| | Labels | Annotations |
|---|---|---|
| Select/query cheyyocha | Yes | No |
| Size | Short values | Pedda values (JSON kuda) |
| Use | Identity, grouping | Tool config, metadata, notes |

Chala AWS/EKS features (load balancer settings, IRSA) annotations tho configure avthayi.

## Interview one-liners

- Labels vs annotations: labels identify/select ki, annotations non-identifying metadata/tool config ki.
- Service ki endpoints empty: selector pods labels tho match avvatledu leda pods Ready kaadu.
- Selector lo version label enduku pettakudadu: Deployment selector immutable, rollout lo version maruthundi.

Lab: [labs/README.md#lab-01](labs/README.md#lab-01-first-deployment-and-service) (chivari "break it" step)
