# 11. Resources, QoS, probes

[Index](README.md) | Prev: [10](10-storage-pv-pvc-storageclass.md) | Next: [12. Namespaces, RBAC](12-namespaces-rbac-serviceaccounts.md)

## requests vs limits

```yaml
resources:
  requests:
    cpu: 250m          # scheduling ki: "naaku kanisam inthe kavali"
    memory: 256Mi
  limits:
    cpu: "1"           # max (CPU throttle avthundi)
    memory: 512Mi      # max (daati vadithe OOMKilled)
```

| | requests | limits |
|---|---|---|
| Evaru vadatharu | **Scheduler**: node lo enough free requests unna chota pettadaniki | **kubelet/kernel**: runtime lo enforce |
| CPU daatithe | Free CPU unte vadukovachu | **Throttle** (slow avthundi, kill avvadu) |
| Memory daatithe | Free unte vadukovachu, kani node pressure lo first evict | **OOMKilled** (exit code 137) |

### Units

- CPU: `1` = 1 vCPU. `500m` = 0.5 vCPU (m = millicores).
- Memory: `Mi`, `Gi` (binary, 1024). `M`, `G` decimal. `128Mi` vadu. `128m` ante 0.128 bytes, common typo.

### Best practice (simple rule)

- **Memory**: requests = limits (leda daggaraga). Memory compressible kaadu, overcommit OOM ki dari teesthundi.
- **CPU**: requests set cheyyi. CPU limit chala teams pettaru (throttling latency penchuthundi), lekapothe generous ga pettu.
- Requests nijanga app vaade daaniki daggaraga undali. Ekkuva pedithe nodes waste (cost), takkuva pedithe overcommit.

EKS Auto Mode / Karpenter **requests** chusi e size node launch cheyyalo decide chesthundi. Requests lekapothe sariga plan cheyyadu.

## QoS classes

Kubernetes requests/limits batti prathi pod ki class isthundi. Node memory pressure lo **eviction order** ide.

| QoS | Condition | Eviction |
|---|---|---|
| `Guaranteed` | Prathi container ki CPU + memory requests == limits | Last |
| `Burstable` | Kanisam okka request/limit undi, Guaranteed kaadu | Middle |
| `BestEffort` | Requests, limits emi levu | **First** |

```bash
kubectl get pod web-xxx -o jsonpath='{.status.qosClass}'
```

## Namespace defaults: LimitRange, ResourceQuota

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: defaults
  namespace: dev
spec:
  limits:
  - type: Container
    default:            # limits ivvakapothe
      memory: 512Mi
    defaultRequest:     # requests ivvakapothe
      cpu: 100m
      memory: 256Mi
---
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.memory: 16Gi
    pods: "50"
    services.loadbalancers: "1"
```

ResourceQuota lo cpu/memory pedithe, aa namespace lo prathi pod requests ivvali (leda LimitRange defaults). Lekapothe pod create reject.

## Probes

Kubelet container health check chese 3 types:

| Probe | Question | Fail aithe |
|---|---|---|
| **startupProbe** | App start ayyinda? | Success ayye varaku migatha probes run avvavu. Fail threshold daatithe restart |
| **readinessProbe** | Traffic ki ready aa? | Service endpoints nundi remove. **Restart cheyyadu** |
| **livenessProbe** | App bathike unda (hang kaaledu kada)? | Container **restart** |

### Example

```yaml
startupProbe:
  httpGet: { path: /healthz, port: 8080 }
  failureThreshold: 30
  periodSeconds: 5           # 30 x 5 = 150s varaku start ki time
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 3
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 2
```

Probe mechanisms: `httpGet` (200-399 success), `tcpSocket` (port open aa), `exec` (command exit 0), `grpc`.

### Probe mistakes (production outages ki common cause)

- **Liveness lo dependency check** (DB ping) pettadam: DB down aithe anni pods restart loop, problem inka worse. Liveness lo app own health matrame.
- Readiness lo dependencies check cheyyochu, kani anni pods okesari NotReady aithe total outage. Jagratha.
- Slow-starting app (Java/Spring) ki liveness `initialDelaySeconds` takkuva: start avvakamunde kill, CrashLoop. **startupProbe** vadu.
- `timeoutSeconds` default 1s. GC pause / load lo probe timeout, unnecessary restarts.
- Readiness lekapothe rolling update lo ready kaani pods ki traffic, errors.

## OOMKilled debug

```bash
kubectl get pod web-xxx
# STATUS: OOMKilled leda CrashLoopBackOff, RESTARTS perugutundi
kubectl describe pod web-xxx | grep -A5 "Last State"
#   Last State: Terminated  Reason: OOMKilled  Exit Code: 137
kubectl top pod web-xxx --containers     # actual usage (metrics-server kavali)
```

Fix: memory limit penchu leda app memory leak/heap settings (Java `-XX:MaxRAMPercentage=75`) chudu.

## CPU throttling

Pod slow, kani restart avvatledu, CPU limit daggara usage: throttling. `kubectl top` lo limit daggara undi ante limit penchu leda teesey.

## Interview one-liners

- requests vs limits: requests scheduling guarantee; limits runtime cap.
- CPU limit daatithe vs memory limit daatithe: CPU throttle, memory OOMKill.
- Readiness vs liveness: readiness traffic control, liveness restart.
- startupProbe enduku: slow start apps ni liveness kill cheyyakunda.
- BestEffort pods risk: node pressure lo first evict.

Lab: [labs/README.md#lab-09](labs/README.md#lab-09-probes-resources-and-oomkilled)
