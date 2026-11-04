# 09. ConfigMap, Secret

[Index](README.md) | Prev: [08](08-ingress-gateway-api.md) | Next: [10. Storage](10-storage-pv-pvc-storageclass.md)

## Enduku

Same image dev, qa, prod lo run avvali. Config (DB host, feature flags, log level) image lo hardcode chesthe prathi env ki separate image build cheyyali.
Config ni image nundi bayataki theesi Kubernetes objects lo pedatham: non-sensitive ki **ConfigMap**, sensitive ki **Secret**.

## ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "dev"
  LOG_LEVEL: "info"
  app.properties: |              # full file kuda
    server.port=8080
    feature.x=true
```

Create shortcuts:

```bash
kubectl create configmap app-config --from-literal=APP_ENV=dev --from-literal=LOG_LEVEL=info
kubectl create configmap nginx-conf --from-file=nginx.conf
kubectl create configmap env-cm --from-env-file=.env.dev
```

Size limit 1 MiB.

## Pod lo vadadaniki 3 ways

### 1. Specific keys env vars ga

```yaml
env:
- name: APP_ENV
  valueFrom:
    configMapKeyRef:
      name: app-config
      key: APP_ENV
```

### 2. Anni keys env vars ga

```yaml
envFrom:
- configMapRef:
    name: app-config
- secretRef:
    name: db-secret
```

### 3. Files ga (volume mount)

```yaml
spec:
  containers:
  - name: app
    volumeMounts:
    - name: config
      mountPath: /etc/app          # prathi key oka file: /etc/app/app.properties
      readOnly: true
  volumes:
  - name: config
    configMap:
      name: app-config
      items:                       # optional: konni keys matrame
      - key: app.properties
        path: app.properties
```

### Update ayithe em jarugutundi (important)

| Method | ConfigMap update chesthe |
|---|---|
| env / envFrom | **Update avvadu.** Pod restart kavali |
| Volume mount | ~1 min lo file update avthundi (kubelet sync). App file re-read cheyyali |
| Volume `subPath` mount | **Update avvadu** |

Restart cheyyadaniki:

```bash
kubectl rollout restart deployment/web
```

Helm/Kustomize lo common trick: ConfigMap content hash ni pod template annotation ga pettadam (`checksum/config: <sha>`). Content maarithe annotation maruthundi, automatic rollout.
Kustomize `configMapGenerator` name ki hash suffix add chesi idi automatic ga chesthundi.

`immutable: true` set chesthe ConfigMap marchalemu, kani kubelet watch cheyyadu, performance better (pedda clusters lo).

## Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:                  # plain text rayochu, Kubernetes base64 chesthundi
  username: appuser
  password: REPLACE_ME
```

`data:` vadithe values base64 lo ivvali: `echo -n 'appuser' | base64`.

```bash
kubectl create secret generic db-secret --from-literal=username=appuser --from-literal=password='REPLACE_ME'
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 -d
```

### Secret types

| Type | Use |
|---|---|
| `Opaque` | Generic |
| `kubernetes.io/tls` | TLS cert + key (`tls.crt`, `tls.key`) |
| `kubernetes.io/dockerconfigjson` | Private registry pull (`imagePullSecrets`) |
| `kubernetes.io/service-account-token` | Legacy SA tokens |

### Secret "secret" kaadu (by default)

- base64 **encoding**, encryption kaadu. Evaraina decode cheyyochu.
- Namespace lo `get secret` RBAC permission unna evaraina chadavochu.
- Pod create permission unna vallu kuda secret mount chesi chadavochu.
- Git lo Secret YAML commit cheyyaku.

Production lo cheyyalsinavi:
1. **etcd encryption at rest**: EKS lo Kubernetes secrets ki KMS envelope encryption (kotha EKS clusters lo default ga on).
2. **RBAC** tho `secrets` access restrict.
3. Source of truth ga **AWS Secrets Manager / SSM Parameter Store** vadu, cluster loki sync cheyyi:
   - **External Secrets Operator**: Secrets Manager nundi Kubernetes Secret create/refresh chesthundi.
   - **Secrets Store CSI Driver + AWS provider**: volume ga mount.
4. Git lo pettali ante **Sealed Secrets** leda **SOPS** (encrypted form).

External Secrets example:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret               # ee name tho Kubernetes Secret create avthundi
  data:
  - secretKey: password
    remoteRef:
      key: prod/shop/db
      property: password
```

## Private registry image pull

```bash
kubectl create secret docker-registry regcred \
  --docker-server=ghcr.io --docker-username=<user> --docker-password=<token>
```

```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

EKS lo ECR images ki secret avasaram ledu. Node IAM role tho pull avthayi.

## Env vs file: edi vadali

- Simple values, app env vars expect chesthe: env.
- Config files (nginx.conf, application.yaml), certs: volume.
- Secrets ki volume konchem better: env vars process dumps, crash logs lo leak ayye chance.

## Interview one-liners

- ConfigMap vs Secret: rendu key-value; Secret sensitive data ki, base64, separate RBAC, tmpfs lo mount.
- Secret encrypted aa: default ga base64 matrame; etcd encryption at rest enable cheyyali (EKS KMS).
- ConfigMap update aithe pod lo env maruthunda: ledu, restart kavali. Volume mount aithe file update avthundi.

Lab: [labs/README.md#lab-07](labs/README.md#lab-07-configmap-and-secret)
