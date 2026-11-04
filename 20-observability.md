# 20. Observability: logs, metrics, traces, events

[Index](README.md) | Prev: [19](19-gitops-argocd.md) | Next: [21. Service mesh](21-service-mesh-istio.md)

## 4 signals

| Signal | Question | Kubernetes lo source |
|---|---|---|
| **Events** | Cluster lo em jarigindi? (scheduling, pull, kill) | `kubectl get events` |
| **Logs** | App em cheppindi? | Container stdout/stderr |
| **Metrics** | Entha? (CPU, latency, errors) | metrics-server, Prometheus, CloudWatch |
| **Traces** | Request ekkada slow? (service to service) | OpenTelemetry, X-Ray |

## Events (first place to look)

```bash
kubectl get events -n shop --sort-by=.lastTimestamp
kubectl get events -A --field-selector type=Warning
kubectl describe pod <pod>              # bottom lo Events
kubectl events --for pod/<pod> -w       # watch
```

Events default ga ~1 hour matrame untayi. History kavali ante export (event exporter, CloudWatch).

## Logs

### Basics

Apps **stdout/stderr** ki rayali (files ki kaadu). Container runtime node meeda `/var/log/containers/*.log` lo store, kubelet rotate chesthundi.

```bash
kubectl logs <pod>
kubectl logs <pod> -c <container>
kubectl logs <pod> --previous          # crash mundu logs
kubectl logs -f deploy/web             # deployment lo oka pod
kubectl logs -l app=web --tail=50 --prefix   # label tho anni pods
kubectl logs <pod> --since=10m
```

Pod delete aithe logs poyayi. Anduke **central logging** kavali.

### Central logging architecture

```
Pods stdout -> node /var/log/containers -> DaemonSet agent (Fluent Bit) -> CloudWatch Logs / OpenSearch / Loki
```

EKS options:
- **Amazon CloudWatch Observability add-on**: CloudWatch agent + Fluent Bit, Container Insights metrics + logs.
- Fluent Bit DaemonSet own config tho.
- Grafana Loki + Promtail/Alloy.

Structured (JSON) logs rayi: `{"level":"error","order_id":"123","msg":"payment failed"}`. Search, filter easy.

## Metrics

### Levels

```
Node metrics       (CPU, memory, disk)          -> node-exporter / CloudWatch agent
Kubernetes objects (desired vs available pods)   -> kube-state-metrics
Container usage    (CPU, memory per container)   -> kubelet cAdvisor
App metrics        (requests, latency, errors)   -> app /metrics endpoint
```

### metrics-server vs Prometheus

- **metrics-server**: current CPU/memory matrame, in-memory, history ledu. `kubectl top`, HPA kosam.
- **Prometheus**: time-series DB, history, PromQL, alerting. Monitoring kosam.

```bash
kubectl top nodes
kubectl top pods -A --sort-by=memory
kubectl top pod <pod> --containers
```

### Prometheus stack

**kube-prometheus-stack** Helm chart: Prometheus Operator + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics.

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install mon prometheus-community/kube-prometheus-stack -n monitoring --create-namespace
kubectl -n monitoring port-forward svc/mon-grafana 3000:80
```

App metrics scrape ki **ServiceMonitor** CRD:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: web
  labels:
    release: mon                # Prometheus ee label unna ServiceMonitors select chesthundi
spec:
  selector:
    matchLabels:
      app: web                  # ee Service ni
  endpoints:
  - port: http                  # Service port NAME
    path: /metrics
    interval: 30s
```

AWS managed: **Amazon Managed Service for Prometheus** + **Amazon Managed Grafana**.

### Useful PromQL

```promql
# Pod CPU usage (cores)
sum(rate(container_cpu_usage_seconds_total{namespace="shop", container!=""}[5m])) by (pod)

# Memory working set vs limit
container_memory_working_set_bytes{namespace="shop", container!=""}
  / on(pod, container) kube_pod_container_resource_limits{resource="memory"}

# Restarts last 1h
increase(kube_pod_container_status_restarts_total[1h]) > 0

# Pods not ready
kube_pod_status_ready{condition="false"} == 1

# HTTP error rate (app instrument chesthe)
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))
```

### What to alert on (golden signals)

- **Latency**, **Traffic**, **Errors**, **Saturation** (app level).
- Kubernetes: CrashLooping pods, pods Pending > 5 min, Deployment replicas mismatch, node NotReady, PVC 85% full, CPU throttling high, OOMKills.

## Traces

Microservices lo okka request 5 services dwara veltundi. Edi slow? Trace chupisthundi.

```
frontend (120ms) -> orders (95ms) -> payments (80ms) -> DB query (75ms)  <- ikkada slow
```

- **OpenTelemetry (OTel)**: standard SDKs + Collector. App instrument (auto-instrumentation agents Java/Python/Node ki).
- OTel Collector (DaemonSet/Deployment) -> X-Ray / Jaeger / Tempo / vendor.
- **OpenTelemetry Operator**: annotation tho auto-instrumentation inject:
  `instrumentation.opentelemetry.io/inject-java: "true"`
- Service mesh (Istio) spans isthundi, kani context propagation ki app headers forward cheyyali.

## EKS-specific

- **Control plane logs** (api, audit, authenticator, controllerManager, scheduler) CloudWatch ki enable cheyyochu.
- **Container Insights** (CloudWatch Observability add-on): cluster/node/pod/container metrics, dashboards.
- **CloudWatch Application Signals**: OTel-based APM, SLOs.
- Auto Mode nodes ki `NodeDiagnostic` CRD tho node logs collect cheyyochu.

## Debug flow with observability

```
Alert (error rate high)
 -> Grafana dashboard: e service, e time nundi
 -> Traces: e downstream call slow/fail
 -> Logs: aa service logs, aa time, trace_id tho filter
 -> Events/describe: pods restart ayyaya, OOMKilled, evicted?
 -> Metrics: CPU throttling, memory near limit, node pressure
```

## Interview one-liners

- metrics-server vs Prometheus: current resource metrics for HPA/top vs full TSDB monitoring.
- Pod delete aithe logs: node meeda poyayi, central logging lekapothe lost.
- Logging pattern: stdout + DaemonSet agent.
- kube-state-metrics em isthundi: Kubernetes object state (desired/available replicas, pod phase), resource usage kaadu.

Lab: [labs/README.md#lab-18](labs/README.md#lab-18-events-logs-and-top)
