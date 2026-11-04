# 21. Service mesh, Istio

[Index](README.md) | Prev: [20](20-observability.md) | Next: [22. EKS Auto Mode](22-eks-auto-mode.md)

Prerequisites: [07 Services](07-services-dns.md), [08 Ingress/Gateway API](08-ingress-gateway-api.md), [15 networking](15-networking-deep-dive.md), [17 controllers/CRDs](17-controllers-crds-operators.md)

Ee prerequisites clear aithe Istio easy. Istio = CRDs + controller (istiod) + proxies. Kotha concept emi ledu.

## Problem

`frontend`, `orders`, `payments` services. Production lo kavali:
- Retries, timeouts, circuit breaking
- Canary: kotha version ki 10% traffic
- Services madhya encryption (mTLS), identity-based authorization
- Per-request metrics, tracing

Prathi service code lo prathi language lo rayadam duplicate work. **Service mesh** ee pani network layer lo chesthundi, app code marchakunda.

## Architecture

### Data plane: Envoy sidecars

- Prathi pod lo app pakkana `istio-proxy` (Envoy) container inject avthundi (mutating admission webhook, [16](16-security.md)).
- `istio-init` init container leda Istio CNI plugin iptables rules set chesi pod in/out traffic antha Envoy dwara pampisthundi.

```
frontend app -> frontend Envoy ==(mTLS)==> orders Envoy -> orders app
```

### Control plane: istiod

- `istio-system` namespace lo.
- Istio CRDs (VirtualService, etc.) + Services/Endpoints watch chesi Envoy config ga convert, xDS API tho anni proxies ki push. (Controller pattern, [17](17-controllers-crds-operators.md).)
- Certificate authority: prathi workload ki ServiceAccount batti SPIFFE identity cert (`spiffe://cluster.local/ns/shop/sa/orders`), auto-rotate.

### Ambient mode (sidecar-less)

- Prathi node meeda **ztunnel** DaemonSet: L4 + mTLS.
- L7 features (routing, retries) kavalante namespace/service ki **waypoint** proxy (Envoy Deployment).
- Pod restart avasaram ledu, sidecar overhead ledu.

```bash
kubectl label namespace shop istio.io/dataplane-mode=ambient
```

## Install, sidecar injection

```bash
istioctl install --set profile=demo -y           # demo = learning ki, prod ki "default" + tuning
kubectl label namespace shop istio-injection=enabled
kubectl rollout restart deployment -n shop        # existing pods ki sidecar
kubectl get pods -n shop                          # READY 2/2 (app + istio-proxy)
istioctl analyze -n shop
```

## Traffic management

### DestinationRule: destination policies, subsets

```yaml
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: orders
  namespace: shop
spec:
  host: orders
  trafficPolicy:
    connectionPool:
      http:
        http1MaxPendingRequests: 100
    outlierDetection:              # circuit breaking: errors ichhe pods ni temporary eject
      consecutive5xxErrors: 5
      interval: 30s
      baseEjectionTime: 60s
  subsets:
  - name: v1
    labels: { version: v1 }
  - name: v2
    labels: { version: v2 }
```

### VirtualService: routing rules

```yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: orders
  namespace: shop
spec:
  hosts: [orders]
  http:
  - match:
    - headers:
        x-beta-user:
          exact: "true"
    route:
    - destination: { host: orders, subset: v2 }    # beta users -> v2
  - route:
    - destination: { host: orders, subset: v1 }
      weight: 90
    - destination: { host: orders, subset: v2 }
      weight: 10                                   # canary 10%
    timeout: 5s
    retries:
      attempts: 3
      perTryTimeout: 2s
      retryOn: 5xx,connect-failure
```

Canary ki rendu Deployments (`orders-v1`, `orders-v2`), same `app: orders` label (Service rendu ni select), different `version` label (subsets).

### Gateway (ingress into mesh)

```yaml
apiVersion: networking.istio.io/v1
kind: Gateway
metadata:
  name: shop-gw
  namespace: shop
spec:
  selector:
    istio: ingressgateway
  servers:
  - port: { number: 80, name: http, protocol: HTTP }
    hosts: ["shop.example.com"]
---
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: frontend
  namespace: shop
spec:
  hosts: ["shop.example.com"]
  gateways: [shop-gw]
  http:
  - route:
    - destination: { host: frontend, port: { number: 80 } }
```

