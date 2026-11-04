# 08. Ingress, Gateway API

[Index](README.md) | Prev: [07](07-services-dns.md) | Next: [09. ConfigMap, Secret](09-configmap-secret.md)

## Enduku

LoadBalancer Service = L4 (TCP), prathi service ki oka LB.
**Ingress** = L7 (HTTP/HTTPS). Okka load balancer tho chala services ki host/path batti route:

```
                     +--> /api   --> api-svc  --> api pods
https://shop.com --> ALB
                     +--> /      --> web-svc  --> web pods
```

TLS termination, host-based routing, path-based routing anni okka chota.

## Ingress controller kavali

Ingress object okkate emi cheyyadu. Daanni chadivi nijanga load balancer configure chese **Ingress controller** kavali:

| Controller | Ela pani chesthundi |
|---|---|
| EKS Auto Mode built-in (`eks.amazonaws.com/alb`) | AWS ALB create chesthundi. Install avasaram ledu |
| AWS Load Balancer Controller | Same idea, standard EKS lo Helm tho install |
| ingress-nginx | Cluster lo NGINX pods + oka LoadBalancer Service |
| Traefik, HAProxy, Kong | Vere options |

Cluster lo e controller unnayo **IngressClass** cheptundi:

```bash
kubectl get ingressclass
```

Empty aithe Ingress create chesina emi jaragadu (ADDRESS empty ga untundi).

## EKS Auto Mode lo ALB setup (3 objects)

```yaml
# 1. IngressClassParams: AWS-specific settings
apiVersion: eks.amazonaws.com/v1
kind: IngressClassParams
metadata:
  name: alb
spec:
  scheme: internet-facing        # leda internal
---
# 2. IngressClass: "ee class ni Auto Mode handle chesthundi"
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: eks.amazonaws.com/alb
  parameters:
    apiGroup: eks.amazonaws.com
    kind: IngressClassParams
    name: alb
---
# 3. Ingress: routing rules
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80
```

IngressClass, IngressClassParams cluster-level, okkasari create chesthe chalu. Tarvata prathi app ki Ingress matrame.
Auto Mode ALB ki subnets tags kavali: public subnets ki `kubernetes.io/role/elb=1`, private ki `kubernetes.io/role/internal-elb=1`. eksctl/console tho create chesina cluster lo usually already untayi.

```bash
kubectl get ingress web          # ADDRESS lo ALB DNS name vasthundi (2-3 min)
kubectl describe ingress web     # events lo errors
```

## Ingress rules detail

```yaml
spec:
  ingressClassName: alb
  tls:
  - hosts: [shop.example.com]
    secretName: shop-tls           # ingress-nginx lo. ALB lo ACM cert IngressClassParams/annotation tho
  rules:
  - host: shop.example.com         # host-based
    http:
      paths:
      - path: /api
        pathType: Prefix           # /api, /api/, /api/orders anni match
        backend:
          service: { name: api, port: { number: 80 } }
      - path: /
        pathType: Prefix
        backend:
          service: { name: web, port: { number: 80 } }
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service: { name: admin, port: { number: 80 } }
```

`pathType`:
- `Prefix`: path segments prefix match (`/api` -> `/api/x` match, `/apix` kaadu).
- `Exact`: exact match matrame.
- `ImplementationSpecific`: controller istam.

ALB lo HTTPS ki ACM certificate: Auto Mode `IngressClassParams.spec.certificateARNs` leda auto-discovery (host name tho match ayye ACM cert). Details: [EKS docs, IngressClassParams reference](https://docs.aws.amazon.com/eks/latest/userguide/auto-configure-alb.html).

## Ingress limitations

- Konni features (header routing, traffic weights, timeouts) controller-specific **annotations** tho matrame. Controller marchithe annotations anni marchali.
- HTTP matrame properly. TCP/UDP/gRPC routing standard ga ledu.
- Okka object lo infra team + app team concerns kalisipothayi.

Ee problems solve cheyyadaniki **Gateway API** vachindi.

## Gateway API (Ingress next generation)

Roles ga split chesindi:

```
GatewayClass   (infra provider: "e controller")          <- cluster admin / cloud
   └─ Gateway  (listener: port 443, hostname, TLS)        <- platform team
        └─ HTTPRoute / GRPCRoute / TCPRoute (routing)      <- app team, own namespace lo
```

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shop-gw
spec:
  gatewayClassName: <controller-class>   # e.g. istio, amazon-vpc-lattice, envoy-gateway
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: web
spec:
  parentRefs:
  - name: shop-gw
  rules:
  - matches:
    - path: { type: PathPrefix, value: /api }
    backendRefs:
    - name: api-v1
      port: 80
      weight: 90                # traffic split standard ga, annotations lekunda
    - name: api-v2
      port: 80
      weight: 10
  - backendRefs:
    - name: web
      port: 80
```

Gateway API CRDs cluster lo default ga undavu, install cheyyali. Controller kuda kavali (Istio, Envoy Gateway, AWS Load Balancer Controller recent versions, VPC Lattice controller).
Kotha projects ki Gateway API direction. Ingress ippatiki chala widely used, deprecated kaadu.

## Ingress vs LoadBalancer vs Gateway API

| | Service LoadBalancer | Ingress | Gateway API |
|---|---|---|---|
| Layer | L4 | L7 HTTP | L4 + L7 |
| LBs | Prathi service ki okati | Shared | Shared |
| Traffic split | No | Annotations | Native (`weight`) |
| Role separation | No | No | Yes |
| EKS Auto Mode | NLB built-in | ALB built-in | Separate controller kavali |

## Security note

`internet-facing` ALB/NLB ante internet lo evaraina reach avvochu. Learning lab lo:
- Test ayyaka immediately delete cheyyi (ALB/NLB gantaku charge).
- Possible aithe `kubectl port-forward` tho test cheyyi, LB avasaram ledu.
- NLB ki `loadBalancerSourceRanges` tho nee IP matrame allow cheyyi.

## Interview one-liners

- Ingress controller lekapothe Ingress: emi jaragadu, just object.
- Ingress vs Service LoadBalancer: L7 shared LB vs L4 per-service LB.
- Gateway API advantage: role-oriented, portable traffic splitting, header matching, multiple protocols.

Lab: [labs/README.md#lab-06](labs/README.md#lab-06-ingress-with-eks-auto-mode-alb)
