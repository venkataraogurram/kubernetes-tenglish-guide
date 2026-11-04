# 02. Manifest easy ga rayadam ela

[Index](README.md) | Prev: [01](01-kubernetes-enduku-architecture.md) | Next: [03. Pods](03-pods.md)

## Short answer: guruthupettukovalsina avasaram ledu

Experienced engineers kuda manifest motham memory nundi rayaru. Ee 4-step workflow follow chestharu:

1. **Generate**: `kubectl create ... --dry-run=client -o yaml` tho skeleton.
2. **Explain**: field ardham kaakapothe `kubectl explain`.
3. **Edit**: kavalsinavi add, avasaram lenivi delete.
4. **Validate**: `--dry-run=server` inka `kubectl diff` tho apply cheyyakamundu check.

Guruthupettukovalsindi rendu matrame: **4 top-level fields** inka **labels ela connect avthayo** ([04](04-labels-selectors-annotations.md)).

## Prathi manifest lo 4 top-level fields

```yaml
apiVersion: apps/v1      # e API group/version lo ee kind undi
kind: Deployment         # e resource type
metadata:                # identity: name, namespace, labels, annotations
  name: web
spec:                    # desired state: neeku em kavalo
  ...
```

| Field | Enduku | Mandatory? |
|---|---|---|
| `apiVersion` | Ee kind e API group lo undo cheptundi. Deployment ki `apps/v1`, Service/Pod/ConfigMap ki `v1` | Yes |
| `kind` | E resource create cheyyali | Yes |
| `metadata.name` | Namespace lopala unique name | Yes (leda `generateName`) |
| `metadata.namespace` | E namespace lo | No, default `default` (leda `kubectl -n`) |
| `spec` | Desired state | Chala kinds ki yes. ConfigMap/Secret lo `spec` ki badulu `data` |
| `status` | Current state | **Nuvvu rayavu.** Kubernetes fill chesthundi |

E kind ki e `apiVersion` ani gurthu pettukovalsina pani ledu:

```bash
kubectl api-resources | grep -i deployment
# NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
# deployments   deploy       apps/v1      true         Deployment
```

## Step 1: Generate

```bash
kubectl create deployment web --image=nginx:1.27 --replicas=2 --port=80 \
  --dry-run=client -o yaml > deploy.yaml

kubectl create service clusterip web --tcp=80:80 --dry-run=client -o yaml > svc.yaml
kubectl create configmap app-config --from-literal=APP_ENV=dev --dry-run=client -o yaml
kubectl create secret generic db-secret --from-literal=password=REPLACE_ME --dry-run=client -o yaml
kubectl create job hello --image=busybox:1.36 --dry-run=client -o yaml -- echo hi
kubectl create cronjob nightly --image=busybox:1.36 --schedule="0 2 * * *" --dry-run=client -o yaml -- echo hi
kubectl create namespace dev --dry-run=client -o yaml
kubectl create serviceaccount app-sa --dry-run=client -o yaml
kubectl create role pod-reader --verb=get,list,watch --resource=pods --dry-run=client -o yaml
kubectl create rolebinding read-pods --role=pod-reader --serviceaccount=default:app-sa --dry-run=client -o yaml
kubectl create ingress web --rule="example.com/*=web:80" --dry-run=client -o yaml
kubectl run tmp --image=busybox:1.36 --dry-run=client -o yaml -- sleep 3600   # standalone Pod
```

`--dry-run=client` ante cluster ki pampadu, local ga YAML generate chesthundi anthe. Full list: `kubectl create --help`.

Generate ayina YAML lo ivi teesesi:
- `creationTimestamp: null`
- `status: {}`
- `strategy: {}`, `resources: {}` (empty unte)

## Step 2: Explain

```bash
kubectl explain deployment                         # top level
kubectl explain deployment.spec                    # spec lo fields
kubectl explain deployment.spec.template.spec.containers.resources
kubectl explain pod.spec --recursive | less        # full tree, structure chudadaniki best
```

Output lo `-required-` ani unte adi mandatory. Example (real output):

```
  replicas              <integer>
  revisionHistoryLimit  <integer>
  selector              <LabelSelector> -required-
  strategy              <DeploymentStrategy>
  template              <PodTemplateSpec> -required-
```

## Step 3: Edit (Deployment line by line)

