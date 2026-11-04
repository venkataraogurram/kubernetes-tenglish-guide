# 17. Controllers, CRDs, Operators

[Index](README.md) | Prev: [16](16-security.md) | Next: [18. Helm, Kustomize](18-helm-kustomize.md)

Prerequisites: [01 reconcile loop](01-kubernetes-enduku-architecture.md)

Ee topic ardham aithe Istio, Karpenter, Argo CD, External Secrets, cert-manager anni "magic" laaga kanipinchavu. Anni ide pattern.

## Controller = watch + reconcile

```
            +----------------+
            |  kube-apiserver|
            +-------+--------+
     watch (events) |  ^  update (create/patch objects, status)
                    v  |
            +----------------+
            |   Controller   |
            |  reconcile()   |
            +----------------+
```

```
reconcile(object):
    desired = object.spec
    actual  = real world lo state (pods, AWS resources, files...)
    actions = diff(desired, actual)
    apply(actions)
    object.status = actual summary
    return (error unte retry)
```

Important properties:
- **Level-triggered**, edge-triggered kaadu: "em maarindi" kaadu, "ippudu state enti, em undali" ani prathi sari full compare. Event miss ayina next reconcile fix chesthundi.
- **Idempotent**: 10 sarlu run chesina same result.
- **Eventually consistent**: ventane kaadu, konchem sepu lo desired state ki.

### Built-in examples

| Controller | Watches | Creates/Manages |
|---|---|---|
| Deployment | Deployments | ReplicaSets |
| ReplicaSet | ReplicaSets, Pods | Pods |
| Job | Jobs | Pods |
| EndpointSlice | Services, Pods | EndpointSlices |
| Node | Nodes | Taints, evictions |
| Namespace | Namespaces | Deletion of contents |

### Owner references, garbage collection

Child objects lo `ownerReferences` untayi:

```bash
kubectl get pod web-7c5d-x8k2p -o jsonpath='{.metadata.ownerReferences}'
# [{"kind":"ReplicaSet","name":"web-7c5d", "controller":true ...}]
```

Parent delete aithe garbage collector children ni delete chesthundi (cascading delete). Deployment delete -> RS delete -> Pods delete.

```bash
kubectl delete deployment web --cascade=orphan   # pods/RS unchi Deployment matrame delete
```

### Finalizers

Object delete chesinappudu `finalizers` unte, aa controller cleanup (e.g. AWS load balancer delete, EBS volume detach) complete chesi finalizer remove chese varaku object `Terminating` lo untundi.

```bash
kubectl get ingress web -o jsonpath='{.metadata.finalizers}'
```

"Namespace/PVC Terminating lo stuck" ki common reason: controller down leda cleanup fail, finalizer remove avvatledu. Finalizer ni force ga remove cheyyadam last resort, endukante AWS lo resources orphan avvochu.

## CRD (CustomResourceDefinition)

Kubernetes API ki **kotha resource type** add cheyyadam.

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.demo.venkat.dev
spec:
  group: demo.venkat.dev
  names:
    kind: Website
    plural: websites
    singular: website
    shortNames: [ws]
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [message]
            properties:
              message: { type: string }
              replicas: { type: integer, minimum: 1, maximum: 5 }
    additionalPrinterColumns:
    - name: Message
      type: string
      jsonPath: .spec.message
```

Apply chesaka:

```bash
kubectl get websites          # pani chesthundi
kubectl explain website.spec  # schema nundi docs
```

```yaml
apiVersion: demo.venkat.dev/v1
kind: Website
metadata:
  name: hello
spec:
  message: "Namaskaram"
  replicas: 2
```

**CRD okkate data store matrame**. etcd lo object save avthundi, kani emi jaragadu. Daanni chusi pani chese **controller** kavali.

## Operator = CRD + controller + domain knowledge

Operator ante oka application ni manage chese human operator knowledge ni code lo pettadam:
- PostgreSQL operator (CloudNativePG): `kind: Cluster` create chesthe StatefulSet, Services, replication, backups, failover anni.
- cert-manager: `kind: Certificate` -> Let's Encrypt tho cert teesukuni Secret lo pedathundi, renew chesthundi.
- Prometheus Operator: `kind: ServiceMonitor` -> Prometheus scrape config.
- Karpenter: `kind: NodePool` -> EC2 instances.
- Istio: `VirtualService` -> Envoy config.
- Argo CD: `kind: Application` -> Git nundi sync.

Ee repo lo chusina chala "advanced" tools anni ide pattern ani gamaninchu.

## Controller ela rastharu (overview)

- Go lo **Kubebuilder** / **Operator SDK** (controller-runtime library) standard.
- Python lo **kopf**.
- Reconcile function rayadam main pani, watches/caching/retries framework chusukuntundi.

Pseudo-code (Website operator):

```go
func (r *WebsiteReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    var site demov1.Website
    if err := r.Get(ctx, req.NamespacedName, &site); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)   // delete ayyindi, emi cheyyakku
    }
    // desired: ConfigMap (message tho index.html) + Deployment (nginx, replicas) + Service
    cm := buildConfigMap(site)
    dep := buildDeployment(site)
    ctrl.SetControllerReference(&site, dep, r.Scheme)       // ownerReference -> GC
    // CreateOrUpdate: lekapothe create, unte spec match cheyyi
    controllerutil.CreateOrUpdate(ctx, r.Client, cm, mutateFn)
    controllerutil.CreateOrUpdate(ctx, r.Client, dep, mutateFn)
    site.Status.Ready = dep.Status.ReadyReplicas == *dep.Spec.Replicas
    r.Status().Update(ctx, &site)
    return ctrl.Result{}, nil
}
```

## spec vs status (convention)

- `spec`: user rasthadu (desired).
- `status`: controller rasthundi (observed). `conditions` list standard: `Ready`, `Progressing`, `Degraded` with `reason`, `message`.

```bash
kubectl get deployment web -o jsonpath='{.status.conditions}' | jq
kubectl wait --for=condition=Available deployment/web --timeout=120s
```

## Interview one-liners

- Controller ante: resource watch chesi actual state ni desired ki reconcile chese loop.
- CRD vs operator: CRD kotha API type; operator = CRD + controller with app-specific logic.
- Finalizer enduku: delete mundu external cleanup guarantee.
- Level-triggered enduku important: missed events tho kuda correct state ki converge.

Lab: [labs/README.md#lab-15](labs/README.md#lab-15-crd-without-and-with-a-controller)
