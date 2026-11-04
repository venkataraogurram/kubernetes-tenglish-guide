# 16. Security: securityContext, Pod Security, images, admission

[Index](README.md) | Prev: [15](15-networking-deep-dive.md) | Next: [17. Controllers, CRDs](17-controllers-crds-operators.md)

Prerequisites: [12 RBAC](12-namespaces-rbac-serviceaccounts.md), [15 NetworkPolicy](15-networking-deep-dive.md)

## Layers (4C model)

```
Cloud (IAM, VPC, EKS endpoint access)
 └─ Cluster (RBAC, admission, audit logs, secrets encryption)
     └─ Container (image, securityContext, runtime)
         └─ Code (dependencies, app vulnerabilities)
```

Okka layer weak aithe migathavi kapadalevu. Prathi layer lo basics.

## securityContext

Container ki Linux-level restrictions. Default ga chala containers **root** ga run avthayi. Container escape aithe node risk.

```yaml
spec:
  securityContext:                    # pod level
    runAsNonRoot: true
    runAsUser: 10001
    runAsGroup: 10001
    fsGroup: 10001                    # volumes ee group ki writable
    seccompProfile:
      type: RuntimeDefault            # dangerous syscalls block
  containers:
  - name: app
    image: myapp:1.0
    securityContext:                  # container level (pod level ni override)
      allowPrivilegeEscalation: false # setuid tho root avvalemu
      readOnlyRootFilesystem: true    # container FS lo rayalemu
      privileged: false
      capabilities:
        drop: ["ALL"]                 # Linux capabilities anni teeseyi
        add: ["NET_BIND_SERVICE"]     # port <1024 bind kavalante matrame
    volumeMounts:
    - name: tmp
      mountPath: /tmp                 # readOnlyRootFilesystem tho app ki writable /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

Common errors ivi set chesaka:
- `container has runAsNonRoot and image will run as root`: image lo `USER` ledu. `runAsUser` ivvu leda Dockerfile lo `USER 10001`.
- `Read-only file system`: app `/tmp`, cache dirs ki rasthondi. emptyDir mount cheyyi.
- Standard `nginx` image root kavali (port 80, cache dirs). Non-root ki `nginxinc/nginx-unprivileged` (port 8080).

## Pod Security Standards (PSS) + Pod Security Admission (PSA)

Built-in admission controller. Namespace label tho levels enforce:

| Level | Ardham |
|---|---|
| `privileged` | Restrictions emi levu (system namespaces) |
| `baseline` | Known dangerous things block: privileged, hostNetwork, hostPath, hostPID |
| `restricted` | Best practice: non-root, drop ALL caps, seccomp, no privilege escalation |

```bash
kubectl label ns shop \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/warn=restricted \
  pod-security.kubernetes.io/audit=restricted
```

- `enforce`: violate chese pods reject.
- `warn`: kubectl lo warning matrame.
- `audit`: audit log lo record.

Migration tip: modata `warn` matrame pettu, warnings fix chesi, tarvata `enforce`.

```bash
kubectl label ns shop pod-security.kubernetes.io/enforce=restricted --dry-run=server   # existing pods violate chesthaya chupisthundi
```

Old **PodSecurityPolicy** Kubernetes 1.25 lo remove ayyindi. Ippudu PSA + policy engines.

## Policy engines (custom rules)

PSA fixed levels matrame. Custom rules ("images ECR nundi matrame", "prathi pod ki team label", "`:latest` tag vaddu") ki:

- **Kyverno**: YAML policies, Kubernetes-native, easy.
- **OPA Gatekeeper**: Rego language.
- **ValidatingAdmissionPolicy**: built-in (CEL expressions), extra install ledu.

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: no-latest-tag
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups: ["apps"]
      apiVersions: ["v1"]
      operations: ["CREATE", "UPDATE"]
      resources: ["deployments"]
  validations:
  - expression: "object.spec.template.spec.containers.all(c, !c.image.endsWith(':latest'))"
    message: "':latest' tag allowed kaadu, fixed version vadu"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: no-latest-tag
spec:
  policyName: no-latest-tag
  validationActions: ["Deny"]
```

## Admission controllers (concept)

```
kubectl apply -> AuthN -> AuthZ (RBAC) -> Mutating admission -> Schema validation -> Validating admission -> etcd
```

- **Mutating**: request ni marchesthundi (Istio sidecar inject, default values add).
- **Validating**: allow/deny (PSA, Kyverno, Gatekeeper).
- Webhooks down aithe (`failurePolicy: Fail`) pod creation motham block avvochu. Production incidents ki common cause.

## Image security

- Fixed tags leda **digests** (`image@sha256:...`). `:latest` vaddu.
- Minimal base images: distroless, alpine, chainguard. Takkuva packages = takkuva CVEs.
- Scan: ECR enhanced scanning (Inspector), Trivy, Grype in CI.
- Sign + verify: cosign / Notation, admission lo verify.
- Private registry (ECR) + pull-through cache for public images.

## Secrets, API access, audit

- Secrets: [09](09-configmap-secret.md) (KMS encryption, External Secrets, RBAC).
- EKS API endpoint: private endpoint leda public CIDR restrict.
- Control plane logging: EKS `audit`, `authenticator` logs CloudWatch ki enable cheyyi. "Evaru e secret chadivaru" ani telusukovadaniki.
- Runtime threat detection: GuardDuty EKS Runtime Monitoring, Falco.

## Hardened Deployment template

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-web
spec:
  replicas: 2
  selector:
    matchLabels: { app: secure-web }
  template:
    metadata:
      labels: { app: secure-web }
    spec:
      serviceAccountName: secure-web
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        seccompProfile: { type: RuntimeDefault }
      containers:
      - name: web
        image: nginxinc/nginx-unprivileged:1.27
        ports:
        - containerPort: 8080
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities: { drop: ["ALL"] }
        resources:
          requests: { cpu: 50m, memory: 64Mi }
          limits: { memory: 128Mi }
        readinessProbe:
          httpGet: { path: /, port: 8080 }
        volumeMounts:
        - { name: tmp, mountPath: /tmp }
      volumes:
      - { name: tmp, emptyDir: {} }
```

## Interview one-liners

- PSP replacement: Pod Security Admission (PSS levels) + Kyverno/Gatekeeper/VAP.
- runAsNonRoot enduku: container escape aithe root privileges tho node compromise risk thaggisthundi.
- Mutating vs validating admission: modify vs accept/reject.
- Cluster security first steps: RBAC least privilege, PSA restricted, NetworkPolicy default deny, image scanning, secrets encryption, audit logs.

Lab: [labs/README.md#lab-14](labs/README.md#lab-14-pod-security-restricted)
