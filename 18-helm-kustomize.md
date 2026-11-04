# 18. Helm, Kustomize

[Index](README.md) | Prev: [17](17-controllers-crds-operators.md) | Next: [19. GitOps](19-gitops-argocd.md)

## Problem

Same app dev, qa, prod lo deploy. Differences: replicas, image tag, resources, domain. 3 copies of YAML maintain cheyyadam = drift, mistakes.
Third-party apps (Prometheus, ingress-nginx, Argo CD) ki 50+ YAMLs, prathi okkati manual ga rayalemu.

Rendu solutions:
- **Kustomize**: templates levu. Base YAML + environment-specific **patches** (overlays). `kubectl` lo built-in.
- **Helm**: Go **templates** + `values.yaml`. Package manager: charts install/upgrade/rollback, versioning.

## Kustomize

### Structure

```
app/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   └── service.yaml
└── overlays/
    ├── dev/
    │   └── kustomization.yaml
    └── prod/
        ├── kustomization.yaml
        └── replicas-patch.yaml
```

### base/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
- deployment.yaml
- service.yaml
labels:
- pairs:
    app.kubernetes.io/part-of: shop
  includeSelectors: false
```

### overlays/prod/kustomization.yaml

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: prod
namePrefix: prod-
resources:
- ../../base
images:
- name: nginx                  # base lo image name
  newTag: "1.27.3"
patches:
- path: replicas-patch.yaml
configMapGenerator:
- name: app-config
  literals:
  - LOG_LEVEL=warn             # name ki hash suffix, content maarithe automatic rollout
```

### overlays/prod/replicas-patch.yaml (strategic merge patch)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web                    # base name tho match
spec:
  replicas: 5
  template:
    spec:
      containers:
      - name: web
        resources:
          requests: { cpu: 500m, memory: 512Mi }
```

### Commands

```bash
kubectl kustomize overlays/prod          # final YAML chudu (apply cheyyakunda)
kubectl apply -k overlays/prod
kubectl diff -k overlays/prod
kubectl delete -k overlays/prod
```

Pros: plain YAML, readable, template syntax ledu. Cons: complex conditional logic kastam, packaging/versioning ledu.

## Helm

### Concepts

| Term | Ardham |
|---|---|
| Chart | Package: templates + default values + metadata |
| Values | Customization inputs (`values.yaml`, `--set`, `-f`) |
| Release | Cluster lo install ayina chart instance (name tho) |
| Repository | Charts store (HTTP repo leda OCI registry like ECR) |
| Revision | Prathi upgrade ki number, rollback ki |

### Third-party chart install

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo redis
helm show values bitnami/redis > redis-values.yaml     # e values marchochu chudu
helm install my-redis bitnami/redis -n cache --create-namespace -f redis-values.yaml
helm list -A
helm status my-redis -n cache
helm upgrade my-redis bitnami/redis -n cache -f redis-values.yaml
helm history my-redis -n cache
helm rollback my-redis 1 -n cache
helm uninstall my-redis -n cache
```

`helm upgrade --install` = lekapothe install, unte upgrade. CI/CD lo idi vadatharu.

### Own chart create

```bash
helm create web
```

```
web/
├── Chart.yaml          # name, version (chart), appVersion (app)
├── values.yaml         # defaults
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── _helpers.tpl    # reusable template snippets
│   └── NOTES.txt       # install tarvata message
└── charts/             # dependencies
```

### Template example

`values.yaml`:

```yaml
replicaCount: 2
image:
  repository: nginx
  tag: "1.27"
service:
  port: 80
resources:
  requests: { cpu: 100m, memory: 128Mi }
ingress:
  enabled: false
```

`templates/deployment.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "web.fullname" . }}
  labels:
    {{- include "web.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "web.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "web.selectorLabels" . | nindent 8 }}
      annotations:
        # templates/configmap.yaml unte matrame: content maarithe hash maari pods restart
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
      - name: web
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: 80
        resources:
          {{- toYaml .Values.resources | nindent 10 }}
```

`templates/ingress.yaml` lo conditional:

```yaml
{{- if .Values.ingress.enabled }}
apiVersion: networking.k8s.io/v1
kind: Ingress
...
{{- end }}
```

Template syntax quick ref:
- `{{ .Values.x }}`: values nundi. `.Release.Name`, `.Release.Namespace`, `.Chart.Name` kuda.
- `{{- ... }}` / `{{ ... -}}`: whitespace trim.
- `| nindent N`: newline + N spaces indentation. YAML indentation errors ki main source, idi gamaninchu.
- `toYaml`, `default`, `quote`, `required`, `if/else`, `range`, `include`.

### Debug

```bash
helm lint ./web
helm template my-web ./web -f values-prod.yaml          # rendered YAML, cluster ki pampadu
helm install my-web ./web --dry-run=server --debug
helm get values my-web                                  # cluster lo release ki vadina values
helm get manifest my-web                                # actual applied YAML
```

Cluster lo Helm release info `sh.helm.release.v1.<name>.v<rev>` Secrets lo store avthundi.

## Edi eppudu

| Situation | Choice |
|---|---|
| Third-party software install | **Helm** (vendors charts isthaaru) |
| Own app, environments madhya chinna differences | **Kustomize** |
| Own app, chala teams reuse, many options | **Helm** |
| Helm chart + nee own small tweaks | Kustomize `helmCharts` leda Argo CD lo Helm + Kustomize |

Chala companies: infra add-ons Helm tho, own apps Kustomize leda Helm, deploy Argo CD tho.

## Interview one-liners

- Helm vs Kustomize: templating + package mgmt vs template-free patching.
- Helm rollback ela: revision history Secrets lo, `helm rollback <rel> <rev>`.
- `helm template` vs `--dry-run`: template local render; dry-run server tho validate.
- ConfigMap change tho pods restart Helm lo: checksum annotation.

Lab: [labs/README.md#lab-16](labs/README.md#lab-16-kustomize-overlays)
