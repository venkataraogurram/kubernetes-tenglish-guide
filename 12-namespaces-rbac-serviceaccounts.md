# 12. Namespaces, RBAC, ServiceAccounts

[Index](README.md) | Prev: [11](11-resources-qos-probes.md) | Next: [13. Scheduling](13-scheduling-affinity-taints.md)

## Namespaces

Cluster ni logical ga divide chesthundi: teams, environments, apps.

```bash
kubectl get ns
kubectl create ns dev
kubectl get pods -n dev
kubectl config set-context --current --namespace=dev     # default namespace marchu
```

- Names namespace lopala unique. `dev/web`, `prod/web` rendu undochu.
- RBAC, ResourceQuota, LimitRange, NetworkPolicy namespace-scoped.
- **Namespace security boundary kaadu** by default. Network isolation ki NetworkPolicy, access ki RBAC kavali.
- Default namespaces: `default`, `kube-system` (system components), `kube-public`, `kube-node-lease`.

Namespaced vs cluster-scoped:

```bash
kubectl api-resources --namespaced=true    # pods, services, deployments, secrets...
kubectl api-resources --namespaced=false   # nodes, namespaces, PV, StorageClass, ClusterRole, CRDs...
```

Namespace delete chesthe **lopala unna anni** delete. Jagratha.

## Authentication vs Authorization

```
request --> AuthN (evaru nuvvu?) --> AuthZ (idi cheyyochaa? RBAC) --> Admission (rules ok aa?) --> etcd
```

- Kubernetes lo "User" object ledu. Users external ga authenticate avtharu (EKS lo **IAM**).
- **ServiceAccount** pods kosam identity, Kubernetes object.

## RBAC: 4 objects

| Object | Scope | Pani |
|---|---|---|
| `Role` | Namespace | E resources meeda e verbs allowed |
| `ClusterRole` | Cluster | Same, cluster-wide leda cluster-scoped resources (nodes) ki |
| `RoleBinding` | Namespace | Role (leda ClusterRole) ni subject ki ivvadam, aa namespace lo |
| `ClusterRoleBinding` | Cluster | ClusterRole ni subject ki cluster motham |

Subjects: `User`, `Group`, `ServiceAccount`.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: dev
rules:
- apiGroups: [""]               # "" = core group (pods, services, configmaps, secrets)
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: dev
subjects:
- kind: ServiceAccount
  name: app-sa
  namespace: dev
- kind: Group
  name: dev-readers             # EKS access entry lo map chesina group
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Verbs: `get`, `list`, `watch`, `create`, `update`, `patch`, `delete`, `deletecollection`, plus special: `exec` ki `pods/exec` resource meeda `create`.

RBAC **additive** matrame. "Deny" rule ledu. Emi ivvakapothe emi cheyyalemu.

Trick: ClusterRole create chesi, multiple namespaces lo RoleBinding tho reuse. Built-in ClusterRoles: `view`, `edit`, `admin`, `cluster-admin`.

### Permissions check

```bash
kubectl auth can-i create deployments -n dev
kubectl auth can-i list secrets -n dev --as=system:serviceaccount:dev:app-sa
kubectl auth can-i --list -n dev
kubectl auth whoami
```

## ServiceAccounts

Prathi namespace lo `default` SA untundi. Pod ki SA ivvakapothe adi.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: dev
---
# pod spec lo
spec:
  serviceAccountName: app-sa
  automountServiceAccountToken: false     # Kubernetes API call avasaram lekapothe token mount cheyyaku
```

Pod lo token `/var/run/secrets/kubernetes.io/serviceaccount/token` lo mount avthundi (short-lived, auto-rotated projected token).
App Kubernetes API call cheyyadu ante `automountServiceAccountToken: false` best practice.

## EKS: humans ki access (IAM -> Kubernetes)

EKS lo IAM principal ni Kubernetes permissions ki map cheyyadaniki **Access Entries** (kotha way, `aws-auth` ConfigMap replace chesindi):

```bash
aws eks create-access-entry --cluster-name <cluster> \
  --principal-arn arn:aws:iam::<account>:role/DevRole \
  --kubernetes-groups dev-readers                          # RBAC binding lo vade group

# leda AWS managed access policy
aws eks associate-access-policy --cluster-name <cluster> \
  --principal-arn arn:aws:iam::<account>:role/DevRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=dev
```

`kubectl` call -> IAM credentials tho token -> EKS IAM identity verify -> access entry -> Kubernetes groups/policies -> RBAC.

## EKS: pods ki AWS access (S3, DynamoDB, etc.)

Pod lo AWS access keys pettaku. Node IAM role ki permissions ivvaku (aa node meeda anni pods ki vasthayi). Options:

### EKS Pod Identity (simpler, recommended)

```bash
aws eks create-pod-identity-association --cluster-name <cluster> \
  --namespace dev --service-account app-sa \
  --role-arn arn:aws:iam::<account>:role/AppS3ReadRole
```

Role trust policy principal `pods.eks.amazonaws.com`. SA meeda annotation avasaram ledu. Auto Mode lo Pod Identity agent built-in.

### IRSA (IAM Roles for Service Accounts, older)

Cluster OIDC provider + role trust policy + SA annotation:

```yaml
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::<account>:role/AppS3ReadRole
```

Rendu lo AWS SDK automatic ga credentials theesukuntundi (default credential chain).

Verify pod lopala:

```bash
kubectl exec -it <pod> -- aws sts get-caller-identity
```

## Least privilege checklist

- `cluster-admin` evariki ivvaku (break-glass tappa).
- Developers ki namespace-scoped `edit`/`view`.
- `secrets`, `pods/exec`, `create pods` jagratha ga: secrets chadavadaniki dari.
- Prathi app ki own ServiceAccount, own IAM role.
- `automountServiceAccountToken: false` default.

## Interview one-liners

- Role vs ClusterRole: namespace scope vs cluster scope.
- RoleBinding lo ClusterRole vadocha: yes, aa namespace ki matrame permissions vasthayi.
- ServiceAccount enduku: pod identity Kubernetes API ki (and Pod Identity/IRSA tho AWS ki).
- IRSA vs Pod Identity: rendu pod-level IAM; Pod Identity lo OIDC setup, SA annotation avasaram ledu, cross-cluster reuse easy.

Lab: [labs/README.md#lab-10](labs/README.md#lab-10-rbac-and-serviceaccount)