```yaml
apiVersion: apps/v1            # MANDATORY
kind: Deployment               # MANDATORY
metadata:
  name: web                    # MANDATORY
  labels:                      # OPTIONAL: Deployment object meeda tag (filter/search ki)
    app: web
spec:
  replicas: 2                  # OPTIONAL: default 1
  selector:                    # MANDATORY: e pods ni ee Deployment manage chesthundi
    matchLabels:
      app: web
  template:                    # MANDATORY: create cheyyalsina pod blueprint
    metadata:
      labels:                  # MANDATORY (practically): selector tho MATCH avvali
        app: web               # name ikkada raayam, enduko 05 lo chudu
    spec:
      containers:              # MANDATORY: kanisam okka container
      - name: web              # MANDATORY: pod lopala container name
        image: nginx:1.27      # MANDATORY: e image. ":latest" vaddu, fixed tag vadu
        ports:                 # OPTIONAL: informational, traffic block/allow cheyyadu
        - containerPort: 80
        resources:             # OPTIONAL kani production lo MUST (11 lo detail)
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            memory: 256Mi
        readinessProbe:        # OPTIONAL kani production lo MUST
          httpGet:
            path: /
            port: 80
```

## Step 3: Edit (Service line by line)

```yaml
apiVersion: v1                 # MANDATORY: core group, kabatti "v1" matrame
kind: Service                  # MANDATORY
metadata:
  name: web                    # MANDATORY: DNS name avthundi -> http://web
spec:
  type: ClusterIP              # OPTIONAL: default ClusterIP
  selector:                    # practically MANDATORY: e pods ki traffic
    app: web                   # Deployment template labels tho MATCH avvali
  ports:                       # MANDATORY
  - port: 80                   # Service port (clients ikkada call chestharu)
    targetPort: 80             # Container port (ivvakapothe port value ne)
    protocol: TCP              # OPTIONAL: default TCP
```

## Step 4: Validate

```bash
kubectl apply -f deploy.yaml --dry-run=server   # apiserver validation + admission, create cheyyadu
kubectl diff -f deploy.yaml                     # cluster lo unna daaniki em marutundo chupisthundi
kubectl apply -f deploy.yaml                    # nijanga apply
```

`--dry-run=client` YAML syntax/schema matrame chusthundi. `--dry-run=server` cluster side rules (namespace unda, quotas, admission policies) kuda check chesthundi. Apply cheyyakamundu server dry-run alavatu chesko.

## Multiple resources okka file lo

`---` tho separate cheyyochu:

```yaml
apiVersion: apps/v1
kind: Deployment
...
---
apiVersion: v1
kind: Service
...
```

Leda folder motham: `kubectl apply -f ./manifests/` (recursive ki `-R`).

## YAML mistakes (most common)

- **Tabs vaddu**, spaces matrame. 2 spaces indentation standard.
- List items `-` tho: `containers:` kinda `- name: web`. `-` miss aithe "expected array" error.
- Numbers lanti strings ni quote cheyyi: `value: "8080"`, `"true"`. env values anni strings undali.
- `:` tarvata space mandatory: `name: web` correct, `name:web` wrong.
- Indentation tappu aithe field ignore avvochu leda `unknown field` error. `kubectl apply --dry-run=server` idi pattukuntundi.

## Existing resources nundi nerchukovadam

```bash
kubectl get deploy -n kube-system
kubectl get deploy <name> -n kube-system -o yaml     # real-world manifest
kubectl get deploy web -o yaml | kubectl neat        # kubectl-neat plugin unte status/managedFields teesesthundi
```

## Editor setup

VS Code / Kiro lo:
- **YAML** (Red Hat) extension: schema validation, autocomplete.
- **Kubernetes** extension: cluster browse, explain on hover.
- File top lo ee line pedithe schema autocomplete pakka ga pani chesthundi:
  `# yaml-language-server: $schema=https://raw.githubusercontent.com/yannh/kubernetes-json-schema/master/master-standalone-strict/deployment-apps-v1.json`

## Summary

```
Generate (kubectl create --dry-run) -> Explain (kubectl explain) -> Edit -> Validate (--dry-run=server, diff) -> Apply
```

Lab: [labs/README.md#lab-01](labs/README.md#lab-01-first-deployment-and-service)