Istio Kubernetes **Gateway API** (`Gateway` + `HTTPRoute`, [08](08-ingress-gateway-api.md)) kuda support chesthundi. Kotha setups ki adi recommended direction.

### ServiceEntry: external services

```yaml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: stripe
spec:
  hosts: [api.stripe.com]
  ports:
  - { number: 443, name: https, protocol: TLS }
  resolution: DNS
  location: MESH_EXTERNAL
```

`outboundTrafficPolicy: REGISTRY_ONLY` pedithe ServiceEntry lo leni external hosts block. Egress control.

## Security

### mTLS

```yaml
apiVersion: security.istio.io/v1
kind: PeerAuthentication
metadata:
  name: default
  namespace: shop
spec:
  mtls:
    mode: STRICT          # PERMISSIVE = mTLS + plaintext rendu accept (migration ki)
```

Migration: PERMISSIVE tho start, anni workloads mesh lo ki vachhaka STRICT.

### AuthorizationPolicy

```yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: payments-allow-orders
  namespace: shop
spec:
  selector:
    matchLabels: { app: payments }
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/shop/sa/orders"]   # ServiceAccount identity
    to:
    - operation:
        methods: ["POST"]
        paths: ["/charge"]
```

NetworkPolicy (IP/port, L3/L4) vs AuthorizationPolicy (identity + HTTP method/path, L7). Rendu kalisi defense in depth.

## Observability

- Envoy metrics: `istio_requests_total`, `istio_request_duration_milliseconds` (source, destination, response code labels tho). Prometheus scrape.
- **Kiali**: mesh graph, traffic flow, config validation.
- Tracing: Envoy spans generate, kani app `traceparent` / `x-b3-*` headers forward cheyyali.

```bash
# istioctl download chesina folder (istio-<version>/) lo samples untayi, installed version tho match avthayi
kubectl apply -f istio-*/samples/addons/prometheus.yaml
kubectl apply -f istio-*/samples/addons/kiali.yaml
istioctl dashboard kiali
```

## Debugging

```bash
istioctl analyze -A                              # config mistakes
istioctl proxy-status                            # anni proxies istiod tho sync aa
istioctl proxy-config routes <pod> -n shop       # Envoy routes
istioctl proxy-config clusters <pod> -n shop
istioctl proxy-config endpoints <pod> -n shop
kubectl logs <pod> -c istio-proxy -n shop        # Envoy access logs
istioctl x describe pod <pod> -n shop            # ee pod ki e policies apply avthunnayi
```

Common issues:
- Pod `1/1` (sidecar ledu): namespace label ledu leda label pettaka restart cheyyaledu.
- Protocol detect avvadam ledu: Service port `name: http-...` leda `appProtocol: http` ivvu.
- STRICT mTLS tarvata non-mesh clients fail: PERMISSIVE leda client ni mesh lo ki.
- 503 `UH`/`NR` Envoy response flags: no healthy upstream / no route. VirtualService host, DestinationRule subset labels check.
- Init containers network access fail: sidecar inka start avvaledu. Native sidecar mode / `holdApplicationUntilProxyStarts`.

## Trade-offs

- Sidecar per pod: CPU/memory overhead, konchem latency (prathi hop). Ambient takkuva.
- Operational complexity: upgrades (canary upgrades with revisions), CRD learning curve.
- 3-4 services ki overkill. Linkerd lighter alternative. AWS App Mesh discontinued, EKS lo Istio common choice. VPC Lattice AWS-managed service-to-service networking alternative.

## Interview one-liners

- Service mesh ante: service-to-service traffic ki infrastructure layer (mTLS, routing, observability) app code marchakunda.
- Istio components: istiod (control plane), Envoy sidecars / ztunnel+waypoint (data plane).
- VirtualService vs DestinationRule: routing "ekkadiki" vs destination ki "ela" (subsets, LB, circuit breaker).
- Sidecar vs ambient: per-pod Envoy vs per-node ztunnel + optional waypoint.

Lab: [labs/README.md#lab-19](labs/README.md#lab-19-istio-canary-optional)
